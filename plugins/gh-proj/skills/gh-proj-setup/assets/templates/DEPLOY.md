# <project> — deploy and operations

_How this ships, what it needs, and how to get it back. Last revised: <YYYY-MM-DD>._

## How a change reaches production

<The actual mechanism, stated plainly enough that nobody has to read the CI config to know whether
their push is about to deploy. Name the trigger — a merge to `main`, a tag, a manual run — the
workflow file, and where it lands.>

> ⚠️ <If a merge auto-deploys, say so here, in bold, at the top. This is the single fact people
> most need and most often don't have.>

## Environments

| Environment | URL | Deployed from | Notes |
|---|---|---|---|
| Production | <…> | <branch/tag> | <…> |
| Staging | <…> | <…> | <or "none — say so rather than leaving the row blank"> |

## Configuration

**Names only. No values in this file, ever.**

| Variable | Purpose | Required? |
|---|---|---|
| `<NAME>` | <what it controls> | yes / optional — <behaviour when unset> |

Secrets live in <the untracked `.env`, the secret manager, `~/.config/<slug>/…`>. Say where, not
what.

## Health and verification

```bash
curl -s <health url>
```

<What a healthy response looks like, field by field where it matters. What "degraded" means and
whether it still serves traffic. How long after a deploy to expect green.>

## Rollback

<The exact commands, in order, for the two cases: a bad deploy with no data change, and one with a
data change. If rollback for the second case is "you can't, roll forward", write that — an
unwritten rollback plan is discovered during the incident.>

## Migrations

<Where they live, how they're numbered, who applies them and how. Whether the code must tolerate
the schema not being there yet — and if so, that it's a hard rule, not a nicety.>

## Scheduled work

| Job | Schedule | Where it runs | Fails how |
|---|---|---|---|
| <…> | <…> | <…> | <silently? alerts where?> |

## Alerts

<Which channel, which webhook path (path, never the URL), and what actually fires. See
`~/.config/<slug>/discord-webhook`.>
