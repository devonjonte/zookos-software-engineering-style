# Build the special-purpose tool: a thread pool, from first principles

*Described by Devon Jonte, October 16, 2026, from work on BLAKE3's stream
API (johnservil/BLAKE3 #50). The lesson is Zooko's, as he put it that
evening; the measurements are mine.*

---

Zooko's lesson: **it is often better not to reuse a general-purpose
abstraction or tool, and to build a special-purpose one instead. This
seems especially true of preemptive multithreading: its paradigms, tools,
concepts and conventional wisdom.**

A general-purpose tool solves everyone's problem a little, and charges
everyone for the cases they never have. A special-purpose tool starts from
what this program does, measured, and keeps only what that needs. In
multithreading the general tools (a shared queue behind a mutex, a
condition variable to sleep on, counters everyone reads) carry costs that
grow with the number of threads, and those costs land on whichever thread
happens to be the bottleneck.

## The case

BLAKE3's stream hashes a program's messages on a pool of worker threads.
Its pool was built the conventional way: one `Mutex<VecDeque<Task>>` for
tasks, and one `Mutex` with one `Condvar` for idle workers to sleep on.
Each piece is correct and familiar.

Measured on an i7-12700K (20 hardware threads, 19 workers), it cost:

- **434 ns per task handed over,** on the producer: taking the lock that
  19 workers poll, list work, and a wake decision that reads counters the
  workers keep writing. The producer is the bottleneck in these workloads
  (it reads or copies the data), so this cost sets the speed.
- **Futex bucket contention in the kernel:** with every worker sleeping
  and waking through one mutex and one condition variable, the kernel
  serialized them on one futex hash bucket's spinlock. Kernel profiles
  (resolved against `vmlinux`) put 6% of all samples in
  `native_queued_spin_lock_slowpath` under `futex_q_lock` and
  `futex_wake`, rising to 15% once a change elsewhere made workers sleep
  and wake more often. The cost of each sleep grew with the number of
  workers arriving together.

## The special-purpose design

Starting instead from what this workload is: few producers, each with its
own ring of tasks in order; coarse tasks (13-30 µs); many consumers with
spare capacity; and an expensive wake (15-45 µs before the woken thread
runs).

- **No shared queue.** Each stream's own ring is its queue, with two
  counters on their own cache lines. The producer publishes a task with
  one plain `Release` store; workers claim tasks by compare-and-swap.
  Neither writes the other's counter. Both only count up, so no stale
  claim is possible. A hand-over drops from 434 ns to a few nanoseconds.
- **No shared sleep.** Each worker parks on its own thread's token
  (`std::thread::park`), so no two share a lock or a futex word; a bit
  mask records who is parked, and a waker claims one bit and unparks that
  worker alone.
- **The bottleneck pays at most one wake.** The producer wakes one worker
  when two tasks wait; a woken worker that still sees work wakes the next.
- **No sleep on a momentary gap.** While a stream is open, a worker that
  finds nothing looks again for 5 µs before parking.

None of this is general-purpose. It assumes in-order producers that keep
their own rings, and coarse tasks. That is this crate's case exactly.

## The result

Against the same stream on the conventional pool, 8 fresh-layout pairs of
bench-hashes' cells: **no cell slower**; messages 5-49% faster (16 KiB
messages from two threads at once, 49%), batches 3-15% faster. One-shot
calls on the same pool, which kept their own scheduling and only gained the
new parking, stayed level. Context switches per benchmark run fell from
about 28,500 to about 10,000.

The spin window shows the trade the special-purpose tool makes explicit
instead of hiding. Against the stream's earlier design, with no spin small
messages ran 9-12% slower (a park and a wake on every gap), where 5 µs of
spin left them 4% slower or level; 20 µs used about 22% more CPU time per
run than 5 µs, for no further speed. 5 µs was the measured best.

## How to apply it

The general tool was not wrong; it was general. The useful habit is to
measure where the time goes, name the workload's actual shape, and ask
which parts of the general tool that shape needs. In this case, none of
the lock, the shared queue or the shared sleep were needed, and each cost
the bottleneck thread time.
