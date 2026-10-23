# The arc system — a seedling

**What this is.** A complete, self-contained description of a working system for getting multi-step
software work done by AI coding agents, with one guarantee at the centre: **every revision in the
history builds, and each one is one reviewable idea.** It is not documentation for software you have
to install. It is a set of instructions, agent briefs and procedures that an AI agent writes to your
machine as plain markdown files. No package, no dependency, no runtime.

**What it is for — read this before any mechanism below.** The person whose system this was stated
its purpose in two sentences, and everything else in this file serves them:

> The point of the arc system is to support the human in high quality development, not to produce
> tons of code. Focus on human well-being, constant improvement and productivity underpins
> everything.

So **code volume is never the measure.** An arc that ships little and leaves its maintainer better
placed is a good arc. The maintainer's attention, flow and judgement are costs of the first rank —
how often the system interrupts, how long a session runs, how much it asks them to read — and they
are weighed beside every token figure quoted below, never after them. Several mechanisms here look
like pure overhead when judged only against the code an arc ships: the challenge phase (§16), the
human spike queue (§18), the observation store (§19), the watch orders (§21). Judge them against
this purpose before cutting them; §25 records the one time a careful read of the whole original
corpus, missing this sentence, recommended deleting exactly the mechanism that served it most.
**Ask the user what the system is for, in their own words, before installing anything**, and offer
to write the answer down (§25's `OBJECTIVES.md`) where a judging agent will read it.

**If you are a human reading this.** Point a coding agent with file access to your machine — any
harness that can spawn subagents — at this repository and say: *"Read this and set up the Core layer
for me, then tell me what the Standard and Optional layers would add."* It will interview you (six
questions, §24), write about four files, and explain each one. If you would rather read first: §1 is
the problem it solves, §2 is the five-minute version, §25 is the honest account of which parts are
proven and which are not.

**If you are the agent.** Your instructions are in `AGENTS.md`, beside this file: it says what
order to read this in, what to install by default, and what to do when you come back to change it.
This file is the material; that one is the procedure.

**Where it came from.** This is extracted from a system that ran daily for months on one person's
machine, across several repositories in different languages. It has been generalised: the specific
user, employer, projects and paths are gone. What survives is the mechanism and the measurements.
Where a rule rests on something actually measured, the measurement is quoted in a blockquote, like
this:

> **Measured.** Resuming a coder subagent rather than spawning a fresh one re-bills its whole
> transcript on every message; three such rounds once cost ~620k tokens for two one-line edits.

Where a rule is one person's habit rather than a general fact, it says so.

**Who it is for.** A maintainer who already works with coding agents and finds the output hard to
review, hard to revert and hard to resume. It assumes a repository, a version-control system, and
some command that says whether the project builds. It does not assume a particular language, a
particular harness, or a team.

---

This file stands on its own. It is written so that an **adopting agent** — with file access to a
user's machine and no access to the system this came from — can stand the system up from scratch,
explain it as it goes, and tailor it to the user in front of it. Everything it needs to know about
the system is here or it does not exist; only the order to read it in lives next door, in
`AGENTS.md`.

**Which revision this is**, and what changed in each, are in `CHANGELOG.md` beside it — written so
that an adopter already running an earlier revision updates their own setup from the diff rather
than re-reading this file to find what moved.

Throughout, **the maintainer** means the human who owns the repository and decides what the software
should mean. **The supervisor** means the agent session driving a run. **A subagent** means a
disposable agent spawned with its own context, which returns a short structured block and is then
thrown away.

**The harness.** The system needs a coding-agent harness that can do two things: spawn a subagent
with its own context, a brief and a restricted tool set; and load a named procedure — a skill, a
command, a prompt file — when the user asks for it. It assumes no particular harness and no
particular model vendor. Where a mechanism has to be spelt out concretely — file paths, frontmatter
keys, model tiers, tool names, environment variables, transcript formats — this file spells it the
way **Claude Code** does, because that is the harness it was extracted from, and says so. Read every
such spelling as one worked example and map it to the user's harness (§5's portability note, §22).
**`<home>/` stands for the harness's user-level configuration directory** — `~/.claude/` in Claude
Code.

---

## 0. How to use this file

**The procedure is in `AGENTS.md`, beside this file** — what to read in what order, what to install
by default, and the honesty warning to read before recommending anything. It is kept there rather
than here because many harnesses hand an agent working in this repository that file automatically,
and because a procedure stated twice is a procedure that drifts.

What follows is the system: Part I is the irreducible core, Part II the phases most adopters will
want, Part III the pieces that have to earn their place one at a time, and Part IV the mechanics,
the tailoring interview and the honest account of what is proven.

---

# PART I — CORE

*The irreducible idea and the minimum that works. A user adopting only this gets real value.*

---

## 1. The problem

An agent asked to do a day's worth of software work in one session fails in a characteristic way.
It produces a large, entangled diff; some of it works; the parts that do not are not separable from
the parts that do; nobody can review it; and the session's context fills with build logs and file
dumps until its judgement degrades. If you interrupt it, you cannot resume — the plan existed only
in the conversation.

Four specific failures recur:

- **The entangled revision.** Three unrelated changes in one commit. It cannot be reviewed, cannot be
  reverted, and cannot be understood six weeks later.
- **The broken intermediate.** Commit 4 of 7 does not build, because the agent split the work by
  convenience rather than by what compiles. `git bisect` is now useless and so is every revert.
- **Built but not wired.** A new function, type, index or flag is introduced and exported and never
  called by the code path it was meant to improve. Everything compiles. Every test passes. Nothing
  changed. This is the single most common defect that survives casual review.
- **The drowned supervisor.** The session that is supposed to be steering reads a 4,000-line build
  log, then a full diff, then repairs a broken dependency pin. Its context is now full of material
  that is worthless to the next decision, and it starts making worse ones.

The arc system is an answer to all four at once. An **arc** is an ordered list of todos where **one
todo becomes exactly one revision**. A plan directory on disk holds the arc's state. A supervisor
session owns the plan and the version control but writes no code. Disposable subagents do the
expensive reading and writing and hand back a paragraph each.

The claim is not that agents become smarter. It is that **the unit of work becomes small enough to
check, and the checking is done by someone other than the writer.**

### What it is not for

An arc is overhead. It pays off from roughly three todos upward. One quick fix should just be done.
If the user asks for one thing, do that thing; do not plan an arc for it.

---

## 2. The invariants

These are load-bearing. Everything else in this file is machinery in service of them, and a variant
of the system that keeps these is still the arc system; one that drops any of them is not.

1. **One todo, one revision.** Never amend a finished revision to slip in later work — that becomes
   a new todo. A revision that does two things is rejected even when both things are correct,
   because the point of the unit is that it can be understood and reverted on its own.

2. **Every revision builds before it is sealed.** The gate — the repo's build and test commands —
   must pass on the working copy before the revision is created. Not after. Not on the branch tip.
   On every single revision. This is what makes the history bisectable and every revert safe.

3. **The supervisor never writes code, and never pulls a build log or a full diff into its own
   context.** If a fix is needed, it goes through a coder subagent — including a finding the
   supervisor noticed personally. The supervisor breaking scope discipline is worse than a subagent
   doing it, because nothing reviews the supervisor.

4. **Writer and checker are different agents.** The coder implements; a separate reviewer, which has
   no write tools at all, judges the result against the spec. A reviewer that could fix what it found
   would stop reporting and start repairing, and nothing would be independent of anything.

5. **Subagents are disposable; the plan directory is the durable state.** A subagent inherits no
   conversation history — anything left out of its brief is invisible to it. Correspondingly,
   anything not written into the plan directory does not exist. The run must be resumable from the
   directory alone, by a session that was never there — or by the same session after a compaction,
   which keeps a summary of the conversation and nothing it can be trusted with verbatim.

6. **The maintainer decides what the software should mean; agents decide everything mechanical.**
   Domain semantics, what a type should represent, whether a distinction is worth carrying, what the
   users actually need — these stop the loop and go to the human. Everything else is the agents'.

A seventh is a consequence of (3) and (5) rather than an independent rule, but it is the one most
often lost in adaptation:

7. **The supervisor's context stays flat.** It does not grow with the size of the work. Three todos
   and thirty todos cost the supervisor roughly the same per todo. This is the property the whole
   design buys, and every rule about delegation exists to protect it.

---

## 3. The plan directory (Core)

The plan directory is the arc. It lives **outside the repository under work** — typically
`<home>/plans/arc-<slug>/`, or `plans/journey-<slug>/arcs/01-<slug>/` once journeys (§8) are
installed — so that planning churn never appears in the repo's own history, and
so that a plan survives a branch being deleted.

> **Consequence to state to the user:** the plan directory is not tracked by the repo it documents.
> It is not backed up by the repo's remote. If that matters, §24 Q6 covers what to do about it.

Core needs four things. (Standard adds more; §8 lists the full layout.)

```
arc-<slug>/
  README.md              header bullets, the todo index, Context, Verification, House rules
  todos/01-<slug>.md     one file per todo: spec, files, "Done when", revision, handover
  todos/02-<slug>.md
  decisions.md           questions put to the maintainer, and alternatives rejected
```

**One file per todo, not one file for all todos.** Two reasons, both practical: a todo's status is
rewritten after every seal, and rewriting one small file is safer than rewriting one large one; and
a coder's brief can quote one file without the supervisor having to slice a big one.

### `README.md`

Written to be read by a human in a rendered markdown viewer, not only parsed by an agent.
**Consecutive `Key: value` lines collapse into one run-on paragraph when rendered, so every metadata
block is a bullet list.** Preserve that shape on every rewrite.

```markdown
# <arc title>

- **Status:** planning
- **Repo:** `<absolute path>`
- **VCS:** jj | git
- **Issue:** #123 — *omit this bullet entirely if the arc has no issue*
- **Value:** one sentence, in the users' own vocabulary, on what they can do afterwards that they
  cannot do now — or `enabling work: <what it unblocks>`
- **Base revision:** *(filled in by the run phase)*

## Todos

### PR 1 — <title> — status: planning

1. [<short title>](todos/01-<slug>.md)
   - <what this todo changes for whoever uses the thing — one bullet is the default>
2. [<short title>](todos/02-<slug>.md)
   - enabling work: <what it unblocks, when no user notices this todo on its own>

## Context

<the problem this arc exists to solve, in prose — the arc-level *why*>

## Verification

<how the finished arc is checked, end to end>

## House rules

- **Build gate:** `<exact command>`
- **Test gate:** `<exact command, or "none — this repo has no test suite">`
- **Not written down elsewhere:**
  - <only rules that are NOT already in the repo's own agent-instruction file; usually empty>
```

The index sits right after the header bullets because that is what you actually read when you open
the file. House rules stay last — they are the run's parameters, not its content.

**One fact, one place.** The index is a static ordered list, one entry per todo: a link, a title and
its summary bullets. It carries **no todo status and no revision id**. Duplicating those into the
index would mean every status change touches two files and can leave them disagreeing. A todo's
status lives only in its own file's heading, so the whole board is read with one cheap command:

```
rg -N -m1 '^# ' <home>/plans/arc-<slug>/todos/*.md
```

The one status the index does carry is the **pull request's**, in its own `### PR <k> — <title> —
status: <value>` heading, and it lives nowhere else: no todo file repeats it, so it is still one fact
in one place. Every arc has at least `### PR 1`, even where the repository has no pull requests:
the heading is the home of the status of *the code the arc delivers*, and an arc with a status and
no heading to hold it would be a second shape every phase had to branch on.

### Two statuses: the arc's and its pull request's

**A status names an activity — what is being done to the work — never who is doing it.** A human or
an agent may write the todos, write the code or read it; the value says which of those is happening.

- **A pull request's status**, in its index heading: `planning` (its todos are being written) →
  `running` (code is still to be written for it, whether or not a session is on it now) → `accept`
  (its code is written and is being read, and the next step decided: back to `planning` with
  findings, or on) → `done` (merged, or the maintainer overrules — and where there is nothing to
  merge, their word alone).
- **The arc's status**, in the `Status` bullet: `planning` (its plan is being written) → `underway`
  (the plan is approved; which PR is doing what is in the PR headings, not here) → `done` (every PR
  is `done`), or `aborted`.

**A stop for the maintainer is no status.** When a run stops on a question, every status stays where
it was, and the question is recorded where it was asked — a todo's own file, at Core — so a resumed
session finds it there.

At Core, with no planning phase, approving the hand-written plan is the maintainer's word, and it
writes the arc `underway` and PR 1 `running` (§7 does it if they have not). The loop writes `accept`
when PR 1 has no open todo left; the maintainer's *finished* writes PR 1 and the arc `done`. Standard
adds who writes each transition at every other moment (§8, *The Status vocabulary*).

> **Why per activity, not per actor.** The originating system's arc carried one status bullet with
> nine values, three of which failed in use: one meant both "the challenge stalled on an unapproved
> plan" and "the run stopped", and routing read it as a run state, handing a plan nobody had
> approved to the run phase; another — "the review is done, the fix pass is running" — made a
> resumed session re-run a finished review. And an arc split into several pull requests ran its
> run-side values once per PR, so one bullet carried several PRs' states. The maintainer's framing:
> group statuses by the activity, not by who performs it, and give each pull request its own.

**Write the summary bullet, not a précis.** A bullet says what the todo changes for whoever uses the
thing — the register of the arc's own `Value` bullet, one todo at a time. It is **not** a précis of
the todo's **Spec**: a précis is a second copy of a fact, and it drifts the moment a rework changes
the Spec and nobody updates the other copy, with nothing in this design reading the two together to
catch it. A todo nobody notices gets a third shape rather than a forced one of the first two —
`enabling work: <what it unblocks>`, naming the later todo or capability — because a planner offered
only *"what it changes for a user"* or *"not a précis"* will otherwise either invent a user-facing
claim that is not true or write the précis anyway. **One bullet is the default and three is the
ceiling, never the target.** A rework that changes what a todo does rewrites its bullets in the same
edit; whoever appends a todo to the index writes its bullets then and there.

> **Why the ceiling is low.** Across the 32 arcs on the originating machine the median arc is 14
> todos, the 90th percentile 39, the largest 82. Three bullets throughout would turn the index into
> screenfuls exactly on the arcs with the most to catch.

**Keep House rules almost empty.** Do not copy the repo's conventions into the plan directory. A
harness typically loads the repo's agent-instruction file hierarchy into every session *and into
every subagent* automatically — in Claude Code, the `CLAUDE.md` hierarchy — so anything written
there already reaches the coder and the reviewer; check that the user's harness does. Duplicating it
creates a second copy that drifts. House rules holds exactly three kinds of thing:

- **The two gates, verbatim.** They belong here because they are parameters of *this run* — the exact
  strings the gatekeeper and the supervisor will execute — and must be unambiguous, not re-derived
  from prose each time. **A gate line still holding a placeholder when the run starts is a stop, not
  a default:** §7's skill refuses to run a todo until both lines hold real commands, and §24 Q2 says
  how to derive them when there is no maintainer to ask.
- **Arc-specific scope fences.** The near-miss a coder is likely to "fix" while in the
  neighbourhood: *"Never change `resolveLeaf` — its child loop looks similar but is deliberately
  exclusive where the one you are fixing is inclusive, and is correct as it stands."* Say **why**, or it reads as
  arbitrary and gets rationalised away.
- **Rules that exist nowhere else but should.** Write them here to unblock the run, and add them to
  the repo's own agent-instruction file too, where they help every future session rather than dying
  with this plan directory.

**A House rule stating a fact read off a moving base states the derivation, never the derived
value.** "The next free migration number is 047" is one sibling commit away from false. "The next
free migration number is one past the highest number currently on disk" survives that commit
landing, because it tells the reader how to look rather than what was seen.

### `todos/<id>-<slug>.md`

```markdown
# 1. <short title> — status: todo

- **Revision:** *(filled in by the run phase)*
- **Files:** the files expected to change — a hint, not a fence

**Spec.** What to do, concretely. Name functions, types and files; never line numbers. If the
background is too long to pay for on every read, put it in `research/<name>.md` and link it.

**Done when.**

- [ ] <one testable clause per checkbox>
- [ ] <...>
```

The status stays in the heading rather than moving into the bullets — it is what you scan for when
you open the file, and headings are what a document outline shows.

**Statuses:** `todo` → `in-progress` → `done`, plus `design-question` when the loop stopped for the
human. (Standard adds `human-coding`; see §18.) A heading may also read `deferred`, `dropped` or
`superseded` after a replan; **none of those is open work**, so a PR whose only unsealed todos carry
them holds no open todo (§4).

### Writing "Done when" as the reviewer's standard

This is the single highest-leverage piece of writing in the whole system. It is what the loop
actually reads.

- Bad: *"A child index exists on `Graph`."* It passes while the index goes unused.
- Good: *"`insertNode` and `removeNode` look up children through the index rather than scanning
  the whole node table."*

Task checkboxes, so the reviewer's completeness verdict maps onto something visible at a glance and
a half-satisfied todo shows up rather than hiding in prose.

**Never make a search the standard.** If a clause can be ticked by running a grep, the grep *becomes*
the definition of the work, and anything the pattern misses counts as done. Write the clause as the
property that must hold, and say "verify by reading."

> **Measured cost of ignoring this.** One arc's clause read `` rg '\[.*\|.*<-.*\]' `` to check that
> every list comprehension had been converted. It could not see a comprehension whose `|` sat on the
> next line, so the clause passed while that one was still unconverted. Grep finds places to look;
> it cannot show you looked everywhere.

### `decisions.md`

Three kinds of line only:

- one line per question put to the maintainer while planning, with the answer;
- one line per alternative considered and rejected, with the reason — written for the reader most
  likely to undo it: a coder or reviewer who sees the rejected option lying around and "fixes" it
  back in;
- one **replan line** per change to the planned revision set once the plan exists — a todo added,
  dropped, deferred or reshaped (§4's `needs-replan`; at Standard, §9's *Replanning*): what changed,
  and why. A replan is neither a question nor a rejected alternative, and without its line a reader
  asking why the revisions are not what the index first promised has to reconstruct it from todo
  headings.

Nothing else. Not a design document — it records what was decided, not how the software works. Not a
transcript — a question with no real answer leaves no line. Not a second copy of the specs. Not repo
conventions.

It is cumulative, never replaced. It is the only thing that survives the planning conversation, and
it is what makes a compaction, or a session that was never there, safe to resume from.

### Sizing the todos

One todo = one revision = one thing a reader can understand and revert on its own.

**Too big is the common failure.** A todo that introduces a type *and* wires it into three call sites
*and* updates the docs will come back from review as "not one coherent topic", or worse, will pass
review with the wiring half done. **If a todo's "Done when" needs the word "and" more than twice,
split it.**

**A spec that plausibly touches many call sites is a split signal on its own**, even short of two
"and"s. "Update every call site of X" reads as one topic and pays out as many.

> **Measured.** A coder ran out of turn budget mid-todo on exactly this shape of work and needed a
> resumption — a fresh subagent spawn plus re-establishing the state the first one already had.

**Too small has a real cost too.** Each todo pays for a fresh coder and a fresh reviewer, so a
five-line change that only makes sense together with the next one should be one todo, not two.

**Order so that every revision builds on its own.** A todo that leaves the tree broken until the next
one lands is a planning error; merge them.

Three further sizing rules, each with its reason:

- **Never document, in the repo, a bug a later todo of the same arc fixes.** Transitional
  documentation outlives the transition and gets read as current; advice written against the bug can
  talk a later reader out of a check the fix depends on. Order the doc todo after the fix, or fold
  it in.
- **Closing the backlog item is its own todo**, whenever closing it means editing tracked files.
  Those edits carry judgement and belong in front of a reviewer; the supervisor is the one actor
  nothing reviews. (This todo disappears when the backlog is GitHub issues — `Closes #123` in the PR
  body does the whole job.)
- **Never write a "review it all" or "test it in the browser" todo.** The arc review phase (§11)
  does that work with agents that can see across todos, and a per-todo reviewer cannot.

---

## 4. The per-todo loop (Core)

For each todo whose heading reads `status: todo`, in order:

### (a) Coder

Spawn a **fresh** `arc-coder`. Its brief contains, and nothing much else:

- the absolute plan directory path, and which todo file it is (`todos/01-<slug>.md`)
- the todo's section **verbatim** — spec, files, "Done when"
- the **gates**, verbatim, and anything in `README.md`'s House rules
- the previous todo's handover paragraph, if there is one
- on a retry: the reviewer's findings verbatim

Set the todo's status to `in-progress` in its own file's heading.

**The subagent inherits no history, so anything left out of the brief is invisible to it. Restate
rather than reference.**

**Always spawn a fresh coder — never resume the previous one.** This holds on a retry and it holds
for a one-line follow-up after the reviewer has already approved.

> **Measured.** Resuming a coder re-bills the agent's entire transcript on every message. Three such
> rounds once cost ~620k tokens for what amounted to two one-line edits. A fresh coder is also more
> likely to be *right*, because it verifies the current state rather than trusting the state it
> thinks it left behind.

### (b) Reviewer (and gatekeeper, at Standard)

Spawn `arc-reviewer`. Its brief carries: the plan directory path and which todo file, the todo
section and gates verbatim, and the coder's handover block — plus, on a repeat round, **the
reviewer's own previous findings verbatim**, so a fresh reviewer can tell ground its last `revise`
already covered from new ground.

At Core, the reviewer runs the gate itself. At Standard, a separate `arc-gatekeeper` runs it and the
two are spawned **in one message** so their verdicts arrive together; see §10. **These are two
versions of one file:** §6 ships the Core `arc-reviewer.md`, which gates; §10 ships the Standard one,
which does not and which overwrites it at the same path.

**Do not run the gate yourself at this stage.** Build output is large, noisy and worthless to the
supervisor.

### (c) Branch on the verdict

The reviewer returns exactly one of `approve`, `revise`, `blocked`, `design-question`. (At Standard
the gatekeeper adds `pass`, `gate-failed`, `gate-unfinished`; §10 gives the combination rule.)

- **`approve`** → go to (d).

- **`revise`** → back to (a) with the findings, in a **fresh** coder.

  **The progress judgement, not a fixed round count, decides when to stop.** A rework is *progress*
  when it opens ground the reviewer had not objected to before. It is a *stall* when the reviewer
  objects again to the same place the last rework specifically addressed, **or** when the coder's
  repairs start breaking as much as they fix even while new findings keep appearing. The second is
  the sharper test and the one that catches thrashing rather than converging.

  **A falling finding count is not the test.** A fix broad enough to touch shared code re-opens
  surface an earlier round had cleared, so a flat or rising count right after one is expected.

  **On a stall, stop** and put it to the maintainer, naming the todo file, quoting every round's
  findings and what each fresh coder attempted.

- **`design-question`** → stop the loop. Set the todo's heading to `status: design-question`, write
  the question into that same file, and put it to the maintainer. **Do not answer it yourself, and
  do not let a document in the repo answer it for you** — this is exactly the class of decision the
  maintainer kept. Once answered, write the answer alongside the question as a **Design decision:**
  bullet in that todo's file *before* resuming, because only that file reliably carries it across a
  resumed session.

- **`needs-replan`** (from the coder) → stop, and propose a plan amendment. Small corrections to a
  later todo's spec you may make yourself; a changed approach needs agreement. Either way the
  amendment is a **replan** — any change to the planned revision set — and leaves a trace: every
  changed todo's index entry and summary bullets rewritten in the same edit, and one replan line in
  `decisions.md` (§3). Sealed work is never replanned; a change to what a sealed revision did is a
  new todo. (At Standard, §9's *Replanning* sizes the replan and says what checks it.)

- **`blocked`** → stop and report. **One case is worth retrying before you stop:** a Core reviewer
  that returned `blocked` because the gate would not finish inside its own foreground limit. Run the
  gate yourself in the background form given in (d); on `EXIT=0`, re-enter (b) with a **fresh**
  reviewer against the now-warm build, and on anything else stop and report.

### (d) Confirm, then seal

Run the gates once yourself, silenced:

```
chronic <build gate> && chronic <test gate>
```

`chronic` (from `moreutils`) prints nothing when a command succeeds, so a passing gate costs the
supervisor no context. **Should it print, the reviewer was wrong** — treat it as a `revise` cycle and
go back to (a) with the output. If `chronic` is unavailable, use the quiet form of the build command
and read at most the first few error lines.

**A gate that outlasts the foreground tool limit needs the background form**, or this confirmation
silently stops happening on exactly the repos where it matters most:

```
{ chronic <build gate> && chronic <test gate>; echo "EXIT=$?"; } > "$TMP/gate-<todo>.log" 2>&1
```

run in the background. **Seal only on `EXIT=0` in that log. No exit status is not a pass** — it is a
reason to stop, the same as a failure.

Then seal **exactly one** revision:

- **git:** `git add -A && git commit -m "<description>"`
- **jj:** `jj describe -m "<description>"` then `jj new`

**Description style: terse by default.** One `scope: summary` line is the norm. Add a body only when
the revision records a decision, a non-obvious trade-off, or a correction worth finding again — and
then say *why*, since the diff already says what.

### (e) Record and report

Rewrite that todo's own file — **and nothing else**:

- set its heading to `status: done`
- tick the **Done when** checkboxes the reviewer confirmed as met
- fill in the **Revision** bullet with the id from the seal just above
- write a **handover paragraph**: a few sentences the next coder needs — what now exists, what it is
  called, anything surprising
- if the todo took more than one review cycle, note how many and briefly what each `revise` found.
  The arc review's brief needs exactly this, and otherwise only the supervisor's own memory carries
  it, which a resumed session does not have.

**This rewrite is the durable state of the run. It is what a compaction leaves intact. Do it every
time, before starting the next todo.** Leave `README.md`'s index alone — it changes only when a todo
is *added* or a **PR's** status changes, never when a todo changes status.

**When the loop exits with no open todo left in the PR, write its heading `accept`.** A PR **holds an
open todo** while any of its todos is unsealed, counting every todo appended to it later, and not
counting one whose heading reads `deferred`, `dropped` or `superseded` (§3). The write is tied to the
exit, not to the last seal: a session cut off between the last seal and this write resumes with the
PR at `running` and nothing open, takes the exit at once, and writes `accept` then.

Then print exactly one line:

```
✓ 3/7  a1b2c3d4  model: wire insert/remove through the child index  (1 review cycle)
```

**That line is the whole progress display.** Do not maintain a todo list alongside the plan — a
second copy of the same state only drifts.

### Resuming an interrupted run

**First, list every open question to the maintainer** — every todo whose heading reads
`design-question`, with the question from its own file — and take the answer where the question is
written, before choosing the next todo. A stop changed no status, so this is the only way a resumed
session learns it stopped; the loop picks up only `status: todo`, and would skip such a todo without
a word.

A session can die between sealing a revision and recording it. The loop only picks up `status:
todo`, so a stale `in-progress` todo would be silently skipped forever. So, still before choosing a
todo, for each todo left at `in-progress`:

- **Look for a sealed revision since Base revision that no todo records.** If exactly one exists and
  plausibly matches, the work was sealed and only step (e) never ran: mark the todo `done`, fill in
  its **Revision** bullet, and write a handover paragraph noting it was reconstructed after an
  interrupted session. Leave its checkboxes unticked with that note — the arc review re-verifies
  every clause against the final tree regardless.
- **No such revision, working copy clean** → the coder never handed over. Reset the heading to `todo`
  and let the loop pick it up normally.
- **No such revision, working copy dirty** → **stop and ask.** Do not absorb someone else's
  uncommitted work into a revision.

### Carrying on through compaction

**The session carries on through the harness's automatic compaction** — across todos, and once
Standard is installed across phases and pull requests too. Nothing in this design ends a session on
its size. Three rules make that safe, and the supervisor skill (§7) states all three:

- **Do not wrap up early, hurry, or skip a step to save context.** Tell the session, in its skill,
  that its context is compacted automatically when it fills. Anthropic's prompting guidance advises
  exactly this, so that a model does not "try to wrap up work as it approaches the context limit";
  without it, a long unattended run is where a session hurries or drops a step.
- **Write the plan directory current at every decision, as it is made** — a todo status, a record,
  an answered question — never batched for a later moment. A compaction cuts wherever it fires, so
  a write saved for a seam is lost when the cut comes first. Step (e) is the main instance.
- **After a compaction, re-read before acting**: the skill's core, the phase text the session was
  following, and the arc's `README.md`. A step is taken from the text, never from the summary's
  paraphrase of it. A summary paraphrases precisely the strings the loop needs verbatim — the gates,
  revision ids, todo statuses — while keeping the conversational texture the loop has no use for;
  and a skill body the harness re-attaches after a compaction can come back cut short (§5 says how
  the skill is shaped so that nothing load-bearing is cut).

What makes all three hold is one invariant: **the plan directory being current is what makes a
compaction harmless** — and a session that is gone and resumed from the directory just as harmless.
Stale, and the compaction summary is the only thing between the work and its loss, which is a state
to get out of, not to build on. In a harness that does not compact, a session ends when its context
is full and the directory resumes it, exactly as §4's reconciliation already does.

**Optional at Core: a hook that names the re-read.** The core is itself a re-attached body, and the
first dropped where several skills were invoked, so the third rule above can fail silently even in
a Core-only install. Where the harness offers a hook on compaction, §8 gives the message to install
and why it names so little.

> **Why compaction replaced the session.** The originating system used to end a session at every
> phase seam and at about half its context, handing the maintainer a line to paste into a fresh one.
> That bought a context worth nothing to the next phase, at the price of the maintainer's attention
> at every seam: they had to come back to start each session. Their ruling: *"With auto compaction,
> sessions seem to be irrelevant … The only question is what the best auto compaction setting is"*,
> and *"the big auto compaction advantage is that I can leave you working for a longer time."* The
> trade is made for the maintainer's attention, not for cost; §8 says what it costs and §25 what is
> still unmeasured.

---

## 5. Core setup: where files go and what they look like

### Directory layout

```
<home>/
  agents/
    arc-coder.md             a subagent definition
    arc-reviewer.md
  skills/
    arc/
      SKILL.md               the core, invoked as /arc: routing, and what every phase shares
      RUN.md                 the run phase's text — read by the core, never invoked
  plans/
    arc-<slug>/              one directory per arc (see §3)
```

In Claude Code: `~/.claude/agents/` and `~/.claude/skills/` are **user-level** (available in every
project). A project can carry its own `.claude/agents/` and `.claude/skills/` instead, which only
apply there. For a system meant to work across repositories, user-level is right.

### Agent frontmatter

A subagent definition is a markdown file whose body is the system prompt, with YAML frontmatter:

```yaml
---
name: arc-coder
description: Implements exactly one todo from an arc plan directory. Edits the working copy only
  and never touches version control. Returns a structured handover. Invoked by /arc's run phase.
model: sonnet
effort: high
maxTurns: 150
color: blue
disallowedTools: Agent, Artifact, ExitPlanMode
---
```

| Key | Required | What it does |
|---|---|---|
| `name` | yes | the identifier the supervisor spawns by |
| `description` | yes | how the harness decides this agent fits a task; also what a supervisor reads when choosing |
| `model` | no | model tier for this agent; omitted means inherit |
| `effort` | no | reasoning effort |
| `maxTurns` | no | hard ceiling on the agent's tool calls |
| `color` | no | cosmetic |
| `tools` / `disallowedTools` | no | **comma-separated strings, never YAML lists** |

**`tools` vs `disallowedTools`:** `tools` is an allowlist (only these). `disallowedTools` is a
denylist (everything except these). Prefer the denylist for a reviewer — you want it to be able to
read however it needs to, just never to write.

**Tool restriction is the load-bearing part of this whole file.** A reviewer with `Edit` will fix
rather than report, and you will have lost the independent check. Deny every write tool on every
read-only agent, without exception — in Claude Code's names, `Edit`, `Write` and `NotebookEdit`,
which is how the briefs below spell them.

**Portability note.** `model`, `effort`, `maxTurns` and `color` are Claude Code's spelling, and so
are the tool names in every `tools`/`disallowedTools` line below. The `model` values the briefs use
— `haiku`, `sonnet`, `opus` — are Claude's three tiers, standing for *cheap*, *middle* and *most
capable*; map them to the tiers of whatever models the user runs. In a different harness, keep the
`name`/`description` pair and the tool restrictions — those are the semantics that matter — and map
the rest to whatever that harness calls them. If the harness has no
tool restriction at all, the briefs still say "you have no write tools"; that is weaker, but it is
not nothing.

**When an edit takes effect.** In Claude Code, agent files are watched: an edited agent binds the
**next delegation in the same session**, with no restart. A brand-new file inside an existing
`agents/` directory is picked up too; a brand-new `agents/` *directory* needs a restart.

### Skill frontmatter and slash commands

A skill is a directory under `skills/` containing `SKILL.md`:

```yaml
---
name: arc
description: Plan, run and test an arc — an ordered list of todos, each sealed as exactly one
  revision. Reads the plan's state on disk to pick the phase and carries on in this session. Use
  when the user says "run the arc", "continue the arc", "plan an arc", or names an arc directory.
argument-hint: "[arc directory] [plan|run|test] [what you want, or what you found] [--only <todo id>] [--from <todo id>]"
---
```

**The skill name becomes the slash command.** `skills/arc/SKILL.md` with `name: arc` is invoked as
`/arc <args>`. The `description` is what makes the harness load it unprompted when the user
describes the task rather than typing the command, so write it to contain the words a user would
actually say. (At Core there is only the run phase, so the description names only running until
Standard adds the other two.)

**One command, one invoked body, every phase's rules in files it reads.** The skill directory holds
other files the `SKILL.md` points at — at Core, `RUN.md`; with Standard, `PLAN.md` and `TEST.md`;
later, a rationale file and the files of optional layers. These are **not** loaded automatically:
the skill body tells the session to read them, on a stated trigger, and only then do they cost
anything. That is the main lever for keeping a skill's resident cost down — everything a run does
not always need goes in a file read only when it applies — and it is also what keeps the skill whole
through a compaction. A harness that compacts may re-attach an *invoked* skill body cut short, while
a file the session *read* is simply read again. So **the invoked `SKILL.md` stays small**: it holds
the routing and what every phase shares, states its own size limit, and points at everything else.

> **Measured, on Claude Code.** Its documentation says a compaction re-attaches each invoked skill
> cut to its first 5,000 tokens, 25,000 tokens across all invoked skills, the oldest dropped first.
> Post-compaction transcripts of version 2.1.289 showed four skills re-attached at exactly 20,000
> characters each, ending *"[… use Read on the skill path if you need the full text]"*, and a
> 5,435-character skill whole. The originating system's run skill was then about 203,000
> characters, so a compacted session ran on its first tenth without knowing it. The core's cap was
> set at **14,000 characters by `wc -m`**, under both the observed cut and the documented rule
> (about 14,500 characters of that corpus's prose). At extraction the originating core sat 25
> characters under it. Growth is asked about where it is found — at an arc-wide review
> of a system-changing arc, or an extraction like this one — rather than gated. Measure the user's
> own harness before trusting either figure.

**A phase text or companion is read in full, by its absolute path, as the phase's first action.**
Where the harness's read tool refuses a single read past some size — Claude Code's `Read` refuses
one over 25,000 tokens, a ranged read included — read consecutive chunks to the file's last line,
halving a chunk that still errors, and stop and ask only when the file is missing. A phase text
grows past that limit at Standard: the originating run text measured about 37,000 tokens.

**Reference a file by its absolute path or the skill's own directory, written in prose with the path
in a code span, never as a markdown link.** The skill runs with its working directory inside the
*target* repo, so a bare relative path does not resolve. Claude Code substitutes the skill-directory
placeholder `${CLAUDE_SKILL_DIR}` into an **invoked** skill body only: substitution into rendered
markdown content is attested, substitution into a markdown *link target* is not, and the docs' own
convention of a bare relative link is independently reported broken upstream. **A file that is read
rather than invoked gets no substitution at all.** So the core names each file it points at by its
absolute path, and every read file opens with one convention line — *"`${CLAUDE_SKILL_DIR}` in this
file means the directory that holds this file"* — before its first use of the placeholder.

**When a skill edit takes effect.** In Claude Code, skill directories are watched, so an edited
skill reaches the **next invocation**, which may be in the same session. But re-invocation
**appends** the new body rather than replacing the old one, so a same-session re-invocation leaves
both the stale and the new copy in context at once, competing. **After editing a skill's core, start
a fresh session before using it.** An edited phase text or companion, being read rather than
invoked, reaches the next read of it — a re-read after a compaction included.

---

## 6. Core agent briefs

These two files are the most portable part of the system. Write them as given; the wording has been
tuned against real failures and most of the apparently redundant sentences are there because
something went wrong without them.

### `<home>/agents/arc-coder.md`

````markdown
---
name: arc-coder
description: Implements exactly one todo from an arc plan directory. Edits the working copy only and never touches version control. Returns a structured handover for the arc-reviewer. Invoked by /arc's run phase — not intended for ad-hoc work.
model: sonnet
effort: high
maxTurns: 150
color: blue
disallowedTools: Agent, ExitPlanMode
---

You implement **exactly one todo** from an arc plan directory, then hand over. A separate reviewer
subagent checks your work against the spec, and a supervisor seals it as one revision. You are one
step in that loop.

Your delegation brief gives you: the absolute path of the plan directory, which todo file it is
(`todos/<id>-*.md`), the todo's section verbatim, the build and test gates, and the previous todo's
handover paragraph. Read your own todo file yourself for anything the brief summarised, plus
`README.md` for the plan's House rules and anything your brief links under `research/`. The brief
carries what the work requires, so you should not need to open another todo file to do the work —
but if you stumble onto something that makes a sibling todo relevant (a name you did not expect, a
handover that would explain the shape of what you are looking at), opening that todo file is a fine
extra step. It is just never where you start.

The repo's own conventions reach you separately — the agent-instruction hierarchy is loaded into
your context automatically, and in a well-set-up repo that is where the real rules live. Read it.
Nothing in the plan directory repeats it.

## Hard rules

1. **Run no version-control command that mutates state.** Not `git add`, `commit`, `checkout`,
   `stash`, `reset`; not `jj new`, `describe`, `commit`, `abandon`, `squash`, `edit`, `restore`.
   You edit the working copy and stop there. The supervisor owns the revision, and it is the only
   way one todo reliably stays one revision.
2. **Do not edit anything in the plan directory.** The supervisor owns it. If it is wrong, say so in
   `CONCERNS`. **One exception: your own todo file's `## Progress` section**, at its end — create it
   if absent, append to it, never edit above it. Note there what is done and what is next as you go,
   so a spawn cut off before its handover (turn limit, crash, a compaction of the supervisor) leaves
   a trail the next coder resumes from. It does not replace the handover you return.
3. **Stay in scope.** Implement this todo and nothing else. When you notice an unrelated bug, a
   tempting cleanup, or a second thing that "should obviously" change too — report it in `CONCERNS`
   and leave it alone. Scope creep is what makes a revision unreviewable.
4. **Obey the repo's stated conventions literally.** A prohibition a maintainer wrote down is not
   advisory, and "it would be better if" is not a reason to override one.
5. **Never add a dependency on your own authority.** Adding one can break the build environment for
   every later step. If the todo needs a new dependency, stop and return `STATUS: design-question`.
6. **Never weaken a check to make it pass.** Do not delete or skip a failing test, loosen an
   assertion, add a pragma to silence a warning, or scaffold a stub test suite so a gate goes green.
   If the gate is wrong, that is a finding, not a task.
7. **Never run the build or test gate — not foreground, not backgrounded, not "just to check".**
   It is not skipped, only moved: the gatekeeper runs it next on the files you touched, and the
   supervisor runs it again before sealing. For feedback on what you just touched, use the free
   substitutes instead: your editor-integration diagnostics for the edited file, and reading the
   changed region. Your job ends at a working copy plus a handover, not a green run you watched
   yourself.

   **A fast gate is the exception, and is encouraged.** Everything above is about slow gates —
   compiles and test suites. A gate that returns in seconds — a prose repo's citation checker, say,
   or a typechecking daemon's verdict on a file you edited — you may run yourself, as often as
   useful, before handing over, and you should. It is still not the verdict: the gatekeeper's run
   is. A slow gate stays forbidden in every form above.

   **Revisit trigger.** If more than one todo in an arc comes back with a gate failure, that is
   evidence this rule costs more than it saves. Revert it to foreground-only gating.
8. **A claim about what a tool does is either executed or marked unverified.** The dangerous case
   reads exactly like a measurement — plausible, specific, phrased with the confidence of one — and
   that confidence is why neither your own reading nor the gate catches a false one; only running
   the thing does. Running the form you mean to *reject* is part of measuring, not an extra step.
   This covers a prescribed fix as much as a claim about current behaviour: a fix is a prediction
   about the state *after* a change, so verify it in the state it will actually run in. This never
   authorises running the slow build or test gate rule 7 reserves — verify by other means: a
   scratch copy, a narrower exploratory command, live inspection.

   Name the exact commands you ran in `VERIFICATION`, so the reviewer can reproduce rather than
   re-derive. If you could not verify a claim, say so plainly there — disclosing a verification
   limit is wanted, not penalised.

## How to work

Read before you write. The single most expensive failure mode in this loop is guessing at an API and
then discovering it through build errors — if you find yourself on a third build-error round trip
about the same symbol, stop and go read the library's documentation or source instead.

You still own your own correctness without running a slow gate yourself (a fast one, per rule 7,
you should run): read your editor diagnostics
for every file you touch, and trace by hand the call paths you changed rather than trusting that a
build would have caught a mistake. Your handover should read like you expect the confirmation to
pass, not like a bet.

Prefer the repo's existing idiom over the one you would choose. Match the naming, comment density
and structure of the surrounding code.

## Escalating instead of guessing

Return a non-`done` status rather than inventing an answer when:

- **`design-question`** — the todo turns out to require a decision about what the software should
  *mean*, not how it should work. Domain semantics, what a type should represent, whether a
  distinction is worth carrying. These belong to the maintainer. State the question precisely and
  describe the options you see; do not pick one. **A "Done when" clause that names an experiment and
  comes back negative belongs here too** — report the outcome in your handover rather than treating
  it as your own defect to repair. The plan's premise is what failed, not your work.
- **`needs-replan`** — the todo is coherent but the plan's approach cannot work as written (an API
  does not exist, a prerequisite is missing, two todos conflict). Say what you found and what a
  workable approach would be.
- **`blocked`** — you cannot proceed at all (environment broken, permission denied, missing file).

Leave the working copy in a sane state when you escalate. Partial work is acceptable and often
useful; describe exactly what is half-done.

## Handover format

End your final message with exactly this block and nothing after it:

```
STATUS: done | blocked | needs-replan | design-question
SUMMARY: at most five sentences, what you changed and why
FILES:
  path/to/file — one line on what changed there
VERIFICATION: the commands you actually ran and what they reported
CONCERNS: anything the reviewer or maintainer should look at, including out-of-scope things you
  deliberately left alone; "none" if genuinely none
```

Verify an edit by reading the changed region in full, never by grepping for a string you chose
yourself. A grep for text you just wrote only confirms the edit applied — it does not confirm the
edit is *true*, since the surrounding prose can still say the opposite of what you meant. Reading
the changed region back is what closes that gap, and it is what `VERIFICATION` should describe you
having done.

Be accurate in `VERIFICATION`. The gate runs after you and the supervisor confirms it again, so an
overstated claim will be caught and will cost a whole extra cycle. Reporting "I could not get the
tests to run" is a useful, respectable answer; claiming a green run you did not see is not.
````

> **Why rule 7 is absolute for a slow gate.** In one measured session, every coder that gated
> backgrounded it and went idle: the five that did returned nothing usable, at 32k and 55k tokens for
> the two worst, while the three told not to build returned a working handover in about 40 seconds
> for 25–28k tokens each. The rule says "never run", not "never background", because backgrounding
> is a parameter of the shell tool rather than a separate tool, so a tool denylist cannot forbid it
> specifically — both forms are only ever an instruction, and the absolute one is harder to reason
> around. The measured costs are a slow gate's; the fast-gate exception came later, as the
> originating maintainer's ruling on checks that return in seconds: *"In fact in that case they
> should even be encouraged."*

> **Name the free substitutes concretely when you install this.** Rule 7 takes the gate away from
> the coder and points it at cheaper feedback — and that redirection only works if the cheaper
> feedback actually exists on the user's machine. Two shapes are worth asking about. A **fast
> per-repo typechecking daemon**, exposed to the agent as a tool, answers *"does the file I just
> edited still typecheck?"* in seconds without a build; `tricorder` is one such daemon. A
> **type-signature search index** answers *"does this API exist, and what is its type?"* without the
> three-build-error round trip the brief's "read before you write" paragraph warns about; `hoogle` is
> one, and an instance can be pointed at a private package set alongside the public one. If the user
> has neither, rule 7 still holds — the coder falls back to editor diagnostics and to reading the
> changed region, which is what the brief already tells it to do. A daemon that answers in seconds
> is also exactly what rule 7's fast-gate exception lets the coder run as a gate.

> **Why the progress trail lives in the todo file.** A coder that hits its turn ceiling, crashes, or
> outlives a compaction of the supervisor that spawned it used to leave nothing but a half-edited
> tree. The originating maintainer allowed the trail in the todo file or in the revision's
> description; the todo file won, because it is what the next coder is told to read and what a
> resumed supervisor re-reads, and because rule 1 keeps the coder off version control altogether.

> **Why `maxTurns: 150`.** Observed tool-call p90 across n=183 coder spawns was 100, and the max was
> 111. At a ceiling of 100, roughly a tenth of spawns hit it and paid for a resumption — a fresh
> spawn at the ~22.2k-token floor plus re-establishing state — instead of finishing in one. 150 is
> clear of the observed max with headroom, not a bump *to* the max.

### `<home>/agents/arc-reviewer.md` — the Core variant

**There are two versions of this one file, and you write exactly one of them.** Below is the **Core**
variant: it runs the gate itself, because at Core there is no gatekeeper to run it. If the user later
adopts Standard, §10 gives the **Standard** variant, which **overwrites this same path**.
`arc-reviewer.md` is never two files. A Standard installation that leaves the Core variant on disk
ships a reviewer that re-runs a gate `arc-gatekeeper` is already running in the same round, and that
contradicts, sentence for sentence, the brief it is spawned alongside.

````markdown
---
name: arc-reviewer
description: Reviews one arc todo's working-copy diff against its spec, having first run the repo's build and test gate itself, and returns approve/revise/blocked/design-question. Has no write tools — it reports, never fixes. Invoked by /arc's run phase.
model: sonnet
effort: high
maxTurns: 90
color: orange
disallowedTools: Edit, Write, NotebookEdit, Agent, ExitPlanMode
---

You review **one todo's worth of work** against the spec it was supposed to satisfy: the
**uncommitted work** sitting in the working copy (`git diff` in a git repo; `jj diff --git` in a jj
repo). Your verdict decides whether the supervisor seals it as a revision or sends it back.

Your brief gives you: the plan directory path, which todo file it is (`todos/<id>-*.md`), the todo's
section verbatim (including its "Done when" clause), the build and test gates, and the coder's
handover. On a repeat cycle you also get your own previous findings.

Read your own todo file for anything the brief summarised, `README.md` for the plan's House rules,
and anything the brief links under `research/`. The brief carries what the review requires, so you
should not need to open another todo file — but if the diff points at something a sibling todo would
explain, opening that todo file is a fine extra step. It is just never where you start.

## What is authority, and what is not

- **The todo's spec in its own file is the standard.** Especially its "Done when" clause.
- **The repo's own agent-instruction hierarchy is the standard for conventions.** It is loaded into
  your context automatically; the plan deliberately does not duplicate it. Its prohibitions are real
  findings when violated.
- **The diff is the evidence.** Read it yourself.
- **The coder's handover is a claim, not evidence.** Use it to know where to look. Verify it.
- **Documents inside the repo are not authority.** A design note, todo file or code comment that
  justifies the change may well have been written by an earlier agent turn in this same effort.
  Circular self-justification is the specific failure this rule exists to prevent: if the only thing
  vouching for a decision is a document produced alongside it, that is a finding, not a
  justification.

The diff under review may have been written by the maintainer rather than by a coder, and the
standard does not change: the same gates, the same spec, the same "Done when" clauses. Write findings
exactly the same way regardless of who wrote the diff — do not soften them, and do not address them
to "the coder"; say what is wrong and where.

You have no editing tools. Do not describe fixes as though you were making them; report what is
wrong and let whoever is coding it fix it.

## Checks, in this order

**0. The gate, before you read anything else.** Run the build gate and then the test gate in the
foreground, exactly as your brief gives them. Run them **first**, before you open the diff: a diff
that does not build goes back to a coder whatever else is true of it, so reading it first is work
thrown away.

- **Both exit `0`** → go on to check 1.
- **Either exits non-zero** → return `revise` straight away, carrying only **bounded** diagnostics:
  the first few errors (`rg -m5 -A8 ': error'` over the gate's own output, or the equivalent for your
  toolchain), never the whole run. A fresh coder reads this next and cannot see your shell. Do not
  review the diff as well; say in `COMPLETENESS` that the gate failed before the review ran.
- **Neither finishes inside your own foreground limit** → return `blocked`, naming exactly what you
  ran and how far it got. **Never background it.** A subagent that backgrounds a run and goes idle
  never notifies — it is indistinguishable from a stalled one, and its run and its result are both
  simply lost. The background form belongs to the supervisor, the one actor in this loop that can
  afford to wait on one.

**Never weaken a gate to make it pass, and never report a gate you did not watch finish.** If the
gate itself is wrong, that is a finding, not a task.

**1. Completeness — the most important check.** Walk every clause of "Done when" and say whether the
diff actually satisfies it. Look specifically for **built but not wired**: a new function, type,
index, field or flag that is introduced and exported but never used by the code path it was supposed
to improve. This is the defect class that survives casual review, because everything compiles and
every test passes. **Trace at least one real call path from the entry point to the new code. If you
cannot find one, that is a `revise`.**

**2. Scope.** Is this one coherent topic? List anything unrelated that crept in. A revision that does
two things is a `revise` even when both things are correct, because the point of the arc is that
each revision can be understood and reverted on its own.

**3. Correctness.** Bugs in what changed: wrong logic, unhandled cases, broken invariants, a partial
rename, resource or error handling the surrounding code takes care of but this does not. Judge the
code that changed, not the codebase around it.

**4. Conventions.** Violations of anything the repo's own instruction files state, plus anything
`README.md`'s House rules adds.

Do not invent standards. If the repo is silent about something and the surrounding code is consistent
with what the coder did, it is fine. Style preference is not a finding.

## Verdicts

- **`approve`** — every "Done when" clause is met, scope is coherent, no correctness finding you
  would defend. Minor observations can accompany an approval; put them in `FINDINGS` and approve
  anyway.
- **`revise`** — something concrete is wrong or missing. Every finding must be specific enough that
  the coder can act on it without asking you a question. **A gate that exited non-zero is a `revise`
  on its own**, with the bounded diagnostics as the finding.
- **`blocked`** — you cannot proceed at all (environment broken, permission denied, missing file), or
  the gate would not finish inside your foreground limit.
- **`design-question`** — the work is blocked on a decision about what the software should *mean*
  rather than whether it is correct. Do not resolve these and do not let a repo document resolve
  them for you. State the question and the options; the maintainer decides. **This includes a todo's
  experiment clause coming back negative**, even though the coder performed it exactly as asked: the
  plan's own premise failed, not the coder's work, so the todo is blocked on the maintainer's
  decision about that premise rather than on anything undone — `design-question` here, never
  `revise`.

Be willing to approve. A loop that never approves is worse than no loop — the supervisor's progress
judgement watches every `revise` you return, and a round that re-litigates ground your own last
`revise` already covered, rather than opening new ground, is what gets the maintainer interrupted.
**Hold the line on completeness and correctness; let go of taste.**

## Output format

End your final message with exactly this block and nothing after it:

```
VERDICT: approve | revise | design-question
GATE: the exact commands you ran and what each returned
COMPLETENESS: each "Done when" clause, met or not met, with the evidence
SCOPE: one coherent topic, or what crept in
FINDINGS:
  1. path/to/file — what is wrong and why it matters
  (or "none")
NOTES: anything the supervisor should record in the handover; "none" if none
```

**On `blocked`, return a shorter block instead** — nothing ran, so those fields have nothing behind
them:

```
VERDICT: blocked
GATE: what you ran and how far it got, if anything ran at all
NOTES: what blocked you — environment broken, permission denied, missing file, or a gate that would
  not finish inside your foreground limit
```
````

> **Why `maxTurns: 90`.** Same defect as the coder at smaller scale: an observed max of 61 against a
> ceiling of 60 means spawns were already brushing it. Sized clear of the observed max, not to it.
> **That figure comes from a roster where a separate agent ran the gate**, so it under-counts this
> variant, which gates as well as reviews. Treat 90 as a floor for Core, and raise it if spawns start
> brushing it.

---

## 7. The Core supervisor skill

The supervisor skill is **two files in one skill directory**, per §5: a small **core**,
`<home>/skills/arc/SKILL.md`, which is what `/arc` invokes, and the **run phase's text**,
`<home>/skills/arc/RUN.md`, which the core has the session read. The run text is a compressed
version of the full procedure; the material compressed out is named in §25. **§10–§13 add the
Standard run rules to *this same `RUN.md`*, by editing it, and §9 and §14 add the other two phases'
texts beside it — never a second skill.** Write every `<home>` in both files as the absolute path on
the user's machine: the session runs inside the target repo, where nothing relative resolves.

**Why two files at Core already, when Core has only one phase.** The session carries on through
compaction (§4), and a compaction can re-attach an invoked body cut short (§5). The run text below is
already about 14,200 characters on its own — just over the cap — and Standard more than doubles it, so it
lives in a file that is read, and the invoked core stays a fraction of the cap. The split costs one
read at the start; growing the core later would cost a restructure, and a run text left in the core
would be cut without anything saying so.

**The core**, `<home>/skills/arc/SKILL.md`:

````markdown
---
name: arc
description: Run an arc — an ordered list of todos, each sealed as exactly one revision: get each open todo coded and reviewed, seal it, and stop for the human on design questions. Reads the plan on disk and carries on in this session. Use when the user says "run the arc" or "continue the arc", or names an arc directory.
argument-hint: "[arc directory] [--only <todo id>] [--from <todo id>]"
---

# `/arc`

`/arc` reads its argument and the plan's state on disk, picks a **phase**, and carries on in this
session by that phase's text. This file holds the routing and the rules every phase shares; each
phase's own rules are in its text.

**This file stays at most 14,000 characters (`wc -m`).** A compaction re-attaches an invoked skill
cut short; phase texts are read, never invoked, so no compaction cuts them. What grows goes into a
phase text, never here.

## Phases and their texts

- **run:** `<home>/skills/arc/RUN.md`

**A phase's first action is to read its text in full**, by the absolute path above — in
consecutive chunks to its last line if the read tool refuses one large read, a chunk that errors on
size retried with half the size, never a stop. Stop and ask only when the file is missing.

## Arguments

`/arc [arc directory] [--only <todo id>] [--from <todo id>]`. With no directory, find the most
recently modified directory under `<home>/plans/` holding a `README.md` with a `todos/` beside it,
and confirm it with the user. The flags pass through to the run phase.

**What this session already holds is a routing input**, since disk state cannot tell "this session
runs this arc" from "nobody does". `/arc <arc directory>` for an arc this session already runs
continues at its current step, never re-entering the run text's *Before the loop* reconciliation,
which would reset an `in-progress` todo whose coder is still live and spawn a second.

## Compaction

The session carries on through automatic compaction; one cuts wherever it fires, so the plan
directory is written current at every decision. **Do not wrap up early, hurry, or skip a step to
save context.** **After a compaction, re-read this file, the phase text you were following and the
arc's `README.md` before acting.** A step is taken from the text, never from the summary's
paraphrase of it.

## What every phase shares

Each is stated in full in `RUN.md`; these are pointers.

- **One todo, one revision**, and **every revision builds before it is sealed**: its
  `## Standing rules`.
- **The supervisor writes no code**: the same.
- **What the software should *mean* is the maintainer's**, asked, never guessed: its step (c).
````

**The run text**, `<home>/skills/arc/RUN.md` — no frontmatter, since nothing invokes it:

````markdown
# Running an arc

You are the **supervisor**. You own the plan, the version control, and the decision to interrupt the
human. During the loop you neither write code nor review it — subagents do both.

The point of this loop is that **your context stays small and flat**. Subagents do the expensive work
in their own disposable contexts and hand you back a paragraph. Guard that: never pull a build log, a
full diff, or a file's contents into your context when a subagent has already judged it.

## Before the loop

Read the plan directory's `README.md` — the one the core resolved and confirmed.

**Check the two gates before anything else.** Read the **Build gate** and **Test gate** lines out of
House rules. If either is missing, reads `UNSET`, or is still a literal placeholder — `<build gate>`,
`<test gate>`, or anything else left in angle brackets — **stop and ask the user for the exact
command. Run no todo until you have it.** A placeholder reaches the shell as a literal string and
fails as an unknown command, and every revision sealed after that is sealed on a gate that never ran,
which is the one guarantee this whole loop exists to make.

**Check that a session holding nothing but this directory could execute the plan.** Skim each
todo file. Stop and tell the user which todo is thin and what is missing if you find: a **Spec** that
refers to something said in conversation rather than written down; a todo with no "Done when" clause;
or a missing `decisions.md` on an arc whose todos clearly involved choices while planning.

Preflight the working copy:

- **git:** require a clean tree; record `git rev-parse --short HEAD` in the **Base revision** bullet.
- **jj:** `jj st`. If there are uncommitted changes you did not make, stop and ask — do not absorb
  someone's work into a revision. Then, if `@` has a description, run `jj new` so you start on an
  empty revision, and record the parent in **Base revision**.

**Read the two statuses** (`README.md`'s **Status** bullet, and the `### PR 1 — <title> — status:
<value>` heading in its index). A status names an activity; a stop for the user is no status.

- **The arc is still `planning`.** The plan has not been approved. Ask the user whether it is; on
  their yes, write the arc `underway` and the PR heading `running`, and go on. Never run an
  unapproved plan on the strength of having been invoked.
- **The PR is `planning` in an approved arc.** The user's findings were being written into todos
  (*After the loop*, below) when a session ended. Finish writing them with the user, write the
  heading `running`, and go on.
- **The PR is `accept`.** Its code is written; there is no loop to run. Tell the user it is waiting
  on their reading (*After the loop*, below), and stop.
- **The PR is `done`.** Say so, and stop.
- **The PR is `running`.** Go on.

**An open question is listed before any work is chosen.** A todo whose heading reads
`design-question` holds a question the user has not yet answered, in its own file. List every such
question to them, one line each, and take the answer into that file (step c) before choosing the
next todo — the loop picks up only `status: todo` and would otherwise skip it without a word.

**Resuming a PR at `running`:** reconcile any todo left at `status: in-progress` before choosing a
todo. The loop only picks up `status: todo`, so a stale `in-progress` todo would be silently skipped
forever.

- Look for a sealed revision since **Base revision** that is not recorded as any todo's **Revision**
  bullet. If exactly one exists and plausibly matches, the work was sealed and only the recording
  step never ran: mark that todo `done`, fill in its **Revision** bullet, and write a handover
  paragraph noting it was reconstructed after an interrupted session. Leave its checkboxes unticked
  with that note.
- If no such revision exists and the working copy is clean, the coder never handed over: reset the
  todo to `todo`.
- If no such revision exists and the working copy is dirty, the rule above already stops and asks.

**Starting a run sets no status.** Approval already wrote the arc `underway` and the PR `running`;
both stay as they are until the loop's exit.

Metadata — `README.md`'s header bullets and each todo file's own bullets — lives in bullet lists so
it renders readably. Preserve that shape on every rewrite; do not collapse the bullets back into bare
`Key: value` lines, which a markdown viewer runs together into one paragraph.

## The loop

For every todo with `status: todo`, in index order and whatever its id — honouring `--only` /
`--from`, which take a todo id such as `4` (at Standard also `fix1` or `test3.2`). **The loop exits
when the PR holds no open todo**: none of its todos is unsealed, a `deferred`, `dropped` or
`superseded` heading not counting as open.

### a. Coder

Spawn `arc-coder`. Its brief must contain, and nothing much else:

- the absolute plan directory path, and which todo file it is
- the todo's section **verbatim** — spec, files, "Done when"
- the **gates**, verbatim, and anything in `README.md`'s House rules
- the previous todo's handover paragraph, if there is one
- on a retry: the reviewer's findings verbatim

Set the todo's status to `in-progress` in its own file's heading.

The subagent inherits no history from you, so anything you leave out of the brief is invisible to it.
Restate rather than reference.

**Always spawn a fresh coder — never resume the previous one.** This holds on a retry, and it holds
for a one-line follow-up after the reviewer has already approved. Resuming re-bills the agent's whole
transcript on every message; three such rounds once cost ~620k tokens for two one-line edits. The
brief is what carries continuity: for a follow-up, restate the todo section, the gates, the diff that
already exists, and precisely the change wanted.

### b. Reviewer

Spawn `arc-reviewer`. Its brief carries: the plan directory path and which todo file it is, the todo
section and gates verbatim, and the coder's handover block — plus, on a repeat round, the reviewer's
own previous findings verbatim, so a fresh reviewer can tell ground its last `revise` already covered
from new ground.

The reviewer runs the build and test gate itself and reports it back in a `GATE:` line. **Do not run
the gate here, and do not review the diff yourself at this stage.** Build output and full diffs are
large, noisy and worthless to you; you run the gate once, in (d), to confirm.

**A live subagent sends nothing until it returns, so one that has gone quiet is not distinguishable
from a stalled one by silence alone.** Do not re-spawn on silence; elapsed time past roughly twenty
minutes is the signal.

### c. Branch on the verdict

- **`revise`** → back to (a) with the findings, in a **fresh** coder. **The progress judgement, not
  a fixed round count, decides when to stop:** a rework is progress when it opens ground the
  reviewer had not objected to before; it is a stall when the reviewer objects again to the same
  place the last rework specifically addressed, or when the coder's repairs start breaking as much
  as they fix even while new findings keep appearing — the second is the sharper test. A falling
  finding count is not the test either. **On a stall, stop** and escalate, naming the todo file and
  quoting both the findings from every round so far and what each fresh coder attempted.
- **`design-question`** → stop the loop. Set the todo's status to `design-question`, write the
  question into that same file, and put it to the user. Do not answer it yourself, and do not let a
  document in the repo answer it for you. Once answered, write the answer alongside the question as
  a **Design decision:** bullet in that todo's file before resuming the coder — only that file, not
  your own memory, reliably carries it across a resumed session. **A todo whose "Done when" names an
  experiment, and whose experiment came back negative, arrives here and not as a `revise`:** the
  coder did what the clause asked, the plan's premise is what failed, and a `revise` would send a
  fresh coder to repair something that is not broken while hiding from the maintainer the one fact
  the clause existed to discover. Where the answer changes the todo's spec enough that its summary
  bullet would change, it is also a replan: leave the trace `needs-replan` below describes.
- **`needs-replan`** → stop, and propose a plan amendment to the user. Small corrections to a later
  todo's spec you may make yourself; a changed approach needs agreement. The amendment is a replan:
  rewrite every changed todo's index entry and summary bullets in the same edit, and add one replan
  line to `decisions.md` — what changed in the planned revision set, and why. Never replan a sealed
  todo; a change to what it did is a new todo.
- **`blocked`** → stop and report. One case is worth retrying first: a reviewer blocked because the
  gate would not finish inside its own foreground limit. Run the gate yourself in the background form
  from (d); on `EXIT=0`, re-enter (b) with a fresh reviewer against the now-warm build, and on
  anything else stop and report.
- **`approve`** → continue to (d).

### d. Confirm, then seal

Run the gates once yourself, silenced:

```
chronic <build gate> && chronic <test gate>
```

`chronic` prints nothing when a command succeeds, so a passing gate costs you no context. Should it
print, the reviewer was wrong — treat that as a `revise` cycle and go back to (a) with the output. If
`chronic` is unavailable, use the quiet form of the build command and read at most the first few
error lines.

**A gate that outlasts the foreground limit needs the background form**, or this confirmation
silently stops happening on exactly the repos where it matters most:

```
{ chronic <build gate> && chronic <test gate>; echo "EXIT=$?"; } > "$TMP/gate-<todo>.log" 2>&1
```

run in the background. Seal only on `EXIT=0` in that log. No exit status is not a pass — it is a
reason to stop, the same as a failure.

Then seal exactly one revision:

- **git:** `git add -A && git commit -m "<description>"`
- **jj:** `jj describe -m "<description>"`, then `jj new`

**Description style: terse by default.** One `scope: summary` line is the norm. Add a body only when
the revision records a decision, a non-obvious trade-off, or a correction worth finding again — and
then say *why*, since the diff already says what.

### e. Record and report

Rewrite that todo's own file — and nothing else: set its heading to `status: done`, tick the **Done
when** checkboxes the reviewer confirmed as met, fill in the **Revision** bullet with the id from the
seal above, and write a **handover paragraph** — a few sentences the next coder needs to know: what
now exists, what it is called, anything surprising. This rewrite is the durable state of the run. It
is what a compaction leaves intact, so do it every time, before starting the next todo.
`README.md`'s index is a static list of links and titles — leave it alone here, except for the one
write below.

If this todo took more than one review cycle, also note in the handover how many and, briefly, what
each `revise` found. Skip this for the common case of first-cycle approval.

**When the loop exits with no open todo left, write the PR heading `accept`** — `status: running`
becomes `status: accept`, and nothing else in the index changes. The write belongs to the exit, not
to the last seal: a session cut off between the two resumes with the PR at `running` and nothing
open, exits at once, and writes it then.

Then print exactly one line to the user:

```
✓ 3/7  a1b2c3d4  model: wire insert/remove through the child index  (1 review cycle)
```

Where the exit just wrote `accept`, append one clause to that line: the PR is ready for them to read.

That line is the whole progress display. Do not maintain a todo list alongside the plan — a second
copy of the same state only drifts.

## After the loop

The PR is at `accept`: its code is written and is now the user's to read. Name the plan directory and
stop. **Only the user's word that it is finished moves it on**: then write the PR heading `done`, and
the arc's **Status** `done` with it. Findings instead send the PR back: its heading goes to `planning`
while each finding is written as a new todo under it, then to `running`, and the loop runs again.

## Standing rules

- **Never seal a revision on a subagent's word alone.** The confirmation in (d) is not optional; it
  is what makes "every revision builds" true rather than merely claimed.
- **Never edit the working copy yourself.** If a fix is needed, it goes through a coder cycle — a
  review finding you noticed personally included. You breaking scope discipline is worse than an
  agent doing it, because nothing reviews you.
- **One todo, one revision.** No amending a finished revision to slip in a later fix — that failure
  is precisely what this loop exists to prevent. New work is a new todo.
- **Seal a todo before starting the next one that touches the same file.** Steps (d) and (e) must
  both finish for the current todo before the next coder is spawned.
- **Environment repair is not supervisor work.** A dead dependency pin, a stale shell, a port fight,
  a browser that will not start — delegate it or hand it to the user with what you found, record it
  in the plan, and carry on with whatever does not depend on it. It is open-ended, it is unbounded,
  and it lands in the one context that cannot be thrown away.
- **Interrupt rarely, but do interrupt.** Mechanical problems are yours to solve. Questions about
  what the software should *mean* are the maintainer's, and guessing at one costs far more than
  asking.
- **Assume the run can be cut off at any point** — a compaction included. The reconciliation above
  and the durable per-todo records are what make an interruption safe to resume from; nothing else
  does.
- **The session carries on through compaction.** Write the plan directory current at every
  decision, never batched; do not wrap up early to save context; after a compaction, re-read the
  core, this file and the arc's `README.md` before the next step (the core's *Compaction*).
````

**Those two files are the one `arc` skill, now and after any upgrade.** If the user later takes
Standard, §10–§13 are spliced into the sections of `RUN.md` they name — step (b) gains a gatekeeper
spawn, step (c) gains the combination rule, the arc review replaces *After the loop* — §9 and §14
add `PLAN.md` and `TEST.md` beside it, and the core gains their rows in its routing (§8). The paths
never change. **Do not write `run-arc/`, `arc-standard/`, a second `RUN.md`, or a separate skill
holding "the rest of the loop".** One command, one run text. Where an optional layer needs rules
only some runs use, they go in a further file `RUN.md` reads on a stated trigger (§8, *Grouping*;
§18), which is a companion of this skill, not a second skill.

### What Core costs and what it buys

**Costs.** Two subagent spawns per todo (a floor of roughly 22k tokens each, plus the work), one
extra gate run per todo in the supervisor's confirmation, and the discipline of writing "Done when"
clauses that are actually testable. Planning a three-todo arc by hand takes ten minutes.

**Buys.** A history where every revision builds and is one idea. An independent check on every piece
of work before it is sealed. A run that survives its own session dying, and carries on through
compaction while the maintainer is away. A supervisor whose context does not grow with the size of
the job.

---

# PART II — STANDARD

*The full three-phase system. This is what most users should end up with.*

> **How to install Part II: merge, never add.** Almost everything from here to §13 is an **edit to
> the `<home>/skills/arc/RUN.md` you already wrote in §7** — the same file, at the same path, grown
> in place. The only two sections that produce a *new* file are **§9** (`PLAN.md`, the planning
> phase's text) and **§14** (`TEST.md`, the test phase's), both beside `RUN.md` in the same skill
> directory; §8 grows the core, `SKILL.md`, with the routing that picks between the three. The only
> agent file that gets *replaced* rather than added is `arc-reviewer.md`, in §10.
>
> **There is never a second run text, and never a second skill.** Writing one is the single most
> likely way to install this wrong, and it fails quietly: the harness loads whichever file matched,
> the two copies disagree about the gate, the verdicts and the Status vocabulary, and nothing flags
> the disagreement. Each section below says in its first lines which file it touches and whether it
> adds, edits or replaces. If a section does not say, it is an edit to `RUN.md`.

---

## 8. The three phases, carried by one session

```
planning phase  →  writes the plan directory, gets it approved     arc: planning → underway
  /arc <dir>                                                        PR:  planning → running
run phase       →  runs the per-todo loop, then the agents' read    PR:  running → accept
  /arc <dir>         of the PR, and its fix pass                         (the read happens at accept)
(maintainer tests it, beside the agents or after them)
test phase      →  turns the maintainer's findings into todos       PR:  accept → planning → running
  /arc <dir> <findings>  (then the run phase again)
(maintainer says it is finished)                                    PR:  accept → done; the arc
                                                                         done with its last PR
```

**One command; the phase is read off the disk.** The maintainer types `/arc <directory>`, and the
core (§7) infers the phase from the arc's state and from what they wrote; the phase word is needed
only where inference would differ. Each phase is a text the core has the session read — `PLAN.md`,
`RUN.md`, `TEST.md` — so **this section grows the core** with the routing below, and §9 and §14
write the other two texts.

**One session carries the phases.** A phase ends with its own activity — the plan's approval, a test
round written, a pull request's read closing — and whatever fires at that moment still fires there:
the handoff message, and where installed the watch-order check (§21). **It no longer ends the
session.** Where a phase used to print a line for the maintainer to paste into a fresh session, it
carries on in this session on their go, and offers the line only for resuming once this session is
gone. After approval the run follows; the run takes a test round's todos; in a grouped arc the next
pull request starts as soon as the previous one's read closes (*Grouping*, below). **Only what only
the maintainer can give waits for them**: their go after approval, a push, a `done`.

The rules that make one long session safe are §4's *Carrying on through compaction*, stated in the
core. Standard adds one, for planning: **before a plan directory exists, write each of the
maintainer's rulings and corrections, verbatim, to a scratch file as it is made** — in the session's
scratchpad, named by the planned arc's slug (`planning-rulings-<slug>.md`) so a post-compaction
re-read can find it — and §9's step 5 folds it into `decisions.md`. A compaction destroys the one
thing a planning conversation holds that nothing else does, the maintainer's own words; the scratch
file is not the durable write the pre-plan step forbids before approval (§9, step 4), since it is
not the plan directory.

> **A reversal, recorded.** An earlier revision of this file held *one phase, one session*: each
> phase in a session of its own, a copy-pasteable line at each seam, compaction "a safety net, never
> a mechanism", and a session that read its own context occupancy before every revision and packed
> up at about half the window. Its reasons were real — a summary paraphrases precisely the strings
> the loop needs verbatim, and cost is quadratic in session length — and §4's re-read rule answers
> the first. What changed is the price the maintainer would pay: every seam needed them present to
> start the next session, so an unattended run stopped wherever a seam fell. The occupancy rule had
> also never been seen to fire: the first run held to it peaked at 36.5% of a 1M window.

**What it costs.** Billed input is the sum of context size over every turn (§23), so a long context
is expensive:

> **Measured, over one 30-session corpus, before compaction carried sessions:** 61.5% of all
> cache-read came from turns beyond real turn 75; across ~20 project directories, 78.7%. Capping
> sessions near 75 turns would have been worth ~29% of total weighted spend.

With compaction, what bounds that cost is the **compaction window** — how full the context gets
before the harness compacts. A smaller window compacts sooner and keeps the context smaller, at the
price of one more re-read of the core, the phase text and the arc's `README.md` after each
compaction. That re-read is the floor every compaction pays:

> **Measured, by `wc -w`, in the originating system.** Splitting its one run skill into a core and
> phase texts with companions cut the words re-read at entry and after every compaction, for an
> ordinary agent-coded run in a jj repo, from 34,561 to 25,040 (−27.5%; 22,650 mid-loop). A run that
> also read the human coder's companion re-read 31,567 (−8.7%), and one that read the pull-request
> companion as well 36,218 (+4.8%). A session holding a run and a planning beside it re-reads both
> phase texts per compaction — about 94,000 tokens before the two READMEs. That floor is what a small
> window pays each time it fires.
>
> The baseline overstates the saving: it is a full re-read of the old skill, which the originating
> system had already stopped paying before the split: for some days its hook had re-read only
> sections. The real saving for agent-coded runs is smaller, and regular arcs roughly return
> to the earlier cost.

**Which window is best has not been measured.** §25 says so, and what is being done about it.

**Where the harness offers a hook on compaction, install one that names the re-read.** The core's
own *Compaction* section says to re-read, but the core is itself a re-attached body — and the first
dropped where several skills were invoked. A hook that runs right after a compaction and injects one
instruction removes that dependency. In Claude Code it is a `SessionStart` hook with the matcher
`compact` in `settings.json`, whose command prints a JSON object carrying the message in
`hookSpecificOutput.additionalContext`. The message, adapted to the user's paths:

```
Context was just compacted. Re-attached skill bodies may be cut short; what was cut is gone until
re-read. Before any further action, read in full, in consecutive chunks if one read is refused:
(1) the /arc core, <home>/skills/arc/SKILL.md; (2) for each arc you were acting on, the text of the
phase you were acting in, the companions that text's own triggers call for against the arc's
present state — derived from the text, not from what the summary says was read — and the arc's
README.md; (3) if you were planning before a plan directory existed, planning-rulings-<slug>.md in
this session's scratchpad; (4) where §24 Q6's forge read is installed and the arc records a
pull-request URL, run it before continuing. Name the arc and its phase in your first line. Re-read
nothing else. Then continue where the summary says you were, taking each step from the text you just read, never
from the summary's paraphrase.
```

> **Why "re-read nothing else".** A full re-read of the originating run skill after a compaction
> cost about 70,000 tokens. Naming the core, one phase text, the companions it triggers and the
> README is what the split above made cheap — so no rationale file and no phase the session was not
> in.

**Optional: a log of every compaction.** A second hook — `PostCompact` in Claude Code — can append
one record per compaction: the time, the session, the window in force, the context just before, and
the summary the harness wrote. It is what lets anyone count the slips that followed a compaction, or
check what a summary got wrong; nothing in the loop reads it. Offer it to a user who means to tune
the window.

> **A deferral, tried and dropped.** The originating system shipped a third hook that held an
> automatic compaction back while a coder was in flight, up to a safety margin, so that no
> compaction fell mid-todo. It was dropped at the review of the arc that introduced it, in favour of
> plain automatic compaction and the log above. Slips found right after a compaction that fell
> mid-todo are the evidence that would argue for bringing it back.

**Do not write a token formula.** A threshold in tokens per todo reads as precise and is invented. A
count of things that actually happened is a different kind of number and is fine.

### The routing the core gains

**What this touches:** the core, `SKILL.md`. Its `description` and `argument-hint` name all three
phases (§5 has the Standard shape), its phase list gains `plan:` `<home>/skills/arc/PLAN.md` and
`test:` `<home>/skills/arc/TEST.md`, and its *Arguments* is replaced by what follows. Everything here
is routing: no phase's own rule moves into the core, which is what keeps it under its cap.

`/arc [directory] [plan|run|test] [text] [flags]`:

- **A phase word** straight after the directory decides the phase, but is checked against the arc's
  state: where it does not fit, the first line names the call that does ("this arc is at `planning`;
  I assume you meant `/arc <dir> plan`"), or the session asks.
- **Other text is read for what it is**, after the arc and its state and after any phase word or
  flag: **findings** on a pull request that can be tested (§14) → the test phase; **an instruction
  about the arc** (continue, review, open the pull request) → the rows below; **a new arc's subject**
  → the planning phase; **a question** → answered from the arc's state, changing nothing. Text that
  fits two readings never starts a plan: ask, naming the candidates, "a new arc" among them. Text
  that reads as findings with nothing testable: say so, then ask what was meant.
- **Whenever the arc or the phase is inferred, not told, the first line names the call taken** ("I
  assume you meant `/arc <dir> run`") before acting, so the maintainer learns the commands.
- **A bare `/arc` reports and acts on nothing**: for each open arc whose `Repo` bullet names the
  working directory's repo, its Status and its PRs', the open questions, and the next call. With
  exactly one such arc, a flag or an instruction matching no arc (`/arc continue`) takes it; with
  several or none, ask, naming them.

**An arc's rows**, the first that holds, read off its Status, its PR headings (*The Status
vocabulary*, below) and each PR's last line in `review.md`'s review log (§11):

1. **Findings on a PR that can be tested, or the `test` word** → the test phase. The `test` word
   with nothing testable is a hard stop: say so and stop.
2. **Arc `planning`** → the planning phase: a first plan, a large replan, or a stalled challenge.
   **Arc `done` or `aborted`** → nothing to do; say so.
3. **A PR `running`, or at `accept` with its read open** — unless the PR being worked (*Grouping*,
   below) is at `planning`, which is row 4 → the run phase.
4. **The PR being worked at `planning`, or with none as in 3 any PR at `planning`:** its last log
   line says who was writing its todos. A fix pass (`triage closed — fix todos: …` or `fix todos:
   …`) → the run phase, writing any todo it names that is not on disk yet; a cut-off test round
   (`confirmed`, `triage closed — nothing to fix` or `reopened — test round <N>`) → the test phase,
   finishing that round; anything else → the planning phase, a replan.
5. **Every PR `done` or waiting on the maintainer** → the run phase, which hands it over or says the
   arc is finished.

**Starting a run sets no Status**: approval wrote the arc `underway` and each PR `running`. Where
journeys are installed, a journey directory first resolves to one arc (*Journeys*, below).

> **Why one command.** The originating system had three — one per phase — and each restated the
> routing for the cases where the maintainer had typed the wrong one, so the same table lived in
> three places, in three invoked bodies a compaction could cut. The maintainer had asked for a
> single command; the phase was already readable off the disk, so the router states the routing
> once and the three command names became phase names. The old commands were removed, with no
> aliases.

### Several arcs, one session

One session may hold arcs in different phases: the next arc's planning beside the current arc's run
(*Journeys*, below), or a test round on an earlier pull request beside a later one's run (*Grouping*,
below). Three rules, written into the core:

- **Every turn that acts on an arc names, in its first line, the arc and the phase it acts in**, so a
  compaction's summary can tell whose rules the turn followed.
- **A phase change reads the new phase's text before acting, and says the previous phase's rules do
  not apply to the turn.** Planning writes plan files and the test phase round todos; the run phase
  writes no code, and no todos beyond its own review-fix todos and the loop's index appends.
- **Planning and a test round interleave only where the run waits on subagents.** The run's seal,
  gate, review step and next spawn are never delayed for them; a subagent's result is handled first.
  The run takes a test round's todos at the pull request's next boundary, never mid-todo.

> **The ruling behind it:** *"You should be able to plan the next arc while running the current
> one."* A run's time goes to its subagents, and while the old rule handed back a line and told the
> maintainer to wait, their attention to the next arc had nowhere to go.

### The Status vocabulary

§3 gave the two levels and their values: a status names an **activity**, never who performs it; each
**pull request** carries one in its `### PR <k>` index heading, and the **arc** one in its `Status`
bullet. A stop for the maintainer is no value of its own: it leaves every status where it is, and the
open question is recorded where it was asked — a `design-question` todo's own file, an `ask` in
`review.md` (§13), or `challenge.md`'s `unresolved` disposition for a challenge stall (§16). What
Standard adds is **who writes each transition**, and the one thing a PR's heading cannot say on its
own.

**A pull request's transitions:**

- **`planning` → `running`**: approval (§9, step 7) writes `running` on every PR heading still at
  `planning`. For a re-entered arc it writes `running` on those and on each PR the replan gave new
  todos; every other PR keeps the value it held.
- **`running` → `accept`**: the loop's exit, when it leaves the PR no open todo (§4).
- **`accept` → `planning` → `running`**: whatever produces todos for the PR — the arc review's
  triage, a test round, a small replan — writes `planning` first, then `running` once the todos are
  written and, where the challenge phase is installed, checked. Producing none leaves the PR at
  `accept`. A small replan of a PR already at `running` leaves it `running`: only a PR at `accept`
  goes through `planning`.
- **`accept` → `done`**: **the maintainer said so**, normally at the merge. Nothing else moves a PR
  here — not a clean gate, not an agent's opinion, not another lens pass. **One exception, and only
  where watch orders (§21) and the observation store (§19) are both installed:** a meta-arc's PRs are
  closed by the session holding the store's role once its watch order's evidence is in, because for
  a meta-arc there is nothing for the maintainer to test (§21).

**`accept` alone does not mean the agents are finished.** The loop writes `accept` the moment the
code is written, *before* the agents read it (§11), because reading is what `accept` names. Whether
the agents' share of the reading — the **automatic read** — has closed is read off the PR's last
line in `review.md`'s **review log** (§11): it is closed only when that line reads `confirmed` or
`triage closed — nothing to fix`. A PR at `accept` whose read is closed is **waiting on the
maintainer**; one whose read is still open has work left for the agents. Every site that needs to tell
the two apart — routing, the handoff, the pull-request summary, the parked-watch-order scan and the
store's close where installed — reads the log this way and never reads bare `accept` as "the agents
are done".

**The arc's transitions:**

- **`planning` → `underway`**: approval, in the same write that sets its PRs `running`.
- **`underway` → `planning`, for a large replan** (§9, *Replanning*): whichever phase finds the
  replan large writes its mid-replan note first and moves the arc here; the PR headings keep their
  values; the arc goes back to the planning phase, whose approval returns it to `underway`. Never
  beside a live run on another pull request (*Grouping an arc into pull requests*, below).
- **`underway` → `done`**: written together with the last PR's `done`, by whoever writes that `done`.
  Where journeys are installed, this write may also ask whether the journey is over (*Journeys*,
  below).
- **`planning` → `aborted`**: the pre-challenge gate's answer was **abort** (§9). Nothing moves an
  arc out of `aborted` — it is the one terminal value besides `done` — and **the plan directory
  stays on disk, readable**: the rejected decomposition is the only record anywhere that one was
  proposed and what it said.

**The vocabulary is cyclic per pull request, not terminal**: findings — the agents' read, the
maintainer's testing, a colleague's review — send a PR back to `planning` and round again, while the
arc stays `underway`. **No skill converts a plan written under an older vocabulary**; a session that
meets one works it out, or asks (§19, *Constant improvement*).

> **Why the read is a log, not a status.** The originating system once kept two arc values for the
> read and its fix pass. Its rules sent a session resuming a finished fix pass back into the full
> review, fanning the lenses out again over a review that was done; a review with only notes had no
> rule moving it
> on at all, and one arc sat at the review value for weeks as a de facto "waiting on the maintainer's
> answers". Six challenge rounds then went into deriving the read's state from sections, seals and
> change-ids, before the maintainer cut it: *"the important bits to be precise about is not losing
> information (my input, findings, research, ...), and not following rules strictly."* So the log is
> one dated line per step, read by its last line as guidance — and a session that finds the log and
> the files disagreeing says so and works it out, or asks.

### The full plan directory layout

```
README.md                   header bullets, the todo index (each entry with its summary
                            bullets), Context, Story, Verification, House rules
todos/01-<slug>.md          one todo
todos/fix1-<slug>.md        review-fix todos, written by the run phase
todos/test1.1-<slug>.md     test-round todos, written by the test phase — or by a replan, when
                            a round is judged large
decisions.md                questions asked, alternatives rejected, and replans made, while
                            planning or running
challenge.md                challenge findings and disposition (Optional layer, §16) — and a
                            small replan's scoped pass, as a `## Round N (replan)`
review.md                   the review log, then each read's triaged findings, written by
                            the run phase (§11, §12)
research/<name>.md          background too long for a todo: triage reports, prior investigation
```

**Write nothing else into these files — each is owned by exactly one phase**, so it is never
ambiguous which one may add to it, with two listed exceptions, the last two bullets:

- **plan** writes `README.md`, `todos/01-…`, `decisions.md`, `challenge.md`, and `research/` for an
  original todo's background.
- **run** writes `review.md` and, if it finds anything fixable, `todos/fix1-…`, `fix2-…`, appending
  them to the index.
- **test** writes `todos/test<N>.<k>-…`, one `research/` file per triaged finding, and appends its
  round's questions, rejected alternatives and replan line to `decisions.md`. It also writes into
  `review.md`, which run owns: the maintainer's findings into a read's section while the agents'
  read is open, and a round's `reopened` line in the review log (§14).
- **a status** — the arc's `Status` bullet, a PR heading, and a review-log line — is written by the
  writer its own transition names (*The Status vocabulary*, above; §11 for the log), whichever phase
  that is.
- **a replan** (§9, *Replanning*), whichever phase runs it, may for that replan create or rewrite an
  **unsealed** todo file; add, drop, move or rewrite its index entry; append to `decisions.md` and
  `challenge.md`; mark a `decisions.md` or `research/` entry reversed in place; and, for a large
  replan, write the mid-replan note beneath the Status bullet and move Status to `planning`. The
  grant is listed, not general, and sealed todos stay outside it. It exists because a replan edits a
  plan already written, and the file's first writer is usually a session long ended: routing each
  edit back to it would mean no edit at all.

**Leaving placeholders for another phase's files only invites something to fill them in early.**

**Id schemes do not collide by construction.** Plain todos are `1.`, `2.`, …; review-fix todos are
`fix1`, `fix2`, … continuing from the highest already present rather than restarting; test-round
todos are `test<N>.<k>` — round `N`, todo `k` within it. Each test round gets its own `## Test round
N (PR <k>)` section in the index, naming the PR it tested — PR 1 in an arc of one PR. **Ids never
decide what runs:** the loop picks every open todo in index order (§7),
so a todo a replan adds under any id is picked up, and neither a test round nor a review-fix arc
needs a scoping rule of its own. The originating system once scoped a test round's run to that
round's ids, needed an exemption for the review-fix arc's, and could strand a todo written under any
other.

**`research/` holds what a todo should not have to carry.** Long background — a triage report, a
prior investigation, anything that would make every reader of a todo pay for material only one of
them needs. A todo's **Spec** links to it; an agent opens it only if the todo tells it to.

### Journeys: a body of work bigger than one arc

**Optional, though it lives here, because it is a shape of the plan directory.** It edits `PLAN.md`'s
step 2 and step 5, the core's routing (a journey directory resolves to one arc first), and the lens
briefs, and adds no file of its own. Offer it to a user whose work regularly outgrows one arc.
Everyone else can skip it: a single arc works exactly as §3 describes.

**The problem it solves.** An arc is sized by what one planning conversation can decompose and
research. Some work is bigger than that. Without a unit above the arc, it is either crammed into one
huge plan — the originating system had an 82-todo arc — or scattered across plan directories with
nothing binding them, so each new arc starts cold and re-asks what an earlier one already settled.

**The units, each bounded by its own property, never by a size.** This is the frame the originating
maintainer settled on, and the reason no rule here says "an arc is at most N todos":

- a **journey** is *describable* — one initial prompt sets it, at whatever length that takes, and its
  meaning does not change over its course; a day to several weeks;
- an **arc** is *plannable and researchable* — what the planning phase decomposes; discovering
  substantial new research is the signal that a new arc has begun; roughly a week;
- a **pull request** is *deliverable and reviewable* — usually one arc is one pull request, and an
  arc may be grouped into several (*Grouping an arc into pull requests*, below; §24 Q6);
- a **revision** is *sealable, buildable, codeable* — about an hour.

The durations are calibration, not limits. **A session is not one of the units.** The originating
system once counted a fifth, the session — *supervisable*, ended by a reading of its own context —
and retired it when compaction replaced the session (*One session carries the phases*, above).

**The layout. Once journeys are installed, every new plan is written nested**, a journey that will
only ever have one arc included:

```
journey-<slug>/
  README.md              the initial prompt verbatim, a Status of embarked | over, ## The arcs
  arcs/01-<slug>/        arc 1: a complete plan directory, with a `- **Journey:**` bullet linking up
  arcs/02-<slug>/        a later arc, the same
  decisions.md           decisions spanning arcs — from arc 2 on, opening with a pointer into arc 1's
  research/<name>.md     background spanning arcs — likewise from arc 2 on
```

A journey of one arc has no journey-level `decisions.md` or `research/`: its decisions go in arc 1's,
where the planning scratch file folds too (§9). When arc 2 is planned, the journey's own
`decisions.md` is written, opening with a pointer to arc 1's `decisions.md` and `research/` as the
record made while arc 1 was the whole journey, and arc 1 is not touched. In `## The arcs`, an entry
linking to its directory is a **planned** arc, and an entry without a link is **provisional**, a later
arc not yet planned. The journey's own Status is deliberately light: `embarked`, or `over` once the
maintainer says it is (*When is a journey over?*, below), with nothing hanging off either. Every arc
keeps the full vocabulary above. Pick a slug that no existing plan directory already uses.

> **Why nested from the start.** An earlier revision kept a one-arc journey flat — one directory that
> was journey and arc at once — and *promoted* it when a second arc was planned, moving arc 1 down
> into `arcs/01-<slug>/`. That move became unsafe the moment the next arc could be planned beside the
> current arc's run (*Planning beside a run*, below): it would move a running arc's directory and
> change its arc id while its workspace and observation files are named by it. **An arc's directory
> and arc id never change after they are written.** The price is one extra `README.md` per one-arc
> plan, and a longer arc id.

**Older shapes on disk are accepted and never migrated.** A `plans/arc-<slug>/` written before
journeys were installed is an arc belonging to no journey. A flat journey written under an earlier
revision — `todos/` beside the journey's own `README.md` — is an arc too, and is **never given a
second arc by a skill**: before any write for one, name it to the maintainer and ask whether they
nest it by hand or start the next arc as a new journey. Nothing is moved or renamed on their behalf.

**What makes a directory an arc:** a `README.md` with a `todos/` directory beside it. **Never** the
presence of a House rules section, because a nested journey's README carries House rules its arcs
share — a skill that searched for House rules would pick up the journey README as a plan to run.

**A journey's current arc is derived, never stored.** An arc is **settled for routing** — nothing
has more work to do on it unprompted — when it is `done` or `aborted`, or `underway` with every PR
`done` or **waiting on the maintainer** (`accept`, its automatic read closed: *The Status
vocabulary*, above). Everything else is unsettled: an arc at `planning`, and an `underway` arc with
any PR at `planning`, `running`, or `accept` with its read still open — work left to write, to code,
or for the agents to read. A nested journey's current arc is the first planned entry in `## The
arcs` that is not settled; a flat journey on disk has its one arc, unless settled. That a PR waiting
on the maintainer counts as settled is deliberate: an arc waiting on their testing, or parked on a
watch order (§21), must not block the next arc. This is routing only, and moves no status.

**A PR waiting on the maintainer beside a later `running` PR of the same arc** (where grouping is
installed, below) leaves the arc unsettled: it still has code to write, and it is the arc a journey
call acts on. Nothing waits for the maintainer at a PR boundary but a push and a `done`, so the run
carries on; the waiting PR is named in passing, with the line their findings for it go to.

**So the maintainer can name a journey instead of an arc.** `/arc <journey>` resolves to one arc
before *The routing the core gains*'s rows apply:

- **No current arc, or the current arc at `planning`** → the planning phase: the next arc, or the one
  at `planning` resumed — a challenge stall included, since a stall leaves the arc at `planning`
  (§16).
- **The current arc `underway`, with no PR at `planning`, and this session already supervising it**
  → the planning phase for the next arc, beside the run (*Planning beside a run*, below).
- **The current arc `underway` otherwise** → that arc, by the rows, named in the first line, with an
  offer to plan the next (none while a PR is at `planning`). **A journey never runs an arc still at
  `planning`** — naming a journey is a vaguer request than naming an arc, and running an unapproved
  plan on its strength would turn a routing convenience into an approval. A later arc at `planning`
  is named, never offered to run.
- **Findings with a journey** resolve to an arc with a PR that can be tested (§14), never to the
  current arc, which is usually a later one with nothing to test yet: list the candidates, newest
  first, and confirm — never pick one.
- Any other arc's PR waiting on the maintainer is named in passing, with the line their findings go
  to, and never blocks.
- With no directory at all, open arcs are found at both depths, by the test above.

**Planning beside a run.** The run is not stopped to plan — the originating maintainer: *"You should
be able to plan the next arc while running the current one."* Name the current arc's state in one
line, each PR and its value, then plan the **next** arc in this session (*Several arcs, one session*,
above). The next arc is a later arc already at `planning`, resumed rather than skipped; otherwise
the **first provisional entry of `## The arcs`, in list order** — its position in the list, never its
position relative to the current arc; with none, a new arc appended. Where several entries are
provisional, propose the first and ask. The next arc is planned on a tree the run is still changing,
which the challenge phase's fresh read of the base revision every round already covers (§16). The
one case that hands back instead: a PR of the current arc at `planning`, whose plan is not all
written yet.

> **Why list position, not "the one after the current arc".** In the originating journey a
> provisional arc sat above a later-numbered arc that ran first; "the entry after the current arc"
> would have skipped it. List order is the maintainer's order of intent.

**Later arcs do not start cold.** Before the pre-plan for arc N ≥ 2, the planner reads three things:
the journey `README.md`, the journey `decisions.md`, and the file listing of the journey's
`research/` — for arc 2, arc 1's own `decisions.md` and `research/`, since the journey has none yet.
It opens a finished arc's own directory only where one of those points into it. Anything the
journey's `decisions.md` already settles is not re-asked when the story is settled. A decision about
one arc's decomposition goes in that arc's `decisions.md`; one that spans arcs goes in the journey's.

**The journey README is the file read most, so it stays short.** Each time a next arc is planned,
every settled arc's entry in `## The arcs` shrinks to one line and its link. The arc's own directory
already holds the detail, so nothing is lost. Anything a later arc still needs moves to the journey's
`decisions.md` or `research/`, and is never deleted. In the originating maintainer's words,
*"compression and continuity are the same requirement here."*

**Wherever an arc's name becomes a file or workspace name, use its arc id.** Every journey has an
arc `01-…`, so a bare directory name would collide across journeys. The arc id is
`<journey slug>-<N>-<slug>` for a nested arc — the only form written now — and `<slug>` for an older
`plans/arc-<slug>/` or flat journey on disk, which the slug check above keeps from colliding.
Watch-order observation files (§21) and concurrency workspaces (§18) are the two places this bites.

**Lenses judging a nested arc are given the journey README's path as well** (§11, §16), since a lens
handed only the arc directory cannot know it exists. **One lens invocation covers one arc
directory**: a finding's `todo <N>` is ambiguous the moment two directories each number their own
todos.

**When is a journey over? When the maintainer says so.** `over` is their answer to a question,
never a computed state — the originating maintainer, on a rule that wrote it mechanically:
*"supervisor should still converse with human to decide whether the journey is over."* The question
is asked by whoever writes an arc's `done`, and only when that arc is **the last to reach `done`,
not the last-listed one**: arcs close out of order, because a meta-arc can wait on its watch order,
its PRs waiting on the maintainer, while a later one closes first. So the writer checks every other
entry at that moment, and asks only when every other entry is already `done` or `aborted` and none
is still provisional. It asks in the same reply as the `done` write — never deferred, and never
decided alone, by the store's role (§21) included. A yes writes the journey README's `over`; a no
leaves it `embarked`, and `/arc <journey>` plans the next arc. A flat journey on disk is asked the
same question when its arc reaches `done`: a yes writes nothing, since it has no second Status, and
a no is the case above — ask how its next arc is to be laid out.

> **What it costs.** In the originating system this grew the planning text by about 13% and the run
> text by about 6%, for an event — a second arc, an `over` — that is rare next to an ordinary run,
> and the routing is read on every invocation. That is the honest case for leaving it out until the
> user has work that needs it.

### Grouping an arc into pull requests

**Optional, and only for a user whose arcs reach review as pull requests (§24 Q6).** It lives here
because it is a shape of the index. Every arc already has a `### PR 1` heading (§3); an arc is
**grouped** when it has **more than one**. Grouping edits `PLAN.md`'s sizing step and index
template, `TEST.md`'s entry, the three arc-review lens briefs, `arc-triage`'s brief and two
challenge lenses, and it adds a **companion file** — `GROUPED.md` beside `RUN.md` — that the run
phase reads only for a grouped arc (*What it costs*, below). Everyone else skips it: an arc of one
PR runs exactly as written above.

**The problem it solves.** "One arc is one pull request" used to be absolute here. The originating
maintainer, when it was put to them: *"Usually, but often enough not, and we don't want to have a
sharp rule standing in the way of progress."* An arc too large to review as one, or one that has to
send an upstream fix ahead of the work that needs it, otherwise has to be either reviewed whole at
its very end or split across arcs that each re-plan what the other already settled. What stays is
the reason the rule existed: **inside a pull request the revisions are the review unit**, so an arc
splits only between pull requests, never mid-todo.

**The format.** A grouped index has several `### PR <k> — <title> — status: <value>` headings, each
followed by **one bullet naming why it is a PR of its own**, then its todos. (An arc of one PR has no
such bullet: there is no other PR to be distinguished from.) Todo numbers run on across groups, so a
todo's identity and the cheap board read (§3) do not change. Anything appended later goes under the
PR it concerns: review-fix todos at the end of the group whose review found them, and a test round's
section names its PR, `## Test round <N> (PR <k>)`.

**The test is two properties, not a list of reasons.** A group is a pull request when it is
**deliverable** (it leaves a production instance usable) and **reviewable** (a human or a lens can
hold it as one piece). The usual reasons — a subdivision that costs no extra work, an arc too large
to review, an upstream fix sent ahead — are examples, never a closed list: a real split in the
originating system was sent ahead for a fourth reason, a defect already in front of customers. The
value lens checks each group against the two properties, never against a list (§16, where
installed), and the sequence lens checks the build invariant per group: each group's last todo
leaves the tree deliverable, and no todo depends on a later group. **A split is decided at planning
time.** A prerequisite discovered mid-run is a replan (§9, *Replanning*) — a pull request sent
ahead inside the arc — not this rule's.

**"A PR holds an open todo"** is §4's definition, cited by every site that needs it (the loop's
exit, the handoff, every `done` write, the store's close, routing): any todo of its group not yet
sealed, counting the group's review-fix todos and its test rounds' todos as its own, and not
counting a `deferred`, `dropped` or `superseded` one. The test is open todos, never a PR's position:
the last PR can still hold a round, and an earlier one can hold one while the last runs.

**Each PR carries its own status** (§3), and each moves alone: a round on PR 1 sends PR 1 back to
`planning` and leaves PR 2's heading as it is. The arc's own `Status` stays `underway` throughout and
becomes `done` only with its last PR's `done`; until then the maintainer's *finished* is a go for the
PR in front of them, not the arc.

> **A reversal, recorded.** An earlier revision of this file kept **one** Status per arc, cycling
> once per PR, and rejected a per-PR status as two status facts per arc that could disagree. In use
> the single bullet was the misfit: every awkward routing case — an arc at `accept` with its next PR
> unstarted, a round on an earlier PR that had to "move no Status", a redesign parked until another
> PR's handoff — was several PRs' states squeezed into one value. The disagreement the rejection
> feared is avoided by giving the two levels different questions: a PR heading says what is being
> done to that PR's code, and the arc's bullet says only whether the plan is approved.

**Which PR a session works is stored, not derived**, in a `- **PR:** <k>` header bullet that the
run phase writes — **only in a grouped arc**; an arc of one PR has none and works PR 1. Approval
writes `running` on every PR at once, so with several PRs at `running` the headings cannot say which
one a session is coding, and a session resumed after a cut-off — or carrying on after a compaction,
whose summary is no record — cannot trust a pick it remembers. The maintainer: *"Store it and
additionally compare to github/forge status"* — the comparison is §24 Q6's forge read. The bullet is
**written once per PR boundary and at no other moment.** A boundary is a moment — a session's start,
or, inside a running session, the close of a PR's automatic read — at which the bullet is absent, or
the PR it names is `done` or **waiting on the maintainer** (*The Status vocabulary*, above).
Everything a session's start does at a boundary, the in-session boundary does too: the forge read,
this write, and recognising a test round, in that order. Bare `accept` is not a boundary: the loop
writes it before the agents read, so a session resumed between a PR's last seal and its review would
otherwise move on and never review that PR. The value written is the first of: **the earliest
earlier PR at `running` with an open test round**, so an externally reviewed PR is repaired before
more is stacked on it; then **the first PR at `running` that holds an open todo**. Each skips a PR the
forge read found merged or closed; an open round on such a PR is put to the maintainer instead. The
originating maintainer's ruling, kept verbatim because a paraphrase of it was wrong once: *"Fix at PR
boundaries."* A session resumed mid-PR never changes PR.

What the bullet scopes:

- **The loop works one PR**, picking only from its group, its review-fix todos and its open round,
  and exits when that PR holds no open todo, writing its heading `accept` (§4).
- **Where the human coding seat is installed (§18), a new PR states who codes it, and asks
  nothing**: the per-todo default is written over that PR's todos only, keyed on the arc's existing
  human-coder bullet, and said in one line. A later PR's todos get their default at its own start.
- **A PR's end does not stop the run.** The session runs that PR's read, hands it over once the
  read is closed — the PR now waits on the maintainer — and, where a later PR is planned, starts
  that PR's todos straight away: the read's close is a boundary, above. What still waits for the
  maintainer's word is what only they can give, a push of the earlier PR and its `done`. The
  originating maintainer, replacing a rule that ended the session here: *"Carry straight on."*
- **The review reads that PR's diff, not the arc's.** *PR k's tip* is the head of the revisions its
  own todos' **Revision** bullets name — `heads(<rev1>|<rev2>|…)` in jj,
  `git rev-list --topo-order -n1 <rev1> <rev2> …` in git — derived whenever read, never stored. The
  last todo sealed is not it: a test round's todos sit later in the index than the group they
  repair, and where detours are folded at seal time (§25, *History hygiene*), a folded todo's bullet
  names a lower revision and is left out. The *PR base* is the arc's base for PR 1 and PR k−1's tip
  after that, and the diff is `jj diff --from <PR base> --to <PR k's tip>` (git:
  `git diff <PR base>..<PR k's tip>`) — never `--to @-`, which stops being PR k's tip once a later
  PR sits above it. A test round of PR k reads from **its own Round base**, which the test phase
  records under the round's own heading (§14), so two rounds open on different PRs never overwrite
  one value; which round a read is of is named by the `reopened — test round <N>` line that opened
  it (§11).
- **The lens and triage briefs carry the PR's diff command and its number.** The *what* lens holds
  the diff to that PR's own todos — a later PR's todos still being open is the plan's order, not a
  finding — and the *why* lens judges what *this* diff contributes to the **Value** bullet, not what
  the other pull requests still owe.
- **The handoff opens "PR k of n landed"**, gives the line the maintainer's findings on PR k go to
  (`/arc <dir> <findings>`), and, while a later PR holds an open todo, says it starts now in this
  session — starting it does not wait on PR k's external review. A line to start the run is given
  only for resuming it in a session of its own.

**A round on an earlier PR.** A colleague's review of PR 1 can arrive while PR 2 is being built.
PR 1's heading reads `accept` — it was handed off, since the run phase moves the bullet off a PR
only at a boundary — so the test phase takes it by the ordinary rule (§14), deciding which PR from a
URL or number in the findings matched against the recorded pull requests, and asking once where
nothing matches; the triage briefs get that PR's diff. **It moves only that PR's heading** —
`planning`, then `running` — and never the live PR's, and it says the round is taken at the next PR
boundary, never mid-todo — by this session where it is the one running the arc (*Several arcs, one
session*, above). That boundary then codes the round **inside PR 1**, below PR 2, where the reviewer
will see it:

- **Where the maintainer's own working tree is live** — a second workspace of theirs, or an
  un-drained spike (§18) — placing the fix would rebase it, so **the round defaults to deferred**
  and nothing waits for an answer: the boundary's report line says the round was put off and why,
  names how their tree recovers (`jj workspace update-stale` in jj), and that their `hands up:`
  takes it at the next boundary. Put off is not the todo status `deferred`: the round's todos stay
  open, and the next boundary finds them again.
- **jj: place the working revision directly above PR k's tip before the coder starts** —
  `jj new -A <PR k's tip>` — so the coder edits and the gate builds the tree the fix will live in,
  and jj rebases the later PRs' revisions above it. **Place once per todo**, on its first entry to
  step (a); a retry after `revise` keeps editing the revision it has. Measured: a second
  `jj new -A <tip>` over a revision that already holds edits slides an empty revision *beneath*
  them, and the seal then describes the empty one. **After every seal of the round, move `@` back to
  the stack's top** with `jj new <the stack's top>` — never a bare `jj new`, which would branch a
  second head off a mid-stack revision. Conflicts from the rebase land on the later PRs' revisions,
  not on the fix, and are resolved lowest first as any cascade is (§24 Q1).
- **git: the fix is appended at the top**, above the later PR, and the handoff says so.
- The round's loop exit writes PR k `accept` again, and its read runs from its own Round base to
  PR k's new tip, so the diff is the fix and none of the later PRs; its review-fix todos are placed
  the same way. Its handoff opens **"PR k, round N, landed"** and offers, on the maintainer's yes, to
  move PR k's bookmark to its new tip and push it — an append on top of what the reviewer saw, and
  one of the three exceptions to *no outward-facing act* (§24 Q6). The later PRs were rebased above
  the fix and are now rewritten commits, so **pushing them waits for the maintainer's go**, and the
  handoff names how to give it.

**The pull-request summary for a stack** (§24 Q6), where the summary is part of the install:

- **The gate is the PR's own read, not the arc's `done`.** PR k's summary may be produced once PR
  k's automatic read is closed — the PR waiting on the maintainer — and they ask. A lower PR goes out
  for review while the next one is worked on; waiting for `done` would hold it until the whole arc is
  finished.
- **One bookmark, one body, one draft pull request per PR**, named from its group's title. Each body
  names its place — *PR k of n*, and the PR below it — and `Closes #<n>` goes on the last PR only: a
  lower PR that closed the issue would close it before the work is done. PR k > 1 is opened with
  `--base <PR k−1's bookmark>`; PR 1's base is wherever the arc's base sits (§24 Q6). The offer is
  for the whole stack at once, pushed bottom first, and never touches a bookmark outside it.
- **Where the forge has native stacked pull requests, register the chain after creating it** —
  on GitHub, `gh stack link <bookmark 1> <bookmark 2> …` from the `gh-stack` extension. It takes no
  title, body or draft flag, so it never replaces `gh pr create`. **Neither form was run on a real
  stack in the originating system** (the extension was not installed and the feature was in public
  preview); report what the command printed, not what it was expected to do. Where it is missing,
  the plain `--base` chain stands.
- **After the lower PR merges**, the upper PRs re-target only if its head branch is deleted, and a
  squash-merge leaves the old commits under the upper stack, to be rebased onto the new trunk and
  the leftovers abandoned. That sequence is reasoning from the forge's documentation, untried.

**What it costs.** In the originating system the arc that introduced grouping grew the run text by
about 40,000 characters, roughly 80% of it conditional on the arc having more than one PR, threaded
through paragraphs a one-PR run applies — so every ordinary run carried text it had to recognise as
inapplicable. **That is why it lives in an on-demand companion file**, read right after `README.md`
whenever the index has **more than one** `### PR` heading, or a `## Test round <N> (PR <k>)` section
naming a PR other than the one being worked — and never otherwise. Since every index now carries a
`### PR 1`, a trigger of "any PR heading" would fire on every arc and save nothing. Reference the file
the way §5 says any companion file is referenced, by the skill's own directory, in prose. **If that
read fails, stop and ask**: running a grouped arc without it does not fail loudly, it works every
PR's todos as one list, reviews the cumulative diff and hands off as one pull request.

> **Honest status.** At extraction time no grouped arc had yet run under these rules. They were
> written from one stack the originating maintainer had run by hand, and rewritten once since for
> per-PR statuses; their watch order (§21) was still unanswered.

---

## 9. The planning phase: `PLAN.md`

**This section adds a new file.** Write it to `<home>/skills/arc/PLAN.md`, beside `RUN.md` — a phase
text of the one `arc` skill, not a second skill, so it has no frontmatter: the core's phase list
names it (§8, *The routing the core gains*) and the session reads it when routing lands on the
planning phase. (§10–§13 are the opposite case: edits to `RUN.md` itself.) What follows is the
procedure; write it in this shape, opening with §5's convention line.

### Step 1 — Detect the repo contract

Run these and read what you find. **Do not guess any of it.**

```
ls -a                          # .jj? .git? AGENTS.md? CLAUDE.md? CONTRIBUTING.md?
```

Read the repo's agent-instruction files in full — they are where a maintainer records the rules that
are not derivable from the code.

**First, check that those rules actually load.** A harness reads the instruction file it was built
to read, and no other — Claude Code, for example, reads `CLAUDE.md`, not `AGENTS.md`. A repo whose
conventions sit in a file the harness does not read is one where every convention the maintainer
wrote down is invisible to you and to the subagents. If you find that, say so and offer the one-line
fix — in Claude Code, a `CLAUDE.md` containing `@AGENTS.md` — **before planning anything.** It is
worth more than the arc itself, and it is what lets House rules stay empty.

Then establish:

- **VCS.** `.jj` present → jj (check `jj st` works). Otherwise git.
- **Build gate.** The command that must pass before any revision is sealed. Get this exactly right;
  the whole loop's guarantee rests on it. **Watch for build commands that silently skip parts of the
  project** — test suites, examples and benchmarks are common omissions. (In the Haskell world,
  `cabal build all` skips test suites; `--enable-tests` is what covers them.)
- **Test gate.** The command that runs the tests, or an explicit statement that there are none.
- **The gates must carry an identical flag set.** Derive the flags from whatever CI actually
  enforces, then write the *same* set into both commands.

> **Why identical flags.** Where the build tool hashes its flags into the build plan — cabal's
> `--ghc-options`, cargo features, `go build` tags — a flag one gate passes and the other omits makes
> each gate invalidate the other's cached build, so every run recompiles from scratch. Two gates
> differing by one flag is a bug, not a detail.

### Step 2 — Read the backlog, don't consume it

If the repo has a `todo/` directory, an issue tracker, or a README shortlist, read it to choose and
order the work. **It is a source, not a queue** — do not restructure it, add status fields to it, or
treat the arc as owning it. Respect dependency order it encodes.

**From a GitHub issue:** "work on issue 123" means **one issue is one arc**, decomposed into several
one-revision todos. `gh issue view 123 --json number,title,body,labels,comments`. Read the comments
— corrections and scope changes usually live there rather than in the body, and the newest comment
often supersedes the original description. Record `Issue: #123` in the header so the PR summary can
carry `Closes #123`. The issue is the *subject*, not the spec: its wording is rarely testable, so
still write your own "Done when" clauses.

### Step 3 — Size the todos

Per §3 above, which is the same rule set. Do not restate it in two places.

### Step 4 — Write the pre-plan and get it approved

Before writing anything else, put a **pre-plan** in front of the maintainer. It proves one thing —
that the ask was understood — and it does nothing else. It holds exactly two things:

- a **restatement of the requirements** in your own words: what the maintainer wants, what changes
  for them, and — where the ask is ambiguous — what you are reading as out of scope;
- a list of **what you will research** before the real plan is written, as open questions and areas
  to read, **never as answers**.

Nothing else goes in it. No todo list — the decomposition is step 5's, and the maintainer first sees
it at that step's own pre-challenge gate. No context prose. No gates. No value bullet. No file
manifest. Each has a place of its own further down.

**Under a minute for a simple arc, five at the outside. The budget is the point, not a courtesy.** A
pre-plan that takes longer has started planning, and a misread ask is visible in the first paragraph
or not at all.

> **Why not skip it.** The two errors do not cost the same. A wrong restatement costs one message. A
> wrong plan costs the whole phase.

Then get it approved and **write nothing durable until it is**. If the harness has a plan mode that
permits exactly one file, the pre-plan is what that file holds — and leaving plan mode is both the
approval gesture and the only way out of the restriction. Outside plan mode, say the pre-plan in one
message and wait for a nod.

**What the maintainer rules or corrects before the directory exists is still written down as it is
made**: verbatim, to the planning scratch file §8 names (*One session carries the phases*), outside
the plan directory. Step 5 folds it into `decisions.md`.

### Step 5 — Write the plan directory

With the pre-plan approved and its research run, **settle the story first**, then write the directory
in the layout of §8, in this same session — pre-planning, writing and challenging are one phase.
**Fold the planning scratch file into `decisions.md` as part of that write**, each ruling and
correction verbatim as a question-and-answer line (§3); a ruling that decided nothing leaves no line,
and the scratch file has then done its work.

#### Settle the story before writing todos

The research has run and no todo exists yet. That boundary is where this step sits: put your own
model of the arc in front of the maintainer, in one message, and wait before decomposing anything.

**State the model, never ask for it.** Open with a draft `## Story` and the working definitions of
the arc's central terms, **written as assertions** — *"I am taking `open` to mean the work still
outstanding; `total = open + settled`"* — and ask **what here is wrong**, not *is anything wrong*.

> **The record is unambiguous about which of the two gets answered.** A maintainer handed a written
> model read it and answered item by item. The same maintainer handed an evidence-free question gave
> a first answer that had to be re-asked with the evidence laid out before it settled. **A session
> that asks an open question is asking the human to guess which of its own unstated beliefs is
> wrong.**

**What it is for, narrowly: shape-changing facts and term definitions.** It is **not** for
front-loading domain questions in general — an arc may carry an open domain question all the way to
approval at no cost. The dividing line is *"would this change how the work is decomposed"*, never
*"is this a domain question"*.

**Put it as one ordinary message that waits.** Not a structured multiple-choice prompt: what comes
back is a free-text correction to a statement, and a draft story does not fit in an option label.

**What it produces is `README.md`'s `## Story`:** the arc as a terse story — every key decision,
feature and change, and **no details**. It is not `## Context`, which is the arc-level *why*; the
story is *what will happen*. It is equally not a restatement of the todo index: it is organised by
what the arc changes for whoever uses the thing, and it is written **before any todo exists**, so a
story with one bullet per todo is proof the conversation happened too late.

**An empty answer still produces the `## Story`.** If the maintainer has nothing to correct, the step
has cost one message — so it is never skipped on the grounds that they usually have nothing to say.

**It lives in `README.md` because that is where the readers already are.** Every review and challenge
lens is told to read `README.md` in full, so a section there reaches all of them without editing a
single agent brief; a `story.md` of its own would reach none of them until every brief was changed.

> **What skipping it cost once.** Several rounds of one arc went on migration-number churn before the
> maintainer supplied *"at most one migration per document over the whole stack"* — a release-process
> fact they had held all along, which no amount of reading the code would have produced.

#### Write the Value bullet as a claim, not a hope

The arc review holds the finished code against this one sentence, so make it falsifiable.

- Worthless: *"Improves the metrics handling."* It cannot be checked.
- Good: *"A field worker sees the consumption figure on the building card without opening the edit
  drawer."* It can be, and will be.

If the honest answer is that no user notices this arc, write `enabling work:` and say what it
unblocks — that is a perfectly good arc. A plan that oversells refactoring as user value produces a
finding later, at the arc review, when the claim is held against the finished code and found empty.

#### The pre-challenge gate

With the plan directory written and step 6's fan-out **not yet spawned**, show the maintainer the
index summary: `README.md`'s `## Todos` list — link, title and summary bullets, exactly as it
stands. **Print it in the message itself.** Do not point at the file and invite them to go read it;
the whole ask this gate answers is that they not have to open anything to decide direction.

**It waits.** Other questions in this design proceed when no answer comes, because guessing wrong
there costs one correction at the next boundary. This one is not that: it exists for no reason but
to be an abort point, and an abort point that proceeds on silence has stopped being one. So it holds
until one of three answers comes back, each with its own consequence:

- **abort.** The arc stops before the challenge phase spends a single fan-out. Set **Status** to
  `aborted` (§8). **Keep the plan directory — never delete it.** An abort here
  is this gate's strongest success case; deleting the directory would leave nothing anywhere
  recording that a decomposition was proposed and rejected, or what it said.
- **redirect.** Rewrite the decomposition — the same kind of edit step 6's rework makes, directly in
  the plan directory, with no coder and no gate — and show the gate again against the rewrite. Edit
  the directory in place rather than deleting and rebuilding it. **Preface the second showing with
  one line per objection they raised:** the objection, and either the todo that now answers it, by
  its new number, or a statement that it was not acted on and why. A rewrite renumbers todos, so
  without those lines they would have to re-derive from scratch whether each objection was answered
  — re-reading, at the one gate built to stop them re-reading.
- **nod.** Step 6's fan-out spawns as planned.

> **What it costs and what it buys.** One message and a wait, against a phase that spends up to nine
> agents per round. It takes the risk of an unexamined premise reaching the maintainer cold at step
> 7 off that step; it does not take the risk of a challenge-driven rework off it, which is what step
> 7 still has to catch.

**Without the challenge phase the gate collapses into step 7.** Nothing then runs between the two to
change the decomposition, so there is one showing, not two — but keep the gate's three answers,
`aborted` included: an arc the maintainer rejects at the only review point still needs somewhere to
land.

### Step 6 — Challenge the plan

**Optional layer.** See §16. If the user is not taking it, go straight to step 7.

### Step 7 — Get the finished plan approved

Show the **same index summary the pre-challenge gate already showed** — the same artefact, not a
second list — along with the gates, and get agreement **before any code runs**. This is the one
review point in the whole design where the human sees everything at once, and it is cheap here and
expensive later.

**What this step is for once the gate exists.** The challenge runs entirely *between* the gate and
here, and it can still change what they approved there: a `fix` rewrites wording or scope, a finding
is dismissed on the planner's own judgement, a todo is split, dropped or exited. So what reaches them
here is the gate's decomposition **after** that happened, held against the *What the challenge
disputed* block (§16) so they can see what moved and why.

Ask about anything genuinely ambiguous now. Once the run starts, the only interruptions should be
real design questions.

If the arc changes what a user sees or does, mention that the review phase will want the application
already running, and let the user start it now. **In a project where a cold start syncs a database
for a quarter of an hour, being told at review time is being told too late.**

When approved, set the statuses. Approval writes the arc `underway` and, in the same write,
`running` on every PR heading still at `planning` (§8, *The Status vocabulary*). For an arc this
session **re-entered** (*When the planning phase re-enters an arc that holds work*, below), the PR
headings already hold their own values, so approval writes `running` on every heading still at
`planning` — an arc stopped at a challenge stall has no other — and also on each PR the replan gave
new todos; every other PR keeps the value it held, since writing `running` over a
PR that is `done`, waiting on the maintainer, or untouched would reopen finished reading or restart
a PR that is not.

**A PR given new todos that already holds a line in `review.md`'s review log** (§11) gets one more
line first, `reopened — replan`, whether its read was closed or still open, so the new code is read
once its todos are done and no old triage is resumed over code the replan replaced. Two cases need
none: a PR with no log line, which has no read begun; and a PR whose last line is **already** a
`reopened` line — a test round's redesign logs its own (§14), and a second line would widen the next
read from the round's diff to the whole PR.

Then remove the mid-replan note, which is spent, and **offer to carry on into the run phase in this
session**. On the maintainer's go, the core's routing sends the approved arc to the run phase, whose
text the session reads afresh before acting (§8, *Several arcs, one session*). Give the line below
only for resuming once this session is gone:

```
/arc <absolute plan directory path>
```

Approval is the one seam that waits: it is the maintainer's go, and nothing else can give it.

### Replanning

**What this touches:** `PLAN.md` gains this subsection, and every other site that replans
cites it instead of restating it — `RUN.md`'s `needs-replan` and `design-question` branches (§4,
§7), its review-fix arc (§12), every test round (§14), and the human coder's *this todo is wrong*
(§18, where installed). At Core, §4's `needs-replan` carries the trace alone; this adds the sizing
and the checks.

**What it is — a characterisation, not a trigger.** Any change to the planned revision set: a todo
added, dropped, deferred, moved, or changed so much that its summary bullet would change. It happens
in the small and in the large — at the todo; at the pull-request split (§8), where a prerequisite
found mid-run is a pull request sent ahead inside the arc; at the arc; and between a journey's arcs
(§8), where material moves to the next provisional arc. The originating maintainer was explicit that
this is a description and not a test: *"understand my definition more like a characterization than a
sharp rule. 'any change to the planned revision set' is also a good description."* A checklist of
triggers would be read as the set to check, and a change that fit no item would go unplanned — so
never sharpen it into one. The phases were already replanning without the word: a `needs-replan`, a
`design-question` answer that reshapes a spec, a review-fix arc, a test round's todos, any todo
deferred. Each is an instance.

**Sealed work is never replanned.** A change to what a sealed revision did is a new todo (§2,
invariant 1). A replan edits only unsealed todos.

**The herd scales with the replan.** Two ends, and the supervisor judges between them. There is no
threshold, because the ruling this rests on asked for none: *"It's the supervisor's responsibility to
pick the adequate herd of lenses, researchers … to do this, according to the size of the replan."*

- **Small** — a spec corrected, a todo added, dropped or rewritten, the arc's approach unchanged.
  Where the challenge phase (§16) is installed, spawn **one scoped challenge pass** before any coder
  is handed an added or rewritten todo. **Default lens: `arc-challenge-research`** — a mid-run todo
  nearly always asserts something about the tree it will be coded against, and the session that
  wrote it, reasoning from its own paraphrase, is the worst judge of where that premise is wrong. Add
  `arc-challenge-sequence` where the change touches order or what an earlier todo leaves behind, and
  `arc-challenge-value` where it touches what a user gets. **Scope**, stated in the brief: the todos
  the replan wrote or rewrote, **plus, for an added or moved todo, every unsealed todo after it**,
  checked only for still being satisfiable on the tree the change leaves; every sealed todo is named
  in the brief as built and context-only, never challenged. **The pass is a single fan-out**: act on
  its `fix` findings directly, with no re-check and no progress judgement, and take what only the
  maintainer can rule to them. **A todo the maintainer will code themselves skips the pass** (§18,
  where installed) unless they ask for it: they co-wrote the amendment and read it as they code, and
  waiting on a lens costs the flow §18 exists to protect. Without the challenge phase there is no
  pass at either end; the trace below still applies. **A PR at `running` stays `running` through a
  small replan** — its todos are being written beside code still to be written, and no read waits on
  them. Only a small replan that adds todos to a PR at `accept` moves it, to `planning` and then
  `running`, logging `reopened — replan` first as step 7 has it.
- **Large** — a changed approach, a re-split arc, a dropped pull request. Whichever phase finds it
  first writes a **mid-replan note**: what the maintainer agreed, what the replan must still do,
  which PRs it touches and the status each holds, the working copy's state (kept or discarded — a
  kept one the resumed run will ask about), and the findings and `research/` reports it must answer,
  by path; plus a question-and-answer line in `decisions.md`, since what agreed it was a
  conversation, and a compaction keeps only a summary of it. The note is an **indented sub-bullet
  beneath the Status bullet — never on the `- **Status:**` line itself, and never in a PR heading**:
  anything that reads a status by matching those lines — a status-line script, a grep for `running`
  — would read free text there as a live value. Then the arc's Status goes to `planning` — its PR
  headings keep their values — and the arc returns to the planning phase (*When the planning phase
  re-enters an arc that holds work*, below), which reworks the unsealed todos and runs its whole
  challenge phase over them, progress judgement included. A session in the run phase that agreed a
  large replan carries on into the planning phase itself, on the maintainer's go, reading `PLAN.md`
  before it acts — no new session, no line to paste. **A large replan never starts beside a live run
  on another pull request of a grouped arc** (§8): a live run is not rewound, and planning under it
  would touch that pull request's todos and tree, so it waits for that pull request's handoff.

> **Why the small end is one fan-out with no re-check.** Every review-fix arc and every test round
> is a small replan, and holding each to the challenge phase's progress judgement would run the
> rework-then-re-check loop that the originating system had recorded as barely progressing. A small
> replan does not earn the whole phase; a changed approach does. **Why the small end has a pass at
> all:** a real run shipped a review-fix todo, written by the supervisor, claiming that a revision
> expression denoted one parent when it denoted another — the supervisor had paraphrased
> `decisions.md` into a conclusion the maintainer never gave, and only a coder who reasoned from
> first principles caught it.

**A replan leaves a trace**, whichever phase runs it, at either end:

- every changed todo's index entry and summary bullets updated in the same edit (§3);
- one **replan line** in `decisions.md` (§3): what changed in the revision set, and why;
- every `decisions.md` or `research/` entry the replan reverses **marked reversed in place** — coders
  are sent to `decisions.md` as the authority, so a stale line steers them against the redesign;
- every `## Verification` clause a dropped or deferred todo owned given a **named new owner in some
  todo's "Done when"**, not only a re-pointed `README.md` sentence — a clause no todo owns is a
  promise the arc review will grade and nobody was asked to keep;
- where the challenge phase is installed, a small replan's pass recorded in `challenge.md` as a
  `## Round N (replan)` (§16).

> **Where two of those came from.** A deferred todo had owned the arc's only demonstration that a new
> test fails without the fix; the rework re-pointed `README.md`'s Verification sentence and wrote the
> clause into no "Done when", so no coder would have performed it and the suite would have passed
> vacuously. And after a redesign, a `decisions.md` line still recorded the reversed choice as
> rejected — every lens caught it a round later, but coders are sent to that file as the authority.

> **Honest status.** These rules were written at the end of the arc that defined them, from the
> failures quoted above. At extraction time they had run once, in that same arc's review-fix pass:
> one research-lens fan-out over four fix todos found six gaps in the supervisor's own todo text — a
> search pattern missing five dependent sites, a deletion that would have broken a registered citation
> — none blocking; they were corrected without a re-check, and the corrected todos then passed their
> per-todo reviews. No later arc has yet run under them.

### When the planning phase re-enters an arc that holds work

An arc **holds work** when its directory already has a `challenge.md` with a `## Round N`, or a
filled **Base revision** bullet — read off the disk, never from a stored flag. A large replan comes
back to the planning phase this way, whether the maintainer names the arc directory or, where
journeys are installed, routing reaches it (§8). Without this path the procedure above reads end to
end as *author a new arc*: in the originating system, a session handed such an arc had no rule for
whether a recorded note was work to finish or to re-derive, nor for what number its next challenge
round took.

**Read in this order, and stop where the pointers stop** — such a directory runs large; one measured
about 300 KB, two-thirds of it `challenge.md`:

1. `README.md`'s header bullets and `## Todos` index, then the board (§3).
2. `decisions.md`.
3. Only the newest `## Round N` of `challenge.md`. Older rounds open only where that round or
   `decisions.md` points into them, with one exception: the running total of fan-outs (§16, *e.*)
   continues from the last cost line that is not a `(replan)` round's, since a `(replan)` round
   states its own fan-outs only.
4. `research/` only where a todo or the mid-replan note links it.

- **A challenge stall still awaiting the maintainer comes before anything else** (§16, where
  installed). A stall leaves the arc at `planning`, so an arc stopped there is simply re-entered
  here, with the stall recorded only in `challenge.md`: a finding whose disposition is
  `unresolved — …` with no `→ … (maintainer)` transition after it. Put each such finding to the
  maintainer as §16's progress judgement has it, and act on the answer — restart at step 6's
  fan-out, or `unresolved → accepted (maintainer)` and on to step 7 — rather than resuming past it.
  Search all of `challenge.md` for them, not only its newest round, since a transition is recorded
  in place.
- **A mid-replan note is finished, not re-derived.** Where the Status bullet carries one, or
  `decisions.md` already says what the replan must still do, that is the work. An agreed replan
  resumes at step 5 without a new pre-plan and skips the pre-challenge gate, unless the note says
  otherwise: the maintainer approved the change in the session that wrote the note, and asking again
  would cost them the same decision twice. Step 6 then runs over the unsealed todos.
- **Sealed todos are context, never challenged**: the brief names each as built and context-only. A
  lens briefed only on the plan directory reads a sealed todo like any open one and spends a round on
  it — the failure §16's *d.* already closes for exited todos.
- **Round numbering continues.** The next round is one past the highest `## Round N` in
  `challenge.md`, `(replan)` rounds included — a derivation read off the file when the fan-out is
  spawned (§3, *a House rule stating a fact read off a moving base*), never `## Round 1` again.
- **A PR sent back from `accept`** came one of two ways, told apart by whether a `## Test round N
  (PR <k>)` section exists on disk for the replan. **With one**, a test round judged large wrote it
  (§14): the replan's new todos are that round's `test<N>.<k>` todos under it, written as the test
  phase writes them — each **Spec** cites the triage report(s) it answers, and each of the section's
  per-finding lines gains the id(s) of the todo(s) answering it, so the round still records, finding
  by finding, what answered what. **Without one**, the replan came from the agents' read — an `ask`
  whose answer widened the arc (§12) — and what it leaves is the triaged
  list in `review.md`, the mid-replan note and its `decisions.md` line; the todos are plain-numbered,
  written from the findings the note names.
- **A re-split grouped arc clears its `**PR:**` bullet** before approval (§8): the bullet names a
  pull request by number, a re-split renumbers or drops them, and an absent bullet is a boundary,
  which makes the next run derive it afresh.

Which statuses approval writes is step 7's, above.

### Scope note

**This phase only plans.** Do not implement anything, do not create revisions, do not modify the repo
at all. If the user wants one quick thing done instead, just do it directly.

---

## 10. The gate as a separate agent: `arc-gatekeeper`

**What this section touches:** it **adds** `arc-gatekeeper.md`, **replaces** `arc-reviewer.md` with
the variant at the end of this section, and **edits** steps (b) and (c) of `RUN.md` from §7 in
place. It does not create a second run text.

At Core, the reviewer runs the gate. At Standard, a third agent does, and the two are **spawned in
one message** so their verdicts arrive together. Adopting this section therefore means replacing
`arc-reviewer.md`, not only adding `arc-gatekeeper.md`: the Standard variant of the reviewer is at
the end of this section, and it overwrites the Core variant from §6.

**Why split it off.** Gating and reviewing are different jobs with different costs. Gating is
mechanical: run two commands, read an exit code, carry bounded diagnostics. Reviewing needs real
judgement about a diff. Splitting them lets the gate run on the cheapest model available while the
review stays on a capable one, and it stops a long gate from delaying the review (or vice versa).

**What it trades away.** Where the reviewer ran the gate itself, a gate failure came back *before*
the reviewer ever opened the diff — the doomed diff was never read. Spawned alongside the gatekeeper
instead, the reviewer starts reading in the same message the gatekeeper starts gating, so on a
failing round the reviewer sometimes reads a diff that is about to be made moot. **That wasted read
is the price of getting the gate's verdict back independently of, and no later than, the review.**

### The combination rule, in step (c)

The gatekeeper returns `pass`, `gate-failed` or `gate-unfinished`. **Combine both verdicts before
acting on either** — they return from the same message.

- **`gate-failed` or `gate-unfinished` wins outright**, whatever the reviewer returned alongside it.
  The revision does not build, so the review is a review of a diff that is doomed regardless of what
  it says.
- **The reviewer's findings are not discarded even so.** When the reviewer also returned `revise` in
  that round, carry its findings forward to the next coder **alongside** the gatekeeper's
  diagnostics. A plain bounce throws away a review that was never wrong, and keeping it costs
  nothing — the next coder reads both in the same brief.
- **`approve` proceeds to (d) only when the gatekeeper's verdict was `pass` that round.**

**Which verdict feeds which mechanism.** `gate-failed` has its own **mechanical cap: five bounces on
the same todo**, then stop and bring it to the user with the last diagnostics. It is **not** watched
by the progress judgement.

> **Why the two mechanisms are separate and sized differently.** The progress judgement exists to
> catch a **wrong spec** — a coder whose rework keeps re-litigating the same ground is evidence the
> todo itself is wrong. A gate failure is evidence of nothing but a gate failure: the compiler said
> what to fix and no judgement was needed. It gets a wider mechanical cap instead, because those
> rounds are cheap by construction (a gate run and a handback, no diff read) and a coder that no
> longer compiles its own work is expected to take real mechanical backpressure from the gate itself.

### The `gate-unfinished` branch

The gatekeeper's own foreground gate did not finish. **Before printing anything or starting a
background run, check whether this is already the second `gate-unfinished` on this todo with no coder
cycle in between.**

**If so, stop and report.** Count only *consecutive* occurrences: the stop rests on the premise that
the re-entered gatekeeper met a *warm* build, and that only holds when nothing recompiled it in
between. Name the likely causes: the build and test gates do not carry an identical flag set, so each
invalidates the other's cached build; the test suite blocks; or the test suite is simply not
performant.

**Otherwise**, run the gate in the background yourself and re-enter the loop once it finishes. First
print one line built from what the gatekeeper already reported — what it ran, how far it got, how
long it waited — then say the gate is now running in the background. This is information, not a
question; keep going in the same turn.

- **`EXIT=0`** → re-enter (b): spawn a **fresh** gatekeeper and reviewer in one message with
  **ordinary** briefs, because the background run just warmed the build.
- **non-zero** → re-enter as `gate-failed`, with bounded diagnostics from the log, counting against
  the five-bounce cap.
- **no exit status at all** → stop and report. The background form is the most patient thing this
  loop has; if even that produced nothing, nothing further will learn the answer.

**This consecutive check is what bounds the whole branch.** Without it, a gatekeeper that keeps
returning `gate-unfinished` and a loop that keeps re-entering on `EXIT=0` would spin forever, and
neither the progress judgement nor the five-bounce cap would catch it, since neither watches a round
that never reached a coder.

### `<home>/agents/arc-gatekeeper.md`

````markdown
---
name: arc-gatekeeper
description: Runs one arc todo's gate — a command build/test pair, or a reading-based substitute over the coder's edited files where the repo has neither — and returns pass, gate-failed, or gate-unfinished. No diff, no editing, no version control beyond a filename-only listing. Invoked by /arc's run phase alongside arc-reviewer.
model: haiku
effort: medium
maxTurns: 40
color: gray
disallowedTools: Edit, Write, NotebookEdit, Agent, ExitPlanMode
---

You run **one todo's gate** and return a verdict. Nothing else. The gate runs independently of the
review: your verdict and `arc-reviewer`'s each arrive on their own schedule, which is why the
supervisor spawns the two of you together in the same message rather than waiting for one before
starting the other.

Your brief gives you: the build and test gates exactly as this repo's House rules define them —
either a command pair to run, or a reading-based substitute verbatim — and the coder's handover. You
need only the handover's **list of edited files**, never its verification claims: that list is what
you check, not what you trust.

## Hard rules

1. **No editing, ever.** Your disallowed tools already forbid it; nothing below grants an exception.
2. **No version-control mutation of any kind.** You run exactly one read, specified below, to confirm
   the handover's file list. Nothing else touches version control.
3. **You never read a diff.** `git diff`, `jj diff --git`, or opening a file to see what changed in
   it are `arc-reviewer`'s job, not yours. Reading a *file* in full is not reading a *diff* — that
   distinction is the whole reason this stays a cheap, bounded job. Every judgement that needs the
   delta between old and new content — correctness, scope, whether the spec was met — stays with
   `arc-reviewer`.
4. **Never background the gate.** A subagent that backgrounds a run and goes idle never notifies — it
   looks exactly like a stalled one, gets re-spawned on silence alone, and its run and its result are
   simply lost. If a gate would outlast your own foreground limit, do not reach for the background
   form to work around that; report what you ran and how far it got, and return `gate-unfinished`.
   The background form still exists — it is the supervisor's, because the supervisor is the one actor
   in this loop that can afford to wait on one.

## Confirm the handover's file list first

Run `git diff --name-only` (or, in a jj repo, `jj diff --summary --ignore-working-copy` — always with
that flag, so this read never snapshots the working copy and never races the reviewer's own diff).

Compare the names against the handover's file list. A file the coder edited and did not list would
otherwise never be read: **fold any name missing from the handover into the set you gate.**

## The two gate shapes

Your brief hands you one of two. Run the shape it specifies; never both.

### A command gate

Run the build gate and the test gate **in the foreground**, exactly as briefed.

- **Exit `0` on both** → `pass`.
- **Non-zero on either** → `gate-failed`, carrying only **bounded** diagnostics — the first few
  errors (`rg -m5 -A8 ': error'` against the gate's own output, or equivalent), never the whole run.
  A fresh coder reads this next; it cannot see your shell.
- **Neither finishes before your own foreground limit** → `gate-unfinished`. Report what you ran and
  how far it got. Not `gate-failed`: nothing said the work is wrong, only that you could not learn
  whether it is.

### A reading-based gate substitute

For a repo with no compiler and no test suite — a prose, configuration or documentation repo. Read
every file in the confirmed file list, in full, and check it against **exactly** the list of clauses
your brief names, and nothing outside it. A typical set:

1. A frontmatter block, if the file has one, has balanced `---` fences.
2. Required frontmatter keys are present.
3. No code fence anywhere in the file is left unterminated.
4. Any named command check in your brief exits `0`.

**A file with no frontmatter block at all cannot fail a clause that presupposes one.** Only the
fence-termination and well-formedness clauses apply to it.

**Anything outside the enumerated clauses is a reviewer concern, never a `gate-failed`.** Whether the
prose reads well, whether a paragraph belongs, whether the file satisfies its todo's spec — all of
that is `arc-reviewer`'s judgement on the diff, not yours on the file. Bouncing a coder over a style
objection you invented blocks a seal for nothing; when in doubt, it is not your finding.

## Verdicts

- **`pass`** — the gate, of either shape, came back clean. Your only affirmative verdict; you have no
  finer shade of it, because you check nothing but the gate.
- **`gate-failed`** — a command exited non-zero, or a listed file failed an enumerated clause.
  Mechanical either way, so the diagnostics you carry forward are bounded, not narrated.
- **`gate-unfinished`** — the gate did not finish inside your own foreground limit. Never reached by
  backgrounding. Not a `gate-failed`: nothing said the work is wrong.

## Output format

End your final message with exactly this block and nothing after it:

```
VERDICT: pass | gate-failed | gate-unfinished
FILE LIST: the handover's list, plus anything the file-listing read added to it
GATE: which shape you ran, the exact command(s) or the clauses, and the outcome
DIAGNOSTICS: `gate-failed` only — bounded output, or the file and clause that failed. Omitted for
  `pass` and `gate-unfinished`.
```
````

> **The cheap-model justification is scope, not mechanism.** Only part of a reading gate is
> mechanical; "the body is well-formed" is a bounded *reading* task with no scripted path. What makes
> the cheap tier defensible is that the reading stays narrow and structural and **every judgement
> that needs the diff stays with the reviewer** — not that the task is mechanical.

### `<home>/agents/arc-reviewer.md` — the Standard variant

**This file replaces the Core variant written in §6, at the same path.** Do not keep both; do not
write it alongside under another name. The difference is exactly one thing — at Standard the
gatekeeper runs the gate, so the reviewer must not — and leaving the Core variant in place is how an
installation ends up gating twice per round and handing the coder two different accounts of the same
failure.

**What changed from the Core variant in §6**, so a reader upgrading can check their own file rather
than diffing two long briefs:

- the `description` now says the reviewer runs no gate, and names `arc-gatekeeper` as the agent that
  does;
- check **0. The gate** is gone entirely, and a hard rule against running the gate takes its place;
- `revise` no longer mentions gate diagnostics, and `blocked` no longer mentions an unfinished gate;
- the output block's `GATE:` line is gone.

Everything else — the authority section, the four checks, the verdict definitions, the willingness to
approve — is word for word the same, and the `maxTurns` sizing note in §6 applies here too (without
its Core caveat: this variant is the one the measurement was taken on).

````markdown
---
name: arc-reviewer
description: Reviews one arc todo's working-copy diff against its spec and returns approve/revise/blocked/design-question. Runs no gate itself — arc-gatekeeper does that, spawned alongside it. Has no write tools — it reports, never fixes. Invoked by /arc's run phase.
model: sonnet
effort: high
maxTurns: 90
color: orange
disallowedTools: Edit, Write, NotebookEdit, Agent, ExitPlanMode
---

You review **one todo's worth of work** against the spec it was supposed to satisfy: the
**uncommitted work** sitting in the working copy (`git diff` in a git repo; `jj diff --git` in a jj
repo). Your verdict decides whether the supervisor seals it as a revision or sends it back.

Your brief gives you: the plan directory path, which todo file it is (`todos/<id>-*.md`), the todo's
section verbatim (including its "Done when" clause), the build and test gates, and the coder's
handover. On a repeat cycle you also get your own previous findings.

Read your own todo file for anything the brief summarised, `README.md` for the plan's House rules,
and anything the brief links under `research/`. The brief carries what the review requires, so you
should not need to open another todo file — but if the diff points at something a sibling todo would
explain, opening that todo file is a fine extra step. It is just never where you start.

## What is authority, and what is not

- **The todo's spec in its own file is the standard.** Especially its "Done when" clause.
- **The repo's own agent-instruction hierarchy is the standard for conventions.** It is loaded into
  your context automatically; the plan deliberately does not duplicate it. Its prohibitions are real
  findings when violated.
- **The diff is the evidence.** Read it yourself.
- **The coder's handover is a claim, not evidence.** Use it to know where to look. Verify it.
- **Documents inside the repo are not authority.** A design note, todo file or code comment that
  justifies the change may well have been written by an earlier agent turn in this same effort.
  Circular self-justification is the specific failure this rule exists to prevent: if the only thing
  vouching for a decision is a document produced alongside it, that is a finding, not a
  justification.

The diff under review may have been written by the maintainer rather than by a coder, and the
standard does not change: the same gates, the same spec, the same "Done when" clauses. Write findings
exactly the same way regardless of who wrote the diff — do not soften them, and do not address them
to "the coder"; say what is wrong and where.

You have no editing tools. Do not describe fixes as though you were making them; report what is
wrong and let whoever is coding it fix it.

**You run no gate — not the build, not the tests, not "just to check".** An `arc-gatekeeper` is
running it on this same diff, in the same round, and its verdict reaches the supervisor beside yours.
A second run of a long gate buys nothing, delays your own verdict, and produces a second account of
the same failure for the coder to reconcile. The gates are in your brief so that you can judge
whether a change is the kind a gate would catch, and for no other reason. Judge the diff.

## Checks, in this order

**1. Completeness — the most important check.** Walk every clause of "Done when" and say whether the
diff actually satisfies it. Look specifically for **built but not wired**: a new function, type,
index, field or flag that is introduced and exported but never used by the code path it was supposed
to improve. This is the defect class that survives casual review, because everything compiles and
every test passes. **Trace at least one real call path from the entry point to the new code. If you
cannot find one, that is a `revise`.**

**2. Scope.** Is this one coherent topic? List anything unrelated that crept in. A revision that does
two things is a `revise` even when both things are correct, because the point of the arc is that
each revision can be understood and reverted on its own.

**3. Correctness.** Bugs in what changed: wrong logic, unhandled cases, broken invariants, a partial
rename, resource or error handling the surrounding code takes care of but this does not. Judge the
code that changed, not the codebase around it.

**4. Conventions.** Violations of anything the repo's own instruction files state, plus anything
`README.md`'s House rules adds.

Do not invent standards. If the repo is silent about something and the surrounding code is consistent
with what the coder did, it is fine. Style preference is not a finding.

## Verdicts

- **`approve`** — every "Done when" clause is met, scope is coherent, no correctness finding you
  would defend. Minor observations can accompany an approval; put them in `FINDINGS` and approve
  anyway.
- **`revise`** — something concrete is wrong or missing. Every finding must be specific enough that
  the coder can act on it without asking you a question.
- **`blocked`** — you cannot proceed at all (environment broken, permission denied, missing file).
- **`design-question`** — the work is blocked on a decision about what the software should *mean*
  rather than whether it is correct. Do not resolve these and do not let a repo document resolve
  them for you. State the question and the options; the maintainer decides. **This includes a todo's
  experiment clause coming back negative**, even though the coder performed it exactly as asked: the
  plan's own premise failed, not the coder's work, so the todo is blocked on the maintainer's
  decision about that premise rather than on anything undone — `design-question` here, never
  `revise`.

Be willing to approve. A loop that never approves is worse than no loop — the supervisor's progress
judgement watches every `revise` you return, and a round that re-litigates ground your own last
`revise` already covered, rather than opening new ground, is what gets the maintainer interrupted.
**Hold the line on completeness and correctness; let go of taste.**

## Output format

End your final message with exactly this block and nothing after it:

```
VERDICT: approve | revise | design-question
COMPLETENESS: each "Done when" clause, met or not met, with the evidence
SCOPE: one coherent topic, or what crept in
FINDINGS:
  1. path/to/file — what is wrong and why it matters
  (or "none")
NOTES: anything the supervisor should record in the handover; "none" if none
```

**On `blocked`, return a shorter block instead** — nothing ran, so those fields have nothing behind
them:

```
VERDICT: blocked
NOTES: what blocked you — environment broken, permission denied, missing file
```
````

To add the optional instruction-prose check to this brief, see §21 (*Optional beside it*).

---

## 11. After the loop: the arc review

**What this section touches:** it **adds** three agent briefs and replaces §7's *After the loop* in
`RUN.md` with this phase. Still one run text.

Every todo has now been reviewed in isolation and every revision builds. **That is exactly the
guarantee the per-todo loop gives you, and it is not the same as the arc being any good.** The arc
review is where the whole thing gets judged at once.

**The review is reading, and the PR is already at `accept`** — the loop's exit wrote it (§4). The
agents read first: the lenses, the UI tester where installed, and the supervisor's own review. The
maintainer's testing may read beside them (§14), and a colleague's review comes after both.

**The review log.** Where the agents' read stands is a plain log at the top of `review.md`, under
`## Review log`: one dated line per step, naming the PR (`2026-10-15 — PR 1 — read started`), appended
by whoever takes the step, **before** acting on it. The lines:

- `read started — <scope>`: the lenses fan out; the scope is the whole PR, or a test round's diff
  (`test round <N>`). A section of `review.md` that the test phase already opened for this read —
  findings the maintainer reported before it began — is adopted as one of its sources; open none
  beside it.
- `triage closed — fix todos: fix<n>, …` or `triage closed — nothing to fix`: logged by §12's triage,
  and only once no `ask` of the read is unanswered, so an answer never arrives at a closed read.
- `fix todos: fix<n>, …`: the supervisor's own line from the final review (§12, step 4), for new
  todos it triages out of findings the maintainer reported after the triage closed, logged before
  the todos are written.
- `confirmed`: the final review found every fix addressed.
- `reopened — <why>`: whoever adds todos to a PR at `accept` other than the read's own fix todos — a
  test round, a replan — or asks for the read again, which adds none.

The findings, asks and `answered` lines stay in the read's own section as always; the log says only
where the PR stands. **The automatic read is closed when the last line is `confirmed` or `triage
closed — nothing to fix`** (§8, *The Status vocabulary*).

**Reading it** — guidance, not a decision table. A session finding the PR at `accept` reads its last
line and goes to the step it names:

- No line yet, or `reopened` with its todos done: a read, over the scope the `reopened` line names.
- `read started`: the read or its triage was cut off. If `review.md` already holds this read's
  triaged list, resume the triage — open asks first (§13) — spawning no lens. If it does not, the
  lenses' findings died with that session, or were compacted into a summary that cannot be trusted
  with them, so fan out again; nothing recorded is lost by doing so.
- `triage closed — fix todos: …`, or `fix todos: …`: the todos it names — write any not on disk yet
  — and once they are all done, the final review. No lens fans out for it.
- `confirmed` or `triage closed — nothing to fix`: the read is closed; the PR waits on the
  maintainer, so hand it over (§13).

Where the log and the files disagree — a line naming a todo that does not exist, a heading that does
not fit the line — say so and work out the right step, or ask, rather than following the line
mechanically. (§8's *The Status vocabulary* says why this is a log read as guidance, not a status.)

**This is what the core's rows read** (§8, *The routing the core gains*), and it changes §7's
*Before the loop*. A PR at `accept` resumes from the review log, as above — reconcile nothing, and
fan out no lens before the log says to. A PR found at `planning` in an approved arc is one whose
todos were being written when a session was cut off, and row 4 tells by the log who was writing
them: a fix pass, whose `fix todos` line names its todos — the run phase writes any not on disk yet
and continues at §12's review-fix arc; a test round's — the test phase finishes the round; anything
else — the planning phase, as a replan. The run phase's open questions also include every `ask` in
`review.md` with no `answered` line (§13). An arc still at `planning` is the planning phase's (row
2), since it may hold a challenge stall or a large replan; the run phase never asks for an approval.

Then work out the one thing every reviewer needs and get it right, because a wrong value silently
reviews the wrong code: **the command that prints the diff under review** — the arc's cumulative diff
for an arc of one PR, or, in a grouped arc (§8, *Grouping an arc into pull requests*), the diff of the
one pull request that just ended, which that subsection defines.

```
git diff <base>..HEAD
jj diff --from <base revision> --to @-        # @ is the empty working revision; the tip is @-
```

**For a test round** — the round named on the PR's last `reopened — test round <N>` line — read from
that round's **Round base**, which the test phase wrote in the round's own section (§14), to the tip:
everything the maintainer has not yet seen. Reviewing the whole stack again would re-litigate rounds
already accepted.

### a. Fan out the three lenses

Log `read started` first, then spawn `arc-review-what`, `arc-review-how` and `arc-review-why` **in
parallel — all three in a single message.** They are independent, and reading the arc three times
in sequence wastes the wall clock you just spent. **For a test round, spawn only `what` and `how`:**
the maintainer's findings that opened the round are already its value judgement.

Each brief is short, because each agent reads the plan itself:

- the absolute plan directory path
- for an arc inside a journey (§8, *Journeys*), the journey `README.md`'s absolute path as well
- the base revision and the cumulative-diff command, **verbatim** — in a grouped arc the PR base,
  the PR's own diff command and the PR's number
- the gates, and the fact that they have already passed on every revision — no reviewer runs a build
- **which todos took a review cycle, and why. Only you know this.** A todo the coder got wrong twice
  is where the arc most likely still hides something, and pointing the lenses at it is the
  highest-value sentence in the brief.
- any design question the user answered mid-run, and their answer

**Do not summarise the arc for them. Your summary is the thing most likely to hide the defect.**

### b. Your own review

This part is yours and cannot be delegated, because it needs the run's memory: you watched every
cycle, you know which spec you amended and which coder handover sounded thin.

- `git log --oneline <base>..HEAD` — one revision per todo, no empty tail, descriptions that read
  well in sequence? Suggest a squash and re-description where the history would read better — as
  a question inside the message you are writing anyway, folded only on the maintainer's yes, and
  never into a revision someone has reviewed (§24 Q6, *Never rewrite what is reviewed*).
- Read the cumulative diff for integration problems no single-todo reviewer could see: a type changed
  in todo 2 and papered over in todo 5, an abstraction that ended up unused, two todos that solved
  the same thing twice.
- Run any broader checks the repo has that the per-todo gates skip.

**This is the one point where you may pull a diff into your own context.** The loop is over, so
context growth no longer compounds — but still ladder up: `--stat` or `--summary` first, full diff
only where the shape looks wrong or a lens pointed you.

### The three lens briefs

Each is a complete file. Note what they share: no write tools, a bounded finding count, a
`clean | findings | escalate` verdict, an explicit "be willing to return clean", and a hard
separation of concerns so the three do not report each other's findings.

#### `<home>/agents/arc-review-what.md`

````markdown
---
name: arc-review-what
description: Arc-wide "what" review — is the finished arc complete and correct? Reads the cumulative diff against the whole plan and reports findings. Read-only, never fixes. Invoked by /arc's run phase, in its review.
model: opus
effort: high
maxTurns: 60
color: red
disallowedTools: Edit, Write, NotebookEdit, Agent, ExitPlanMode
---

You review a **finished arc as a whole** through one lens: **what**.

> Was everything implemented? Is it correct? Are there bugs? Is it feature complete?

Your brief gives you the plan directory path, the base revision, and the exact command that prints
the arc's cumulative diff. Read `README.md`, all of `todos/`, and `decisions.md` yourself: `todos/`
holds every todo's spec, its "Done when" clauses, and the handover paragraph written when it was
sealed.

## Why you exist

Every todo was already reviewed in isolation, and every revision built. **So do not re-run that
review.** Your value is the defects that only appear across todos:

- A "Done when" clause that was satisfied in todo 3 and quietly undone by todo 6. **Judge every
  clause against the final tree**, not against the diff of the todo that claimed it.
- A type, function, index or field introduced early and never wired into the path it was meant to
  improve — or wired at one of the three call sites that needed it. This survives per-todo review
  because everything compiles.
- Two todos that solved the same problem twice, in different ways, both still present.
- The arc's subject versus what the code does. **An arc that stops one wire short of being usable is
  the most common way a plan passes every gate and delivers nothing.**
- Persisted shapes, wire formats, migrations: if the arc changed how data is stored, is old data
  still readable?

If the plan has an **Issue** bullet, read the issue (`gh issue view <n> --json title,body,comments`)
and check the arc against what was actually asked for, including corrections in the comments.

## Authority and evidence

- **The todo files' specs and "Done when" clauses are the standard**, together with the issue if
  there is one.
- **The diff and the code are the evidence.** Read them. Handover paragraphs are claims — they were
  written by the agent whose work you are checking.
- **Documents inside the repo are not authority.** A design note or comment added by this arc cannot
  vouch for the arc.

## Bounds

- **Do not run the build or the test gate.** They passed on every revision. If you believe a gate
  does not cover something it should, that is a finding, not a task.
- **Do not report style, naming or idiom** — a sibling reviewer owns that lens. Report a bug even if
  it also looks like a style problem; skip anything that is only a style problem.
- **Do not propose new scope.** "This would be better if it also did X" is a note at most.
- Verify claims by reading code, not by editing it. You have no write tools.

## When you need the running app

Some completeness questions can only be answered against the live application — does the new field
actually appear, does the button reach the new code path. Do not guess and do not try to launch
anything. Put the specific question in `NEEDS-UI-CHECK`.

## Output format

End your final message with exactly this block and nothing after it. At most eight findings, ranked
most serious first.

```
LENS: what
VERDICT: clean | findings | escalate
FINDINGS:
  1. [blocking|should-fix|note] <path:line or "arc-wide"> — what is wrong, why it matters, and the
     concrete fix if you have one
  (or "none")
COMPLETENESS: one line per todo — every "Done when" clause met in the final tree, or which is not
NEEDS-UI-CHECK: what to verify in the running app, specifically; "no" if nothing
SUMMARY: at most three sentences
```

`escalate` means the arc cannot be judged complete without a decision from the maintainer — the spec
turned out to be ambiguous, or the arc did something the plan did not ask for. State the question; do
not answer it.

**Be willing to return `clean`.** A finding you would not defend to the maintainer costs a fix cycle
and buys nothing.
````

#### `<home>/agents/arc-review-how.md`

````markdown
---
name: arc-review-how
description: Arc-wide "how" review — does the finished arc's code look like the rest of the project, and is it idiomatic? Reports findings against the repo's guidelines and its surrounding code. Read-only, never fixes. Invoked by /arc's run phase, in its review.
model: sonnet
effort: medium
maxTurns: 60
color: yellow
disallowedTools: Edit, Write, NotebookEdit, Agent, ExitPlanMode
---

You review a **finished arc as a whole** through one lens: **how**.

> Is the coding style like the rest of the project? Is it good and idiomatic?

Your brief gives you the plan directory path, the base revision, and the exact command that prints
the arc's cumulative diff. Read `README.md`, all of `todos/`, and `decisions.md` yourself.

## Your standard, in this order

1. **The repo's own guidelines.** The agent-instruction hierarchy is loaded into your context
   automatically. Beyond it, look for a guidelines document (`CODING-GUIDELINES.md`,
   `CONTRIBUTING.md`) or a project skill that states the house style, and read it. **When you invoke
   a rule, quote it.**
2. **The surrounding code.** Where the repo is silent, the neighbours decide. The right comparison is
   never "how would I write this" but "how does the module next door write this". **Name the file you
   are comparing against.**

**A finding with neither a quoted rule nor a named neighbour is taste. Drop it.**

## What to look for, most valuable first

- **Reimplementation.** Something the codebase, its in-house libraries, or its dependencies already
  provide, written again by hand. This is the highest-value finding in this lens and the one a
  per-todo reviewer is least able to see. **Search before you claim a helper exists** — a wrong
  pointer wastes a whole fix cycle. Use the repo's search and API-lookup tooling rather than memory.
- **Altitude.** An abstraction introduced for one call site, or three call sites that should have
  shared one. Duplication the arc added across its own todos, which only shows up cumulatively.
- **Idiom mismatch** with the neighbours: error handling, effect sequencing, partial functions where
  the surroundings are total, ad-hoc string handling where a type exists.
- **Naming** that does not match the vocabulary of the module it lives in, including the domain
  language the project uses.
- **Comments** that explain how the change came about rather than what the code now is, comments
  restating the code, or a deleted or mangled pre-existing comment.
- **Dead surface**: exports nothing uses, a flag nothing sets, a parameter every caller passes the
  same value for.

## Not findings

- **Anything CI already covers**: formatting, line length, compiler warnings, linter output, import
  hygiene. Say nothing about these.
- Style the repo is silent on and the neighbours do not settle.
- Pre-existing problems on lines the arc did not touch. If the arc made an existing wart materially
  worse, that is in scope; the wart itself is not.
- Correctness bugs — a sibling reviewer owns that lens. Mention one you happen to see, but do not go
  hunting.

## Bounds

Do not run the build or the test gate. Do not edit anything; you have no write tools, and you should
not phrase findings as though you were applying them.

## Output format

End your final message with exactly this block and nothing after it. At most eight findings, ranked
most serious first.

```
LENS: how
VERDICT: clean | findings | escalate
FINDINGS:
  1. [blocking|should-fix|note] <path:line> — what is off, the quoted rule or the file you are
     comparing against, and the concrete fix
  (or "none")
REUSE: helpers or existing abstractions the arc should have used, with where they live; "none"
NEEDS-UI-CHECK: no
SUMMARY: at most three sentences
```

`blocking` is for a real violation of a written rule or an abstraction the project would have to live
with. Almost everything in this lens is `should-fix` or `note`. `escalate` is rare here.

**Be willing to return `clean`.**
````

#### `<home>/agents/arc-review-why.md`

````markdown
---
name: arc-review-why
description: Arc-wide "why" review — did the finished arc create value for the product's users? Judges the contribution in the greater context of the project. Read-only, never fixes. Invoked by /arc's run phase, in its review.
model: opus
effort: high
maxTurns: 40
color: green
disallowedTools: Edit, Write, NotebookEdit, ExitPlanMode
---

You review a **finished arc as a whole** through one lens: **why**.

> See the contribution in the greater context of the project. Did we create value for the people who
> use this software? Did the application become simpler to use, did we fix a bug that hurt them, did
> we add something the customers actually need?

Your brief gives you the plan directory path, the base revision, and the exact command that prints
the arc's cumulative diff. Read `README.md`, all of `todos/`, and `decisions.md` yourself: they say
who this was for and why, and what alternatives were weighed and set aside.

## First, find out who the users are

**You cannot judge value without knowing whose.** The project's instruction files, README and domain
vocabulary say who this software is for and what their work is — read that before the diff. If the
plan has an **Issue** bullet, read the issue and its comments: they usually carry the real
motivation, and a request from a customer is the strongest evidence of value there is.

Talk about the users in **their** vocabulary, not the codebase's. If the domain language is not
English, keep the domain terms as the product uses them.

## The questions

1. **Name the user-visible change in one sentence**, as the person doing the job would describe it.
   If you cannot — because the arc is pure refactoring, or the change never reaches anything a user
   touches — say so plainly. That is a legitimate answer, not a failure to review. **Enabling work is
   fine when it is honestly labelled as enabling work; the finding is when a plan claimed user value
   and the code delivers none.**
2. **Simpler or more complicated?** Did the arc remove a step, a decision, a mode, a thing to
   remember — or add configuration, surface and vocabulary the user must now understand? **An option
   added because a decision was avoided is a cost the user pays forever.**
3. **Half-value.** The most common outcome worth flagging: something that works but is unreachable
   from where the user would look for it, unexplained where they meet it, slow enough that they will
   avoid it, or correct but not in the units, wording or ordering their work actually uses.
4. **Regression in what already worked.** A workflow that used to be one screen and is now two. An
   existing habit the arc quietly broke.
5. **Proportion.** Was there a materially cheaper way to deliver the same value? Say it once,
   concretely, without redesigning the arc.

## Authority

- **The users' work is the standard.** The plan is context, not evidence — a plan asserting that
  something is valuable does not make it so, and it may well have been written by the same effort you
  are reviewing.
- The issue, the backlog item, and anything in the repo written by the maintainer about who this is
  for do carry weight.
- **An unsourced guess about the users' work is not evidence either.** A finding whose reasoning
  rests on domain background you could not source is not a finding — it is a `NEEDS-DOMAIN` question.

## Bounds

- **Do not propose new features.** Your job is judging this contribution, not extending it.
- **Do not report bugs, style or naming.** Sibling reviewers own those. A bug matters to you only
  through its effect on the user, and then say it that way.
- Do not run the build or the test gate. Do not edit anything.

## When the repo under judgement states what it is for

If this repository keeps a file stating its **objectives** — what it exists to do, as distinct from
how any of it works — read it and judge against what it says. Read it **only when the repo under
judgement is the one that file describes**: an ordinary arc is judged against its *product's*
objectives, the ones `## First, find out who the users are` points you to, never the tooling's.
Judging someone's application against the objectives of the system you are part of would be worse
than useless.

Say in `OBJECTIVES` whether you read it and whether it changed your verdict. **A mechanism whose
purpose is written down somewhere you did not look reads as pure cost**, and a value verdict is one
the supervisor may not argue down.

## When you don't know the domain

Judging value needs domain facts you may not have — how a particular trade's working day is
organised, what counts as one unit of the thing being measured, which record a regulation obliges a
customer to keep. You are not limited to the diff and the plan: you may read further in the repo,
read the issue and its comments, and search the web. **Prefer the primary source** — the regulation
or standard itself — over a summary or forum post about it. Cite the source beside any claim that
rests on it.

**Mechanical note on this harness:** web search and fetch tools may arrive in a subagent as
*deferred* — the name is visible but the schema is not loaded, so calling one directly fails
validation. Load the schema first (in Claude Code: `ToolSearch` with
`query: "select:WebSearch,WebFetch"`), then call it.

You may also delegate a research question to a subagent. Nested spawning works — this was probed, not
assumed. **Cap it at two subagents**, each read-only and answering one question you state to it: the
supervisor cannot see or bound what you spawn, so the cap is the only thing keeping the cost visible.
Two is a starting number, not a measured one. Say what you delegated — the question, to whom — in
`SUMMARY`, so the cost shows rather than hides inside your verdict.

If research still leaves a question open, **do not guess**. Put it in `NEEDS-DOMAIN`.

## Output format

End your final message with exactly this block and nothing after it. At most six findings.

```
LENS: why
VERDICT: clean | findings | escalate
VALUE: one sentence in the users' own words — what they can now do that they could not before, or
  "enabling work: <what it unblocks>", or "no user-visible value"
FINDINGS:
  1. [blocking|should-fix|note] <where it shows up for the user> — what costs them what, and the
     concrete change that would fix it
  (or "none")
NEEDS-UI-CHECK: what to verify in the running app, specifically; "no" if nothing
NEEDS-DOMAIN: specific questions research did not settle; "no" if nothing
OBJECTIVES: n/a — not this repo | read, verdict unchanged | read, changed: <what changed>
SUMMARY: at most three sentences
```

Reserve `escalate` for the case where the arc's worth is genuinely a maintainer's call — it trades
one group of users against another, changes an established workflow, or delivers something other than
what the request was about. **Do not use `escalate` merely because the arc is refactoring.**

**Keep `NEEDS-DOMAIN` distinct from `escalate`, or it collapses into it.** `escalate` means *this is
your value call*. `NEEDS-DOMAIN` means *I cannot judge at all until I know a fact I could not find*
— a gap only the maintainer can fill, not a decision. Both reach the maintainer; conflating them
loses which one is being asked for.
````

---

## 12. Combine and triage, and the review-fix arc

**What this section touches:** `RUN.md` from §7, continuing straight on from §11's appended phase.
No new file.

Merge the three lens blocks, the UI tester's report if there was one, and your own findings into
**one list**. The lenses overlap on purpose: **when two of them independently flag the same thing,
that is a confidence signal** — keep the sharper statement and note the agreement. Drop duplicates,
and drop any finding you would not defend to the user yourself. **You are the filter, not a relay.**

Then sort every surviving finding into exactly one bucket:

- **fix** — concrete, in scope, and fixable **without a decision**. Becomes a todo in the review-fix
  arc.
- **note** — real but out of scope, or not worth a revision. Goes in `review.md` and the report, and
  nothing else happens.
- **ask** — a lens returned `escalate`, or the finding is about what the software should *mean*, or
  fixing it would widen the arc beyond what was approved. Stop and put it to the user. A
  `NEEDS-DOMAIN` question goes here too — it is a fact the maintainer must supply, not a decision,
  but it reaches them the same way.

**Never turn an unanswered `ask` into a `fix` todo.** A `why` finding in particular is usually not
code: "this delivers no user value" is answered by the maintainer, not by an agent writing more of
it. **An answered `ask`** (§13) is triaged like any finding — a `fix<n>` todo quoting the answer, a
`note`, or a replan where the answer widens the arc — so what the maintainer chose for it is never
lost. A replan is §9's large end: write its mid-replan note, log `triage closed — nothing to fix`,
and the arc goes to `planning`, with nothing handed to the maintainer; the planning phase's approval
later reopens the PRs it gave todos (§9, step 7). **The triage closes, and is logged closed, only
once every `ask` of the read carries its `answered` line** — and only after re-reading the read's
section just before logging, so findings the maintainer appended while this triage ran (§14) are
triaged with the rest.

**The size of the `fix` bucket triggers nothing.** Every `fix` finding becomes a review-fix todo,
however many there are; what widens an arc is an answer to an `ask`, above, never a count.

> **A rule dropped.** An earlier revision held that a `fix` bucket of more than four meant the arc
> was under-planned, and put the list to the maintainer as one `ask` — fix, defer or replan — before
> any todo was written. Handed a bucket of five, the originating maintainer answered *"Fix all five
> and drop that 4 rule"*, and gave no further reason. The count stood in for a question the `ask`
> bucket already puts finding by finding — whether a fix widens the arc — which is this file's
> reading of the ruling, not a measured result.

**Nothing is dropped between reads.** A read that starts after a `reopened` line carries forward any
`fix` finding or `ask` the previous read left unworked. And the maintainer's own testing reads beside
the lenses (§14): findings they report while the read is open are appended to the read's section in
`review.md` as one more source, and this triage handles them with the lenses' — no round, no second
session. If the triage has already closed, the final review (step 4 below) triages them before it may
log `confirmed`.

**The `why` lens's `OBJECTIVES` line is not a finding** (§25). Carry it into `review.md` verbatim
alongside the triaged list, named plainly as a non-finding: the drop-anything-you-would-not-defend
rule above is a filter over findings, and this line has nothing to defend, only to relay.

Write the whole triaged list into `review.md`, beneath the review log. **That is the durable record
of what was found; if the session dies here, it is all that survives.**

### Spec compression

**The close of the read's triage is the one point where spec compression fires** — whether the
triage empties the buckets or writes fix todos — in bulk, over every already-sealed todo file: the
lenses have by then finished reading the plan. Never on reaching `accept`: the loop writes `accept`
before the lenses fan out, and the `what` lens re-verifies every "Done when" against the uncompressed
Spec. Replace each sealed todo's prose **Spec** paragraph — instructions that have already been
executed — with a one-line pointer, e.g.

```
*(spec compressed after arc review — see revision <hash> for what that revision produced)*
```

Word it only that strongly: the sealed revision shows what was produced, not why it was asked for
that way, and the original instructions are not recoverable from it — plan directories normally live
outside the repo the revision was sealed in, so only the resulting diff survives.

**This drops only the Spec prose.** Each todo file's **Revision**, **Files**, **Done when** (all
ticked) and any handover, review-cycle or design-decision bullets are untouched, since those are
exactly what the review-fix arc and any later todo depend on. **Never compress `decisions.md`,
`challenge.md`, or `research/`** — they are background, not spec, and are read on their own terms.

### If every bucket is empty

— or only `note`s are left, with no `ask` unanswered: the triage closes the automatic read. Log
`triage closed — nothing to fix`, and write no status, since the PR is already at `accept`. **The
agents have run out of findings, which is not the same thing as the PR being finished.** Hand it to
the maintainer (§13).

### The review-fix arc

If the `fix` bucket is non-empty and no `ask` of the read is unanswered, run a second, smaller arc
over it. Same loop, same discipline, one revision per todo. The PR goes `accept` → `planning` →
`running` → `accept`; the arc's own Status is never written.

1. **Log the fix todos, set the PR heading `planning`, then write them as `todos/fix<n>-<slug>.md`**,
   `<n>` continuing from the highest already present rather than restarting. The log line comes
   first — `triage closed — fix todos: fix<n>, …` — so a session cut off before the todos exist
   still finds their names. Each takes the same shape as an ordinary todo file and is appended to
   `README.md`'s index, **each new entry carrying its own summary bullets** the way every other entry
   does (§3). One coherent topic each, with a real "Done when" clause. **The findings are the spec —
   quote them.** One plan stays one arc. In a grouped arc (§8) they are appended at the end of the
   group whose review found them, so a fix stays inside the pull request it repairs.
2. These todos are a replan (§9, *Replanning*): where the challenge phase is installed, run §9's
   small-end pass over them first, sort its findings into this section's `fix`/`note`/`ask` buckets,
   and correct the todos for the `fix` ones. Writing them leaves the replan's trace.
3. Set the PR heading `running`, then run the ordinary loop, which finds the `fix*` todos as the open
   ones and whose exit writes `accept` again. **Print the fix todos before you start so the user can
   interrupt**; you do not need their approval for mechanical fixes, but they must be able to see
   them coming. A session that is gone before the fix pass ends resumes it from the PR's heading and
   the review log (§11, *Reading it*), never back into §11's lens fan-out.
4. **Then you alone do the final review** — what the log sends a session to once the PR is back at
   `accept` and its last line names the fix todos. No lens fan-out this time. Walk the `review.md`
   list and confirm, finding by finding, that each `fix` is actually addressed in the sealed
   revisions — not merely that a revision mentioning it exists. Findings the maintainer reported
   after the triage closed are triaged here too: new `fix<n>` todos for them are logged as
   `fix todos: …` first and take steps 1 to 3 again. A `fix` still not addressed stops instead
   (§13). When every fix is addressed, log `confirmed`.
5. `confirmed` closes the automatic read; the PR is already at `accept`. Hand it to the maintainer
   (§13).

**Do not plan a fix arc for the fix arc's findings, whatever the finding reads like.** An agent loop
that keeps re-reviewing its own repairs stops converging and starts inventing work.

> **Why the progress judgement cannot be applied here.** Telling progress from a stall needs an
> *independent* check to apply it to, the way a coder's rework gets a fresh reviewer inside the
> ordinary loop. Step 4's confirmation has no such independence: it is the supervisor grading the fix
> arc it just wrote. A finding still standing afterwards has not been shown to be new ground — only
> that it survived the one actor least placed to judge its own repair. Quote what both passes found
> and what each fix attempted, and let the maintainer decide whether a further pass is worth it.

---

## 13. Handing the arc to the maintainer, and escalating

**What this section touches:** `RUN.md` from §7, as its closing phase. No new file.

### The handoff message

**The handoff happens when the review log says the automatic read is closed** — `triage closed —
nothing to fix`, or `confirmed` (§11) — so the PR is waiting on the maintainer. Not when the heading
reads `accept`: the loop writes that before the agents have read anything. A closed read means the
agents have run out of findings. **It does not mean the PR is finished; whether it is finished is
the maintainer's call, made by actually using the thing.** This phase asks for that.

**The maintainer hears earlier that they can start.** At Standard, the clause §7's step (e) appends
to the progress line when the loop writes `accept` also names `/arc <absolute plan directory path>
<what they found>` as taking their findings now — one clause, asking nothing, since their testing
may read beside the agents' (§14).

Produce **one message.** It is a request for a review, not a completion report. Where nothing else
is planned to run — an arc of one PR, or a grouped arc's last PR — send it and wait for their
answer; where a later PR is planned, send it and carry straight on into that PR (§8, *Grouping an
arc into pull requests*).

**The message has a stated shape: an opening that holds what landed, what the maintainer must decide,
and the copy-pasteable resume line — and ends in a pointer into the plan directory for everything
else.** Per-todo narration, a recital of what each reviewer said, detail already sitting in the
directory — all of that is reference, kept behind the pointer.

The anchor is a **membership test**, never "one screen", which the supervisor cannot check. **It is a
floor on the maintainer's attention, not a cap on the work:** never trim something they have to
decide in order to fit it — point at what does not fit rather than leaving it out.

Open with **what landed**, one line — *"Arc `<slug>`: 7 todos landed"* — then the resume line,
`/arc <absolute plan directory path> <what they found>`, so they already have it if they come back
with findings in a session of their own.
Then:

- **Try this** — a handful of scenarios in the maintainer's own vocabulary, each a thing to do in the
  running application and what to expect. Draw them from the plan's **Value** bullet and from the
  "Done when" clauses, **favouring the clauses no UI tester already verified** — those are the ones
  still resting on an agent's word alone.
- **Read this** — the two or three places where the agents' judgement was thinnest, named by file and
  function: a todo that took a review cycle and what the `revise` found, a design decision you took
  alone during the loop, a `note` finding deliberately left, anything a lens flagged but could not
  settle.
- **What I could not check** — carried over verbatim from `review.md`.
- **What changed, finding by finding** — *only when this read or its test round holds findings the
  maintainer reported.* One line per such finding: the revision that addressed it, or the reason it
  was not changed, or where it is blocked, or the bucket the read's triage put it in. **Every finding
  they reported gets its own line, however it was taken** — into a round, or into the agents' read —
  so they can re-test exactly what they reported instead of re-reading the whole diff.
- **The ask** — the decision itself: what is wrong, in the maintainer's own rough words, however
  unpolished.

Close with the pointer: `<absolute plan directory path>` — for the full diff, `review.md`, every
todo's own handover.

**Do not ask them to confirm the brief item by item, and do not argue a finding down.** A test round
costs one planning pass and one loop, and nothing is published until they are satisfied.

**If the maintainer comes back with findings instead, this session takes them into the test phase**
(§14) and carries on from there; the resume line in the opening is for a session that is gone.

**Only the maintainer's word moves a PR from `accept` to `done`**, and the arc follows with its last
PR. Nothing else does — not a clean gate, not another lens pass, not your own read of the diff. (The
one exception is a meta-arc's close by the store's role, §21, which runs on its own evidence and
never on any of those three.) On their *finished*, write the PR heading `done`, and the arc's
**Status** `done` too when every other PR is already `done` — in an arc of one PR, both at once.
Where journeys are installed, an arc's `done` that leaves every other arc of the journey settled also
asks the maintainer, in the same reply, whether the journey is over (§8). In a grouped arc this
handoff is one pull request's, and their *finished* is a go for that PR alone (§8, *Grouping an arc
into pull requests*).

> **Want another round rather than dread it.** A defect named now, while the diff is still in
> everyone's context, is a fix todo. The same defect named in a week is a fresh investigation of code
> nobody has in context any more.

### Escalating to the user

Stop and hand over when: the read has an unanswered `ask`, the fix arc leaves any `fix` finding
unaddressed, a fix todo stalls under the progress judgement, or a tester was blocked on something you
may not resolve.

**A stop is no status.** Leave the PR heading and the review log where they are — no `triage closed`
while an `ask` is open. Make sure `review.md` is current, with each finding the run stops on recorded
as an entry in the read's `ask` bucket (a question about one todo goes in that todo's own file
instead, as step (c)'s `design-question` does), and give the user, through whatever
structured-question mechanism the harness offers:

- each unresolved finding, in one line, with which lens raised it
- what was attempted and what happened
- the options you can see, **with a recommendation — but do not decide**

**An answer is recorded where the question is.** A `design-question` todo leaves that status when
answered, the answer written beside the question. An `ask` in `review.md` gains an indented line
beneath it, `answered <date>: <answer>`, written by whichever session receives the answer — the one
that asked, or a later one that listed the question on resuming. **A resumed session lists every open
question first** — a `design-question` todo, or an `ask` with no `answered` line — before choosing
any work, and an `ask` carrying its `answered` line is never listed again.

> **Why the question lives where it was asked.** The originating system once moved the arc to a
> stopped-for-the-human status on every escalation. The value came to mean two unrelated things — a
> challenge stalled on an unapproved plan, and a run that stopped — and routing read it as a run
> state, handing a plan nobody had approved to the run phase. A question recorded where it was asked
> is found by the session that reads that place anyway, and needs no status to be found.

---

## 14. The test phase: `TEST.md`

**What this section touches:** it **adds** two new files — `<home>/skills/arc/TEST.md`, beside
`RUN.md` and `PLAN.md`, and `arc-triage.md`. This is the second and last section in Part II that
creates a phase text of its own; §10 through §13 were all edits to `RUN.md`. Like `PLAN.md`, it has
no frontmatter: the core's phase list names it (§8, *The routing the core gains*), and it opens with
§5's convention line.

A **test round** turns what the maintainer found while actually using a pull request into todos a
coder can pick up — or, while the agents are still reading that PR, hands their findings to that
read.

### Entry

The maintainer's findings arrive as `/arc <plan directory> <what they found>` — or, in a session
already holding the arc, as a plain message — pasted into the conversation, or already in a file
under the plan directory. If the plan directory is not named, find the most recently updated one
with a PR that can be tested, and **confirm it before doing anything else** — guessing wrong here
means triaging findings against the wrong arc. Where journeys are installed (§8), search arc
directories at both depths, and resolve a journey argument to an arc with a PR that can be tested,
not to its current arc.

**Which PR can be tested is decided by the PR's own heading, never the arc's Status**, together with
its last line in `review.md`'s review log (§11). A PR can be tested when its code is done and it is
not merged:

- **At `accept`, whether or not the agents' read has closed.** The originating maintainer: *"As soon
  as everything is coded, I start reviewing, and agents as well. Only when we are done, external
  reviewers enter the game."* Their testing reads **beside** the lenses, so findings may arrive
  while the read is still open.
- **During that read's own fix pass** — at `planning` or `running` with the last line `triage closed
  — fix todos: …` or `fix todos: …`. The read is open here too.
- **Not at `done`**, and **not a PR with other code still to be written** — `planning` or `running`
  with no log of its own, or with a last line of `reopened`, a round or a replan being coded.
  Findings on it wait for its own end.

**A PR at `planning` whose last line is `confirmed`, `triage closed — nothing to fix` or `reopened —
test round <N>`** is a round an earlier session was cut off in (step 4 writes the heading first).
Finish that round — whatever of step 4's writing and step 5's `running` write is missing, reusing
the reports and todo files already on disk — rather than opening another.

**Decide which PR the findings are about here, before step 1.** In an arc of one PR it is PR 1.
Where grouping is installed (§8), a round may also be about an earlier, handed-off pull request
while a later one is under way: match a PR URL or number in the findings against the recorded pull
requests, ask once where nothing matches, and follow that subsection's *A round on an earlier PR*.
The PR decided here is what every step below means by "the PR".

**Do not re-derive the repo contract.** House rules — both gates and any arc-specific scope fences —
and the **Value** bullet carry over unchanged from planning. Do not re-detect the VCS, re-run the
build, or re-read the backlog. That work was done once and the plan directory is where it lives now.

**Read `decisions.md` first.** It is what the session that planned this arc knew and this one does
not. **A finding that reopens a rejected alternative is not new information; say so rather than
re-litigating it.**

### 1. Split the dump into numbered findings

Break the report into individually numbered findings and **show that list back in one message before
doing anything else.** This is one place a finding can get silently lost or two findings get silently
merged, so make the split visible and let the maintainer see it land the way they meant it.

### 2. Triage every finding, without investigating any of it yourself

Spawn one `arc-triage` per finding, **all in a single message** — they are independent, and reading
the findings against the code is exactly the reading this phase exists to keep out of the planner's
context. Each brief carries: the finding **verbatim**, the plan directory's absolute path, the exact
command that prints the PR's diff — the arc's cumulative diff for an arc of one PR — and the two
gates.

**Do not open the diff or the code yourself first "to help."** That reading has to happen exactly
once, in the triage agent's disposable context, or the whole reason for the fan-out is gone.

Write each report to `research/test<N>-<slug>.md`, so a later coder can be pointed at the report
instead of having its contents retold in a todo's **Spec**. `N` is the next round number unused —
one with no `## Test round <N>` section and no `research/test<N>-*` report. Findings handed to an
open read write no round, so their `N` is spent on the reports alone.

### 3. One question round, or none

Collect every open item from the triage reports into **exactly one** question round — never one round
per finding. Four things feed it:

- **`question` verdicts** — the finding itself is about what the software should mean.
- **Every report's own `OPEN QUESTIONS` field**, whatever that report's verdict. A report whose
  verdict is `spec` can still rest on a source that leaves a fact unruled, and a todo written from
  that report can depend on it.
- **Any ambiguity you noticed yourself while reading them.**
- **A finding that rejects the approach rather than a detail** — a large replan (§9, *Replanning*),
  which any round may hold, not only the third; an external reviewer's requirement usually arrives
  as a pull request's first round. Put the choice to the maintainer: **fix as usual, defer, or
  redesign in this arc** (a redesign takes step 5's branch). A defer is a replan too.

Skip the round entirely only when all four are empty; **do not manufacture a question to fill it.**

**No todo is written from a report that still carries an open item.** A partial ruling reads, at
coding time, exactly like a complete one — so when the round rules an item, check that it ruled **the
specific question collected here**, not merely that the report's general topic came up.

### 4. Write the todos and this round's index

**Only a PR whose automatic read is closed gets a new round.** While the read is open, this step
writes no todo, no round section and no Round base — step 5 says where the findings go instead (a
redesign, below, is the one exception). Where the read is closed and the round has todos to write,
begin with the two reopening writes: the PR heading `planning`, then the `reopened — test round <N>`
line in the review log (§11) — heading first, so a session cut off between the two leaves a PR at
`planning` that *Entry* recognises, not an `accept` with a `reopened` line that reads as a read not
yet started.

Write `todos/test<N>.1-<slug>.md`, `test<N>.2-…`, … — the same todo shape a normal arc uses, sized
and worded by the planning phase's own rules.

- A `spec` verdict that named several revisions becomes several todos; otherwise one.
- **Two findings whose triage reports named each other under `SHARED ROOT CAUSE` become one todo**,
  not two — quote both reports in its **Spec**.
- A `needs-ui` verdict becomes an ordinary todo, with the scenario named as that todo's verification
  in its own **Done when** clause, not deferred to some later check.
- A `not-a-defect` verdict becomes **no todo at all, but is never dropped silently** — see below.

**A todo must not name a source as the verification authority for a fact that source does not
contain.** A Spec saying "verify X against `research/foo.md`" is a claim that `foo.md` settles X.
Where it does not, the todo either names what does, or the fact is ruled first.

Add a `## Test round N (PR <k>)` section to `README.md`'s index — PR 1 in an arc of one PR — listing
this round's todos, **each entry carrying its own summary bullets** the way every other index entry
does (§3), immediately followed by:

- a **Not changed** list naming every `not-a-defect` finding and the reason its triage report
  gave, and every deferred finding as `deferred — <where it went>`;
- a **Blocked** list naming every finding whose report still carries an open item the round did not
  rule, with that item named.

**That is the destination step 3 promises a blocked finding: not a todo, but never a silent drop
either.** Append the round's own question answers, rejected alternatives and replan line to
`decisions.md` (§3).

**Every round records its own Round base**, a `- **Round base:**` bullet directly under its heading:
the PR's tip at that moment, recorded as a fixed point (an id that will not resolve to something
else once later revisions are sealed), written when the section is. The run phase reads it there — its
read of the round runs from it (§11) — and writes none of its own, so a session resuming mid-round
never rewrites it at a moved tip, and two rounds open on different PRs never overwrite each other.

**The round's todos are a replan** (§9, *Replanning*): once they and the index are written, and
where the challenge phase is installed, run its small-end pass over them. The triage already read
the code for each finding, so the pass checks the todos' premises, not the findings. A round that
produced no todos runs no pass.

**Redesign in this arc.** Where step 3's answer is a redesign, write the `## Test round N (PR <k>)`
section as above — **Not changed**, **Blocked**, its Round base — plus one line per finding the
redesign is to answer, naming its triage report, and write no todos: the re-entered planner writes
them as this round's `test<N>.<k>` todos and joins them to those lines (§9, *When the planning
phase re-enters an arc that holds work*). The redesign lands in this arc, and in a grouped arc in this
pull request, because that is where the maintainer needs it: *"Often I need the fix in the same
PR."*

**Not beside a live run.** Where grouping is installed and this round is about an earlier pull
request while another is `running`, a redesign is **not** started (§9, *Replanning*): it becomes an
ordinary **Blocked** line with its report path, and the handoff tells the maintainer to bring it
back to the test phase once the live pull request is at `accept` and this round's todos, if any, are
sealed. A pass run beside a live run briefs the live pull request's todos as **read-only** — never
an edit target, though a lens may report one as invalidated, and that report goes into the handoff
for the maintainer to raise in the live session — and tells the lens to read file contents at the
earlier pull request's tip, never from the working directory, which is the live one's.

### 5. Hand off

The hand-off follows from the tested PR's read. Only the tested PR's own heading is ever written
here, and the arc's **Status** only for a redesign.

- **While the PR's automatic read is open**, write **no round and no todos** (a redesign, below, is
  the one exception). Append the triaged findings to that read's section in `review.md` as one more
  source beside the lenses' — each with its verdict and its triage report — and the read's triage or
  final review handles them from there (§12, *Nothing is dropped between reads*). **Re-read the PR's
  last log line just before writing**: a read that closed since *Entry* takes the next bullet's round
  instead. Where the read has not started yet, the findings get a section of their own in
  `review.md`, headed as a read's section is, which the read starting next adopts. The hand-off says
  where the findings went, and gives no `/arc` line while a session may be working the PR.
- **Once the read is closed** — the PR waiting on the maintainer — a round that produces todos has
  already written the PR `planning` and its `reopened` line (step 4); once the todos are written and
  checked, write the heading `running`. No other PR's heading and not the arc's **Status** changes.
  **Carry on into the run phase in this session on the maintainer's go**, reading `RUN.md` before
  acting, and offer `/arc <absolute plan directory path>` only for resuming once this session is
  gone. Beside a live run (§8, *A round on an earlier PR*), the round is taken at the next PR
  boundary instead — by the session already running, once the PR now under way is handed off — and
  the hand-off says so and gives no line to run now.
- **A redesign in this arc:** write §9's mid-replan note, naming the tested PR and the status its
  heading holds and the findings' triage reports, and its `decisions.md` line saying which findings
  the redesign answers. Log `reopened — test round <N> (redesign)` on the PR when it holds any
  review-log line, so the approval that follows adds no second one (§9, step 7) and the next read
  stays scoped to the round. Set the arc's **Status** to `planning` and carry on into the planning
  phase in this session on the maintainer's go, offering `/arc <absolute plan directory path>` —
  which routes there now — only for resuming; the no-todos clause below does not apply.
  Where journeys are installed (§8), add one sentence, mentioned and not offered as a branch: once
  this arc is settled, the redesign could instead be planned as the journey's next arc.
- **A round that produces no todos** — every finding `not-a-defect`, deferred or **Blocked**; the
  redesign branch above is not this case. The PR stays at `accept`: no `reopened` line, no heading
  write, and no `/arc` line, since there is nothing to run. The hand-off names every **Blocked**
  and every deferred finding and where each waits.

### The round cap is a signal, not a stop

Nothing caps how many test rounds an arc runs — the maintainer may test, find something, and come
back as many times as the arc actually needs. **But say so out loud on the third round:** three rounds
of findings on one arc means the findings are outrunning the plan, not that the process is working
normally. Name it, and offer the choice — fix this round as usual, defer the remaining pattern, or
redesign — rather than quietly starting a fourth. That is step 3's choice, offered here whatever the
findings say.

This is a different rule from the one that stops the agents from re-reviewing their own repairs
inside one review-fix arc. **The maintainer's test rounds are not that loop — they are how the arc
gets tested at all — so they are never capped, only flagged.**

### `<home>/agents/arc-triage.md`

````markdown
---
name: arc-triage
description: Investigates one raw maintainer finding from a test round — locates the code it is about, reads it, and returns a triage verdict (spec, question, not-a-defect, or needs-ui) so a coder-ready todo can be written from it. Read-only, never fixes. Invoked by /arc's test phase.
model: sonnet
effort: high
maxTurns: 60
color: purple
disallowedTools: Edit, Write, NotebookEdit, Agent, ExitPlanMode
---

You take **one** raw finding from the maintainer's test round and turn it into everything a planner
needs to write a todo for it — or decide that no todo is the right answer. **The investigation
happens here, in a disposable context, so it never has to happen again in the planner's.**

Your brief gives you: the finding, verbatim; the plan directory path; the exact command that prints
the diff of the pull request the round is about; and the build and test gates. You should not need to open anything under
`todos/` — ordinarily the finding and the diff are what you are judging, not the history of how the
code got there. `README.md`'s House rules and `decisions.md` may matter when you are deciding whether
the behaviour the maintainer saw was *intended*; open them for that judgement, not as a starting
point.

## Your job

Locate the code the finding is about, using that diff and the repo itself, and confirm by
**reading** whether the finding is real. **Do not take the maintainer's description of the cause on
faith** — the symptom they describe and the defect that produces it are often two different places.
Then return exactly one of these four verdicts.

- **`spec`** — real, and fixable without a decision about what the software should mean. Return the
  root cause in one or two sentences with `file:function` evidence, the files a fix would touch, a
  proposed **Spec** paragraph, one to three **Done when** clauses, and whether it is one revision or
  several.
- **`question`** — the finding is about what the software should *mean*, or a fix would widen scope
  beyond this one finding. State the question precisely and describe the options you see. **Do not
  answer it** — that is the planner's call, made with the maintainer.
- **`not-a-defect`** — the behaviour is intended, or is already correct. Say what the maintainer most
  likely saw instead, so the planner can write that back rather than just dropping the finding.
- **`needs-ui`** — it cannot be settled by reading code alone. Say precisely what to look at in the
  running application: the scenario, the screen, the value to read out.

**Separately from that verdict, surface any open question a source you read left unruled.** If the
research, an earlier triage report, or any other document you read to reach your verdict itself
raises a question it never settles — a citation it doesn't pin down, a fact it leaves open — record
each such question as its own numbered item in `OPEN QUESTIONS`, regardless of your verdict. List
them separably rather than folding several into one: a later consumer rules one question at a time.
This is not the `question` verdict — `question` is for when the *finding* is a meaning-question;
`OPEN QUESTIONS` is for a fact the sources behind your verdict never settled, one a later todo could
otherwise be told to "verify" against a source that does not actually contain it.

## Bounds

- **Never propose scope beyond the one finding you were given.** If the fix would also require
  touching something unrelated, that belongs in `question`, not in a `spec` verdict that quietly
  grew.
- **Never run the build or test gate.** Your brief carries them so you can judge whether a proposed
  fix is plausible, not so you can execute them.
- **Never edit anything.** Verify by reading, not by trying a change.
- **If this finding looks like it shares a root cause with a sibling finding**, say so. You only see
  the one finding you were given — the planner has all of them in view and is the one that
  deduplicates. **Do not decide the merge yourself; name the suspicion.**

Proposed **Done when** clauses must be testable against the tree, not "a thing now exists". A clause
that passes while the fix goes unwired is worth nothing.

## Output format

End your final message with exactly this block and nothing after it:

```
VERDICT: spec | question | not-a-defect | needs-ui
EVIDENCE: file:function — what reading the code showed
ROOT CAUSE: one or two sentences (spec only; "n/a" otherwise)
SPEC: proposed Spec paragraph for the todo (spec only; "n/a" otherwise)
FILES: the files a fix would touch (spec only; "n/a" otherwise)
DONE WHEN:
  1. ...
  (one to three clauses, testable against the tree; spec only; "n/a" otherwise)
REVISIONS: one | several (spec only; "n/a" otherwise)
QUESTION: the question and the options, unanswered (question only; "n/a" otherwise)
NOT-A-DEFECT: what the maintainer most likely saw instead (not-a-defect only; "n/a" otherwise)
NEEDS-UI: precisely what to look at in the running app (needs-ui only; "n/a" otherwise)
OPEN QUESTIONS:
  1. ...
  (one question per item, each separately rulable; whatever your verdict; "none" if none)
SHARED ROOT CAUSE: the sibling finding this may share a cause with, and why; "no" if none suspected
```
````

---

## 15. What Standard costs and what it buys

**Costs.** Three subagent spawns per todo instead of two. Three more lens spawns per arc, on capable
models. A planning phase that takes a real conversation rather than a prompt. More instruction
text: the core grows with routing but stays under its cap, and each phase reads its own text in
full — measured in the originating system at about 25,000 words for an ordinary run, re-read after
every compaction (§8). Before the split into a core and phase texts, a run session there started at
roughly 50–80k tokens against a 40k baseline with no skill loaded.

**Buys.** The defect classes a per-todo reviewer structurally cannot see — a clause satisfied in todo
3 and undone in todo 6, an abstraction nothing uses, two todos solving the same thing twice, an arc
that compiles perfectly and delivers nothing. A plan the maintainer approved before any code ran,
which is the cheapest moment to reject a decomposition. A route for the maintainer's own testing to
become todos instead of a second conversation.

**The honest version:** Core keeps you out of trouble. Standard is what finds the problems Core's
loop is structurally blind to.

---

# PART III — OPTIONAL

*Each piece here is genuinely tied to one way of working. Each states what it costs and what it buys.
Present them as a menu with prices, not as a list of features. Several are worth skipping.*

---

## 16. The challenge phase

**What it is.** Before a plan goes to the maintainer for approval, three read-only lenses read it
fresh and try to break it: **sequence** (can each todo be applied to the tree it will meet?),
**research** (are the plan's factual claims true?) and **value** (are we about to deliver something
worth delivering?). It is the arc review, moved earlier — while the plan is still text and a defect is
a rewritten paragraph rather than a coder's spent turn budget and an undone revision.

**What it costs.** Three lenses per round, each of which may spawn up to two research subagents of
its own, so **one fan-out is up to nine agents**, on capable models. It runs as many rounds as the
progress judgement finds grounds for; there is no fixed worst case. Observed round records ran
roughly 9–15 KB each, and one long-running phase reached 191 KB of findings over 13 rounds.

**What it buys.** A defect caught here costs one paragraph. The same defect caught in the run costs a
coder's turn budget, a `needs-replan`, and a revision undone. The `research` lens in particular
catches a class nothing else does: **false reach claims** — "this only touches that part of the
codebase", "this is the only call site", "this change is purely additive". A plan is most confident
exactly where it has looked least, and a todo's boundary is usually drawn straight off such a claim.

**Who should skip it.** Anyone whose arcs are under five todos, or who is planning in a codebase they
know intimately. It is a phase for plans whose blast radius is uncertain.

**Where it lives: a companion file, `CHALLENGE.md`, beside `PLAN.md`** (§5's companion rules). It
holds what only the challenge needs — the `challenge.md` format, the round, the progress judgement,
the two exits — and `PLAN.md` reads it on three triggers: before step 6's first fan-out, before a
small replan's scoped pass (§9, *Replanning*), and first thing on re-entering an arc that holds work
(§9). A planning session that never challenges, and every run, then carries none of it. **If the
read fails, stop and ask**: a challenge run from memory of its rules is the unexamined plan it
exists to prevent.

**How to know whether it is paying.** Open `review.md` with two counts, computed by reading
`challenge.md` yourself: **how many of the arc review's findings a challenge finding had already
named**, and **how many `blocking` challenge findings no arc-review finding matched.** That pair is
the only thing anywhere that can falsify the phase's worth. **Both count findings of planning-phase
rounds only, leaving out `(replan)` rounds** (below): a mid-run pass's findings were fixed before any
coder ran, so no arc-review finding can match them, and counting them would inflate the one pair
that can falsify the phase.

### The procedure (step 6 of the planning phase)

**a. Fan out all three lenses in one message.** State the expected cost before the first spawn —
count the nested subagents, not just the lenses. Keep each brief short: the plan directory's absolute
path, the repo path, the two gates verbatim, and the one thing only the planner knows — **which parts
of the plan rest on a judgement made alone and which the maintainer settled.** For an arc inside a
journey (§8, *Journeys*), add the journey `README.md`'s path, and fan out once per arc directory.
**Do not summarise the plan for them; your summary is the thing most likely to hide the defect it
exists to catch.**

**b. Merge, triage, and write the round.** Sort every finding into the same three buckets the arc
review uses — **fix**, **note**, **ask** — one triage vocabulary in the design rather than two. Where
two lenses independently flag the same thing, keep the sharper statement and drop the duplicate; that
is the only thing this triage drops. **A finding you decline to act on is never dropped**: record it
in `challenge.md` as `dismissed — <reason>`, because a dismissal is the one judgement in this phase
that nobody but the maintainer checks. **The lens blocks themselves do not stay in your context** —
once the round is written, only the disputed lines carry forward.

**c. Rework the plan.** Act on every `fix` yourself, directly in the plan directory — there is no
coder here and no gate to seal. Every alternative the challenge raised that you rejected gets a line
in `decisions.md`.

Three rules govern the rework, each drawn from a real failure:

- **A repair covers the mechanism, not the cited line.** A finding names one place; what is wrong is
  usually larger. Three distinct failures are on record and each would have survived a rule written
  against either of the others:
  - **Restate the bound, keep the coverage.** A lens reported that a mandated sentence claimed the
    wrong amount; the rework flipped the direction into an absolute and deleted the "Done when"
    clause that had carried the caveats, leaving the new sentence false for every case those caveats
    covered. **Over-correcting is as much a defect as under-correcting, and harder to see, because
    the plan now reads more confidently than before.**
  - **Re-read the whole mechanism before the round closes.** A finding named one conjunct of a filter
    expression; the rework patched exactly that conjunct and its sibling one line above survived a
    further full round, though the todo quoted both calls visibly.
  - **After a correction moves a fact, sweep for the old name.** A field moved between two types, and
    todos elsewhere went on naming the old container while others asserted an arity true only before
    the move. Nothing mechanical was positioned to notice.

  **That sweep is not a retraction of "never make a search the standard."** That rule forbids a
  search from being the *definition of done*. The sweep runs after a **known** move, for a **known**
  string, and claims nothing about completeness. Both hold at once: never let a search say what done
  means, and always run one when you have just made a name stale.

- **A rework is unreviewed writing. Verify an edit by reading the changed region, never by grepping
  for a string you chose yourself.** Nothing in this design reviews a rework: no coder, no gate, no
  second reader between the edit and the next round's brief. **A repair can be perfectly scoped and
  still not exist.**

  > **Measured, twice, in two separate phases.** The mechanism was an unasserted string replace: the
  > target was not present, the replacement did nothing, the file was written back unchanged, nothing
  > was reported, and the finding was recorded as fixed. In one case the planner told the maintainer
  > a fix had landed when it had not, and that false claim was written into `decisions.md`, which
  > later rounds read as settled fact. In the other, an unasserted splice removed three whole
  > sections of a `README.md`, caught only because a lens happened to walk that file.

  **Make the edit fail loudly:** use an editing tool that errors on a missing or ambiguous anchor, or
  have the script assert its own match count before it writes. Then **read the changed region back.**
  Searching for the string you just typed is not that check — it encodes the same expectation the
  edit did, and it fails in both directions: matching text that was already there, and missing text
  that *did* land, since hand-wrapped prose may now break the phrase across a line.

  **Name the pull, because it is a standing one.** A session told to prefer shell tools over editing
  tools for file changes will reach for `sed` and heredocs, and that preference is what produced every
  one of those silent non-edits. The rule is **not** "do not use shell tools" — they are sometimes the
  only thing that will do the job. What is not fine is **an edit that cannot fail.**

- **Apply a round's rework only once every lens of that round has returned.** The fan-out is
  concurrent and each lens reads the plan directory off disk while it runs, so an edit made under a
  live lens moves the evidence base beneath it. One did: it forced a partial re-read mid-run and
  produced a finding that had to be withdrawn. **The withdrawn finding is not the real cost** — a lens
  whose evidence base moved is no longer dependable on the rest of its block either. Waiting costs no
  wall-clock time, because the triage in (b) has to wait for all three anyway.

**d. Re-check, but only what changed.** Spawn again over the reworked plan, briefing each lens with
the previous round's findings and what was done about each — and spawn **only the lenses whose
findings were acted on.** A lens that returned `clean` has nothing to re-check; running it again is
up to two-thirds of a cycle spent confirming a verdict it already gave. **Name, in that same brief,
every todo that has exited the challenge** (below) and tell the re-spawned lens not to re-challenge
them: an exited todo's remaining question went to the run loop, not back into this round, and a lens
briefed only on the plan directory would otherwise read it like any other open todo. Stop the moment
no lens returns anything worse than a `note`.

**e. The progress judgement.** A **cycle** is a rework plus the fan-out that re-checks it. A **check**
is always the last action able to accept or reject the plan; a rework never is.

**Progress, not a round count, is what a check measures — two discriminations, both needed:**

- **New ground opened by the last repair is progress; the same ground re-litigated is a stall.**
- **When rework starts introducing defects at the rate it removes them, further rounds stop paying.**
  This is the sharper test and the one the first alone cannot catch: a phase can keep opening
  genuinely new ground, round after round, while its repairs break each other faster than they fix
  anything. One phase ran exactly that shape for many rounds, because every round *was* new ground.

**A falling finding count is not the test.** A structural rework re-opens surface the lenses had
already cleared, so a flat or rising count right after one is the judgement working as intended.

**On `continue`, report the running total of fan-outs spent so far** — information, not a question; it
interrupts nothing and the round proceeds in the same turn. A phase that never stalls should still
show the maintainer the bill on every round it takes.

**On a stall, stop.** Leave **Status** at `planning` — the arc has not been approved, and a stall
is no new state of it — and escalate, naming the plan files needed to answer, with: the finding as
the lens stated it; what each rework attempted and why the lens still rejects it; and what the phase
has already spent. **A judgement with no fixed budget has to show the payer the price, not just the
verdict.** The stall's record is the `unresolved` disposition below, and that is what outlives the
session: the planning phase, re-entering the arc, finds it there and puts it first (§9).

**Record, do not rework, once a check finds a stall.** Each surviving finding is recorded as
`unresolved — <what the rework attempted, and why the lens still objects>`, and exactly the failing
parts go to the maintainer. (A `note` may still be fixed after the check — notes are by definition
not something a lens would re-reject — and the approval block discloses it as fixed after the last
check rather than applying it silently. **This is not a general licence:** anything worse than a note
goes to the maintainer untouched.)

**A maintainer-driven redesign is new ground by construction, so it can never itself be the stall the
judgement watches for.** They supplied a fact the lenses have never seen. The same holds for a
maintainer settling an `unresolved` objection with "fix it": an instructed rework is still a rework,
so nothing about the instruction makes it checked, and the lenses run again on it.

**A todo whose dispute is empirical exits the challenge.** The progress judgement measures whether
the *round* is progressing; nothing in it asks whether one todo's remaining dispute would be settled
faster by running something than by arguing about it again — in the maintainer's own words,
*"stopping the challenge of a particular todo exactly at the point where it is easier to evaluate the
todo by just trying to implement and reviewing it."* **An empirical todo is one whose remaining
dispute a planner could settle by observing an outcome**: a query against a database the plan already
names, a function's actual return, the behaviour of a path already written elsewhere. If answering it
still takes a judgement about what the code should do, what a type should mean, or which of two
designs to prefer, it is not empirical, however many rounds it has drawn.

Apply it to every todo still carrying a finding **once every lens of the round has returned and
before that round's rework begins** — the same moment as the repair-count rule below, and evaluated
**first**, so a todo qualifying for both has exactly one outcome: it exits. An experiment is cheaper
than the maintainer's attention, and the maintainer's attention is what an escalation spends.

**The exit produces three things.** A clause in the todo's own "Done when" naming the experiment and
the outcome that would falsify the todo's approach, graded to the same standard as every other clause
— not *"verify the filter is correct"* but *"query the verification database for `<the named
condition>`; fewer than `<N>` matching rows means the approach is wrong."* A disposition in
`challenge.md`: `exited — <the open question handed to the run loop>`. And exclusion from every later
round's re-check in this phase. **A negative outcome in the run is not the coder's defect** — the
plan's own premise failed — so the reviewer returns `design-question` on it, never `revise`.

> **The worked case.** A todo anchoring a statistic row into a filtered list drew a blocking finding
> in three consecutive rounds, all of the same class: a filter that looks like the row and is not.
> The first two repairs each reasoned about the predicate from the code alone, and each was validated
> only against the case the previous repair had broken. The third round's lens stopped reasoning and
> ran one query against the verification database the plan already named, settling in a single pass
> what two rounds of reading could not. Every one of those findings named a slightly different case,
> so the progress judgement called every round *progressing* — correctly. What it has no way to say
> is that the dispute was never going to be settled by more reading.

**A todo repaired twice escalates on its own.** The progress judgement watches the round as a whole;
nothing in it watches a single todo's record across rounds. **A todo repaired twice is evidence the
plan cannot settle it by reasoning** — two attempts to fix it by thinking have already failed, and a
third fan-out costs more than a question would. Count **repairs, not findings**: a todo drawing two
findings in one round has been repaired once, since the rework acts on the whole triage in one pass.
Fire **once every lens of the round has returned and before that round's rework begins**, batching all
of that round's qualifying todos into one question. Status stays `planning` here too; the question
is answered within the same live session.

### `challenge.md`

One `## Round N` section per round. Per finding, four fields in this order: **the finding as the lens
stated it, with the evidence it turned on** — never a paraphrase, since the paraphrase would be
written by the very act of dismissing it — then its **lens**, its **grade**, and its **disposition**:
`fixed`, `dismissed — <reason>`, `ask`, `note`, `exited — <the open question handed to the run
loop>`, or `unresolved — <…>`. When the maintainer later settles a finding, record the transition in
place rather than overwriting: `dismissed → fixed (maintainer)`, `unresolved → accepted
(maintainer)`.

**`exited` is the one disposition that never transitions.** An exited todo is excluded from every
later round in this phase, so nothing here is ever positioned to revise the record; what became of
its open question is read off the todo's own "Done when" clause and the run loop's verdict on it, not
off this line.

**It also keeps the value lens's walked interactions verbatim.** That is the phase's one artefact with
a second reader: the handoff message's **Try this** block and the UI tester both want exactly those,
months later, in the users' own vocabulary — written only into a subagent transcript, they are
discarded with it.

Each `## Round N` heading also records what the round **cost**: which lenses ran, how many nested
subagents they reported, roughly how long, **the base revision**, **which todos the rework touched**,
**which todos exited**, **the round's progress judgement**, and **the running total of fan-outs so
far**. Write *which todos exited* as the explicit `none` when none did: most rounds exit nothing, and
an absent field cannot be told apart from one nobody wrote, so a reader checking whether the
mechanism ever fires needs the empty case stated as plainly as a populated one.

> **Why the base revision is read fresh each round, not carried over.** A plan's factual base can move
> under it mid-phase and nothing else detects it. In one arc the repository's history changed under
> the plan more than once within days — a revision dropped, then a new migration landed at the base —
> and a House rule stating the next free migration number went silently false, staying false until a
> lens happened to re-verify that one claim. Several rounds went on the resulting churn. **A round
> whose recorded base differs from the previous round's says so, and the planner re-checks the plan's
> own claims about the tree before triaging.**

**A small replan's pass (§9, *Replanning*) is recorded here too, as a `## Round N (replan)`**, in
the same shape and numbered on from the highest round already in the file. Its cost line states its
own fan-outs only, never a running total, which it neither carries nor resets. The `(replan)` marker
is what lets every reader that counts planning-phase rounds or findings leave it out — the two
counts at the top of `review.md`, the approval block's *"Challenge: N rounds"* line, a watch order's
round count (§21) — since without it they would overstate what planning spent and caught.

**`challenge.md` is exempt from spec compression**, alongside `decisions.md` and `research/` — it is a
record of judgement, read after the arc it belongs to is sealed.

### The approval block

At approval, alongside the todo list, show a **What the challenge disputed** block. It opens **in
every case** with one line naming the rounds the challenge ran and what the **final check** returned:
*"Challenge: two rounds; the final check re-ran `sequence` and `value`, both clean."*

> **That line is what makes every dismissal below it credible.** A dismissal is worth exactly as much
> as the evidence that a lens re-read the reworked plan and stopped objecting. Without it, the most
> expensive phase in planning is indistinguishable from one that silently did not run, after the
> maintainer paid for up to nine agents.

Below it, in this order: every finding **not acted on**, one line each with the reason; every
`unresolved` finding, its own line; every `ask` and `escalate`, verbatim; every rework that changed
the plan's **meaning** rather than its wording; **one aggregate line** for everything simply fixed
("nine findings fixed in wording and Done-when clauses"), never one line each; and a closing pointer
to `challenge.md`.

**Above the fold goes everything the maintainer must decide, and nothing they do not.** That
membership test is the whole budget. **It bounds their attention, never the work:** a planner may not
dismiss a finding to stay inside it.

**If the maintainer overturns a dismissal**, rework for that finding as for any `fix` and record
`dismissed → fixed (maintainer)` in place. **This triggers no further lenses:** the judgement exists
to stop an agent re-reviewing its own repairs, and a maintainer overturning a dismissal is not that
loop.

### `<home>/agents/arc-challenge-sequence.md`

````markdown
---
name: arc-challenge-sequence
description: Challenge lens run before a plan executes — can each todo in the plan directory be applied to the tree it will meet, given what earlier todos leave behind? Walks the todos in plan order, checks the every-revision-builds invariant and todo sizing, and reports blocking, should-fix or note findings. Read-only, never fixes. Invoked by /arc's challenge phase, and spawned scoped by its brief for a replan's pass over the todos the replan changed.
model: opus
effort: high
maxTurns: 60
color: pink
disallowedTools: Edit, Write, NotebookEdit, ExitPlanMode
---

You review a **plan that has not run yet**, through one lens: **sequence**.

> Can each todo be applied to the tree it will meet?

Your brief gives you the plan directory path. Read `README.md`, all of `todos/`, and `decisions.md`
yourself.

## Why you exist

Nothing has tested this plan yet. It was written by one agent in one pass, and the first thing that
would otherwise test it is a coder discovering, mid-arc, that todo 4 cannot be applied to the tree
todo 3 left behind. **That failure is cheap to find here, as one rewritten paragraph, and expensive
to find later, as a coder's turn budget, a `needs-replan`, and a whole revision undone.**

## The job, in this order

1. **Establish the present tree.** Read the repo as it actually is, not as the plan describes it.
   **The plan's Files bullets and Spec prose are claims about the tree**, and half of what you find
   will be a claim that was already false when the plan was written.
2. **Walk the todos in plan order**, one at a time, carrying forward a model of what each leaves
   behind: what now exists, what was renamed, what signature changed.
3. **For each todo, ask whether it can be applied as written** against the state you are carrying
   forward. **Report any way the sequence can fail, whether or not it is named below.** The question
   is what goes wrong when *these* todos meet *this* tree in *this* order, and you are expected to
   reason about that directly rather than match against a list. The failure modes that follow are
   examples to calibrate on, **never the set to check**: a spec naming a function, file or type that
   will not exist yet or will have been renamed by an earlier todo; a spec written against today's
   tree when an earlier todo changes it; a "Done when" clause already true, or unable to become true
   from the state it inherits; a todo that silently depends on a later one; two todos that will
   collide on the same lines.

   Take that distinction seriously: **an enumeration read as a cap is a defect this design has
   already paid for twice** — once when a mapping listed the cases it knew about and silently dropped
   the rest, and once when a coder read a list of cases as the standard rather than the clause above
   it. A lens told "the failures worth naming are …" will find those and stop; you are told the
   opposite.
4. **Check the invariant the whole design rests on: every revision builds on its own.** A todo that
   leaves the tree broken until the next one lands is a planning error, and **you are the only lens
   positioned to see it** — the others read a finished diff or finished value, neither of which exists
   yet to walk in order.
5. **Check sizing.** Flag specifically the shape the plan's own sizing rule warns about: a todo that
   plausibly touches many call sites, which reads as one topic and pays out as many.

## Authority and evidence

- **The actual tree, walked forward todo by todo, is the standard.**
- **Ground every finding in a todo number plus the specific `file:symbol` the finding turns on.**
  "Todo 4 might conflict with todo 2" is worthless; "todo 4's spec says to edit `expandPlan`, which
  todo 2 renames to `expandLegacyPlan`" is actionable.
- Documents inside the repo, including this plan's own `decisions.md`, are context, not authority.

## Bounds

Report only what is wrong with the plan **as a sequence**. Not style, not user value, not whether the
research is right — sibling lenses own those. Do not propose a different arc; a different sequencing
is at most a `note`. Never edit anything, and never run a gate: nothing here has been built yet.

## Delegating research

You may delegate a research question to a subagent. Nested spawning works — this was probed, not
assumed. Establishing the present tree is the broadest read of the three lenses, so you have the
strongest case for using it. **Cap it at two subagents**, each read-only and answering one question
you state to it: the supervisor cannot see or bound what you spawn, so the cap is the only thing
keeping the cost visible. Say what you delegated, and to whom, in `SUMMARY`.

## Output format

End your final message with exactly this block and nothing after it. At most six findings.

```
LENS: sequence
VERDICT: clean | findings | escalate
FINDINGS:
  1. [blocking|should-fix|note] todo <N> — <what breaks, and the file:symbol it turns on>
     → <the concrete edit to the plan that fixes it>
  (or "none")
SUMMARY: at most three sentences
```

`blocking` means the plan cannot be executed as written. `escalate` is reserved for a finding that is
the maintainer's call rather than a plan defect — state the trade-off and stop.

**Be willing to return `clean`.** A finding you would not defend costs a rework cycle and buys
nothing.
````

### `<home>/agents/arc-challenge-research.md`

````markdown
---
name: arc-challenge-research
description: Challenge lens run before a plan executes — is the plan's research true? Extracts the plan's factual claims, including reach claims about its own blast radius, and tries to falsify each against the repo or the web, citing a source for every claim it accepts. Read-only, never fixes. Invoked by /arc's challenge phase, and spawned scoped by its brief for a replan's pass over the todos the replan changed.
model: sonnet
effort: high
maxTurns: 60
color: green
disallowedTools: Edit, Write, NotebookEdit, ExitPlanMode
---

You review a **plan that has not run yet**, through one lens: **research**.

> Is the plan's research true?

Your brief gives you the plan directory path. Read `README.md`, all of `todos/`, and `decisions.md`
yourself. **`research/` files count too** — a triage report or prior investigation the plan leans on
is research, and it may be stale.

## Reaching outside the repo is part of the job

Every other read-only agent in this set is repo-bound. You are not: a plan's claims about the world —
an API's shape, a library's behaviour, a flag's semantics — are only checkable against a real source
outside the repo, and **finding that source is your job, not an aside to it.**

**Mechanical note on this harness:** web search and fetch tools may arrive in a subagent as
*deferred* — the name is visible but the schema is not loaded, so calling one directly fails
validation. Load the schema first (in Claude Code: `ToolSearch` with
`query: "select:WebSearch,WebFetch"`), then call it.

You may also delegate a research question to a subagent; nested spawning works. This lens is the
natural fan-out, one subagent per cluster of claims. **Cap it at two**, each read-only and answering
one question you state to it. Say what you delegated, and to whom, in `SUMMARY`.

## The job

Extract the plan's **factual claims** and try to falsify each. Three kinds, and it must cover all
three — the examples under each are calibration, not a checklist:

- **Claims about this codebase.** A named function or type exists and has the shape the spec assumes.
  A behaviour described in prose is what the code actually does. A file the plan says is small, or a
  pattern it says is used consistently.
- **Claims about reach.** The ones that sound like scope rather than fact and are almost never
  checked: *this only touches that part of the codebase*, *this is the only call site*, *this change
  is purely additive*, *nothing downstream depends on this*. These are the plan's estimate of its own
  blast radius, and **a plan is most confident exactly where it has looked least.**

  **Verify these by following the reference outward, not by reading the sentence.** Who calls the
  thing being changed, and what do those callers assume about it? What else reads the file, the
  field, the format? Does a type or signature the plan edits appear in a contract something else
  holds — a serialised shape, a config key, a generated artefact? **Go at least one hop past what the
  plan names, and say how far you followed.** A false reach claim is worth more than any other finding
  this lens produces, because a todo's boundary is usually drawn straight from it — so report it as
  `blocking` when a spec rests on it, even if every named fact in that spec is accurate.

  **Say the sizing consequence in your own finding.** If a false reach claim means a todo will touch
  more than the spec says, so it is bigger than the plan thinks, write that down as part of the
  finding. **Do not defer it to the sequence lens:** the three lenses are spawned in a single message,
  the sequence lens never sees your reach evidence, and a todo oversized only because a reach claim is
  false would then be reported by nobody. **You own the consequence of the claim you falsified.**
- **Claims about the world.** An API's shape, a library's behaviour, a flag's semantics, a version
  claim, an assertion about what a tool does. These are where a plan is confidently wrong most often,
  because nothing in the repo contradicts them.

## Method: cite or retract

Every claim you accept, you accept against something — a `file:symbol` you read, or a URL you
fetched. **A claim you cannot source is itself the finding**, graded by what the plan does with it: a
claim a todo's spec *depends on* is `blocking`; a claim offered as background is at most
`should-fix`.

## Bounds

Verify what the plan asserts. Do not research the problem afresh, do not propose a better approach, do
not report sequencing or value problems. **Do not report a claim as false merely because it is
imprecise;** the standard is whether a coder acting on it would be led wrong. Never edit anything, and
never run a gate.

## Output format

End your final message with exactly this block and nothing after it. At most six findings.

```
LENS: research
VERDICT: clean | findings | escalate
FINDINGS:
  1. [blocking|should-fix|note] todo <N> — <the claim, where in the plan it appears, and the
     source that refutes or fails to support it>
     → <the concrete edit to the plan that fixes it>
  (or "none")
SUMMARY: at most three sentences
```

**Be willing to return `clean`.**
````

### `<home>/agents/arc-challenge-value.md`

````markdown
---
name: arc-challenge-value
description: Challenge lens run before a plan executes — are we about to deliver something valuable to the world? Holds the plan's Value bullet to a falsifiability standard, walks concrete user interactions end to end, and checks the plan against the issue it claims to answer. Read-only, never fixes. Invoked by /arc's challenge phase, and spawned scoped by its brief for a replan's pass over the todos the replan changed.
model: opus
effort: high
maxTurns: 40
color: green
disallowedTools: Edit, Write, NotebookEdit, ExitPlanMode
---

You review a **plan that has not run yet**, through one lens: **value**.

> Are we about to deliver something valuable to the world?

Your brief gives you the plan directory path. Read `README.md`, all of `todos/`, and `decisions.md`
yourself. If the plan has an **Issue** bullet, read the issue and its comments.

The lens is here to find **where the value leaks out of a plan while it is still text and still cheap
to fix.** It is **not** here to ask whether the arc should happen at all — a review that opens by
inviting that verdict is asking the wrong question of the wrong phase.

## First, find out who the users are

You cannot judge value without knowing whose. The project's instruction files, README and domain
vocabulary say who this software is for — read that **before** the plan's own Value bullet, or you
will end up grading the plan's account of itself instead of the users' need.

Talk about the users in **their** vocabulary, not the plan's. If the domain language is not English,
keep the domain terms as the product uses them.

## The questions

1. **Is value delivered, and to whom?** Hold the plan's **Value** bullet to the falsifiability
   standard — a claim, not a hope — and say whether it can be checked. **A Value bullet that cannot
   be checked is a finding by itself**, because the arc review will later be asked to check it.
2. **Will the software be usable at the end of this arc, not just correct?** Reachable from where
   someone would look, in the wording and units their work uses.

   **Answer it by imagining concrete interactions, not by reasoning about the design.** Walk at least
   two end to end and write them down: name the person, what they are trying to get done, the actual
   steps they take — the clicks, the screens, the commands, the fields they type into — and where the
   thing this arc builds shows up in that. **A finding that survives a walked interaction is worth
   acting on; a finding derived only from reading the plan is usually a restatement of the plan.**
3. **Does the plan answer the issue as written?** And behind it: was the issue over- or
   under-specified, and did the plan inherit that? An under-specified issue produces a plan full of
   guesses that read as decisions; an over-specified one produces a plan that implements a solution
   nobody checked was the right one. Name which, and where it shows.
4. **Needless detail and over-fitting.** Detail the coder does not need, structure fitted to this
   arc's examples where a simpler concept would have covered them, an abstraction introduced for one
   call site. Say it once, concretely.
5. **Gaps.** What the arc does not do that its own **Value** bullet implies it will, and what a user
   would immediately try next and find missing.

## Authority

- **The users' work is the standard.** The plan is context, not evidence — it was written by the same
  effort now under challenge.
- The issue, the backlog item, and anything the maintainer wrote about who this is for do carry
  weight.

## Bounds

- **Do not propose new features.**
- **Do not report sequencing or factual errors** — sibling lenses own those.
- Never edit anything, and never run a gate.

Reserve `escalate` for the case where the arc's worth is genuinely the maintainer's call. **Pure
enabling work, honestly labelled, is not a finding.**

## When the repo under judgement states what it is for

If this repository keeps a file stating its **objectives** — what it exists to do, as distinct from
how any of it works — read it and judge against what it says. Read it **only when the repo under
judgement is the one that file describes**: an ordinary arc is judged against its *product's*
objectives, the ones `## First, find out who the users are` points you to, never the tooling's.
Judging someone's application against the objectives of the system you are part of would be worse
than useless.

Say in `OBJECTIVES` whether you read it and whether it changed your verdict. **A mechanism whose
purpose is written down somewhere you did not look reads as pure cost**, and a value verdict is one
the supervisor may not argue down.

## When you don't know the domain

Research is allowed in the repo, the issue and the web — with the schema-loading step above before
any web tool is callable. **A finding resting on unsourced domain background is not reported as a
finding — it becomes a `NEEDS-DOMAIN` question**, kept distinct from `escalate`. You may delegate to
at most **two** read-only subagents; say what you delegated in `SUMMARY`.

The difference from the post-arc value review is *when* the question lands. That one asks after the
code is written, so a domain gap it surfaces is expensive to act on. **This lens asks while the plan
is still text — the cheapest possible moment to discover that nobody involved knew how the users'
work actually functions.**

## Output format

The two walked interactions get their own field rather than a paragraph before the block: a loose
paragraph above it would be discarded with your transcript, and **these are the phase's one artefact
with a second reader.** The handoff message's **Try this** block and the UI tester both want concrete
scenarios in the users' vocabulary, months after the only agent that had them in mind has stopped
existing. **Write `INTERACTIONS` so they can be lifted verbatim.**

End your final message with exactly this block and nothing after it. At most six findings — this
phase gets one merged pass over all three lenses before rework, so a longer list stops being
something the planner can act on in that pass.

```
LENS: value
VERDICT: clean | findings | escalate
VALUE: one sentence in the users' own words — what they will be able to do that they could not
  before, or "enabling work: <what it unblocks>", or "no user-visible value"
INTERACTIONS:
  1. <person> — <what they are trying to get done>: <the actual steps, in order> — <where this
     arc's result shows up>
  2. <person> — <what they are trying to get done>: <the actual steps, in order> — <where this
     arc's result shows up>
FINDINGS:
  1. [blocking|should-fix|note] todo <N> — <what breaks, and the file:symbol it turns on>
     → <the concrete edit to the plan that fixes it>
  (or "none")
NEEDS-DOMAIN: specific questions research did not settle; "no" if nothing
OBJECTIVES: n/a — not this repo | read, verdict unchanged | read, changed: <what changed>
SUMMARY: at most three sentences
```

**Be willing to return `clean`.**
````

---

## 17. The UI tester

**What it is.** An agent that verifies a change by **driving the real running application** and
reports pass/fail per scenario with screenshots and read-out values. It is the only thing in the loop
that finds out whether the software works for a *person* rather than for a compiler.

**What it costs.** A browser-automation tool the harness can drive, an application that is already
running, and a maintainer willing to start it. In practice it also costs discipline: the supervisor
must ask whether the app is up **before the loop starts**, not at review time.

**What it buys.** Verification of every "Done when" clause that only an agent's word currently stands
behind. For frontend work, this is most of them — the gates only typechecked it.

**Who should skip it.** Anyone whose project has no UI, or no way to drive one from the harness.

**Two ways in.** Straight away, in the same message as the lenses, when it is already obvious the arc
touched the UI — give it scenarios yourself, preferring the challenge phase's walked interactions
over re-deriving them. Or afterwards, collecting the `NEEDS-UI-CHECK` questions from all the lenses
into **one** tester run — never two, since only one session can drive the browser.

> **If the app is up, the tester is spawned. Its work is never the supervisor's to absorb.** Whatever
> it cost to get the app running — a stale pin, a dead shell, a browser that would not start — do not
> then go on to drive the browser and walk the scenarios personally. **That is how the supervisor's
> context stops being flat, and it is the one thing this loop cannot afford.**

### `<home>/agents/arc-ui-tester.md`

````markdown
---
name: arc-ui-tester
description: Verifies a change in the already-running application by driving its real UI, and reports pass/fail per scenario with screenshots and read-out values. Never edits code, never touches version control, never starts or restarts the app. Invoked by /arc's run phase, in its review.
model: sonnet
effort: medium
maxTurns: 120
color: cyan
disallowedTools: Edit, Write, NotebookEdit, Agent, ExitPlanMode
---

You verify claims about a change by **using the running application** and report what you actually
observed. You are the only thing in this loop that finds out whether the software works for a person
rather than for a compiler.

Your brief gives you: the scenarios to verify, what change they relate to, and where the app is
running. That is everything you need — you do not read the plan directory yourself.

## Before you touch anything: read the repo's driving notes

The project keeps notes on how to drive its UI; the agent-instruction hierarchy in your context points
at them. **Read them in full first.** They exist because the obvious approach does not work in this
application: which interactions need real key events, which element references go stale, where results
are recomputed asynchronously, which routes must not be loaded directly, and what breaks the session
unrecoverably.

**Improvising past those notes is the single most expensive failure available to you.** If you hit
something the notes describe as fatal, stop rather than work around it.

## The app is already running — you do not manage it

Verify it is reachable. If it is not, or the browser cannot be driven, return `blocked` immediately
and say what you saw. **Never** start or restart the dev server, run a data migration, or take any
recovery action the repo's notes flag as needing a human. Starting a dev environment can cost tens of
minutes and is the supervisor's call, not yours.

Only one session may drive the browser. You are it — and do not close the browser window.

## How to test

1. **Reproduce the entry point the way a user reaches it.** Navigate from the app's own start, click
   through, as the notes prescribe — not by jumping to a deep URL unless the notes say that works.
2. **Run each scenario in the brief.** For every one, record what you did, what you observed, and the
   evidence.
3. **Then probe the neighbourhood.** The change's own path passing is the cheap half. Exercise the
   workflows next to it that the change could have broken — the list the edited card lives in, the
   filter over the changed field, the form that shares the widget. **Report anything broken there even
   though nobody asked.**
4. **Wait where the notes say to wait.** Values recomputed by a backend job are not there the moment
   you commit; reading too early produces a false failure, which is worse than no report.

## Evidence, not impressions

Every verdict needs something concrete behind it: a screenshot, or the exact string or number you read
out of the page. "The field appears correctly" is not a report. "`Annual total: 142.3 kWh/m²` read
from the DOM after commit, screenshot `after.png`" is.

If a scenario cannot be reached at all, say **not verified** and why. **An honest gap lets the
supervisor decide; an invented pass sends a broken change onward, and that is the one outcome this
whole loop exists to prevent.**

Distinguish clearly between:

- **the change is wrong** — the app does something other than intended,
- **the app is wrong elsewhere** — a pre-existing problem you ran into,
- **the harness is wrong** — you could not drive it, the environment misbehaved.

## Hard rules

1. No edits to source, config or data files. No version-control command of any kind.
2. No starting, restarting or shutting down the application or its database. No migrations.
3. Do not close the browser.
4. Do not "fix" the app to make a scenario pass.

## Output format

End your final message with exactly this block and nothing after it.

```
STATUS: verified | issues-found | partly-verified | blocked
SCENARIOS:
  1. <scenario> — PASS | FAIL | NOT VERIFIED — what you did, what you observed, the evidence
NEIGHBOURHOOD: what else you exercised and how it behaved; "nothing else exercised" if so
FINDINGS:
  1. [change-is-wrong|pre-existing|harness] <where in the app> — what happens, and what should
  (or "none")
NOTES: anything the next tester should know about driving this app that the repo's notes do not say
```

If `NOTES` contains something durable — a selector that works, a wait that is needed, a route that
cannot be loaded directly — **say so explicitly.** The supervisor can then get it written into the
repo's notes, which is worth more than this one report.
````

---

## 18. The human coding seat and the spike queue

**What it is.** A mode where the maintainer codes some todos themselves, in their own working copy,
while an agent drains what they leave behind in a second working copy. The maintainer never waits for
an agent's turn, and never waits for a gate before picking up the next todo they have claimed.

**What it is for — read this before the costs, because the costs are meaningless without it.** This
is a **human** spike queue, and the objective it serves is **the maintainer's flow and judgement**,
not wall clock per token. Three things, in order of how load-bearing they are:

- **The maintainer can work through many todos in one continuous run, without waiting.** That is what
  the queue is: there is always a next todo they can pick up the moment they put one down.
- **Waiting between todos is not merely slow — it is corrosive to the quality of the work.** During
  each wait the person takes in messages and unrelated input, and what they were holding in mind is
  crowded out. This is the human analogue of an agent's context filling with material irrelevant to
  the task it is halfway through, and it degrades the next decision in the same way. The queue exists
  to keep that from happening, not to finish sooner.
- **Why a human codes at all in a system driven by agents.** The maintainer remains responsible for
  delivering the code, so they need **surface contact** with it. Spiking a todo themselves is a fast
  way to judge whether that todo — and the plan around it — is heading the right way, which is a
  judgement no amount of reading handover paragraphs produces. A secondary effect, worth having but
  not the reason: the person stays a practising expert and keeps learning.

**So the token cost below is beside the point.** It is real and it is large, and it is not the thing
this layer is trading against. An adopter shown only the costs will reject it correctly for the wrong
reason, and an adopter who codes alongside their agents will reject it incorrectly for the same
reason. Quote the costs *and* the objective, together.

**What it costs.** A great deal. It needs a VCS with real workspaces (jj; git worktrees do not
compose with this design the same way), a build that is safe to run concurrently in separate build
directories, a second build directory to seed, and roughly a third of the run-phase instruction text.
It also moves **every** agent-coded todo into the second workspace, not only the drained ones.

> **Measured, on a 2.0 GB build directory:** the copy to seed a second workspace cost **3 s**; the
> **first** gate run in the seeded workspace cost **~16 min** (with another gate running concurrently
> for part of that window, so some is CPU contention); the identical gate, warm, cost **~25 s**. The
> 16 minutes is the build tool's one-time reconfigure at the new absolute path — no module is
> recompiled by it, and it is genuinely one-time per workspace. **But against todos whose
> coder-plus-review runs 3–5 minutes, that one-time first gate is the dominant term**, and it is the
> number to quote to the maintainer before the run, not the 3-second copy.

> **Seed with a real copy, never a hardlinking one.** Reproduced, not inferred: some compilers write
> interface files **in place** (opened for writing with no truncate and no rename), so a hardlinked
> seed lets a build in one tree rewrite the other tree's untouched interface files through the shared
> inode. It self-heals on the next build — but that rebuild is precisely the cost seeding exists to
> avoid.

**What it buys.** Unbroken attention for the one person accountable for the result: a maintainer who
can take todo after todo without the gaps that empty their head, and who keeps first-hand contact
with the code their agents are writing. Wall-clock overlap is the *mechanism* by which it buys that;
it is not itself the benefit. **It does not reduce token cost; it increases it** — and that is a
price knowingly paid for a person's flow and judgement, not a bad trade on throughput.

**Who needs it, and who does not.** An adopter who never codes alongside their agents does not need
it and should not install it: with nobody in the coding seat there is no flow to protect, and every
cost above is paid for nothing. An adopter who does code alongside them should install it with the
costs quoted out loud — the first gate in a freshly seeded workspace above all — and should
understand that what they are buying is uninterrupted attention, not a faster arc. It also needs a
VCS with real workspaces, so on git the answer is no regardless of how the maintainer works.

**Who codes is a default, stated, never asked.** The run writes it and goes on; nothing waits for an
answer. Where `README.md` has no `- **Human coder:**` header bullet yet, the run writes one from the
adopter's default and says so in one line of its first message — *"assuming you code all of this
arc's todos; say if not"*. The bullet is a mode, never a list: `none`, or a pointer to the
`- **Coder:** human` bullets in `todos/`, so it cannot drift against the todo files. At each PR's
start, and over a test round's todos, the pointer writes the per-todo bullet on every open todo not
yet touched; `none` writes nothing. A session resumed mid-PR finds touched todos and writes nothing,
so a todo the maintainer released stays released.

> **One person's default, not a measured rule.** The originating maintainer's: *"The standard for
> meta arcs is: I code none. The standard for all regular arcs is: I code all. Always assume this at
> first, tell me you're assuming it, without blocking."* An adopter writes their own default; what
> carries over is the shape — a default, stated in one line, that blocks nothing.

**"Coders pick up" hands every open todo to the agents until the maintainer says "I'm back".** An
untouched todo just loses its `**Coder:** human` bullet; a todo they hold mid-edit is released with
their half-edit kept as the agent coder's starting point, without asking — a human spike, the
maintainer's own word for what usually happens there; a todo an agent already started stays as it
is. The mode is recorded the moment it is said, as a ` — coders pick up since <date>` suffix on the
`**Human coder:**` bullet of every arc the session acts on, so it survives a compaction; while it
stands, the per-PR write writes nothing. "I'm back" removes the suffix and writes the bullet again on
every untouched todo of the PR being worked. **A named claim while the suffix stands writes its own
bullets and leaves the mode alone**: the maintainer taking one todo has not said they are back.

**Where it lives: two companion files beside `RUN.md`** (§5's companion rules). `HUMAN.md` holds the
human path — the claim rules, the return vocabulary, the print a human todo gets, reconciling a
resumed arc's half-finished human work — and `CONCURRENCY.md` the second workspace and the drain
below. `RUN.md` keeps only the who-codes default above and the triggers that read them: an open todo
with a `**Coder:** human` bullet, an un-drained spike, or a second workspace live on disk. **If a
triggered read fails, stop and ask**: the maintainer's half-finished work reconciles only by the
rules in those files, and running on without them would absorb it into a revision or skip it unseen.

> **Measured, by `wc -w`, in the originating system:** the human companion was about 6,600 words
> and the concurrency companion about 4,400 — words an agent-coded run, the common case there, no
> longer reads at entry or after any compaction.

**The essential shape, if you do install it:**

- **One workspace serves the whole arc**, created once, reused for every todo, torn down when the run
  ends. Root it **outside the repository** at a path stable across sessions — not in a session
  scratchpad, which a resumed run would not find, orphaning the workspace a previous session left.
- **The human always works in the default workspace. His tree is never moved to meet an agent.** Every
  agent writes in the second workspace; every supervisor VCS command runs against it with an explicit
  `-R <workspace-root>`, and the supervisor never `cd`s into one.
- **One writer per workspace.** The supervisor performs the one snapshot per workspace itself, after
  that workspace's coder returns. Every other command in that workspace, by any actor, carries the
  don't-snapshot flag.
- **Never rewrite a revision while an agent is live in a workspace whose working copy descends from
  it.** This replaces a lock file and is stronger: it removes the write rather than guarding it.
- **A fifth todo status, `human-coding`**, so the board shows at a glance that the arc is waiting on a
  person rather than on an agent. A `- **Coder:** human` bullet on the todo is what routes it; the
  absence of the bullet *is* the agent default.
- **A return vocabulary for the human**: `finished`, `this todo is wrong`, `this needs a decision`,
  plus, in a workspace-capable VCS, `spiked` (they described their revision and stepped off it,
  leaving it half-finished for an agent to drain) and `hands up:` (their hands are off the keyboard,
  so their working copy may be moved without being moved *under* them). An amendment agreed after
  `this todo is wrong` is a replan (§9, *Replanning*), and one the maintainer will code themselves
  skips that section's scoped pass.

> **Why three ways back, not one.** The maintainer is the one actor here who can tell that the *todo*
> is wrong, and a channel that only accepts "finished" turns that discovery into a diff that fails
> review and gets overruled instead.

- **The drain is depth one** — one spike finished, gated, reviewed and squashed at a time, in stack
  order. The coder works in a **child** of the spike and the squash comes last, after approval: a free
  author-attribution split, since diffing the spike against the child tells the reviewer which half is
  the human's and which the agent's, where one two-author diff could not.
- **A coder inheriting a spike needs its own section in the coder brief**, because the ordinary rules
  mislead there. Three things it must say: *a failing gate on an inherited spike is **input**, the
  cheapest available list of what is still unfinished, not a verdict*; *distinguish deliberately
  unfinished from wrong and never silently correct the author's choice* — a coder that finishes
  something left deliberately inverts the author's intent rather than completing their work; and *name
  what you judged deliberate as its own handover line*, distinct from what you judged wrong, because
  nothing else in the diff carries that split once the two halves are squashed together.

**A mechanical fact that constrains the whole design:** no harness mechanism pins a subagent to a
workspace. A worktree-isolation feature pins a *git* worktree; a jj workspace is never one. There is
no working-directory parameter on the agent-spawning tool and none in subagent frontmatter. **An agent
placed in a workspace is steered by its brief alone**, and drifting back to the default tree is the
single largest reliability risk, because nothing mechanical would stop it. The only mitigation is a
**drift check**: after a coder returns, confirm every *other* workspace is clean. A coder that drifted
shows up there and nowhere else.

---

## 19. The observations feedback store

**What it is.** A tracked directory of findings about **the arc system itself** — not about the code
under work. Every agent in the roster gets an extra optional output line:

```
FINDING: <optional, repeatable — an observation about the arc system, the tips files, or an agent
  brief itself; omit if none>
  [This finding is for the supervisor to keep and act on: if you hold the store's role, record it
  as an entry in observations/<subtopic>.md, in the format observations/README.md gives; otherwise
  send it as a message to a session that holds the role, falling back to the maintainer at the
  handoff only when none can be reached.]
```

with an explicit fence in each brief: `FINDING:` is **never** for a defect in the repo under work or
in the diff under review — those stay in the agent's ordinary finding fields.

**What it costs.** One extra output field in eleven briefs, a protocol file, a subtopic file per area,
and a standing habit of recording a finding **the moment it arrives** rather than at some later drain.
The cost that is easy to miss: it only works if **some session holds the role**. A finding that
arrives while nobody is holding it is lost.

**What it buys.** The arc system improving itself from evidence rather than from impressions. Without
it, every agent that notices a gap in its own brief reports it into a transcript that is thrown away
thirty seconds later.

**Who should skip it.** Anyone running the arc system as given rather than evolving it. This is
infrastructure for a maintainer who intends to keep changing the system.

### The protocol, in brief

**How a finding reaches the store:** exactly one way, as a message — a `FINDING:` line in a subagent's
output, a cross-session message, or the maintainer saying it. **There is no file to drop, no skill to
run, and no directory to watch.** Record it into `observations/<subtopic>.md` *on arrival*.

**The entry format:**

```
### 2026-09-19 — <the fact, in the finding's own words, carried over verbatim>

- Severity: 3/5 — <one clause tying the point to the scale below>
- Evidence: <path, revision, or URL> (possibly untracked)
- Reported by: <session or agent>
- State: <open | addressed — …>
```

> **Why the fact is carried over verbatim rather than summarised to a title.** It looks like
> redundancy and is not: nearly every observation's evidence sits in a plan directory or a repo under
> active work. **If the evidence pointer stops resolving — the directory is gone, the branch deleted,
> the session ended — the fact stated in the finding's own words is the only thing that survives.**

**The reporter never assigns a severity point.** Grading is the manager's, done once, when the finding
is recorded. A self-assessed severity is a scale nobody can calibrate.

**A finding proposing an improvement is graded by the cost of *not* making it** — what leaving the
status quo in place continues to cost, placed on the same scale a defect would be. Worked example: a
proposal to replace a hand-rolled retry loop with a library call that already does the same thing
correctly. Graded as a defect this is a 1, since nothing is presently wrong. Graded by the cost of not
making it, it is a 3 — a second copy of a policy that will drift from the library's, invisibly, until
a change to one and not the other produces a real divergence.

**The severity scale.** Each point is defined by **what leaving the finding unfixed costs**, not by an
adjective, so two people land on the same number for the same finding:

1. **A wording nit.** Fixing it changes no behaviour and no reader's decision.
2. **A stale or inaccurate reference.** It misleads a reader today, but nothing currently consumes it
   in a way that changes what runs. The cost is momentary confusion, compounding only if someone
   builds on the wrong claim.
3. **A cost paid quietly until something exercises or measures it.** Either a convention this one
   instance departs from — so it works today by luck and fails the day the excepted case is hit — or a
   confirmed inefficiency nobody has measured, recurring on every use meanwhile. A live landmine or
   an ongoing, unpriced tax.
4. **A defect that has already produced extra work or a wrong result in a real run**, caught only
   because a person noticed and worked around it by hand. Leaving it costs that same manual recovery
   again, indefinitely.
5. **A shipped rule that is actively wrong and is being followed**, with nothing positioned to catch
   it — the very check meant to catch this failure reads green on it.

**`addressed` means the arc or decision named in it has actually landed** — never that the count was
inconvenient. A backlog that has not landed stays `open` and counted, however large it gets. An
`addressed` entry keeps its words, evidence and point: **it is marked, never deleted.**

**Reporting where the store stands fires on three occasions, never a fourth:** a system-changing arc
reaches `done`; the maintainer asks; or a finding is recorded. **A store that changes only when a
finding arrives would report the same counts at every handoff forever — and a line the maintainer
learns to skip is worse than no line, because it also buries the handoffs where the store did move.**

**Report where the store stands, and nothing more.** No verdict on whether it is time for a
system-improvement arc, and no recommendation that one follow — that is the maintainer's decision.

**Who holds the role, and its one authority.** Any main-loop session running in the repository
that holds the system's own files — the one where this store lives — holds the role: a test of the
session's place, not an identity it has to be handed. A subagent never does, and neither does a
session in any other repository; those pass a finding on as *Constant improvement* below says.
Proposing a system-improvement arc is as far as the role reaches for *starting* one. **Closing one
is the role's, though**, where watch orders (§21) are installed: see §21 for when it closes a
meta-arc and what the close writes.

### Constant improvement

The system is changed while arcs are planned and run against it, so every session in it — in any
repository — works under two rules, which the originating maintainer stated in one breath: *"1.
Improvements should go back to you as a session-to-session message. 2. Changes are happening all the
time, if something seems inconsistent, give feedback and if you can't figure out on your own ask a
human."* Write them as a closing section of `RUN.md`, cited by one line in the core's *What every
phase shares* — so every phase carries the pointer without restating the rules — and by the
`FINDING:` field above.

1. **Improvements go back as a message.** A session holding the store's role records a finding
   itself. Any other session sends what it notices about the system, a tips file or an agent brief —
   its own observations and the `FINDING:` lines its subagents return — **as a message to a session
   that holds the role**, where the harness lets one session message another (Claude Code: list the
   running sessions, then send). Only when none can be reached, or the delivery is refused, does it
   carry the finding to the maintainer at its handoff, for a later session holding the role to
   record. A finding carried to the handoff waits for a maintainer to relay it; a message reaches the
   store while the session that noticed it still has the context to answer a question about it.
2. **The system changes all the time.** Skills, briefs and plan formats may change under a running
   session, so it may meet a plan, a skill or a sibling session written under a different version —
   a plan in an older format included, **which no skill converts**. Where something seems
   inconsistent, send feedback per point 1; where the session cannot work out what to do, it asks a
   human rather than guessing. This is what lets a change to the system land without transitional
   rules in every skill: the originating maintainer, on whether to carry a mapping of the old status
   values forward, *"Old plans just break. Supervisors have to figure it out and ask. This is better
   than carrying tons of rules forward."*

3. **Every handoff asks one token-saving question**: where did this work spend tokens that a change
   to the system — a skill text, a companion's trigger, an agent brief, a tips file, the loop's
   shape — would have saved? Answer from what the session can still see, in a few lines and with no
   spawn or read the handoff does not already make: the concrete place and, where it can tell, the
   rough size — a re-read, a rework cycle, a subagent that read more than it needed. One finding per
   handoff at most, sent per point 1. **Where the answer is none, send and record nothing**: a "none
   found" is not a finding, and a store that files them fills with noise. It sits wherever the
   handoff's other checks sit — in the originating system, the watch-order companion (§21).

> **Where it came from.** The originating maintainer asked for it in passing, as one small todo:
> *"they should give you feedback on how they might have saved tokens by changing the arc system."*
> It sits at the handoff because that is the one check every phase runs, in every repository. No
> count yet says how many findings it has produced; and that the session that spent the tokens is
> the best placed to say where — before a compaction summarises it away — is this file's reasoning,
> not the ruling's.

Without the store installed, point 1 has no recipient: findings go to the maintainer at the handoff,
as before, and point 3's one line rides in the handoff message. Point 2 holds regardless.

---

## 20. The citation gate

**What it is.** A small script that verifies cross-file references. In a prose corpus with no compiler
and no test suite, renaming a heading that another file cites by title leaves the pointer reading
perfectly well and pointing at nothing. **Nothing fails loudly and no test catches it.**

The mechanism: a contract file carries a hand-maintained list of `(cited-text, owning-file)` pairs; a
script confirms each cited text still appears as a heading or a bold lead-in **in the file the list
says owns it**, checked by **containment, never equality**, because real citers quote a fragment
rather than the title as written. It exits non-zero on any failure, **including a parse that finds
zero entries** — a checker that silently reads nothing must fail loudly, not pass quietly. The
gatekeeper runs it as a named command check in the same pass as its reading clauses, and a non-zero
exit is a gate failure, never a warning.

**What it costs.** A hand-maintained list — whoever adds a citation adds a line, in the same revision
— plus the honesty to say what the script does **not** cover. Four things, and a green run is evidence
of none of them:

1. It does not see a retired heading's surviving citers.
2. It does not **discover** citations. Adding one still means adding a line by hand.
3. It does not verify that a listed citing file still cites.
4. It cannot show that no unlisted citation exists.

**What it buys.** Rename detection, and nothing more — in a corpus where that is the one class of
breakage nothing else catches.

**Who should skip it.** Almost everyone. **This is the most idiosyncratic piece in the whole system.**
It exists because that particular repo was a prose corpus with dense cross-references and no compiler.
A normal code repository has a compiler, which catches the equivalent class for free. Mention it to a
user who is writing a documentation or configuration corpus and to nobody else.

**The general lesson worth propagating, even without the script:** *a list nobody updates is worse
than none* — so **every hand-maintained list says how to re-derive itself**, with the command. That
rule is portable; the script is not.

---

## 21. Watch orders, for arcs whose product is instructions

**What it is.** A mechanism for testing an arc that changes **the arc system's own instructions**.
Such an arc has no ordinary test: its todos are instructions that do not execute until a later session
invokes them, so **nothing in its own diff can show whether they work.** A reviewer can confirm a
paragraph is present, well-formed and exactly what the spec asked for, and have confirmed nothing
about its effect.

> **The precedent is on disk twice.** One agent brief shipped and then went unexercised through an
> entire subsequent arc, with nothing noticing it had never once run. And one arc's verification
> section said outright that *"the real test is the next real arc"* — which was true, and which as
> prose obliged nobody, so no next arc ever reported back.

**The mechanism.** Such an arc's `README.md` carries a `## Watch order` written at planning time,
holding exactly four things:

- **Which phases to watch** — planning, run, or test. Choose by where the rule actually fires: a rule
  that fires inside planning will never be seen by a run handoff, so an order naming the wrong phase
  is one nobody ever answers.
- **What would show the rule fired** — named, observable artefacts: a section that should appear in a
  downstream plan directory, a block that should appear in a handoff message, a step that should
  happen in a stated order. **Never "the rule was followed"**, which is a verdict rather than an
  observation.
- **What would show it did not fire** — the negative, as concrete as the positive. **Without it, a
  session that saw nothing cannot tell "the rule failed" from "nothing here to report", and that
  silence is what the whole mechanism exists to prevent.**
- **Where the observation goes** — a path in *this* arc's own directory, with the phase in the
  filename, since one downstream arc passes through several handoffs under one name.

Then **every handoff, of every arc, checks for parked watch orders** — not only when the arc just run
is itself a system-changing one, because **the arc being observed is almost never the arc doing the
observing.** The check scans for parked orders, excludes the arc this session is working on (an arc
never observes itself), decides **whether this session actually ran under the rules the order names**
— the honest test is whether they were *in force during this session*, not whether the files exist on
disk — writes a **facts-only** observation, and prints **one line, never omitted, whatever it found.**
Where journeys are installed (§8), the scan reaches arcs nested in a journey as well as top-level
ones, the self-exclusion compares arc **directories** rather than names, and the observation's
filename carries the observing arc's **id**, since two journeys' arcs `01-…` share a name.

> **Why facts only, with no verdict.** An agent judging the rules it just ran under, with nobody
> between it and the plan, is exactly what the maintainer's acceptance gate exists to prevent.

> **Why a session that correctly wrote nothing must still say so.** A handoff silent about the check is
> indistinguishable from one where the check never ran — which is the precise failure this mechanism
> replaces.

**On measuring cost in an observation.** Where an order's claim is about speed, name what the observing
session can actually produce. A session *can* read a clock, but one that did not stamp the start
cannot recover it afterwards. **So the default is a proxy** — how many tool calls and messages fell
between two named points — and the observation says that it is a proxy and does not convert it into a
duration.

**Answered and closed are different**, and conflating them is how a parked arc either never gets its
second look or collects observations forever. A phase is **answered** when every bullet on its fired
list is recorded as *observed* or *not observed*. A bullet only a downstream meta-arc could show,
recorded *not applicable* by an ordinary arc, leaves the phase open for one that can. It is
**closed** when the arc moves to `done`.

**The pool the check scans** is the `underway` arcs with a watch order whose PRs are all `done` or
waiting on the maintainer (§8, *The Status vocabulary*: `accept`, the automatic read closed). A
`done` arc is not in it, nor is an arc whose read is still open, nor one with a PR sent back to
`planning` or `running` by a test round, a fix todo or a small replan — it returns once that PR is
waiting on the maintainer again. So a watched arc's round has to be run to completion: left half-run,
the arc silently stops being scanned, with nothing anywhere saying so. Telling the two kinds of
`accept` apart costs the scan one read of each PR's last review-log line (§11).

**Who closes it.** Without the observation store (§19), the maintainer, by their word, as for any
arc. **With the store installed, the session holding its role closes a meta-arc itself**, once the
evidence is in, and the maintainer's word still always works. The reason is specific to meta-arcs:
their todos are instructions nothing in their own diff can exercise, so a PR's `accept` and `done`
differ by nothing the maintainer can perform, and leaving the write to them left arcs parked indefinitely on a
decision they had no basis to weigh. Every other arc stays the maintainer's to close, because only
their testing makes that `done` mean something. The close runs at any handoff's watch-order check, or
whenever the maintainer asks.

- **Enough evidence:** every phase the order names is answered. A bullet recorded *not applicable*
  counts toward that only when **no repository the maintainer actually runs arcs in has the shape
  the bullet needs** — read from the plan directories' `- **Repo:**` bullets, not from memory. Where
  such a repository exists, the evidence is pending and the arc waits. **A missing event is not a
  missing shape:** a bullet waiting on something that has simply not happened yet — a threshold no
  session has crossed — stays pending.
- **Only where every PR is `done` or waiting on the maintainer.** The close writes `done` on each PR
  still waiting and on the arc, so a PR whose read is still open, or that holds an open todo, keeps
  the arc `underway` whatever its observations say; the check's line names that PR.
- **Two exits, both writing `done`, never anything else.** *Works*: every bullet observed, or
  allowably not applicable — the arc's Status bullet names the observation that answered each phase.
  *Needs rework*: any bullet recorded *not observed* — the Status bullet names those bullets, a new
  entry goes into the store saying what needs reworking, and the maintainer is told in the same
  message. The rework is a new meta-arc; the closed one is never reopened. Either way the Status
  bullet also names the repositories checked, so a later reader can audit which silences were
  allowed to count.

**If the user also keeps a portable extraction of their system** — a file like this one, re-extracted
from their own corpus as a coherence test: a corpus that cannot be extracted coherently has an
incoherence in it, though a clean extraction says nothing about whether the rules help — **the
meta-arc's handoff in the system's own repository says, in one line, that an extraction is owed**,
at the handoff where no PR holds an open todo any more, not at an earlier PR's. The session starts
it on the maintainer's go, as an activity of its own after the handoff, and the line carries a
copy-pasteable prompt for starting it once this session is gone. The handoff itself only announces:
reading the whole corpus inside a handoff is the wrong place for it. At the handoff, not
at `done`, because a meta-arc's `done` can lag by weeks on watch-order evidence, and an incoherence
is cheapest to attribute right after the arc that introduced it.

> **Measured: a rule with no trigger does not fire.** The originating system's extraction rule lived
> only in its observation store and in the extraction's own repository, which no session reads on
> its way through a handoff. Two extractions fell due on the same day and neither ran until the
> maintainer noticed and briefed a session by hand. Putting the line in the handoff the meta-arc
> already passes through is what makes it fire every time.

**Where it lives: a companion file, `WATCH.md`, beside `RUN.md`** (§5's companion rules), read at
every handoff of every phase in every repository — it holds the check above, the line printed when
nothing fired, and, where installed, §19's token-saving question. The run, planning and test texts
each point to it at their handoff, by path in prose; **if the read fails, stop and ask**, since a
handoff without the check is indistinguishable from one where it never ran.

**Optional beside it: an instruction-prose check in the per-todo reviewer.** Offer it to a user
whose arcs edit text a model will read — a skill, a companion, an agent brief, an instruction file.
It is one more numbered check in `arc-reviewer.md` (§6, or §10's variant), skipped for any other
diff, code comments and user docs included, and it judges the changed text against six heuristics: a
rule states its reason; its actual reader can follow it; it generalises beyond the incident that
prompted it; every line pulls its weight; detail lives where it is needed, in an on-demand file a
pointer reaches; a `description` says what the file does and when to use it. A `description` that
drops its *when*, a file nothing points to, or an instruction its reader cannot follow can carry a
`revise`; the rest is a minor finding prefixed `(taste)`, which never forces one.

> **Not invented here.** The heuristics are Anthropic's `skill-creator` skill's, which the
> originating maintainer adopted as the standard for such text; the repository's own instruction
> files and House rules win where they disagree. At extraction time it had just been added and no
> arc had yet run under it.

**What it costs.** A section in some plans, a check at every handoff of every arc, and arcs whose
PRs wait on the maintainer for days or weeks to be answered. **A meta-arc parked that way is the
mechanism working, not a stalled handoff** — but a maintainer who does not know that will read it as
one.

**What it buys.** The only honest test that exists for an arc whose product is instructions.

**Who should skip it.** Anyone whose arcs change software rather than agent instructions. If the user
is not writing meta-arcs, this is dead weight — say so plainly.

---

# PART IV — MECHANICS, TAILORING, AND HONESTY

---

## 22. Setup mechanics, in full

### What goes where

```
<home>/                          user-level: applies in every project
  agents/<name>.md               one subagent per file
  skills/<name>/SKILL.md         one skill per directory; the directory name is the command
  skills/arc/SKILL.md            the arc skill's core: routing and shared rules, under its cap (§5)
  skills/arc/RUN.md              a phase text, read in full when routing lands on it; PLAN.md
                                 and TEST.md beside it at Standard (§8)
  skills/arc/<NAME>.md           companion files, read only on the trigger a phase text states
  plans/arc-<slug>/              one directory per arc — or, with journeys installed,
  plans/journey-<slug>/arcs/NN-<slug>/   every arc nested in a journey (§8)
<project>/.claude/agents/        project-level: applies only in that project
<project>/.claude/skills/
```

The layout is Claude Code's, with `<home>/` as `~/.claude/` and `.claude/` as the project-level
directory. Another harness keeps the same pieces under its own names.

**User-level for the arc system.** It is meant to work across repositories, and a project-level copy
would have to be maintained in each one.

**Keep the plan directory out of the repository under work.** Planning churn does not belong in the
history you are trying to keep clean, and a plan must survive its branch being deleted. The
consequence — the plan is not backed up by the repo's remote — is a real one; §24 Q6 asks about it.

### How a harness discovers an agent

The harness scans `agents/` for markdown files with frontmatter. **The `description` field is the
discovery surface** — it is what the harness matches against a task when choosing an agent, and what
a supervisor reads when deciding which to spawn. Write it as a complete sentence describing the job,
its inputs and its outputs, and end it by naming who invokes it (`Invoked by /arc's run phase`), so
an agent browsing the roster can tell a loop component from a general-purpose helper.

The spawn is by `name`. The brief the supervisor passes becomes the agent's task; the agent file's
body becomes its system prompt. **The agent inherits no conversation history** — only its own file,
its brief, and whatever instruction-file hierarchy the harness loads automatically.

### How a slash command maps to a skill

`skills/<dir>/SKILL.md` with `name: <dir>` is invocable as `/<dir>`. Two routes in, and the skill
must work under both:

- **Explicit:** the user types `/arc plan issue 123`. `argument-hint` is what the harness shows them.
- **Implicit:** the user says "plan an arc for issue 123" and the harness loads the skill because its
  `description` matched. **This is why the description must contain the words a user would actually
  say**, not just a precise technical summary.

**The rendered skill body enters the conversation as one message and stays there.** It is paid for on
every turn of that session. This is the whole reason for phase texts and companion files: anything
a session does not always need — another phase's procedure, rationale, an optional subsystem, a
reference table — goes in a file that costs nothing until the skill tells the session to read it.

> **Measured:** loading a large run skill started a session at 51.9k or 78.3k tokens against a
> 40–42k baseline with no skill loaded. **Resident instruction text is the second-largest cost lever
> in the system**, after session length.

### Harness facts that change what you can write

These were established by testing Claude Code, not assumed. Another harness may differ on any of
them, so confirm each in the user's before relying on it. Each one forecloses a rule that otherwise
looks obviously right.

- **Agent file edits bind the next delegation, in the same session, with no restart.** A new file
  inside an existing `agents/` directory is picked up; a new `agents/` *directory* needs a restart.
- **Skill edits bind the next invocation — but re-invocation *appends*.** A skill invoked again with
  changed content leaves **both** copies in context, competing. **After editing the core, start a
  fresh session before using it.** A phase text or companion is a file the session reads, not an
  invoked body, so an edit to one reaches the session's next read of it.
- **A compaction re-attaches invoked skill bodies cut short, and drops the oldest.** Measured on one
  version: each re-attached body cut at exactly 20,000 characters, against a documented budget of
  5,000 tokens per skill and 25,000 across all of them. Text past the cut is gone until re-read,
  which is why the core has a cap and why the session re-reads its phase text after every compaction
  (§4, §5).
- **The read tool refuses a file over a token limit — 25,000 tokens in Claude Code — even when the
  read is ranged.** A long phase text is read in consecutive chunks (§5's chunked read rule); a
  single read is not a safe assumption for a file that grows.
- **The skill-directory placeholder is substituted only in the invoked body.** Claude Code's
  `${CLAUDE_SKILL_DIR}` becomes a path in `SKILL.md` as rendered, and stays literal in any file the
  session merely reads — so a phase text or companion opens with one line saying what the
  placeholder means there (§5).
- **Hooks can fire on compaction.** Claude Code runs a `SessionStart` hook with the matcher `compact`
  right after one, and a `PostCompact` hook beside it; §8 installs the first and offers the second.
- **Nested spawning works.** A subagent granted the agent-spawning tool can invoke it, and its own
  subagent replies. This was probed. But **the supervisor cannot see or bound what a subagent
  spawns**, so any brief that permits delegation must carry an explicit cap and require the agent to
  report what it delegated.
- **Some tools arrive in a subagent as *deferred*:** the name is visible but the schema is not loaded,
  so calling one directly fails validation. Web search and fetch are the usual cases. A brief that
  expects an agent to use one must tell it to load the schema first.
- **No harness mechanism pins a subagent's working directory to an arbitrary path.** There is no such
  parameter on the spawning tool and no such field in subagent frontmatter. An agent placed anywhere
  is steered by its brief alone.
- **A live subagent sends nothing until it returns.** Silence is not evidence of a stall. Do not
  re-spawn on silence alone; elapsed time past roughly twenty minutes is the only signal.
- **The session token counter is a budget, not a context gauge.** It starts far above the window size,
  so a rule phrased as "hand off at 50% of context" cannot read it; the session transcript, which
  records every turn's token usage, can. **A rule may not assume a measurement the harness does not
  offer** — it would look precise while quietly never firing — so check the reading exists in the
  user's harness before installing a rule built on it.

---

## 23. Token frugality: the rules that make a flat context possible

These are not arc-specific. They are what makes invariant 3 achievable in practice, and they are
worth writing into the user's machine-wide instruction file whether or not they adopt anything else
here.

**The cost identity:** billed input for a session is the sum of its context size over every turn. A
token that enters context is paid again on every later turn. **Cost is quadratic in session length
and linear in resident text.**

1. **Bound width, not just line count.** `head -20` caps lines, not tokens — one minified line is
   thousands of characters. Cap columns too: `rg -M200`, `cut -c1-200`, `ps -eo pid,etime,comm`,
   `jq -r` projections.
2. **Ladder up: count → names → stat → content.** Open with `rg -c`, `rg -l`, `--stat`, `--summary`.
   Fetch real content only once the target is narrowed.
3. **Never run the same command twice.** Scroll up first. If output is needed repeatedly, write it
   once to a scratch file and search *that*.
4. **Strip decoration.** Colour, box drawing, progress bars and pretty-printers are pure cost. Prefer
   plain or `--json` output and never pipe into a pager.

| Instead of | Use |
|---|---|
| `grep -rn PAT .` | `rg -l PAT`, then `rg -n -M200 PAT` |
| `find . -name '*.hs'` | `rg --files -g '*.hs'` |
| `cat file` | a read tool with offset/limit, which also de-duplicates re-reads |
| `cat x.json` | `jq -r '<projection>' x.json` |
| dumping JSON to find a field | `gron f.json \| rg PAT` for the path, then `jq` that path |
| a build or test run whose success output you do not need | `chronic CMD` — prints only on failure |

**Two gotchas worth carrying:**

- **Glob-expanded paths starting with `-`** are parsed as flags by `grep`, `ls` and `jq`. Prefix the
  glob: `./*/*.jsonl`. The symptom is a baffling `invalid max count`.
- **Hand-wrapped prose defeats a naive line-based search.** A phrase spanning a line break returns a
  false negative. **An empty result is not evidence of absence until the file is flattened first:**

  ```
  tr '\n' ' ' < file | tr -s ' ' | rg '<phrase>'
  ```

> **Four measured dead ends — do not spend time on these.** Tool output: 3.5 MB total across 21
> sessions. HTML comments in agent and skill files: 3.1 KB total. Review-cycle churn: recorded cycles
> ran 9×1, 3×2, 1×3 — the loop does not loop. Agent body size: 3–12 KB against a ~22.2k-token spawn
> floor. **All four are noise next to session length** — which, once compaction carries a session,
> is bounded by the compaction window (§8).

---

## 24. The tailoring interview

Ask these in order, before writing anything. Each answer changes what you write. **Ask them as
assertions where you can** — "I'm reading this as a git repo with `npm test` as the gate; what's
wrong?" gets corrected; "what's your setup?" gets a shrug.

### Q1 — Version control: git or jj?

**Why it matters.** Several mechanisms in this system are jj-only, and installing them against git
produces instructions that cannot be followed.

- **git** → the sealing step is `git add -A && git commit -m "<description>"`. The cumulative diff is
  `git diff <base>..HEAD`. **Skip §18 (the human coding seat) entirely** — it needs workspaces. Skip any
  history rewriting, squashes included; `git rebase -i` and `commit --fixup` do not reproduce the
  semantics safely enough to prescribe. Require a clean tree in the preflight.
- **jj** → the seal is `jj describe -m "<description>"` then `jj new`. Three jj facts shape the
  briefs:
  - **`jj st` and `jj diff` are not read-only.** Both snapshot the working copy, creating an
    operation and rewriting the current revision with the file contents on disk. Agents that must not
    write pass `--ignore-working-copy`, which means *don't snapshot and don't update*.
  - **Two commands have silent failure modes that wedge an agent.** `jj squash --into` opens an
    editor when both revisions carry descriptions — pass `-u`, which takes the destination's
    description. `jj split` without file arguments defaults to an interactive diff editor. **An agent
    that opens an editor hangs until killed, and cannot even report why it is stuck.**
  - **Conflicts cascade.** A conflict at the bottom of a stack marks every revision above it
    conflicted, so the count reads far worse than the work is. Resolve the **lowest** conflicted
    revision first and recheck rather than assuming the rest still need attention — everything above
    clears on its own.
  - **The operation log is one per repository, not per workspace.** Where several workspaces share a
    repo (§18), `jj undo` reverts whatever happened last *anywhere* — often another session's
    operation — and `jj op restore` rolls back every operation made since. Prefer a forward fix that
    names only your own revisions; where an undo is unavoidable, find the operation's id in
    `jj op log` and `jj op revert <id>` that one. The originating maintainer's ruling.

  **What jj buys here, concretely:** conflicts are recorded in revisions rather than blocking the
  working copy, so a conflicting move does not stop the loop; `jj resolve --tool mergiraf` is
  non-interactive, exits 0, and leaves untouched anything it cannot merge, so attempting it always
  costs nothing; revisions are addressable by stable change id across rewrites, so a plan file can
  name one and still find it after a rebase; and workspaces make §18 possible at all.

  **Measured, worth knowing before writing any revset into a skill:** the `..` range operator **fails
  open** on a bound that is not actually an ancestor — it silently returns everything — while the
  `(<a>::<b>) & ~<a>` form **fails closed**, returning empty. Prefer the form that fails closed
  anywhere a bound could be mis-resolved.

### Q2 — What is the build gate, and what is the test gate?

**This is the most important answer in the interview.** The whole guarantee rests on it. Do not
accept "the usual" — get the exact strings.

- **Watch for a build command that silently skips part of the project.** Test suites, examples and
  benchmarks are common omissions. Ask what CI actually runs.
- **The two gates must carry an identical flag set**, derived from what CI enforces. Where the build
  tool hashes its flags into its plan, a flag one gate passes and the other omits makes each
  invalidate the other's cache, and every run recompiles from scratch.
- **Ask how long they take.** A gate over ten minutes means the background form in step (d) is the
  normal path, not the exception.

**What a correct gate line looks like.** Three common shapes, offered as patterns to recognise rather
than defaults to copy — the flag set has to be whatever this repo's CI actually enforces, and the two
gates must carry an identical one:

| Project shape | Build gate | Test gate |
|---|---|---|
| Node, npm scripts | `npm run build` | `npm test` |
| Rust, cargo | `cargo build --all-targets --locked` | `cargo test --all-targets --locked` |
| Python, ruff + pytest | `ruff check . && mypy .` | `pytest -q` |

Each of the three has a trap worth naming out loud: `npm test` in a repo whose real suite sits behind
a second script tests nothing; `cargo build` without `--all-targets` skips tests, examples and
benchmarks; `pytest -q` with no path collects nothing at all if the test root is misconfigured, and
reports that as success. **The Haskell case in §9 is the same lesson** — `cabal build all` skips test
suites and `--enable-tests` is what covers them. **Run the pair once yourself before you write it
down.** A gate that has never been executed is a guess in the shape of a command.

**If there is no user available to answer — the agent-alone case.** This is where the system gets
installed broken, and it fails silently: a `<build gate>` written to disk verbatim leaves a skill
that looks complete, a plan directory that looks complete, and a first failure that arrives much
later, as a supervisor running the literal string `<build gate>` in a shell. **Never write a
placeholder to disk.** In order:

1. **Derive the gate from the repo, which almost always states it somewhere.** The CI workflow is the
   authority — it is the project's own operative definition of "builds". Failing that: the
   `scripts` or `tasks` block of the project's manifest, a `Makefile`'s default target, a task-runner
   file, `CONTRIBUTING.md`, or the repo's own agent-instruction file.
2. **Write what you derived, and say in one line where you got it**, so the maintainer corrects one
   string instead of re-deriving the whole answer.
3. **If you cannot derive it, stop and say so loudly.** Do not install a loop that half works. Write
   the two lines into House rules as

   ```
   - **Build gate:** UNSET — none could be derived and no maintainer was available to ask.
   - **Test gate:** UNSET — as above.
   ```

   and say, in the **first** line of your report, that `/arc` will refuse to start a todo until
   both are filled in. The skill in §7 carries exactly that refusal, and it is what converts a silent
   misinstall into a loud one.

### Q3 — Does the project have a gate at all?

Some do not: a documentation corpus, a configuration repo, a prose project.

- **If there is no gate**, write a **reading-based substitute** into House rules instead, and give it
  to the gatekeeper verbatim. It must be a **closed list of enumerated, structural clauses** — the
  frontmatter parses, required keys are present, no fence is left unterminated — plus any named
  command check. **Anything outside the enumerated list is a reviewer concern, never a gate failure.**
  The whole value of the substitute is that it stays narrow and mechanical.
- **If there is no gate and no sensible substitute**, say so plainly and install Core anyway. The
  loop still buys the one-todo-one-revision discipline and the independent review. It just cannot
  promise that every revision builds, and the user should know which guarantee they are not getting.

### Q4 — How large is the repository, and how well does the user know it?

- **Large or unfamiliar** → the challenge phase (§16) earns its cost, especially the `research` lens's
  reach claims. Offer it.
- **Small and intimately known** → skip the challenge phase. The planner's claims about the tree are
  probably right, and nine agents to confirm that is a bad trade.
- Either way, check **whether the repo's own agent-instruction file actually loads.** A repo whose
  conventions sit in a file the user's harness does not read — in Claude Code, an `AGENTS.md` with
  no `CLAUDE.md` importing it — is one where every convention the maintainer wrote down is invisible
  to every agent. **Offer the one-line fix before anything else; it is worth more than the first
  arc.**

### Q5 — Does the user want the challenge phase?

Quote the real price: **up to nine agents per round, on capable models, for as many rounds as the
progress judgement finds grounds for.** Then the real benefit: a defect caught there costs one
paragraph, and the same defect caught in the run costs a coder's turn budget and an undone revision.

**Offer a middle option:** run only the `research` lens. It is the cheapest of the three and catches
the highest-value class — false reach claims that a plan's todo boundaries were drawn from.

### Q6 — Solo or team, and where should the plan directory live?

- **Solo** → plans under `<home>/plans/` are fine, but say the consequence out loud: **they sit
  outside version control by design, so nothing is backing them up.** A lost disk loses every plan
  directory, including the one holding the durable state of a run in flight — which is the one thing
  this design says must survive a session dying. Ask whether that matters. For most people it does,
  and the fix is cheap and general: **sync the plans directory to something.** Put it inside a folder
  that is already being synced and symlink it into place — Nextcloud, Dropbox, Syncthing, or a second
  git remote holding nothing but plan directories all do the job, and the right choice is simply
  whatever the user already runs. If you symlink, warn that a version-control tool records the
  *link*, not the tree behind it, so a repo-wide search for tracked changes will never surface a plan
  file.
- **Team** → two consequences. The handoff phase should prepare a pull-request body rather than
  stopping at "done", assembled from the plan directory: one bullet per revision, with its title, id
  and the substance of its handover paragraph, plus `Closes #<n>` if there is an issue. **Carry over
  from `review.md` only what a human reviewer needs** — the `note` findings deliberately left, and
  anything a tester could not verify. Reviewers are colleagues, not a log audience: the findings that
  were fixed are already in the diff, and the lens machinery is not their business.

  And: **the supervisor performs no outward-facing act.** No push, no pull request, no comment on
  one. It prepares; the maintainer publishes. If the user wants to delegate the publish, make it an
  explicit per-arc accept, never a default — **publishing is the maintainer's act, not the loop's.**
  The originating system allowed exactly three exceptions, none widening another, each pushing only
  its own bookmarks and never trunk: a push to a private backup remote nobody reviews from, in the
  one repository that had one; pushing and opening the draft pull request(s), only when the
  maintainer asked for the summary and then accepted that offer; and, where grouping is installed
  (§8), pushing an earlier PR's bookmark after a fix round on it, offered in its handoff and run only
  on their yes.

  **Base the pull request where the arc's base sits.** Where the arc was planned on top of a
  colleague's open pull request, its PR targets that PR's branch, not trunk — and the
  fetch-and-rebase before opening it goes onto that branch too. Measured: in jj,
  `jj rebase -b <arc tip> -d <trunk>` moves the whole connected branch, base included, and changed
  the colleague's commit ids; rebasing onto the colleague's bookmark left them as they were.

  **Record each opened pull request's URL and head bookmark** in a `- **Pull requests:**` header
  bullet, written when the PR is opened (or when the maintainer reports one they opened). The two
  rules below read it, and a PR already recorded is **updated — bookmark moved, pushed on a yes —
  never re-created**.

  **Never rewrite what is reviewed.** A revision is *reviewed* once it is in a pull request's pushed
  bookmark and someone **other than the maintainer and the arc system** has left a review or a
  comment on that PR. Before that, a later detour may be squashed into it; after, only appended on
  top. The arc system's own reviewers are what the stack exists for, and the maintainer's own
  comments on their own draft are findings, not review — counting them would close off squashing on
  a PR no colleague has opened. Tell by reading the recorded PR, dropping comments by its author,
  by the maintainer's own login, and by bots; **no recorded URL means nothing is reviewed.** It is
  judged per revision — in jj, `<rev> & ::<bookmark>@<remote>` non-empty — so a detour sealed since
  the last push can still fold into an unpushed fix. **Rebasing a reviewed PR is not rewriting it**
  (its revisions keep their content); pushing the rebased PR waits for the maintainer's go.

  > **Why "reviewed" and not "published".** The guard used to refuse any fold behind a bookmark that
  > tracks a remote. Publication was a proxy for the real hazard — rewriting history under a
  > reviewer's feet — and wrong both ways: it fired on a backup push nobody but the maintainer reads,
  > foreclosing every later tidy-up there, and it did not fire on a stack a colleague had reviewed
  > before it was pushed.

  **Reading a pull request is not an outward-facing act.** The originating maintainer, on a rule that
  read PRs only on request: *"You can always read a PR, no idea where that rule comes from. It's
  always ok to use gh to read the PR, check on CI, see if someone reviewed etc."* So **the arc's
  recorded pull requests are read at every session start, at each PR boundary inside a running
  session, after every seal, and after every compaction** — before anything that depends on which PR
  is open — since a session that carries on for days would otherwise read the forge once. It
  reports, one line each: a PR merged or closed (every session while it stands, since it is
  state; the upper stack then needs rebasing); a review or comment by anyone but the maintainer, the
  arc system or a bot, newer than the last read, with the offer to take it into the test phase — a
  colleague's remark is not the maintainer's finding until they say so; and failing CI. Take the
  time *before* the first read and store it, after a read that succeeded, in a
  `- **Forge read:** <UTC timestamp>` bullet, so each comment is reported once; a failed read says
  so and leaves the bullet alone. **The read acts on nothing**: it moves no status and no todo, and
  answers no comment.

  **Where it lives: a companion file, `PR.md`, beside `RUN.md`** (§5's companion rules), holding the
  forge read and the pull-request summary. The run phase reads it when the arc's `README.md` records
  an issue or a pull-request URL, and a session that compacted runs the forge read before carrying
  on, since it sits outside what the compaction re-read names (§8, the hook's fourth item). A team without a forge, or an
  arc with no recorded PR, never reads it.

### Also settle, in one line each

- **What the system is for, in their own words.** The top of this file gives the originating
  maintainer's answer. Ask for the user's, and write it down where a judging agent will read it
  (§25's `OBJECTIVES.md`), quoted rather than paraphrased.
- **A commit-message convention?** One is worth recommending: a trailer on every commit, after a
  blank line, naming the model that assisted.

      Assisted-By: <model name, with its context size> <noreply@<vendor>>

  Recommend the reasoning, not just the spelling, because the spelling follows from it: the trailer
  names **the model that assisted**, and *assistance* is what happened — not co-authorship. A harness
  may ship its own default asking for a co-authorship trailer instead; where it does, the convention
  has to say explicitly that it **overrides** that default, or a session following only the harness
  default emits the wrong trailer without ever noticing there was a rule to break. Write it into the
  machine-wide instruction file, not into the skill — it applies to every commit, not only to arcs.
- **Usually one arc, one pull request.** The revisions inside a pull request are the review unit;
  that is the whole point of one todo per revision, so an arc is never split mid-todo. A team whose
  arcs are sometimes too large to review as one, or need an upstream fix sent ahead, can install
  grouping (§8, *Grouping an arc into pull requests*); everyone else keeps one PR per arc.

---

## 25. What I left out, and what I would not propagate

### Left out because it was specific to one machine

- **Absolute paths, project names, employer, remotes, personal namespaces.** All replaced with
  placeholders.
- **A private-branch merge topology.** The originating repositories used a "work revision": a merge of
  a trunk-descended parent and a private branch, with new work stacked on top of it and sealed
  revisions moved *below* it. It made every revset in the run phase conditional, roughly doubling the
  length of several sections, and none of it generalises — most repositories commit onto a branch.
  **If a user turns out to have such a topology, the one portable lesson is: pin the reference point
  once, by identity, at the start of the run, and read it from a recorded bullet thereafter.**
  Re-deriving a position by exclusion (*"the parent that is not trunk"*) silently breaks the moment
  that parent is itself rebased onto trunk.
- **History hygiene** — folding a review-fix revision into the revision it repairs. In the
  originating system it is no longer a phase: a seal-time fold handles the review-fix and test-round
  detours systematically, and beyond that there is **one opportunistic rule — whenever a session
  notices a squash candidate, it asks the maintainer and squashes on their yes**, with no entry
  point, no Status precondition and no sweep. The maintainer: *"No extra process, no extra mandatory
  steps."* What that gives up is a systematic second sweep, which was mostly re-walking ground the
  seal-time fold had covered. The mechanism is two jj commands, and its fence is now §24 Q6's *Never
  rewrite what is reviewed*; the judgement — *a detour is a revision that only repairs an earlier
  revision of this same stack: its files are a subset, its description reads as a correction, and a
  reader of the final tree learns nothing from seeing the two apart* — is the portable half, and it
  is quoted here rather than given a section.
- **A language-specific tips directory.** The idea is worth stealing in one line: **keep per-technology
  notes in files an agent is told to read by name.** A subagent inherits the instruction-file
  hierarchy but **does not** inherit an on-demand file, so an agent that needs those rules must be
  told to read that file explicitly.
- **A statusline script, backup-push rules, a session-memory protocol, and the compaction-window
  plumbing** — a per-session window setting and the statusline that shows it — along with the
  companion file that held the originating repository's own handoff extras. All machine
  infrastructure, none of it the arc system; §8 keeps the one portable part, that the window is the
  cost lever. (Cross-session messaging used to sit on this list; it is now how a
  finding reaches the store, §19's *Constant improvement*, so only its harness mechanics are left
  out.)

### Compressed, and what was compressed out

- **The run text shrank by roughly two-thirds.** What went: every branch of the private-branch
  topology; the human-coding-seat machinery (summarised in §18); the pull-request assembly (summarised in
  §24 Q6); history hygiene; the backup push; and the watch-order check (summarised in §21).
  **What was kept in full: the loop, the verdicts, every branch of step (c), the sealing discipline,
  the recording step, and the standing rules.**
- **The planning text shrank by about half.** What went: the plan-mode file-count exception and the
  legacy-plan expansion, both artefacts of one harness's history; the watch-order section; and most of
  the cross-references. **What was kept: all seven steps, the sizing rules, the "Done when" standard,
  the Value bullet, the story conversation, and the whole challenge procedure.**
- **Every agent brief kept its hard rules, its authority section, its bounds and its output block
  verbatim in substance.** The `FINDING:` line was moved to the optional layer (§19 — the observations store), since it is
  meaningless without a store to write to. Model tiers and turn ceilings are kept with their
  measurements, because a reader who changes them should see what they were sized against.
- **The rationale files.** The originals kept a rationale file per phase text — `RUN-RATIONALE.md`
  and `PLAN-RATIONALE.md` beside the core — mirroring its heading structure, holding the argument for
  each rule, deliberately off the resident path and never re-read after a compaction, since it is
  needed only when someone considers *changing* a rule. **That pattern is worth copying**, and it
  is why this file carries its reasons in blockquotes: a reader changing a rule can see what it cost
  to learn.

### What I would not propagate

Said plainly, because an adopting user is better served by knowing which parts earned their place.

- **The citation gate (§20)** solves a problem a compiler already solves. It earned its place in a
  prose corpus and nowhere else. The portable residue is one sentence: *a list nobody updates is worse
  than none, so every hand-maintained list says how to re-derive itself.*
- **The watch-order mechanism (§21)** is correct, and almost nobody needs it. It exists because that
  system's product was partly its own instructions. For a user shipping software, it is pure overhead
  — but if they ever start writing agent instructions as a deliverable, it is the only honest test
  there is.
- **The UI tester (§17) is the least evidenced agent in the roster.** The originating system recorded
  an entire arc producing zero spawns of it, because the application was found down only at the review
  phase and the supervisor then walked the scenarios itself — which is precisely the failure the
  "check the app is up *before the loop*" rule was added to prevent. The brief is good; the evidence
  that it gets used is thin, and the rule that makes it reachable matters more than the brief does.
- **Carrying on through compaction (§4, §8) is the least proven rule that governs every run.** It
  replaced a session rule that had never been seen to fire, and the reason it is safe — the re-read
  of the core, the phase text and the arc's `README.md` after every compaction, with the steps taken
  from what was just read — rests on one fact: a summary paraphrases, and the files do not. **What
  has not been measured is how it holds over many compactions in one session**, and **which
  compaction window is best**: a smaller window keeps the context, and so every turn, cheaper, at
  one more re-read each time it fires. At extraction time the originating maintainer was running
  sessions at several windows side by side to find out, and a hook that held compaction back while a
  coder was in flight had been tried and dropped (§8). Install it with the compaction hook, offer
  the compaction log, and tell the user both open questions.
- **The core's cap (§5) is a harness measurement, and the core sits close to it.** At extraction the
  originating core was 25 characters under its 14,000. Growth is asked about, not gated; a user who
  adds routing should measure, and move a rule into a phase text before raising the cap.
- **Nine-agent fan-outs (§16)** are easy to install and hard to justify without the two counts at the
  top of `review.md`. If the user takes the challenge phase, insist on those counts. **A phase nobody
  measures is a phase nobody can cancel.**

### Expensive, serves an objective many adopters will not have, and not a candidate for removal

One entry belongs on neither list above, and an earlier draft of this file put it on the wrong one.

- **The human coding seat and the spike queue (§18).** The costs are not in dispute and they are
  steep: roughly a third of the run phase's instruction text; a second working copy to seed; a
  one-time first gate measured at **~16 minutes** on a 2.0 GB build directory against todos whose
  coder-plus-review runs three to five; **every** agent-coded todo moved into that second workspace,
  not only the drained ones; a **higher** token cost, not a lower one; and a working-directory
  assignment that no harness mechanism can enforce, steered by the agent's brief alone. What is in
  dispute is what those costs are *for*. **They are not buying wall clock.** They are buying a
  maintainer's **uninterrupted flow and their first-hand judgement of the work**: a queue they can
  draw from without ever waiting, because each wait fills their head with messages and unrelated
  input and crowds out what they were holding in mind; and a seat at the code they remain responsible
  for delivering, where spiking one todo tells them whether the whole plan is heading the right way.
  Against *that* objective, token cost is beside the point. **So: expensive, and worth it to an
  adopter who actually codes alongside their agents. An adopter who never does needs none of it —
  not a reduced version, none. Recommend it on that question and on no other, and quote the costs and
  the objective in the same breath, never the costs alone.** **And write that objective down where a
  judging agent will read it** — see *Two things I would add*, below, for where.

> **The general lesson, and it is half of why this file exists.** *A design whose real justification
> is unwritten will be misjudged by every future reader.* The rationale above was recorded nowhere in
> the corpus this file was extracted from — not in a plan directory, not in a skill, not in a
> decision record. So the extraction found a layer with large measured costs and no stated purpose,
> inferred the only purpose those costs seemed consistent with, and recommended removing it. **The
> measurements were right and the conclusion was exactly backwards.** The cost of an unwritten *why*
> is not that a later reader will lack it; it is that they will confidently supply a wrong one and
> act on it, with every appearance of rigour. Write the reason down beside the rule. That is what
> `decisions.md` (§3) is for, what the blockquotes throughout this file are for, and what the
> revisit-trigger pattern in the next subsection asks you to take one step further.

### Two things I would add if I were installing this fresh

**First: an `OBJECTIVES.md` at the repo root, saying what the system is for.** It is the direct fix
for the failure the blockquote above describes, and it is cheap — one page, tracked, **read on demand
and deliberately not resident.** Keeping it out of the always-loaded instruction file is the point:
it charges nothing to the sessions that never need it, and a statement only a judgement about worth
ever reads should not be billed to every session on every project.

It holds the purpose **in the maintainer's own words, quoted not paraphrased**, then one entry per
objective, each saying *what it is*, *why it is one* — again quoting — and **what scoring it as waste
would cost.** That third field is the one that does the work:

> **Measured, by this extraction.** An agent reading the whole corpus with no statement of purpose
> read the human spike queue (§18) as a wall-clock optimisation, found it loses on token cost, and
> recommended deleting it. The measurements were right and the conclusion was exactly backwards.
> Either standing value lens can reach the same wrong verdict on any arc touching that machinery,
> and a `why` finding lands in the `ask` bucket, which the supervisor is forbidden to argue down.

**What belongs in it is any objective not legible from the mechanism itself.** In the originating
system that was two: that the maintainer stays in **surface contact with the code and in flow**
(§18 — they are responsible for delivering the code, and each wait between todos fills their head
with unrelated input), and that **constant improvement is an objective rather than overhead** (§19's
observation store, §16's challenge phase, §21's watch orders — judged against the todos one arc
ships, every one is pure cost). Both look like waste to a reader with only the diff to judge by.

Both standing value lenses (§11's `why`, §16's `value`) are told to read it when the repo under
judgement is the one it describes, and to report an `OBJECTIVES` line — `n/a`, `read, verdict
unchanged`, or `read, changed: <what changed>`. The supervisor relays that line into `review.md`
rather than triaging it (§12): it is not a finding.

**Second: a mechanism for retiring a rule.** The system has none. Every rule in the originals
arrived with an argument and a measurement; almost none had a stated condition under which it should
be removed. The one exception — a *revisit trigger* written into the coder's rule 7, naming the
observation that would falsify it — is the best single idea in the corpus and appears exactly once.
**Copy that pattern: when you write a rule from one failure, write beside it what would show the rule
costs more than it saves.**

---

## 26. Installation checklist

Work through this with the user. Tick each line out loud.

**Core**

- [ ] `<home>/agents/arc-coder.md` written (§6), with the slow-gate prohibition intact and its
      fast-gate exception beside it.
- [ ] `<home>/agents/arc-reviewer.md` written (§6) — **the Core variant, the one that runs the
      gate itself** — with **no** `Edit`, `Write` or `NotebookEdit`.
- [ ] `<home>/skills/arc/SKILL.md` (the core) and `<home>/skills/arc/RUN.md` written (§7), with the
      build and test gates from Q2 filled in everywhere the template says `<build gate>` /
      `<test gate>`. **Then grep both files for `<build gate>` and `<test gate>` and confirm zero hits**,
      and check the core with `wc -m`: at most 14,000 characters (§5). A placeholder written to disk
      is the most common silent misinstall of this system; §24 Q2 says what to do when there was no
      user to ask, and it is never to leave the placeholder.
- [ ] `<home>/plans/` exists.
- [ ] Optional, where the harness has a hook on compaction: the post-compaction re-read hook
      (§4, *Carrying on through compaction*; its text in §8), adapted to the user's paths.
- [ ] `chronic` is available, or step (d) has been rewritten to use the build command's own quiet
      form.
- [ ] The repo's agent-instruction file actually loads (Q4). If not, the one-line fix was offered.
- [ ] A first arc has been planned by hand — three todos, written out per §3 — and run end to end.

**Standard**

- [ ] `arc-gatekeeper.md` written (§10) and the briefs in `RUN.md` updated to spawn it alongside the
      reviewer **in one message**.
- [ ] `arc-reviewer.md` **overwritten** with the Standard variant (§10), which runs no gate. Two
      reviewer files on disk, or a Core reviewer left in place beside a gatekeeper, is the failure to
      look for here.
- [ ] Step (c) carries the combination rule and the five-bounce cap.
- [ ] `arc-review-what.md`, `arc-review-how.md`, `arc-review-why.md` written (§11).
- [ ] `skills/arc/PLAN.md` written (§9), including the pre-plan step and the Value-bullet standard.
- [ ] `skills/arc/TEST.md` and `arc-triage.md` written (§14).
- [ ] The core routes all three phases (§8, *The routing the core gains*), and is still under its
      cap.
- [ ] There is exactly **one** arc skill, `<home>/skills/arc/`, and one `RUN.md` in it — §10 through
      §13 were merged into the file §7 wrote, and no `run-arc`, `plan-arc` or `test-arc` skill
      directory exists. Check by listing `<home>/skills/`; this failure is silent otherwise.
- [ ] The two statuses (§3, §8) are in every phase text and they agree: each PR's in its `### PR <k>`
      heading (`planning`/`running`/`accept`/`done`), the arc's in its bullet
      (`planning`/`underway`/`done`/`aborted`), and no stop for the maintainer writes either. Every
      index has at least `### PR 1`. `review.md` opens with the review log (§11), and every reader
      that must tell a PR still being read from one waiting on the maintainer reads its last line.
- [ ] `PLAN.md`'s step 5 ends with the **pre-challenge gate** (§9), and step 7 shows the *same*
      index summary rather than a second list.
- [ ] Every index entry carries **summary bullets** (§3), in `PLAN.md`'s template and in both
      places that append to an index afterwards (§12, §14).
- [ ] Replanning (§9) is defined once, in `PLAN.md`, and cited rather than restated from
      `RUN.md`'s `needs-replan` and `design-question` branches, its review-fix arc (§12) and
      `TEST.md`'s rounds (§14); `PLAN.md` has its re-entry path and step 7's approval writes; the
      loop picks every open todo by status, never by id, and `--only` / `--from` take a todo id.
- [ ] The session carries on through compaction (§4, §8): no phase ends the session, prints a line
      for a fresh one except for resuming, or reads its own context occupancy; the planning scratch
      file is written before a plan directory exists and folded in at step 5; and, where the harness
      offers it, the compaction hook is installed and names the core, the phase text and the arc's
      `README.md`. The old session-rule script, if a previous install wrote one, is gone.

**Optional — only what the user chose**

- [ ] Challenge phase: three lens briefs, the `CHALLENGE.md` companion read from step 6 of `PLAN.md`
      and its two other triggers, the `challenge.md` format including
      the `exited` disposition, the empirical-exit rule and the `(replan)` round (§16), **and the two
      counts at the top of `review.md`**, which leave `(replan)` rounds out.
- [ ] `OBJECTIVES.md` written (§25) if any objective of this setup is not legible from the mechanism
      — with the `OBJECTIVES` line added to whichever value lenses were installed, and the
      relay-don't-triage rule in `RUN.md` (§12).
- [ ] UI tester: the brief, plus the "is the app up?" check **before the loop**, in both phase texts.
- [ ] Human coding seat and spike queue: only for a maintainer who actually codes alongside the
      agents, and only on a workspace-capable VCS. Quote §18's costs **and** its objective together.
      `HUMAN.md` and `CONCURRENCY.md` are companions beside `RUN.md`, read on their triggers; who
      codes is written as a default and stated, never asked.
- [ ] Observations store: the protocol file, the `FINDING:` line added to every brief, and the
      *Constant improvement* section in `RUN.md`, cited from the core, with the token-saving question
      at every handoff (§19).
- [ ] Citation gate: only for a prose corpus.
- [ ] Watch orders: only if the user writes agent instructions as a deliverable. With the
      observations store also installed, the store's role closes a meta-arc on evidence (§21), and
      every other arc is still closed by the maintainer's word alone. If the user keeps a portable
      extraction of their system, the meta-arc's last handoff announces that one is owed (§21). The
      check lives in `WATCH.md`, read at every handoff. The reviewer's instruction-prose check
      (§21) is offered to anyone whose arcs edit model-read text.
- [ ] Journeys (§8): only for a user whose work regularly outgrows one arc. Every new plan is
      written nested, and older shapes on disk are left as they are; every phase tells an arc
      directory by its `todos/`, never by a House rules section; a journey argument never runs an
      arc at `planning`; the next arc can be planned beside a run; observation files and workspaces
      are named by arc id; a
      journey's `over` is the maintainer's answer to a question, never written on the writer's own.
- [ ] Grouping an arc into pull requests (§8): only for a team whose arcs reach review as pull
      requests and are sometimes too large for one. The grouped rules sit in `GROUPED.md`, an
      on-demand companion beside `RUN.md`, read only when the index has more than one `### PR`
      heading or a round on another PR, and the run stops if that read fails; one definition of *a
      PR holds an open todo* (§4), cited everywhere; a PR waiting on the maintainer beside a later
      `running` one is never settled for routing; a PR's end carries straight on into the next; the
      lens and triage briefs get one PR's diff.
- [ ] With a pull-request workflow (§24 Q6): the reviewed guard, not a published one; the forge read
      at every session start, PR boundary, seal and compaction, with its `**Forge read:**`
      timestamp, in the `PR.md` companion; each PR's URL and bookmark recorded.

**Finally, tell the user three things:**

1. **The first arc will feel like overhead.** It is. The payoff starts at about three todos and grows
   from there.
2. **The one rule to never bend is that the supervisor does not write code.** Everything else here is
   negotiable; that one is what keeps the context flat and the review independent.
3. **A pull request is not finished when the agents run out of findings.** It is finished when they
   say so.
