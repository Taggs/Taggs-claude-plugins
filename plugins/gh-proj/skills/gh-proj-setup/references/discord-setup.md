# Per-project Discord channel and webhook

## Why one channel per project

A single shared channel turns every alert into a reading comprehension task: which project failed?
Separate channels answer that by where the message landed. It also means a noisy project can be
muted without muting the others, and a project's history stays readable.

## Where the webhook lives

```
~/.config/<project-slug>/discord-webhook     # chmod 600, one line, no trailing newline needed
```

`<project-slug>` is the lowercase project name with no spaces — `acme-api`, `hoot-and-dodge`.
Match an existing directory if the project already has one rather than creating a second.

**The URL is a credential.** Anyone holding it can post to that channel as the webhook. Never print
it, never echo it, never paste it into a report, a commit, a test command shown to the user, or a
`.env` that gets committed. When a command needs it, read it from the file inside the command.

## Creating the channel — the user's steps

Claude can't create a Discord channel; hand these over verbatim:

1. In the Discord server, **+** next to the channel list → **Create Channel** → text channel, named
   for the project (e.g. `#acme-api-alerts`).
2. Hover the new channel → **Edit Channel** (gear) → **Integrations** → **Webhooks** →
   **New Webhook**.
3. Name it after what posts to it (e.g. "Acme API backend"), optionally set an avatar, then
   **Copy Webhook URL**.
4. Paste it back.

## Storing it

```bash
mkdir -p ~/.config/<slug>
printf '%s' "<pasted url>" > ~/.config/<slug>/discord-webhook
chmod 600 ~/.config/<slug>/discord-webhook
```

## Verifying

Discord rejects requests from default HTTP client user agents, so send a real one. This reads the
URL from the file rather than interpolating it into a visible command:

```bash
curl -sS -X POST -H 'Content-Type: application/json' \
  -A 'Mozilla/5.0 (X11; Linux x86_64)' \
  -d '{"content":"Webhook configured for <project> — setup test."}' \
  "$(cat ~/.config/<slug>/discord-webhook)" -w '%{http_code}\n'
```

`204` with no body is success. `401`/`404` means the webhook was deleted or the URL is wrong;
`429` is rate limiting — wait and retry once.

## Wiring it into the app

If the project sends its own alerts, it reads the URL from an environment variable — commonly
`DISCORD_WEBHOOK_URL` — set in the untracked `.env` from the file above. The code path should be
**inert when the variable is unset**, so a developer without the secret isn't blocked and doesn't
see errors.

Record in `docs/DEPLOY.md`, under env/secrets:

```markdown
| `DISCORD_WEBHOOK_URL` | Alerts channel for this project. Value in `~/.config/<slug>/discord-webhook` (chmod 600). Optional — alerting is inert when unset. |
```

The path and the variable name are documentation. The value never appears in the repo.

## What should post there

Worth a message: run start and completion for long jobs, deploy success and failure, health-check
state transitions, spend-guardrail trips, anything that wakes someone up. Not worth a message:
per-request logs, anything that fires more than a few times an hour. A channel that gets muted
because it's noisy is the same as no channel, and you find out the day it matters.
