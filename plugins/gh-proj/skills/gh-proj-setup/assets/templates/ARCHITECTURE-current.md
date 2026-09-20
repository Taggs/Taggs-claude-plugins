# <project> — architecture, as it is now

_How the system is shaped today. **Why** it's shaped this way is in
[`ARCHITECTURE-decisionlog.md`](ARCHITECTURE-decisionlog.md), cited inline as `[D-nn]` — don't
restate the reasoning here, or the two will drift. Last revised: <YYYY-MM-DD>._

## Shape

<A paragraph, then a diagram if it earns its place. What the pieces are and how a request or a job
moves through them.>

## Components

| Component | Responsibility | Lives in |
|---|---|---|
| <name> | <one line — what it owns, and what it deliberately doesn't> | `<path>` |

## Data

<Stores, the tables or collections that matter, what owns each one, and where migrations live.
Enough that someone can find the schema, not a copy of it.>

## External dependencies

| Service | Used for | What happens when it's down |
|---|---|---|
| <name> | <…> | <the degradation path — this column is the one people need at 2am> |

## Runtime and configuration

<How it runs locally and in production, and the environment variables it reads — **names only,
never values**. Point at `DEPLOY.md` for how it ships.>

## Known rough edges

<Things that are true but shouldn't be permanent, each with its issue number. An honest list here
stops the next person rediscovering them one at a time.>
