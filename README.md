# WebApp documentation

Central documentation for the WebApp (Laravel + Vue) project: deployment, status, troubleshooting, and supporting services.

## Contents

- **[publishing-docs.md](publishing-docs.md)** – How to commit doc changes here, push WebApp-docs, and bump the `docs/` submodule in WebApp (`beta` / `main`).
- **[syncing-branches.md](syncing-branches.md)** – Reconcile `beta` and `main` after parallel commits (merge workflow and when to reset).
- **[AgentContext.md](AgentContext.md)** – Environment and conventions for AI agents.
- **[deployment.md](deployment.md)** – Deploying updates to Production and Beta (Docker), Caddy, WebAppDb, Laravel scheduler, berth meanings, rollback.
- **[deploy-status.md](deploy-status.md)** – Current deploy status, URLs, and useful commands.
- **[troubleshooting.md](troubleshooting.md)** – Composer, PHP, profile upload (413), unit tests.
- **[td-trust.md](td-trust.md)** – Network Rail data stack (STOMP → MySQL; see [`network-rail-data`](../../network-rail-data)).
- **[rtt-api.md](rtt-api.md)** – Realtime Trains API playground (`/rtt`), token storage, reference data sync, integration checklist.
- **[bus-departures.md](bus-departures.md)** – bustimes.org departure boards, PTP display rules, public wall display, integration checklist.
- **[invoicing-quotes.md](invoicing-quotes.md)** – Projects index, quote versioning, sections, PDF layout, routes checklist, deploy steps.
- **[caddy/Caddyfile](caddy/Caddyfile)** – Caddy reverse proxy and SSL config; copy to your Caddy config on the host.
