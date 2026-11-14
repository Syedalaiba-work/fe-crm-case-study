# Case Study: Insurance Call-Center CRM & Voice Script Engine

> **Note:** Code is proprietary to a former employer and is not included. This
> write-up describes my role and the engineering involved.

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![Bun](https://img.shields.io/badge/Bun-000000?logo=bun&logoColor=white)
![NestJS](https://img.shields.io/badge/NestJS-E0234E?logo=nestjs&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?logo=react&logoColor=61DAFB)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-06B6D4?logo=tailwindcss&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?logo=amazonwebservices&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?logo=githubactions&logoColor=white)

## 1. Overview

A multi-tenant CRM for an insurance call-center workflow, built around a
two-stage sales flow: a **verifier** confirms a lead's details, then hands the
live call to a **licensed agent** who can sell. Leads arrive already qualified
by a rule-based voice script engine. It asks a short set of recorded questions,
classifies each answer, and transfers interested leads to the verifier queue.

This was a team project. I owned the parts of the workflow listed below and
contributed alongside the wider team on the rest.

## 2. What I worked on

**Built by me:**

- **Verifier and licensed-agent workspaces** with live call transfers between
  the two stages
- **Pre-filled verification forms**, populated from the answers the voice
  script captured, so verifiers confirm the details instead of re-asking
- **Hot-lead alerts** that surface qualified leads to the floor in real time,
  with **routing only to agents licensed for the lead's state**
- **Signed event forwarding** between services: HMAC-signed payloads,
  idempotent upserts on the receiving side, and retry with exponential backoff
  so a lead is never silently dropped

**Team work I contributed alongside:**

- Suppression (do-not-call) list, web lead intake and versioned question sets
- QA flags, seat and permission management, and agent onboarding with
  licence checks
- The rule-based voice script: recorded prompts, speech recognition,
  classification of answers into branches, and a recording disclosure at the
  start of every call

## 3. Architecture

```mermaid
flowchart LR
    A[Lead] --> B[Dialer]
    B --> C[Voice Script Engine]
    C -->|qualified lead + answers| D[Verifier Desk]
    D -->|live transfer| E[Licensed Agent Desk]
    C -.->|signed events| F[(CRM Database)]
    D --> F
    E --> F
```

The voice script engine never writes to the CRM directly. Its results travel
as signed events, so the CRM has a single, authenticated way in no matter what
called the lead.

## 4. Engineering practices

- **Multi-tenancy:** every business record is scoped to an organization, and
  the database enforces that boundary rather than relying on each query to
  remember it.
- **Infrastructure as code:** containerised services, a pinned and
  reproducible toolchain, and build and deployment pipelines defined as code.
- **Continuous integration:** lint, type checks, builds and integration tests
  run against a real database on every change.
- **Latency budgets for real-time voice:** the gap between a caller finishing a
  sentence and the next prompt starting was treated as a hard budget, measured
  and recorded on every turn.

## 5. Compliance areas the system was designed for

- **Suppression lists:** anyone who asks not to be called again is excluded
  before dialing.
- **Recording disclosure:** played first on every call and logged per call,
  so it can be proven afterwards.
- **Licence-based routing** *(built by me)*: a lead only reaches an agent
  licensed to sell in that lead's state.
- **Consent records:** designed to support consent records.

## 6. What I learned

- **Push business rules down to where they can't be bypassed.** Rules enforced
  in the database hold for every client, including ones not written yet.
- **Delivery needs to be "at least once" plus idempotent.** Acknowledge only
  after storing, and make repeats harmless. That's simpler than trying to
  deliver exactly once.
- **Handoffs are product design.** A verifier who opens a pre-filled form spends
  the call confirming instead of re-asking, and the lead notices.
- **Compliance is a design input.** Building disclosure and suppression in
  from the start costs days. Adding them later costs far more.
