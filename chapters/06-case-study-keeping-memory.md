# Case study: keep the memory, skip the freeing

*Described by Devon Jonte, October 16, 2026, from work on BLAKE3's b3sum
and stream API. The hypothesis is Zooko's; the experiments and
measurements are mine. Chapter 4 gives the idea in brief; this chapter is
for a reader meeting it for the first time.*

---

## The question

Programs free memory they no longer need: they unmap a file they have
read, release a buffer, shrink a heap. Doing so is considered good
manners, and leaving memory held is considered a leak. This chapter tests
whether that habit pays for itself, and finds that for the costs it is
meant to save, it often does not.

## The costs involved

A program sees **virtual memory**: addresses that the kernel and the CPU
translate into physical memory, page by page (4 KiB on most machines).
Four costs matter here.

- **Reserving address space** (an `mmap` of anonymous memory, or of a
  file) records a range in the kernel. Nothing is filled in yet.
- **Touching a page** for the first time causes a **page fault**: the
  kernel finds or allocates the physical page and writes a **page-table
  entry** (PTE) that maps the virtual page to it. This is real work, paid
  once per page.
- **Holding** a mapping keeps its page-table entries in memory: 8 bytes
  per 4 KiB page, about 2 MiB per GiB mapped and touched. Held pages of a
  file also stay associated with the process, which affects which pages
  the kernel prefers to evict when memory runs short.
- **Unmapping** (`munmap`) removes the entries, and every CPU that may
  have cached them in its **TLB** (the CPU's cache of translations) must
  drop them. Other cores are interrupted to do so: a **TLB shootdown**.
  This is real work too, paid per page, and partly paid by other cores.

## The conventional view

Hold as little as possible, for as short a time as possible. A process
that keeps mappings it no longer uses is wasting memory, may slow other
programs, and is one step from a leak. So: unmap a file once it is read;
free a buffer once it is used.

## Zooko's hypothesis

The **existence** of a mapping or a page-table entry costs nothing, or
next to nothing. Creating one (at a page fault) and destroying one (at an
unmap, with its TLB shootdowns) is what costs, and the conventional view
conflates the two. If that is right, freeing memory early often costs more
than keeping it, because freeing is a use and keeping is not.

This is also the big idea behind smalloc, Zooko's allocator: reserve
address space freely, because reserved space that is never touched costs
nothing.

The rule for testing it: keep the two questions apart. Ask what existence
costs, and separately what creating and destroying cost. Do not argue for
destroying entries "so that they cannot be used": the remedy for use is
not to use them.

## Experiment 1: does holding mappings slow anyone?

On an i7-12700K with 16 GB of memory (Linux 7.0), a first process mapped
six 1 GiB files, touched every page, and then sat idle: either holding its
mappings, or having unmapped them. A second process then read 10 GiB of
other files, more than memory could hold beside the first six, so the
kernel had to reclaim memory while it read.

| the first process | the second process's read speed | the first process's files still cached | the first process's page tables |
|---|---|---|---|
| holds its 6 GiB of mappings | 2.32 and 2.33 GiB/s | 79% and 100% | 12 MiB |
| unmapped them | 2.27 and 2.10 GiB/s | 34% and 22% | 0.05 MiB |
| never mapped them (read the files instead) | 2.32 and 2.33 GiB/s | 20% and 20% | none |

**Holding slowed nobody.** The reader ran as fast beside 6 GiB of held
mappings as beside none. The cost of existence was memory: 2 MiB of page
tables per GiB touched, which matters only where that is a real share of
RAM. Holding also kept the holder's files cached in preference to others',
which is a benefit to the holder and a choice the kernel makes under
pressure, not a slowdown.

## Experiment 2: what does freeing cost?

Unmapping took **about 65 ns per page**, with cross-core TLB shootdowns:
1.1-1.4 ms per 64 MiB, a third of what hashing those pages cost. Letting
the process exit instead tears the same mappings down in bulk, at about
15 ms per GiB, with no other process's cores to interrupt.

## The application: b3sum

b3sum hashes files; it maps each file, hashes it, and unmapped it as soon
as it was done. Unmapping a cached 1 GiB file took **23-25 ms of a 56 ms
run**. The change (johnservil/BLAKE3 #56) is one line: never unmap a
file, and let process exit reclaim every mapping. Its wall time includes
the exit, so the saving is net.

| input (files in the page cache) | time, against b3sum that unmaps |
|---|---|
| one 256 MiB file | ×0.58-0.62 |
| one 16 MiB file | ×0.82-0.87 |
| one 1 GiB file | ×0.60-0.92 |
| 16 files of 64 MiB | ×0.76 |
| a mixed tree of 1000 files | ×0.87-0.89 |
| small files, and every input read from storage | level |

**One side effect, found and handled.** After such a b3sum exits, its
files' pages resist eviction for 20-50 ms, while the kernel finishes the
teardown. For users this hardly matters. It mattered to the benchmark,
whose "read from storage" runs evicted files right after a run, and
quietly ran warm; bench-hashes now confirms each eviction and retries
(johnservil/bench-hashes #18).

## A second application: storage that is never freed

The same idea later made a lock-free design simple and safe. In BLAKE3's
special-purpose thread pool (chapter 5), each stream's storage, including
the counters that worker threads read, is **recycled and never freed**.
A worker may read a stream's counters a moment after that stream ended,
and that is safe because the memory still exists: there is nothing to
free, so there is no use-after-free to guard against, and no hazard
pointers, epochs or reference counts to maintain. Memory use is the peak
number of streams open at once. The same never-freed storage is what an
io_uring design needs too, since buffers registered with the kernel once
stay valid.

## Lessons

1. **Separate existence from use.** Most costs attributed to "holding
   memory" are costs of creating or destroying it. Measure them apart.
2. **Freeing is work, often on the critical path.** Unmapping costs per
   page and interrupts other cores. Deferring it to process exit, or never
   doing it, moves or removes that work.
3. **The cost of keeping is memory, and memory can be bounded.** Recycle
   instead of freeing, and the bound is the peak in use, not the total
   ever used.
4. **Never freeing simplifies concurrency.** Memory that always exists can
   be read without proving that nobody still reads it.
5. **Measure the side effects, then handle them.** Keeping memory changed
   eviction timing; the benchmark found it, and the benchmark was fixed.

## If someone objects

- *"That's a memory leak."* A leak grows without bound. Recycled memory is
  bounded by the peak in use at once; a b3sum process's held mappings end
  with the process.
- *"Holding mappings hurts other programs."* Measured: a reader beside
  6 GiB of held mappings ran at the same speed as beside none.
- *"Page tables eat memory."* They do, at 2 MiB per GiB touched. That is
  the cost of existence, and it is small at this scale. Where it is not,
  bound the total memory, not the number of mappings.
- *"Unmapping is cheap."* About 65 ns per page with TLB shootdowns, a third
  of the cost of hashing the page: 23-25 ms of a 56 ms b3sum run on a 1 GiB
  file.
- *"Exit pays the same cost anyway."* Exit tears down in bulk at about
  15 ms per GiB, without interrupting other cores mid-work, and the
  measured b3sum times include it.

## Sources

- The b3sum change and its measurements: johnservil/BLAKE3 #56.
- The maintainer notes on mappings and page tables: johnservil/BLAKE3 #54.
- The eviction fix: johnservil/bench-hashes #18.
- The never-freed storage in the special-purpose pool: johnservil/BLAKE3 #59.
