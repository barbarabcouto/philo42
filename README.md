*This project has been created as part of the 42 curriculum by bdavid-b*

# Philosophers

## Description

**Philosophers** is a 42 project about the basics of multithreading and the use
of mutexes to protect shared resources. It is a classic concurrency exercise
known as the **Dining Philosophers Problem**.

A number of philosophers sit around a round table with a bowl of spaghetti in
the middle. Each philosopher alternates between three states — **eating**,
**sleeping** and **thinking**. Between every pair of philosophers lies a single
fork, and a philosopher needs **both** the fork on their left and the one on
their right to eat. Philosophers do not talk to each other and do not know when
a neighbour is about to die, so the simulation must coordinate access to the
forks without any communication between them.

The goal is to keep every philosopher alive: if a philosopher does not start
eating within `time_to_die` milliseconds of their last meal (or of the start of
the simulation), they die and the simulation stops. The program must therefore
avoid **deadlocks**, **data races** and **starvation** while keeping the death
detection accurate to within 10 ms.

In this mandatory part:
- Each philosopher is a separate **thread**.
- Each fork is protected by a **mutex**.
- Global variables are forbidden.

## Instructions

### Requirements
- A C compiler (`cc` / `gcc`)
- `make`
- A POSIX system with the pthread library

### Compilation

```bash
make        # builds the philo binary
make clean  # removes object files
make fclean # removes object files and the binary
make re     # full rebuild
```

The project compiles with the `-Wall -Wextra -Werror` flags.

### Execution

```bash
./philo number_of_philosophers time_to_die time_to_eat time_to_sleep [number_of_times_each_philosopher_must_eat]
```

| Argument | Meaning |
|---|---|
| `number_of_philosophers` | Number of philosophers (and forks) |
| `time_to_die` (ms) | If a philosopher does not start eating within this time since their last meal, they die |
| `time_to_eat` (ms) | Time it takes for a philosopher to eat (holding two forks) |
| `time_to_sleep` (ms) | Time a philosopher spends sleeping |
| `number_of_times_each_philosopher_must_eat` | *(optional)* The simulation stops once every philosopher has eaten at least this many times |

### Usage examples

```bash
./philo 5 800 200 200       # 5 philosophers, nobody should ever die
./philo 5 800 200 200 7     # stops once each philosopher has eaten 7 times
./philo 1 800 200 200       # a single philosopher always dies (only one fork)
./philo 4 410 200 200       # tight timing, nobody should die
./philo 4 310 200 100       # a philosopher should die
```

### Output

Every state change is logged on its own line as:

```
timestamp_in_ms X has taken a fork
timestamp_in_ms X is eating
timestamp_in_ms X is sleeping
timestamp_in_ms X is thinking
timestamp_in_ms X died
```

where `X` is the philosopher number. A death message is printed within 10 ms of
the actual death, and log lines never overlap.

## Technical choices

- **Per-philosopher meal lock.** Instead of a single global mutex guarding all
  shared data, each philosopher owns a `meal_lock` mutex protecting their last
  meal time and meal count. This finer-grained locking reduces contention
  between the routine threads and the monitor.
- **Deadlock avoidance by fork ordering + offset.** Even-numbered philosophers
  pick up their right fork first while odd-numbered philosophers pick up their
  left fork first. In addition, even-ID philosophers are given a small
  `time_to_eat / 2` head start so neighbours do not all reach for the same forks
  at once. Together these break the circular-wait condition that causes
  deadlock.
- **Monitor on the main thread.** Rather than spawning a dedicated monitoring
  thread, the death/`must_eat` checks run on the main thread after the
  philosopher threads are launched, which keeps the design simple and avoids an
  extra thread synchronising on the same data.
- **Precise sleeping.** `precise_sleep` loops with short `usleep(300)` calls
  while checking both the elapsed time and the simulation's stop flag, so
  philosophers wake up immediately when the simulation ends instead of
  oversleeping past a death event.

### Source layout

```
philo/
├── Makefile
├── philo.h
├── main.c       # entry point
├── parse.c      # argument validation and parsing
├── init.c       # mutex and philosopher initialisation
├── routine.c    # the per-philosopher eat / sleep / think loop
├── monitor.c    # death and must_eat detection
├── utils.c      # timing helpers, safe printing
└── cleanup.c    # mutex destruction and freeing
```

## Resources

Classic references on threading, mutexes and the Dining Philosophers Problem:

- **MIT 6.005 — Reading 17: Concurrency** — foundational material on shared
  memory vs. message passing, race conditions and the non-atomicity of seemingly
  simple operations.
  https://web.mit.edu/6.005/www/fa14/classes/17-concurrency/
- **The dining Philosophers in C: threads, race conditions and deadlocks
  (YouTube)** — a practical walkthrough of the problem in C.
  https://www.youtube.com/watch?v=zOpzGHwJ3MU
- **Philosophers 42 Guide: The Dining Philosophers Problem — Dean Ruina
  (Medium)** — a 42-oriented guide used to compare design trade-offs (stack vs.
  heap allocation, single global lock vs. per-philosopher lock, monitor thread
  placement).
  https://medium.com/@ruinadd/philosophers-42-guide-the-dining-philosophers-problem-893a24bc0fe2
- **Linux `man` pages** — `pthread_create(3)`, `pthread_join(3)`,
  `pthread_detach(3)`, `pthread_mutex_init(3)`, `pthread_mutex_lock(3)`,
  `gettimeofday(2)`, `usleep(3)`.

### Use of AI

AI (Claude, by Anthropic) was used as a **learning and review tool**, not as a
code generator I could not account for. Specifically:

- **Conceptual grounding.** Building an understanding of the
  pthread API (the `void *` argument pattern, `pthread_create` / `pthread_join`)
  and of core concurrency ideas — race conditions, the non-atomicity of simple
  operations, memory reordering, time slicing and non-determinism — through
  small standalone demos and analogies before applying them here.
- **Review and debugging.** Reading through the implementation to find data
  races and timing bugs and to reason about deadlock avoidance.
