cf-agent-a2a-skill-script

Demo of **two Agent Buildpack apps** on Cloud Foundry communicating peer-to-peer via the **native A2A capabilities of Agent Buildpack 0.32**. Each agent ships a **SkillRunner MCP sidecar** — an extension to the Agent Buildpack — enabling local Python skill script execution.

## Architecture

```
CF app: agent-a2a-alpha          CF app: agent-a2a-beta
┌──────────────────────┐          ┌──────────────────────┐
│  agent-a2a-alpha     │          │  agent-a2a-beta      │
│  (orchestrator)      │          │  (worker / scripts)  │
│                      │          │                      │
│  Native A2A tools:   │◄────────►│  Native A2A tools:   │
│  list_a2a_peers      │  HTTP    │  list_a2a_peers      │
│  call_a2a_peer       │          │  call_a2a_peer       │
│                      │          │                      │
│  SkillRunner sidecar │          │  SkillRunner sidecar │
│                      │          │  (tanzu-network-     │
│                      │          │   release, etc.)     │
└──────────────────────┘          └──────────────────────┘
```

- **A2A communication** is handled entirely by Agent Buildpack 0.32's built-in tools (`list_a2a_peers`, `call_a2a_peer`).
- **Script execution** (e.g. querying Tanzu Network) is handled by the SkillRunner MCP sidecar — an extension to the Agent Buildpack that exposes skill Python scripts as MCP tools, shared from [`cf-agent-skill-script`](https://github.com/yannicklevederpvtl/cf-agent-skill-script).

## Apps

| CF App | Role | Skills |
|---|---|---|
| `agent-a2a-alpha` | Orchestrator — delegates tasks to beta |  |
| `agent-a2a-beta` | Worker — runs skill scripts, replies to alpha | `python-script-smoke`, `tanzu-network-release` |

Each app ships an `a2a-peers.yaml` mapping peer aliases to CF app routes. Update URLs after first push.

## Contents

| Path | Purpose |
|---|---|
| `manifest.yml` | Both apps: Python + agent buildpacks, **`Qwen3.6`** + **`skill-runner-mcp-local`**, `skill-runner` sidecar. |
| `apps/agent-a2a-alpha/` | Push tree for the orchestrator agent. |
| `apps/agent-a2a-beta/` | Push tree for the worker agent. |
| `apps/*/skill_runner/` | SkillRunner MCP sidecar (synced from [cf-agent-skill-script](https://github.com/yannicklevederpvtl/cf-agent-skill-script)). |
| `apps/*/a2a-peers.yaml` | Peer alias → CF app route mapping. |
| `apps/*/.agents/skills/` | Agent skills (SKILL.md prompt instructions + optional Python scripts). |
| `scripts/setup_mcp_binding.sh` | One-time per space: create CUPS **`skill-runner-mcp-local`** tagged `mcp-server`. |
| `scripts/sync_shared.sh` | Sync `skill_runner/` sidecar from `cf-agent-skill-script` into each app push tree. |

## Prerequisites

- **cf CLI v8** (`sidecars` requires CAPI v3)
- Multi-buildpack support: `python_buildpack` then `agent_buildpack`
- [`cf-agent-skill-script`](https://github.com/yannicklevederpvtl/cf-agent-skill-script) cloned as a sibling directory (provides the `skill_runner/` sidecar)
- Service instances in the target space:
  - **`Qwen3.6`** — LLM service
  - **`skill-runner-mcp-local`** — user-provided MCP service (created by `scripts/setup_mcp_binding.sh`, once per space)

## Deploy

**1. First time in a space** — create the MCP CUPS:

```bash
bash scripts/setup_mcp_binding.sh
```

**2. Sync the SkillRunner sidecar** from `cf-agent-skill-script` into each app push tree:

```bash
bash scripts/sync_shared.sh
```

**3. Configure before push** — edit these files for your foundation (do not commit real foundation-specific values if you fork this repo):

**`manifest.yml`** — set `services:` to match your space:

```bash
cf services   # list instance names in the target space
```

Update both apps if needed (defaults: `Qwen3.6`, `skill-runner-mcp-local`):

```yaml
services:
  - <your-llm-service>
  - skill-runner-mcp-local
```

**`apps/agent-a2a-alpha/a2a-peers.yaml`** and **`apps/agent-a2a-beta/a2a-peers.yaml`** — replace `<CF_APPS_DOMAIN>` with your apps domain (e.g. `apps.example.com`):

```yaml
# alpha points to beta:
url: https://agent-a2a-beta.apps.example.com

# beta points to alpha:
url: https://agent-a2a-alpha.apps.example.com
```

You can derive the domain from `cf target` or your foundation docs — you do not need to push first if app names match `manifest.yml`.

**4. Push both agents:**

```bash
cf push -f manifest.yml
```

To push a single agent:

```bash
cf push -f manifest.yml agent-a2a-alpha
cf push -f manifest.yml agent-a2a-beta
```

**After push (optional)** — if A2A peers fail to connect, confirm routes and fix `a2a-peers.yaml`, then push again:

```bash
cf app agent-a2a-alpha | grep routes
cf app agent-a2a-beta | grep routes
cf push -f manifest.yml
```

## SkillRunner MCP tools (both agents)

Sidecar runs at `http://127.0.0.1:8765/mcp/`, registered via `skill-runner-mcp-local` CUPS.

| Tool | Role |
|---|---|
| `ping` | Liveness check. |
| `list_skill_scripts` | JSON list of all `scripts/**/*.py` across skills. |
| `run_skill_script` | Run `python3 <script> [args…]` with `cwd` = skill root; `script_relpath` must stay under that skill's `scripts/`. |
| `run_smoke_skill` | Shortcut for `python-script-smoke/scripts/smoke_check.py`. |

## Sample prompts

**On agent-a2a-alpha** — delegate to beta:

> Ask agent beta to use its `tanzu-network-release` skill to list the latest versions of the `cf` product and return the raw output.

**On agent-a2a-beta** — run locally:

> Use the `tanzu-network-release` skill to give me the latest version of the `genai` product.

## Verify

```bash
# Sidecar health (SSH into the app)
cf ssh agent-a2a-beta
curl -sS http://127.0.0.1:8765/health

# In the debug UI for agent-a2a-alpha, confirm:
#   - MCP server "skill-runner-mcp-local" connected
#   - Tools: ping, list_skill_scripts, run_skill_script, run_smoke_skill
#   - A2A peers: beta listed under "A2A PEERS"

# Smoke-test A2A connectivity from alpha's chat:
# "Run the a2a-smoke skill"
```

## Security note

Policy A allows execution of any Python file shipped under `skills/*/scripts/`. The MCP port is bound to `127.0.0.1` — never expose it on a public route.
