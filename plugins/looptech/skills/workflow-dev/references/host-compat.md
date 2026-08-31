# Host compatibility — Claude Code, Codex, Cursor, omp

The same plugin ships to four hosts. Skills stay host-agnostic; only the
**invocation surface** and the **native tool names** change. Detect the host
once (from available tools / env) and keep that mapping for the rest of the
session.

## Detect the host

| Signal | Host |
|---|---|
| `Skill` tool, `CLAUDE_PLUGIN_ROOT`, `/plugin` slash commands | **Claude Code** |
| `spawn_agent` / `CODEX_HOME`, skills invoked with `$name` | **Codex** |
| Cursor `Task` tool, `.cursor/` project config, `/skill-name` | **Cursor** |
| `omp` CLI/TUI binary, `/skill:<name>` slash syntax, `~/.omp/` | **omp** |
| Ambiguous | Prefer the host whose tools you actually have. Never invent a missing tool. |

## Project Profile — where it lives

The Profile is the same YAML/Markdown block on every host. **Read both files
when they exist; write both when you create or update it.**

| File | Who reads it |
|---|---|
| `CLAUDE.md` | Claude Code (and any host that also opens it) |
| `AGENTS.md` | Codex, Cursor, Gemini, Copilot, omp |

- If only one file has the Profile, that file is authoritative. Copy it into
  the other file (create `AGENTS.md` if missing) so the next host does not
  start blind.
- If both have a Profile and they **diverge**, stop and ask which one wins
  before writing anything. Do not silently pick.
- Sub-project copies follow the same rule (`<sub>/CLAUDE.md` and
  `<sub>/AGENTS.md`).
- "Facts come from the Project Profile" always means **those files**, never
  a host-only settings panel.

## Load a skill

Do **not** hardcode a host-only invocation. Read the skill's `SKILL.md` and
follow it.

| Host | How the user / agent loads a skill |
|---|---|
| Claude Code | `Skill("looptech:<name>")` or `/looptech:<name>` |
| Codex | `$<name>` (e.g. `$workflow-dev`, `$init`) or auto-trigger from the description |
| Cursor | `/<name>` (plugin command or skill) |
| omp | `/skill:<name>` or auto-trigger from the description at session start |

Sibling plugin skills (`memory-graph:memory-vault`,
`memory-graph:memory-vault-setup`) load the same way after that plugin is
installed.

## Spawn a subagent

The Delegation Mandate does not change. Only the spawn API does. Resolve the
**role** first (`agent-roles.md`); then pass the resolved catalog ID through
the host's native field. Never put a vendor model name in the skill text.

| Host | Spawn | How the role ID is passed |
|---|---|---|
| Claude Code | native `Task` / subagent tool | `model` (and native effort if the host has one) from `agents.claude.<role>` or the catalog |
| Codex | native `spawn_agent` (or current equivalent) | per-child model if the host still allows it; else inherit the parent and **still spawn** |
| Cursor | native `Task` / named agent | spawn the plugin agent **by name** (`expert-backend-go`, `expert-security`, `plan`, `review`, …). `Task(model:)` is often rejected except for `fast`; named agents in the plugin `agents/` directory are the reliable path |
| omp | natural-language delegation ("Use the `plan` subagent to…") | the agent's own frontmatter `model:` field, or a role alias (`"@review"`) mapped in `modelRoles` — **see the warning below**, plain `model: inherit` does not work |

- Never skip a spawn because the host renamed the tool.
- Never tell a subagent to "go load the skill and read the plan files".
  Paste the handoff (`subagent-handoff.md`) into the child prompt.
- If the host cannot spawn at all, say so and stop — do not silently do the
  child's work in the parent (except the ≤ 100 character exception).
- If the host rejects the resolved ID, report, fall back to the session
  default, keep the role, continue.

## ⚠ omp does not resolve `model: inherit`

Every agent in `agents/*.md` ships `model: inherit`. In Claude Code, Cursor,
and Codex that means "use the same model as the main conversation" (an
*omitted* `model` field means the same thing on all three). **omp does not
implement that convention for custom subagents** — spawning a looptech agent
on a fresh omp install fails with `Error: No model selected` until a model is
resolved another way. Confirmed empirically: the `plan` agent errored until a
per-agent override was set.

Do not "fix" this by rewriting `model: inherit` in the shared agent files —
that value is correct for the other three hosts. Instead, on omp, run the
`looptech:omp-setup` command once (see `commands/omp-setup.md`): it writes
omp's own per-agent override map (`task.agentModelOverrides`) via
`omp config set`, independent of the frontmatter. Tell the user about this
requirement the first time you detect omp as the host, and point them at
`looptech:omp-setup` — do not silently spawn and let it fail.

## Restart wording

Say "restart this agent session" (Claude Code / Codex / Cursor), not
"restart Claude Code", unless you are giving a host-specific command.
