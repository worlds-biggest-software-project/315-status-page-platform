# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Status Page Platform · Created: 2026-05-24

## Philosophy

A status page platform manages a clear hierarchy: organisations operate status pages, status pages display components (services), components have status values that change during incidents, incidents progress through lifecycle states with updates, subscribers receive notifications about incidents and maintenance, and uptime monitors feed component status changes automatically. A normalized relational model gives each concept its own table with explicit foreign keys, enabling database-level enforcement of the ownership chain and clean separation between configuration (components, monitors), runtime state (incidents, checks), communication (subscribers, notifications), and AI enrichment (draft updates, post-mortem narratives).

This mirrors the conceptual model that SRE teams already carry: a status page is a public view of service health, components are the services being monitored, incidents are events that affect those services, and updates are the communication timeline. Each maps to a table with typed columns and constraints.

The normalized model also supports the ITIL 4 incident taxonomy cleanly: severity levels, impact scope, escalation state, and SLA target tracking are all explicit columns rather than metadata buried in JSONB.

**Best for:** Teams building a production-grade status page platform where data integrity across the component→incident→update→notification pipeline is critical, where ITIL-aligned incident severity and SLA tracking need explicit columns, and where uptime monitoring, subscriber management, and post-mortem workflows are first-class features.

**Trade-offs:**
- **Pro:** Database-enforced relationships from page to component to incident to update
- **Pro:** Subscriber notifications as explicit rows enable delivery tracking and retry logic
- **Pro:** Monitor checks as typed rows enable uptime percentage calculation
- **Pro:** ITIL-aligned severity levels and SLA targets as typed columns
- **Con:** 22 tables — complex schema
- **Con:** High join count for "show me the current status of all components with active incidents"
- **Con:** Monitor type-specific configuration requires either wide columns or a metadata table
- **Con:** Notification channel configuration varies by channel type

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1 | REST API specification for status page management |
| RFC 4287 (Atom) | Incident feed in Atom format |
| RFC 3339 | Timestamps in API responses and webhook payloads |
| RFC 7807 | Problem details format for API errors |
| Standard Webhooks | HMAC-SHA256 signed outbound webhooks |
| OpenMetrics / Prometheus | Ingest uptime metrics via /metrics endpoint |
| OTLP (OpenTelemetry) | Ingest traces/metrics for AI incident correlation |
| SAML 2.0 / OIDC | SSO for private status pages |
| DMARC / DKIM / SPF | Email authentication for subscriber notifications |
| ITIL 4 | Incident severity taxonomy (P1–P4) |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Organisations & Status Pages

```sql
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
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, user_id)
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
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
    require_sso BOOLEAN NOT NULL DEFAULT FALSE,
    branding_logo_url TEXT,
    branding_favicon_url TEXT,
    branding_accent_color TEXT,
    custom_css TEXT,
    custom_header_html TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    locale TEXT NOT NULL DEFAULT 'en',
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
CREATE TABLE component_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES status_pages(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    sort_order INT NOT NULL DEFAULT 0,
    is_collapsed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comp_groups ON component_groups(page_id, sort_order);

CREATE TABLE components (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES status_pages(id) ON DELETE CASCADE,
    group_id UUID REFERENCES component_groups(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'operational' CHECK (status IN (
        'operational', 'degraded_performance', 'partial_outage',
        'major_outage', 'under_maintenance'
    )),
    sort_order INT NOT NULL DEFAULT 0,
    is_visible BOOLEAN NOT NULL DEFAULT TRUE,
    start_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_components_page ON components(page_id, sort_order);
CREATE INDEX idx_components_group ON components(group_id);
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
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    identified_at TIMESTAMPTZ,
    monitoring_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_incidents_page ON incidents(page_id, created_at DESC);
CREATE INDEX idx_incidents_status ON incidents(page_id, status) WHERE status != 'resolved';
CREATE INDEX idx_incidents_active ON incidents(page_id, started_at DESC)
    WHERE status NOT IN ('resolved', 'postmortem');

CREATE TABLE incident_components (
    incident_id UUID NOT NULL REFERENCES incidents(id) ON DELETE CASCADE,
    component_id UUID NOT NULL REFERENCES components(id) ON DELETE CASCADE,
    component_status TEXT NOT NULL DEFAULT 'degraded_performance' CHECK (component_status IN (
        'degraded_performance', 'partial_outage', 'major_outage', 'under_maintenance'
    )),
    PRIMARY KEY (incident_id, component_id)
);

CREATE TABLE incident_updates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id UUID NOT NULL REFERENCES incidents(id) ON DELETE CASCADE,
    status TEXT NOT NULL,
    body TEXT NOT NULL,
    is_ai_generated BOOLEAN NOT NULL DEFAULT FALSE,
    ai_model_version TEXT,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_incident_updates ON incident_updates(incident_id, created_at DESC);

CREATE TABLE incident_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    title_template TEXT NOT NULL,
    body_template TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'minor',
    component_ids UUID[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);
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
CREATE INDEX idx_maintenance_upcoming ON scheduled_maintenances(page_id, scheduled_start)
    WHERE status = 'scheduled';

CREATE TABLE maintenance_components (
    maintenance_id UUID NOT NULL REFERENCES scheduled_maintenances(id) ON DELETE CASCADE,
    component_id UUID NOT NULL REFERENCES components(id) ON DELETE CASCADE,
    PRIMARY KEY (maintenance_id, component_id)
);

CREATE TABLE maintenance_updates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    maintenance_id UUID NOT NULL REFERENCES scheduled_maintenances(id) ON DELETE CASCADE,
    status TEXT NOT NULL,
    body TEXT NOT NULL,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Uptime Monitors

```sql
CREATE TABLE monitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    component_id UUID NOT NULL REFERENCES components(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    monitor_type TEXT NOT NULL CHECK (monitor_type IN (
        'http', 'tcp', 'dns', 'ssl', 'icmp'
    )),
    target TEXT NOT NULL,
    interval_seconds INT NOT NULL DEFAULT 60,
    timeout_seconds INT NOT NULL DEFAULT 10,
    regions TEXT[] NOT NULL DEFAULT '{us-east-1}',
    http_method TEXT DEFAULT 'GET',
    http_expected_status INT DEFAULT 200,
    http_headers TEXT[],
    http_body TEXT,
    dns_record_type TEXT,
    dns_expected_value TEXT,
    ssl_check_expiry BOOLEAN DEFAULT TRUE,
    ssl_expiry_warn_days INT DEFAULT 14,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    auto_update_component BOOLEAN NOT NULL DEFAULT TRUE,
    consecutive_failures_threshold INT NOT NULL DEFAULT 3,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_monitors_component ON monitors(component_id);
CREATE INDEX idx_monitors_active ON monitors(is_active, interval_seconds) WHERE is_active = TRUE;

CREATE TABLE monitor_checks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    monitor_id UUID NOT NULL REFERENCES monitors(id) ON DELETE CASCADE,
    status TEXT NOT NULL CHECK (status IN ('up', 'down', 'degraded')),
    response_time_ms INT,
    status_code INT,
    error_message TEXT,
    region TEXT NOT NULL,
    checked_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (checked_at);

CREATE INDEX idx_checks_monitor ON monitor_checks(monitor_id, checked_at DESC);
CREATE INDEX idx_checks_status ON monitor_checks(monitor_id, status, checked_at DESC);
```

---

## Subscribers & Notifications

```sql
CREATE TABLE subscribers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES status_pages(id) ON DELETE CASCADE,
    email TEXT,
    phone TEXT,
    webhook_url TEXT,
    slack_webhook_url TEXT,
    channel_type TEXT NOT NULL CHECK (channel_type IN (
        'email', 'sms', 'webhook', 'slack', 'teams', 'rss'
    )),
    is_confirmed BOOLEAN NOT NULL DEFAULT FALSE,
    confirmation_token TEXT,
    unsubscribe_token TEXT NOT NULL DEFAULT gen_random_uuid()::TEXT,
    component_ids UUID[],
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscribers_page ON subscribers(page_id);
CREATE INDEX idx_subscribers_email ON subscribers(email) WHERE email IS NOT NULL;

CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    incident_id UUID REFERENCES incidents(id),
    maintenance_id UUID REFERENCES scheduled_maintenances(id),
    notification_type TEXT NOT NULL CHECK (notification_type IN (
        'incident_created', 'incident_updated', 'incident_resolved',
        'maintenance_scheduled', 'maintenance_started', 'maintenance_completed'
    )),
    channel TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'sent', 'delivered', 'failed', 'bounced'
    )),
    error_message TEXT,
    sent_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_subscriber ON notifications(subscriber_id, created_at DESC);
CREATE INDEX idx_notifications_incident ON notifications(incident_id);
CREATE INDEX idx_notifications_status ON notifications(status) WHERE status IN ('pending', 'failed');
```

---

## Post-Mortems & AI

```sql
CREATE TABLE post_mortems (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id UUID NOT NULL REFERENCES incidents(id) ON DELETE CASCADE UNIQUE,
    title TEXT NOT NULL,
    summary TEXT,
    root_cause TEXT,
    impact_description TEXT,
    timeline TEXT,
    corrective_actions TEXT,
    is_published BOOLEAN NOT NULL DEFAULT FALSE,
    is_ai_generated BOOLEAN NOT NULL DEFAULT FALSE,
    ai_model_version TEXT,
    published_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_draft_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id UUID NOT NULL REFERENCES incidents(id) ON DELETE CASCADE,
    draft_type TEXT NOT NULL CHECK (draft_type IN (
        'incident_update', 'post_mortem', 'maintenance_notice'
    )),
    draft_content TEXT NOT NULL,
    context_sources TEXT[] NOT NULL DEFAULT '{}',
    model_name TEXT NOT NULL,
    model_version TEXT,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'accepted', 'edited', 'rejected'
    )),
    accepted_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_drafts ON ai_draft_queue(incident_id, created_at DESC);
```

---

## Webhooks & API Keys

```sql
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
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Current status of all components on a page

```sql
SELECT c.name, c.status, c.description, cg.name AS group_name
FROM components c
LEFT JOIN component_groups cg ON cg.id = c.group_id
WHERE c.page_id = 'page-uuid'
  AND c.is_visible = TRUE
ORDER BY cg.sort_order NULLS LAST, c.sort_order;
```

### Active incidents with affected components

```sql
SELECT i.title, i.status, i.severity, i.started_at,
       ARRAY_AGG(c.name) AS affected_components
FROM incidents i
JOIN incident_components ic ON ic.incident_id = i.id
JOIN components c ON c.id = ic.component_id
WHERE i.page_id = 'page-uuid'
  AND i.status NOT IN ('resolved', 'postmortem')
GROUP BY i.id
ORDER BY i.started_at DESC;
```

### Uptime percentage for a component (last 30 days)

```sql
SELECT m.name,
       COUNT(*) FILTER (WHERE mc.status = 'up') * 100.0 / COUNT(*) AS uptime_pct,
       AVG(mc.response_time_ms) AS avg_response_ms
FROM monitors m
JOIN monitor_checks mc ON mc.monitor_id = m.id
WHERE m.component_id = 'component-uuid'
  AND mc.checked_at >= now() - INTERVAL '30 days'
GROUP BY m.id, m.name;
```

### Pending notifications for retry

```sql
SELECT n.id, n.channel, n.notification_type, n.error_message,
       s.email, s.webhook_url, n.created_at
FROM notifications n
JOIN subscribers s ON s.id = n.subscriber_id
WHERE n.status = 'failed'
  AND n.created_at >= now() - INTERVAL '24 hours'
ORDER BY n.created_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Org & Pages | 4 | organisations, org_members, users, status_pages |
| Components | 2 | component_groups, components |
| Incidents | 4 | incidents, incident_components, incident_updates, incident_templates |
| Maintenance | 3 | scheduled_maintenances, maintenance_components, maintenance_updates |
| Monitors | 2 | monitors, monitor_checks (partitioned) |
| Subscribers | 2 | subscribers, notifications |
| Post-Mortems & AI | 2 | post_mortems, ai_draft_queue |
| Integrations | 2 | webhooks, api_keys |
| **Total** | **21** | |

---

## Key Design Decisions

1. **Component status as a column, not derived** — `components.status` stores the current status directly. While it could be derived from active incidents, an explicit column enables fast public page rendering (single table scan) and supports manual status overrides when monitoring disagrees with reality.

2. **Incident-component many-to-many** — `incident_components` links incidents to affected components with a per-component status. An incident can affect multiple components at different severity levels (API: major outage, Dashboard: degraded performance).

3. **Incident updates as separate rows** — `incident_updates` stores each status update (investigating → identified → monitoring → resolved) as a row. This provides the timeline view that every status page displays and enables AI-generated vs. human-authored distinction.

4. **Monitor checks partitioned by time** — `monitor_checks` is partitioned by `checked_at` for efficient uptime percentage queries over time ranges. Old partitions can be archived without affecting active monitoring.

5. **Subscribers with channel-type discrimination** — `subscribers` uses a `channel_type` discriminator with nullable columns for each channel's address (email, phone, webhook_url). Component-level subscriptions use `component_ids` array to filter notifications.

6. **Notifications as delivery records** — `notifications` tracks each delivery attempt with status (pending → sent → delivered/failed/bounced). Failed notifications can be retried. This enables delivery analytics and SLA tracking for notification latency.

7. **Post-mortems as first-class entities** — `post_mortems` is a separate table linked 1:1 to incidents. Fields match the industry-standard post-mortem structure: summary, root cause, impact, timeline, corrective actions. AI-generated drafts are flagged.

8. **AI draft queue** — `ai_draft_queue` stores AI-generated incident updates and post-mortem drafts with status tracking (pending → accepted/edited/rejected). This enables measuring AI draft acceptance rates and improving prompts over time.
