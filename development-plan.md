# Status Page Platform — Phased Development Plan

> Project: 315-status-page-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files into a concrete, phased build. The schema anchors on **data-model-suggestion-1 (Entity-Centric Normalized Relational)** as the source of truth, with a selective adoption of the **suggestion-2 hybrid JSONB pattern** for the two genuinely polymorphic surfaces: per-type monitor configuration and per-provider notification-channel configuration. The event-sourcing ideas from suggestion-3 are deferred to an append-only `audit_events` table rather than a full event store, to keep the MVP tractable while preserving SLA-grade audit trails.

---

## Core Requirements (Synthesis)

**What it does:** An AI-native, open-source status page platform that closes the gap between monitoring signals and customer-facing incident communication. Teams model their services as components, run built-in uptime checks, manage incidents through a lifecycle, and notify subscribers — with AI drafting incident updates and post-mortems from monitoring context.

**Who uses it:** SRE/DevOps teams managing customer-facing SLAs; engineering teams at SaaS companies; customer-success teams communicating during incidents; enterprise IT running internal status pages.

**Key differentiators:** (1) built-in monitoring → status-page automation with no manual lag; (2) AI-written incident updates from monitoring signal context; (3) AI post-mortem narrative generation; (4) fully open-source under a permissive licence (MIT), avoiding the AGPL constraint that limits OpenStatus reuse.

**MVP scope (features.md "Must-have"):** component health model with configurable statuses; incident lifecycle with templated messaging; subscriber notifications via email + webhook; custom domain with automatic SSL; REST API with API-key auth and OpenAPI 3.1 spec; inbound webhook receiver to auto-update component status.

**v1.1 scope ("Should-have"):** built-in uptime monitoring (HTTP/TCP/DNS/SSL); scheduled maintenance with pre-notification; Slack + Teams notifications; private pages with SSO (SAML 2.0 / OIDC); AI-generated incident update drafts; historical uptime charts.

**Backlog ("Nice-to-have"):** on-call scheduling; monitoring-as-code (YAML/Terraform); AI post-mortem workflow; subscriber impact segmentation; third-party status aggregation; multi-language pages.

**Deployment model:** Self-hosted-first (Docker Compose, OneUptime/Uptime Kuma precedent) with a managed-SaaS-compatible multi-tenant architecture. Public status pages must render fast and survive traffic spikes.

**Integration surface:** outbound webhooks (Standard Webhooks spec, HMAC-SHA256), inbound monitoring webhooks, Atom/RSS feeds, SMTP email (DMARC/DKIM/SPF), Slack/Teams, LLM provider (Anthropic), Prometheus/OTLP ingest (later), SAML/OIDC IdPs.

**Standards to implement:** OpenAPI 3.1, RFC 7807 (problem+json), RFC 3339 timestamps, RFC 4287 (Atom), Standard Webhooks + HMAC-SHA256 (FIPS 198-1), W3C SSE for live updates, OAuth 2.0 / OIDC + SAML 2.0 for SSO, OWASP API Security Top 10, ITIL 4 severity taxonomy.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **Python 3.12** | The differentiating value is AI (incident drafts, post-mortems, impact classification). Python has the strongest LLM tooling and the cleanest async story for the I/O-bound workloads here (HTTP/DNS/SSL checks, webhook fan-out, SMTP). Keeps the whole stack in one language. |
| API framework | **FastAPI** | First-class async, Pydantic v2 models, and automatic **OpenAPI 3.1** generation — a day-one requirement from README/standards. Native dependency-injection fits API-key auth and tenant scoping. |
| Data validation | **Pydantic v2** | Request/response schemas, config models, and webhook payload validation. Drives the OpenAPI contract and JSON Schema (Draft 2020-12) output. |
| Database | **PostgreSQL 16** | Suggestion-1 relies on UUID PKs, `CHECK` constraints, array columns, partial indexes, `JSONB`, and **time-range partitioning** for `monitor_checks`. SQLite cannot do partitioning or partial indexes well. Postgres is the production target for every comparable platform. |
| ORM / migrations | **SQLAlchemy 2.0 (async) + Alembic** | Async ORM matches FastAPI; Alembic gives versioned, reviewable migrations including partition setup. |
| Task queue | **Celery + Redis** | Async workloads — monitor scheduling, webhook delivery with retries, email fan-out, LLM calls — must not block the API. Celery Beat schedules monitor runs; Redis is broker + result backend + cache for public-page reads. |
| Cache / pub-sub | **Redis 7** | Caches the public-page projection (read-heavy, spike-prone) and backs **SSE** live updates via pub/sub. |
| LLM provider | **Anthropic Claude (via SDK), provider-abstracted** | The README positions AI as first-class. An `LLMProvider` interface keeps the call site provider-neutral; Claude is the default. Prompt caching reduces cost on repeated incident context. |
| Frontend (admin) | **Next.js 16 (App Router) + shadcn/ui + Tailwind** | Admin dashboard (components, incidents, monitors, subscribers). Server Components keep the bundle small; shadcn gives accessible primitives. |
| Frontend (public page) | **Server-rendered Next.js route + SSE** | Public status pages must load fast and update live without polling (W3C SSE). Cacheable at the edge; falls back to static render. |
| Email delivery | **SMTP via `aiosmtplib`, pluggable provider** | Self-hosted users bring their own SMTP/SES. DKIM signing via `dkimpy`. Deliverability guidance (DMARC/DKIM/SPF) documented. |
| Monitoring checks | **`httpx` (HTTP), `dnspython` (DNS), stdlib `ssl`+`socket` (TCP/SSL)** | Pure-Python, async-friendly probes; no external monitoring dependency — this *is* the built-in monitor. |
| Webhooks (outbound) | **Standard Webhooks spec + `hmac` (SHA256)** | Conform to the cross-vendor spec from day one (`webhook-id`, `webhook-timestamp`, `webhook-signature`). |
| Auth (API) | **API keys (hashed, prefixed) + scopes** | MVP API auth. Keys stored as SHA-256 hash + display prefix; never recoverable. |
| Auth (SSO, v1.1) | **`python3-saml` (SAML 2.0) + `authlib` (OIDC)** | Enterprise private-page access. |
| Testing | **pytest + pytest-asyncio + httpx.AsyncClient + testcontainers** | Unit + integration; testcontainers spins real Postgres/Redis for integration tests; `respx` mocks outbound HTTP/LLM. |
| Code quality | **ruff (lint+format) + mypy (strict) + bandit** | Bandit covers OWASP-relevant static checks on the API. |
| Package manager | **uv** | Fast, lockfile-based dependency management for the Python service. |
| Containerisation | **Docker + docker-compose** | Self-hosted distribution: `api`, `worker`, `beat`, `postgres`, `redis`, `web`. |
| CI | **GitHub Actions** | Lint, type-check, test matrix, Docker build, OpenAPI spec diff check. |

### Project Structure

```
status-page-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile                      # multi-stage build for api/worker
├── docker-compose.yml
├── docker-compose.dev.yml
├── alembic.ini
├── .github/workflows/ci.yml
├── openapi/
│   └── openapi.json                # generated, committed, diff-checked in CI
├── migrations/                     # Alembic versions
│   └── versions/
├── src/
│   └── statuspage/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── engine.py           # async engine/session
│       │   ├── models.py           # SQLAlchemy ORM (suggestion-1 schema)
│       │   └── partitions.py       # monitor_checks partition management
│       ├── schemas/                # Pydantic request/response models
│       │   ├── component.py
│       │   ├── incident.py
│       │   ├── maintenance.py
│       │   ├── monitor.py
│       │   ├── subscriber.py
│       │   └── problem.py          # RFC 7807 ProblemDetail
│       ├── api/
│       │   ├── deps.py             # auth, tenant scoping, db session
│       │   ├── errors.py           # RFC 7807 exception handlers
│       │   └── v1/
│       │       ├── components.py
│       │       ├── incidents.py
│       │       ├── maintenance.py
│       │       ├── monitors.py
│       │       ├── subscribers.py
│       │       ├── pages.py
│       │       ├── webhooks_in.py  # inbound monitoring receiver
│       │       └── public.py       # public page JSON + SSE + Atom feed
│       ├── services/               # business logic (no framework deps)
│       │   ├── incidents.py
│       │   ├── components.py
│       │   ├── subscribers.py
│       │   ├── monitors.py
│       │   ├── projection.py       # public-page read model + cache
│       │   └── uptime.py
│       ├── integrations/
│       │   ├── webhook_out.py      # Standard Webhooks delivery
│       │   ├── email.py            # SMTP + DKIM
│       │   ├── slack.py
│       │   └── teams.py
│       ├── ai/
│       │   ├── provider.py         # LLMProvider interface + Anthropic impl
│       │   ├── prompts.py          # prompt templates
│       │   ├── incident_drafts.py
│       │   └── postmortem.py
│       ├── monitoring/
│       │   ├── probes/             # http.py, tcp.py, dns.py, ssl.py
│       │   ├── scheduler.py        # Celery Beat dispatch
│       │   └── evaluator.py        # check → component status transition
│       ├── auth/
│       │   ├── apikeys.py
│       │   ├── saml.py             # v1.1
│       │   └── oidc.py             # v1.1
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks.py            # deliver_webhook, send_email, run_check, ai_draft
│       └── feeds/atom.py           # RFC 4287 Atom feed
├── web/                            # Next.js admin + public page
│   ├── package.json
│   └── app/
│       ├── (admin)/...
│       └── status/[slug]/...
└── tests/
    ├── conftest.py                 # db/redis fixtures (testcontainers)
    ├── fixtures/                   # sample payloads, SARIF-style golden files
    ├── unit/
    ├── integration/
    └── e2e/
```

Grouping is by concern (api / services / integrations / ai / monitoring), so each phase adds modules without restructuring.

---

## Phase 1: Foundation & Data Model

### Purpose
Stand up the runnable skeleton: project scaffolding, configuration, the full Postgres schema (suggestion-1), async DB access, the FastAPI app with RFC 7807 error handling, health checks, and the Docker Compose stack. After this phase the service boots, migrations apply cleanly, and `/healthz` and `/openapi.json` respond — every later phase plugs into this base.

### Tasks

#### 1.1 — Project scaffolding & configuration

**What**: Create the Python package, dependency manifest, tooling config, and an env-driven settings model.

**Design**:
- `pyproject.toml` with deps (fastapi, uvicorn, pydantic, pydantic-settings, sqlalchemy[asyncio], asyncpg, alembic, celery, redis, httpx, dnspython, aiosmtplib, anthropic) and dev deps (pytest, pytest-asyncio, respx, testcontainers, ruff, mypy, bandit).
- `config.py`:
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="SP_", env_file=".env")
    database_url: PostgresDsn
    redis_url: RedisDsn
    secret_key: SecretStr                       # signs unsubscribe/confirm tokens
    public_base_url: AnyHttpUrl
    smtp_host: str | None = None
    smtp_port: int = 587
    anthropic_api_key: SecretStr | None = None
    ai_enabled: bool = False
    default_check_region: str = "local"
    log_level: str = "INFO"
```
- ruff config (line length 100, select E/F/I/UP/B), mypy strict, bandit baseline.

**Testing**:
- `Unit: Settings loads from env vars → all fields populated, defaults applied.`
- `Unit: missing required DATABASE_URL → ValidationError naming the field.`
- `Unit: SecretStr fields → repr() does not leak the secret.`

#### 1.2 — Database schema & migrations

**What**: Implement the full suggestion-1 schema as SQLAlchemy ORM models plus an initial Alembic migration, with `monitor_checks` range-partitioned by `checked_at`.

**Design**:
- ORM models for all 21 tables from data-model-suggestion-1: `organisations, org_members, users, status_pages, component_groups, components, incidents, incident_components, incident_updates, incident_templates, scheduled_maintenances, maintenance_components, maintenance_updates, monitors, monitor_checks, subscribers, notifications, post_mortems, ai_draft_queue, webhooks, api_keys`.
- Adopt the suggestion-2 hybrid pattern on exactly two columns to avoid wide type-specific columns:
  - `monitors.config JSONB NOT NULL DEFAULT '{}'` holds type-specific fields (http method/headers/expected_status; dns record_type/expected_value; ssl warn_days). Keep `monitor_type`, `target`, `interval_seconds`, `timeout_seconds`, `regions`, `is_active`, `auto_update_component`, `consecutive_failures_threshold` as typed columns.
  - `subscribers.channel_config JSONB` and `webhooks`/notification channel specifics as JSONB where provider-variable.
- CHECK constraints exactly as specified (component status enum, incident status/severity/impact enums, etc.).
- Partial indexes per suggestion-1 (`idx_incidents_active`, `idx_notifications_status`, `idx_subscribers_email`, `idx_monitors_active`).
- `monitor_checks` declared `PARTITION BY RANGE (checked_at)`; `db/partitions.py` provides `ensure_partition(month: date)` creating `monitor_checks_YYYY_MM`.
- Append-only `audit_events(id, org_id, actor_id, entity_type, entity_id, action, payload JSONB, created_at)` for SLA-grade audit (the deferred suggestion-3 idea).
- All timestamps `TIMESTAMPTZ` (RFC 3339 on the wire).

**Testing**:
- `Integration (real PG via testcontainers): alembic upgrade head → all 22 tables exist; \d monitor_checks shows partitioned.`
- `Integration: alembic downgrade base then upgrade head → idempotent, no errors.`
- `Integration: insert component with status 'banana' → CHECK constraint violation.`
- `Integration: delete status_page → cascades to components, incidents, subscribers (FK ON DELETE CASCADE).`
- `Unit: ensure_partition(2026-06) → emits CREATE TABLE ... PARTITION OF with correct bounds.`

#### 1.3 — App factory, DB session, RFC 7807 errors, health

**What**: FastAPI app factory with async session dependency, RFC 7807 error handling, request-id middleware, and health endpoints.

**Design**:
- `db/engine.py`: `create_async_engine`, `async_sessionmaker`; `get_session()` dependency yields a session and rolls back on exception.
- `schemas/problem.py`:
```python
class ProblemDetail(BaseModel):     # RFC 7807, media type application/problem+json
    type: str = "about:blank"
    title: str
    status: int
    detail: str | None = None
    instance: str | None = None
    errors: list[dict] | None = None   # field-level validation detail
```
- `api/errors.py`: exception handlers mapping `RequestValidationError` → 422 problem+json, `HTTPException` → problem+json, `IntegrityError` → 409, catch-all → 500 (no stack leak; logs with request id). OWASP API4/API8 alignment: no internal detail in 500 bodies.
- `/healthz` (liveness) and `/readyz` (checks DB + Redis).
- Middleware injects `X-Request-ID` (incoming or generated) into logs and `instance` field.

**Testing**:
- `Integration: GET /healthz → 200 {"status":"ok"}.`
- `Integration: GET /readyz with DB down → 503, problem+json.`
- `Integration: POST malformed JSON to any endpoint → 422 application/problem+json with errors[] naming fields.`
- `Integration: handler raising IntegrityError → 409 problem+json, no SQL text in body.`
- `Integration: every response carries X-Request-ID.`

#### 1.4 — Docker Compose stack

**What**: Multi-stage Dockerfile and compose files for the full local stack.

**Design**:
- Services: `postgres:16`, `redis:7`, `api` (uvicorn), `worker` (celery), `beat` (celery beat), `web` (Next.js). Healthchecks gate startup order.
- `docker-compose.dev.yml` mounts source for hot reload; runs `alembic upgrade head` on api start via entrypoint.

**Testing**:
- `E2E: docker compose up → api /readyz returns 200 within 60s; migrations applied automatically.`
- `E2E: docker build (prod target) → image builds, runs uvicorn as non-root.`

---

## Phase 2: Tenancy, Auth & API Keys

### Purpose
Introduce the ownership chain (organisation → status page) and the API-key authentication + scoping that every management endpoint depends on. After this phase, callers authenticate with scoped keys and all data access is tenant-isolated — a prerequisite for OWASP API1 (broken object-level authorisation) compliance.

### Tasks

#### 2.1 — Organisations, users, status pages CRUD

**What**: Service + endpoints to create organisations, status pages (with slug uniqueness per org), and manage members.

**Design**:
- `services/components.py`-style service modules return domain results, raise typed errors (`NotFound`, `Conflict`, `Forbidden`).
- Endpoints: `POST/GET/PATCH/DELETE /v1/pages`, `GET /v1/pages/{id}`. Slug auto-generated from name, unique per org.
- Pydantic schemas separate `Create`, `Update`, `Read` (no internal fields leaked — OWASP API3 excessive-data-exposure).

**Testing**:
- `Integration: create page with duplicate slug in same org → 409 problem+json.`
- `Integration: create page → Read schema omits org internal columns.`
- `Unit: slugify("My API!") → "my-api".`

#### 2.2 — API-key issuance & verification

**What**: Generate, hash, scope, and verify API keys.

**Design**:
```python
# format: sp_live_<prefix8>.<secret32>   shown once at creation
def issue_key(org_id, user_id, name, scopes) -> tuple[str, ApiKey]:
    raw = "sp_live_" + token_urlsafe(8) + "." + token_urlsafe(32)
    prefix = raw[:16]
    key_hash = sha256(raw.encode()).hexdigest()
    # store key_hash (UNIQUE), key_prefix, scopes; return raw once
SCOPES = {"pages:read","pages:write","incidents:write",
          "components:write","subscribers:write","monitors:write"}
```
- `api/deps.py::require_scope(scope)` dependency: reads `Authorization: Bearer sp_live_...`, hashes, looks up active key, sets `request.state.org_id`, checks scope, updates `last_used_at` async.

**Testing**:
- `Unit: issue_key → raw verifies against stored hash; prefix matches first 16 chars.`
- `Integration: request with valid key + correct scope → 200.`
- `Integration: valid key missing scope → 403 problem+json.`
- `Integration: revoked (is_active=false) key → 401.`
- `Integration: key for org A cannot read org B's page → 404 (not 403, to avoid existence leak).`

---

## Phase 3: Components & Incident Lifecycle (Core Value)

### Purpose
The heart of the product: model component health and run incidents through the ITIL-aligned lifecycle (investigating → identified → monitoring → resolved) with a timeline of updates. After this phase the platform can manage real incidents via the API — the minimum that makes it a status page.

### Tasks

#### 3.1 — Components & component groups

**What**: CRUD for component groups and components with status, ordering, visibility.

**Design**:
- Endpoints under `/v1/pages/{page_id}/components` and `/component-groups`.
- `ComponentStatus = Literal["operational","degraded_performance","partial_outage","major_outage","under_maintenance"]`.
- Status changes write an `audit_events` row and publish a `component.status_changed` event (consumed in Phase 5 by webhooks/notifications and projection cache invalidation).

**Testing**:
- `Integration: create component default status → 'operational'.`
- `Integration: PATCH status operational→major_outage → audit_events row written, event published.`
- `Integration: list components → ordered by group sort_order NULLS LAST, then component sort_order.`

#### 3.2 — Incident lifecycle & updates

**What**: Create incidents, attach affected components with per-component status, append updates, and transition status with timestamping.

**Design**:
- State machine: `investigating → identified → monitoring → resolved` (forward + allowed back-transitions investigating↔identified↔monitoring; resolved is terminal except reopen→investigating). `postmortem` is a post-resolution flag, set in Phase 8.
- Transition side effects: setting `identified`/`monitoring`/`resolved` stamps the matching `*_at` column; resolving an incident reverts attached components to `operational` unless overridden.
```python
def add_update(incident_id, status, body, *, is_ai_generated=False, model_version=None, actor):
    # validate transition; insert incident_updates row; update incident.status + *_at;
    # apply incident_components statuses to components; emit incident.updated event
```
- `incident_components` carries per-component status (API: major_outage, Dashboard: degraded) per suggestion-1 decision #2.

**Testing**:
- `Unit: transition resolved→investigating allowed (reopen); investigating→resolved sets resolved_at.`
- `Unit: illegal transition monitoring→identified... (define allowed set) → ValueError.`
- `Integration: create incident affecting 2 components → both components reflect their incident_components status.`
- `Integration: resolve incident → attached components return to 'operational'; resolved_at set.`
- `Integration: each add_update → new incident_updates row; timeline ordered DESC.`

#### 3.3 — Incident templates

**What**: Reusable templated incident messaging per org.

**Design**:
- `incident_templates(title_template, body_template, severity, component_ids[])`. Templates use `{component}`, `{eta}` placeholders rendered server-side at apply time.
- `POST /v1/incidents?from_template={id}` pre-fills title/body/severity/components.

**Testing**:
- `Unit: render template "{component} degraded" with component="API" → "API degraded".`
- `Integration: create incident from template → fields pre-populated, components attached.`
- `Integration: duplicate template name in org → 409.`

---

## Phase 4: Public Status Page, Feeds & Live Updates

### Purpose
Expose the customer-facing surface: a fast, cacheable public JSON projection, an RFC 4287 Atom feed, an SSE stream for live updates, and the server-rendered public page. This is what end users actually see; it must survive traffic spikes (incident.io noted status-page traffic spikes during outages).

### Tasks

#### 4.1 — Public-page projection & cache

**What**: A read model assembling current component statuses, active incidents, recent history, and upcoming maintenance into a single cached JSON document per page.

**Design**:
- `services/projection.py::build_public_view(page_id) -> PublicView` runs the suggestion-1 "current status" and "active incidents" queries, plus 90-day incident history and overall status rollup (`operational` if all green else worst component status).
- Cached in Redis under `page:{slug}:view` with a short TTL; invalidated on `component.status_changed` / `incident.updated` / `maintenance.*` events (pub/sub).
- `GET /status/{slug}` (public, no auth) returns the cached `PublicView`; `ETag` + `Cache-Control`.

**Testing**:
- `Integration: GET /status/{slug} → overall_status reflects worst component.`
- `Integration: change a component status → cache invalidated; next read shows new status.`
- `Integration: 1000 concurrent reads → all served from cache (DB query count == 0 after warm).`
- `Integration: unknown slug → 404 problem+json.`

#### 4.2 — Atom feed (RFC 4287)

**What**: `GET /status/{slug}/feed.atom` returning incident history as a valid Atom feed.

**Design**:
- `feeds/atom.py` builds `<feed>` with one `<entry>` per incident (and per major update), RFC 3339 `<updated>`, `<id>` as tag URI. Content type `application/atom+xml`.

**Testing**:
- `Unit: feed validates against Atom schema (committed fixture validator).`
- `Integration: feed contains entry per incident; timestamps RFC 3339.`

#### 4.3 — SSE live updates (W3C EventSource)

**What**: `GET /status/{slug}/events` streaming status changes over Server-Sent Events.

**Design**:
- Subscribes to the page's Redis pub/sub channel; emits `event: status_change\ndata: {json}\n\n`. Heartbeat comment every 15s. Auto-reconnect honoured via `Last-Event-ID`.

**Testing**:
- `Integration: open SSE, change component status → client receives status_change event within 1s.`
- `Integration: idle stream → heartbeat received every ~15s.`

#### 4.4 — Public page UI (Next.js)

**What**: Server-rendered `status/[slug]` route with live SSE updates and component/incident timeline display.

**Design**:
- Server Component fetches `PublicView`; client island subscribes to SSE and patches the DOM. shadcn components for status badges; respects `branding_accent_color`, `custom_css`, `locale`.

**Testing**:
- `E2E (Playwright): visit /status/demo → components and overall banner render; trigger incident via API → page updates without reload.`

---

## Phase 5: Subscribers, Notifications & Outbound Webhooks

### Purpose
Close the communication loop: let end users subscribe (email/webhook), deliver notifications reliably on incident and maintenance events, and emit Standard-Webhooks-compliant outbound webhooks. This is a table-stakes feature absent from Uptime Kuma and a core MVP requirement.

### Tasks

#### 5.1 — Subscriber management & confirmation

**What**: Subscribe/confirm/unsubscribe flows with component-level subscription filtering.

**Design**:
- `POST /status/{slug}/subscribers` (public) creates a `pending` subscriber, sends a double-opt-in confirmation email with a signed token; `GET /subscribe/confirm/{token}` confirms. `unsubscribe_token` in every email footer.
- Component-level subscriptions via `component_ids[]` (suggestion-1 decision #5).

**Testing**:
- `Integration: subscribe email → subscriber is_confirmed=false, confirmation task enqueued.`
- `Integration: confirm with valid token → is_confirmed=true; invalid/expired token → 400.`
- `Unit: token signed with secret_key; tampering → verification fails.`

#### 5.2 — Notification fan-out (email + webhook)

**What**: On incident/maintenance events, create `notifications` rows and dispatch via Celery, filtered by component subscription and tracked through delivery states.

**Design**:
- Event consumer enqueues `send_notification(notification_id)` per matching confirmed subscriber. Only notify subscribers whose `component_ids` intersect the affected components (or who subscribe to all).
- `notifications.status` state machine: `pending → sent → delivered|failed|bounced`; retries with exponential backoff (Celery `autoretry_for`, max 5).
- Email: `integrations/email.py` renders templated HTML/text, DKIM-signs, sends via SMTP.

**Testing**:
- `Integration (mocked SMTP): incident created affecting API → only subscribers to API or all-components get notifications rows.`
- `Integration: SMTP transient failure → status 'failed', retried; permanent 5xx → 'bounced'.`
- `Integration: notification lifecycle pending→sent→delivered recorded with timestamps.`

#### 5.3 — Outbound webhooks (Standard Webhooks spec)

**What**: Deliver HMAC-signed webhooks to configured `webhooks` endpoints on subscribed events.

**Design**:
- `integrations/webhook_out.py`: payload `{ "type": "incident.created", "data": {...}, "timestamp": rfc3339 }`. Headers `webhook-id`, `webhook-timestamp`, `webhook-signature: v1,<base64(hmac_sha256(secret, f"{id}.{ts}.{body}"))>`.
- Retries on non-2xx with backoff; delivery attempts logged to `audit_events`.

**Testing**:
- `Unit: signature matches Standard Webhooks reference vector for known secret/payload.`
- `Integration (respx): event fires → POST to webhook.url with correct headers; verifies signature server-side.`
- `Integration: endpoint returns 500 → retried; after max attempts → recorded failed.`
- `Integration: webhook subscribed only to 'incident.resolved' → not called on 'incident.created'.`

---

## Phase 6: Inbound Webhook Receiver & Maintenance

### Purpose
Deliver the remaining MVP items: an inbound webhook receiver that lets external monitoring tools auto-update component status (the Instatus-style ingestion that removes manual lag), plus scheduled maintenance windows with subscriber pre-notification.

### Tasks

#### 6.1 — Inbound monitoring webhook receiver

**What**: Authenticated endpoint that maps inbound monitoring payloads to component status changes and optional auto-incident creation.

**Design**:
- `POST /v1/ingest/{page_id}` authenticated by API key or a per-page ingest token. Body: `{ "component": "<id-or-name>", "status": "<ComponentStatus>", "message": "...", "auto_incident": true }`.
- Maps to `set_component_status`; if `auto_incident` and status is an outage, opens an incident (idempotent: reuses an open incident for the same component within a debounce window).
- Adapters for common shapes (generic, Prometheus Alertmanager) behind a `parse_payload(source)` dispatcher.

**Testing**:
- `Integration: ingest {component:API,status:major_outage,auto_incident:true} → component status set, incident opened.`
- `Integration: second ingest within debounce → no duplicate incident; existing incident updated.`
- `Integration: Alertmanager-shaped payload → parsed to correct component/status.`
- `Integration: unauthenticated ingest → 401.`

#### 6.2 — Scheduled maintenance

**What**: Schedule maintenance windows, pre-notify subscribers, and auto-transition scheduled→in_progress→completed.

**Design**:
- `scheduled_maintenances` + `maintenance_components` + `maintenance_updates` per suggestion-1. Celery Beat job transitions windows on schedule when `auto_transition=true`, setting affected components to `under_maintenance` during, restoring after.
- Pre-notification (`maintenance_scheduled`) sent at creation; `maintenance_started`/`completed` on transition.

**Testing**:
- `Integration: create maintenance → maintenance_scheduled notifications enqueued.`
- `Integration: beat tick at scheduled_start → status in_progress, components under_maintenance, started notification sent.`
- `Integration: at scheduled_end → completed, components restored to prior status.`

---

## Phase 7: Built-in Uptime Monitoring

### Purpose
Deliver the headline differentiator: native HTTP/TCP/DNS/SSL monitoring that drives component status automatically, eliminating the third-party-monitoring gap that every standalone status page tool (Statuspage, Instatus, Freshstatus) suffers. Feeds uptime history and the AI layer.

### Tasks

#### 7.1 — Probe implementations

**What**: Async probes for HTTP, TCP, DNS, and SSL checks returning a normalized result.

**Design**:
```python
@dataclass
class ProbeResult:
    status: Literal["up","down","degraded"]
    response_time_ms: int | None
    status_code: int | None
    error_message: str | None
class Probe(Protocol):
    async def run(self, monitor: MonitorConfig) -> ProbeResult: ...
```
- HTTP: `httpx` GET/POST, assert `http_expected_status`, classify slow-but-ok as `degraded`.
- TCP: open socket to host:port within timeout.
- DNS: `dnspython` resolve `dns_record_type`, compare `dns_expected_value`.
- SSL: fetch cert, `down` on handshake failure, `degraded` if expiry within `ssl_expiry_warn_days`.
- All read type-specific fields from `monitors.config` JSONB.

**Testing**:
- `Integration (respx): HTTP 200 within timeout → up; 500 → down; >threshold latency → degraded.`
- `Unit (mock resolver): DNS record mismatch → down with error_message.`
- `Integration: SSL cert expiring in 5 days, warn_days=14 → degraded.`
- `Integration: TCP to closed port → down.`

#### 7.2 — Scheduler & check persistence

**What**: Dispatch active monitors on their interval, run probes in workers, and persist checks to the partitioned table.

**Design**:
- Celery Beat `dispatch_due_monitors()` every 10s queries `idx_monitors_active` for monitors due (last_check + interval ≤ now) and enqueues `run_check(monitor_id)`.
- `run_check` runs the probe, inserts a `monitor_checks` row into the current month partition (auto-create via `ensure_partition`), updates rolling failure count.

**Testing**:
- `Integration: monitor with interval 60s → dispatched roughly once/minute (fake clock).`
- `Integration: run_check writes to correct monthly partition; next-month rollover creates new partition.`

#### 7.3 — Status evaluation & auto-update

**What**: Translate consecutive check results into component status transitions.

**Design**:
- `monitoring/evaluator.py`: after each check, if `auto_update_component`, apply hysteresis — `consecutive_failures_threshold` consecutive `down` → set component `major_outage` (and optionally auto-open incident); first `up` after downtime → `operational` (and resolve auto-incident).
- Reuses Phase 3 `set_component_status` / incident services and Phase 5 notification fan-out.

**Testing**:
- `Unit: 2 failures with threshold 3 → no status change; 3rd failure → major_outage.`
- `Integration: monitor recovers → component operational, auto-incident resolved, subscribers notified.`

#### 7.4 — Uptime history & charts API

**What**: Endpoints exposing uptime percentage and response-time series per component.

**Design**:
- `GET /v1/components/{id}/uptime?days=30` runs the suggestion-1 uptime query (uptime_pct, avg_response_ms) plus daily buckets for the public-page chart.

**Testing**:
- `Integration: seeded checks 99/100 up → uptime_pct == 99.0.`
- `Integration: empty range → uptime_pct null, no error.`

---

## Phase 8: AI-Native Layer

### Purpose
Deliver the project's core thesis: AI as a first-class layer. Generate incident-update drafts from monitoring context, draft post-mortems on resolution, and translate natural-language incident descriptions into structured fields — features absent across the incumbent landscape.

### Tasks

#### 8.1 — LLM provider abstraction

**What**: Provider-neutral interface with an Anthropic implementation and prompt caching.

**Design**:
```python
class LLMProvider(Protocol):
    async def complete(self, system: str, user: str,
                       *, max_tokens: int, cache: bool = True) -> LLMResult: ...
class AnthropicProvider:  # uses anthropic SDK; sets cache_control on system prompt
    ...
```
- Disabled cleanly when `ai_enabled=false` / no API key (endpoints return 409 "AI not configured").

**Testing**:
- `Unit (mock SDK): complete returns text + token usage; cache flag sets cache_control.`
- `Integration: ai_enabled=false → AI endpoints return 409 problem+json.`

#### 8.2 — Incident-update drafting

**What**: Generate a draft incident update from monitoring signals + incident context, stored in `ai_draft_queue` for human review.

**Design**:
- `ai/prompts.py` template (structure):
  - System: "You are an SRE incident communicator. Write a concise, factual customer-facing status update. Do not speculate beyond the provided signals. Match the {status} lifecycle stage. 2–4 sentences."
  - User: serialized context — affected components, recent `monitor_checks` (status/error/latency), current incident status, prior updates.
- `POST /v1/incidents/{id}/ai-draft` → enqueues `generate_incident_draft`; result written to `ai_draft_queue` (status `pending`). Operator accepts/edits/rejects; acceptance creates a normal `incident_updates` row with `is_ai_generated=true`.

**Testing**:
- `Integration (mock LLM): request draft → ai_draft_queue row pending with context_sources populated.`
- `Integration: accept draft → incident_updates row created, is_ai_generated=true; queue row status 'accepted'.`
- `Integration: reject draft → no incident_update created; status 'rejected'.`

#### 8.3 — Post-mortem generation

**What**: On incident resolution, draft a structured post-mortem (summary, root cause, impact, timeline, corrective actions).

**Design**:
- `POST /v1/incidents/{id}/postmortem/ai` builds context from the full incident timeline + correlated checks, fills the `post_mortems` fields (suggestion-1), `is_ai_generated=true`, `is_published=false`. Operator edits and publishes.

**Testing**:
- `Integration (mock LLM): generate → post_mortems row created with all narrative fields, unpublished.`
- `Integration: publish → published_at set; appears in Atom feed / public page.`

#### 8.4 — Natural-language incident command

**What**: Translate a plain-English description into structured severity, affected components, and ETA.

**Design**:
- `POST /v1/pages/{id}/incidents/parse` with `{ "text": "API is throwing 500s for EU users, fix in ~30m" }` → structured-output LLM call returning `{severity, impact, component_ids[], eta, title}` validated against a Pydantic schema; component names fuzzy-matched to IDs.

**Testing**:
- `Integration (mock LLM returning structured JSON): parse → valid IncidentDraft; unknown component name → flagged, not silently dropped.`
- `Unit: LLM returns malformed JSON → validation error surfaced as 422, no incident created.`

---

## Phase 9: Enterprise — Private Pages, SSO & Custom Domains

### Purpose
Unlock enterprise sales: private/password-protected pages, SAML 2.0 / OIDC SSO for internal status pages, and custom domains with automatic SSL. These gate access to internal pages and are required by enterprise buyers per standards.md.

### Tasks

#### 9.1 — Private pages & password protection

**What**: Gate public pages behind a shared password or authenticated session.

**Design**:
- `status_pages.is_public=false` + optional password hash. Unauthenticated access to a private page → 401/login. Session cookie signed with `secret_key`.

**Testing**:
- `Integration: private page without auth → 401; with correct password → 200 + session cookie.`

#### 9.2 — SSO (SAML 2.0 / OIDC)

**What**: Enterprise SSO for private internal pages.

**Design**:
- `auth/saml.py` (`python3-saml`) SP metadata + ACS endpoint; `auth/oidc.py` (`authlib`) authorization-code flow. Per-org IdP config. On success, establish a page session.

**Testing**:
- `Integration (mock IdP): SAML ACS with valid signed assertion → session established.`
- `Integration: tampered/expired assertion → 401.`
- `Integration (mock OIDC): code exchange → session; invalid state → rejected.`

#### 9.3 — Custom domains with automatic SSL

**What**: Serve pages on customer domains with auto-provisioned TLS.

**Design**:
- `custom_domain` (suggestion-1) + ACME (`certbot`/`acme` lib) HTTP-01 challenge handler; domain ownership verified via CNAME/TXT before issuance. Reverse proxy (Caddy in compose) terminates TLS using issued certs.

**Testing**:
- `Integration: add custom_domain → verification token issued; CNAME present → verified, cert provisioning triggered (mock ACME).`
- `Integration: request to verified custom domain → resolves to correct page.`

---

## Phase 10: Hardening, OpenAPI Publication & Release

### Purpose
Make the platform production-ready and externally consumable: a stable, committed OpenAPI 3.1 spec, rate limiting, an OWASP API Security audit, observability, and release packaging. After this phase the platform is deployable and its API contract is publishable for SDK generation.

### Tasks

#### 10.1 — OpenAPI 3.1 spec publication & contract test

**What**: Export, commit, and CI-diff the OpenAPI spec.

**Design**:
- Build step writes `openapi/openapi.json` from the FastAPI app. CI fails if generated spec diverges from committed spec (forces intentional contract changes). All error responses documented as `application/problem+json`.

**Testing**:
- `Integration: generated spec validates against OpenAPI 3.1 meta-schema.`
- `CI: spec drift between code and committed file → build fails.`

#### 10.2 — Rate limiting & OWASP API audit

**What**: Per-key rate limits and a documented audit against the OWASP API Security Top 10.

**Design**:
- Redis token-bucket limiter keyed by API key / IP (public endpoints). 429 + `Retry-After` as problem+json. Audit checklist mapping each OWASP API risk (API1 object-level auth via tenant scoping, API3 data exposure via Read schemas, API4 resource limits via rate limiting, etc.) to implemented controls; bandit + dependency scan in CI.

**Testing**:
- `Integration: exceed rate limit → 429 problem+json with Retry-After.`
- `Integration: cross-tenant object access attempt → 404 (re-verifies API1).`
- `CI: bandit and pip-audit pass with no high-severity findings.`

#### 10.3 — Observability & audit log export

**What**: Structured logging, Prometheus metrics, and exportable audit events.

**Design**:
- `/metrics` (OpenMetrics) exposes request counts/latency, notification delivery success rate, monitor check throughput, AI draft acceptance rate. `GET /v1/audit` (scoped) exports `audit_events` for SLA evidence.

**Testing**:
- `Integration: /metrics → valid OpenMetrics text exposition.`
- `Integration: actions across incidents/notifications produce queryable audit_events.`

#### 10.4 — Release packaging & docs

**What**: Production compose, versioned images, and self-host quickstart.

**Design**:
- Tagged multi-arch images; `docker-compose.yml` production profile with Caddy TLS, Postgres backups note, env reference. Quickstart doc: bring-up in < 5 min (matches Better Stack onboarding benchmark).

**Testing**:
- `E2E: fresh `docker compose up` from release tag → create page, component, incident, subscribe, receive notification, view public page — full happy path passes.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Model        ─── required by everything
    │
Phase 2: Tenancy, Auth & API Keys       ─── requires 1
    │
Phase 3: Components & Incidents (CORE)   ─── requires 2
    │
    ├── Phase 4: Public Page, Feeds, SSE        ─── requires 3 ─┐ can parallel
    ├── Phase 5: Subscribers & Notifications    ─── requires 3 ─┘ with each other
    │        │
    │   Phase 6: Inbound Webhooks & Maintenance ─── requires 3, 5
    │        │
    │   Phase 7: Built-in Uptime Monitoring     ─── requires 3 (uses 5 for notify)
    │        │
    │   Phase 8: AI-Native Layer                ─── requires 3, 7 (best signal)
    │
Phase 9: Enterprise (Private/SSO/Domains) ─── requires 4 (can parallel with 6–8)
    │
Phase 10: Hardening & Release            ─── requires all
```

**MVP line (features.md "Must-have")** completes at **Phase 6**: Phases 1–6 deliver components, incident lifecycle, public page, email + webhook notifications, OpenAPI-described API with key auth, and the inbound monitoring webhook.

**Parallelism opportunities:**
- Phases 4 and 5 can be built concurrently once Phase 3 lands.
- Phase 9 (enterprise) is independent of Phases 6–8 and can proceed in parallel.
- Within Phase 7, probe implementations (7.1) can be built independently of the scheduler (7.2).

**v1.1 line ("Should-have")** completes at **Phase 9** (monitoring, maintenance, Slack/Teams, SSO, AI drafts, uptime charts). Backlog items (on-call, monitoring-as-code, impact segmentation, third-party aggregation, multi-language) layer on after Phase 10.

---

## Definition of Done (per phase)

Every phase must satisfy this checklist before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests for the phase pass; coverage does not regress.
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes for the affected packages.
5. `bandit` reports no new high-severity findings.
6. Alembic migration(s) created and `upgrade head` / `downgrade` verified against a real Postgres (testcontainers).
7. New REST endpoints appear in the generated OpenAPI 3.1 spec, with `application/problem+json` error responses documented (RFC 7807).
8. New config options added to `Settings` and the env reference doc.
9. New outbound webhook event types documented and conform to the Standard Webhooks spec.
10. Docker build succeeds and the full `docker compose up` stack reaches `/readyz` 200.
11. The phase's headline capability is demonstrated end-to-end (the E2E test named in the phase passes).
```
