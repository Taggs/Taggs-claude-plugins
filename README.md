# Taggs-claude-plugins

A [Claude Code](https://claude.com/claude-code) plugin marketplace. One plugin so far: `gh-proj`,
which gives a GitHub project the scaffolding a work session needs, then audits the session against
it on the way out.

## Install

Add the marketplace, then install the plugin:

```
/plugin marketplace add Taggs/Taggs-claude-plugins
/plugin install gh-proj@taggs-claude-plugins
```

To pull later changes:

```
/plugin marketplace update taggs-claude-plugins
```

Requires the [`gh` CLI](https://cli.github.com), authenticated with the `repo` and `project`
scopes (`gh auth login -s project`). The Discord half of `gh-proj-setup` is optional and inert if
you skip it.

## Plugin: `gh-proj`

Two counterpart skills. `gh-proj-setup` puts the structure in place; `gh-proj-session-close` checks
the record still matches reality and writes the handoff. They share a vocabulary, so running either
one alone still works — but the pair is the point.

### Skill: `gh-proj-setup`

Makes a project or repo **work-session compliant** — able to start and end a session without
anyone reconstructing state from memory. Idempotent: run it against a half-configured project and
it fills only the gaps.

It puts in place:

- **One project board** where every issue carries a Status, Type, Priority, one core label, a Start
  date and a Target date, with real blockers recorded as issue relationships rather than prose.
- **Milestones with due dates** — the only phase axis, so phases don't get re-invented as board
  fields.
- **Five core docs** in `docs/` — `PRD.md`, `ARCHITECTURE-current.md`,
  `ARCHITECTURE-decisionlog.md`, `DEPLOY.md`, `next-steps.md` — plus `docs/research/` for
  everything else. Templates for each ship in `assets/templates/`.
- **A Discord channel and webhook** for the project, stored at
  `~/.config/<project-slug>/discord-webhook`, so its notifications stay separable from every other
  project's.

Creating is safe, overwriting isn't: it creates missing boards, fields, milestones and doc stubs
directly, but never overwrites an existing doc, renumbers a decision log, or changes a status or
date someone already set — those get reported and proposed.

Invoke with `/gh-proj-setup`, or let Claude offer it when a repo has no board, no milestones or no
`docs/` folder.

### Skill: `gh-proj-session-close`

Audits, at the end of a session, what `gh-proj-setup` put in place. It:

- sweeps the board and milestones for wrong statuses, missing or stale dates, and taxonomy drift —
  issues with no Type or Priority, missing labels, unrecorded blockers, epics masquerading as
  fields;
- opens issues for work that happened but never became one, and closes the ones that demonstrably
  shipped;
- checks the core docs are collated and current;
- re-reads the session for decisions that were made but never logged;
- rewrites `docs/next-steps.md` as a plan for the next session.

It applies board metadata fixes directly and *proposes* anything prose — doc edits, decision-log
entries, moving a target date that has already slipped. It never pushes to a protected branch,
merges, deploys, runs a migration, prints a secret, or edits an existing decision-log entry.

Invoke with `/gh-proj-session-close`, or let Claude offer it when a long session reaches a natural
stopping point — this is exactly the work that gets skipped when someone is tired.

### Shared references

Both skills read the same three files, kept byte-identical in each skill's `references/`. Edit one,
edit the other.

| File | What it holds |
|---|---|
| `work-item-taxonomy.md` | The field rules both skills enforce — what Type, Priority, the core labels and the date fields mean, and what "compliant" is in detail. |
| `github-board-recipes.md` | The `gh` and GraphQL calls that actually work against org-level issue fields, including the ones the CLI reports as empty. |
| `known-projects.md` | Per-workspace overrides. Ships as a **template**; your real conventions belong in `~/.config/claude/known-projects.md`, which is machine-local and never committed. Both skills prefer that file when it exists. |

`gh-proj-setup` additionally has `references/discord-setup.md` (channel + webhook creation, and how
to handle the URL as a credential).

## Repo layout

```
.claude-plugin/marketplace.json   marketplace manifest
plugins/<plugin>/
  .claude-plugin/plugin.json      plugin manifest
  skills/<skill>/
    SKILL.md                      the skill itself
    references/                   loaded on demand
    assets/                       templates copied into target repos
README.md
```

## Adding a plugin

1. Create `plugins/<name>/` with a `.claude-plugin/plugin.json` and a `skills/` directory.
2. Add an entry to the `plugins` array in `.claude-plugin/marketplace.json`.
3. Bump `metadata.version` in the marketplace manifest and `version` in the plugin manifest.
4. Push, then `/plugin marketplace update taggs-claude-plugins`.

## License

MIT — see [LICENSE](LICENSE).
