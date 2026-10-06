# Working in this repo

Three files, one job: hand the arc system to someone who does not have it.

- **`README.md` — human-written. Never edit it.** Not a typo, not a link, not a reflow.
- **`SEEDLING.md`** — the system itself: the instructions, agent briefs and procedures that an
  adopting agent writes to a user's machine as plain markdown. It is the deliverable.
- **`CHANGELOG.md`** — which revision `SEEDLING.md` is at, and what changed in each. The version
  number lives there and nowhere else.
- **`AGENTS.md`** — this file: what an agent working *in this repo* needs. Procedure lives here, the
  system's own content lives in `SEEDLING.md`, and neither should start holding the other's half.

## If you are setting the system up for someone

`SEEDLING.md` is the material; this is the order to use it in. Section numbers below are its.

1. **Read all of Part I (Core).** It is the irreducible system. Do not skip to the briefs.
2. **Run the tailoring interview in §24.** Six questions. The answers decide what you write. **Q2's
   answer is a literal string that goes into several files**, and the templates carry
   `<build gate>` / `<test gate>` where it belongs. If there is no user available to answer, §24 Q2
   says what to do instead — and it is never to write the placeholder to disk.
3. **Write the Core files** per §5 (layout), §6 (the two agent briefs), §7 (the supervisor skill).
   Core alone is a working system and delivers real value.
4. **Offer the Standard layer (Part II)** and add it if the user wants planning and review phases,
   which most will. **Install it by growing the files you already wrote, never by writing a second
   copy of one.** §9 and §14 add two new skills; everything else in Part II is an edit to the
   `run-arc` skill and to the agent briefs from step 3.
5. **Present the Optional layer (Part III) as a menu with prices**, not as a list of features. Each
   optional piece states what it costs and what it buys. Quote both.
6. Check your work against §26, the installation checklist.

**Do not install all of it by default.** The system this was extracted from had accreted for months
around one person's workflow. A user who gets Core on day one and grows into Standard in week two
ends up with a system they understand. A user handed everything at once ends up with a system they
obey.

**A note on honesty.** Parts of this system are measured and earn their place. Parts are plausible
and unproven. §25 says which is which, including the parts it says not to propagate. Read it before
you recommend anything.

## If you are updating `SEEDLING.md` itself

**Every closed meta-arc on the originating system produces a new revision of that file**, and the
reason is an adopter who already runs an earlier revision: they update their own setup **from the
diff** rather than re-reading several thousand lines to find what moved. A revision that changes
nothing an adopter must act on still gets an entry saying so.

Record it in `CHANGELOG.md`, never in `SEEDLING.md`: bump the version line at the top of that file
and put the new entry above the others, newest first. One bullet each: the version, the date, one
sentence on what the revision is, then only what an adopter has to *do*. A changelog nobody can scan
is not a changelog, and the moment an entry starts narrating the arc that produced it, nobody scans
it.

```
- **vN** — <date> — <one sentence on what this revision is>.
  **To update from v(N-1):** <which files to re-write, or "nothing to do">.
```

**Keep `SEEDLING.md` standing on its own.** An adopting agent reads it end to end with no access to
the system it came from, so a new rule arrives with the reason it exists beside it — in a blockquote
where there is a measurement, and saying plainly where it is one person's habit instead. Never make
its content depend on this file; only the reading order lives here.

**Never renumber its sections.** §1–§26 are cited by number from end to end of the file, and from
the installation checklist in §26. New material attaches to the section it belongs under; it does
not take a number of its own.

**An edit that cannot fail is the failure mode in a file this size.** §16 carries the measured
account — an unasserted string replace whose target was absent wrote the file back unchanged and
reported success, twice, in two separate phases. Use an editing tool that errors on a missing or
ambiguous anchor, or have the script assert its own match count before it writes, and then read the
changed region back. Searching for the string you just typed is not that check.
