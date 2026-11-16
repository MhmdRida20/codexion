# Codexion — To Research (recap level)

You already know mutexes and the OS-course basics, so this is a checklist, not a
course. For each topic: what to recall, why Codexion needs it, and one self-check
question. If you can answer the question, skip the topic.

Tags: **RECAP** = you have seen it, refresh only. **NEW** = probably not covered in
depth in an OS course, spend the time here.

Total: about 3 hours. Topics 3, 7 and 8 are where the project is won or lost.

**Skip entirely: semaphores.** No `sem_*` function is allowed in this project.

---

## 1. Threads with pthreads — RECAP, 10 min

- `pthread_create(&tid, NULL, routine, arg)`: routine is `void *f(void *)`, one pointer argument → pass a struct.
- Threads share the heap and everything reachable by pointer; each has its own stack.
- `pthread_join` waits for a thread and is the only way to end one cleanly here (no `pthread_detach`, no `exit` allowed).

Codexion: one thread per coder + one monitor thread; all must be joined.

Self-check: what happens to the other threads if `main` returns while they still run?

## 2. Race conditions and mutexes — RECAP, 10 min

- Data race: two threads touch the same memory, at least one writes, no synchronization.
- Critical section; a mutex makes it mutually exclusive.
- Reading a shared variable needs the lock too, not only writing.
- Keep critical sections short; never sleep while holding a lock.

Codexion: dongle state, `last_compile_start`, compile counters, stop flag, stdout.

Self-check: the monitor only *reads* `last_compile_start`. Why does it still need the mutex?

## 3. Condition variables — NEW, 45 min — the most important topic

- A cond var lets a thread sleep until some **condition on shared state** may have changed.
- `pthread_cond_wait(&cond, &mutex)`: atomically unlocks the mutex and sleeps; on wake-up it re-locks the mutex before returning.
- Always `while (!condition) pthread_cond_wait(...)`, never `if`: spurious wake-ups exist, and another thread may take the resource between the signal and your wake-up.
- Lost wake-up: a signal sent when nobody waits is forgotten. That is why the condition lives in a variable checked under the mutex, and why the state is changed under the same mutex before signalling.
- `signal` wakes one waiter (which one is not guaranteed), `broadcast` wakes all. With a priority queue you usually broadcast and let each waiter check "am I the top?".
- `pthread_cond_timedwait(&cond, &mutex, &abstime)`: same, with a deadline. `abstime` is an **absolute** `struct timespec` on `CLOCK_REALTIME` (same clock as `gettimeofday`), not a duration. Returns `ETIMEDOUT` on timeout; you still re-check the condition.

Codexion: waiting for a dongle (queue), waiting out a cooldown (timedwait), waking everyone on stop (broadcast).

Self-check: a dongle is free but cooling for 30 more ms and nobody will signal. How does the waiting coder wake up at the right time?

## 4. Deadlock and Coffman's conditions — RECAP, 20 min

- Four conditions, all needed at once: mutual exclusion, hold and wait, no preemption, circular wait.
- Prevention = break one by design. Usual choices: global resource ordering (breaks circular wait) or take-all-or-nothing (breaks hold and wait).
- Dining philosophers is the textbook case; Codexion is that problem with extra rules.
- The README must name the conditions and say which one you break.

Self-check: 4 coders each take their left dongle at the same moment. Name the four conditions in that picture, then say which one your design removes.

## 5. Starvation, fairness, liveness — RECAP, 10 min

- Deadlock-free does not mean starvation-free: a thread can lose every race forever.
- Liveness = "something good eventually happens" for every thread.
- Barging: a newcomer takes a resource ahead of threads already waiting. A queue respected by everyone prevents it.
- Convoy effect: threads line up behind one another and run one at a time although more could run in parallel.

Codexion: required explicitly ("no coder should be starved under edf"), and the convoy at t=0 is a real risk.

Self-check: why does a plain mutex not guarantee FIFO order among the threads waiting for it?

## 6. Scheduling policies: FIFO and EDF — NEW, 20 min

- FIFO: serve in arrival order. Needs a strict order → a sequence counter, since two arrivals can share the same millisecond.
- EDF (Earliest Deadline First): serve the request whose deadline is closest. Here deadline = `last_compile_start + time_to_burnout`. Classic real-time policy.
- Tie-breaker: needed so equal deadlines always resolve the same way (deterministic).
- Feasibility: no scheduler can save parameters that are impossible. Work out for yourself why a coder's cycle is roughly `max(compile + debug + refactor, k × (compile + cooldown))` with k = 2 for an even table and 3 for an odd one — you will use it to predict test results.

Self-check: 3 coders, compile 200, cooldown 0. Why can only one compile at a time, and what is the smallest `time_to_burnout` that can work?

## 7. Binary heap / priority queue — RECAP, 30 min

- Min-heap stored in an array: parent `(i - 1) / 2`, children `2i + 1` and `2i + 2`.
- Push: append at the end, sift up. Pop: take the root, move the last element to the root, sift down. Both O(log n). Peek is O(1).
- The policy lives in one compare function → same heap code for fifo and edf.
- A heap is not stable: equal keys come out in no particular order unless the compare function has a second criterion.

Codexion: mandatory, hand-written, and a likely recode target ("change the priority rule").

Self-check: insert keys 5, 3, 8, 1 into an empty min-heap and draw the array after each step; then pop once.

## 8. Time handling in C — NEW, 20 min

- `gettimeofday(&tv, NULL)` → seconds + microseconds; ms = `tv_sec * 1000 + tv_usec / 1000`. Store in `long`, not `int`.
- Log timestamps are relative: `now - start_time`.
- `usleep(n)` sleeps **at least** n microseconds, often more. Precise sleeping = loop of short `usleep`s that checks the clock (and the stop flag) until the target time.
- `struct timespec` (seconds + **nano**seconds) is what `pthread_cond_timedwait` wants; converting from ms means handling the nanosecond overflow into seconds.

Codexion: the 10 ms burnout rule and every tight test depend on this.

Self-check: convert "now + 250 ms" into a `struct timespec` on paper, starting from a `struct timeval`.

## 9. Monitor thread and stop flag — NEW, 15 min

- One watcher thread polls shared state on a short period and raises a flag; workers check the flag at every step.
- The flag is shared data → mutex. Setting it is not enough: threads blocked on a cond var must be woken with `broadcast`.
- A flag + mutex + cond var together is what the subject calls a "custom event".
- Check-then-act races: "check stop, then print" must happen under one lock or a line can slip out after the stop.

Self-check: the monitor sets stop while a coder is inside `pthread_cond_wait`. What exactly must happen for that coder's thread to reach its `return`?

## 10. Debugging tools — NEW, 20 min

- `valgrind --leak-check=full --show-leak-kinds=all ./codexion …` → leaks and invalid memory use.
- `valgrind --tool=helgrind ./codexion …` (or `--tool=drd`) → data races and inconsistent lock order.
- `cc -fsanitize=thread -g` → ThreadSanitizer, faster than helgrind. Do not combine with valgrind.
- All of them slow the program a lot: timings stretch, coders may burn out under the tool with parameters that pass normally. Use generous parameters when checking for races.

Self-check: helgrind reports "lock order violated". What bug is it warning you about, even if the program has never frozen?

## 11. Norm rules that shape the design — RECAP, 5 min

- 25 lines per function, 4 parameters max, 5 variables max, 5 functions per file, 80 columns.
- No `for`, `do…while`, `switch`, ternary, `goto`; declarations at the top; no declaration + assignment on one line.
- Consequence: pass one context pointer instead of many parameters, and split early — wait loops and init/cleanup functions get long quickly.

---

## Where to look

- Man pages first: `man 3 pthread_cond_wait`, `man 3 pthread_cond_timedwait`, `man 3 pthread_mutex_lock`, `man 2 gettimeofday`, `man 3 usleep`.
- *Operating Systems: Three Easy Pieces* (free online, https://pages.cs.wisc.edu/~remzi/OSTEP/): ch. 27 Thread API, ch. 30 Condition Variables, ch. 31 for dining philosophers, ch. 32 Common Concurrency Problems (deadlock). Chapter 30 alone covers topic 3.
- Wikipedia: "Deadlock (computer science)" for Coffman's conditions, "Earliest deadline first scheduling", "Binary heap".
- Valgrind manual, Helgrind chapter, for reading its reports.

Keep a list of what you actually read: the README "Resources" section needs it.

## Questions to bring back

Write here anything that stayed unclear after the recap; we go through them before the design session.

-
