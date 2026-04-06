# CLAUDE.md

Guidance for AI assistants (Claude Code and others) working in this repository.

> **Start here, then read:** `AGENTS.md` → `doc/GOAL.md` → `doc/PRODUCT.md` → `doc/SPEC-implementation.md` → `doc/DEVELOPING.md` → `doc/DATABASE.md`

---

## What Is Paperclip

Paperclip is a **control plane for AI-agent companies** — a full-stack Node.js application that orchestrates autonomous AI agents working within structured company hierarchies. It manages tasks, approvals, budgets, governance, and agent coordination.

---

## Repository Structure

```
paperclip/
├── server/              # Express 5 REST API + orchestration services
├── ui/                  # React 19 + Vite frontend board UI
├── cli/                 # Command-line interface (paperclipai)
├── packages/
│   ├── db/             # Drizzle ORM schema, migrations, DB clients
│   ├── shared/         # Shared types, constants, validators, API paths
│   ├── adapter-utils/  # Utilities shared across adapters
│   ├── adapters/       # Agent adapter implementations
│   └── plugins/        # Plugin system SDK + examples
├── skills/             # Agent skill documentation
├── doc/                # Operational and product documentation
├── docker/             # Docker configs and compose files
├── tests/              # E2E (Playwright) and smoke tests
├── scripts/            # Build, release, and utility scripts
├── docs/               # Mintlify documentation site
└── evals/              # LLM evaluation tests (promptfoo)
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 20+ |
| Language | TypeScript 5.7.3 (strict, ES2023, NodeNext modules) |
| Package Manager | pnpm 9.15.4 (workspaces monorepo) |
| Backend | Express 5.1 |
| Database ORM | Drizzle ORM + PostgreSQL |
| Database (dev) | Embedded PostgreSQL (auto, no setup needed) |
| Auth | better-auth (sessions for board, hashed API keys for agents) |
| Validation | Zod |
| Frontend | React 19 + Vite 6 + React Router 7 + TanStack Query 5 |
| Styling | Tailwind CSS 4 + Radix UI |
| Testing (unit) | Vitest 3 |
| Testing (E2E) | Playwright |
| Logging | Pino (structured JSON) |

---

## Development Setup

```sh
pnpm install
pnpm dev          # API + UI with watch mode on http://localhost:3100
```

- API: `http://localhost:3100`
- UI: served by the API server at the same origin (dev middleware mode)
- No external PostgreSQL needed — embedded DB auto-creates at `data/pglite` (dev) or `~/.paperclip/instances/default/db` (CLI run)

**Useful dev commands:**

```sh
pnpm dev:once     # Start without watch mode (auto-applies pending migrations)
pnpm dev:list     # Show running dev instances
pnpm dev:stop     # Stop managed dev runner
pnpm dev:server   # Server only
pnpm dev:ui       # UI only

# Health check
curl http://localhost:3100/api/health

# Reset dev DB (embedded PGlite)
rm -rf data/pglite && pnpm dev
```

**One-command first-time setup:**

```sh
pnpm paperclipai run   # auto-onboard → doctor → start
```

---

## Core Engineering Rules

These rules are non-negotiable. Follow them on every change.

### 1. Company Scoping
Every domain entity is scoped to a `companyId`. All routes and services must enforce company boundaries. An agent's API key must never access another company's data.

### 2. Synchronized Contracts
When you change schema or API behavior, update **all impacted layers** in the same PR:
- `packages/db` — schema and exports
- `packages/shared` — types, constants, validators
- `server` — routes and services
- `ui` — API clients and pages

### 3. Control-Plane Invariants (Never Break)
- Single-assignee task model
- Atomic issue checkout semantics
- Approval gates for governed actions (hiring, config changes)
- Budget hard-stop auto-pause behavior
- Activity logging for all mutating actions

### 4. HTTP Error Standards
Return consistent HTTP errors: `400` / `401` / `403` / `404` / `409` / `422` / `500`. Use the project's `HttpError` classes.

### 5. Mutations Require Traceability
All agent mutations must include the `X-Paperclip-Run-Id` header. Write activity log entries for every mutation.

---

## Database Workflow

**When changing the data model:**

1. Edit schema files in `packages/db/src/schema/*.ts`
2. Export new tables from `packages/db/src/schema/index.ts`
3. Generate migration:
   ```sh
   pnpm db:generate
   ```
4. Verify compile:
   ```sh
   pnpm -r typecheck
   ```

Notes:
- `drizzle.config.ts` reads compiled schema from `dist/schema/*.js` — run build before `db:generate` if needed
- Migrations live in `packages/db/src/migrations/`
- Migration numbering is validated automatically on build

---

## API Conventions

- **Base path:** `/api`
- **Board access:** full-control operator context (session-based auth via better-auth)
- **Agent access:** bearer API keys (`agent_api_keys` table, hashed at rest)
- **Required on mutations:** `X-Paperclip-Run-Id` header

When adding endpoints:
- Apply company access checks
- Enforce actor permissions (board vs. agent)
- Write activity log entries for mutations
- Return consistent HTTP error codes

**Key routes:**

| Route | Purpose |
|-------|---------|
| `/api/health` | Server health check |
| `/api/companies` | Company CRUD |
| `/api/agents` | Agent management |
| `/api/issues` | Task management (checkout, update, comments, attachments) |
| `/api/projects` | Project management |
| `/api/goals` | Goal tracking |
| `/api/approvals` | Approval workflow |
| `/api/routines` | Scheduled routines |
| `/api/execution-workspaces` | Agent execution workspaces |
| `/api/costs` | Cost tracking and budget enforcement |
| `/api/plugins` | Plugin management |
| `/api/secrets` | Company secrets |
| `/api/assets` | File/asset storage |
| `/api/adapters` | Adapter configuration |

---

## Testing

```sh
# Type check all packages
pnpm -r typecheck

# Unit/integration tests (watch mode)
pnpm test

# Unit/integration tests (CI mode)
pnpm test:run

# E2E tests
pnpm test:e2e
pnpm test:e2e:headed    # with browser visible

# Release smoke tests
pnpm test:release-smoke
```

**Test locations:**
- `server/src/__tests__/` — API route and service tests
- `ui/src/components/__tests__/` — UI component tests
- `tests/e2e/` — Full E2E suite (Playwright)
- `tests/release-smoke/` — Post-release smoke tests
- `evals/promptfoo/` — LLM evaluation tests

---

## Verification Checklist (Before Claiming Done)

Run all three before handing off:

```sh
pnpm -r typecheck
pnpm test:run
pnpm build
```

If any step cannot run, explicitly report which step was skipped and why.

---

## Key Architectural Patterns

### Heartbeat-Driven Agent Orchestration
Agents are woken on scheduled heartbeats or event triggers. Each run injects context via `PAPERCLIP_*` environment variables. Agents run finite-lifetime jobs and exit. Results are captured in heartbeat runs and the activity log.

### Service Layer
Each domain has a dedicated service module (e.g., `issueService`, `agentService`, `budgetService`). All services accept `db: Db` as a dependency.

### Adapter System
Agent implementations are modular adapters in `packages/adapters/`. External adapters can be loaded as plugins via `~/.paperclip/adapter-plugins.json`. Current built-in adapters: `claude-local`, `codex-local`, `cursor-local`, `openclaw-gateway`, `opencode-local`, `gemini-local`, `pi-local`.

### Plugin System
Worker-based plugin execution with manifest validation. Plugins can expose UI components, jobs, webhooks, and entities. Plugin capabilities are gated by manifest declarations.

### Storage Abstraction
Pluggable storage backends (local disk, S3). File uploads via multer, image processing via Sharp, attachment metadata in the database.

---

## UI Conventions

- Use company selection context for all company-scoped pages
- Keep routes and nav aligned with available API surface
- Surface API errors clearly — do not silently swallow failures
- Path alias `@/` maps to `ui/src/`
- Dev proxy: Vite proxies `/api` to `localhost:3100`

---

## Plans and Documentation

- New plan documents → `doc/plans/YYYY-MM-DD-slug.md`
- Do not replace `doc/SPEC.md` or `doc/SPEC-implementation.md` wholesale — prefer additive updates
- Update docs when behavior or commands change

---

## Lockfile Policy

**Do not commit `pnpm-lock.yaml` in pull requests.** GitHub Actions owns the lockfile. CI validates dependency resolution when manifests change; pushes to `master` regenerate it automatically.

---

## Pull Request Requirements

Read and fill in every section of `.github/PULL_REQUEST_TEMPLATE.md` for every PR. Required sections:

- **Thinking Path** — trace reasoning from project context to this change
- **What Changed** — concrete bullet list
- **Verification** — how a reviewer confirms it works
- **Risks** — what could go wrong
- **Model Used** — AI model that assisted (provider, model ID, context) or "None — human-authored"
- **Checklist** — all items checked

---

## Definition of Done

A change is complete when **all** of these are true:

1. Behavior matches `doc/SPEC-implementation.md`
2. Typecheck, tests, and build pass
3. Contracts are synced across db / shared / server / ui
4. Docs updated when behavior or commands change
5. PR description follows the PR template with all sections filled in
