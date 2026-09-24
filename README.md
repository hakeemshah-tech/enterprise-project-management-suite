# Enterprise Project Management Suite (EPMS)

Internal engineering documentation · Platform Engineering · v1.0.0-enterprise

EPMS is the internal project and workflow management platform: organisations,
workspaces, projects, sprints, tasks, time tracking, and an automation/AI task
execution layer. This document is the entry point for engineers operating,
extending, or deploying the platform. It assumes you have shell access to a
development machine and credentials for the internal container registry.

---

## 1. Architecture Overview

### 1.1 Shape of the system

EPMS is a **workspace monorepo containing two independently buildable
applications** backed by shared stateful infrastructure. It is deliberately
*not* a microservice fleet: the domain is decomposed into modules inside a
single NestJS process rather than into separately deployed services.

```
┌──────────────────────────────────────────────────────────────┐
│  Application container (Dockerfile.dev / Dockerfile.prod)    │
│                                                              │
│   ┌────────────────────┐        ┌────────────────────────┐   │
│   │  Frontend          │        │  Backend               │   │
│   │  Next.js (React)   │ ─────► │  NestJS 11             │   │
│   │  :3001 (dev)       │  HTTP  │  :3000  /api           │   │
│   │                    │ ◄───── │  Socket.IO gateway     │   │
│   └────────────────────┘   WS   └────────────────────────┘   │
└───────────────────────────────┬──────────────────────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              ▼                                   ▼
    ┌───────────────────┐               ┌───────────────────┐
    │  PostgreSQL 16    │               │  Redis 7          │
    │  Prisma ORM       │               │  BullMQ queues    │
    │  postgres_data    │               │  redis_data       │
    └───────────────────┘               └───────────────────┘
```

**Important operational detail:** in the current Compose topology the frontend
and backend run as two processes inside a *single* `app` container
(`concurrently` supervises both; see the root `dev` script). Postgres and
Redis are separate services on the `epms-network` bridge. The two applications
are build-isolated and could be split into separate containers without code
changes; §2.4 describes what that would take. Documentation and CI refer to
them as distinct tiers because they are distinct deployables, not because they
are currently separate containers.

### 1.2 Backend

NestJS 11 on Node 22, organised into ~34 feature modules under
`backend/src/modules/`. Representative slices:

| Concern | Modules |
|---|---|
| Identity & access | `auth`, `users`, `invitations`, `organization-members`, `workspace-members`, `project-members` |
| Core domain | `organizations`, `workspaces`, `projects`, `sprints`, `tasks`, `workflows`, `task-statuses` |
| Task periphery | `task-comments`, `task-attachments`, `task-dependencies`, `task-watchers`, `task-label`, `labels` |
| Delivery & reporting | `gantt`, `time-entries`, `activity-log`, `search` |
| Async & integrations | `queue`, `scheduler`, `email`, `inbox`, `notifications`, `storage`, `automation`, `ai-chat` |
| Operations | `health`, `settings`, `public` |

Cross-cutting concerns live outside `modules/`: `common/` (guards, filters,
interceptors), `middleware/`, `config/`, `gateway/` (Socket.IO), `prisma/`
(client lifecycle), and `seeder/` (see §3.1).

- **Persistence**: Prisma against PostgreSQL 16. Schema and migrations in
  `backend/prisma/`. The Prisma client is generated at image build time and
  again at container start; it is never committed.
- **Auth**: Passport with JWT access/refresh pairs. `JWT_SECRET` and
  `JWT_REFRESH_SECRET` are independent and must be independently generated.
- **Async work**: BullMQ over Redis. Email delivery, inbox sync, notification
  fan-out, and scheduled jobs are queue-backed; `MAX_CONCURRENT_JOBS` and
  `JOB_RETRY_ATTEMPTS` tune the workers.
- **Encryption at rest**: `ENCRYPTION_KEY` protects stored third-party
  credentials (mail account passwords, integration tokens). Rotating it
  without re-encrypting invalidates every stored secret.
- **API surface**: REST under `/api`, documented via Swagger. Health probe at
  `GET /api/health`; both Compose healthchecks and CI gate on it.

### 1.3 Frontend

Next.js (React 19) in `frontend/src/`, served by a custom `server.js` in
development and `server.mjs` in production.

- `components/`: feature-grouped UI over Radix primitives, styled with
  Tailwind; `next-themes` drives light/dark.
- `pages/`: Next.js Pages Router.
- `contexts/` + `hooks/`: client state and data access. No global store
  library; state is context-scoped by feature.
- `lib/`: API client, MCP server bindings, automation capability loader.
- Forms are `react-hook-form` with `zod` resolvers. Validation schemas are the
  contract: when a DTO changes on the backend, the matching zod schema must
  change with it.

The frontend talks to the backend over `NEXT_PUBLIC_API_BASE_URL`. Because it
is inlined at build time, **the production image is environment-specific**;
rebuild when the API origin changes.

---

## 2. DevOps Strategy

Three Compose files and two Dockerfiles cover the environment matrix. They are
not variations on one file: each targets a different consumer.

### 2.1 The environment matrix

| File | Image source | Intended consumer | Code mount | Entrypoint |
|---|---|---|---|---|
| `docker-compose.dev.yml` | builds `Dockerfile.dev` | engineer workstations, CI | bind-mounts repo | `docker/entrypoint-dev.sh` |
| `docker-compose.prod.yml` | builds `Dockerfile.prod` | release validation, self-hosted deploys | none | `docker/entrypoint.sh` |
| `docker-compose.yml` | pulls `registry.internal.company.com/pm-app:latest` | operators running a published build | none | image default |

The distinction that matters: `docker-compose.prod.yml` **builds** the
production image from source, while `docker-compose.yml` **consumes** an
already-published one. Use the former to validate a change, the latter to run
a release.

### 2.2 Dockerfile strategies

- **`Dockerfile.dev`**: single stage on `node:22-slim`. Installs the Prisma
  and Postgres client toolchain, installs workspace dependencies, generates the
  Prisma client. Source is bind-mounted at runtime, so the image is a
  dependency cache rather than a build artifact. Anonymous volumes mask
  `node_modules` at each workspace root so host and container dependency trees
  never collide; this is why a dependency change requires an image rebuild,
  not just a restart.
- **`Dockerfile.prod`**: multi-stage. Builds both workspaces, emits a pruned
  bundle to `/app/epms/dist`, and copies it into a slim runtime stage owned by
  `www-data`. Accepts `BUILD_NUMBER` (commit SHA or timestamp) for build
  provenance. Runs as non-root.

### 2.3 Image publication

Publication targets the internal registry only; nothing is pushed to public
registries.

```bash
npm run docker:build:dev        # local image, no push
npm run docker:build:prod       # local production image, no push
npm run docker:publish:dev      # multi-arch → :dev, :<version>-dev, :<version>-<sha>
npm run docker:publish:stable   # multi-arch → :latest, :<version>, :<version>-<sha>
```

Registry coordinates are overridable via `INTERNAL_REGISTRY` and `IMAGE_NAME`
(defaults: `registry.internal.company.com` / `pm-app`). Every publish emits a
SHA-pinned tag alongside the moving tag: always deploy the SHA-pinned tag;
`:latest` exists for convenience, not for production.

### 2.4 Splitting the tiers

If frontend and backend need independent scaling, the work is:

1. Split `Dockerfile.prod` into two targets, each with its own runtime stage.
2. Replace the `app` service with `frontend` and `backend` services on the same
   network.
3. Point `NEXT_PUBLIC_API_BASE_URL` at the backend service name.
4. Move `db:migrate` / `db:seed` out of the entrypoint into an init job, so two
   replicas cannot race on migrations.

Item 4 is the blocker today: the entrypoint migrates on every container start,
which is safe at one replica and unsafe at more.

### 2.5 CI

`.github/workflows/enterprise-build.yml` runs three stages:

1. **Static analysis**: Prettier and ESLint, on the host. Fails fast before
   any image is built.
2. **Integration**: a two-leg matrix (`backend`, `frontend`). Each leg brings
   up `docker-compose.dev.yml` under its own `COMPOSE_PROJECT_NAME`, so volumes
   and networks are isolated per leg. Secrets are generated per run with
   `openssl`; no static credentials exist in CI. Both legs gate on
   `/api/health`, run their suite, upload coverage and Playwright reports, and
   tear down with `--volumes`.
3. **Production image build**: verifies `Dockerfile.prod` stays buildable, with
   GitHub Actions layer caching. Does not push; deployment is a separate
   pipeline.

---

## 3. Core Infrastructure

### 3.1 Database seeding

Seeding is implemented as NestJS services in `backend/src/seeder/`
(`users`, `organizations`, `workspaces`, `projects`, `admin-seeder`), not as
loose SQL. They run inside the application context, so they exercise the same
validation and Prisma layer as production code paths.

All seeders are **idempotent**: the entrypoint runs them on every start and
tolerates failure on already-seeded databases.

```bash
npm run db:seed          # full reference dataset
npm run db:seed:admin    # administrative user only
npm run db:seed:clear    # drop seeded rows
npm run db:seed:reset    # clear, then reseed
```

Each has a `test:db:*` counterpart that targets `.env.test` instead of `.env`,
keeping the test database isolated from the development one.

### 3.2 Container orchestration

`docker/entrypoint-dev.sh` is the development bootstrap and runs in a fixed
order: wait for Postgres (TCP, 30 × 2s) → wait for Redis → generate Prisma
client → migrate → seed → seed admin → `exec npm run dev`. Migration and
seeding failures are non-fatal by design, so a container restart against an
already-provisioned database is a no-op rather than a crash loop. Dependency
waits are *not* tolerant: if Postgres or Redis never appears, the container
exits non-zero.

`docker/entrypoint.sh` is the production counterpart: same dependency gating,
no seeding.

### 3.3 Build and install automation

`scripts/` holds three Node utilities, all build- or install-time:

- **`build-dist.js`**: assembles the deployable bundle consumed by
  `Dockerfile.prod` and `npm run start:dist`. Emits `dist/` with a pruned
  `package.dist.json` as its manifest.
- **`postinstall.js`**: runs on `npm install`; validates the workspace layout
  and prepares local tooling.
- **`generate-logo-icons.js`**: regenerates favicons, PWA icons, and
  `site.webmanifest` from `assets/logo/epms-logo.svg`. Run it after a brand
  asset change: `npm run generate:logo:icons`.

There are no publishing, changelog, or release-automation scripts in this
repository. Release mechanics live in the deployment pipeline.

### 3.4 Commit gate

`.husky/pre-commit` enforces Prettier then ESLint. Commit-message linting is
deliberately not enforced; message format is a review concern, not a
mechanical one.

---

## 4. Getting Started

New engineers should use [DOCKER_DEV_SETUP.md](DOCKER_DEV_SETUP.md), which is
the onboarding path. The short version:

```bash
cp .env.example .env
# replace every CHANGE_ME value (see below)
docker compose -f docker-compose.dev.yml up --build
```

Generate real secrets before first run:

```bash
openssl rand -base64 32   # JWT_SECRET
openssl rand -base64 32   # JWT_REFRESH_SECRET
openssl rand -hex 32      # ENCRYPTION_KEY
```

Frontend on `http://localhost:3001`, API on `http://localhost:3000/api`.

### Secrets policy

`.env` is git-ignored and must never be committed. `.env.example` contains
placeholders only; if you ever find a real value in it, treat it as a
disclosed secret and rotate it. Deployed environments take secrets from the
platform secret store, not from `.env` files.

---

## 5. Operational Reference

```bash
# Development
npm run dev                  # both tiers, host-native
npm run dev:frontend         # frontend only
npm run dev:backend          # backend only

# Database
npm run db:migrate           # apply migrations (dev)
npm run db:migrate:deploy    # apply migrations (non-interactive)
npm run db:reset             # drop and rebuild (destructive)
npm run db:studio            # Prisma Studio

# Quality
npm run lint
npm run format
npm run test
npm run test:cov
npm run test:e2e
```

Anything prefixed `test:db:*` or `test:prisma` targets `.env.test`.

---

## 6. Provenance and Licensing

EPMS is a rebranded internal derivative of **Taskosaur**, published by
NetTantra Technologies (India) Private Limited and licensed under the
**Business Source License 1.1**. The upstream license is retained verbatim in
[LICENSE.md](LICENSE.md) and its terms continue to govern this copy.

Two obligations bind day-to-day work on this repository:

1. **The license must stay with the code.** BSL 1.1 requires the license be
   conspicuously displayed on every original or modified copy. Do not remove
   or relocate `LICENSE.md`.
2. **Internal use is permitted; competing hosted offerings are not.** The
   Additional Use Grant permits production use, including hosting for internal
   organisational purposes. It does not permit offering this software to third
   parties on a hosted or embedded basis as a competing product.

Trademark rights are not granted by the license, which is why this deployment
carries its own name and brand assets rather than the upstream ones.

See [NOTICE.md](NOTICE.md) for the attribution summary and the boundary
between upstream code and internal modifications.
