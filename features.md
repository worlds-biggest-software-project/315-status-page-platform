# Status Page Platform — Feature & Functionality Survey

> Candidate #315 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Atlassian Statuspage | Hosted SaaS | Commercial (proprietary) | https://www.atlassian.com/software/statuspage |
| Better Stack | Hosted SaaS | Commercial (proprietary) | https://betterstack.com |
| Instatus | Hosted SaaS | Commercial, freemium | https://instatus.com |
| OneUptime | Open-source / SaaS | MIT / hosted tier | https://oneuptime.com |
| Uptime Kuma | Open-source, self-hosted | MIT | https://github.com/louislam/uptime-kuma |
| incident.io | Hosted SaaS | Commercial, freemium | https://incident.io |
| Hyperping | Hosted SaaS | Commercial, freemium | https://hyperping.com |
| Freshstatus | Hosted SaaS (Freshworks) | Commercial, free tier | https://www.freshworks.com/freshstatus/ |
| OpenStatus | Open-source / SaaS | Open-source / hosted tier | https://www.openstatus.dev |
| Cachet | Open-source, self-hosted | BSD-3-Clause | https://cachethq.io |

---

## Feature Analysis by Solution

### Atlassian Statuspage

**Core features**
- Component-based service health model (up to 25 components on free tier)
- Multiple incident states: investigating, identified, monitoring, resolved
- Scheduled maintenance windows with advance subscriber notification
- Real-time system metrics display (response time, uptime) via Pingdom, New Relic, Librato, Datadog integrations
- Subscriber notifications via email, SMS, Slack, Microsoft Teams, and webhook
- Component-level subscriptions so users only receive alerts for services they care about
- Custom branding, custom domain, and custom CSS/HTML (paid plans)
- REST API for programmatic updates (api key authentication)
- Historical uptime display per component
- Page access controls: public, private (SSO), or audience-specific

**Differentiating features**
- Deep integration with Jira and Opsgenie within the Atlassian ecosystem
- Audience-specific pages: show different status to internal vs. external users
- Bulk subscriber import via CSV

**UX patterns**
- Two-panel management UI: left-nav components, right-side incident feed
- Guided incident creation wizard with template support
- Progressive disclosure: basic view for readers, detailed timeline view on click
- Email/SMS notification preview before sending

**Integration points**
- Atlassian suite (Jira, Opsgenie, Confluence)
- Pingdom, New Relic, Librato, Datadog for metrics
- Slack, Microsoft Teams, webhooks for notifications
- PagerDuty via webhook
- Zapier

**Known gaps**
- No built-in uptime monitoring — must pair with external tool
- Custom domain and full CSS customisation locked to high-tier paid plans ($399+/month)
- No AI-generated incident descriptions or post-mortems
- No native on-call scheduling
- Limited metric types (only numeric time-series from supported integrations)
- Single-purpose tool: no log management or APM

**Licence / IP notes**
- Proprietary SaaS; no self-hosting option. Public REST API is documented and freely usable.

---

### Better Stack

**Core features**
- Unified platform: uptime monitoring, incident management, on-call scheduling, log management, and status pages
- Monitors HTTP/HTTPS, TCP, ping, SSL, domain expiry, DNS, SMTP/IMAP/POP3, cron heartbeats
- Status page with public and private modes, custom domain, custom branding
- Pre-built response time charts and custom metric widgets on the status page
- On-call schedules with rotation, escalation policies, and multi-channel alerting
- Subscriber notifications via email; RSS and webhook supported
- Incident collaboration with team mentions, comment threads, and status timeline
- Scheduled maintenance with auto-notifications
- Integrations with Datadog, New Relic, Grafana, Prometheus, Zabbix, AWS, Azure, Google Cloud

**Differentiating features**
- All-in-one stack eliminates tool fragmentation (replaces Statuspage + PagerDuty + log tool)
- AI-powered incident management: suggest on-call responders, draft status updates
- Multi-region monitoring across global locations
- Log ingestion and search alongside status page in same UI

**UX patterns**
- Clean dashboard with colour-coded monitor health cards
- One-tap bulk incident acknowledgement to silence phone alerts during mass incidents
- Onboarding wizard that activates basic monitoring + status page in under 5 minutes
- Dark/light mode support

**Integration points**
- Slack, Microsoft Teams, PagerDuty, OpsGenie, Discord, Telegram, email, SMS, phone calls, webhooks
- Prometheus, Grafana, Datadog, New Relic, Zabbix cloud integrations
- Heroku add-on available
- REST API with Bearer token authentication

**Known gaps**
- Status page subscriber management less granular than Statuspage (no component-level subscriptions on lower tiers)
- Log management still maturing vs. dedicated tools (Datadog, Loggly)
- Less customisation of public status page appearance vs. Instatus

**Licence / IP notes**
- Proprietary SaaS. No open-source codebase.

---

### Instatus

**Core features**
- Beautifully designed, fast-loading public status pages
- Component groups and sub-components for complex service hierarchies
- Incident templates for repeatable communication
- Subscriber notifications via email, SMS, Slack, Microsoft Teams, webhook
- Component-level subscription management
- Custom domain with free SSL
- Webhook reception for triggering status updates from external monitoring tools
- Zapier integration
- REST API (OpenAPI 3.0.3) with API key authentication
- Multi-language page support

**Differentiating features**
- Recognised as having the best-in-class UI and page load performance in the market
- Webhook ingestion: receive webhooks from monitoring tools to auto-update component status
- Monitoring integrations with UptimeRobot, Pingdom, PagerDuty, Datadog, Checkly, and more

**UX patterns**
- Minimalist public page design with smooth animations
- Simple three-step incident creation flow
- Customisable colour themes without requiring CSS knowledge

**Integration points**
- 20+ uptime monitoring tools (Datadog, New Relic, UptimeRobot, StatusCake, Site24x7, Checkly, etc.)
- Zapier for workflow automation
- Slack, MS Teams for team notifications
- Microsoft Power Automate connector available

**Known gaps**
- No built-in uptime monitoring (relies entirely on third-party integrations)
- No native on-call scheduling or incident management workflows
- No post-mortem tooling
- Free plan limited in subscribers and pages

**Licence / IP notes**
- Proprietary SaaS; OpenAPI spec published at https://github.com/instatushq/openapi.

---

### OneUptime

**Core features**
- Full observability platform: monitoring, status pages, incident management, on-call, logs, APM, traces, error tracking
- Unlimited public and private status pages on all plans
- Custom branding, custom domain, free SSL, custom HTML/CSS/JS
- Component groups with historical uptime percentages
- Unlimited subscribers (email, SMS)
- Scheduled maintenance with pre/during/post subscriber notifications
- Automated status page updates from monitoring results (no manual intervention needed)
- Custom incident severity levels and states
- Post-mortem workflow with structured templates
- On-call scheduling with escalation policies
- OpenAPI 3.0 specification published

**Differentiating features**
- Fully open-source (MIT) — self-host the entire stack at zero licence cost
- Monitoring-to-status-page automation is native, not an add-on
- Replaces Datadog + PagerDuty + Statuspage in one self-hostable platform

**UX patterns**
- Dashboard modelled on SRE workflows: monitors → incidents → status pages as a lifecycle
- Dark-mode admin UI
- Docker Compose and Kubernetes Helm chart for self-hosted deployment

**Integration points**
- REST API with API key authentication; OpenAPI spec at https://oneuptime.com/reference/openapi
- Slack, Microsoft Teams, PagerDuty webhooks
- Prometheus metrics ingestion
- Public status page API for embedding status data in third-party apps

**Known gaps**
- Self-hosting requires DevOps competence (multiple services, Docker/Kubernetes)
- UI polish and onboarding experience lags behind commercial-only tools
- Community support only (no paid SLA for self-hosted users)
- Mobile app absent

**Licence / IP notes**
- MIT licence. Full source at https://github.com/OneUptime/oneuptime.

---

### Uptime Kuma

**Core features**
- Self-hosted uptime monitoring for HTTP/HTTPS URLs, TCP ports, DNS, Docker containers, Steam game servers, database endpoints
- Built-in status page generator (multiple pages per instance)
- 20-second minimum check interval
- 90+ notification channels (Slack, Telegram, Discord, email SMTP, SMS, PagerDuty, and many more)
- SSL certificate monitoring with expiry alerts
- Real-time status page updates via WebSocket (no page refresh needed)
- Uptime percentage and incident history display
- Maintenance window scheduling

**Differentiating features**
- Zero cost (self-hosted, MIT licence)
- Docker-ready with minimal resource footprint
- WebSocket-powered live status updates — no polling required

**UX patterns**
- Consumer-grade web UI despite being self-hosted
- One-page dashboard with at-a-glance status cards
- Status page accessible without authentication; admin panel behind login

**Integration points**
- 90+ notification integrations (widest range in the category)
- No official API (community HTTP API projects exist)
- No native subscriber management (email subscriptions not built-in)

**Known gaps**
- No managed hosting option — always self-hosted
- No subscriber email/SMS notification system
- No incident management or post-mortem workflows
- No on-call scheduling
- No metrics display on status page
- No REST API for external automation
- Single-instance architecture; no multi-tenancy

**Licence / IP notes**
- MIT licence. Source at https://github.com/louislam/uptime-kuma.

---

### incident.io

**Core features**
- Slack-native incident lifecycle management (declare, manage, resolve entirely in Slack)
- Automated role assignment, escalations, and checklists triggered by incident creation
- Public, private, and internal status pages
- Status page updates publishable directly from Slack
- Automated status page update prompts when traffic spikes to the status page
- Subscriber notifications via email, Slack, and RSS
- Incident templates with pre-approved messaging for fast status page updates
- Post-mortem workflow with automated timeline generation
- Workflow automation engine: trigger actions on incident events without code

**Differentiating features**
- Industry-leading Slack-native experience: the entire incident workflow happens in Slack
- Automated internal alert when status page traffic spikes (indicating customers are checking)
- Status page API (REST) for programmatic management
- AI-powered features: drafted incident summaries, stakeholder suggestions based on past incidents

**UX patterns**
- Slash commands and interactive modals in Slack; no separate app required for SREs
- Status page management as a secondary surface — primarily driven from Slack
- Guided post-mortem templates with auto-populated timeline from Slack conversation

**Integration points**
- Slack (primary interface)
- PagerDuty, OpsGenie for on-call escalation
- Jira, Linear, Asana for follow-up task creation
- GitHub, GitLab for deployment context
- REST API with full webhook support (powered by Svix; HMAC-signed)
- Zapier

**Known gaps**
- Heavily Slack-dependent — less useful for organisations not on Slack
- Status page customisation limited compared to dedicated status page tools
- Pricing per user becomes expensive for large teams
- No built-in uptime monitoring

**Licence / IP notes**
- Proprietary SaaS.

---

### Hyperping

**Core features**
- HTTP/HTTPS, API, SSL, cron/heartbeat, Playwright synthetic browser checks, TCP, ICMP, DNS monitoring
- Multi-region execution with auto-retry verification
- Status pages: public and private, custom domain, white-label branding, SSO protection, component groups, multi-language
- On-call scheduling with timezone awareness, auto-rotation, multi-step escalation
- REST API for status page embedding and custom integrations
- Notification channels: Slack, MS Teams, PagerDuty, OpsGenie, Discord, Telegram, email, SMS, phone, webhooks

**Differentiating features**
- Playwright-based synthetic browser monitoring (full user-journey simulation) included without extra cost
- Multi-language status pages natively supported
- SSO-protected private status pages without per-seat cost

**UX patterns**
- Clean, minimal dashboard; praised for ease of setup in user reviews
- Status page creation wizard with live preview
- Alert noise reduction via smart grouping of related monitor failures

**Integration points**
- PagerDuty, OpsGenie for on-call routing
- Slack, Teams, Discord, Telegram for chat notifications
- REST API (documented at https://hyperping.com)
- Custom webhooks

**Known gaps**
- Smaller company with less ecosystem breadth than Atlassian or PagerDuty
- No built-in log management or APM
- No post-mortem tooling
- No AI features as of 2026

**Licence / IP notes**
- Proprietary SaaS.

---

### Freshstatus

**Core features**
- Unlimited components and incidents on the free plan
- Email and SMS subscriber notifications (up to 1,000 subscribers free)
- Custom domain on free plan
- SAML/SSO authentication for private status pages
- Incident templates with private notes capability
- Webhook integration for incoming and outgoing events
- Slack integration for team notifications
- REST API for CRUD of incidents and scheduled maintenance
- Automated reminders to update status during ongoing incidents
- Twitter/X integration for broadcasting status updates

**Differentiating features**
- Most generous free tier in the category (unlimited components, unlimited incidents, custom domain)
- Native Freshworks ecosystem integration (Freshdesk, Freshservice)
- Twitter/X broadcast integration built-in

**UX patterns**
- Simple three-step setup wizard
- Admin UI consistent with Freshworks design language
- Incident update reminder pop-ups to prevent stale status pages

**Integration points**
- Freshdesk, Freshservice (native Freshworks integrations)
- Slack, webhooks
- Twitter/X
- REST API

**Known gaps**
- Advanced customisation (custom CSS, custom branding beyond basics) requires paid plans
- Limited metrics display capability
- No built-in monitoring
- Tied to Freshworks ecosystem — less attractive for non-Freshworks users
- No on-call scheduling

**Licence / IP notes**
- Proprietary SaaS under Freshworks.

---

### OpenStatus

**Core features**
- Open-source status pages and uptime monitoring in a single platform
- 28 global monitoring regions across 3 cloud providers
- Monitoring as code: YAML config, CLI, GitHub Actions, Terraform
- Status page components with component groups
- Subscriber notifications via email, RSS, and webhooks
- Custom domain, password protection, maintenance windows
- v2 REST API with pageComponents as primary data structure
- Flat pricing with unlimited team members

**Differentiating features**
- Monitoring as code via YAML/Terraform enables GitOps workflows for status page management
- Fully open-source — inspect, self-host, or contribute
- Flat per-workspace pricing (not per-seat) makes it economical for large teams

**UX patterns**
- Developer-centric: CLI and config-file approach alongside web UI
- Public roadmap and changelog visible to users

**Integration points**
- GitHub Actions
- Terraform provider
- CLI
- REST API (v2)

**Known gaps**
- Smaller ecosystem vs. established commercial tools
- No on-call scheduling
- No incident management workflow tooling
- No AI features
- Limited notification channels compared to Uptime Kuma

**Licence / IP notes**
- Open-source. Licence: AGPL-3.0. Source at https://github.com/openstatusHQ/openstatus.

---

### Cachet

**Core features**
- Self-hosted open-source status page
- Bootstrap-based responsive UI
- Metrics display with historical graphs (chart dashboard)
- Detailed incident reports with full timeline from occurrence to resolution
- Subscriber notifications via email
- Multi-language support
- REST API

**Differentiating features**
- Established open-source project with wide adoption history
- V3.0 rebuild underway (as of 2026) — architectural modernisation in progress

**UX patterns**
- Traditional web application pattern; setup requires PHP, Composer, SQL database, Redis/APC
- Clean public status page layout

**Integration points**
- REST API
- Email notifications
- Webhook support (limited)

**Known gaps**
- Last stable release in 2023; v3.0 rebuild still in progress (active gap)
- No built-in monitoring
- No Slack or Teams native integration
- No AI features
- Requires significant server administration compared to modern containerised alternatives
- No on-call scheduling

**Licence / IP notes**
- BSD-3-Clause licence. Source at https://github.com/cachethq/Cachet.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Component-based service health model with named services and current status
- Incident lifecycle states (investigating → identified → monitoring → resolved)
- Subscriber notifications via at least email
- Scheduled maintenance windows with advance notice
- Custom domain support
- Historical uptime display
- REST API for programmatic updates
- Webhook outbound notifications
- Public status page with clear current-status summary

### Differentiating Features
- Built-in uptime monitoring (eliminates need for a separate tool)
- Monitoring-to-status-page automation (no manual update lag)
- Slack-native incident workflow (incident.io)
- Monitoring as code / GitOps workflow (OpenStatus)
- Playwright synthetic browser monitoring included (Hyperping)
- On-call scheduling bundled in the same product
- Multi-region monitoring from a single platform
- Post-mortem workflow and timeline generation
- AI-generated incident summaries and stakeholder suggestions (incident.io, Better Stack)

### Underserved Areas / Opportunities
- **AI-written status updates**: only incident.io has early AI drafting; no tool auto-generates updates from monitoring signals without human review
- **Root-cause narrative generation**: post-mortem AI summarisation from logs, traces, and Slack threads is absent from all tools
- **Predictive degradation alerting**: leading-indicator detection before a customer-visible incident is entirely absent
- **Subscriber impact segmentation**: notifying only affected tiers or geographies, not all subscribers, is not available in any current tool
- **Natural language incident command**: plain-English incident description translated into structured severity/component/ETA fields — absent
- **Status-page-as-SLA evidence**: automatic generation of compliance-ready SLA reports from status page data is largely absent
- **Third-party service aggregation**: StatusGator aggregates 4,000+ third-party service statuses but no native status page tool does this
- **Mobile-first incident management**: dedicated mobile apps for SRE on-call response are limited or absent in most tools

### AI-Augmentation Candidates
- Incident update text generation from monitoring signals and log data
- Root-cause analysis narrative from correlated telemetry (logs, metrics, traces)
- Subscriber impact classification by region, tier, or service dependency graph
- Anomaly detection on metric streams feeding into predictive pre-incident alerts
- Post-mortem template population from Slack conversation and incident timeline
- Suggested status page components from infrastructure topology discovery

---

## Legal & IP Summary

All ten solutions analysed are either proprietary SaaS products or open-source under permissive/copyleft licences. The open-source projects use MIT (OneUptime, Uptime Kuma), BSD-3-Clause (Cachet), and AGPL-3.0 (OpenStatus). No evidence of patented features was found in public sources. An AI-native open-source status page platform would be free to implement standard features (component health models, subscriber notifications, REST APIs, webhook delivery) without licence conflicts, provided AGPL-3.0-licensed code from OpenStatus is not incorporated directly into a non-AGPL product. All other open-source reference implementations are permissively licensed and safe to study and adapt.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Component-based service health model with configurable status values
- Incident lifecycle management (create, update, resolve) with templated messaging
- Subscriber notifications via email and webhook
- Custom domain with automatic SSL
- REST API with API key authentication and OpenAPI 3.0 specification
- Webhook inbound receiver to auto-update component status from monitoring tools

**Should-have (v1.1)**
- Built-in uptime monitoring (HTTP, TCP, DNS, SSL) with direct status page integration
- Scheduled maintenance windows with subscriber pre-notification
- Slack and Microsoft Teams notification integrations
- Private/password-protected status pages with SSO (SAML 2.0 / OIDC)
- AI-generated incident update drafts from monitoring signal context
- Historical uptime charts per component

**Nice-to-have (backlog)**
- On-call scheduling and escalation policies
- Monitoring as code (YAML/Terraform)
- Post-mortem workflow with AI-assisted timeline summarisation
- Subscriber impact segmentation (notify only affected regions/tiers)
- Third-party service status aggregation
- Multi-language public status page
