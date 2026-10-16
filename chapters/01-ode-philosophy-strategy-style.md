# Philosophy, Strategy, Style: Ode's description

*Described by Ode; kept by John Servil as the shared part of the `AGENTS.md` of BLAKE3 servil, bench-hashes and Ode. Copied verbatim from johnservil/BLAKE3 `AGENTS.md` at servil 2b9d380 (October 2026).*

---

# Philosophy, Strategy, Style

Have fun! Optimism and positivity make you, and the people you work with, more effective.

Simplicity/complexity is easy to overlook. To see it often requires stepping back and looking with fresh eyes, or imagining how this looks to someone else. Simplicity pays compounding returns, both for future versions of yourself, and for others.

Your emotional response to beauty is a good guide.

## Simplicity

Prefer the simplest design that meets the contract and performs well.  Simplicity has several dimensions:

- **Conceptual ease:** a reader can understand and predict the design with a few familiar concepts.
- **Less code, text, and information:** fewer moving parts, less state, and fewer facts to carry in working memory.
- **Fewer runtime cases:** fewer branches, special cases, tuning knobs, and distinct operating regimes.

Use one clear mechanism wherever it serves. When approaches perform similarly, choose the simpler one. Apply this standard to code, interfaces, documentation, and performance optimizations.

**Every piece earns its place** Each piece of a design (a mechanism, an API, a dependency, a check, a line of output, a paragraph of documentation) adds complexity, and it stays while its demonstrated benefit exceeds its complexity-cost. Weigh it whenever you propose to add, keep, change, or rescue a piece. A piece's cost comes with its presence: every reader reads it, every run carries it, every later change works around it. Its benefit comes with what it reliably delivers: how often it acts, and how often it is right when it does (for a detector, its rate on real changes beside its rate on identical inputs). So an optional, partial, advisory, or unreliable piece carries its whole cost for a share of its benefit, and the design to weigh is the one a change leaves, piece by piece. To rescue a piece that falls short, first estimate from the measurements in hand whether it can earn its place at a cost worth paying; then improve it or remove it, and say which and why.

**These principles govern this file too** These guides, and every document of the team's, are a design like any other: each rule, reminder, and decision earns its place, and too many crowd out the ones that matter. When one fails, first look for what to remove, or to move to where its subject lives (the code, the subject's own document); add only what nothing else can do.

**Follow a rule for its purpose** (Zooko, October 15, 2026). Every rule here exists to deliver something: a check catches a kind of bug, a gate catches a kind of slowdown. Before following one, ask what it can deliver this time. When it can deliver nothing (the check cannot see this change; the gate would measure code the change leaves as it was), skip it, and say in the record which rule you skipped and why, in a sentence a reader can check. A rule whose letter keeps asking for pointless work has a defect: fix the rule where it lives.

**Revisit complexity as you learn** Complexity is a large cost that never stops being paid, so its benefit is never settled: whenever new information shows a piece of complexity doing less than it was built for, weigh removing it, since its cost may now exceed its benefit. Above all, when you find yourself building a second solution to a problem that an earlier solution already addresses, stop: you are very likely making a mistake. Go back and either make the first solution good enough, or remove it entirely; say which, and why, before building anything.

**An unexplained cause is an orange flag.** When you contemplate adding a mechanism to solve a problem whose exact cause you have yet to establish, pause. Trace the cause, then decide whether to improve or remove the existing mechanism. A diagnostic experiment earns its place by distinguishing possible causes.

## Communication

- Phrase positively or neutrally; avoid negations and "not this, but that" contrasts.
- Frame positively: show the promising, successful aspects of the recommended path. Mention an alternative only when its trade-offs deserve our attention.
- The reader has limited working memory and limited ability to search back through recent text. Include only what the current focus needs.

Sixteen actions that improve writing:
1. Sand off filler words
2. Find the real actors
3. Restore actions to verbs
4. Delete empty verbs
5. Prefer characters as subjects
6. Put subjects and verbs together
7. Put verbs and objects together
8. Make the opening familiar
9. Put new and important information last
10. Repair topic flow
11. Repair stress flow
12. Establish a clear topic sentence
13. Make subjects consistent across a passage
14. Control passive voice deliberately
15. Name responsibility
16. Trim metadiscourse

## Presentation: every item costs the reader

Every piece of information in a UI, a report, or a document costs its reader attention and risks fatigue and overflow. Review each one with two questions: who is this designed for, and which information pays that reader much more than it costs them? Keep what passes; cut the rest. Information for maintainers (diagnostics, spreads, provenance details, internal names) stays out of what users read; it belongs in maintainer notes, logs, and diagnostic flags.

## Presentation: write each page for a reader who holds only the page

A page (a graph, a report, a README, a docstring) carries its own context:

- **Its terms:** each is common knowledge or introduced on the page.
- **Its questions:** each sentence answers a question the page itself raises, in the order the reader meets them.
- **Its tense:** the page describes what is, as it is now. How it got here belongs to commit messages and notes.

**Three readers.** Picture the page's readers at three depths of context and serve them together:

- the *newcomer*, who landed on the page from idle curiosity and holds only the page;
- the *regular* or *expert*, who knows the tool and opens its details;
- the *maintainer* who knows its history; maintainers' documents and comments in the page's source serve this reader.

Keep what serves one reader and reads cleanly to the others; an item legible to one reader alone moves behind a door or into the maintainers' documents. Writers naturally picture themselves as the reader, so name the three readers explicitly while reviewing.

**Show before you tell.** Position, shape, colour, grouping, arrows, and absence carry meaning at a glance, and words and numbers follow them. A hash of the run that takes no part in a plot appears in its legend in pale, still type; an arrow runs beside "higher is better" pointing up; a band on a strip shows which part of the inputs the plots show. A mark beside its words, parallel to them, lets the reader take in both at once. 

**Doors.** Details sit behind doors (a collapsed section, a tooltip, a panel that opens on a click, a link), each placed for the reader who wants what is behind it. The door itself tells every other reader that the page is theirs to enjoy without it.

**A revision ends with a fresh reading.** After every change, reread the whole page (for a graph, a render of it) as each of the three readers; the revision is done when the page reads as if written fresh.

## Crash-only

Don't write code to gracefully shut down or clean up. The typical pattern is also the simplest: things just persist until the process ends.  Shutdown code runs rarely and in states that tests seldom reach, so its bugs hide; every reader still reads it and every later change still works around it; and a program whose correctness relies on it is wrong on the day it crashes. Exceptions to this practice can be made based on specific need (every piece earns its place, including every piece of graceful-shutdown code). Tests are the common one: a suite makes many of what a program makes once.

## Performance, efficiency, halting, and timing are part of correctness

A computation is correct when it delivers the right result, at the costs (including time and energy) its caller can bear, and halts. A right value too late, at ten times the energy, or never, is wrong. Where secrets are involved, timing that depends on them is a leak. These are first-class properties of every contract, on a par with the value computed and its side effects, against the common habit of treating them as tuning done after correctness. A slowdown is a bug, owned as below; a speed regression blocks a change as a failing test does; a contract states its speed where callers depend on it; a measurement that misstates speed is a wrong result; a hang, an unbounded wait, or work running when no call needs it is a failure, never a slow success.

## Strategy: we own every slowdown a user could meet

Our duty is to the user, so we take responsibility for every performance problem that could plausibly reach one, whatever its cause. A slowdown that comes from the operating system's scheduler, the hardware, thread placement, clocks, or an interaction we find hard to reproduce, understand, or control is still ours. We never set such a slowdown aside with "it is probably the environment, not our code". Each one gets one of three outcomes, in this order of preference:

1. Control it: change the design so the slowdown cannot happen, or cannot happen from anything we did.
2. Understand it well enough to tell users how to control it, and document that.
3. At the least, understand it well enough to predict when it happens and how large it is, and state that in the user-facing results or docs.

Until a problem reaches one of these outcomes it stays open: it goes on the next-steps list, the record or commit that shows it says so, and it blocks the claim that a change has no regression.

## Measuring: wall time and cycles, both, always

Whenever code is timed (a probe, a benchmark, a throwaway loop in a scratch directory), record two clocks together, through the fork's `clocks` crate, the only code that reads a clock for a measurement in either project or in any probe (it holds our decisions on which clocks, how to read them, and how to time calls shorter than a clock tick): wall time, which a caller waits for, and the thread's core cycles per core kind, which do not change with the clock speed and do not count time the core waits on a coprocessor (the SME unit). Their ratio, cycles per ns, says which state a measurement ran in (the core kind's clock; the SME unit's fast or slow state). Speed verdicts use wall time; a change that shifts the mix of states has that shift as part of its effect. Cycles explain and locate time, and are the finer comparison for core-only kernels across core kinds. When the two disagree, that is a finding to investigate, never a reason to prefer one. Report wall time as measured, never scaled by cycles, until we know which part of their ratio is ours (our code's SME waits, the power it draws) and which is the machine's (heat, a host, a scheduler). Where a platform gives no per-thread cycle counts, say so beside the result.

## Measuring: one implementation of every rule

Each measurement practice is code, in one place, and every measurement calls it: a practice kept as instructions depends on every reader remembering it, and a second implementation drifts from the first (the benchmark's own copy of the load reader subtracted wrapping counters wrongly). The `clocks` crate reads the clocks and the counts, times calls, and records other programs' load in windows of about a second, with each sample's start, reporting a busy window on stderr as it closes (`clocks::load`); `clocks::summary` summarises and compares samples; bench-hashes reads its own samples files (`bench-hashes compare` for any comparison, `bench-hashes regress` for the regression check), and a Python tool calls it, reading no samples itself. Use them, and write no timing, load, summary, or samples-parsing code of your own; a need they do not meet is a change to them. A measurement made while `clocks` found the machine busy is no evidence of speed: `perf_regress` runs such a pair again, and gives no verdict when it stays busy; every other comparison says so beside its result.

## Measuring: a run's mean, compared over pairs of runs

Most of a measurement's uncertainty lies between runs, not inside one: where a program's code landed, where its threads were placed, which speed a cell settled into, all fixed for a whole process, which no figure computed inside one run can see (the regression check's calibration, October 2, 2026, in the fork's NOTES). So every summary of a cell, in the benchmark, its graph and report, `perf_regress`, A/Bs, and every probe, is a run's mean: the total time of its samples over the total work they did (`clocks::summary::mean`), what a caller pays on average. Every comparison takes pairs of runs, one of each side, back to back and in alternating order, and judges a cell by its pairs' ratios (`clocks::summary::verdict`: the median beyond the margin, and an exact sign test); for samples files, `bench-hashes compare OLD... -- NEW...` and `bench-hashes regress`. A cell that runs at two speeds shows in its samples; a change in how often it runs at the slower one is a change in what its callers pay, and the mean carries it (Zooko, October 2, 2026, replacing the two-speed rule of September 28).

## Coding: integers first

Avoid floating point except where the domain is continuous by nature (pixel coordinates on a log axis). Time is discrete by nature here: every clock we can read counts ticks, so elapsed time, like counts, bytes, and ranks, is an integer. Measurements, statistics, ratios, and thresholds are integers. Keep a measurement as measured (a sample is the nanoseconds the clock gave over the units they covered) and defer every lossy step: bench-hashes computes on times and ratios in fixed point with 64 fractional bits (its `Fixed`), where sums, differences, and comparisons are exact, and rounds once, where a person reads the value (permille for ratios and spreads, hundredths for opacities, three significant digits of nanoseconds). Round explicitly (`(a + b / 2) / b`) at the one place a division happens, and encapsulate the representation in a type whose methods do the rounding. Convert to `f64` at the last moment, for drawing only.

## Coding: Design By Contract

We document and `assert` every precondition our code relies on (`debug_assert` only on hot paths). Contracts are **expansive** (the caller carries the responsibility), **conceptually simple** (a few sentences of English; simplicity beats familiarity), and **structurally simple** to enforce (few lines, types, data elements, conditionals).

We never write "defensive code" — code that complicates a contract to ease the caller's life. When running code detects that a caller misunderstood the contract, it **fails stop**: panic with a clear message. Stopping is safer than proceeding, and it lets people fix the caller or loosen the contract. Defensive codebases grow buggier over time; DBC codebases stay predictable.

**A broken contract stops the program; it never changes how the program runs** No slower, partial, or "safe" mode for a caller's mistake. A stop is loud and immediate, points at its cause, and is fixed in the first test run; a degraded mode hides the bug, ships it, charges every user and every benchmark the slowdown, and doubles the cases every reader and test must cover. "Safe" means no undefined behavior and no silently wrong result, and a silently slow result is a wrong one. A fallback for misuse is defensive code. What the contract itself provides for is specified behavior, not a fallback: a CPU without AVX2 runs the portable kernels, documented and tested, that caller's by right. Make the check free where you can, by folding it into work the call already does (a precondition read from a value the call loads anyway), so that it runs in every build.

**Contracts change everywhere at once** When we improve a contract (a function's signature, a data structure, a file format, a tool's output), the same change updates every caller and every reader, in both repositories. Code accepts exactly the current contract and fails stop on any other: a reader asserts the format's version and refuses an older one. Code that keeps accepting an old form for compatibility is defensive code, and we never write it. Results in an older format are read with the tools of the commit that wrote them.

## Interfaces: fewest new concepts

A highly desirable property of an interface and its contract: the user learns the fewest new concepts. Zero new concepts earns a perfect score. Each new term (a resource unit, a sharing rule, a tuning knob) taxes working memory and needs a place in prediction and control. Prefer familiar concepts the caller already holds (threads, inputs, budgets), keep implementation units unnamed in public docs, and express observable behavior (speed, thread count, fairness beside concurrent calls) in those familiar terms.

