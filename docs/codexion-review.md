# Codexion — Full Subject Review (subject v1.5)

Purpose: one place that lists **everything** the subject asks for, plus the things it
implies but never says out loud. Every requirement has an ID (B/A/S/L/D/E) that the
to-do list (`codexion-todo.md`) points back to, so nothing can be silently skipped.

Legend: **(subject)** = written in the PDF. **(derived)** = my conclusion from the PDF;
logical, but not a quote. **(decision)** = the subject is silent, we must choose and
document the choice.

---

## 0. The project in five lines

- N coders sit in a circle, N dongles, one dongle between each pair of neighbours.
- A coder loops: take 2 dongles → **compile** → release both → **debug** → **refactor** → repeat.
- A released dongle is unusable for `dongle_cooldown` ms.
- When two coders want the same dongle, the dongle decides who gets it: `fifo` or `edf`,
  using a **priority queue (heap) you write yourself**.
- A coder who does not *start* a compile within `time_to_burnout` ms of the *start* of
  their previous compile (or of the simulation) burns out; a **monitor thread** must
  report it within 10 ms and the simulation ends.

It is Dining Philosophers with three additions: cooldown, an explicit scheduler with a
heap, and condition variables instead of "a fork is a mutex".

**There is no bonus part in v1.5** — the PDF goes Mandatory → Readme → Submission.
Everything below is mandatory.

---

## 1. Build and submission constraints

| ID | Requirement | Source |
|----|-------------|--------|
| B1 | Written in C | subject |
| B2 | Norm-compliant (every file; one norm error = 0) | subject |
| B3 | Never crashes (segfault, bus error, double free…) — a crash = 0 | subject |
| B4 | All heap memory freed, no leaks | subject (stated twice) |
| B5 | Makefile with `$(NAME)`, `all`, `clean`, `fclean`, `re`; compiler `cc`; flags `-Wall -Wextra -Werror -pthread`; **no unnecessary relink** | subject |
| B6 | Binary name `codexion`; Makefile at repo root; `.c`/`.h` in any folder | subject |
| B7 | **No libft** | subject |
| B8 | Only the allowed external functions (list below) | subject |
| B9 | **No global variables** | subject |
| B10 | `README.md` at repo root (details in section 6) | subject |
| B11 | Only what is in the Git repo is graded | subject |

### Allowed functions (B8)

```
pthread_create  pthread_join
pthread_mutex_init  pthread_mutex_lock  pthread_mutex_unlock  pthread_mutex_destroy
pthread_cond_init  pthread_cond_wait  pthread_cond_timedwait
pthread_cond_signal  pthread_cond_broadcast  pthread_cond_destroy
gettimeofday  clock_gettime  usleep
write  malloc  free  printf  fprintf
strcmp  strlen  atoi  memset
```

What is **missing** from that list matters as much as what is in it — see section 8.

---

## 2. Arguments

```
./codexion number_of_coders time_to_burnout time_to_compile time_to_debug \
           time_to_refactor number_of_compiles_required dongle_cooldown scheduler
```

| ID | Requirement | Source |
|----|-------------|--------|
| A1 | Exactly 8 arguments, **all mandatory**, in this order | subject |
| A2 | Reject negative numbers, non-integers, and any scheduler that is not exactly `fifo` or `edf` | subject |
| A3 | Semantics of each argument (table below) | subject |

| # | Argument | Meaning |
|---|----------|---------|
| 1 | `number_of_coders` | number of coders = number of dongles |
| 2 | `time_to_burnout` (ms) | max time between two compile *starts* (or sim start → first compile start) |
| 3 | `time_to_compile` (ms) | compile duration; both dongles held the whole time |
| 4 | `time_to_debug` (ms) | debug duration |
| 5 | `time_to_refactor` (ms) | refactor duration; afterwards the coder **immediately** tries to get dongles |
| 6 | `number_of_compiles_required` | when **every** coder has compiled at least this many times, stop |
| 7 | `dongle_cooldown` (ms) | after release, dongle is unavailable for this long |
| 8 | `scheduler` | `fifo` or `edf` |

Note vs Philosophers: the "number of meals" argument is **not optional** here, and it
sits in position 6, before the cooldown.

---

## 3. Simulation rules

| ID | Requirement | Source |
|----|-------------|--------|
| S1 | Each coder is a thread created with `pthread_create` | subject |
| S2 | Coders numbered 1..N; coder 1 sits next to coder N; coder k between k-1 and k+1 | subject |
| S3 | N dongles, one between each pair of coders; **1 coder → exactly 1 dongle** | subject |
| S4 | Compiling needs the left **and** right dongle, held for the whole `time_to_compile` | subject |
| S5 | Cycle: compile → put both dongles back → debug → refactor → immediately request dongles again. States are mutually exclusive | subject |
| S6 | Burnout: no compile *start* within `time_to_burnout` ms since the last compile *start* / sim start | subject |
| S7 | Each dongle's **state** is protected by a `pthread_mutex_t`; a `pthread_cond_t` *may* manage the waiting queue | subject |
| S8 | Cooldown: a released dongle cannot be taken until `dongle_cooldown` ms have elapsed | subject ("mandatory") |
| S9 | Fair arbitration: on contention the dongle grants access per `scheduler` — `fifo` = arrival order, `edf` = smallest `last_compile_start + time_to_burnout` | subject ("mandatory") |
| S10 | EDF must be fully deterministic → a **tie-breaker** for equal deadlines is required | subject (rule itself not given → Q1) |
| S11 | A **priority queue (heap)** written by you drives FIFO/EDF | subject |
| S12 | Liveness: under `edf`, with feasible parameters, nobody starves and burns out | subject |
| S13 | A **separate monitor thread** detects burnout precisely and stops the simulation | subject |
| S14 | Simulation stops on first burnout **or** when all coders reached `number_of_compiles_required` | subject |
| S15 | Coders do not communicate with each other and do not know whether another coder is about to burn out | subject |

Consequence of S15 (derived): a coder thread must never read another coder's data to
make a decision. The **dongle** (its queue) arbitrates, using the deadline each coder
hands over with its own request.

---

## 4. Logs

| ID | Requirement | Source |
|----|-------------|--------|
| L1 | Exactly these five lines, nothing else: `<ts> X has taken a dongle`, `<ts> X is compiling`, `<ts> X is debugging`, `<ts> X is refactoring`, `<ts> X burned out` | subject |
| L2 | `<ts>` = timestamp in ms (examples start at 0 → **relative to simulation start**), `X` = coder number | subject + derived |
| L3 | Messages never interleave → a mutex protects output | subject |
| L4 | `burned out` printed **≤ 10 ms** after the real burnout moment | subject (stated 3 times) |
| L5 | Every `is compiling` is preceded by **two** `has taken a dongle` lines from the same coder | subject (example note) |

Derived (a standard check in Philosophers-style evaluations): `burned out` is the
**last** line. Nothing may be printed after it, and nothing after the "all compiled
enough" stop either.

---

## 5. What the mandatory components are (checklist form)

1. Thread per coder (S1)
2. Mutex per dongle protecting its state (S7)
3. Cooldown per dongle (S8)
4. Scheduler: fifo + edf, with tie-breaker (S9, S10)
5. Hand-written heap used by the scheduler (S11)
6. Monitor thread (S13)
7. Log mutex (L3)
8. Two stop conditions (S14)
9. Deadlock-free and starvation-free design (S12 + README section D5)

---

## 6. README.md requirements

| ID | Requirement |
|----|-------------|
| D1 | **Very first line**, italic: `*This project has been created as part of the 42 curriculum by <login>.*` |
| D2 | "Description" section — goal + brief overview |
| D3 | "Instructions" section — compile / install / run |
| D4 | "Resources" section — classic references **and** how AI was used (which tasks, which parts) |
| D5 | "Blocking cases handled" section — deadlock prevention + Coffman's conditions, starvation prevention, cooldown handling, precise burnout detection, log serialization |
| D6 | "Thread synchronization mechanisms" section — which primitives (`pthread_mutex_t`, `pthread_cond_t`, any custom event), how they guard dongles / logging / monitor state, **examples** of races prevented, and how coders and monitor communicate safely |
| D7 | Written in English |

D4 is real: this project is being built with AI guidance, so the README must say so
honestly (e.g. "requirement review, planning, concept explanations, code review").

---

## 7. Evaluation

| ID | Requirement |
|----|-------------|
| E1 | You must be able to explain every line. The subject's AI chapter is explicit: code you cannot explain = failed project |
| E2 | **Recode step possible**: a small live modification in a few minutes (change a behaviour, a display, add a field to a struct…) |

E2 is why the code must stay simple and modular: e.g. "add a third scheduler", "print
the compile count", "change the tie-breaker" should each be a 5-minute edit.

---

## 8. Traps the subject does not spell out (all derived)

These are the items most likely to cost time or a failed evaluation.

**From the allowed-function list**

- T1. **`exit` is not allowed.** Every error path must unwind and `return` from `main`.
- T2. **`pthread_detach` is not allowed.** Every created thread must be `pthread_join`ed,
  so every thread must be able to terminate — including one blocked waiting for a dongle.
- T3. **No `strtol`, no `isdigit`.** `atoi` alone accepts `"12abc"` and overflows
  silently, so A2 needs a small strict parser of your own (digits only, overflow check).
- T4. **No `pthread_condattr_setclock`.** `pthread_cond_timedwait` therefore runs on the
  default clock (`CLOCK_REALTIME`) and takes an **absolute** time. Build that time from
  `gettimeofday` (same clock). Mixing in `CLOCK_MONOTONIC` here is a classic bug.
- T5. **No `fflush` / `setbuf`.** `printf` is line-buffered on a terminal but fully
  buffered when piped to a file. Timestamps stay correct either way; see Q8.
- T6. **No `calloc`** → `malloc` + `memset`. **No semaphores at all** (`sem_*` absent).

**From the rules**

- T7. **A dongle is not "a mutex you hold while compiling".** Because of cooldown and
  the queue, a dongle is a small struct (in-use flag, `available_at` time, waiting heap)
  whose *state* is guarded by a mutex held only for a few instructions.
- T8. **Nobody signals the end of a cooldown.** A coder waiting on a dongle that is free
  but still cooling must wake up by itself → this is what `pthread_cond_timedwait` is for.
- T9. **No barging.** If the dongle becomes available while someone is already queued, a
  newly arriving coder must not grab it ahead of the queue. Only the heap's top may take it.
- T10. **A heap is not stable.** FIFO (and the EDF tie-break) need a monotonically
  increasing sequence number as key, not just a millisecond timestamp, or two requests
  in the same ms have no defined order.
- T11. **Deadlock.** If every coder takes its left dongle and waits for the right one,
  all four Coffman conditions hold. One of them must be broken by design (and named in
  the README).
- T12. **Stop must wake everyone.** On burnout, coders blocked in `pthread_cond_wait`
  must be woken (`broadcast`) and coders sleeping must notice the stop flag, otherwise
  `pthread_join` hangs or the program exits late.
- T13. **`usleep` overshoots.** A plain `usleep(time_to_compile * 1000)` drifts by
  several ms per cycle; with tight parameters coders burn out who should not. Needs a
  precise sleep (short sleeps in a loop against the clock, checking the stop flag).
- T14. **Shared data between coder and monitor.** `last_compile_start` and the compile
  counter are written by the coder and read by the monitor (and by the scheduler for
  EDF) → mutex-protected, always.
- T15. **Deadline race.** "Coder starts compiling" and "monitor declares burnout" must be
  mutually exclusive, and a coder must re-check the stop flag before logging, or you get
  an `is compiling` line after `burned out`.
- T16. **Common start time.** Threads are created one after another; `start_time` and
  every coder's initial `last_compile_start` must be the same instant, otherwise the last
  coders start with an unfair handicap.
- T17. **Special shapes.** 1 coder: left dongle == right dongle, can never compile, must
  burn out cleanly without hanging. 2 coders: both share the *same two* dongles.
- T18. **Partial-failure cleanup.** If `malloc` or the k-th `pthread_create` fails, what
  was already created must be joined / destroyed / freed (B3, B4).
- T19. **Your own lock order.** Besides the dongle deadlock, nesting your own mutexes
  inconsistently (e.g. log mutex inside dongle mutex in one place, the reverse in
  another) creates a second deadlock source. Fix one global order on paper first.
- T20. **Integer width.** Timestamps in ms since the epoch do not fit in `int` → `long`.
- T21. **Convoy at start.** At t=0 every coder requests at once. If each one grabs a
  first dongle and queues for the second, they form a chain where only **one** coder
  compiles at a time instead of ⌊N/2⌋. No deadlock, but with tight parameters someone
  burns out who should not. The design needs a start pattern that lets non-neighbours
  compile in parallel (e.g. a small staggered start, or taking both dongles or none).
  I traced `4 410 200 100 100` by hand: with a naive "lower dongle first" order and
  an unlucky interleaving, coders 3, 2, 1 compile one after another and coder 4 burns
  out at 410 ms, while clean alternation of the two pairs survives.

---

## 9. Open questions and decisions

The subject is silent or ambiguous here. Each needs a choice, written in the README.
My recommendation is given; confirm against what your campus evaluators expect.

| # | Question | Recommendation |
|---|----------|----------------|
| Q1 | **EDF tie-breaker** — the subject says it is "required" but never defines it | Equal deadline → earlier request (sequence number) → lower coder number. Deterministic and easy to defend |
| Q2 | **Zero / limit values** — is 0 valid? | `number_of_coders` ≥ 1 (subject: "one or more coders"); times and cooldown ≥ 0; values above `INT_MAX` rejected. `number_of_compiles_required == 0` → either reject or stop immediately with no output; pick one, be consistent |
| Q3 | **When does a compile count** — at its start or its end? | At the end ("have compiled"). Slightly more literal than counting at start |
| Q4 | **Which clock** | `gettimeofday` everywhere — subject calls it "acceptable and recommended", and it matches `pthread_cond_timedwait` (T4) |
| Q5 | **Architecture** — one queue per dongle, or one central queue for all requests? | Lean: **per-dongle queue + mutex + cond**, because it maps word-for-word onto S7/S9 ("the dongle must grant access") and keeps neighbours that do not compete fully parallel. The public student repos I found mostly use one central queue, so that route is clearly accepted too; to be settled in the design step |
| Q6 | **Which Coffman condition to break** | Per-dongle design → global acquisition order (always lower-numbered dongle first) breaks *circular wait*. Central-queue design → take both or none breaks *hold and wait* |
| Q7 | **1 coder** | Logs `0 1 has taken a dongle`, then `<time_to_burnout> 1 burned out` (the Philosophers convention) |
| Q8 | **`printf` or `write` for logs** | `printf` under the log mutex is simplest and is what the format implies; switch to `write` only if a tester needs unbuffered piped output |
| Q9 | **Initial dongle state** | All dongles available at t=0, no cooldown before first use |
| Q10 | **Burnout comparison** | `now - last_compile_start >= time_to_burnout`; either `>` or `>=` is inside the 10 ms tolerance — just be consistent |

One external unknown: the **evaluation sheet**. It defines the exact test cases and the
recode task. Ask a peer at 42 Beirut who has already been evaluated on Codexion.

---

## 10. Codexion vs Philosophers (if you have seen it)

| Philosophers | Codexion |
|--------------|----------|
| fork = mutex, held while eating | dongle = struct with state, mutex held briefly (T7) |
| whoever locks first wins | explicit scheduler: fifo / edf with a heap |
| no cooldown | cooldown per dongle (T8) |
| mutexes only | condition variables + timed waits |
| death measured from last meal start | burnout measured from last compile start (same idea) |
| meals argument optional | all 8 arguments mandatory |
| eat → sleep → think | compile → debug → refactor |

---

## 11. Traceability — every requirement has a home in the to-do list

| Requirement | To-do phase |
|-------------|-------------|
| B1, B2, B6, B7, B9 | P2 (skeleton), re-verified in P7 |
| B3, B4 | P2 (cleanup paths), P7 (valgrind) |
| B5 | P2 (Makefile), P7 (relink test) |
| B8 | P7 (forbidden-function check) |
| B10, D1–D7 | P8 |
| B11 | P0 (repo), P8 (final push) |
| A1–A3 | P2 (parsing), P7 (argument tests) |
| S1, S2, S3, S5 | P5 |
| S4, S7, S8, S9, S10 | P4 |
| S11 | P3 |
| S6, S13, S14 | P6 |
| S12 | P1 (design), P7 (stress tests) |
| S15 | P1 (design rule), P5 |
| L1, L2, L3, L5 | P2 (logger), P5 |
| L4 | P6, P7 |
| E1, E2 | every phase's "explain" checkpoint, P8 |
| T1–T21, Q1–Q10 | P1 decides, later phases implement (each to-do item names its T/Q) |
