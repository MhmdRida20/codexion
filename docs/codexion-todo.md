# Codexion — To-Do / Progress Tracker

How to use: tick `[x]` when an item is done **and** you can explain it. Codes in
brackets point to `codexion-review.md` (B/A/S/L/D/E = requirements, T = traps,
Q = open decisions). Time estimates are rough guesses for planning, not promises.

Rule for every phase: it is finished only when its **Explain checkpoint** passes —
you say the answer out loud (or to me) without looking at the code. That is the
defense, rehearsed early (E1).

Estimated total: **~18–26 hours** of focused work.

---

## P0 — Preparation (~2–3 h, most of it the research recap)

- [ ] Read the subject PDF once yourself, start to finish (30 min; chapters IV–VII are the core)
- [ ] Go through `codexion-research.md` — recap level only, stop when the self-check questions feel easy
- [ ] Ask a peer who was already evaluated on Codexion: what does the eval sheet test, and what was the recode task? (E2)
- [ ] Confirm your exact 42 login for the README first line (D1)
- [ ] Create the repo, add a `.gitignore` (objects, binary), first commit (B11)
- [ ] Check tools on your machine: `cc`, `make`, `norminette`, `valgrind` (helgrind comes with it)

**Explain checkpoint:** describe the full life of one coder and all the ways the
simulation can end, in under a minute.

---

## P1 — Design on paper (~1.5–2 h) — no code yet

The cheapest hour of the project: every decision made here saves a rewrite later.

- [ ] Decide the open questions Q1–Q10 and write each answer in the "Decisions" table at the bottom
- [ ] Choose the architecture: per-dongle queue vs central queue (Q5)
- [ ] Choose the deadlock-prevention rule and name the Coffman condition it breaks (Q6, T11)
- [ ] Draw the structs: simulation/config, coder, dongle, heap + request node
- [ ] Write the **lock map**: for every shared variable, which mutex protects it (T14)
- [ ] Fix one global **lock order** and write it down (T19)
- [ ] Write the "acquire a dongle" algorithm in pseudocode: enqueue → wait while (not top OR in use OR cooling OR not stopped…) → take (T7, T8, T9)
- [ ] Decide the start pattern so that non-neighbours compile in parallel from t=0; trace `4 410 200 100 100` by hand on paper to prove it (T21)
- [ ] Write the stop protocol in pseudocode: who sets the flag, who wakes whom, who joins whom (T2, T12)
- [ ] Plan the file split with the Norm in mind (max 5 functions per file, 25 lines per function, 4 parameters)
- [ ] Check the design against S15: no coder reads another coder's state

**Explain checkpoint:** why can your design never deadlock? Why can a coder never
starve under edf? Walk through 2 coders fighting for one dongle step by step.

---

## P2 — Skeleton and foundations (~2–3 h)

- [ ] Makefile: `NAME = codexion`, rules `all` `clean` `fclean` `re`, `cc`, `-Wall -Wextra -Werror -pthread`, objects depend on the header, no relink (B5, B6)
- [ ] Header with all structs and prototypes, include guards, no globals (B9)
- [ ] Strict integer parser: digits only, rejects empty / sign / letters / overflow (A2, T3)
- [ ] Argument validation: count == 8, ranges per Q2, scheduler via `strcmp` exactly `fifo` / `edf` (A1, A2)
- [ ] Error output on `stderr` + non-zero return, **no `exit`** (T1)
- [ ] Time helpers: `now_ms()` with `gettimeofday`, elapsed-since-start (Q4, T20)
- [ ] Precise, stop-aware sleep (T13, T12)
- [ ] Init: allocate coders + dongles, init every mutex / cond (S3, S7)
- [ ] Cleanup: destroy every mutex / cond, free everything; works after a **partial** init too (B4, T18)
- [ ] Logger: one function, log mutex, prints `<ts> <id> <msg>`, refuses to print once stopped (L1, L2, L3, T15)

**Explain checkpoint:** what happens, line by line, if the 3rd `pthread_mutex_init`
fails? Why can't you use `atoi` alone?

---

## P3 — Priority queue / heap (~1.5–2 h)

- [ ] Request node: coder id, priority key, sequence number (T10)
- [ ] Heap as an array: init (capacity), push (sift-up), pop (sift-down), peek, free (S11)
- [ ] One compare function: key first, then sequence number, then coder id (Q1, S10)
- [ ] Key selection by scheduler: fifo → arrival sequence; edf → `last_compile_start + time_to_burnout` (S9)
- [ ] Tiny standalone test `main` (not submitted): push in random order, pop, check order — for both policies and for equal keys
- [ ] Norm check on the heap files now, not at the end (B2)

**Explain checkpoint:** index of parent / children in the array; why push and pop are
O(log n); why a heap alone does not give FIFO order for equal keys.

---

## P4 — Dongle logic (~3–4 h) — the heart of the project

- [ ] Dongle struct: mutex, cond, in-use flag, `available_at`, waiting heap (S7, T7)
- [ ] `dongle_take`: lock → push request → wait loop → pop → mark in use → unlock (S9, T9)
- [ ] Wait loop uses `pthread_cond_timedwait` when the dongle is free but cooling, plain `pthread_cond_wait` otherwise; always re-checks the predicate (S8, T8, T4)
- [ ] `dongle_release`: lock → mark free → `available_at = now + cooldown` → broadcast → unlock (S8)
- [ ] The wait loop exits when the stop flag is set, and removes its own request cleanly (T12)
- [ ] `take_both`: acquisition in the order fixed in P1; log `has taken a dongle` after each one (S4, L5, T11)
- [ ] 1-coder case: takes the single dongle, waits for a second that never comes, leaves on stop without hanging (S3, T17, Q7)
- [ ] 2-coder case verified by hand: both dongles shared by the same pair (T17)

**Explain checkpoint:** why is the wait a `while` and not an `if`? Who wakes a coder
when a cooldown ends? What stops a newly arrived coder from jumping the queue?

---

## P5 — Coder routine (~2–3 h)

- [ ] Thread function: receives its own coder struct, loops until stop (S1, S5)
- [ ] Start gate: all coders begin from the same `start_time`; initial `last_compile_start = start_time` (T16)
- [ ] Compile step: set `last_compile_start` under its mutex, log `is compiling`, precise sleep `time_to_compile`, bump compile counter per Q3, release both dongles (S4, S6, T14, T15)
- [ ] Debug step: log `is debugging`, precise sleep
- [ ] Refactor step: log `is refactoring`, precise sleep, then straight back to requesting dongles (S5)
- [ ] Coder left/right dongle indexing correct for coder 1 and coder N (S2)
- [ ] Thread creation loop with failure handling; `pthread_join` on every created thread (T2, T18)

**Explain checkpoint:** which variables of a coder are touched by more than one
thread, and which mutex covers each? What does a coder do if stop is set while it
holds one dongle?

---

## P6 — Monitor and stop (~1.5–2 h)

- [ ] Monitor thread, separate from the coders (S13)
- [ ] Loop: for each coder, compare `now` to `last_compile_start + time_to_burnout` under the coder's mutex (S6, T14)
- [ ] On burnout: set stop flag and print `burned out` so that it is the last line and ≤ 10 ms late (L4, T15)
- [ ] Check "all coders reached `number_of_compiles_required`" → set stop, no message (S14)
- [ ] Polling period short enough for the 10 ms rule (about 1 ms), without burning a full CPU core
- [ ] Stop wakes every waiter: broadcast on every dongle cond (T12)
- [ ] `main`: join coders, join monitor, cleanup, return (T1, T2)

**Explain checkpoint:** prove the burnout line can never be more than 10 ms late.
Why can no `is compiling` appear after `burned out`?

---

## P7 — Testing and hardening (~3–5 h)

Feasibility rule of thumb used below (derive it yourself in the research file):
a coder's cycle is about `max(compile + debug + refactor, k × (compile + cooldown))`,
k = 2 for an even number of coders, 3 for odd. Burnout is expected when
`time_to_burnout` is below that. "Expected" results are predictions to verify.

**Arguments (A1, A2)** — all must print an error and return non-zero, no crash:
- [ ] too few / too many arguments
- [ ] `-5`, `abc`, `12abc`, `""`, `4.5`, `+`, `99999999999`
- [ ] scheduler `FIFO`, `fif`, `edff`, `random`
- [ ] `number_of_coders` = 0; your chosen behaviour for the other zeros (Q2)

**Behaviour**
- [ ] `./codexion 1 800 200 200 200 5 0 fifo` → one dongle taken, burns out at ~800, program ends
- [ ] `./codexion 2 800 200 200 200 5 0 fifo` → no burnout, ends after 5 compiles each
- [ ] `./codexion 4 800 200 200 200 5 0 edf` → no burnout
- [ ] `./codexion 5 800 200 200 200 5 0 edf` → no burnout (odd count)
- [ ] `./codexion 4 410 200 100 100 5 0 fifo` → tight: survives only if the two pairs alternate cleanly (T21, T13)
- [ ] `./codexion 4 310 200 100 100 5 0 fifo` → expected burnout at ~310
- [ ] `./codexion 4 800 200 200 200 5 100 fifo` → no burnout with cooldown
- [ ] `./codexion 4 500 200 100 100 5 100 edf` → expected burnout at ~500 (cooldown makes it infeasible)
- [ ] same feasible sets under both `fifo` and `edf`
- [ ] larger tables: 10, 50, 100+ coders with comfortable timings
- [ ] long run (several minutes) on a feasible set: nobody burns out (S12)

**Log correctness** (a small script or `awk`/`grep` over the output)
- [ ] every line matches one of the five formats exactly (L1)
- [ ] timestamps never go backwards (L2, L3)
- [ ] each `is compiling` has exactly two `has taken a dongle` by the same coder before it (L5)
- [ ] two neighbours never compile at overlapping times (S4)
- [ ] gap between a dongle's release and its next take ≥ `dongle_cooldown` (S8) — add a temporary debug print of the dongle id to check, then remove it
- [ ] burnout timestamp − (`last compile start` + `time_to_burnout`) ≤ 10 ms (L4)
- [ ] nothing printed after `burned out`, and nothing after the compile-count stop (T15)
- [ ] fifo really serves in arrival order; edf really serves the earliest deadline (S9) — temporary debug prints

**Tooling**
- [ ] `valgrind --leak-check=full --show-leak-kinds=all` on: normal end, burnout end, invalid args (B4)
- [ ] `valgrind --tool=helgrind` (and/or `--tool=drd`): zero data races, zero lock-order warnings (T14, T19)
- [ ] build once with `-fsanitize=thread -g` as a second opinion (not together with valgrind)
- [ ] `norminette` on every `.c` and `.h` (B2)
- [ ] `make` twice → second run does nothing; `touch` the header → rebuilds; `make re`, `clean`, `fclean` behave (B5)
- [ ] forbidden-function check: `nm -u codexion` shows only allowed functions (B8)
- [ ] no global variables: check with `nm codexion` for unexpected data symbols (B9)

**Explain checkpoint:** for one burnout test and one survival test, explain the
result with arithmetic before running it.

---

## P8 — README and defense prep (~2–3 h)

- [ ] First line, italic, exact sentence with your login (D1)
- [ ] Description (D2)
- [ ] Instructions: build, run, argument table, examples (D3)
- [ ] Resources: references you really used + honest AI-usage paragraph (D4)
- [ ] "Blocking cases handled": deadlock + Coffman, starvation, cooldown, burnout precision, log serialization (D5)
- [ ] "Thread synchronization mechanisms": each primitive, what it protects, race examples, coder ↔ monitor communication (D6)
- [ ] Document every decision Q1–Q10 (tie-breaker especially)
- [ ] English, spell-checked (D7)
- [ ] Recode drill, 5 minutes each, without help: change the tie-breaker; add a log line or change a log text; add a field to the coder struct and print it; add a trivial third scheduler (E2)
- [ ] Mock defense: explain the whole program top-down to a peer (E1)
- [ ] Final clean clone → `make` → run → `norminette`; then push (B11)

---

## Decisions (fill in during P1)

| # | Question | Decision | Date |
|---|----------|----------|------|
| Q1 | EDF tie-breaker | | |
| Q2 | Zero / limit values | | |
| Q3 | Compile counted at start or end | | |
| Q4 | Clock | | |
| Q5 | Architecture | | |
| Q6 | Coffman condition broken | | |
| Q7 | 1-coder behaviour | | |
| Q8 | `printf` or `write` | | |
| Q9 | Initial dongle state | | |
| Q10 | Burnout comparison | | |

## Progress log

| Date | Phase | What was done / what is next |
|------|-------|------------------------------|
| 2026-11-13 | P0 | Subject reviewed, review + to-do + research files created. Next: research recap, then P1 design session |
