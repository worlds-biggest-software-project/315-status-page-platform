# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Status Page Platform · Created: 2026-05-24

## Philosophy

A status page platform has a stable relational core (organisations own pages, pages display components, components are affected by incidents, subscribers receive notifications) surrounded by highly variable data: monitor configurations differ by type (HTTP checks need method/headers/expected status; DNS checks need record type/expected value; SSL checks need expiry thresholds), notification channel configurations vary by channel (email needs SMTP settings; Slack needs webhook URLs; Teams needs connector URLs), branding customisation varies per page, AI-generated content evolves as models improve, and post-mortem structures vary by team convention. A JSONB hybrid keeps the invariant incident communication pipeline relational (every incident has a status, severity, and timeline) while configuration, channel-specific, and AI-generated data lives in JSONB columns.

This is particularly effective for a status page platform because:
- **Monitor configurations vary by type** — An HTTP monitor needs method, headers, and expected status code; a DNS monitor needs record type and expected value; an SSL monitor needs expiry warning threshold. JSONB stores each monitor type's configuration without type-specific columns.
- **Notification channels vary by provider** — Email needs SMTP/SES settings; Slack needs a webhook URL; Microsoft Teams needs a connector URL; SMS needs a phone provider config. JSONB handles all channel configurations.
- **Incident updates are often consumed as a block** — The public status page displays the full incident timeline at once. Embedding updates as a JSONB array on the incident row eliminates a join for the most common read pattern.
- **AI outputs evolve** — Today's AI drafts incident updates; tomorrow's might draft post-mortems from correlated telemetry. JSONB avoids migrations as AI features mature.

The result is a 12-table schema that covers the full platform with dramatically simpler queries than the 21-table normalized model.

**Best for:** Rapid MVP development, teams shipping a status page platform quickly, and platforms where multi-type monitor support, flexible notification channels, and evolving AI features matter more than constraint enforcement on variable fields.

**Trade-offs:**
- **Pro:** 12 tables — dramatically simpler than the 21-table normalized model
- **Pro:** Monitor configuration is schema-free — adding a new monitor type needs no migration
- **Pro:** Notification channel config unified in one column
- **Pro:** Incident timeline inline — single row read for the entire incident history
- **Con:** JSONB fields lack database-level constraints
- **Con:** Monitor-specific queries require JSONB extraction
- **Con:** Application must validate monitor and channel configurations
- **Con:** Inline incident updates limit per-update indexing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1 | REST API specification |
| RFC 4287 (Atom) | Incident feed generation |
| RFC 3339 | Timestamps in JSONB payloads |
| Standard Webhooks | HMAC-SHA256 signed outbound webhooks |
| OpenMetrics | Monitor metric ingest format |
| OTLP | Telemetry ingest for AI correlation |
| SAML 2.0 / OIDC | SSO config in status_pages JSONB |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| ITIL 4 | Incident severity values |

---

## Core Tables

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT,
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"timezone": "America/New_York", "locale": "en",
    --  "notifications": {"email_on_incident": true, "slack_on_incident": false}}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    billing_plan TEXT NOT NULL DEFAULT 'free',
    members JSONB NOT NULL DEFAULT '[]',
    -- [{"user_id": "uuid", "name": "Alice", "role": "owner"},
    --  {"user_id": "uuid", "name": "Bob", "role": "admin"}]
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"default_severity": "minor", "auto_resolve_after_hours": 24,
    --  "notification_channels": {
    --    "slack": {"webhook_url": "https://hooks.slack.com/...", "channel": "#incidents"},
    --    "teams": {"connector_url": "https://outlook.office.com/webhook/..."},
    --    "email": {"from_address": "status@example.com", "reply_to": "ops@example.com"}
    --  }}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE status_pages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    custom_domain TEXT,
    is_public BOOLEAN NOT NULL DEFAULT TRUE,
    branding JSONB NOT NULL DEFAULT '{}',
    -- {"logo_url": "https://...", "favicon_url": "https://...",
    --  "accent_color": "#2563eb", "custom_css": "body { font-family: ... }",
    --  "header_html": "<div class='banner'>...</div>"}
    sso JSONB,
    -- {"provider": "okta", "entity_id": "https://...",
    --  "sso_url": "https://okta.example.com/app/...",
    --  "certificate": "MIIC...", "protocol": "saml"}
    locale TEXT NOT NULL DEFAULT 'en',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_pages_org ON status_pages(org_id);
CREATE INDEX idx_pages_domain ON status_pages(custom_domain) WHERE custom_domain IS NOT NULL;
```

---

## Components

```sql
CREATE TABLE components (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES status_pages(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT,
    group_name TEXT,
    status TEXT NOT NULL DEFAULT 'operational' CHECK (status IN (
        'operational', 'degraded_performance', 'partial_outage',
        'major_outage', 'under_maintenance'
    )),
    sort_order INT NOT NULL DEFAULT 0,
    is_visible BOOLEAN NOT NULL DEFAULT TRUE,
    monitor JSONB,
    -- HTTP: {"type": "http", "target": "https://api.example.com/health",
    --        "method": "GET", "expected_status": 200,
    --        "headers": {"Authorization": "Bearer ..."},
    --        "interval_seconds": 60, "timeout_seconds": 10,
    --        "regions": ["us-east-1", "eu-west-1"],
    --        "consecutive_failures_threshold": 3,
    --        "auto_update_component": true, "is_active": true}
    -- TCP: {"type": "tcp", "target": "db.example.com:5432",
    --       "interval_seconds": 30, "timeout_seconds": 5}
    -- DNS: {"type": "dns", "target": "example.com",
    --       "record_type": "A", "expected_value": "93.184.216.34"}
    -- SSL: {"type": "ssl", "target": "example.com",
    --       "check_expiry": true, "expiry_warn_days": 14}
    uptime_stats JSONB NOT NULL DEFAULT '{}',
    -- {"uptime_24h": 99.95, "uptime_7d": 99.98, "uptime_30d": 99.92,
    --  "avg_response_ms_24h": 145, "last_check_at": "2026-05-24T12:00:00Z",
    --  "last_check_status": "up", "consecutive_failures": 0}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_components_page ON components(page_id, sort_order);
```

---

## Incidents

```sql
CREATE TABLE incidents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES status_pages(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'investigating' CHECK (status IN (
        'investigating', 'identified', 'monitoring', 'resolved', 'postmortem'
    )),
    severity TEXT NOT NULL DEFAULT 'minor' CHECK (severity IN (
        'critical', 'major', 'minor', 'maintenance'
    )),
    impact TEXT NOT NULL DEFAULT 'none' CHECK (impact IN (
        'none', 'minor', 'major', 'critical'
    )),
    affected_component_ids UUID[] NOT NULL DEFAULT '{}',
    affected_component_statuses JSONB NOT NULL DEFAULT '{}',
    -- {"component-uuid-1": "major_outage", "component-uuid-2": "degraded_performance"}
    updates JSONB NOT NULL DEFAULT '[]',
    -- [{"status": "investigating", "body": "We are investigating reports of API errors.",
    --   "is_ai_generated": false, "created_by": "user-uuid",
    --   "created_at": "2026-05-24T10:00:00Z"},
    --  {"status": "identified", "body": "Root cause identified: database connection pool exhaustion.",
    --   "is_ai_generated": true, "ai_model": "claude-sonnet-4-6",
    --   "created_by": "user-uuid", "created_at": "2026-05-24T10:15:00Z"},
    --  {"status": "resolved", "body": "Connection pool increased. Monitoring for stability.",
    --   "is_ai_generated": false, "created_by": "user-uuid",
    --   "created_at": "2026-05-24T11:00:00Z"}]
    post_mortem JSONB,
    -- {"summary": "Database connection pool exhaustion caused 45-minute API outage.",
    --  "root_cause": "Connection leak in payment service after v2.3.1 deployment.",
    --  "impact_description": "35% of API requests returned 503 for 45 minutes.",
    --  "timeline": [
    --    {"time": "10:00", "event": "Monitoring alert: API error rate > 5%"},
    --    {"time": "10:05", "event": "On-call engineer paged"},
    --    {"time": "10:15", "event": "Root cause identified: connection pool exhaustion"},
    --    {"time": "10:45", "event": "Hotfix deployed: pool size increased to 200"},
    --    {"time": "11:00", "event": "Error rate returned to baseline"}
    --  ],
    --  "corrective_actions": ["Add connection pool monitoring alert", "Review deployment checklist"],
    --  "is_ai_generated": true, "ai_model": "claude-sonnet-4-6",
    --  "is_published": false, "published_at": null}
    ai JSONB NOT NULL DEFAULT '{}',
    -- {"draft_updates": [
    --    {"body": "We are investigating elevated error rates on the API.",
    --     "confidence": 0.85, "context_sources": ["monitor:api-health", "alert:error-rate"],
    --     "status": "pending", "created_at": "2026-05-24T10:01:00Z"}
    --  ],
    --  "suggested_severity": "major", "suggested_components": ["api", "payments"],
    --  "anomaly_correlation": {"metrics": ["api_error_rate", "db_pool_exhaustion"],
    --                          "confidence": 0.92}}
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    identified_at TIMESTAMPTZ,
    monitoring_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_incidents_page ON incidents(page_id, created_at DESC);
CREATE INDEX idx_incidents_active ON incidents(page_id, started_at DESC)
    WHERE status NOT IN ('resolved', 'postmortem');
CREATE INDEX idx_incidents_components ON incidents USING GIN (affected_component_ids);
```

---

## Scheduled Maintenance

```sql
CREATE TABLE scheduled_maintenances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES status_pages(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'scheduled' CHECK (status IN (
        'scheduled', 'in_progress', 'completed'
    )),
    affected_component_ids UUID[] NOT NULL DEFAULT '{}',
    updates JSONB NOT NULL DEFAULT '[]',
    -- [{"status": "scheduled", "body": "Planned database upgrade.",
    --   "created_by": "user-uuid", "created_at": "2026-05-20T10:00:00Z"},
    --  {"status": "in_progress", "body": "Maintenance has started. Expect brief downtime.",
    --   "created_by": "user-uuid", "created_at": "2026-05-24T02:00:00Z"}]
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end TIMESTAMPTZ NOT NULL,
    actual_start TIMESTAMPTZ,
    actual_end TIMESTAMPTZ,
    auto_transition BOOLEAN NOT NULL DEFAULT TRUE,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_maintenance_page ON scheduled_maintenances(page_id, scheduled_start);
```

---

## Monitor Checks

```sql
CREATE TABLE monitor_checks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    component_id UUID NOT NULL REFERENCES components(id) ON DELETE CASCADE,
    status TEXT NOT NULL CHECK (status IN ('up', 'down', 'degraded')),
    response_time_ms INT,
    status_code INT,
    region TEXT NOT NULL,
    error_message TEXT,
    details JSONB,
    -- HTTP: {"headers": {"content-type": "application/json"}, "body_size_bytes": 1024}
    -- DNS: {"resolved_value": "93.184.216.34", "ttl": 3600}
    -- SSL: {"issuer": "Let's Encrypt", "expires_at": "2026-08-24", "days_until_expiry": 91}
    checked_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (checked_at);

CREATE INDEX idx_checks_component ON monitor_checks(component_id, checked_at DESC);
CREATE INDEX idx_checks_status ON monitor_checks(component_id, status, checked_at DESC);
```

---

## Subscribers & Notifications

```sql
CREATE TABLE subscribers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES status_pages(id) ON DELETE CASCADE,
    channel_type TEXT NOT NULL CHECK (channel_type IN (
        'email', 'sms', 'webhook', 'slack', 'teams', 'rss'
    )),
    contact JSONB NOT NULL,
    -- Email: {"email": "user@example.com"}
    -- SMS: {"phone": "+1-555-0123", "provider": "twilio"}
    -- Webhook: {"url": "https://example.com/hook", "secret": "..."}
    -- Slack: {"webhook_url": "https://hooks.slack.com/...", "channel": "#ops"}
    -- Teams: {"connector_url": "https://outlook.office.com/webhook/..."}
    component_ids UUID[],
    is_confirmed BOOLEAN NOT NULL DEFAULT FALSE,
    unsubscribe_token TEXT NOT NULL DEFAULT gen_random_uuid()::TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscribers_page ON subscribers(page_id);

CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    incident_id UUID REFERENCES incidents(id),
    maintenance_id UUID REFERENCES scheduled_maintenances(id),
    notification_type TEXT NOT NULL,
    channel TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'sent', 'delivered', 'failed', 'bounced'
    )),
    delivery JSONB NOT NULL DEFAULT '{}',
    -- {"sent_at": "2026-05-24T10:01:00Z", "delivered_at": "2026-05-24T10:01:05Z"}
    -- {"sent_at": "2026-05-24T10:01:00Z", "error": "smtp_timeout", "retry_count": 2}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_subscriber ON notifications(subscriber_id, created_at DESC);
CREATE INDEX idx_notifications_pending ON notifications(status) WHERE status IN ('pending', 'failed');
```

---

## Incident Templates & Integrations

```sql
CREATE TABLE incident_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    -- {"title_template": "{{component}} experiencing issues",
    --  "body_template": "We are investigating reports of {{severity}} impact to {{component}}.",
    --  "severity": "minor", "component_ids": ["uuid-1", "uuid-2"]}
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
    delivery_log JSONB NOT NULL DEFAULT '[]',
    -- [{"event": "incident.created", "status": 200, "at": "2026-05-24T10:01:00Z"}]
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

## Example Queries

### Current page status with component health

```sql
SELECT name, status, group_name, description,
       uptime_stats->>'uptime_30d' AS uptime_30d,
       uptime_stats->>'avg_response_ms_24h' AS avg_response_ms
FROM components
WHERE page_id = 'page-uuid'
  AND is_visible = TRUE
ORDER BY group_name NULLS LAST, sort_order;
```

### Active incident with full timeline

```sql
SELECT title, status, severity, started_at,
       affected_component_ids,
       updates
FROM incidents
WHERE page_id = 'page-uuid'
  AND status NOT IN ('resolved', 'postmortem')
ORDER BY started_at DESC;
```

### Monitor configuration by type

```sql
SELECT name, monitor->>'type' AS monitor_type,
       monitor->>'target' AS target,
       (monitor->>'interval_seconds')::INT AS interval,
       uptime_stats->>'last_check_status' AS last_status
FROM components
WHERE page_id = 'page-uuid'
  AND monitor IS NOT NULL
  AND (monitor->>'is_active')::BOOLEAN = TRUE;
```

### AI draft acceptance rate

```sql
SELECT COUNT(*) FILTER (WHERE (d->>'status') = 'accepted') AS accepted,
       COUNT(*) FILTER (WHERE (d->>'status') = 'edited') AS edited,
       COUNT(*) FILTER (WHERE (d->>'status') = 'rejected') AS rejected,
       COUNT(*) AS total
FROM incidents,
     jsonb_array_elements(ai->'draft_updates') AS d
WHERE page_id = 'page-uuid'
  AND created_at >= now() - INTERVAL '90 days';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core | 3 | users (settings JSONB), organisations (members, settings JSONB), status_pages (branding, SSO JSONB) |
| Components | 1 | components (monitor config, uptime_stats all JSONB; groups via group_name) |
| Incidents | 1 | incidents (updates array, post_mortem, ai all JSONB) |
| Maintenance | 1 | scheduled_maintenances (updates array JSONB) |
| Monitors | 1 | monitor_checks (details JSONB; partitioned) |
| Subscribers | 2 | subscribers (contact JSONB), notifications (delivery JSONB) |
| Templates & API | 3 | incident_templates (config JSONB), webhooks (delivery_log JSONB), api_keys |
| **Total** | **12** | |

---

## Key Design Decisions

1. **Monitor configuration inline on component** — `components.monitor` JSONB stores the full monitor config (type, target, method, expected status, regions, thresholds). No separate `monitors` table. Each component has at most one monitor, so the 1:1 relationship is naturally a column. Adding a new monitor type (e.g., gRPC, GraphQL) requires no schema change.

2. **Uptime stats inline on component** — `components.uptime_stats` stores pre-computed uptime percentages and average response times. These are updated by a background worker aggregating `monitor_checks`. The public page reads components in a single query without joining or aggregating checks.

3. **Incident updates as JSONB array** — `incidents.updates` stores the full investigation → identified → monitoring → resolved timeline as a JSONB array. Most incidents have 2-6 updates. The public page displays the complete timeline from a single row read. Trade-off: per-update queries require JSONB extraction.

4. **Post-mortem inline on incident** — `incidents.post_mortem` JSONB stores the full post-mortem (summary, root cause, impact, timeline, corrective actions). No separate table. Post-mortem is a 1:1 with the incident and is typically read alongside the incident.

5. **AI outputs inline on incident** — `incidents.ai` stores draft updates, suggested severity, suggested components, and anomaly correlation data. New AI features are added to this JSONB without migrations.

6. **Subscriber contact as JSONB** — `subscribers.contact` stores channel-specific contact information (email address, phone number, webhook URL, Slack webhook). The `channel_type` discriminator determines the shape. Adding a new notification channel (e.g., Discord, Telegram) requires no schema change.

7. **Component groups as a column** — `components.group_name` is a simple TEXT column instead of a separate `component_groups` table. Grouping is done by `GROUP BY group_name` in queries. This trades a foreign key for simplicity — appropriate for status pages where groups are display-only and have no independent attributes.

8. **12 tables total** — Compared to 21 in the normalized model. The JSONB approach trades constraint enforcement for development speed, particularly effective for the many type-specific and channel-specific fields in a status page platform.
