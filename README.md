# Agent Team — Claude Code Multi-Agent Pipeline

A fully nested, Claude Code-native multi-agent system that builds complete full-stack applications from a single project description. Give it a project idea and it orchestrates 20 specialized agents across 5 teams to produce production-ready code, Docker infrastructure, and a CI/CD pipeline.

---

## Architecture Overview

```
You (in Claude Code)
    │
    │  "Build me a project management app"
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                     ORCHESTRATOR (CLAUDE.md)                    │
│          Manages pipeline order and gate checks                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
         ┌─────────────────▼──────────────────┐
         │         PHASE 1 (blocking)          │
         │    system-design coordinator        │
         │                                     │
         │   spawns ──► architect (blocking)   │
         │   spawns ──► api-designer    ┐      │
         │   spawns ──► data-modeler    │ parallel
         │              (writes handoffs)      │
         └─────────────────┬──────────────────┘
                           │ gates pass
         ┌─────────────────▼──────────────────────────────────────┐
         │                  PHASE 2 (parallel)                     │
         │                                                         │
         │  frontend coordinator    │   backend coordinator        │
         │                          │                              │
         │  spawns ──► figma-converter (blocking)                  │
         │  spawns ──► frontend-lead ┐  │  spawns ──► backend-lead │
         │  spawns ──► ui-engineer   │  │  spawns ──► api-engineer ┐│
         │             parallel      │  │  spawns ──► db-engineer  ││
         │                           │  │             parallel      ││
         └───────────────────────────┴──┴───────────────────────────┘
                           │ both complete, gates pass
         ┌─────────────────▼──────────────────┐
         │         PHASE 3 (blocking)          │
         │       devops coordinator            │
         │                                     │
         │  spawns ──► devops-lead      ┐      │
         │  spawns ──► container-engineer│ parallel
         │  spawns ──► cicd-engineer (blocking after above)
         └─────────────────┬──────────────────┘
                           │ gates pass
         ┌─────────────────▼──────────────────┐
         │         PHASE 4 (blocking)          │
         │      code-review coordinator        │
         │                                     │
         │  spawns ──► static-analyst    ┐     │
         │  spawns ──► integration-checker│ parallel
         │  spawns ──► review-lead (blocking — aggregates + signs off)
         └─────────────────┬──────────────────┘
                           │
                    APPROVED / NEEDS_FIXES
```

---

## Agent Roster (20 agents)

### Team Coordinators — spawn and orchestrate their members

| Agent | File | Model | Role |
|-------|------|-------|------|
| `system-design` | `.claude/agents/system-design.md` | opus | Phase 1 coordinator |
| `frontend` | `.claude/agents/frontend.md` | opus | Phase 2a coordinator |
| `backend` | `.claude/agents/backend.md` | opus | Phase 2b coordinator |
| `devops` | `.claude/agents/devops.md` | opus | Phase 3 coordinator |
| `code-review` | `.claude/agents/code-review.md` | opus | Phase 4 coordinator |

### System Design Team

| Agent | File | Model | Produces |
|-------|------|-------|---------|
| `architect` | `.claude/agents/architect.md` | opus | ARCHITECTURE.md, DOCKER_TOPOLOGY.md, architecture.json |
| `api-designer` | `.claude/agents/api-designer.md` | opus | API_SPEC.md |
| `data-modeler` | `.claude/agents/data-modeler.md` | opus | DB_SCHEMA.sql, schema-summary.md |

### Frontend Team

| Agent | File | Model | Produces |
|-------|------|-------|---------|
| `figma-converter` | `.claude/agents/figma-converter.md` | sonnet | tokens.ts, COMPONENT_SPECS.md, design-brief.md |
| `frontend-lead` | `.claude/agents/frontend-lead.md` | sonnet | Project scaffold, api.ts, auth store, routing |
| `ui-engineer` | `.claude/agents/ui-engineer.md` | sonnet | All React components and pages |

### Backend Team

| Agent | File | Model | Produces |
|-------|------|-------|---------|
| `backend-lead` | `.claude/agents/backend-lead.md` | sonnet | Express app, middleware, config |
| `api-engineer` | `.claude/agents/api-engineer.md` | sonnet | Routes, controllers, services, Zod schemas |
| `database-engineer` | `.claude/agents/database-engineer.md` | sonnet | Models, migrations |

### DevOps Team

| Agent | File | Model | Produces |
|-------|------|-------|---------|
| `devops-lead` | `.claude/agents/devops-lead.md` | sonnet | docker-compose.yml, nginx config, .env.example |
| `container-engineer` | `.claude/agents/container-engineer.md` | sonnet | Dockerfile.frontend, Dockerfile.backend |
| `cicd-engineer` | `.claude/agents/cicd-engineer.md` | sonnet | ci.yml, deploy.yml |

### Code Review Team

| Agent | File | Model | Produces |
|-------|------|-------|---------|
| `static-analyst` | `.claude/agents/static-analyst.md` | sonnet | static-findings.md |
| `integration-checker` | `.claude/agents/integration-checker.md` | sonnet | wiring-matrix.md |
| `review-lead` | `.claude/agents/review-lead.md` | opus | review-report.md, review-signoff.json or errors.json |

---

## How Teams Interact

Communication between teams is entirely file-based through `memory/handoffs/`. No agent passes data to another agent directly in its prompt.

### Handoff flow

```
system-design ──► system-design-to-frontend.json ──► frontend coordinator
             ──► system-design-to-backend.json  ──► backend coordinator

frontend     ──► frontend-to-devops.json ──► devops coordinator
backend      ──► backend-to-devops.json  ──► devops coordinator

devops       ──► devops-to-review.json   ──► code-review coordinator

code-review  ──► review-signoff.json  (APPROVED)
             ──► errors.json          (NEEDS_FIXES → orchestrator re-routes to team)
```

### Intra-team scratch space

Team members pass information to each other within the same phase:

```
memory/intra-team/system-design/
    architect-brief.md      ← architect → api-designer + data-modeler
    schema-summary.md       ← data-modeler → api-engineer + database-engineer

memory/intra-team/frontend/
    design-brief.md         ← figma-converter → frontend-lead + ui-engineer

memory/intra-team/code-review/
    static-findings.md      ← static-analyst → review-lead
    wiring-matrix.md        ← integration-checker → review-lead
```

### Gate checks

The orchestrator verifies specific files exist before advancing each phase. If a gate fails, the responsible team is re-invoked with the specific gap noted — the pipeline does not proceed on incomplete output.

---

## What Gets Generated

After a successful pipeline run, your project lives in `output/`:

```
output/
├── docs/
│   ├── ARCHITECTURE.md       ← system overview, tech stack, ASCII diagram
│   ├── API_SPEC.md           ← every endpoint with request/response schemas
│   ├── DB_SCHEMA.sql         ← complete PostgreSQL DDL
│   ├── COMPONENT_TREE.md     ← pages, routes, component hierarchy
│   ├── DOCKER_TOPOLOGY.md    ← services, ports, networks, volumes
│   ├── COMPONENT_SPECS.md    ← per-component props, states, API wiring
│   └── review-report.md      ← code review findings and sign-off
│
├── frontend/                 ← Next.js 14 + TypeScript + Tailwind app
│   ├── package.json
│   ├── src/
│   │   ├── app/              ← App Router pages
│   │   ├── components/
│   │   │   ├── ui/           ← Button, Input, Modal, Toast, Spinner...
│   │   │   └── features/     ← domain components
│   │   ├── lib/api.ts        ← typed fetch wrapper
│   │   ├── store/auth.ts     ← Zustand auth store
│   │   └── types/index.ts    ← TypeScript interfaces
│   └── .env.example
│
├── backend/                  ← Express 4 + TypeScript + PostgreSQL API
│   ├── package.json
│   ├── src/
│   │   ├── app.ts            ← Express + middleware
│   │   ├── middleware/       ← JWT auth, Zod validation, error handler
│   │   ├── routes/           ← one file per resource
│   │   ├── controllers/      ← thin request handlers
│   │   ├── services/         ← business logic
│   │   └── models/           ← parameterized pg queries
│   ├── migrations/           ← SQL migration files
│   └── .env.example
│
└── devops/                   ← Docker + CI/CD
    ├── Dockerfile.frontend   ← multi-stage Next.js build
    ├── Dockerfile.backend    ← multi-stage TypeScript build
    ├── docker-compose.yml    ← full stack: nginx + frontend + backend + postgres
    ├── docker-compose.prod.yml
    ├── nginx/default.conf    ← reverse proxy config
    ├── .env.example          ← all env vars for the full stack
    ├── .github/workflows/
    │   ├── ci.yml            ← typecheck + lint + build + docker on PRs
    │   └── deploy.yml        ← push to ghcr.io on main merge
    └── scripts/
        ├── start.sh
        └── healthcheck.sh
```

---

## How to Start Using It

### Step 1 — Open Claude Code in this directory

```bash
cd /path/to/agent-team
claude
```

### Step 2 — Give it a project

Just describe your idea naturally. Examples:

```
Build me a task management app where teams can create projects, assign tasks,
set deadlines, and track progress. Include user auth and a dashboard.
```

```
Build a recipe sharing platform where users can post recipes with ingredients
and steps, browse by category, save favorites, and leave ratings.
Figma URL: https://figma.com/...
```

```
Build a SaaS billing dashboard where companies can manage subscriptions,
view invoice history, update payment methods, and see usage metrics.
```

### Step 3 — Watch the pipeline run

Claude Code will:
1. Spawn `system-design` → architect designs the system, then api-designer + data-modeler run in parallel
2. Spawn `frontend` + `backend` simultaneously → both teams build in parallel
3. Spawn `devops` → containerizes everything
4. Spawn `code-review` → static analysis + wiring checks run in parallel, then review-lead signs off

You'll see status updates after each phase:
```
Phase 1 complete — Next.js + Express + PostgreSQL stack. Moving to parallel frontend + backend.
Phase 2 complete — frontend and backend implemented. Moving to DevOps.
Phase 3 complete — Docker stack ready. Running code review.
Phase 4 complete — APPROVED. Full stack ready. See output/docs/review-report.md.
```

### Step 4 — Run the stack

```bash
cd output
cp devops/.env.example .env
# Edit .env with your values
docker compose -f devops/docker-compose.yml up --build
```

Visit `http://localhost` — nginx serves the frontend and proxies `/api/*` to the backend.

---

## Resuming an Interrupted Pipeline

If the pipeline was interrupted, Claude Code reads `memory/context/project-status.md` at session start and picks up from the last incomplete phase. You don't need to restart from scratch.

---

## Review Loop

If Code Review finds CRITICAL issues, the orchestrator automatically:
1. Reads `memory/handoffs/errors.json` to identify which team owns each issue
2. Re-invokes that team with the specific fix instructions
3. Re-runs Code Review
4. Repeats up to 3 times before escalating to you

---

## Parallel Execution Summary

| Phase | What runs in parallel |
|-------|----------------------|
| System Design Phase B | `api-designer` + `data-modeler` |
| Phase 2 | `frontend` team + `backend` team (entire teams) |
| Frontend Phase B | `frontend-lead` + `ui-engineer` |
| Backend Phase B | `api-engineer` + `database-engineer` |
| DevOps Phase A | `devops-lead` + `container-engineer` |
| Code Review Phase A | `static-analyst` + `integration-checker` |

---

## Tech Stack (default)

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14, TypeScript, Tailwind CSS, Zustand |
| Backend | Express 4, TypeScript, Node.js 20 |
| Database | PostgreSQL 16 (raw `pg` queries, parameterized) |
| Auth | JWT + bcrypt |
| Proxy | nginx |
| Containers | Docker + docker-compose |
| CI/CD | GitHub Actions → ghcr.io |

The architect agent will adapt this stack if your project description suggests different requirements.

---

## File Structure

```
agent-team/
├── CLAUDE.md                          ← orchestrator instructions
├── README.md                          ← this file
├── .claude/
│   ├── agents/                        ← all 20 Claude Code subagents
│   │   ├── system-design.md           ← team coordinators (spawn members)
│   │   ├── frontend.md
│   │   ├── backend.md
│   │   ├── devops.md
│   │   ├── code-review.md
│   │   ├── architect.md               ← system design members
│   │   ├── api-designer.md
│   │   ├── data-modeler.md
│   │   ├── figma-converter.md         ← frontend members
│   │   ├── frontend-lead.md
│   │   ├── ui-engineer.md
│   │   ├── backend-lead.md            ← backend members
│   │   ├── api-engineer.md
│   │   ├── database-engineer.md
│   │   ├── devops-lead.md             ← devops members
│   │   ├── container-engineer.md
│   │   ├── cicd-engineer.md
│   │   ├── review-lead.md             ← code review members
│   │   ├── static-analyst.md
│   │   └── integration-checker.md
│   ├── rules/                         ← pipeline rules and quality gates
│   └── config.json
├── memory/
│   ├── handoffs/                      ← inter-team JSON contracts
│   ├── decisions/                     ← immutable architectural decisions
│   ├── intra-team/                    ← scratch space within teams
│   └── context/project-status.md     ← pipeline progress tracker
├── output/                            ← everything generated goes here
│   ├── docs/
│   ├── frontend/
│   ├── backend/
│   └── devops/
└── playbooks/                         ← workflow playbooks for reference
```
