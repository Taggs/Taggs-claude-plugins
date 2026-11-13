---
name: gh-proj-setup
description: "Make a project or repo work-session compliant: give it a project board where every issue and milestone carries a status and start/target dates, a docs/ folder holding PRD, architecture, decision-log, deploy and next-steps docs with a research/ subfolder for everything else, and its own Discord channel and webhook so its notifications stay separate from other projects'. Use whenever the user asks to set up, configure, onboard, standardise or 'make compliant' a project or repo, when starting work in a repo that has no board, no milestones or no docs/ folder, or when they invoke /gh-proj-setup — and offer it when a close-out audit keeps finding the same structural gap, because that means the scaffolding, not the session, is what's missing. Its counterpart is gh-proj-session-close, which audits at the end of a session what this skill puts in place."
---

# Session setup

A project is **work-session compliant** when a session can start and end without anyone
reconstructing state from memory. Four things make that true:

1. Every piece of work is an issue, on **one** board, carrying a Status, a Type, a Priority, one
   core label, a Target date and a Start date — and its real blockers as relationships.
2. Issues roll up to milestones that have due dates. Milestones are the only phase axis.
3. Five core docs exist and are where they're expected, and everything else is in `docs/research/`.
4. The project has its own Discord channel and webhook, so its notifications are separable from
   every other project's.

The vocabulary and the field rules live in `references/work-item-taxonomy.md`. **Read it before
Step 1** — it is what "compliant" means in detail, and both this skill and `gh-proj-session-close`
enforce exactly it.

This skill puts those in place. It is idempotent — run it on a half-configured project and it
fills only the gaps.

## Authority

**Creating is safe; overwriting is not.** Create missing boards, fields, milestones, doc stubs and
folders directly. Never overwrite an existing doc, never renumber an existing decision log, never
change a status or date someone already set — report those and propose.

**Never invent product intent.** A PRD written from guesswork is worse than no PRD, because the
next reader believes it. Interview for anything that can't be derived from the code.

## Step 0 — Survey before changing anything

Produce the compliance table first, so the user sees the size of the job and can cut it down:

```
Org-level        <issue fields and issue types the org already defines — CHECK FIRST>
  Issue fields   <name [type] options…>   ← these outrank any project field of the same meaning
  Issue types    <Task / Bug / Feature / …>
Board            ✓ / ✗ / partial   <which board, or "none found">
  Single board   ✓ / ✗             <issues appearing on more than one board>
  Duplicates     <project fields that duplicate an org issue field>
  Status field   ✓ / ✗             <options>
  Priority field ✓ / ✗             <options>
  Start date     ✓ / ✗
  Target date    ✓ / ✗
  Issues on it   n of m
Issue fields     ✓ / ✗ / partial
  Type set       n of m            <org types available?>
  Priority set   n of m
  One core label n of m
  Target date    n of m
  Relationships  <blockers stated in prose but not linked>
Milestones       ✓ / ✗ / partial   <n with due dates, of m>
  Phase/epic field or label duplicating milestones?   ✓ / ✗
docs/PRD.md                   ✓ / ✗
docs/ARCHITECTURE-current.md  ✓ / ✗
docs/ARCHITECTURE-decisionlog.md ✓ / ✗
docs/DEPLOY.md                ✓ / ✗
docs/next-steps.md            ✓ / ✗
docs/research/                ✓ / ✗   <n loose docs that belong there>
Discord channel + webhook     ✓ / ✗
```

Check your project conventions first — `~/.config/claude/known-projects.md` if it exists,
otherwise `references/known-projects.md`. If the workspace is listed, its conventions replace the
defaults below, and several boxes may already be ticked in ways the defaults wouldn't predict.
Read the repo's `CLAUDE.md` too.

Then confirm the plan in one pass rather than asking per item. Setup is a lot of small decisions;
batching them is the difference between five minutes and an interrogation.

## Step 1 — The board

Commands are in `references/github-board-recipes.md`.

### First: check for org-level fields, before creating anything on the board

**An org can define issue-level fields that every board mirrors read-only.** Look before you build,
or you will create a project field that duplicates one the org already has — and the two then
disagree, which is the exact failure this skill exists to prevent.

```bash
gh api orgs/<ORG>/issue-fields --jq '.[]|"\(.name) [\(.data_type)] \([.options[]?.name]|join(", "))"'
gh api orgs/<ORG>/issue-types  --jq '.[]|"\(.name) — \(.description)"'
```

**Whatever the org defines wins.** Prefer an org issue field over a project field of the same
meaning, always — it lives on the issue, follows it across every board, shows in the issues UI, and
survives a board being rebuilt. Create a project field only for something genuinely board-local,
such as an iteration.

Three things that will otherwise cost you an hour:

- A mirrored field **cannot be written through the project**. `updateProjectV2Field` answers
  *"Only custom fields can be updated. Fields derived from issues or pull requests must be updated
  through their respective APIs"* — that message means org field, not permissions. Write the value
  with the `setIssueFieldValue` mutation instead.
- A mirrored field **reads back empty everywhere you'd normally look**: `gh project field-list`
  shows it with no options, and `gh project item-list` shows no value. Neither is evidence it is
  unset. Its values are a distinct GraphQL type (`ProjectV2ItemIssueFieldValue`), so check the
  issue with `gh api repos/O/R/issues/N --jq .issue_field_values` instead — that is the honest read.
- If the org already has, say, `Priority`, `Start date` and `Target date` as issue fields, then
  rules 5 and 7 are satisfied at the **issue** level. A project-level `Start date` alongside an org
  `Start date` is duplication to flag, not a second place to write.

Then the board itself:

- Find or create it: `gh project list --owner <org>`, else
  `gh project create --owner <org> --title "<name>"`. Org-level beats repo-level as soon as a
  second repo is involved.
- **One board holds everything.** Check every board in the org for overlap first. An issue on two
  boards has two Statuses and two sets of dates, and nothing downstream can be trusted. Where
  several boards exist, consolidate onto one and use the built-in **Repository** field (derived, no
  maintenance) plus milestones to separate streams — not a second board.
- **Status** (single select): `Backlog / Plan / Ready / In progress / In review / Done`. Create all
  six at once — adding one later replaces the whole option set and **clears every stored value**
  (see the recipes file). Keep a board's existing options if it already has a working set, but
  `Ready` must exist, because it's what `docs/next-steps.md` maps onto. Resist adding *more* —
  boards with nine statuses get used as boards with two.
  What each status means, and why `Plan` exists, is in taxonomy §8.
- **Priority** (single select): `Urgent / High / Medium / Low` — **only if the org has no Priority
  issue field**. If it does, it is already there and already correct; don't add a second one.
- **Start date** and **Target date**, both `DATE` — again, only if the org doesn't define them.
  Boards made from the default template have neither. Without them, at one level or the other, the
  board can't answer "what is due", which is the question it exists for.
- Optionally `Size` if the user plans by it. Don't add fields nobody will fill, and **never** add a
  Phase or Epic field — that's what milestones are.
- **Add every open issue.** An issue that isn't on the board is invisible to planning, and the ones
  that fall off are reliably the awkward ones.

A field's single-select options can't be edited through the CLI once created. If an existing field
has the wrong options, check whether any item actually uses it: if none do, deleting and recreating
costs nothing; if some do, it needs the GraphQL mutation or the user's hands in the UI — and
**the mutation clears every stored value on that field**, so snapshot the board first and plan to
restore. The UI does not. Prefer the UI.

Setting values:

- **Status** from evidence — closed → Done, open PR → In review, work underway → In progress,
  named in `docs/next-steps.md` → **Ready** when it is unblocked and nothing needs deciding first,
  **Plan** when it is refined but waiting on a decision or has an open `blocked_by`, everything
  else → Backlog. **Parked work is Backlog, never Ready**: deferred items, items
  gated on work nobody has started (a later phase, a gold set, a paid account), and watch-only items
  with nothing to do until something happens. `Ready` means someone could start it today.
- **Start date**: the first commit or PR on that work where you know it (mark it actual), otherwise
  the estimate — target minus expected effort in working days.
- **Target date**: required on every issue. Order the open work into waves by dependency, give each
  wave a date, then show the whole list for one confirmation pass. Don't fabricate deadlines
  silently — a made-up date nobody agreed to looks exactly like a real commitment on the board, and
  it poisons every later "is this slipping?" question. Say which dates are low-confidence.

## Step 2 — Milestones

Milestones are the **only** phase axis. If the project also has a Phase or Epic board field, or
`phase:*`/`epic:*` labels, migrate their values into milestones and retire the duplicate — see
`references/work-item-taxonomy.md`.

- Every milestone needs a **due date**, computed **bottom-up**: the latest Target date among its
  open issues (for an all-closed milestone, the last `closed_at`). A due date invented top-down and
  contradicted by its own contents is worse than none.
  `gh api -X PATCH repos/O/R/milestones/<n> -f due_on=2026-10-15T00:00:00Z` — full ISO timestamp;
  a bare date is rejected.
- Every open issue belongs to a milestone, or is explicitly backlog. Say which; don't leave it
  ambiguous.
- Milestones are **phases with an end**, not buckets. "Phase 2: payments live" closes;
  "Improvements" never does, and a milestone that can't close tells you nothing about progress.
- A cross-repo phase becomes a same-named milestone in **each** repo with work in it — GitHub
  milestones don't span repos.
- Close any milestone that is empty or fully delivered. Never leave an open issue inside a closed
  milestone; `gh api .../milestones` defaults to `state=open`, so query `?state=all` or you won't
  see that case.
- Where a project has none, propose three or four covering the next quarter, derived from the open
  issues rather than from ambition.

**Two phase taxonomies over the same issues is a conflict, not a merge.** An issue holds one
milestone, so when a product phase and a delivery phase both claim it, collect every affected issue
and put the choice to the user in one pass with a recommendation — usually that the nearer deadline
wins and the other milestone keeps only its delivered work.

## Step 3 — Issue fields

Backfill every issue so nothing depends on remembering. Definitions and the reasoning are in
`references/work-item-taxonomy.md`; commands in `references/github-board-recipes.md`.

- **Type** (taxonomy §3), **Priority** (§5) and **exactly one core label** (§4) on every issue.
  Types are org-wide, so adding one is the user's decision, not a session's. Leave the user's own
  labels alone; flag any that duplicate a field.
- **Relationships**: convert blockers stated in prose into real blocked-by links, and checklist
  subtasks into sub-issues. Only link what an issue, a decision record or an obvious ordering
  actually states — an invented blocker stops work that could have proceeded.
- **Assignee** on everything with an owner.

## Step 4 — Docs

Create what's missing from `assets/templates/`. Each template is a skeleton: headings plus a line
saying what belongs under each. Never fill a template with placeholder prose that reads like
content — a fabricated PRD is read as a real one.

| File | How to fill it |
|---|---|
| `docs/PRD.md` | **Interview.** Who it's for, the problem, what's in and out of scope this phase, how you'll know it worked. Not derivable from code. |
| `docs/ARCHITECTURE-current.md` | Draft from the code, then have it checked. This one you can do most of. |
| `docs/ARCHITECTURE-decisionlog.md` | Starts with the appending rules. Add `D-001` only if there's a real decision to record — an empty log with good rules beats a padded one. |
| `docs/DEPLOY.md` | How it ships, env vars **by name only, never values**, health check, rollback. |
| `docs/next-steps.md` | Seed with current state and a first 5-hour plan. `gh-proj-session-close` maintains it after that. |
| `docs/research/` | Create it, then move loose non-core markdown in. Leave `README`, `CLAUDE.md`, `CONTRIBUTING`, `LICENSE`, `CHANGELOG` where they are. |

Finally, add a short conventions block to `CLAUDE.md` (create it if absent) naming this doc set and
the board URL, so a future session finds them without being told. That line is what makes the setup
stick.

## Step 5 — The project's own Discord channel

One channel per project, so an alert tells you which project it came from by where it landed rather
than by how carefully you read it. Full walkthrough in `references/discord-setup.md` — read it
before starting, since the channel itself has to be created by the user in the Discord UI.

The short version:

1. Check for an existing webhook: `~/.config/<project-slug>/discord-webhook`. If one is there,
   nothing to do — verify and move on.
2. Ask the user to create the channel and a webhook on it, and to paste the URL.
3. Store it at `~/.config/<project-slug>/discord-webhook`, `chmod 600`. **Never print the value**,
   not in a report, not in an echo, not in a test command you show.
4. Verify with one test post using a real User-Agent.
5. Record the **path** in `docs/DEPLOY.md` under the env/secrets section, and the env var name the
   app reads it from. The path is documentation; the URL is a credential.

## Step 6 — Verify and report

Re-run the Step 0 survey and show before → after. Then, terse:

```
Created    — what now exists that didn't
Filled     — statuses, dates, milestones set
Needs you  — the PRD interview, target dates to confirm, the Discord channel
```

Anything still `✗` gets a line saying what it's waiting on.
