# HelpDesk

An IT support ticketing system with **SLA tracking**. Requesters log tickets; agents work them through a workflow; the system tracks every deadline and reports compliance.

A second full stack project, deliberately built on a different stack from [CampusTrack](../campustrack) to show range: **Django · GraphQL (Strawberry) · Vue 3 + TypeScript · PostgreSQL · MongoDB · Docker · GitHub Actions + Azure DevOps (deploys to Azure Container Apps)**.

## Features

- **Roles**: requesters see and comment on only their own tickets; agents see everything, move tickets through `open → in progress → resolved`, and view the SLA dashboard.
- **SLA engine**: deadline set at creation from priority (urgent 4h, high 8h, medium 24h, low 72h). A ticket breaches if it is resolved late or still open past its deadline. Dashboard shows open, breached, resolved and compliance %.
- **Audit trail** in MongoDB: append-only record of who did what to each ticket (agents only).
- **GraphQL API** with depth limiting, introspection disabled in production, and N+1-safe queries.

## Architecture

```
Browser ──> nginx (Vue SPA) ──/graphql──> Django + Strawberry ──> PostgreSQL (tickets, users)
                                                   │
                                                   └──────────────> MongoDB (audit log)
```

| Layer | Choices |
|---|---|
| Frontend | Vue 3 (`<script setup>`), TypeScript, Pinia, Tailwind CSS, Vite, Vitest + Vue Test Utils |
| Backend | Django 6, Strawberry GraphQL, JWT (PyJWT), Django ORM + migrations, WhiteNoise, Gunicorn |
| Data | PostgreSQL 16 (relational data, indexes); MongoDB 7 (audit documents) |
| DevOps | Docker (non-root), docker-compose with healthchecks, GitHub Actions, Azure DevOps → Azure Container Apps |

Design notes are in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Run it

```bash
cp .env.example .env     # set POSTGRES_PASSWORD and DJANGO_SECRET_KEY
docker compose up --build

# create the first agent (staff user); requesters can be added in the admin site
docker compose exec backend python manage.py createsuperuser
```

Open http://localhost:8080. Add requester accounts at http://localhost:8080/admin/ (leave "Staff status" unticked). Staff users are agents.

### Run without Docker

```bash
cd backend && python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
export DJANGO_DEBUG=1            # SQLite + in-memory audit log, GraphiQL enabled at /graphql
python manage.py migrate && python manage.py createsuperuser
python manage.py runserver

cd frontend && npm install && npm run dev     # proxies /graphql to localhost:8000
```

Set `POSTGRES_HOST` (and `POSTGRES_*`) to use PostgreSQL, and `MONGO_URL` to use MongoDB.

## GraphQL examples

```graphql
mutation { login(username: "alice", password: "…") { token isAgent } }

mutation { createTicket(title: "VPN down", description: "Cannot connect", priority: "urgent") {
  id slaDueAt slaBreached } }

query { tickets(status: "open", search: "vpn", page: 1, pageSize: 10) {
  total items { id title priority status slaBreached comments { author body } } } }

# agents only
mutation { updateTicketStatus(id: "1", status: "resolved") { status resolvedAt } }
query { slaReport { open breachedOpen resolved compliancePercent } }
query { auditTrail(ticketId: "1") { actor action at } }
```

Send the token as `Authorization: Bearer <token>`.

## Tests

```bash
cd backend && pytest        # 31 tests: SLA rules, access control, workflow, audit log, query counts
cd frontend && npm test     # 18 tests: GraphQL client, Pinia stores, components
```

Backend tests use SQLite by default and also pass against PostgreSQL (set `POSTGRES_*`; CI does this with a Postgres service container). The MongoDB audit log is tested with `mongomock`; the same class talks to real MongoDB in Docker.

## Secure coding practices

- Object-level authorisation in one place (`visible_tickets`): requesters can never read or comment on other people's tickets, and "not yours" looks identical to "does not exist".
- Role checks on agent-only operations via GraphQL permission classes; covered by tests.
- JWT bearer tokens (signed, expiring); login does not reveal whether the username or password was wrong.
- Query **depth limit** and **introspection disabled** outside `DJANGO_DEBUG`; page size capped at 100.
- Secrets from environment only. The app refuses to start without `DJANGO_SECRET_KEY` unless `DJANGO_DEBUG=1`.
- ORM-only database access (no string-built SQL); status transitions and rows locked with `select_for_update` to avoid lost updates.
- Containers run as non-root; nginx adds security headers; CORS limited to configured origins.
- Trade-off: the SPA keeps the token in `sessionStorage`. HttpOnly cookies plus CSRF would be the next hardening step.

## Performance

List queries use `select_related`/`prefetch_related`, and a test asserts the query count stays constant as tickets grow (no N+1). Indexes: `(status, sla_due_at)` for SLA queries and `(requester, -created_at)` for each user's ticket list.

## Delivery

- CI: [GitHub Actions](.github/workflows/ci.yml) runs tests against Postgres, checks for missing migrations, type-checks and builds the frontend, builds both images.
- CD: [azure-pipelines.yml](azure-pipelines.yml) tests, builds images in Azure Container Registry, then updates two Azure Container Apps behind an approval-gated `production` environment.
- Agile artefacts and runbook: [docs/AGILE.md](docs/AGILE.md), [docs/RUNBOOK.md](docs/RUNBOOK.md).

### Deploy to Azure (outline)

The pipeline is a template and has not been run against a real subscription.

1. Create a resource group, an Azure Container Registry, a Container Apps environment, managed PostgreSQL, and a MongoDB-compatible store (e.g. Cosmos DB for MongoDB).
2. Create two Container Apps (`backend`, `web`) with secrets and environment variables from `.env.example` plus `POSTGRES_HOST`, `MONGO_URL`, `DJANGO_ALLOWED_HOSTS`, `CORS_ORIGINS`.
3. In Azure DevOps create a service connection named `azureServiceConnection` and a variable group `helpdesk-azure` with `ACR_NAME`, `RESOURCE_GROUP`, `BACKEND_APP`, `WEB_APP`.
4. Push to `main`.

## Roadmap

- Ticket assignment and escalation rules (auto-escalate when 80% of SLA time has elapsed)
- Email notifications and attachments
- GraphQL subscriptions for live updates
- Pagination controls in the UI
