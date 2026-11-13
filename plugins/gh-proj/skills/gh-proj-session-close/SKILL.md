---
name: gh-proj-session-close
description: "Close out a work session so the record matches reality: audit the project board and milestones for wrong statuses, wrong dates and taxonomy drift (missing issue Type, Priority, labels, blockers, epics masquerading as fields) and fix them, open issues for residual work and close the ones that genuinely shipped, check the repo's core docs are collated and current, sweep the session for decisions that never got logged, and rewrite docs/next-steps.md with a plan for the next 5-hour session. Use this whenever the user is wrapping up, finishing, ending, closing out or signing off a session, asks for a handoff or 'what's left', or invokes /gh-proj-session-close — and offer it unprompted when a long working session reaches a natural stopping point, because this is exactly the work that gets skipped when someone is tired. Its counterpart is gh-proj-setup, which builds the board and docs this skill audits."
---

# Session close-out

A session ends in one of two states: the record matches what happened, or it doesn't. When it
doesn't, the cost lands on the next session — reconstructing status from memory, rediscovering a
decision, re-solving a problem that was already solved. This skill spends twenty minutes so that
doesn't happen.

Work through the steps in order. Step 0 is not optional: the audit is worthless if it is run
against the wrong repo or the wrong set of conventions.

## Authority — what to change without asking, and what to propose

| Apply directly | Propose, then wait |
|---|---|
| Board metadata: Status, Start date, assignee, milestone membership, board membership | Any edit to PRD, architecture, API spec, deploy runbook, or any other prose doc |
| Missing date fields on the board | New decision-log entries (append-only — a wrong one stays forever) |
| Milestone due dates that were never set; closing a milestone whose issues are all closed | Moving a target date that has already slipped |
| Opening issues for residual work | Closing an issue whose acceptance criteria you cannot verify |
| Closing issues whose acceptance criteria demonstrably shipped | |
| Rewriting `docs/next-steps.md` — it is a between-sessions memo and is meant to be replaced | |

Never, in any mode: push to a protected branch, merge, deploy, apply a migration, print a secret,
or edit an existing decision-log entry. Leave doc changes uncommitted unless the user asks for a
commit; if they do and the branch is protected, branch and open a PR.

When a proposal is shown, show the actual text — the paragraph as it is and as it would be. "The
architecture doc is out of date" is not a proposal, it's a complaint.

## Step 0 — Scope the session

Establish, before touching anything:

- **Which repos the session touched.** Use the conversation first, then verify:
  `git -C <repo> log --since="<session start>" --oneline`. A workspace with sibling repos will
  often have two in play; each gets its own pass.
- **The session window.** First to last commit or PR of the day, or ask. You need it for date
  accuracy and for "what did we actually finish".
- **Today's date** from the environment, never inferred from file contents.
- **The repo's own conventions.** Read `CLAUDE.md` and the existing `docs/next-steps.md`. Where
  they differ from the defaults below, the repo wins — report the difference instead of
  "correcting" it. Imposing a naming convention during close-out is scope you weren't given.

## Step 1 — Board and milestone audit

Read `references/work-item-taxonomy.md` first — it defines milestone/issue/type/label/priority/
relationship/date/status and is what the checks below enforce. Commands are in
`references/github-board-recipes.md`; read that before the first write, because several of its
traps look like data problems rather than tooling problems.

On a GitHub repo, start with `gh project list --owner <org>`, then pull every item with its fields
in one call and audit from that JSON rather than a call per item. Before counting anything as
missing, check the field key against one raw item — `Target date` reads back as `'target date'`,
and guessing the camelCase spelling makes a fully-populated board look empty.

**One board per issue.** If the same issue appears on two boards it has two Statuses and two sets
of dates, and no rule below can hold. Detect it early: pull every board in the org and intersect.
Consolidating is a structural change, so propose it rather than doing it — but say plainly that
the rest of the audit is unreliable until it's fixed.

**Status drift** — for each, the fix is in the right-hand column:

| Symptom | Fix |
|---|---|
| Issue closed, item not Done | Set Done |
| Merged PR closed the issue, item still In progress | Set Done |
| Open PR linked, item still Todo/Backlog | Set In review |
| Item has no Status at all | Set from evidence; Backlog when there is none |
| Refined issue whose next action is a decision, not code | `Plan` — not `Backlog` (it is defined) and not `Ready` (nobody can start) |
| Item is `Ready` but has an open `blocked_by` | `Plan` if the blocker is a decision, else report it — `Ready` claims it is executable |
| Item is `Ready` but **parked**: deferred, gated on work nobody has started (a later phase, a gold set, a paid account, "bundle with the next run"), or a watch-only item with no action until something happens | **Backlog**, with a one-line comment saying why. `Ready` is only for work someone could start today |
| Issue worked this session still in Backlog | In progress, or Done if it shipped |
| Repo issue that is not on the board | Add it, then give it a Status |
| Item the user owns with no assignee | Assign to their login |

**Date accuracy** — dates are the half of the board everyone forgets, and a board with no dates
can't answer "what is due", which is the only question it exists to answer:

| Symptom | Fix |
|---|---|
| Board has no Start date / Target date field | Create both as DATE fields |
| Done item with no Target date | Set it to the completion date |
| In progress item with no Start date | Set it to the first commit or PR on that work |
| Start date after Target date | Report — one of the two is wrong and you can't tell which |
| Target date passed, item not Done | **Ask.** Don't move it silently |
| Milestone with no due date | Propose one from its remaining items' target dates |
| Milestone with every issue closed | Close it |
| Milestone overdue with open issues | Report, with the open count and what's blocking |

Never rewrite a passed target date (taxonomy §7). Surface each slip as one line — issue, old date,
what stopped it — and let the user set the new date.

**Taxonomy drift** — the fields that decay fastest because nothing forces them at creation time.
Full definitions in `references/work-item-taxonomy.md`.

Before reporting anything as unset, run `gh api orgs/<ORG>/issue-fields`: org fields are only
mirrored onto the board and read back empty there (see the taxonomy's opening section for how to
read and write them). Skipping this reports a fully-populated project as unset.

| Symptom | Fix |
|---|---|
| Issue with no Type | Set it (taxonomy §3) |
| Core label contradicting Type (`bug` on a non-Bug, Bug without `bug`) | Fix the label; Type is the authority (taxonomy §4) |
| Issue with no Priority | Set it; **Medium** unless the text justifies otherwise |
| Issue with zero or two of `enhancement`/`bug`/`documentation` | Set exactly one (taxonomy §4) |
| `phase:*`, `priority:*`, `status:*` labels | Propose removal — they duplicate a field and will disagree with it |
| A blocker stated only in prose ("waiting on the API key") | Create the real blocked-by link |
| Subtasks as checkboxes in the body | Propose converting to sub-issues |
| Open issue inside a closed milestone | Reopen the milestone or move the issue out |
| A separate `Phase`/`Epic` board field or label | Propose migrating it to milestones — one phase axis, not two |

Set Type, Priority and the core label directly. Removing a user's own labels, converting checklists
and collapsing a Phase field are proposals: each one might be deliberate.

**Status vs `docs/next-steps.md`** (taxonomy §8): promote every issue the memo names to at least
`Plan`; promote to `Ready` only if it has no open blocker and no unanswered question; never demote
an issue for being absent from the memo, and never walk `In progress`/`In review`/`Done` backwards.
Being **parked** is a different reason and does demote: a parked `Ready` item goes to Backlog (the
Status drift table above), because a `Ready` column full of parked work hides what can actually start.

## Step 2 — Issues: close what shipped, open what's left

**Closing.** For each issue the session claims to have completed, open the issue body and check the
acceptance criteria as written. Verify against the merged diff, not the commit message — the
message says what someone meant to do. If one criterion is unmet, leave the issue open and say
which one. Close with a comment naming the PR or commit that did it, and an honest
`state_reason` (`completed` vs `not_planned`).

Watch for the inverse too: GitHub auto-closes an issue when a PR body says `Closes #N`, including
when the PR only meant to reference it. An issue that closed without its criteria being met gets
reopened with a comment saying why.

**Opening.** Everything the session found, deferred, or half-finished and that has no tracker entry
becomes an issue — search first so you don't duplicate one. Each is created complete, because every
field left for later is one that never gets filled:

- a title that names the outcome, and context that will still be actionable in three months
- acceptance criteria — an issue without them is a note, and notes get ignored
- **Type** — `Bug` only if what the session discovered is a defect; otherwise `Task` or `Feature`
- **Priority** (Medium unless the text says otherwise) and exactly one core label
- a milestone, an assignee, and on the board: Status, Start date, Target date
- a link back to the PR or discussion it came from, and a **blocked-by link** if it waits on
  something

A residual item that is genuinely the user's alone — an API key, a document to read, an approval,
a decision only they can make — is still an issue, and it is worth flagging separately in the
report. Those are usually the long pole, and they are the one category that can't be delegated.

## Step 3 — Docs collation

The default core set, at `docs/`:

| File | Holds |
|---|---|
| `PRD.md` | What is being built, for whom, and what counts as done |
| `ARCHITECTURE-current.md` | How the system is shaped now |
| `ARCHITECTURE-decisionlog.md` | Why it's shaped that way — `D-nn` entries, newest first, never edited |
| `DEPLOY.md` | How it ships, what it needs, how to roll back |
| `next-steps.md` | The between-sessions memo |
| `research/` | Everything non-core: investigations, spikes, vendor notes, old plans |

Check and report:

- **Missing core doc.** Report it and offer to scaffold — `gh-proj-setup` does that properly.
- **Loose markdown** at the repo root or sitting in `docs/` that isn't core. Propose moving it to
  `docs/research/`. Leave `README`, `CLAUDE.md`, `CONTRIBUTING`, `LICENSE`, `CHANGELOG` alone.
- **Stale content**: a core doc that describes something this session changed. Name the doc, the
  paragraph, and the replacement text.
- **Cross-references that now lie**: a link to a moved file, or a sentence like "the only X left is
  Y" when Y shipped today. These are cheap to find and they are what makes a reader stop trusting
  the doc set.
- **Duplication**: the same fact or decision stated in two docs. Pick one canonical home and link
  from the other. Two copies drift apart and then neither is believed.

## Step 4 — Decision-log sweep

Walk the session for decisions that changed the shape of the system and never got written down.

A decision qualifies when someone reading the code in six months would ask "why is it like this?",
or when it forecloses an option, or when it was contested and resolved. Naming, formatting, and
which test to write first are not decisions.

For each one, draft an entry: the next free `D-nn`, dated the day it shipped (not the day it was
discussed), stating the decision, the reasoning, and — where there was one — the failure that
forced it. Show the drafts and wait. The log is append-only, so a wrong entry is permanent.

If a decision this session contradicts an existing entry, the fix is a **new** entry that supersedes
it, cross-linked both ways. Never edit the old one. A log that gets rewritten to match the present
tells you nothing about why anything changed, which is the only thing it was for.

## Step 5 — Rewrite next-steps.md

Keep the file's existing header, voice and cross-links; replace the body. The template is in
`assets/next-session-block.md`. Four sections:

1. **Current state** — what is true now, in a few lines. What's deployed, what's green, what's
   blocked.
2. **Residual from this session** — everything started and not finished, plus everything deferred,
   each linked to its issue, each with one clause on why it stopped.
3. **Next session (5 h)** — the plan:
   - Time-boxed items summing to about **4.5 hours**, not 5. A plan that budgets the whole session
     overruns every time, and the overrun always eats the close-out.
   - Each item names its issue and the test that proves it's done.
   - Ordered so anything that blocks other work comes first.
   - Items needing the user personally listed separately, with their own estimate.
   - If the residual list alone exceeds 4.5 hours, say so and propose what gets cut, rather than
     writing a plan that can't happen.
4. **Decisions worth not relitigating** — a pointer to the decision log, not a copy of it.

Don't let this file turn into a session diary. It is a memo about now; history belongs to git and
the issues.

**Every issue this file names must be at least `Plan` on the board (Backlog if parked), and `Ready` if unblocked and not parked** (taxonomy §8, Step 1). Do that write as
part of this step, not as an afterthought — writing the memo and setting the statuses are the same
act, and splitting them is how they drift.

## Step 6 — Report

Terse. These headings, nothing else:

```
Fixed      — board and issue changes actually applied, one line each
Proposed   — doc edits and decision-log entries awaiting your OK
Conflicts  — what needs a decision, one line + a recommendation each
Blocked    — what a permission or tooling limit stopped, and the exact step to finish it
Needs you  — items only the user can move, with links
Next       — the one-line shape of the next session
```

Drop any heading with nothing under it. No preamble, no restatement of the steps.

## Known project conventions

For repos with established setups that differ from the defaults, read
`~/.config/claude/known-projects.md` if it exists — it is the private, machine-local copy and wins.
Otherwise `references/known-projects.md`, which ships as a template. Read whichever applies when
working in a workspace it names.
