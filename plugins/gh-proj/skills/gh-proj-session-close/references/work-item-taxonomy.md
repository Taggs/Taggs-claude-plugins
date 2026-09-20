# Work-item taxonomy — the standard

The vocabulary rules that both `gh-proj-setup` and `gh-proj-session-close` enforce. They exist because
the same idea drifts into three names otherwise — epic/phase/milestone, task/subtask/issue — and
once it has, no filter across the project can be trusted.

Where a project already has a different convention documented in its `CLAUDE.md`, that wins; record
the divergence rather than converting it silently.

**Where a field lives matters as much as what it holds.** If the org defines an issue field
(`gh api orgs/<ORG>/issue-fields`) for Priority, a date or anything else below, that is the field —
it sits on the issue, follows it across every board, and survives a board being rebuilt. A project
field of the same meaning is a duplicate, and duplicates disagree. Check the org before creating
anything, and read values from `gh api repos/O/R/issues/N --jq .issue_field_values`, because a
mirrored field reads back empty through the project CLI whether or not it is set.

## 1. Milestone is the only word for a phase

Epics and phases **are** milestones. Never a separate `Phase` board field, never `epic:*` or
`phase:*` labels, never an umbrella issue standing in for one. Those all fragment the same axis.

- A milestone is a **phase with an end**, so it can close. "Launch P2 — Web Store" closes;
  "Improvements" never does and tells you nothing.
- Milestones are per-repo in GitHub. A cross-repo phase becomes a same-named milestone in each
  repo that has work in it.
- An issue can hold **exactly one** milestone. Two competing phase taxonomies over the same issues
  is a conflict to escalate, not to resolve quietly — see *Conflicts* below.
- **Due date required.** Compute it bottom-up: the latest Target date among the milestone's open
  issues. For a milestone with only closed issues, use the last `closed_at`.
- Never leave an open issue inside a closed milestone. Either reopen the milestone or move the
  issue out.

## 2. Issue is the only word for a task

Tasks and subtasks **are** issues and sub-issues. A subtask is a real sub-issue via the sub-issues
API — not a checkbox in the body, which nothing can filter, count or schedule.

## 3. Type — set on creation, always

| Type | Means | Test |
|---|---|---|
| **Task** | Planned work | It was on the plan before anyone started |
| **Bug** | Arising or unplanned work | It appeared because of something that happened — a defect, a review finding, a forced cleanup |
| **Feature** | A collection of tasks | It has sub-issues |

Planned versus arising is the primary split, and Type is what carries it. A Task that acquires
sub-issues is **upgraded to Feature** at that moment; a Feature that loses its last sub-issue goes
back to Task.

Note the consequence and accept it deliberately: unplanned non-defect work is typed `Bug`, so bug
counts measure unplanned work rather than defects. If a project wants defect rate as a signal, it
needs a fourth type, which is an org-level decision, not one to make during a session.

## 4. Labels — exactly one of three

`enhancement`, `bug`, `documentation`, chosen by **what the change is**, not by how it arose. An
unplanned docs sweep is Type `Bug` and label `documentation`; the two fields answer different
questions.

Anything else in the label list belongs to the user. Never auto-add and never auto-remove those —
but *do* flag labels that duplicate a field (`phase:*` against milestones, `priority:*` against
Priority, `status:*` against Status or Relationships) and propose removing them, because a
duplicated field is guaranteed to disagree with itself eventually.

## 5. Priority — set on every issue

`Urgent` / `High` / `Medium` / `Low`. **Medium is the default** whenever the issue text doesn't
justify something else. An unset Priority is worse than a wrong one: it silently drops the issue
out of any priority-ordered view.

## 6. Relationships — in the field, not in the prose

"Blocked on the API key" written in an issue body is invisible to every query. Use the real
fields:

- **blocked-by / blocks** — the issue dependencies API. Works across repos in the same org.
- **parent / child** — the sub-issues API.

Infer a link only from something explicit: the issue says it, a decision record says it, or one
issue plainly cannot start until another ships. Don't invent a dependency graph from theme
similarity — a wrong blocker stops work that could have proceeded.

## 7. Dates — Target required, Start expected

- **Target date on every issue.** No exceptions; it's what makes the board answer "what's due".
- **Start date on every issue too**, *estimated* until it is *actual*. The estimate is the target
  minus the expected effort in working days. It becomes actual the first time Status moves to
  **In progress** — that transition is the moment to overwrite it.
- Start must never be after Target.
- **A target date that has already passed is never rewritten silently.** Surface the slip and let
  the owner set the new date. A board that heals its own deadlines is not evidence of anything.

## 8. Status — and the next-steps rule

`Backlog` → `Ready` → `In progress` → `In review` → `Done`.

- **Any issue named in `docs/next-steps.md` is `Ready`.** That file is the statement of what's next,
  so the board must agree with it. The rule runs **one way only**: being named promotes an issue to
  `Ready`; being absent never demotes one. The memo plans the next session — it is not a census of
  everything that is ready, and demoting on absence sweeps dated, unblocked, launch-critical work
  into `Backlog` and makes the board lie.
- Never walk a status backwards. `In progress`, `In review` and `Done` are further along than
  `Ready`; leave them.
- Closed issue → `Done`. Open PR linked → `In review`. `Backlog` means **not yet refined**, so send
  an issue there only when it genuinely lacks definition — not merely because nothing is scheduled.

## Conflicts — stop and ask

These cannot be resolved by a rule, so collect them and put them to the user with a recommendation:

- One issue that two phase taxonomies both claim.
- A milestone whose name implies a phase that another milestone also covers.
- An issue whose type is genuinely ambiguous because nobody remembers whether it was planned.
- A label that might be a user's own taxonomy rather than a field duplicate.
- Any target date that has already passed.

List them all at once with a one-line recommendation each. A conflict list of fifteen items the
user can answer in one pass beats fifteen interruptions.
