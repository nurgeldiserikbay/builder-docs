# BuildPath — guide for Claude Code

Implement the product from the normative specifications below. Do not treat the landing page as evidence that a feature is already implemented.

## Read in this order

1. docs/PRODUCT_REQUIREMENTS.md
2. docs/TECHNICAL_SPECIFICATION.md
3. docs/IMPLEMENTATION_ROADMAP.md
4. docs/product-blueprint.md

The product requirements define user-visible behavior. The technical specification defines the baseline architecture and safety boundaries. The roadmap defines delivery order. If two documents conflict, prefer the more specific requirement, record the conflict in a decision note and do not silently remove a user capability.

## Product outcome

A person with a phone and internet connection can describe a website or application, understand recommended options, see an evolving result, revise it, stop at any stage or continue an existing project from a chosen stage. The mobile website and Telegram bot/Mini App share project state. A laptop, local IDE or terminal must not be a prerequisite for the core flow.

## Implementation rules

- Inspect repository contents, branch state and existing tests before editing.
- Use the specified Next.js, NestJS, PostgreSQL, Redis/BullMQ, object storage and isolated-worker baseline unless repository evidence justifies a documented adjustment.
- Implement complete vertical slices; avoid placeholder buttons, fake progress and API-only stubs.
- Keep workflow state, permissions, approvals, readiness and audit events deterministic and server-authoritative.
- Treat AI output as a validated proposal. Get user confirmation before changing requirements, code or a live deployment.
- Keep all core views mobile-first and explain each question, option, warning and next action in context.
- Do not execute uploaded or generated code in API/web processes; use the isolation model in the technical specification.
- Update migrations, API schemas, tests, docs and user-facing explanations together.
- Provide local setup with .env.example and safe mocks for unavailable providers. Never commit real credentials.
- Do not claim a check, integration, build, preview or deployment succeeded unless it was actually verified.
- Apply the versioned quality profile to each generated build. Enforce the technical quality gates and truthful unknown states defined in the technical specification; never equate automated scans with certification.
- Do not perform destructive data changes or publish to a live external service without explicit confirmation.

## Completion criteria

Work through the roadmap to connected, runnable user flows. Do not stop after planning or scaffolding. Keep README and specifications synchronized with implementation. At each completed phase, summarize what works, how to run it, verification results, required external credentials and remaining phase work.
