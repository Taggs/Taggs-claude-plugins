# Known project conventions

Real projects drift from any skill's defaults. This file is where that drift gets recorded, so a
session doesn't rediscover it. Read the section for the workspace you're in; everything there
overrides the defaults in `SKILL.md`, because it reflects what the project actually does.

**This copy ships empty on purpose.** Project conventions name internal repos, board IDs and deploy
behaviour, so they don't belong in a public plugin.

## Where your real conventions live

Before reading further, check for a private override and prefer it if it exists:

```bash
cat ~/.config/claude/known-projects.md 2>/dev/null
```

That file is yours, machine-local and never committed. Keep the per-project sections there. If it
doesn't exist, use this file's defaults and offer to start one at the end of the session.

## Adding a project

Copy this skeleton into your private file. Record only what would otherwise have to be
rediscovered — not what the repo's `CLAUDE.md`, the board itself, or `git log` already says.

```markdown
## <Project> — `~/path/to/workspace`

<Is it a git repo, or a workspace root with sibling repos? Which repos, which is which?>
<Anything that auto-deploys on merge, and the standing rule about it.>
<Repos in the org that are explicitly out of scope.>

### Board
<Which project board is the system of record, and its URL. Any retired boards that must not be used.>
<The board's fields, and any field the taxonomy expects that this board deliberately lacks.>
<Whether Priority/dates live on the issue (org issue fields) or on the board — and the IDs if so.>

### Taxonomy
<Deviations from `work-item-taxonomy.md`, and which are deliberate.>
<Milestones in use, and which axis wins when two claim the same issue.>
<Label families that are the user's own and must be kept.>
<Shorthand used in the next-steps memos, e.g. `E#n` = engine issue.>

### Docs
<Where the core docs actually are, what's named differently, what doesn't exist.>
<Any generated doc that must never be hand-edited.>

### Operational
<Discord webhook path (never print the value). Test-account scripts. Cost of any run against prod.>
<Reporting-style preferences.>
```

## Gotchas worth recording when you hit them

These are environment quirks, not project data — they recur across projects and are cheap to note.

- **Org-level issue fields shadow board fields.** When an org defines `Priority`, `Start date` or
  `Target date` as issue fields, boards mirror them read-only: `gh project item-list` reports them
  as empty whether or not they're set. Read with
  `gh api repos/<owner>/<repo>/issues/<n> --jq .issue_field_values`, write with the
  `setIssueFieldValue` mutation. Record the field and option IDs once — they're stable per org.
  Don't recreate the same fields on the board; you'll get two sources of truth for one date.
- **`gh issue view <n>` (bare), `gh issue edit` and `gh pr edit` can fail** with a Projects-classic
  GraphQL error on orgs using the new issue fields. Use `--json` reads and `gh api` writes —
  `github-board-recipes.md` has working forms.
- **A board's "Auto-close issue" workflow makes a status write a close.** If it's enabled and
  intended, don't disable it — but after any bulk board pass, re-check `closed_at >= <session
  start>` and reopen anything closed as a side-effect.
- **Some environments refuse destructive calls** (`gh project field-delete`, label deletion, raw
  schema mutations) while allowing all additive work. Collect what was blocked and hand the user
  one list rather than retrying.
