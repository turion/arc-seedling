# Changelog

**`SEEDLING.md` is at version 3.** The number lives here and nowhere else — the file does not state
its own — so a bump touches one line instead of two that can disagree.

`AGENTS.md` holds the form a new entry takes.

Newest first.

- **v3** — 2026-10-25 — one `/arc` skill that carries on through compaction, one status
  vocabulary for arc, PR and todo, and an optional, unproven backlog shape for a new arc's state.
  **To update from v2:** re-write the supervisor as `skills/arc/SKILL.md` and `RUN.md` (§5, §7, §8),
  replacing `run-arc`; re-write `plan-arc` and `test-arc` as `PLAN.md` (§9) and `TEST.md` (§14),
  deleting the three old skill directories; re-write `arc-coder.md` (§6) and the `Invoked by` line of every
  agent brief; drop `scripts/context-usage.py` and the session rule, and add the compaction hook
  (§8) where the harness offers one; add the per-PR status headings and the review log, which v2
  lacked, wherever §3, §8, §11 and §12 read them. Move into companions, if installed: the challenge
  (§16), the human seat (§18), grouping (§8), the forge read (§24 Q6) and the watch-order check
  (§21), which also gains the token-saving question (§19). Journeys (§8) now nest every new plan.
  Re-write *The Status vocabulary* (§8) with its five-word table and its table of words on disk,
  and count an `aborted` PR as closed beside `done` wherever a rule reads "every PR `done`" (§3,
  §7, §8, §13, §14, §21); have a replan rewind a PR at `accept` or `done` before writing its first
  todo, and route a PR behind a `(redesign)` line to planning (§8, §9). Narrow the compaction hook
  and the core's *Compaction* to arcs with work left and the README's header, index and House rules,
  and have the hook tell a subagent to ignore it (§4, §7, §8). With journeys, write each new arc as
  `arcs/<slug>/`, named by its gist; numbered directories stay. The backlog shape (§8) is a new
  option, offered as an experiment, not an obligation.
- **v2** — 2026-10-12 — the first four arcs of a journey that rebuilt the system's unit hierarchy:
  session length becomes a measured occupancy reading, journeys and pull-request grouping arrive as
  optional shapes of the plan directory, replanning is defined once with a herd sized to the change,
  and a coder may run a fast gate itself.
  **To update from v1:** re-write `arc-coder.md` (§6), `run-arc` (§7–§13), `plan-arc` (§9) and
  `test-arc` (§14), and the three challenge lenses (§16) if installed; add
  `scripts/context-usage.py` (§8) and drop the session-grouping question and `**Session plan:**`
  bullet; with a pull-request workflow, add §24 Q6's *Never rewrite what is reviewed* and its forge
  read; with the observation store and watch orders, re-write them from §19 and §21. Journeys and
  grouping (§8) are new options to offer, not obligations.
- **v1** — 2026-10-06 — the first extraction: this system distilled out of a 6,589-line working
  corpus that had accreted on one machine, generalised off that machine, and reorganised into the
  Core / Standard / Optional layering.
  **To update from v0:** there is no v0. v1 is the baseline every later entry is a diff against.
