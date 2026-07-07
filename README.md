# Interchange

**Where every system meets and routes.**

Interchange is a vendor-neutral integration platform and API gateway. It is the single governed junction for all your external systems: partners never touch your core, and every request passes through one controlled point where it is authenticated, validated, rate-limited, and logged.

## Live demo

Try the fully interactive demo. Log in with one click and use the whole console: register connectors, activate systems, run jobs and watch them flow through the lifecycle, receive notifications, browse the audit trail.

**[Open the live demo](https://interchange-livid.vercel.app/demo)**

Everything you do in the demo lives only in your browser. Refresh the page and it resets to a clean state. The demo makes zero database calls, so it stays pristine for every visitor.

## What it does

- **Pluggable connectors.** A connector is a self-contained module with a small `execute` contract. Adding a connector type is a code change plus a rebuild; registering unlimited partner instances of an existing type is a pure UI action, with no redeploy.
- **Config-driven, not hardcoded.** Systems, integration types, credentials, endpoints, and schedules live in the database. Outbound credentials are encrypted at rest with AES-256-GCM; inbound API keys are hashed. Everything is managed through the admin console.
- **One governed inbound gateway.** External partners authenticate with an API key or bearer token and are confined to their own system's routes. A partner cannot reach another partner's endpoints. Everything is rate-limited and logged.
- **Provisioned-dormant by default.** A fresh deploy ships fully built but idle. Systems start in a setup state, no schedules are armed, notifications are off. Nothing runs, and nothing consumes recurring hosting quota, until a project explicitly activates it.
- **Serverless-native job engine.** Inbound work runs inline. Outbound and scheduled work drains from a Postgres-backed queue with retry, backoff, and a dead-letter queue that supports replay. No Redis, no long-running worker.

## How a request flows

```
partner request
   -> inbound gateway authenticates the caller (API key / bearer)
      and confirms the URL system code matches the caller (no cross-system access)
   -> the system must be ACTIVE (a dormant system rejects traffic)
   -> the bound connector validates the payload against its schema
   -> the connector runs; a job row records the outcome
   -> everything is logged; outbound/scheduled work drains from a queue with retry + DLQ
```

## Architecture and security

- [Architecture](docs/architecture.md): system design, the connector model, the job engine, the dormant posture.
- [Security](docs/security.md): the credential vault, inbound authentication and cross-system confinement, role-based access control, and the threat model.

## Stack

Next.js (App Router) + Drizzle ORM + Neon serverless Postgres, deployed on Vercel. TypeScript throughout, zod for validation.

## Note

This is a public showcase. The source lives in a private repository. All data shown in the demo is generic and fictional.
