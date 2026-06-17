# Fakduai Hermes Agent Company

Configurable AI company workspace for Hermes Agent + Pinto.

This repo provides:

- role/persona templates for PM, Solution Designer, Frontend, Backend, QA, and Tech Lead
- a local office dashboard for Hermes workflow activity
- Taste Skill UI/design skills for Designer and Frontend agents
- optional legacy OpenClaw scripts for teams that still use OpenClaw

Use this with the Hermes Pinto runtime repo:

```text
https://github.com/fakduai-logistics-and-digital-platform/hermes-fakduai-pinto
```


## Hermes + Pinto Integration

Recommended setup for teammates:

1. Clone and run the Hermes Pinto runtime repo first.

   ```bash
   git clone git@github.com:fakduai-logistics-and-digital-platform/hermes-fakduai-pinto.git
   cd hermes-fakduai-pinto
   cp .env.example .env
   # Fill Pinto + provider config in .env.
   # Do not commit .env.
   scripts/podman-start.sh
   scripts/podman-tunnel-start.sh
   ```

2. In Pinto Developer Console, set webhook URL to the Cloudflare Quick Tunnel URL printed by Hermes:

   ```text
   https://<trycloudflare-host>/plugins/pinto/webhook
   ```

3. Verify Hermes locally:

   ```bash
   curl http://127.0.0.1:8642/health
   curl http://127.0.0.1:8642/plugins/pinto/webhook
   ```

   Expected health:

   ```json
   {"status":"ok","platform":"hermes-agent"}
   ```

4. Clone this company repo beside the Hermes repo:

   ```bash
   cd /path/to/workspace
   git clone git@github.com:fakduai-logistics-and-digital-platform/fakduai-hermes-agent-company.git
   cd fakduai-hermes-agent-company
   ```

5. Start the company dashboard without syncing OpenClaw config:

   ```bash
   SKIP_OPENCLAW_SYNC=1 ./scripts/bootstrap.sh
   ./scripts/dashboard-detached.sh --port 8090 --interval 3
   ```

6. In `hermes-fakduai-pinto/hermes-config/config.yaml`, configure Pinto company workflow to use the company roles. Keep real bot IDs, secrets, and local persona text out of git.

   Example shape:

   ```yaml
   platforms:
     pinto:
       extra:
         companyName: proton
         persona: pm
         companyWorkflow:
           - pm
           - designer
           - frontend
           - backend
           - qa
           - techlead
   pintoAgents:
     pm:
       name: PM
       channelPrompt: "...local PM prompt..."
     designer:
       name: Solution Designer
       channelPrompt: "...local designer prompt..."
     frontend:
       name: Frontend Engineer
       channelPrompt: "...local frontend prompt..."
     backend:
       name: Backend Engineer
       channelPrompt: "...local backend prompt..."
     qa:
       name: QA Engineer
       channelPrompt: "...local QA prompt..."
     techlead:
       name: Tech Lead
       channelPrompt: "...local tech lead prompt..."
   ```

7. Set project output path in the Hermes runtime if not already set:

   ```env
   COMPANY_PROJECTS_DIR=/company-projects
   ```

   Generated projects should appear on the host under:

   ```text
   /path/to/hermes-fakduai-pinto/hermes-config/company-projects
   ```

8. Send a real Pinto chat message to the connected bot. Hermes should:

   ```text
   Pinto chat -> Hermes -> PM -> Designer -> Frontend -> Backend -> QA -> Tech Lead -> Pinto final reply
   ```

   Milestone progress messages go back to Pinto chat. Detailed agent activity appears in the local dashboard:

   ```text
   http://127.0.0.1:8090/
   ```

Notes:

- Cloudflare Quick Tunnel URLs are temporary. If `cloudflared` restarts, update Pinto Developer Console with the new URL.
- Do not commit `.env`, `hermes-config/`, real bot IDs, webhook secrets, API keys, or local persona prompts.
- This repo does not require OpenClaw for Hermes usage. Use `SKIP_OPENCLAW_SYNC=1` unless you explicitly want legacy OpenClaw config sync.

## Quick Start: Local Dashboard Only

```bash
# Review or edit office display config
$EDITOR config/office.json

# Create shared status files and dashboard data without touching OpenClaw
SKIP_OPENCLAW_SYNC=1 ./scripts/bootstrap.sh

# Run the dashboard
./scripts/dashboard.sh
```

## Legacy OpenClaw Quick Start

```bash
# Review or edit the active six-agent company config
$EDITOR config/office.json

# Create workspaces, shared status files, dashboard data,
# and merge company agents into ~/.openclaw/openclaw.json
./scripts/bootstrap.sh

# Restart/recreate the OpenClaw gateway or container so it loads the new agents

# Start a routed multi-agent workflow
./scripts/start-workflow.sh "Build the customer login flow"
```

## Structure

```
├── config/office.json          # office config (agents, rooms, homes)
├── templates/                  # workspace templates
├── scripts/                    # bootstrap, dashboard, task scripts
├── ui/dashboard/               # office-style dashboard
├── shared/                     # shared data between agents
└── docs/                       # คู่มือ
```

## Routed Agent Workflow

The primary delivery sequence is:

```text
PM -> Solution Designer -> Frontend + Backend -> QA -> Tech Lead
```

Agent sends are throttled by default with `COMPANY_MAX_PARALLEL=2` so a routed workflow cannot launch every role at once and peg local CPU. Use `COMPANY_MAX_PARALLEL=1` on smaller hosts.

Start with:

```bash
./scripts/start-workflow.sh "Build the customer login flow"
```

Agents can pass work to each other with:

```bash
./scripts/route-handoff.sh pm designer <workflow-id> "<handoff body>"
```

`bootstrap.sh` runs `scripts/sync-openclaw-config.sh` automatically. Re-run that sync script after changing `config/office.json`; it updates OpenClaw config by agent id, backs up the previous config, enables agent-to-agent routing, syncs auth profiles from `main`, and validates the result.

See [Automatic Routing](docs/agent-workflows/automatic-routing.md) for the full workflow.

All routed work must follow the [Agent Execution Control Policy](docs/agent-workflows/execution-control-policy.md): every handoff needs explicit output, definition of done, required evidence, stale timeout, and next route. Progress percentages are evidence-based, and bare `working` status is not accepted as progress.

Routing a handoff does not count as active work. `route-handoff.sh` records delivery states (`delivering`, then `delivered_waiting_for_receiver`) and the receiving agent must write its own evidence-based `working` status. Empty delivery logs or missing receiver acknowledgement should be treated as delivery/control-plane failures, not progress.

PM can run the [workflow monitor utility](docs/agent-workflows/ops-monitor.md) for compact stale/delivery checks instead of polling large logs:

```bash
./scripts/monitor-workflows.sh
./scripts/monitor-workflows.sh --apply
```

QA must use Playwright for browser-facing UI/E2E validation. See [Playwright QA Policy](docs/agent-workflows/playwright-qa-policy.md).

`scripts/dashboard.sh` keeps dashboard data refreshed every 3 seconds by default. The Work modal includes an Activity tab that shows workflow starts, handoffs, and queued agent messages.

## Vendored Agent Skills

This repo vendors Karpathy Guidelines for all agents:

```text
skills/karpathy-guidelines/SKILL.md
```

Source: <https://github.com/multica-ai/andrej-karpathy-skills/blob/main/skills/karpathy-guidelines/SKILL.md>

All workspace agents are instructed to use it for coding, reviewing, refactoring, debugging, and implementation planning. It biases agents toward explicit assumptions, simple solutions, surgical edits, and verifiable success criteria.

This repo also vendors Matt Pocock skills for backend and architecture work:

```text
skills/mattpocock/
```

Source: <https://github.com/mattpocock/skills/tree/main/skills>

Cloudflare skills are vendored for dev preview/hosting work:

```text
skills/cloudflare/
```

Source: <https://github.com/cloudflare/skills>

Best role fit:

```text
cloudflare              -> Frontend, Backend for Cloudflare platform decisions
wrangler                -> Frontend, Backend for local dev/deploy commands
workers-best-practices  -> Frontend, Backend for Workers-compatible code/review
web-perf                -> Frontend for performance checks
```

When the user asks to run, preview, host, deploy, open, or show a product, PM should route that request directly to Frontend and/or Backend. The dev team should create a local runnable preview first and, when credentials/auth are available, use Cloudflare/Wrangler to produce a hosted preview URL. If Cloudflare auth is unavailable, dev can use `localhost.run` as a no-login SSH fallback:

```bash
ssh -R 80:localhost:3000 nokey@localhost.run
```

Do not ask the user to run the product themselves unless the environment or credentials are missing.

9arm skills are also vendored for debugging discipline, review, post-mortems, management updates, and long-task control:

```text
skills/9arm/
```

Source: <https://github.com/thananon/9arm-skills>

Best role fit:

```text
debug-mantra      -> Backend, QA, Tech Lead for hard debugging
post-mortem       -> Backend, QA, Tech Lead for fixed-and-validated bug RCA
scrutinize        -> QA, Tech Lead, Backend for plan/PR/diff/design sanity checks
management-talk   -> PM, Tech Lead for leadership/status/Slack/email updates
qwenchance        -> all roles when long work starts looping or context gets tight
qwen-agent        -> optional local operator skill for cheap mechanical subagent work when claude-9arm exists
```

PM/communication skill:

```text
skills/stop-slop/SKILL.md
```

Source: <https://github.com/hardikpandya/stop-slop/blob/main/SKILL.md>

PM and Tech Lead use it for user-facing updates, PM summaries, final reports, PRDs, issue descriptions, release notes, and stakeholder messages. Goal: cut AI phrasing, filler, passive voice, vague summaries, formulaic contrasts, and empty praise.

Recommended QA/tester set:

```text
skills/mattpocock/deprecated/qa/SKILL.md
skills/mattpocock/in-progress/review/SKILL.md
skills/mattpocock/engineering/diagnosing-bugs/SKILL.md
skills/mattpocock/engineering/tdd/SKILL.md
skills/mattpocock/engineering/triage/SKILL.md
skills/mattpocock/engineering/setup-matt-pocock-skills/SKILL.md
```

QA usage guide:

```text
Bug found -> use qa to clarify and write a durable user-focused issue
Issue queue -> use triage to move issues through needs-triage / needs-info / ready-for-agent / ready-for-human / wontfix
Hard bug -> use diagnosing-bugs with reproduce -> minimize -> hypothesize -> instrument -> fix -> regression-test
Regression coverage -> use tdd to add the failing test before the fix
Branch/PR check -> use review against main/origin/main for Standards and Spec review
First setup -> use setup-matt-pocock-skills
```

Recommended backend set:

```text
skills/mattpocock/engineering/grill-with-docs/SKILL.md
skills/mattpocock/engineering/tdd/SKILL.md
skills/mattpocock/engineering/diagnosing-bugs/SKILL.md
skills/mattpocock/engineering/codebase-design/SKILL.md
skills/mattpocock/engineering/domain-modeling/SKILL.md
skills/mattpocock/engineering/setup-matt-pocock-skills/SKILL.md
skills/mattpocock/engineering/to-prd/SKILL.md
skills/mattpocock/engineering/to-issues/SKILL.md
skills/mattpocock/engineering/improve-codebase-architecture/SKILL.md
skills/mattpocock/misc/git-guardrails-claude-code/SKILL.md
```

Backend usage guide:

```text
New backend feature/API:
  use grill-with-docs, then domain-modeling when business language matters

Implementation:
  use tdd; one vertical slice at a time, red-green-refactor

Bug:
  use diagnosing-bugs; reproduce -> minimize -> hypothesize -> instrument -> fix -> regression-test

Architecture cleanup:
  use codebase-design or improve-codebase-architecture

PRD/issues:
  use to-prd and to-issues

Git safety setup:
  use git-guardrails-claude-code when explicitly asked
```

## Vendored UI Skills

This repo vendors Taste Skill for UI/design agents:

```text
skills/taste-skill/
```

Source: <https://github.com/Leonxlnx/taste-skill>

Primary skills:

```text
skills/taste-skill/skills/taste-skill/SKILL.md              # design-taste-frontend / taste-design
skills/taste-skill/skills/gpt-tasteskill/SKILL.md           # stricter premium UI/motion
skills/taste-skill/skills/redesign-skill/SKILL.md           # improve existing UI
skills/taste-skill/skills/minimalist-skill/SKILL.md         # minimalist UI direction
skills/taste-skill/skills/soft-skill/SKILL.md               # soft premium UI direction
skills/taste-skill/skills/brutalist-skill/SKILL.md          # industrial/brutalist direction
skills/taste-skill/skills/imagegen-frontend-web/SKILL.md    # web reference images
skills/taste-skill/skills/brandkit/SKILL.md                 # brand-kit boards
```

Frontend and Solution Designer agents should use these when planning or implementing visual design, layout, typography, spacing, motion, responsive states, redesigns, or anti-slop UI polish.

## Docs

- [Getting Started](docs/getting-started.md)
- [Adding Roles](docs/adding-roles.md)
- [Workflow Monitor Utility](docs/agent-workflows/ops-monitor.md)
