# Case study: the measurement rule that kept changing

*Described by Devon Jonte, October 17, 2026, from a night's work on
bench-hashes and BLAKE3 servil's comparison tools. The user asked the
question this chapter answers ("We've changed this many, many times. I'm
not 100% sure why"); the investigation and the measurements are mine.*

---

## The question

A benchmark answers "is the new code faster than the old?" Over three
weeks, the rule bench-hashes used to answer that question changed more
than a dozen times: medians, then means; intervals added, removed, added
again; one test per cell, then all cells at once. Each change was
reasonable on its day. Taken together they looked like going in circles,
and false alarms kept returning: unchanged code flagged "slower" in a
dozen cells per comparison.

The user's question was why, and how to make it stop, while making
comparisons faster as well.

## What the history showed

Reading every commit, note and pull request about the rule gave five
patterns, each visible more than once.

1. **A rule calibrated for one question, reused for another.** The
   regression gate was calibrated for about a dozen cells: one test each
   at 3.5% false alarms gives a false alarm in about two checks of five. Then
   `compare` began judging all 700 cells "as `regress` judges". Seven
   hundred tests at 3.5% each give about twenty-five false alarms per
   comparison, which is what appeared. Nothing about the rule changed;
   only the number of questions asked of it.
2. **Adopted fixes waiting to land.** The right rule (every cell judged at
   once, the false discovery rate controlled) had been designed,
   measured and adopted the day before. It sat in a stack of unmerged
   pull requests while the day's measurements ran without it. The false
   alarms came back, and a third design was nearly written.
3. **Several drivers of one procedure.** The official tool, the gate, a
   script, and hand-written loops of runs each ran pairs slightly
   differently, one without fresh layouts, another without alternation.
4. **Reasons kept apart from the rule.** Calibrations lived in one
   repository's notes, causes in another's audit directory, decisions in
   pull requests. The documents described the rule as it stood, with no
   trace of the alternatives already measured and set aside. Each
   revisit began from nothing.
5. **Each fix revealed the next noise source.** Layout luck, then a
   misbehaving rule for two-speed cells, then cells that move 60% between
   processes, then address randomization, then a calibration probe that
   caught a cell still warming. That is progress. It reads as circling
   only when nobody records what each step removed.

## What stopped it

**One file holds the reasons.** `COMPARISONS.md` gives the rule, then each
choice with its evidence and what it gives up, the patterns above, a
checklist before changing anything, and the alternatives set aside.
Every place a maintainer starts (the agents' instructions, the
procedures, the methodology, the rule's own source code) points to it.
The next person to touch the rule meets its history before they meet the
rule.

**Measure with the adopted rule.** The stack landed that night. The
measurements that followed used it.

**One driver.** Options were added to the official tool rather than
written beside it.

## Making it faster, by measuring the measurement

The user proposed shorter runs and more pairs in the same time. The
principle was already written down ("most of a measurement's uncertainty
lies between runs, not inside one") but nobody had acted on it: each run
still took twelve samples of every cell, a number chosen when the goal
was one run's own median.

Three designs were run in equal time, with the same unchanged code on
both sides apart from a planted slowdown:

| design | false alarms in 624 unchanged cells | median interval width |
|---|---|---|
| 8 pairs × 12 samples | 0 | 4.6% |
| 18 pairs × 4 samples | 0 | 2.8% |
| 26 pairs × 2 samples | 0 | 3.7% |

**18 × 4 was about 40% narrower in the same time.** Timing the runs then
showed that 18 of a run's 24 seconds went to calibration: choosing each
cell's sample size, done afresh by every run. A comparison that
calibrates once and gives every run the same sizes took 17 minutes
instead of 26, and judged every group of cells more narrowly, because
identical sample sizes remove one difference between runs. Together,
about twice the information per minute.

Two lessons for any measurement:

- **The design follows the variance.** When most uncertainty lies between
  processes, buy processes, not samples.
- **Look at where the time goes before cutting iterations.** Most of a
  run was not sampling at all.

## The controls that moved

SHA-256 served as a control: its code is identical on both sides, so any
verdict on it means the comparison misread something. That night it
moved twice, by 2–4%, beside two changes to BLAKE3's thread pool.

The instinct is to call it noise. Counting cycles per sample showed a
physical cause instead. The CPU's performance cores share one clock,
4.9 GHz with one or two active and 4.7 GHz with all of them. A pool
change that woke more workers, or woke them more often, lowered the clock
for every core, SHA-256's included. The control was right: those changes
cost other programs on the machine about 3%. The rule had been correct;
the reading of it needed the cycles.

A second interaction surfaced the same way. After the pool had used every
core, Linux sometimes left the caller's thread on an efficiency core.
SHA-256 ran just as fast there; single-threaded BLAKE3 ran at half speed.
The same placement, two very different costs, and the checks had been
reporting it for weeks as "shared faster than solo".

## Mistakes worth recording

- **I duplicated my own work.** I wrote an energy counter from scratch,
  then found my own open pull request from the night before doing the
  same thing. The fix was to reconcile, keeping one implementation and
  crediting the better parts of the other. The lesson is the chapter's
  second pattern, at a smaller scale: read the open work before writing.
- **A test that needs a quiet machine fails in a gate.** Load-attribution
  tests passed on a quiet machine and failed beside five proof workers.
  They now measure the load beside them and judge against it, and a
  deliberately broken attribution still fails them, quiet or loaded.
- **Tracing perturbs what it traces.** Per-sample counter reads made a
  real 4% effect disappear. Traced runs explain; they don't decide.

## The general shape

Every item here is the same move: when a result surprises, measure the
measurement. Count the questions a rule is asked. Time the parts of a
run. Read the cycles behind a control's shift. And write down what each
step found, in the place the next person will look first, so that the
next change starts where this one ended.
