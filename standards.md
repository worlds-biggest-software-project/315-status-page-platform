# Standards & API Reference

> Project: Status Page Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 20000-1 — IT Service Management**
- URL: https://www.iso.org/standard/70636.html
- Defines the process requirements for IT service management including incident management, change management, and service reporting. Specifies how organisations must manage incidents, communicate SLA attainment, and document service availability — the primary business driver for operating a status page.

**ISO/IEC 27001 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- Relevant for status page platforms handling subscriber PII (email addresses, phone numbers). Requires documented data classification, access controls, and incident response procedures. Directly governs how subscriber data and API credentials must be protected.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational standard for delegated authorisation. Required for status page platforms offering SSO integration with enterprise identity providers and for OAuth-based API authentication flows.

**RFC 7522 — SAML 2.0 Profile for OAuth 2.0**
- URL: https://www.rfc-editor.org/rfc/rfc7522
- Defines how SAML 2.0 Bearer Assertions can be exchanged for OAuth 2.0 access tokens. Essential for enterprise customers requiring SAML SSO to access private internal status pages.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://www.rfc-editor.org/rfc/rfc7807
- Defines a standardised JSON error response format (`application/problem+json`) for REST APIs. Recommended for the status page platform's REST API to return machine-readable error details with fields: `title`, `status`, `detail`, `instance`.

**RFC 3339 — Date and Time on the Internet: Timestamps**
- URL: https://www.rfc-editor.org/rfc/rfc3339
- Specifies the timestamp format (`YYYY-MM-DDTHH:MM:SSZ`) used universally in status page event timelines, incident records, and subscriber notification payloads. All platform event records should use RFC 3339 format.

**RFC 5424 — The Syslog Protocol**
- URL: https://datatracker.ietf.org/doc/html/rfc5424
- Defines structured log message format including severity levels and structured data fields. Relevant for ingesting monitoring event data into the status page engine and for producing audit logs in a standard format.

**RFC 4287 — The Atom Syndication Format**
- URL: https://datatracker.ietf.org/doc/html/rfc4287
- Defines the Atom feed format widely used by status pages to provide a machine-readable incident history feed for developer audiences. All major status page tools support Atom/RSS feeds; this is a table-stakes integration.

**RFC 9232 — Network Telemetry Framework**
- URL: https://datatracker.ietf.org/doc/rfc9232/
- IETF informational RFC defining a network telemetry framework for automated network management. Provides architectural guidance relevant to how monitoring data is collected and fed into status page automation.

**W3C Server-Sent Events (SSE)**
- URL: https://www.w3.org/TR/eventsource/
- Defines the EventSource API for streaming real-time updates from server to browser over HTTP. Relevant alternative to WebSockets for pushing live status updates to public status page visitors without polling.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de-facto standard for describing REST APIs. All major status page platforms (OneUptime, Instatus, Atlassian Statuspage, OpenStatus) publish or are migrating to OpenAPI 3.0/3.1 specifications. An AI-native status page platform should publish a versioned OpenAPI spec from day one.

**OpenMetrics (IETF / Prometheus)**
- URL: https://prometheus.io/docs/specs/om/open_metrics_spec/
- The evolving standard (targeting IETF) for transmitting cloud-native metrics, derived from the Prometheus exposition format. Specifies text and Protocol Buffers representations. Status page metric widgets sourcing data from Prometheus-compatible endpoints should support the `/metrics` OpenMetrics format.

**OpenTelemetry Protocol (OTLP)**
- URL: https://opentelemetry.io/docs/specs/otlp/
- The standardised protocol for transmitting traces, metrics, and logs between observability components. An AI-native status page platform that auto-detects incidents from telemetry signals should support OTLP ingest to correlate distributed traces and metrics with status page events.

**Standard Webhooks Specification**
- URL: https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md
- A community-driven specification for standardising webhook delivery, including HMAC-SHA256 payload signing, required headers (`webhook-id`, `webhook-timestamp`, `webhook-signature`), and retry semantics. Adopted by incident.io (via Svix) and others. The status page platform's outbound webhook delivery should conform to this specification.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/draft/2020-12/schema
- Standard for describing and validating JSON data structures. Used to define and document the data models for incidents, components, subscribers, and metrics payloads in the platform's REST API and webhook events.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) / OpenID Connect 1.0**
- URL (OIDC): https://openid.net/specs/openid-connect-core-1_0.html
- OAuth 2.0 for API authorisation; OIDC for user identity layer on top of OAuth. Required for enterprise SSO integration with identity providers (Okta, Azure AD, Google Workspace) to gate access to private status pages.

**SAML 2.0**
- URL: https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
- XML-based SSO standard widely required by enterprise customers. Major status page platforms (Freshstatus, Instatus, Hyperping) offer SAML SSO for private page access. Must-have for enterprise sales.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- Defines the ten most critical security risks for REST APIs including broken object level authorisation, excessive data exposure, and injection. The status page API must be designed and audited against these risks.

**DMARC (RFC 7489) / DKIM (RFC 6376) / SPF (RFC 7208)**
- URLs: https://datatracker.ietf.org/doc/html/rfc7489, https://datatracker.ietf.org/doc/html/rfc6376, https://datatracker.ietf.org/doc/html/rfc7208
- Email authentication standards required to ensure transactional subscriber notification emails (incident alerts, maintenance notices) reliably reach inboxes. Platforms sending notification emails must configure DMARC, DKIM, and SPF for their sending domains.

**HMAC-SHA256 (FIPS PUB 198-1)**
- URL: https://csrc.nist.gov/publications/detail/fips/198/1/final
- The NIST standard for keyed-hash message authentication. Used to sign outbound webhook payloads (per Standard Webhooks spec) so recipient systems can verify authenticity. Standard across all major status page webhook implementations.

---

### ITIL / Service Management Frameworks

**ITIL 4 — Incident Management Practice**
- URL: https://www.axelos.com/certifications/itil-service-management
- The widely adopted IT service management framework defining incident severity levels (P1–P4), escalation paths, SLA targets, and post-incident review processes. A status page platform should model its incident severity taxonomy and SLA reporting on ITIL 4 definitions.

---

## Similar Products — Developer Documentation & APIs

### Atlassian Statuspage

- **Description:** Market-leading hosted status page SaaS with subscriber notifications, component health model, and metric display. Used by thousands of SaaS companies.
- **API Documentation:** https://developer.statuspage.io/
- **SDKs/Libraries:** No official SDK; community libraries exist on GitHub. `statuspage-ruby` and `statuspage-python` are community-maintained.
- **Developer Guide:** https://support.atlassian.com/statuspage/docs/what-are-the-different-apis-under-statuspage/
- **Standards:** REST/JSON; no published OpenAPI spec
- **Authentication:** API key via `Authorization` header

---

### Better Stack (Uptime)

- **Description:** All-in-one uptime monitoring, incident management, on-call scheduling, and status pages. REST API covers monitors, incidents, status pages, and on-call resources.
- **API Documentation:** https://betterstack.com/docs/uptime/api/getting-started-with-uptime-api/
- **SDKs/Libraries:** No official SDK; REST API usable with any HTTP client
- **Developer Guide:** https://betterstack.com/docs/uptime/start/
- **Standards:** REST/JSON
- **Authentication:** Bearer token (`Authorization: Bearer $TOKEN`); global and team-scoped tokens available

---

### OneUptime

- **Description:** Open-source full observability platform (monitoring, status pages, incident management, on-call, APM, logs, traces). Publicly published OpenAPI spec.
- **API Documentation:** https://oneuptime.com/docs/api-reference/api-reference
- **API Reference Portal:** https://oneuptime.com/reference/
- **OpenAPI Specification:** https://oneuptime.com/reference/openapi
- **SDKs/Libraries:** No official SDK; OpenAPI spec enables code generation via openapi-generator
- **Developer Guide:** https://oneuptime.com/docs/status-pages/public-api
- **Standards:** REST/JSON, OpenAPI 3.0
- **Authentication:** API key in request header

---

### Instatus

- **Description:** Fast, beautifully designed hosted status pages with REST API, webhook inbound/outbound, and 20+ monitoring tool integrations.
- **API Documentation:** https://instatus.com/help/api
- **OpenAPI Specification:** https://github.com/instatushq/openapi
- **SDKs/Libraries:** Microsoft Power Automate connector; no official SDK
- **Developer Guide:** https://instatus.com/blog/statuspage-api
- **Standards:** REST/JSON, OpenAPI 3.0.3
- **Authentication:** API key via `Authorization` header

---

### incident.io

- **Description:** Slack-native incident management platform with status pages, workflow automation, and AI-assisted incident drafting. Used by Netflix, Etsy, Intercom.
- **API Documentation:** https://api-docs.incident.io/
- **Webhooks:** https://api-docs.incident.io/tag/Webhooks/
- **SDKs/Libraries:** No official SDK; community Postman collection available
- **Developer Guide:** https://docs.incident.io/
- **Standards:** REST/JSON; HMAC-signed webhooks via Svix (Standard Webhooks compatible)
- **Authentication:** API key; webhook signatures via HMAC-SHA256

---

### Hyperping

- **Description:** Uptime monitoring and status page platform with Playwright synthetic browser checks, multi-language pages, and SSO-protected private status pages.
- **API Documentation:** https://hyperping.com/docs
- **SDKs/Libraries:** No official SDK
- **Developer Guide:** https://hyperping.com/docs
- **Standards:** REST/JSON
- **Authentication:** API key

---

### OpenStatus

- **Description:** Open-source status page and uptime monitoring platform with monitoring-as-code support (YAML, Terraform, GitHub Actions, CLI). AGPL-3.0 licensed.
- **API Documentation:** https://docs.openstatus.dev/
- **GitHub:** https://github.com/openstatusHQ/openstatus
- **SDKs/Libraries:** CLI tool; Terraform provider; no language SDKs
- **Developer Guide:** https://www.openstatus.dev/guides/best-opensource-status-page-2026
- **Standards:** REST/JSON, OpenAPI (v2 in development)
- **Authentication:** API key

---

### Uptime Kuma

- **Description:** Open-source self-hosted monitoring tool with basic status page generation. No official REST API; WebSocket-based real-time updates.
- **API Documentation:** No official API (community reverse-engineered API projects at GitHub)
- **GitHub:** https://github.com/louislam/uptime-kuma
- **SDKs/Libraries:** Community: `uptime-kuma-api` Python library (unofficial)
- **Developer Guide:** https://github.com/louislam/uptime-kuma/wiki/Status-Page
- **Standards:** No formal API standard; WebSocket for real-time
- **Authentication:** Session-based (web login only)

---

## Notes

**Emerging: Monitoring as Code**
OpenStatus's YAML/Terraform approach and its v2 API restructuring around `pageComponents` as the primary resource model represents an emerging pattern — treating status page configuration as version-controlled infrastructure. This pattern is likely to become more common as platform engineering practices mature.

**Emerging: AI-native incident communication**
Jira Service Management introduced native public status pages with AI auto-drafting in 2026, and incident.io ships AI-suggested stakeholder lists. The gap between monitoring telemetry and AI-generated customer-facing prose is the largest underserved area across all current tools.

**Webhook standardisation**
The Standard Webhooks specification (https://github.com/standard-webhooks/standard-webhooks) is gaining traction as a cross-vendor standard for signed webhook delivery. Building outbound webhooks to this spec from day one ensures compatibility with webhook tooling ecosystems (Hookdeck, Svix, Zapier).

**No dedicated ISO or IEEE standard for status page data models**
There is no ISO or IEEE standard that specifically governs public status page data models or incident communication formats. The closest are ISO/IEC 20000-1 (IT service management process) and ITIL 4 (incident severity taxonomy). A new open-source platform has an opportunity to define a de-facto open standard through a well-documented OpenAPI specification and a public data model.
