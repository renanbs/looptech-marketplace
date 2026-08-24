---
name: workflow-dev
description: Start the looptech development workflow — dual brainstorm, /goals, spec+plan, async impl with /plan per task, review, security, unit+e2e/Playwright, PR.
---

Load and follow the skill `workflow-dev` in this plugin (`skills/workflow-dev/SKILL.md`).

Orchestrate the full cycle for the user's current development task. Do not implement code in the parent agent except trivial edits of 100 characters or fewer. Do not start impl before `/goal` exists. Dispatch independent tasks in one `tasks[]` wave. Each impl starts with `## /plan`.
