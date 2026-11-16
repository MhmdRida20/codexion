# Codexion — 2-Week Plan (Sun 15 → Sat 28 Nov 2026)

Target: all mandatory requirements done, tested, README written and pushed by
**Sat 28 Nov**. Evaluation booking comes after that.

Budget: about 13 hours a week (27 h in total, which includes roughly one day of slack).
Long sessions sit on Saturday and Sunday; work days are 1.5–2 h.

Phase codes (P0…P8) refer to `codexion-todo.md`; T/Q codes to `codexion-review.md`.

## Week 1 — understand, design, foundations

| Day | Time | Session | Done when |
|-----|------|---------|-----------|
| Sun 15 | ~1 h | Requirements walkthrough (in chat) + the 6 check questions | You can describe a coder's life and both endings |
| Mon 16 | ~2 h | **All the recap in one evening**: research topics 3, 4, 8 first (condition variables, deadlock, time), then 6, 7, 9 (FIFO/EDF, heap, monitor). Create the repo, check `cc`, `make`, `norminette`, `valgrind` | P0 ticked; your open questions listed |
| Tue 17 | ~1.5 h | **Design session (P1)**: I bring a proposed design, you challenge it and we settle Q1–Q10, structs, lock map, "take a dongle" pseudocode, start pattern (T21) | Decisions table filled |
| Wed 18 | ~1.5 h | **Coding starts — P2 part 1**: Makefile, header, strict number parser, argument validation | Valid args accepted, every invalid case rejected |
| Thu 19 | ~1.5 h | **P2 part 2**: time helpers, precise sleep, init / cleanup, logger | Zero leaks under valgrind |
| Fri 20 | ~2 h | **P3**: heap + small test `main`, Norm pass on everything so far | Heap test passes for fifo, edf, equal keys |
| Sat 21 | ~3 h | **P4 part 1**: dongle struct, `dongle_take` wait loop, `dongle_release`, cooldown | Two threads fighting for one dongle behave correctly in a tiny test |

## Week 2 — simulation, testing, delivery

| Day | Time | Session | Done when |
|-----|------|---------|-----------|
| Sun 22 | ~3 h | **P4 part 2 + P5**: take both dongles, 1- and 2-coder cases, coder routine | Coders loop compile → debug → refactor with correct logs |
| Mon 23 | ~2 h | **P6**: monitor thread, burnout detection, both stop conditions, clean join | **First full simulation run** ends correctly both ways |
| Tue 24 | ~1.5 h | Fix what the first runs reveal; tight-timing cases | `4 410 200 100 100` and the 1-coder case behave as predicted |
| Wed 25 | ~2 h | **P7 part 1**: argument tests, behaviour tests, log-format checks | Behaviour checklist ticked |
| Thu 26 | ~2 h | **P7 part 2**: valgrind, helgrind, norminette, relink, forbidden functions; fixes | Zero leaks, zero races, zero norm errors |
| Fri 27 | ~2 h | **P8**: README (all required sections), recode drills | README complete; each drill done in under 5 minutes |
| Sat 28 | ~2 h | **Buffer** + mock defense + clean clone → `make` → run → final push | Repo is submission-ready |

## What makes two weeks work

- **Recap in one evening.** You already have the OS-course base; only condition
  variables and timed waits deserve real time.
- **Design is a review, not a blank page.** I prepare a proposal before Tuesday so
  the session is spent understanding and challenging it.
- **Nothing mandatory is cut.** The time saved comes from deciding early and not
  rewriting, not from skipping requirements.

## Checkpoints — tell me immediately if one is missed

| Date | Checkpoint | If missed |
|------|------------|-----------|
| Fri 20 | Foundations + heap compile and are Norm-clean | Sat 21 absorbs it, P4 shifts to Sun 22 |
| Mon 23 | First full simulation run | Sat 28 buffer is used; no more slack after that |
| Thu 26 | No leaks, no data races | README shrinks to the required sections only, drills move to Sat 28 |

The slip-prone parts are the dongle logic (Sat 21–Sun 22) and race debugging
(Thu 26). Everything else is predictable.

## How each coding session works

1. I give the spec for the session: which functions, what each takes and returns,
   and the traps that apply.
2. You write the code yourself.
3. You paste it; I review it against the requirements, the Norm and the traps.
4. You explain it back in your own words (the phase's "Explain checkpoint").
