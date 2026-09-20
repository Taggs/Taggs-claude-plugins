# Architecture — decision log

_Why the system is shaped the way it is. Current state lives in
[`ARCHITECTURE-current.md`](ARCHITECTURE-current.md), which cites these entries as `[D-nn]`._

**Appending a new entry:** add it at the top, take the next `D-nn`, and date it the day the
decision shipped — not the day it was discussed. Record the decision, the reasoning, and, where
there was one, the failure that forced it.

**Never rewrite or delete an entry when a decision is reversed.** Add a new one that supersedes it
and cross-link both. A log edited to match the present tells you nothing about why anything
changed, which is the only thing it was for.

Entries are newest-first.

---

### D-001 · <YYYY-MM-DD> · <the decision, as a short sentence>

<The context: what forced the choice. The alternatives that were real, and why each was rejected —
a decision with no rejected alternative was never a decision. What this now means for anyone
working on the system, including what it forecloses.>

- **Supersedes:** <D-nn, or omit this line>
- **Superseded by:** <added later, if ever>
