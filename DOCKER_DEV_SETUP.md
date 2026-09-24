# EPMS: Engineering Onboarding

Welcome to the team. This guide takes you from a clean machine to a running
Enterprise Project Management Suite stack with seeded data. Budget about
30 minutes, most of it waiting on the first image build.

Read [README.md](README.md) alongside this if you want the architectural
picture. This document is the practical path; that one explains why the system
is shaped the way it is.

---

## Before You Start

You need:

- **Docker Engine 20.10+** and **Docker Compose V2** (`docker compose`, not the
  legacy `docker-compose` binary; the commands below assume V2).
- Roughly **8 GB** free disk for images, volumes, and dependencies.
- Ports **3000**, **3001**, **5432**, and **6379** free on the host.

You do *not* need Node, PostgreSQL, or Redis installed locally. Everything runs
in containers. Node 22 on the host is useful for editor tooling, but is not
required to run the stack.

Check your ports before you begin: a local Postgres on 5432 is the single most
common day-one blocker:

```bash
# macOS / Linux
lsof -i :3000 -i :3001 -i :5432 -i :6379

# Windows (PowerShell)
Get-NetTCPConnection -LocalPort 3000,3001,5432,6379 -ErrorAction SilentlyContinue
```

---

## Step 1: Configure Your Environment

```bash
cp .env.example .env
```

`.env.example` ships with `CHANGE_ME` placeholders, not working values. This is
deliberate: no usable secret is ever committed to this repository. Generate
your own before the first run.

```bash
openssl rand -base64 32   # → JWT_SECRET
openssl rand -base64 32   # → JWT_REFRESH_SECRET  (must differ from the above)
openssl rand -hex 32      # → ENCRYPTION_KEY
```

Paste each into `.env`. Set `POSTGRES_PASSWORD` to anything you like locally;
it only has to match itself.

Three rules about `.env`, and they are not negotiable:

1. **`.env` is git-ignored. Keep it that way.** Never commit it, never paste its
   contents into a ticket or chat.
2. **Never put a real value in `.env.example`.** If you add a new variable, add
   a placeholder there so the next engineer knows it exists.
3. **If you ever find a live secret committed anywhere, treat it as disclosed**
   and raise it with Platform Engineering the same day. It gets rotated, not
   quietly deleted.

`ENCRYPTION_KEY` protects stored third-party credentials. If you rotate it on
an existing database, everything encrypted under the old key becomes
unreadable: fine locally, a production incident anywhere else.

---

## Step 2: Bring Up the Stack

```bash
docker compose -f docker-compose.dev.yml up --build
```

The first run takes a while: it builds the dev image, installs the whole
workspace, and generates the Prisma client. Later runs reuse the cache. Drop
`--build` once the image exists, and add `-d` to detach.

You do not need to run migrations or seeds by hand. The entrypoint
(`docker/entrypoint-dev.sh`) does this on every start:

1. Waits for PostgreSQL to accept connections
2. Waits for Redis
3. Generates the Prisma client
4. Applies migrations
5. Seeds reference data
6. Seeds the admin user
7. Starts frontend and backend together

Steps 4-6 are **idempotent** and tolerate failure; restarting against an
already-seeded database is a no-op, not an error. Warnings like
`⚠️ Seeding failed or data already exists` on a restart are expected.

Steps 1-2 are *not* tolerant. If Postgres or Redis never becomes reachable, the
container exits rather than looping.

You are up when you see the dev servers start. Then:

| Surface | URL |
|---|---|
| Frontend | http://localhost:3001 |
| Backend API | http://localhost:3000/api |
| API reference (Swagger) | http://localhost:3000/api/docs |
| Health probe | http://localhost:3000/api/health |

Sign in with the seeded admin account: ask your onboarding buddy for the
current seed credentials; they live in `backend/src/seeder/`, not in this file.

---

## Step 3: Know Your Daily Loop

Source is bind-mounted, so **code changes need no rebuild**. NestJS restarts on
save; Next.js Fast Refreshes.

The exception is dependencies. `node_modules` directories are masked by
anonymous volumes so host and container trees never mix, which means a
`package.json` change requires a rebuild, not a restart:

```bash
docker compose -f docker-compose.dev.yml up --build
```

If something behaves impossibly after a dependency change, you almost certainly
skipped this.

### Commands you will use constantly

```bash
# Logs
docker compose -f docker-compose.dev.yml logs -f app

# Shell into the app container
docker compose -f docker-compose.dev.yml exec app sh

# Stop / start
docker compose -f docker-compose.dev.yml stop
docker compose -f docker-compose.dev.yml up -d

# Restart one service
docker compose -f docker-compose.dev.yml restart app
```

### Database work

Run these inside the container, where `DATABASE_URL` resolves correctly:

```bash
docker compose -f docker-compose.dev.yml exec app npm run db:studio    # Prisma Studio
docker compose -f docker-compose.dev.yml exec app npm run db:migrate   # apply migrations
docker compose -f docker-compose.dev.yml exec app npm run db:seed      # reseed
docker compose -f docker-compose.dev.yml exec app npm run db:reset     # DESTRUCTIVE: drop + rebuild
```

`db:reset` drops your local database. That is fine locally and never something
you run against a shared environment.

### Tests

```bash
docker compose -f docker-compose.dev.yml exec app npm run test:backend
docker compose -f docker-compose.dev.yml exec app npm run test:frontend
docker compose -f docker-compose.dev.yml exec app npm run test:e2e
```

Suites prefixed `test:db:*` target `.env.test`, keeping the test database
separate from your development one. Use them when a test needs to mutate data
destructively.

---

## Step 4: Before You Open a Pull Request

A pre-commit hook runs Prettier, then ESLint. If it blocks you:

```bash
npm run format   # auto-fix formatting
npm run lint     # see remaining problems
```

Fix and re-stage. Do not bypass the hook with `--no-verify`; CI runs the same
checks and will fail the branch anyway.

Commit *message* format is not machine-enforced. Write a clear subject line
explaining why the change exists; reviewers care about that more than a prefix.

CI (`.github/workflows/enterprise-build.yml`) then runs static analysis,
brings up this same `docker-compose.dev.yml` stack for isolated backend and
frontend integration legs, and verifies the production image still builds. If
it passes locally and fails in CI, the difference is almost always a missing
`.env.example` placeholder for a variable you added.

---

## Troubleshooting

**Port already in use.** Something on the host holds 3000, 3001, 5432, or 6379,
usually a local Postgres or a previous stack. Stop it, or remap the host side
in `docker-compose.dev.yml` (`"3100:3000"` changes the host port only).

**Backend exits immediately on start.** Read the logs: if it never reported
`✅ PostgreSQL is ready!`, the database did not come up. Check it directly:

```bash
docker compose -f docker-compose.dev.yml ps postgres
docker compose -f docker-compose.dev.yml logs postgres
```

**Prisma client errors after pulling `main`.** Someone changed the schema.
Regenerate:

```bash
docker compose -f docker-compose.dev.yml exec app npm run db:generate
```

**Frontend loads but every API call fails.** `NEXT_PUBLIC_API_BASE_URL` is
inlined at build time, not read at runtime. If you changed it, rebuild.

**Authentication fails after editing `.env`.** Changing `JWT_SECRET` invalidates
every issued token. Clear site data and sign in again.

**Nothing makes sense any more.** Full reset (destroys local data, keeps images):

```bash
docker compose -f docker-compose.dev.yml down -v
docker compose -f docker-compose.dev.yml up --build
```

---

## Where to Go Next

- [README.md](README.md): architecture, DevOps strategy, infrastructure
- `backend/src/modules/`: one directory per domain concern; start with `tasks`
- `frontend/src/components/`: feature-grouped UI over Radix primitives
- `backend/prisma/schema.prisma`: the data model, and the fastest way to
  understand the domain
- [NOTICE.md](NOTICE.md): licensing and attribution; read this before
  redistributing anything

Questions that this guide did not answer belong in the Platform Engineering
channel. If you hit something that should have been here, add it; onboarding
docs are maintained by the people most recently onboarded.
