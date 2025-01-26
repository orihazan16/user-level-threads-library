# uthreads: A User-Level Threads Library in C++

A small preemptive user-level threading library built from scratch on Linux, with no `pthread` and no kernel threads. Scheduling, context switching, and thread lifecycle management all live in user space.

Built as part of the Operating Systems course at the Hebrew University of Jerusalem.

## Highlights

- **Preemptive round-robin scheduler** driven by a virtual interval timer (`setitimer(ITIMER_VIRTUAL)`) and the `SIGVTALRM` signal.
- **Manual context switching** with `sigsetjmp` / `siglongjmp`. For new threads, the stack pointer and program counter are written directly into the `jmp_buf`, including the address translation that glibc's pointer mangling requires on both x86 and x86-64.
- **Per-thread stacks** (4 KB each) and up to 100 concurrent threads, main thread included.
- **Full thread lifecycle**: `spawn`, `terminate`, `block`, `resume`, and `sleep`, with the READY / RUNNING / BLOCKED / SLEEPING state machine handled correctly.
- **Race-free critical sections**: scheduler data structures are protected by masking the timer signal (`sigprocmask`) while they are modified.
- **Deferred deletion**: a thread that terminates itself is freed only after the switch to the next thread, so the library never frees the stack it is running on.
- **Smallest-free-ID allocation**: thread IDs are recycled through a min-heap, so IDs stay compact and predictable.
- **Defined error handling**: library errors and system errors are reported to `stderr` in a consistent format, and calls return `-1` on failure.

## API

```cpp
int uthread_init(int quantum_usecs);        // start the library; the caller becomes thread 0
int uthread_spawn(thread_entry_point f);    // create a thread, returns its tid
int uthread_terminate(int tid);             // terminate a thread (tid 0 ends the process)
int uthread_block(int tid);                 // move a thread to BLOCKED
int uthread_resume(int tid);                // move a BLOCKED thread back to READY
int uthread_sleep(int num_quantums);        // sleep the running thread for N quantums
int uthread_get_tid();                      // id of the calling thread
int uthread_get_total_quantums();           // quantums since init
int uthread_get_quantums(int tid);          // quantums a given thread has run
```

The full contract for each function is documented in [`uthreads.h`](uthreads.h).

## Design Overview

| Concern | Approach |
|---|---|
| Ready queue | `std::deque<Thread*>` (FIFO, round robin) |
| Thread registry | `std::map<int, Thread*>` keyed by tid |
| Blocked threads | `std::set<Thread*>` |
| Sleeping threads | `std::map` of tid to (thread, remaining quantums), decremented at each quantum start |
| ID allocation | `std::priority_queue` (min-heap) of free IDs |
| Timer | One-shot `ITIMER_VIRTUAL`, re-armed at every context switch |
| Context switch | `sigsetjmp` saves state, `siglongjmp` restores the next thread |

A thread can be both sleeping and blocked at the same time. When its sleep ends, it wakes into `BLOCKED` rather than `READY`, and only returns to the ready queue after an explicit `uthread_resume`.

## Build

The library targets Linux (x86 / x86-64) and needs a C++11 compiler.

```bash
g++ -std=c++11 -Wall -Wextra -c uthreads.cpp -o uthreads.o
ar rcs libuthreads.a uthreads.o
```

## Usage Example

```cpp
#include "uthreads.h"
#include <iostream>

void worker() {
    for (;;) {
        std::cout << "thread " << uthread_get_tid()
                  << ", quantum " << uthread_get_quantums(uthread_get_tid()) << "\n";
    }
}

int main() {
    uthread_init(100000);          // 100 ms quantum
    uthread_spawn(worker);
    uthread_spawn(worker);

    while (uthread_get_total_quantums() < 10) { /* main thread keeps running */ }

    uthread_terminate(0);          // clean up and exit
}
```

Link against the library:

```bash
g++ -std=c++11 main.cpp -L. -luthreads -o demo
```

## Skills Demonstrated

- Systems programming in C/C++
- Signals, timers, and asynchronous control flow
- Context switching and low-level stack / register manipulation
- Concurrency design: scheduling, state machines, critical sections
- Manual memory management and resource cleanup

## Repository Layout

```
uthreads.h     public API and documentation
uthreads.cpp   library implementation
```

