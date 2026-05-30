# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Status Page Platform · Created: 2026-05-24

## Philosophy

A status page platform is fundamentally an incident communication system: components change status, incidents progress through lifecycle states, monitors detect failures, subscribers are notified, and post-mortems are published. Every one of these is a state transition that SRE teams need to audit. An event-sourced model captures every component status change, every incident update, every monitor check, every notification delivery, and every AI draft generation as an immutable event. The event store is the single source of truth; read models are materialised projections optimised for the public status page, incident timeline, and uptime calculations.

This is particularly powerful for a status page platform because:
- **Incident timelines ARE event streams** — The investigating → identified → monitoring → resolved timeline on every status page is literally a sequence of events. Event sourcing models this directly rather than deriving it from mutable state.
- **Component status history is critical** — "When did the API go from operational to major outage?" is answered by querying the event stream directly, not by joining audit tables.
- **Monitor check results are naturally time-series events** — Each check (up/down/degraded) is an immutable observation at a point in time. Event sourcing handles this naturally.
- **Notification delivery needs audit trails** — Enterprise customers need to prove that subscribers were notified within SLA. Event sourcing captures every delivery attempt, retry, and failure as immutable records.
- **AI draft provenance matters** — When AI generates incident updates, the organisation needs to track what context the AI used, what it drafted, and whether a human edited the draft before publishing.

The result is an event store with 30+ event types covering incident lifecycle, component status, monitoring, notification delivery, and AI operations, plus 6 read model tables and 8 reference tables.

**Best for:** Teams building a status page platform for enterprise customers where SLA reporting requires immutable audit trails, where incident timeline accuracy is contractually important, and where notification delivery proof is a compliance requirement.

**Trade-offs:**
- **Pro:** Complete audit trail — every status change, notification, and AI action is immutable
- **Pro:** Incident timelines derived directly from events — no mutable state
- **Pro:** SLA reporting from event data — provable notification delivery times
- **Pro:** Component status history without a separate history table
- **Pro:** AI draft provenance fully captured
- **Con:** Eventual consistency between event store and public status page
- **Con:** Public page rendering requires read model — not a direct table scan
- **Con:** Event schema versioning adds operational complexity
- **Con:** More infrastructure (event bus, projection workers) than a CRUD model

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope fields (ce_source, ce_type, ce_specversion) |
| RFC 4287 (Atom) | Incident feed generated from event projections |
| RFC 3339 | Timestamps in event payloads |
| Standard Webhooks | HMAC-SHA256 signed outbound webhooks |
| OpenMetrics | Monitor check events carry metric-compatible data |
| OTLP | Telemetry correlation events for AI incident analysis |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| ITIL 4 | Incident severity taxonomy in event payloads |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL CHECK (stream_type IN (
        'incident', 'component', 'monitor', 'maintenance',
        'subscriber', 'notification', 'page'
    )),
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    event_version INT NOT NULL,
    payload JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"actor_id": "user-uuid", "actor_type": "user|system|monitor|ai",
    --  "ip_address": "...", "user_agent": "...",
    --  "ce_source": "/statuspage/incidents", "ce_specversion": "1.0",
    --  "correlation_id": "uuid", "causation_id": "uuid"}
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE event_store_2026_q1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE event_store_2026_q2 PARTITION OF event_store
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_q3 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE event_store_2026_q4 PARTITION OF event_store
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_events_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_events_occurred ON event_store(occurred_at);
```

---

## Event Type Registry

```sql
CREATE TABLE event_type_registry (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1
);

-- Incident lifecycle events
INSERT INTO event_type_registry VALUES
    ('incident.created', 'incident', 'New incident declared', 1),
    ('incident.updated', 'incident', 'Incident status or severity changed', 1),
    ('incident.component_affected', 'incident', 'Component linked to incident', 1),
    ('incident.component_status_changed', 'incident', 'Affected component status changed', 1),
    ('incident.resolved', 'incident', 'Incident marked resolved', 1),
    ('incident.postmortem_drafted', 'incident', 'Post-mortem draft created', 1),
    ('incident.postmortem_published', 'incident', 'Post-mortem published publicly', 1),
    ('incident.ai_update_drafted', 'incident', 'AI generated incident update draft', 1),
    ('incident.ai_update_accepted', 'incident', 'AI draft accepted by human', 1),
    ('incident.ai_update_edited', 'incident', 'AI draft edited by human before publishing', 1),
    ('incident.ai_update_rejected', 'incident', 'AI draft rejected by human', 1),

    -- Component events
    ('component.created', 'component', 'New component added to page', 1),
    ('component.status_changed', 'component', 'Component status changed', 1),
    ('component.updated', 'component', 'Component name/description/order changed', 1),
    ('component.deleted', 'component', 'Component removed from page', 1),

    -- Monitor events
    ('monitor.check_completed', 'monitor', 'Monitor check returned result', 1),
    ('monitor.status_changed', 'monitor', 'Monitor transitioned up/down/degraded', 1),
    ('monitor.threshold_breached', 'monitor', 'Consecutive failures exceeded threshold', 1),
    ('monitor.recovered', 'monitor', 'Monitor returned to up after failures', 1),
    ('monitor.config_changed', 'monitor', 'Monitor configuration updated', 1),

    -- Maintenance events
    ('maintenance.scheduled', 'maintenance', 'Maintenance window scheduled', 1),
    ('maintenance.started', 'maintenance', 'Maintenance window started', 1),
    ('maintenance.update_posted', 'maintenance', 'Maintenance progress update posted', 1),
    ('maintenance.completed', 'maintenance', 'Maintenance window completed', 1),

    -- Subscriber events
    ('subscriber.created', 'subscriber', 'New subscriber registered', 1),
    ('subscriber.confirmed', 'subscriber', 'Subscriber confirmed email/phone', 1),
    ('subscriber.unsubscribed', 'subscriber', 'Subscriber opted out', 1),

    -- Notification events
    ('notification.queued', 'notification', 'Notification queued for delivery', 1),
    ('notification.sent', 'notification', 'Notification dispatched to channel', 1),
    ('notification.delivered', 'notification', 'Delivery confirmed', 1),
    ('notification.failed', 'notification', 'Delivery failed', 1),
    ('notification.bounced', 'notification', 'Email bounced', 1),
    ('notification.retried', 'notification', 'Failed notification retried', 1),

    -- Page events
    ('page.created', 'page', 'Status page created', 1),
    ('page.branding_updated', 'page', 'Page branding changed', 1),
    ('page.domain_configured', 'page', 'Custom domain set up', 1),
    ('page.sso_configured', 'page', 'SSO authentication configured', 1);
```

---

## Event Payload Examples

```sql
-- incident.created
-- {"title": "API experiencing elevated error rates",
--  "severity": "major", "impact": "major",
--  "initial_update": "We are investigating reports of API errors.",
--  "affected_components": [{"id": "uuid", "name": "API", "status": "major_outage"}],
--  "trigger": "monitor", "monitor_id": "uuid"}

-- incident.updated
-- {"previous_status": "investigating", "new_status": "identified",
--  "body": "Root cause identified: database connection pool exhaustion.",
--  "is_ai_generated": false, "updated_by": "user-uuid"}

-- incident.ai_update_drafted
-- {"draft_body": "We are investigating elevated error rates on the API endpoint.",
--  "confidence": 0.85, "model": "claude-sonnet-4-6",
--  "context_sources": ["monitor:api-health", "alert:error-rate-spike"],
--  "suggested_status": "investigating", "suggested_severity": "major"}

-- monitor.check_completed
-- {"status": "up", "response_time_ms": 145, "status_code": 200,
--  "region": "us-east-1", "monitor_type": "http",
--  "target": "https://api.example.com/health"}

-- monitor.threshold_breached
-- {"consecutive_failures": 3, "threshold": 3,
--  "last_3_checks": [
--    {"status": "down", "error": "timeout", "at": "2026-05-24T09:57:00Z"},
--    {"status": "down", "error": "timeout", "at": "2026-05-24T09:58:00Z"},
--    {"status": "down", "error": "connection_refused", "at": "2026-05-24T09:59:00Z"}
--  ],
--  "auto_incident_created": true, "incident_id": "uuid"}

-- notification.sent
-- {"subscriber_id": "uuid", "channel": "email",
--  "recipient": "user@example.com", "subject": "Incident: API experiencing errors",
--  "incident_id": "uuid", "notification_type": "incident_created",
--  "provider": "ses", "message_id": "ses-msg-uuid"}

-- notification.failed
-- {"subscriber_id": "uuid", "channel": "webhook",
--  "recipient_url": "https://example.com/hook",
--  "error": "HTTP 503: Service Unavailable", "retry_count": 2,
--  "next_retry_at": "2026-05-24T10:10:00Z"}
```

---

## Stream Snapshots

```sql
CREATE TABLE stream_snapshots (
    stream_id UUID NOT NULL,
    stream_type TEXT NOT NULL,
    snapshot_version INT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Reference Data

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    billing_plan TEXT NOT NULL DEFAULT 'free',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_members (
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    PRIMARY KEY (org_id, user_id)
);

CREATE TABLE incident_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    title_template TEXT NOT NULL,
    body_template TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'minor',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);

CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    secret TEXT NOT NULL,
    events TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Models (CQRS Projections)

```sql
CREATE TABLE rm_status_pages (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL REFERENCES organisations(id),
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    custom_domain TEXT,
    is_public BOOLEAN NOT NULL DEFAULT TRUE,
    branding JSONB NOT NULL DEFAULT '{}',
    sso JSONB,
    locale TEXT NOT NULL DEFAULT 'en',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    overall_status TEXT NOT NULL DEFAULT 'operational',
    active_incident_count INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_pages_org ON rm_status_pages(org_id);
CREATE INDEX idx_rm_pages_domain ON rm_status_pages(custom_domain) WHERE custom_domain IS NOT NULL;

CREATE TABLE rm_components (
    id UUID PRIMARY KEY,
    page_id UUID NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    group_name TEXT,
    status TEXT NOT NULL DEFAULT 'operational',
    sort_order INT NOT NULL DEFAULT 0,
    is_visible BOOLEAN NOT NULL DEFAULT TRUE,
    uptime_24h REAL,
    uptime_7d REAL,
    uptime_30d REAL,
    avg_response_ms REAL,
    last_check_at TIMESTAMPTZ,
    last_check_status TEXT,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_components ON rm_components(page_id, sort_order);

CREATE TABLE rm_incidents (
    id UUID PRIMARY KEY,
    page_id UUID NOT NULL,
    title TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'investigating',
    severity TEXT NOT NULL DEFAULT 'minor',
    impact TEXT NOT NULL DEFAULT 'none',
    affected_components JSONB NOT NULL DEFAULT '[]',
    updates JSONB NOT NULL DEFAULT '[]',
    post_mortem JSONB,
    started_at TIMESTAMPTZ NOT NULL,
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_incidents ON rm_incidents(page_id, created_at DESC);
CREATE INDEX idx_rm_incidents_active ON rm_incidents(page_id)
    WHERE status NOT IN ('resolved', 'postmortem');

CREATE TABLE rm_subscribers (
    id UUID PRIMARY KEY,
    page_id UUID NOT NULL,
    channel_type TEXT NOT NULL,
    contact_display TEXT NOT NULL,
    component_ids UUID[],
    is_confirmed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_subscribers ON rm_subscribers(page_id);

CREATE TABLE rm_maintenance (
    id UUID PRIMARY KEY,
    page_id UUID NOT NULL,
    title TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'scheduled',
    affected_components JSONB NOT NULL DEFAULT '[]',
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end TIMESTAMPTZ NOT NULL,
    updates JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_maintenance ON rm_maintenance(page_id, scheduled_start);

CREATE TABLE rm_notification_stats (
    page_id UUID NOT NULL,
    date DATE NOT NULL,
    channel TEXT NOT NULL,
    queued INT NOT NULL DEFAULT 0,
    sent INT NOT NULL DEFAULT 0,
    delivered INT NOT NULL DEFAULT 0,
    failed INT NOT NULL DEFAULT 0,
    bounced INT NOT NULL DEFAULT 0,
    avg_delivery_seconds REAL,
    PRIMARY KEY (page_id, date, channel)
);
```

---

## Example Queries

### Replay incident timeline

```sql
SELECT event_type, payload, metadata->>'actor_type' AS actor,
       occurred_at
FROM event_store
WHERE stream_type = 'incident'
  AND stream_id = 'incident-uuid'
ORDER BY event_version;
```

### Component status history (last 7 days)

```sql
SELECT payload->>'previous_status' AS from_status,
       payload->>'new_status' AS to_status,
       metadata->>'actor_type' AS changed_by,
       occurred_at
FROM event_store
WHERE stream_type = 'component'
  AND stream_id = 'component-uuid'
  AND event_type = 'component.status_changed'
  AND occurred_at >= now() - INTERVAL '7 days'
ORDER BY occurred_at;
```

### Notification delivery SLA report

```sql
SELECT DATE_TRUNC('day', occurred_at) AS day,
       payload->>'channel' AS channel,
       COUNT(*) FILTER (WHERE event_type = 'notification.sent') AS sent,
       COUNT(*) FILTER (WHERE event_type = 'notification.delivered') AS delivered,
       COUNT(*) FILTER (WHERE event_type = 'notification.failed') AS failed,
       COUNT(*) FILTER (WHERE event_type = 'notification.bounced') AS bounced
FROM event_store
WHERE stream_type = 'notification'
  AND occurred_at >= now() - INTERVAL '30 days'
GROUP BY DATE_TRUNC('day', occurred_at), payload->>'channel'
ORDER BY day;
```

### AI draft acceptance rate

```sql
SELECT COUNT(*) FILTER (WHERE event_type = 'incident.ai_update_drafted') AS drafted,
       COUNT(*) FILTER (WHERE event_type = 'incident.ai_update_accepted') AS accepted,
       COUNT(*) FILTER (WHERE event_type = 'incident.ai_update_edited') AS edited,
       COUNT(*) FILTER (WHERE event_type = 'incident.ai_update_rejected') AS rejected
FROM event_store
WHERE event_type LIKE 'incident.ai_update_%'
  AND occurred_at >= now() - INTERVAL '90 days';
```

### Monitor uptime from events (last 24h)

```sql
SELECT COUNT(*) FILTER (WHERE payload->>'status' = 'up') * 100.0 / COUNT(*) AS uptime_pct,
       AVG((payload->>'response_time_ms')::INT) AS avg_response_ms
FROM event_store
WHERE stream_type = 'monitor'
  AND stream_id = 'monitor-uuid'
  AND event_type = 'monitor.check_completed'
  AND occurred_at >= now() - INTERVAL '24 hours';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), event_type_registry, stream_snapshots, projection_checkpoints |
| Reference Data | 6 | users, organisations, org_members, incident_templates, webhooks, api_keys |
| Read Models | 6 | rm_status_pages, rm_components, rm_incidents, rm_subscribers, rm_maintenance, rm_notification_stats |
| **Total** | **15** | |

---

## Key Design Decisions

1. **Incident timeline from events** — The investigating → identified → monitoring → resolved timeline is built by replaying `incident.created`, `incident.updated`, and `incident.resolved` events. The `rm_incidents` read model materialises the timeline as a JSONB array for fast public page rendering. The event store preserves the exact sequence, timestamps, and actors.

2. **Component status changes as events** — Every `component.status_changed` event captures the previous and new status with a timestamp. This enables component status history queries (when did the API go down and come back?) without a separate status history table.

3. **Monitor checks as events** — `monitor.check_completed` events capture each check result (status, response time, region). Uptime percentages are calculated by filtering these events. The `monitor.threshold_breached` event captures the moment consecutive failures trigger an incident, linking the monitor stream to the incident stream.

4. **Notification delivery as event stream** — Each notification has its own stream: `notification.queued` → `notification.sent` → `notification.delivered` (or `notification.failed` → `notification.retried`). This provides an immutable delivery audit trail for SLA reporting: "subscriber X was notified at time Y via channel Z, delivered in N seconds."

5. **AI draft provenance** — `incident.ai_update_drafted` captures what the AI generated, what context it used, and its confidence. `incident.ai_update_accepted`/`edited`/`rejected` captures the human review decision. Replaying these events reveals AI accuracy over time and informs prompt improvements.

6. **Notification stats as read model** — `rm_notification_stats` aggregates daily delivery metrics (queued, sent, delivered, failed, bounced, avg delivery time) per page and channel. This enables SLA dashboards without querying the event store for every request.

7. **Monitor auto-incident creation** — `monitor.threshold_breached` events include `auto_incident_created: true` and the new `incident_id` when the system automatically creates an incident from monitor failures. This links the monitor stream to the incident stream via event causation.

8. **Partitioned event store** — Quarterly partitions for manageable query performance. Status page events accumulate quickly (monitor checks every 60 seconds per component), so partitioning is particularly important in this domain.
