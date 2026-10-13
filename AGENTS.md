# Project Overview

**hex-ai-mcp** is a maintained downstream fork of [MetaMCP](https://github.com/metatool-ai/metamcp) — an MCP proxy that dynamically aggregates multiple MCP servers into a single unified MCP server with middleware support. Because MetaMCP is itself an MCP server, it plugs into any MCP client (Claude.ai, Claude Code, n8n, etc.).

This fork concentrates on **reliability of a self-hosted gateway that stays connected**: session survival across restarts, OAuth longevity, connection-pool robustness, RBAC access control, and observability. It is owned by HEX s.r.o. and deployed as internal infrastructure for multilingual AI-agent tooling.

## Operational Metadata

**Version:** `v1.0` — Unified Agent Prompts for this project.
**Canonical doc:** this file (`AGENTS.md`).
**Repo:** `git@github.com:HEX-s-r-o/hex-ai-mcp.git`
**Default branch:** `umbrella` (deployable line). `main` mirrors upstream and never carries our changes.

## Repository Structure

```
hex-ai-mcp/
├── apps/
│   ├── backend/          # Express 5.1 + tRPC API, MCP proxy engine, auth, pools
│   └── frontend/         # Next.js 15 management UI (namespaces, endpoints, servers, logs)
├── packages/
│   ├── eslint-config/    # Shared ESLint presets (base, Next.js, React, Express)
│   ├── trpc/             # Shared tRPC router definitions and client setup
│   ├── typescript-config/# Shared tsconfig presets
│   └── zod-types/        # Shared Zod schemas used across backend and frontend
├── docs/                 # Additional documentation (optional deep-dive topics)
├── scripts/              # Operational scripts (ensure-runtime-role.sh, etc.)
├── .github/workflows/    # CI: umbrella-build.yml, ghcr-cleanup.yml
├── docker-compose.yml    # Production compose (loopback-bind, GHCR image)
├── docker-compose.dev.yml# Dev compose (hot reload, ports 12008/12009/9433)
├── Dockerfile            # Multi-stage production build (uv base + Node 20 + pnpm)
├── Dockerfile.dev          # Dev image with tsx/watch support
├── example.env           # Annotated environment template (REQUIRED reading before first boot)
├── UMBRELLA_FORK.md      # Per-change ledger: every patch, upstream relation, rationale
└── README.md             # Full user-facing documentation (concepts, quick start, config)
```

## Build & Development Commands

Install dependencies (pnpm 9 required):
```bash
pnpm install
```

Development (local, requires `.env` from `example.env`):
```bash
# 1. Copy and edit example.env first (POSTGRES_PASSWORD, BETTER_AUTH_SECRET are REQUIRED)
cp example.env .env

# 2. Start backend + frontend + postgres in Docker with hot reload
pnpm run dev:docker

# 3. Stop / clean
pnpm run dev:docker:down
pnpm run dev:docker:clean   # removes volumes
```

Local dev without Docker (requires local Postgres):
```bash
pnpm dev          # turbo runs frontend (12008) and backend (12009) concurrently
```

Build:
```bash
pnpm build        # turbo build across all packages and apps
```

Quality:
```bash
pnpm check-types  # tsc --noEmit across workspace
pnpm lint         # turbo lint (advisory — frontend currently has warnings)
pnpm lint:fix     # auto-fix where available
pnpm format       # prettier --write "**/*.{ts,tsx,md}"
pnpm test         # turbo test (vitest in backend + frontend)
```

Backend-only quick commands:
```bash
pnpm -C apps/backend test
pnpm -C apps/backend db:migrate
pnpm -C apps/backend db:generate
```

## Code Style & Conventions

- **Language:** TypeScript 5.8; strict mode enabled.
- **Formatter:** Prettier 3.x; run `pnpm format` before committing.
- **Linter:** ESLint 9 with flat config; backend uses `--max-warnings 0`, frontend currently advisory.
- **Imports:** `simple-import-sort` plugin enforces grouping; `unused-imports` plugin catches dead imports.
- **Naming:**
  - Functions / variables: `camelCase`
  - React components: `PascalCase`
  - tRPC routers / Zod schemas: `kebab-case.ts` files exporting `PascalCase` schemas
  - Database tables: `snake_case` in Drizzle schema; repository files suffix `.repo.ts`
- **Workspace aliases:** `@repo/zod-types`, `@repo/trpc` for cross-package imports; `@/` alias for intra-app imports.
- **Commits:** PRs are squash-merged; PR title becomes commit subject. No secrets, internal hostnames, or credentials in code, comments, tests, or commit messages.

## Architecture Notes

```
┌─────────────────────────────────────────────────────────────┐
│                        MCP Clients                           │
│  (Claude.ai, Claude Code, n8n, custom agents)                │
└────────────┬────────────────────────────────────────────────┘
             │ Streamable HTTP / SSE / OpenAPI
             ▼
┌─────────────────────────────────────────────────────────────┐
│                    hex-ai-mcp Gateway                        │
│  ┌──────────────┐  ┌─────────────────────────────────────┐  │
│  │  Frontend    │  │  Backend (Express + tRPC)           │  │
│  │  Next.js 15  │  │  ┌─────────┐ ┌──────────────────┐ │  │
│  │  Port 12008  │  │  │  Auth     │ │  MCP Proxy Engine│ │  │
│  │  (UI + proxy)│  │  │better-auth│ │  ┌────────────┐  │ │  │
│  └──────────────┘  │  │  OIDC     │ │  │Session Pool│  │ │  │
│                    │  │  RBAC     │ │  │Conn Mgmt   │  │ │  │
│                    │  └─────────┘ │  │Recovery    │  │ │  │
│                    │              │  │Auditing    │  │ │  │
│                    │  ┌─────────┐ │  └────────────┘  │ │  │
│                    │  │  tRPC   │ │  ┌────────────┐  │ │  │
│                    │  │ Routers │ │  │Tool Call   │  │ │  │
│                    │  │  Admin  │ │  │Audit Log   │  │ │  │
│                    │  │  Public │ │  └────────────┘  │ │  │
│                    │  └─────────┘ └──────────────────┘ │  │
│                    │           Port 12009                 │  │
│                    └─────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────┘
                             │
             ┌───────────────┼───────────────┐
             ▼               ▼               ▼
        ┌─────────┐   ┌─────────┐   ┌─────────────┐
        │Postgres │   │Upstream │   │MCP Servers  │
        │ 16      │   │MCP Svrs │   │(STDIO/SSE)  │
        │(pooled) │   │(HTTP)   │   │             │
        └─────────┘   └─────────┘   └─────────────┘
```

**Key architectural concepts:**
1. **Namespace** — a logical grouping of MCP servers exposed under one endpoint.
2. **Endpoint** — a public URL path (`/:endpoint_name/mcp`) where clients connect.
3. **Server** — an actual MCP backend (STDIO command, SSE URL, or Streamable HTTP URL).
4. **Connection pool** — LRU-managed, cap-limited pool of `ConnectedClient` instances with health sweeps, error-gated recovery, and transport-drop detection.
5. **Session recovery** — persisted `mcp_sessions` table allows gateway restarts to rebuild transports and replay the MCP initialize handshake transparently.
6. **Audit pipeline** — every proxied `tools/call` is logged with caller identity, tool, server, duration, and outcome; stored in `tool_call_audit` with configurable retention.

## Testing Strategy

- **Unit tests:** vitest across both apps.
  - Backend: `apps/backend/src/**/*.test.ts` — 300+ cases covering pool logic, session recovery, error detectors, OAuth, rate limiting, audit middleware.
  - Frontend: `apps/frontend/**/*.test.ts` — component and utility tests.
- **Run backend tests:** `pnpm -C apps/backend test` or `pnpm -C apps/backend test:watch`
- **Run frontend tests:** `pnpm -C apps/backend test` (alias from root via turbo)
- **Coverage:** `pnpm -C apps/backend test:coverage` (vitest --coverage)
- **Type checking:** `pnpm check-types` — **must pass** before PR merge.
- **CI gate:** every merge to `umbrella` passes type-check, lint, backend vitest, and a full production Docker build.
- **Integration / e2e:** no automated e2e suite currently; Docker compose smoke test is the integration validation.

## Security & Compliance

- **Secrets:**
  - `POSTGRES_PASSWORD` and `BETTER_AUTH_SECRET` are **required** with no fallback; compose refuses to render if unset.
  - API keys are SHA-256 hashed at rest; only the last four characters are stored for display.
  - Secrets never appear in API list/get/update responses.
- **Auth:** better-auth (sessions + OAuth2 server) + optional OIDC SSO. Self-registration defaults to disabled.
- **RBAC:** `users.role` is `admin` or `member`; destructive mutations gated behind `adminProcedure`.
- **Rate limiting:** per-IP cap on password sign-in; bounded dynamic client registration.
- **Audit:** append-only `audit_log` (UPDATE/TRUNCATE refused at DB level; DELETE refused inside 30-day window).
- **Runtime DB split:** optional `METAMCP_RUNTIME_DB_PASSWORD` creates a NOSUPERUSER role for request serving, leaving migration superuser separate.
- **Dependency scanning:** pnpm overrides in root `package.json` patch known CVEs; workflow `uses:` pinned to commit SHA; Dockerfile base pinned to digest.
- **License:** MIT (upstream copyright preserved).

## Agent Guardrails

1. **Never modify** `UMBRELLA_FORK.md` unless you are recording a merged fork PR — it is the canonical per-change ledger.
2. **Never modify** `.github/workflows/*.yml` workflow `uses:` pins without updating the commit-SHA test that guards them.
3. **Never touch** `main` branch directly — it mirrors upstream. All deployable work lands on `umbrella`.
4. **Never invent** environment-variable defaults for secrets (POSTGRES_PASSWORD, BETTER_AUTH_SECRET); use `:?` compose semantics or explicit placeholders.
5. **Required review:** any change to OAuth token handling, auth middleware, or database migrations must be flagged for maintainer review before merge.
6. **Migration numbering:** drizzle migrations use `when` timestamps; a rename without bumping `when` above the highest existing value is **silently skipped**.
7. **No credentials** in code, comments, tests, or commit messages — ever.

## Extensibility Hooks

- **Environment variables:** `example.env` documents ~80 tunables: pool caps, TTLs, backoff schedules, retention days, OIDC config, M365 broker, branding overrides.
- **Feature flags (env-gated):**
  - `ALLOW_UNAUTHENTICATED_ENDPOINTS=true` — publishes endpoints with no auth (dangerous; default false).
  - `ADMIN_PLANE_TOKEN_AUTH_DISABLED=true` — killswitch for admin-plane bearer auth.
  - `MCP_ACTIVE_HEALTH_CHECK` / `MCP_AUTO_NUKE_ON_CAPABILITY_CHANGE` — runtime pool/behavior toggles.
- **Plugin points:**
  - New MCP backend types: add to `mcpServersRepository` schema + pool connection factory.
  - New audit retention: `GATEWAY_EVENTS_RETENTION_DAYS`, `TOOL_AUDIT_RETENTION_DAYS`.
  - New auth providers: extend better-auth config in `apps/backend/src/auth.ts`.
- **Branding:** `NEXT_PUBLIC_BRANDING_*` and `BRANDING_*` variables replace product name, org, logo, and description without code changes.

## Further Reading

- [`UMBRELLA_FORK.md`](./UMBRELLA_FORK.md) — every patch, upstream PR mapping, merge rationale.
- [`README.md`](./README.md) — full user guide: quick start, configuration, concepts, client setup.
- [`SECURITY.md`](./SECURITY.md) — vulnerability reporting, threat model, scope, disclosure policy.
- [`CONTRIBUTING.md`](./CONTRIBUTING.md) — dev setup, PR guidelines, CI commands.
- [`example.env`](./example.env) — annotated reference for all environment variables.
- [`docs/`](./docs/) — optional deep-dive documentation (add topics here rather than README when they grow long).
