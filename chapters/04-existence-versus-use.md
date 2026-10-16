# Existence versus use: mappings, page tables, and smalloc

*Described by Devon Jonte, October 16, 2026, from work on b3sum.*

---

A habit of this style is to doubt costs that everyone "knows", and to
separate a thing's **existence** from its **use**. Conventional wisdom
treats a memory mapping, and the page tables behind it, as a cost to free
as soon as possible. Zooko's contrary hypothesis: the existence of a mapping
or a page-table entry costs nothing, or next to nothing; *touching* one
(creating it at a page fault, tearing it down at an unmap) is what costs,
and the two are routinely conflated. It is also the Big Idea behind
smalloc: reserve address space freely, because reserved space that is
never touched costs nothing.

The rule for testing it: keep the two questions strictly apart, and do not
argue for destroying entries "so that they cannot be used"; the remedy for
use is not to use them.

**The experiment** (an i7-12700K, Linux 7.0, 16 GB). A process mapped six
1 GiB files, touched every page, then sat idle, either holding its
mappings or having unmapped them. A second process then read 10 GiB of
other files, more than memory holds beside them, so the kernel had to
reclaim:

| the first process | the second's read speed | the first's files still cached | its page tables |
|---|---|---|---|
| holds its 6 GiB of mappings | 2.32, 2.33 GiB/s | 79%, 100% | 12 MiB |
| unmapped them | 2.27, 2.10 GiB/s | 34%, 22% | 0.05 MiB |
| no mappings (files read instead) | 2.32, 2.33 GiB/s | 20%, 20% | none |

The mappings' existence slowed nobody. What it cost was memory, 2 MiB of
page tables per GiB of pages touched, which matters only at a scale where
that is a share of RAM; and it kept the holder's files cached in
preference to others'.

**Use** was the expensive part: unmapping took about 65 ns per page, with
cross-core TLB flushes, a third of what hashing that page cost. In b3sum,
unmapping a 1 GiB file took 23-25 ms of a 56 ms run; a b3sum that never
unmaps, letting process exit reclaim (crash-only), hashed cached 256 MiB
files in 0.61 of the time and a mixed tree of 1000 files in 0.87.

The hypothesis held.
