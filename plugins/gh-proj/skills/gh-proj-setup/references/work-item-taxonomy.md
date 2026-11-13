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

**Personal accounts are the one exception.** Issue fields exist only for org-owned repos, and not
on draft items. A board over a personal repo (or drafts) uses project-native `Priority`
(Urgent / High / Medium / Low), `Start date` and `Target date` instead — same names, same options,
so the rules below read identically. Never add `End date`, `Phase`, or a P0–P3 scale.

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

Types are the org's own (`gh api orgs/<ORG>/issue-types`), and their descriptions are the
definitions:

| Type | Org description | Use it for |
|---|---|---|
| **Task** | A specific piece of work | Chores, refactors, docs, infra, follow-ups — work that is neither a defect nor new functionality |
| **Bug** | An unexpected problem or behavior | A defect: something behaves differently from what was intended |
| **Feature** | A request, idea, or new functionality | Something a user or stakeholder would notice as new or changed capability |

Type says what the work **is**. It does not say whether it was planned, and it does not depend on
sub-issues: any type can have a parent or children, and nothing is "upgraded" when one is added.
Work discovered mid-session is a `Bug` only if it is a defect; otherwise it is a `Task` or `Feature`.

## 4. Labels — exactly one of three

`enhancement`, `bug`, `documentation`, chosen by **what the change is**. They must agree with Type:

| Type | Core label |
|---|---|
| Bug | `bug` |
| Feature | `enhancement` |
| Task | any of the three — `documentation` for docs work, `enhancement` for improvements, `bug` never |

A `bug` label on a non-Bug, or a Bug without the `bug` label, is a contradiction to fix. Type is the
authority; correct the label.

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

`Backlog` → `Plan` → `Ready` → `In progress` → `In review` → `Done`.

The three pre-work statuses answer **what is the next action**, not how important the issue is:

| Status | Means | The next action is |
|---|---|---|
| `Backlog` | Not yet refined | Write down what it actually is |
| `Plan` | Refined, but a **decision or design** stands between it and code | Someone has to *choose* something |
| `Ready` | Executable now: refined, dated, unblocked | Pick it up and write code |

`Plan` exists because "refined" and "actionable" are different things, and collapsing them forces
two bad options: park a fully-specified issue in `Backlog` and imply it needs definition, or mark it
`Ready` and imply someone can start. Both make the board lie. Typical `Plan` items: a question only
the owner can answer, a design that must be agreed before implementation, an issue whose *work* is
the decision.

**Move `Plan` → `Ready` when the open question is answered**, not when someone feels ready. If you
cannot name the question, the issue is `Ready` (or `Backlog`).

- **Any issue named in `docs/next-steps.md` is at least `Plan`.** That file is the statement of
  what's next, so the board must agree with it. The rule runs **one way only**: being named
  promotes; being absent never demotes. The memo plans the next session — it is not a census of
  everything that is ready, and demoting on absence sweeps dated, unblocked, launch-critical work
  into `Backlog` and makes the board lie.
- **Promote to `Ready` only if the issue has no open blockers.** An issue with an unresolved
  `blocked_by` link is not executable, whatever the memo says about it; give it `Plan` (if the
  blocker is a decision) or leave it where it is and say so in the report. Promoting blocked work
  to `Ready` is how a board ends up claiming five things can start when none can.
- Never walk a status backwards. `In progress`, `In review` and `Done` are further along than
  `Ready`; leave them.
- Closed issue → `Done`. Open PR linked → `In review`. `Backlog` means **not yet refined**, so send
  an issue there only when it genuinely lacks definition — not merely because nothing is scheduled.

## Conflicts — stop and ask

These cannot be resolved by a rule, so collect them and put them to the user with a recommendation:

- One issue that two phase taxonomies both claim.
- A milestone whose name implies a phase that another milestone also covers.
- An issue whose type is genuinely ambiguous (a defect or a change of intended behaviour?).
- A label that might be a user's own taxonomy rather than a field duplicate.
- Any target date that has already passed.

List them all at once with a one-line recommendation each. A conflict list of fifteen items the
user can answer in one pass beats fifteen interruptions.
