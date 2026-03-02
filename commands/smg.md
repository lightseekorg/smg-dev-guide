---
description: "SMG development assistant — routes to the right skill based on what you're doing"
---

Analyze the user's current context and intent, then invoke the appropriate skill:

- Codebase structure, "where does X live", crate layout, how subsystems relate → invoke smg:map
- Commit, PR preparation, quality gates, "am I ready to submit" → invoke smg:contribute
- PR URL, "review this", diff analysis, code review → invoke smg:review-pr
- Implement, add, build, fix, create, modify any feature → invoke smg:implement

Default to `smg:implement` when intent involves writing code.

If the intent is ambiguous, ask which action they need: orient, implement, review, or ship.
