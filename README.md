# Status Page Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source status page platform that closes the gap between monitoring signals and customer-facing incident communication.

Status Page Platform is a candidate project to build incident communication, uptime tracking, and subscriber notification tooling for SaaS, DevOps, and SRE teams. It targets the lag, manual effort, and tool fragmentation that plague today's status page workflows by driving updates directly from monitoring data and AI-drafted narratives.

---

## Why Status Page Platform?

- Atlassian Statuspage starts at $79/month and Enterprise tiers reach $399+/month, yet ships no built-in monitoring — customers pay premium prices and still need a separate uptime tool.
- Standalone status page tools (Statuspage, Instatus, Sorry, Freshstatus) all rely on third-party monitoring integrations, leaving a manual lag between an outage starting and customers being informed.
- All-in-one commercial platforms (Better Stack, PagerDuty, incident.io) consolidate the stack but charge per-user pricing that becomes expensive for large teams.
- Existing open-source options have real gaps: Uptime Kuma has no subscriber notifications or REST API, Cachet's last stable release was in 2023, and OpenStatus is AGPL-3.0 (restrictive for downstream reuse).
- No tool today auto-generates status updates from monitoring signals, produces AI root-cause narratives from telemetry, or segments subscriber notifications by impact — these are open opportunities across the entire incumbent landscape.

---

## Key Features

### Component & Incident Model

- Component-based service health model with configurable status values
- Incident lifecycle management across investigating, identified, monitoring, and resolved states
- Templated incident messaging for repeatable, fast communication
- Scheduled maintenance windows with subscriber pre-notification
- Historical uptime display per component

### Notifications & Subscribers

- Subscriber notifications via email and webhook out of the box
- Slack and Microsoft Teams notification integrations
- Webhook outbound delivery for chat tools, PagerDuty, and downstream systems
- Webhook inbound receiver to auto-update component status from monitoring tools

### Monitoring & Automation

- Built-in uptime monitoring for HTTP, TCP, DNS, and SSL checks with direct status page integration
- Monitoring-to-status-page automation — status updates triggered by check results without manual intervention
- AI-generated incident update drafts from monitoring signal context

### API & Integration

- REST API with API key authentication and an OpenAPI 3.0 specification
- Custom domain with automatic SSL
- Private/password-protected status pages with SSO via SAML 2.0 / OIDC

### Backlog Capabilities

- On-call scheduling and escalation policies
- Monitoring as code via YAML / Terraform
- Post-mortem workflow with AI-assisted timeline summarisation
- Subscriber impact segmentation to notify only affected regions or tiers
- Third-party service status aggregation
- Multi-language public status pages

---

## AI-Native Advantage

Status Page Platform treats AI as a first-class layer rather than a bolt-on. Monitoring anomalies trigger AI-written status updates automatically, removing the lag between an outage starting and customers being informed. Once an incident resolves, AI analyses logs, metrics, and traces to draft a technically accurate post-mortem narrative within minutes. Subscriber impact classification segments which tiers or geographies are affected so notifications are targeted, and predictive degradation alerting flags leading indicators (rising error rates, latency spikes) before an incident becomes customer-visible. SREs can describe an incident in plain English and have it translated into structured severity, affected component, and ETA fields.

---

## Tech Stack & Deployment

The platform is designed for both self-hosted and managed deployment, following the OneUptime and Uptime Kuma precedent of containerised distribution. Integration aligns with existing operational standards: webhook (HTTP POST) for state propagation, RSS/Atom feeds for developer consumption, Prometheus and OpenTelemetry for ingesting health signals, OAuth 2.0 / SAML 2.0 for enterprise SSO, and DMARC/DKIM for reliable subscriber email delivery. The REST API ships with an OpenAPI 3.0 specification so SDKs and integrations can be generated against a stable contract.

---

## Market Context

The incident management and status page market sits within the broader IT operations management market, valued at roughly $8 billion globally in 2025 and growing at ~15% CAGR, driven by SLA expectations in SaaS and cloud-native businesses. Paid SaaS tiers start at $20–$29/month for small teams; enterprise contracts with Statuspage and PagerDuty range from $400–$2,000+/month. Primary buyers are engineering teams at SaaS companies managing customer-facing SLAs, DevOps and SRE teams needing integrated monitoring-to-status-page automation, customer success teams communicating proactively during incidents, and enterprise IT departments managing internal service status.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
