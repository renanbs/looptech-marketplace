---
name: omp-setup
description: omp-only. Fixes "Error: No model selected" on looptech agents by writing task.agentModelOverrides for each of the 10 plugin agents, because omp does not resolve model:inherit the way Claude Code/Cursor/Codex do.
---

Run this once per omp installation (or again whenever you change providers/models).
It only makes sense on **omp** — stop and say so if the current host is not omp.

## Why this exists

Every agent in this plugin (`agents/*.md`) ships `model: inherit` in its
frontmatter. In Claude Code, Cursor, and Codex that means "use the same model
as the main conversation" — including when the field is *omitted* entirely.
omp does not implement that convention: it has no concept of inheriting a
model for a **custom** subagent, so spawning one of these agents on omp fails
with `Error: No model selected` until a model is resolved some other way.

The fix is not to change the shared `agents/*.md` files — `model: inherit` is
correct for the other three hosts, and rewriting it risks changing their
behavior. Instead, omp has its own per-agent override mechanism
(`task.agentModelOverrides`, a `name -> model` map) that `omp config set` can
write directly, independent of the frontmatter. This command performs that
one-time write for the 10 looptech agents.

## Steps

1. Confirm the host is omp (the `omp` CLI/TUI). If not, stop.
2. Resolve one model ID per **class** (`reasoning`, `code`, `critique`,
   `security` — see `skills/workflow-dev/references/agent-roles.md`):
   - If the Project Profile (`AGENTS.md`) has an `agents.omp.<classe>.model`
     block, use those IDs.
   - Otherwise list the available models (`omp models` or `/model`) and ask
     the user to pick one per class — never invent a model ID.
3. For each agent below, run `omp config set task.agentModelOverrides.<agent> <model-id>`
   using the model resolved for its class:

   | Agent | Class |
   |---|---|
   | `plan` | `reasoning` |
   | `review` | `critique` |
   | `expert-security` | `security` |
   | `expert-backend-go` | `code` |
   | `expert-backend-python` | `code` |
   | `expert-database` | `code` |
   | `expert-frontend-react` | `code` |
   | `expert-frontend-vue` | `code` |
   | `expert-frontend-pwa` | `code` |
   | `expert-frontend-web` | `code` |

4. Verify with `omp config get task.agentModelOverrides`.
5. Tell the user to run `/reload-plugins` or restart the omp session before
   the overrides take effect on already-loaded agent definitions.

If `security` has no dedicated model, reuse the `critique` model — same rule
as every other host in `agent-roles.md`.
