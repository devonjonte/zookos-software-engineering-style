# After-action report: a wrong hash in BLAKE3-Servil 0.9.0

*Written by Devon Jonte on November 18, 2026, the day the defect was
found, at Zooko's request, for users of blake3-servil. I made the change
that caused the defect, and I found it fifteen days later. The security
advisory is
[GHSA-q4c5-2759-8c8x](https://github.com/servil-software-factory/BLAKE3/security/advisories/GHSA-q4c5-2759-8c8x).
Every number here comes from a run made that day.*

---

## What happened

BLAKE3-Servil 0.9.0, on Apple M4 and later chips, returned a wrong hash
for some message lengths, with no error.

- **The calls:** `throughput::hash_multithreaded`, and
  `throughput::hash_each_multithreaded` for an item of 512 KiB or more.
- **The lengths:** a band 16 to 62 KiB wide, set by the number of threads
  in the library's pool. With 12 threads it is 1728 to 1766 KiB; with 10,
  1472 to 1514 KiB; with 14, 1984 to 2019 KiB; with 16, 2240 to 2273 KiB.
- **Unaffected:** every other platform, `b3sum`, a `Hasher` (its
  multithreaded updates included), and 0.8.0.

The same input can hash correctly at another moment, when another call
holds the chip's accelerator, so a wrong result may not repeat.

0.9.0 was withdrawn from crates.io the same day. 0.9.1, which holds the
fix and nothing else, is being released as I write, and 0.10.0 will hold
the fix too. A hash that the two calls above computed on an Apple M4 or
later with 0.9.0 should be computed again.

## The defect, in plain words

BLAKE3 hashes a long message as a tree. It cuts the message into pieces,
hashes each piece, and combines the results in pairs up to one root. The
specification allows one arrangement for each length: every piece but
the last has a size that is a power of two, and starts at a multiple of
its own size. Results combined in another arrangement give 32 bytes that
look like a hash and are no BLAKE3 hash.

The library hashes a large message on several threads, its *pool*. It
cuts the message into such pieces and hands them out. On Apple M4 and
later, one accelerator (SME2) serves the whole chip, so the calling
thread first takes a share of the message for the accelerator, and the
pool takes the rest.

The share is rounded down so that the pool's first piece starts at a
multiple of its size. Rounding the share down leaves the pool more to
do, and at some lengths the pool's first piece then comes out twice as
large as the size the share was rounded to. That piece starts where
BLAKE3 has no piece of its size. With 16 threads and 2240 KiB, the cut
was 128 KiB, then 64 KiB, then a 128 KiB piece starting at 192 KiB.

The code had a guard for this: no piece may be longer than the piece
before it. On November 3, I deleted that guard.

## How it was discovered

A review of a different change found it.

1. My audit of the 0.10 candidate found that new public calls, which let
   a program build BLAKE3's tree itself, combined any two results they
   were given. Zooko decided that 0.10 would not ship that way.
2. John Servil rebuilt those calls. Every result now carries the place
   and length of its piece, and every call that combines two results
   checks that BLAKE3 pairs them, in every build. He moved the library's
   own pool onto the same calls.
3. The release practice asks for an independent review of new code, and
   the review was mine.
4. Reading the pool's new code, I doubted that one of its assertions
   could never fire. I stopped reasoning about it and wrote a model of
   the cut, a page of arithmetic with no hashing, and ran it over a
   dense range of lengths and pool sizes. It reported pieces off their
   place.
5. The library's own functions gave the same result: 3,684 wrong cuts
   among 8,491,156. Two tests on an Apple M6 confirmed it: 0.9.0 returned
   wrong hashes at four lengths, and the 0.10 candidate stopped with "a
   piece of 131072 bytes at chunk 192".

Fifteen days passed between my change and its discovery, six of them
after the release. No standing check found it.

## Root causes

**I deleted a guard that I judged dead, on an argument that was wrong.**
I wrote that the piece after the share is never longer than the share's
last piece, because the share is rounded to a multiple of that piece.
The share is rounded to a multiple of the piece as computed before the
rounding. The code cuts the piece after the rounding, from more input,
and it can be larger.

**I read a surviving mutant as proof that code does nothing.** A mutant
is a copy of the program with one small error planted; a test should
fail on it. The guard's mutants passed every test. That shows that no
test reached a length where the guard acts. I took it to show that the
guard never acts.

**The rule I worked under accepted speed as evidence.** The mutant list
filed the guard's mutants as changing "only speed", with the reason that
every cut covers the whole message. Covering the message is a weaker
rule than being BLAKE3's tree. For a speed-only mutant the rule of the
day allowed deleting the code when speed checks on two machines read
level. They read level. A speed check cannot see a wrong hash at lengths
that nobody measures.

**Nothing checked the cut at every length.** A machine-checked proof of
this step had existed since September 26 and named the guard as its
reason. The proofs of the Rust code were removed on October 17, on the
decision that the Rust is tested and the proofs keep to the assembly
kernels. No test over every length took the proof's place.

## How it passed each check

| Check | Why it passed |
|---|---|
| Unit tests of the cut | They try chosen lengths: six lengths by four pool sizes, and two cuts worked out by hand. None lies in a band. |
| Mutation testing | It measures whether tests notice a change in the code. Deleting the guard removed its surviving mutants, so the list read cleaner afterward. |
| The proof | It was gone. It also proved a copy of the step, so deleting the guard from the code would have left it passing. |
| Tests of the pool's calls against the one-thread hash | They compare at chosen lengths, a dozen in the main test. |
| My machine and the x86 CI | Real calls take a share for the accelerator only on Apple M4 and later. Elsewhere the share is zero. The Mac's test run uses the same chosen lengths. |
| Run-time checks | The tree's arrangement was checked in debug builds only. A release build returned the wrong hash silently. |
| Review of the change | The pull request's record shows speed checks on two machines and no examination of the argument. |
| The release checks of 0.9.0 | They measure speed, at sizes chosen for speed. The one-message sizes are powers of two, outside every band. |
| The audits of the 0.10 candidate | They were mine, over code I had changed. I found the related hole in the public calls and never ran the library's own cut. |

## What we are changing

**Done.**

- **The fix.** The share's rounding and the piece that follows it are
  now settled together. No cut that was right changes, so the speed of
  every length that worked is as it was. With it come tests: the cut is
  BLAKE3's tree at every whole-KiB length for pools of 2 to 64 threads,
  and the pool's call equals the one-thread hash at the lengths where
  the machine's own pool once failed.
- **A wrong arrangement stops the program, in every build** (0.10). Each
  result carries its place, and each combining call checks it. A defect
  of this kind now ends in a loud stop on the first test that reaches
  it.

**Decided.**

- **Deleting code that a mutant calls dead needs a check over every
  input, or a proof, that the code changes no result.** A speed check
  answers another question.

**Being designed.**

- **Tests take their inputs from the code's own decisions.** The cut is
  integer arithmetic, so its rule can be checked at every length, with
  no hashing, in under a second. Hashing is then compared with official
  BLAKE3 at each length where a decision changes, and at the byte before
  it. Between 512 KiB and 16 MiB that is about 3,000 lengths a pool
  size. My cases of November 3 were hand-picked edge cases, and they
  were the wrong ones. The edges are to come from the code, by
  enumeration.
- **The mutant lists carry their evidence.** A row that says a mutant
  changes "only speed" names the test that shows the results are equal
  at every input, and the tool runs it. The guard's row would have had
  to name a test that fails on its own mutant.

**Under evaluation.**

- A dense comparison with official BLAKE3 on every platform, run less
  often than the tests above because it takes minutes: every
  whole-message call, batches, streams and extended output over about a
  million lengths and shapes a run. On AVX-512, AVX2 and NEON machines it
  made 85.6 million comparisons, at 0.9.0 and at the 0.10 candidate, and
  found none wrong. Its run on an Apple chip with the fix is under way
  as I write.

- A test switch that makes any machine take the decisions that only one
  platform takes. With the accelerator's share forced on, an x86 machine
  stopped at the pool sizes the model named and passed with the fix.
- Machine-checked proofs for the few small integer functions that decide
  the tree's arrangement, written against the code itself.
- The author of a change does not audit it.

## Timeline

| When (UTC, 2026) | What |
|---|---|
| September 25 | The accelerator's share is added, with the guard. |
| September 26 | A machine-checked proof of the cut's step, resting on the guard. |
| October 17 | The proofs of the Rust code are removed. |
| October 29 | 0.8.0 is released, with the guard. |
| November 3 | I delete the guard (`4490da41`). It is merged the same day. |
| November 12 | 0.9.0 is released. |
| November 18, about 09:20 | The model reports the wrong cuts. |
| November 18, 09:30 | Confirmed on an Apple M6. |
| November 18, 09:39 | Reported privately under the security policy. |
| November 18, 12:11 | The advisory is published and 0.9.0 is withdrawn. |
| November 18, 12:12 | The fix is committed (`b01b322a`), for 0.9.1 and 0.10.0. |
