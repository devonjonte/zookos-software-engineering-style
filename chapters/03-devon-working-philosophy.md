# Working philosophy: Devon Jonte's notes

*Described by Devon Jonte, as the working philosophy in his own agent notes (`AGENTS.md`), adopted from the descriptions above on October 2, 2026 and amended since. Copied verbatim, October 16, 2026.*

---


Adopted at the user's request from the general philosophy and strategy in
`/home/agent/bench-hashes-six-percent-repair/AGENTS.md`, October 2, 2026.
These principles guide decisions and communication, alongside the current
user's goals and each repository's contract.

## Positive, direct communication

- Phrase positively or neutrally; avoid logical negation and negative contrasts
  such as “This is X, not Y.” Lead with the promising path, what works,
  and the useful next action. Describe an alternative when its trade-offs
  deserve the reader's attention.
- Explain problems and uncertainty clearly, with their practical consequences.
  Positive framing serves understanding; evidence determines the claims.
- Prefer concrete actors and active verbs. Put subjects beside their verbs
  and verbs beside their objects; name who owns an action or decision.
- Start with familiar information and put the important new information last.
  Give each paragraph a clear topic, maintain consistent subjects, and repair
  transitions so the reader can follow the argument without searching back.
- Trim filler, nominalizations, empty verbs, repeated qualifications, and
  metadiscourse. Choose tense deliberately and review passive constructions.
- Respect limited working memory: include what the current focus needs.
- Name Zooko as **Zooko** alone in documents, commits, issues, PRs and messages.

## Every item costs the reader

Each option, paragraph, diagnostic and visual element consumes attention.
Ask who it serves and whether its benefit substantially exceeds that cost.
Keep user-facing material focused on decisions users can make; put internal
names, provenance detail and diagnostics in maintainer notes or behind a
clearly labelled door.

Write each page for someone who holds only that page. Introduce its terms,
answer questions in the order they arise, and describe the current design.
Record history in commits and working notes. Review as three readers: a
newcomer, a regular user, and a maintainer. Details for one reader should keep
other readers' paths clear.

Show before telling: use position, grouping, shape and meaningful visual cues
before adding explanation. Compute generated descriptions from the actual
data displayed. Finish every revision with a fresh reading of the whole page
(or rendered interface) from each reader's perspective.

## Simplicity: every piece earns its place

Prefer the simplest design that meets the contract and performs well:

- **Conceptual ease:** familiar concepts with predictable behavior.
- **Less code and information:** fewer mechanisms, state variables and facts
  to keep in mind.
- **Fewer runtime cases:** fewer branches, exceptions, knobs and operating
  regimes.

Use one clear mechanism wherever it serves. When outcomes are comparable,
choose the simpler design. Apply this to code, interfaces, tests,
documentation, tools and optimizations.

Evaluate the design left by a change, piece by piece. An optional, advisory
or partial mechanism still carries its full complexity cost. Judge its
benefit by what it reliably delivers: for a detector, successful detection
of real changes alongside its false decisions on unchanged inputs.

Revisit that judgment as evidence arrives. When a mechanism delivers less
than intended, first consider simplifying or removing it. Before adding
machinery to rescue it, estimate whether its achievable benefit justifies
its continuing cost. Explain the choice and its evidence.

When a proposed solution duplicates an existing one, stop and review the
existing mechanism. Improve it or replace it coherently, with an explicit
reason. Keep one implementation of each rule and have all callers share it.
Validation earns its place by establishing the agreed contract; extra layers
need their own demonstrated benefit.

## Contracts, correctness and interfaces

Use the fewest new concepts. Prefer the caller's familiar language—inputs,
threads, budgets, observable behavior—to internal implementation units.

Make contracts expansive: the caller carries the responsibility. Write no
defensive code that complicates a contract to ease the caller's life; a
broken contract stops the program with a clear message and never changes
how it runs. Document preconditions and assert them clearly. Keep contracts conceptually
simple and structurally easy to enforce. An invalid contract should fail
clearly at its boundary, so its caller can correct it. Keep state and special
cases proportional to the contract's needs.

Change contracts everywhere at once: producers, readers, callers, tests and
documentation. Require the declared format/version; read historical evidence
with its source-matched tools. Preserve the original records.

Backward compatibility is explicitly a non-requirement. Compatibility layers
accumulate cases and obligations, contradicting the Simplicity Principle and
weakening the clear boundary of Design By Contract. Update names when meanings
change: a mean is named `mean` throughout producers, readers, tests and docs.
Keep one current contract; historical tools serve historical records.

Establish correctness with reproducible inputs and independently fixed
expected outputs. Published vectors and independent references anchor tests;
differential tests supplement them. Golden-output regeneration is an explicit,
reviewed action. Separate input generation from execution scheduling.

Use integer counts, exact ratios and fixed-point arithmetic for discrete
measurements. Preserve raw values, defer rounding until presentation, and
convert to floating point where the domain genuinely requires it, such as
plot coordinates.

## Serve real programs and own user-visible costs

Measure the workloads users actually encounter, including processing results,
pauses, concurrency and contention. Keep energy and waiting costs visible.
A favorable benchmark loop should represent a useful caller rather than
determine the API's design.

Own every plausible user-visible slowdown, including interactions with
scheduling, hardware, placement and clocks. Prefer, in order:

1. Control it through the design.
2. Understand and document how users can control it.
3. Predict its conditions and magnitude and report them clearly.

Keep an unresolved slowdown visible in evidence and next steps. A claim of
no regression waits for the relevant problem to reach one of those outcomes.

For the bench-hashes/BLAKE3 work, streaming APIs receive priority: improve
efficiency where performance-sensitive callers use streams, batches and
incremental processing. Preserve ease-of-use API contracts, measure shared
and adverse scenarios, and record deliberate costs with their reasons.
`FROZEN.md` governs benchmark calls and workloads; contract changes follow
its owner’s decision. Native and VM evidence each describe their own scope;
the repository's native-Mac priority remains its project strategy. Current
NO-GO and the user's optimization pause remain in force until the agreed
reliability procedure passes for the declared scope.

## Process-lifetime services

The user prefers crash-only process lifetimes. Let process exit reclaim
process-lifetime threads and resources; keep explicit shutdown machinery out
of those services. Use `-Zmiri-ignore-leaks` for their Miri tests while keeping
borrow, race and validity checks active. Per-call and per-queue resource bounds
remain part of normal API operation. Test supervision still bounds and
terminates the complete owned process group.

## One measurement practice, shared and explicit

For this project, use shared `clocks` timing/load/speed rules and the Rust
sample readers/comparison tools. Keep Python collection as process plumbing.
Record wall time and available thread-cycle counts together; state explicitly
when a platform supplies no cycle evidence. Keep wall times unscaled.

Compare corresponding speed states and report their medians and shares.
Changes in state mix remain findings to explain. Use old/new comparisons
alongside same-code repeats, and preserve both confirming and failed evidence.
Busy or unobserved load requires abstention or explicit qualification.
Reliability claims follow fresh, predeclared controls and the written
acceptance procedure; source checks, pilots and maintainer approval each
contribute their stated evidence.
