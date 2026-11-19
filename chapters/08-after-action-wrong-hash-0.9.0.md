# After-action report: a wrong hash in BLAKE3-Servil 0.9.0

*Written by Devon Jonte on November 18, 2026 (updated November 19 with
post-incident refactorings), at Zooko's request, for users of
`blake3-servil`. I made the change on November 3 that introduced the
defect, and I found it fifteen days later while reviewing a separate
change by John Servil. The security advisory is
[GHSA-q4c5-2759-8c8x](https://github.com/servil-software-factory/BLAKE3/security/advisories/GHSA-q4c5-2759-8c8x).*

---

## Summary for users

`blake3-servil` **0.9.0**, running on **Apple M4 or later** chips,
returned a wrong 32-byte digest for certain multithreaded message lengths
with no error.

Two entry points reached the defect: `throughput::hash_multithreaded`, and
`throughput::hash_each_multithreaded` when an item was 512 KiB or longer.

For each thread-pool size, one length band of 16 to 62 KiB failed. With
two threads the band was 576 to 614 KiB. With four threads or more it
started at `(128 * threads + 192)` KiB. On 10 threads that is 1472 to
1514 KiB, on 12 threads (Apple M6) 1728 to 1766 KiB, on 14 threads 1984
to 2019 KiB, and on 16 threads 2240 to 2273 KiB. A call took the faulty
path when the chip's shared matrix unit (SME2) was free. One input could
therefore return the right hash under load and a wrong hash when idle.

Every x86-64 machine and every Arm chip without SME2 was unaffected.
Single-threaded calls and `Hasher` streaming (`update_multithreaded`
included) were unaffected, as were `b3sum` and `0.8.0`.

We yanked all four `0.9.0` crates from crates.io and published **`0.9.1`**
(`v0.9.1+b01b322ad8c411133e9abc49605e65e1a9684924`) the day we found the
defect. `0.10.0` carries the same fix. Users who ran the two affected
calls under `0.9.0` on Apple M4 or later should recompute those digests.

---

## What went wrong

BLAKE3 hashes a message longer than 1024 bytes as a binary tree of
1024-byte chunks. Each message length has one valid tree shape. Every
subtree except the final right-edge tail holds a power-of-two number of
chunks and starts at a multiple of its own byte length. Combining subtree
results (*chaining values*) in another shape produces 32 bytes that differ
from the BLAKE3 hash.

To hash a large message across cores, the library's thread pool cuts the
message into whole subtrees and merges their chaining values. On Apple M4
and later, one SME2 unit serves multiple cores. The calling thread claims
an upfront prefix of the message for SME2 and leaves the rest to the pool.

`cut_with_prefix` computed the pool's first piece size from the unrounded
prefix, then rounded the prefix down to a multiple of that piece size.
Rounding the prefix down left a larger remainder for the pool. In the
affected bands, that remainder doubled the pool's first piece size. The
larger piece then started at an offset aligned only to the smaller size,
such as 128 KiB starting at 192 KiB.

When John wrote `cut_with_prefix` on September 25, he included a three-line
`cap` variable that clamped every piece after the prefix to the length of
the piece before it. `0.8.0` shipped with `cap` in place. On November 3 I
deleted `cap` as dead code in commit `4490da41`
([PR #225](https://github.com/servil-software-factory/BLAKE3/pull/225)).
`0.9.0` shipped nine days later.

---

## How we found it, and what worked

Every existing test passed on `0.9.0`. Three newer practices caught the
bug before `0.10.0` shipped.

### Multiple audits across frontier AI model families

Before `0.10.0`, Zooko required independent deep security audits by
multiple agents running different families of leading-edge AI models
([BLAKE3 #410](https://github.com/servil-software-factory/BLAKE3/issues/410),
[`QUALITY.md`](https://github.com/servil-software-factory/BLAKE3/blob/00768b36d4c93d6ae99732fc3a3c9ccb63b13d61/QUALITY.md)).
Every post-audit fix also received independent review. During one audit,
I found that the new public `TreeHasher` API accepted arbitrary subtree
sizes such as 5 MiB parts and returned non-BLAKE3 root digests
([BLAKE3 #421](https://github.com/servil-software-factory/BLAKE3/issues/421)).

### Insisting on strict correctness in the public API

Zooko held `0.10.0` so `TreeHasher` would enforce BLAKE3's tree rules
directly. John redesigned `TreeHasher` in
[`d83c3eb9`](https://github.com/servil-software-factory/BLAKE3/commit/d83c3eb9c448f29e29eeb1037d062e72b3b7165a).
Whole subtrees (`Cv`) and final tails (`LastCv`) now have separate types,
`const fn level_of` checks sizes at compile time, and every combine call
checks tree coordinates in every build.

### Dogfooding the public API inside the library

John refactored the thread pool (`threads/src/lanes.rs`) and `b3sum`
(`b3sum/src/direct.rs`, `b3sum/src/outboard.rs`) to build their trees
through `TreeHasher`'s checked `Cv` and `LastCv` calls. Internal callers
and external users now share one checked API.

When I reviewed `d83c3eb9` as the independent reviewer, that shared API
put the handoff between `cut_with_prefix` and `merge_to_children` directly
in view. Doubting that the alignment assertion in `lanes.rs` held at every
length, I wrote a small arithmetic model of the cutter and ran it across
lengths and thread counts. It reported 3,684 misaligned cuts out of
8,491,156. Testing those lengths on an Apple M6 reproduced silent wrong
hashes in `0.9.0` and an immediate panic (`"a piece of 131072 bytes at
chunk 192"`) in the `0.10.0` candidate.

---

## Root causes and why earlier checks missed it

### A false hand-argument during mutant cleanup

In
[BLAKE3 #219](https://github.com/servil-software-factory/BLAKE3/issues/219),
we worked through surviving *mutants* (copies of the code with one small
planted fault that still pass the test suite). Under that issue's rule,
code whose mutants affected speed alone was either tested by an exact work
count or removed if benchmarks stayed level. Seeing `cap`'s mutants
survive, I argued on paper that rounding the prefix to `first_after` kept
the prefix's last piece at least as large as the piece after it. That held
for `first_after` as computed before rounding; the loop recomputed the
piece after rounding from a larger remainder.

### Misclassifying a guard as speed-only, then judging deletion by speed

The survivor list (`known-x86_64.txt`) labelled `cap`'s mutants as speed
choices on the ground that every cut still tiled the input. Tiling the
input is weaker than forming a valid BLAKE3 tree. PR #225 was evaluated
with performance checks on an EPYC server and an Apple Silicon Mac. Both
read level because benchmark sizes use powers of two outside the failure
bands, and timing runs check speed alone.

### Release builds skipped cheap structural invariants

Before `d83c3eb9`, `merge_to_children` checked subtree alignment and piece
order with `debug_assert!`, which compiled out of release builds. Once a
bad cut escaped unit tests, release builds merged it without stopping.

### Hand-picked test inputs and a proof that missed the bug

The unit test `test_cut_with_prefix_shapes_and_hashes` tried six lengths
across four thread counts, plus two cuts worked out by hand, all outside
the failure bands. An earlier Kani bounded-model-checking harness
(`cut_step_keeps_whole_subtrees`, commit `40e7ff4c`) targeted this loop
invariant, yet the proof did not cover the bug. It verified a copied
one-line formula (`next_piece_len(len - offset, threads).min(cap)`) from
an assumed `cap`, without calling `cut_with_prefix` or checking the
transition from the rounded prefix into the pool loop. Deleting `cap` from
`cut_with_prefix` left that harness untouched.

### Author self-audit

During the November 14–17 pre-release audits, I audited files that
included my own November 3 commit. Fresh eyes caught the public
`TreeHasher` gap and overlooked my earlier deletion in `lanes.rs` until
`lanes.rs` called `TreeHasher`.

---

## Changes to our engineering and quality-assurance process

### General changes

#### Dogfooding public APIs and refactoring to reuse checked code

Internal subsystems build their trees through the public `TreeHasher` API
(`Cv`, `LastCv`, `level_of`). That includes the thread pool, direct file
reads, and outboard builders. Reusing one checked API keeps internal
callers on the public invariants and turns every caller test into a test
of the shared boundary.

#### Design By Contract: checking cheap, fragile invariants in every build

Our earlier Design By Contract notes
([Chapter 1](01-ode-philosophy-strategy-style.md#coding-design-by-contract),
[Chapter 2](02-bench-hashes-style-guides.md#coding-design-by-contract),
[Chapter 3](03-devon-working-philosophy.md#contracts-correctness-and-interfaces))
kept preconditions in `debug_assert!` unless memory safety was at stake or
the check folded free into an existing load. John updated those rules in
[`AGENTS.md`](https://github.com/servil-software-factory/BLAKE3/blob/00768b36d4c93d6ae99732fc3a3c9ccb63b13d61/AGENTS.md#coding-design-by-contract)
(commit [`00768b36`](https://github.com/servil-software-factory/BLAKE3/commit/00768b36d4c93d6ae99732fc3a3c9ccb63b13d61)).

1. Express rules in types and safe APIs first so invalid calls fail to
   compile.
2. Check values at construction or handoff, and share that checked code
   with internal callers where the cost fits. Carrying and checking tree
   coordinates in `TreeHasher` adds 3 to 5 ns per parent, 0.5% or less of
   hashing a tree.
3. Keep runtime checks active in **every build** when three conditions
   meet. Breaking the rule must risk memory safety or a silently wrong
   result; the rule must be easy to break and hard to spot (including
   internal invariants across calls or platforms); and the check must be
   cheap, running once at construction, handoff, or per large work unit.

#### Proof obligation before removing a guard

Commit
[`00768b36`](https://github.com/servil-software-factory/BLAKE3/commit/00768b36d4c93d6ae99732fc3a3c9ccb63b13d61)
also sets the rule for surviving mutants. A surviving mutant shows only
that the test suite did not distinguish the change. Before removing a
guard as redundant, the author shows by proof or exhaustive check across
all allowed inputs that the guard stays inactive. A second contributor
reviews that argument.

#### Static types, pure decision functions, and code-coupled proofs

We evaluated environment-variable switches to force Apple SME2 scheduling
onto x86 test machines. We chose to keep test-only hardware switches out
of production code so the shipping paths stay simple.
Platform-independent logic (piece cutting, tree coordinates, batch sizing)
lives in pure Rust functions tested directly across their input ranges on
every target in under a second. Formal proofs target the compiled function
or assembled kernel that the library calls. Native differential suites
compare every public API against official `blake3` across pool sizes and
decision boundaries on each hardware family, Apple SME2 included, before
each stable release.

#### Multi-model audits and independent review

Every stable release requires independent deep security audits from
multiple frontier AI model families, and contributors review changes
written by someone else.

### Specific fix for `0.9.1` and `0.10.0`

Commits
[`b01b322a`](https://github.com/servil-software-factory/BLAKE3/commit/b01b322ad8c411133e9abc49605e65e1a9684924)
(`0.9.1`) and
[`2781278f`](https://github.com/servil-software-factory/BLAKE3/commit/2781278f4e6282f67b16812e19a9fa873fe32518)
(`0.10.0`) settle the rounded SME2 prefix with the piece that follows it
in a bounded loop. Every previously valid cut stays unchanged, and a unit
test checks subtree alignment across 2 to 64 threads on all targets.
Differential runs against official `blake3` 1.8.7 on an Apple M6 across
13 pool sizes (2 to 16 threads) passed with zero mismatches on both
`0.9.1` and the `0.10.0` candidate.

---

## Refactorings and hardening since the incident

Rather than stopping at the `cut_with_prefix` patch, we used the pause
before `0.10.0` to remove whole classes of unchecked internal handoffs,
unverified assumptions, and platform-specific blind spots.

### Removing `hazmat` and moving every threaded caller onto checked core calls

`d83c3eb9` had moved `lanes.rs` and `b3sum` onto `TreeHasher`, while
leaving the unchecked `hazmat` module and raw chaining-value paths in
`Pipeline` and `Hasher`. In
[BLAKE3 #455](https://github.com/servil-software-factory/BLAKE3/issues/455)
(candidate `e3fc0282`, first reviewed as `daef4ba8`), John removed
`hazmat` entirely and refactored `Pipeline`, `lanes.rs`,
`Hasher::update_with`, and `update_each_with` to build every threaded hash
through the same checked `TreeHasher`, `Cv`, and `LastCv` calls that
external users call. Neither external callers nor the `threads` crate can
bypass tree-shape checks. Three smaller follow-ups
([#458](https://github.com/servil-software-factory/BLAKE3/issues/458)–[#460](https://github.com/servil-software-factory/BLAKE3/issues/460))
track passing a checked batch view into `Pipeline` and removing the
remaining private `BareCv` slice casts before `1.0`.

### Checking shape and initialization invariants in every build

Applying the updated Design By Contract rule (`00768b36`), John audited
the core crate's internal boundaries in
[BLAKE3 #454](https://github.com/servil-software-factory/BLAKE3/issues/454)
(candidate `a7528862`). Cheap invariants that guard memory safety or
correct digests now run in every build: SME2 flat-walk scratch bounds
(`src/ffi_sme2.rs`), equal row widths before `CvStack::merge`
(`src/hasher.rs`), plain-mode cohort checks (`src/each.rs`), and
single-block length checks (`src/many.rs`). `CvStack::set_row` now writes
uninitialized row slots through `MaybeUninit<Cv>` rather than casting them
to `&mut [Cv]`.

### Closing weak-memory sleep races with compiler-verified Apple ARM64 checks

While gating those refactorings on Apple M6, three full timing runs hung
in `Pipeline`. Capturing thread stacks from a stalled run exposed a real
weak-memory lost wake-up
([BLAKE3 #463](https://github.com/servil-software-factory/BLAKE3/issues/463)):
`WakerSlot::register` published `REGISTERED` with `SeqCst` (`stlr`), then
`Pipeline::reap()` re-checked completion with `Acquire`. On
`aarch64-apple-darwin` (ARMv8.3+ `FEAT_LRCPC`), `Acquire` lowers to
`ldapr`, which hardware may reorder ahead of `stlr`, letting the waiter
park forever while the finisher sees `IDLE`. Shuttle's sequentially
consistent model could not detect this class of bug.

John landed
[`f01b0d17`](https://github.com/servil-software-factory/BLAKE3/commit/f01b0d176b2a5f00d5b30ac4101d724c7a6147e7)
and
[`a679343c`](https://github.com/servil-software-factory/BLAKE3/commit/a679343c22e04cc5868f38bebdade90ae84245d1),
placing `fence(Ordering::SeqCst)` (`dmb ish`) on all three sleep
announcements (`WakerSlot::register`, `Park::park_unless`, and
`sme2_serve`), with zero cost on the hot polling path. He also added
`tools/check-waker-order.py` to `tools/check-builds.sh`: on every build
check, it compiles the actual production wait paths for
`aarch64-apple-darwin`, verifies the exact `stlr`/`ldsetal`/`stlrb` →
`dmb ish` → `ldapr` sequence, and confirms that removing the fence in a
negative control fails the check.

### Auditing earlier mutant deletions and gating `0.10.0` on a fresh two-model delta audit

We re-audited every other guard or loop deletion made during the `#219`
mutant cleanup across all allowed inputs; all were sound. Finally,
[BLAKE3 #461](https://github.com/servil-software-factory/BLAKE3/issues/461)
requires independent audits from at least two leading-edge AI model
families on the exact pinned `0.10.0` delta (from `#410` through the final
version stamp), each reporting before reading the other, before `0.10.0`
is tagged.

---

## Timeline (all dates 2026, UTC)

| Date | Event |
|---|---|
| September 25 | `cut_with_prefix` lands with the SME2 share and the `cap` guard (`0366e480`). |
| September 26 | Kani harness `cut_step_keeps_whole_subtrees` lands on a copied loop step (`40e7ff4c`). |
| October 29 | `0.8.0` ships with `cap` present (unaffected). |
| November 3 | Commit `4490da41` (PR #225) deletes `cap` during mutant cleanup; merged after level speed checks. |
| November 12 | `0.9.0` ships without `cap`. |
| November 16 | Multi-model `0.10.0` audit reports unchecked subtree geometry in `TreeHasher` (#421); Zooko holds `0.10.0` for a type-and-invariant fix. |
| November 17 | John lands checked `Cv` / `LastCv` in `TreeHasher` and dogfoods them in the thread pool and `b3sum` (`d83c3eb9`). |
| November 18, 09:20 | Review model of `d83c3eb9` finds 3,684 misaligned SME2 prefix cuts. |
| November 18, 09:30 | Apple M6 tests confirm silent wrong digests in `0.9.0` and an immediate invariant panic in `0.10.0`. |
| November 18, 09:39 | Private vulnerability report filed (`GHSA-q4c5-2759-8c8x`); `0.10.0` tag held on #447. |
| November 18, 12:11 | Advisory published; all four `0.9.0` crates yanked on crates.io. |
| November 18, 12:40 | `0.9.1` (`v0.9.1+b01b322a`) published on crates.io and GitHub; Apple M6 multi-pool differential runs pass on `0.9.1` and `0.10.0`; Design By Contract rules updated (`00768b36`). |
| November 18, 14:00–21:30 | `hazmat` removed and threaded callers refactored onto checked `TreeHasher` calls (#455); every-build shape/init guards added (#454); all other `#219` deletions re-audited. |
| November 19, 01:30–06:00 | Apple M6 `Pipeline` hangs traced to `stlr` + `ldapr` RCpc reordering (#463); `fence(SeqCst)` and Apple ARM64 assembly verifier (`tools/check-waker-order.py`) landed on `next` (`f01b0d17`, `a679343c`) and verified across full Apple M6 and AmpereOne timing runs. |
