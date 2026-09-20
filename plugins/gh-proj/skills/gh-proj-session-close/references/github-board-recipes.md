# GitHub Projects v2 — command recipes

Everything here is `gh` CLI. Read the section you need; don't load the whole file for one lookup.

- [Before anything: the classic-Projects trap](#before-anything-the-classic-projects-trap)
- [Finding the board](#finding-the-board)
- [Reading every item with its fields](#reading-every-item-with-its-fields)
- [Field IDs and option IDs](#field-ids-and-option-ids)
- [Setting a status](#setting-a-status)
- [Setting a date](#setting-a-date)
- [Creating a missing date field](#creating-a-missing-date-field)
- [Adding an issue to the board](#adding-an-issue-to-the-board)
- [Issues: read, close, comment, assign](#issues-read-close-comment-assign)
- [Milestones](#milestones)
- [Org issue fields — check these before creating any project field](#org-issue-fields--check-these-before-creating-any-project-field)
- [Issue types](#issue-types)
- [Labels without deleting](#labels-without-deleting)
- [Relationships: blockers and sub-issues](#relationships-blockers-and-sub-issues)
- [Evidence for "did this actually ship?"](#evidence-for-did-this-actually-ship)
- [Traps worth knowing before you debug them](#traps-worth-knowing-before-you-debug-them)

## Before anything: the classic-Projects trap

On repos linked to a Projects (classic) board, these fail with
`GraphQL: Projects (classic) is being deprecated ... (repository.issue.projectCards)`:

- `gh issue view N` (bare), `gh issue edit`, `gh pr edit`

The failure is in the default view's GraphQL selection set, not in your permissions. Working
substitutes, which is what the rest of this file uses:

| Wanted | Use |
|---|---|
| Read an issue | `gh issue view N --json title,body,state,labels,milestone,assignees` |
| Edit an issue | `gh api -X PATCH repos/O/R/issues/N -f title=... -f state=closed` |
| Edit a PR (base, body) | `gh api -X PATCH repos/O/R/pulls/N -f base=main` |
| Assign | `gh api -X POST repos/O/R/issues/N/assignees -f 'assignees[]=LOGIN'` |
| PR file stats | `gh pr view N --json files` (`gh pr diff --stat` does not exist) |

`gh issue create`, `gh issue list`, `gh project *` and every `gh api` call are unaffected.
Try the plain command once; fall back the moment you see that error rather than debugging it.

## Finding the board

```bash
gh project list --owner <ORG>            # number, title, ID (PVT_...)
gh project view <N> --owner <ORG> --format json
```

A repo can feed several org boards. When more than one exists, pick by which one already holds
issues from this repo — `gh project item-list` and look at the `repository` field — rather than by
title.

## Reading every item with its fields

```bash
gh project item-list <N> --owner <ORG> --limit 200 --format json > /tmp/items.json
```

Each item carries `id` (the item ID, `PVTI_...`), `content` (`number`, `title`, `url`, `type`,
`repository`) and the field values keyed by field name, **lowercased with spaces kept** — `Target
date` reads back as `'target date'`, not `targetDate`. Fields with no value are simply absent from
the object, which is exactly the drift you are hunting — so a guessed key and a genuinely empty
field are indistinguishable. Print one raw item and read the real keys before trusting any count.

## Field IDs and option IDs

Every write needs the field's node ID, and a single-select write also needs the option's ID.

```bash
gh project field-list <N> --owner <ORG> --format json
```

`options[]` on a `ProjectV2SingleSelectField` gives each option's `id` and `name`. Cache these once
per run; they don't change mid-session.

## Setting a status

```bash
gh project item-edit \
  --project-id <PVT_...> --id <PVTI_...> \
  --field-id <status field id> --single-select-option-id <option id>
```

One field per invocation — batch by looping, not by passing several flags.

## Setting a date

```bash
gh project item-edit --project-id <PVT_...> --id <PVTI_...> \
  --field-id <date field id> --date 2026-09-19
```

`--clear` instead of `--date` removes the value.

## Creating a missing date field

**Check `gh api orgs/<ORG>/issue-fields` first.** If the org already defines `Start date` /
`Target date`, they exist on every issue and are mirrored onto every board — creating project
fields of the same name builds a duplicate that will silently disagree later.

Only when the org defines none:

```bash
gh project field-create <N> --owner <ORG> --name "Start date"  --data-type DATE
gh project field-create <N> --owner <ORG> --name "Target date" --data-type DATE
```

Match the existing naming if the board already uses different labels; two fields meaning the same
thing is worse than one badly named one.

**Retiring a duplicate**, once you find one: copy every value across first — including on *closed*
items, where someone may have set a date by hand that the other field never received — diff the two
sets until nothing is lost, and only then `gh project field-delete --id <PVTF_...>`. Deleting a
field deletes its values with it, and there is no undo.

## Adding an issue to the board

```bash
gh project item-add <N> --owner <ORG> --url https://github.com/O/R/issues/123
```

Returns the new item ID, which you then need for the status and date writes.

## Issues: read, close, comment, assign

```bash
gh issue list --repo O/R --state open --limit 200 \
  --json number,title,state,labels,milestone,assignees,updatedAt

gh issue view N --repo O/R --json title,body,state --jq '.body'     # acceptance criteria live here

gh api -X POST repos/O/R/issues/N/comments -f body="Closed by #77 (merged 2026-09-19)."
gh api -X PATCH repos/O/R/issues/N -f state=closed -f state_reason=completed
gh api -X POST repos/O/R/issues/N/assignees -f 'assignees[]=LOGIN'
```

`state_reason` is `completed` or `not_planned`; pick honestly — a deferred item closed as
`completed` is a lie the board can't recover from.

Opening one:

```bash
gh issue create --repo O/R --title "..." --body-file /tmp/body.md \
  --milestone "Phase 2" --assignee LOGIN
```

## Milestones

```bash
gh api repos/O/R/milestones --jq '.[] | "\(.number) \(.title) due=\(.due_on) open=\(.open_issues) closed=\(.closed_issues)"'

gh api -X PATCH repos/O/R/milestones/<N> -f due_on=2026-10-15T00:00:00Z   # ISO 8601, Z
gh api -X PATCH repos/O/R/milestones/<N> -f state=closed
gh api -X POST  repos/O/R/milestones -f title="Phase 4: ..." -f due_on=2026-11-01T00:00:00Z
```

`due_on` must be a full timestamp; a bare date is rejected.

**`gh api .../milestones` defaults to `state=open`.** Pass `?state=all` or you will miss a closed
milestone that still contains open issues — a common and invisible piece of drift.

## Org issue fields — check these before creating any project field

An org can define fields that live on the **issue** and that every board mirrors read-only. They
outrank project fields of the same meaning: they follow the issue everywhere and survive a board
being rebuilt.

```bash
gh api orgs/<ORG>/issue-fields --jq '.[]|"\(.name) [\(.data_type)] \([.options[]?.name]|join(", "))"'
```

**Three traps, all of which look like something else:**

1. `gh project field-list` shows a mirrored field **with no options**, and `gh project item-list`
   shows **no value** for it. Neither means unset — the CLI simply doesn't read derived fields.
2. `updateProjectV2Field` on one fails with *"Only custom fields can be updated. Fields derived
   from issues or pull requests must be updated through their respective APIs."* That is the
   diagnostic: it's an org field, not a permissions problem.
3. On the board its values are a distinct GraphQL type, `ProjectV2ItemIssueFieldValue`, not
   `ProjectV2ItemFieldSingleSelectValue` — a query selecting only the latter reports zero.

**The honest read** is the issue itself:

```bash
gh api repos/O/R/issues/N --jq .issue_field_values
```

**The write** is one mutation that can set several fields at once. Node IDs come from the org:

```bash
gh api graphql -f query='
{ organization(login:"<ORG>"){ issueFields(first:20){ nodes{
    ... on IssueFieldSingleSelect { id name options{ id name } }
    ... on IssueFieldDate { id name } } } } }'
```

```bash
NID=$(gh api repos/O/R/issues/87 --jq .node_id)
gh api graphql -f query="
mutation { setIssueFieldValue(input:{ issueId:\"$NID\", issueFields:[
  {fieldId:\"IFSS_…\", singleSelectOptionId:\"IFSSO_…\", rationale:\"why this value\"},
  {fieldId:\"IFD_…\",  dateValue:\"2026-09-23\"}
]}){ clientMutationId } }"
```

`issueFields` is a list, so Priority plus both dates go in **one** call per issue. The input also
takes `confidence` and `suggest: true` — the latter stores the value as a pending suggestion for
human review instead of applying it, which is the right choice when a value is a guess the owner
should confirm. `rationale` (≤ 280 chars) is worth filling on anything inferred; it is the only
record of *why* a value was chosen.

## Issue types

Org-level types (`Task`, `Bug`, `Feature` by default) are set per issue by name:

```bash
gh api orgs/<ORG>/issue-types --jq '.[]|"\(.name) — \(.description)"'   # what exists
gh api -X PATCH repos/O/R/issues/N -f type=Task --jq '"#\(.number) \(.type.name)"'
```

Setting a type the org hasn't defined fails. Types are org-wide, so adding one is a decision for
the user, not a session.

## Labels without deleting

Removing a label is a `DELETE`, which is often refused on a shared repo. A `PATCH` that supplies
the **complete** desired set replaces what's there, which achieves the same removal:

```bash
gh api -X PATCH repos/O/R/issues/N \
  -f 'labels[]=enhancement' -f 'labels[]=area:corpus'
```

Every label not listed is dropped from that issue. Read the current set first — this is a
replacement, so omitting one of the user's own labels silently loses it.

Deleting the label *definition* from the repo is a genuine delete and usually needs the user.

## Relationships: blockers and sub-issues

Both take the **numeric issue id**, not the issue number, and both work **across repos** in the
same org.

```bash
BLOCKER_ID=$(gh api repos/O/OTHER_REPO/issues/27 --jq .id)
gh api -X POST repos/O/R/issues/63/dependencies/blocked_by -F issue_id=$BLOCKER_ID --jq .number
gh api repos/O/R/issues/63/dependencies/blocked_by --jq '.[]|"\(.repository.name)#\(.number)"'
```

Sub-issues, for parent/child:

```bash
CHILD_ID=$(gh api repos/O/R/issues/91 --jq .id)
gh api -X POST repos/O/R/issues/90/sub_issues -F sub_issue_id=$CHILD_ID
```

Any issue carries `issue_dependencies_summary` and `sub_issues_summary` in its JSON, which is the
cheap way to verify a batch landed.

## Evidence for "did this actually ship?"

Never close an issue on the strength of a commit message alone — the message says what someone
intended, the diff says what happened.

```bash
gh pr list --repo O/R --state merged --search "merged:>=2026-09-18" \
  --json number,title,mergedAt,body,closingIssuesReferences
git -C <repo> log --since="2026-09-18" --oneline --no-merges
gh pr view <N> --repo O/R --json files --jq '.files[].path'
```

`closingIssuesReferences` is the authoritative link between a merged PR and the issues GitHub
closed — more reliable than parsing `Closes #N` out of the body, and it catches the case where
GitHub auto-closed an issue the PR only meant to reference.

## Traps worth knowing before you debug them

**Field keys in `item-list` JSON keep their spaces.** A field named `Target date` reads back as
`'target date'`, not `targetDate`. Guessing the camelCase name returns `None` for every item and
makes a fully-populated board look empty — verify against one raw item before trusting any audit
that counts missing values:

```bash
gh project item-list <N> --owner <ORG> --limit 3 --format json | jq '.items[1]'
```

**Single-select options cannot be changed through the CLI.** `gh project field-create` makes a
field with options; nothing in the CLI edits them afterwards. Changing `P0/P1/P2` to
`Urgent/High/Medium/Low` needs either the GraphQL `updateProjectV2Field` mutation with
`singleSelectOptions`, or fifteen seconds in the project's field settings. Deleting and recreating
the field loses every value already stored in it — check whether any item uses it first
(if none do, recreation is free).

**Destructive calls are often refused** on shared org resources — `field-delete`, label deletion,
schema mutations. Do the additive work first, collect what was blocked, and hand the user one list
at the end rather than stopping at the first refusal.

**Batch with a thread pool.** A few hundred `gh api` calls run fine at ~6 concurrent; serially they
take minutes. Always collect and print failures rather than assuming success.
