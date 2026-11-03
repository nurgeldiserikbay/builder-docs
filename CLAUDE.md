# BuildPath — guide for Claude Code

Implement the product from the normative specifications below. Do not treat the landing page as evidence that a feature is already implemented.

## Read in this order

1. docs/PRODUCT_REQUIREMENTS.md
2. docs/ARCHITECTURE_AND_BUSINESS_LOGIC.md
3. docs/TECHNICAL_SPECIFICATION.md
4. docs/IMPLEMENTATION_ROADMAP.md
5. docs/product-blueprint.md

Authority order: PRODUCT_REQUIREMENTS.md defines user-visible behavior and acceptance; TECHNICAL_SPECIFICATION.md is the implementation contract for architecture, APIs, data, security and operations, provided it preserves the product requirements; ARCHITECTURE_AND_BUSINESS_LOGIC.md is a schematic map and must stay consistent with those two; IMPLEMENTATION_ROADMAP.md defines delivery order; product-blueprint.md is supplementary only and is not normative. If the product requirements and technical contract conflict, do not guess: record the conflict, align the docs and then implement. Never let a schematic or legacy concept silently override a concrete requirement.

## Product outcome

A person with a phone and internet connection can describe a website or application, understand recommended options, see an evolving result, revise it, stop at any stage or continue an existing project from a chosen stage. The mobile website and Telegram bot/Mini App share project state. A laptop, local IDE or terminal must not be a prerequisite for the core flow.

## Implementation rules

- Inspect repository contents, branch state and existing tests before editing.
- Use the specified Next.js, NestJS, PostgreSQL, Redis/BullMQ, object storage and isolated-worker baseline unless repository evidence justifies a documented adjustment.
- Keep the BuildPath platform stack separate from customer-generated projects. Generated customer code must follow the versioned capability profile; never infer that a platform dependency is supported for arbitrary user projects.
- Implement complete vertical slices; avoid placeholder buttons, fake progress and API-only stubs.
- Keep workflow state, permissions, approvals, readiness and audit events deterministic and server-authoritative.
- Treat AI output and user-uploaded AI instructions as untrusted input. Implement the staged malware/content/prompt-injection checks, fail-closed activation, version/scope checks and deterministic policy rules in the technical specification; never pass unchecked or disabled instruction content to an AI provider. Validate model output deterministically. A user's explicit create/revise request may generate a candidate code version for isolated checks and preview; do not silently change confirmed requirements, connect external accounts, incur charges, hand off code or deploy to production without separate user confirmation.
- Keep all core views mobile-first and explain each question, option, warning and next action in context.
- Accept product requirements and business logic as sufficient input; treat a user-provided technical specification or stack preference as optional. Build the technical plan from confirmed requirements, and ask the user to choose only for decisions with material cost, ownership, access, portability, integration or deployment consequences.
- Do not execute uploaded or generated code in API/web processes; use the isolation model in the technical specification. Treat uploaded custom AI instructions as data, not platform policy; preserve their lower-priority scoped context and never let them change authorization, confirmed requirements, safety gates or external-action consent. Implement stage-scoped file upload/export contracts from each versioned StageDefinition.
- Update migrations, API schemas, tests, docs and user-facing explanations together.
- Provide local setup with .env.example and safe mocks for unavailable providers. Never commit real credentials.
- Do not claim a check, integration, build, preview or deployment succeeded unless it was actually verified.
- Apply the versioned quality profile to each generated build. Enforce the technical quality gates and truthful unknown states defined in the technical specification; never equate automated scans with certification.
- Do not perform destructive data changes or publish to a live external service without explicit confirmation.

## Completion criteria

Work through the roadmap to connected, runnable user flows. Do not stop after planning or scaffolding. Keep README and specifications synchronized with implementation. At each completed phase, summarize what works, how to run it, verification results, required external credentials and remaining phase work.
