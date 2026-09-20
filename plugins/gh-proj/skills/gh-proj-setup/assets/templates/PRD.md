# <project> — product requirements

_What is being built and what counts as done. Shape and reasoning live in
`ARCHITECTURE-current.md` and `ARCHITECTURE-decisionlog.md`; this file is about the product, not
the code. Last revised: <YYYY-MM-DD>._

## Problem

<Who has it, what it costs them today, and how they currently work around it. One or two
paragraphs. If this can't be written without guessing, stop and ask — the rest of the document
inherits every wrong assumption made here.>

## Users

<Each distinct user, what they're trying to do, and what "good" looks like from where they sit. A
paying customer and an internal admin are different users even in a small product.>

## Scope — this phase

**In**

- <capability, stated as an outcome for a user, not as a component>

**Out, deliberately**

- <thing a reader would reasonably expect to be here — and the one-line reason it isn't>

The "out" list is the load-bearing half. Without it, every feature request looks in scope.

## Success criteria

<How you will know it worked, in terms you could actually check. A number with a date beats an
adjective: "p95 analysis under 20 s on the top 100 domains by <date>", not "fast".>

## Constraints

<Budget, deadlines, platform rules, legal or data-protection limits, anything already decided
elsewhere that this has to live within. Link out rather than restating.>

## Open questions

| # | Question | Blocks | Owner |
|---|---|---|---|
| 1 | <…> | <what can't start until it's answered> | <who> |

Answered questions move out of this table — into the decision log if the answer was a decision, or
into the body above if it was just a fact.
