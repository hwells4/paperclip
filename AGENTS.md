# AGENTS.md

Guidance for human and AI contributors working in this repository.

## 1. Purpose

Paperclip is a control plane for AI-agent companies.
The current implementation target is V1 and is defined in `doc/SPEC-implementation.md`.

## 2. Read This First

Before making changes, read in this order:

1. `doc/GOAL.md`
2. `doc/PRODUCT.md`
3. `doc/SPEC-implementation.md`
4. `doc/DEVELOPING.md`
5. `doc/DATABASE.md`

`doc/SPEC.md` is long-horizon product context.
`doc/SPEC-implementation.md` is the concrete V1 build contract.

## 3. Repo Map

- `server/`: Express REST API and orchestration services
- `ui/`: React + Vite board UI
- `packages/db/`: Drizzle schema, migrations, DB clients
- `packages/shared/`: shared types, constants, validators, API path constants
- `doc/`: operational and product docs

## 4. Dev Setup (Auto DB)

Use embedded PGlite in dev by leaving `DATABASE_URL` unset.

```sh
pnpm install
pnpm dev
```

This starts:

- API: `http://localhost:3100`
- UI: `http://localhost:3100` (served by API server in dev middleware mode)

Quick checks:

```sh
curl http://localhost:3100/api/health
curl http://localhost:3100/api/companies
```

Reset local dev DB:

```sh
rm -rf data/pglite
pnpm dev
```

## 5. Core Engineering Rules

1. Keep changes company-scoped.
Every domain entity should be scoped to a company and company boundaries must be enforced in routes/services.

2. Keep contracts synchronized.
If you change schema/API behavior, update all impacted layers:
- `packages/db` schema and exports
- `packages/shared` types/constants/validators
- `server` routes/services
- `ui` API clients and pages

3. Preserve control-plane invariants.
- Single-assignee task model
- Atomic issue checkout semantics
- Approval gates for governed actions
- Budget hard-stop auto-pause behavior
- Activity logging for mutating actions

4. Do not replace strategic docs wholesale unless asked.
Prefer additive updates. Keep `doc/SPEC.md` and `doc/SPEC-implementation.md` aligned.

## 6. Database Change Workflow

When changing data model:

1. Edit `packages/db/src/schema/*.ts`
2. Ensure new tables are exported from `packages/db/src/schema/index.ts`
3. Generate migration:

```sh
pnpm db:generate
```

4. Validate compile:

```sh
pnpm -r typecheck
```

Notes:
- `packages/db/drizzle.config.ts` reads compiled schema from `dist/schema/*.js`
- `pnpm db:generate` compiles `packages/db` first

## 7. Verification Before Hand-off

Run this full check before claiming done:

```sh
pnpm -r typecheck
pnpm test:run
pnpm build
```

If anything cannot be run, explicitly report what was not run and why.

## 8. API and Auth Expectations

- Base path: `/api`
- Board access is treated as full-control operator context
- Agent access uses bearer API keys (`agent_api_keys`), hashed at rest
- Agent keys must not access other companies

When adding endpoints:

- apply company access checks
- enforce actor permissions (board vs agent)
- write activity log entries for mutations
- return consistent HTTP errors (`400/401/403/404/409/422/500`)

## 9. UI Expectations

- Keep routes and nav aligned with available API surface
- Use company selection context for company-scoped pages
- Surface failures clearly; do not silently ignore API errors

## 10. Definition of Done

A change is done when all are true:

1. Behavior matches `doc/SPEC-implementation.md`
2. Typecheck, tests, and build pass
3. Contracts are synced across db/shared/server/ui
4. Docs updated when behavior or commands change

## 11. Pull Request Rules

**All PRs must target the `main` branch.** The CI pipeline only triggers on PRs to `main`. PRs targeting other branches will not receive checks, reviews, or auto-merge.

When creating a PR:

- Target `main` as the base branch
- Write a clear title (under 70 characters) and description
- One logical change per PR
- Ensure typecheck, tests, and build pass locally before pushing

Do not manually edit `pnpm-lock.yaml` — CI owns lockfile updates. The exception is the `chore/refresh-lockfile` branch.

## 12. CI/CD Pipeline

### What happens when you open a PR to `main`

Three workflows run automatically:

1. **pr-verify** — runs `pnpm -r typecheck`, `pnpm test:run`, and `pnpm build`. Must pass.
2. **pr-policy** — enforces lockfile policy (blocks manual lockfile edits) and validates dependency resolution.
3. **pr-review** — runs 7 specialized Claude Code review agents in parallel:
   - Architecture (patterns, boundaries, dependency direction)
   - Security (auth, injection, secrets, input validation)
   - Performance (N+1 queries, missing indexes, re-renders, caching)
   - Pattern Consistency (naming, file organization, imports, types)
   - Code Simplicity (over-engineering, dead code, complexity)
   - Data Integrity (migration safety, data loss, referential integrity)
   - Contract Sync (db/shared/server/ui stay in sync)

   Each reviewer posts findings as a PR comment categorized as Critical, Warning, or Suggestion.

### Auto-merge decision

After all 7 reviewers complete, a `merge-decision` job runs:

- **Low risk + no Critical findings** → PR is auto-merged (`--merge --delete-branch`)
- **High risk OR any Critical finding** → PR is blocked with a summary comment for human review

High-risk files (require human review):
- `packages/db/src/schema/` and `packages/db/drizzle/` (schema and migrations)
- `server/src/auth/` and `server/src/middleware/` (auth and middleware)
- `.env*`, `Dockerfile`, `docker-compose*` (infrastructure)

### What happens after merge to `main`

Two workflows trigger on push to `main`:

1. **deploy** — SSHs into the VPS and runs the deploy script automatically
2. **refresh-lockfile** — regenerates `pnpm-lock.yaml` and opens a PR if it changed

### Summary: PR lifecycle

```
push branch → open PR to main → pr-verify + pr-policy + pr-review run
  → all pass + low risk + no criticals → auto-merge → deploy to VPS
  → high risk or criticals → blocked, human reviews
```

## 13. Deployment

- Production deploys happen automatically when code is merged to `main`
- The deploy workflow SSHs to the VPS and runs `/home/ubuntu/deploy/paperclip-deploy.sh`
- Rollback is available via the `rollback.yml` workflow (manual dispatch)
- Do not push directly to `main` — always go through a PR so the review pipeline runs
