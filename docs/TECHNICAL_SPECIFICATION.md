# BuildPath — техническая спецификация

**Статус:** базовая архитектура для реализации.  
**Стиль:** модульный монолит с асинхронными worker-процессами; микросервисы для первой версии не требуются.

## 1. Технологический стек

- **Monorepo:** pnpm workspaces и Turborepo либо эквивалентный task runner.
- **Веб-клиент и Telegram Mini App:** Next.js App Router, React, TypeScript, PWA и общая библиотека адаптивных UI-компонентов.
- **API:** NestJS, TypeScript, REST API с префиксом /api/v1 и OpenAPI-контрактами.
- **Workers:** Node.js, BullMQ; длительные задачи выполняются вне HTTP-запросов.
- **Основная база:** PostgreSQL, Prisma migrations.
- **Очереди и кэш:** Redis с BullMQ. Redis не является источником истины.
- **Файлы:** приватное S3-совместимое object storage, короткоживущие signed URLs.
- **Telegram:** Bot API с webhook; bot и Mini App используют общий API и права.
- **Наблюдаемость:** структурированные логи, метрики, трассировка и correlation IDs.
- **Разработка:** Docker Compose; запуск и настройка по .env.example.

Провайдеры ИИ, email, object storage, построения сайта, Git и hosting подключаются через интерфейсы-адаптеры. Доменный код не импортирует SDK конкретного поставщика.

## 2. Структура репозитория

- apps/web — Next.js, мобильный веб, workspace и Telegram Mini App.
- apps/api — NestJS REST API, авторизация, webhook endpoints.
- apps/worker — очереди ИИ, импорта, генерации сайта, build, test и deploy.
- apps/telegram-bot — маршрутизация updates и форматирование сообщений.
- packages/contracts — DTO, схемы валидации, OpenAPI types.
- packages/domain — бизнес-сущности, правила состояний и use cases.
- packages/database — Prisma schema, migrations, repositories.
- packages/ui — доступные responsive-компоненты.
- packages/providers — адаптеры к внешним сервисам.
- packages/config — shared TypeScript, lint и environment validation.

Правила домена не должны находиться в React-компонентах, Telegram handlers или контроллерах.

## 3. Компоненты системы

1. **Web client:** вход, проекты, интервью, требования, варианты, редактор результатов, маршрут, согласования, предпросмотр и настройки.
2. **Telegram bot:** проверка подписи webhook, сопоставление аккаунта, доступные участнику проекты, статусы, уведомления, быстрые комментарии и решения.
3. **API:** валидация запросов, авторизация на каждый ресурс, управление данными и постановка фоновых задач.
4. **Worker:** обработка длительных заданий с idempotency, повторами и фиксируемым результатом.
5. **Build sandbox:** одноразовая изоляция для запуска пользовательского/сгенерированного кода с ограничением CPU, памяти, диска, времени и сети.
6. **Provider adapters:** типизированные интерфейсы для ИИ, хранения, почты, Git, Telegram и deploy.
7. **Preview gateway:** выдаёт опубликованный build по отдельному origin с ограниченным сроком действия и без application cookies.

## 4. Основная модель данных

Все пользовательские сущности принадлежат workspace. Все API-запросы обязаны проверять membership и границы workspace/project/stage.

### 4.1 Сущности

- **User:** email, имя, locale, verified state, timestamps.
- **Workspace:** owner, название, план/лимиты, настройки.
- **WorkspaceMember:** user, роль, приглашение, статус.
- **Project:** workspace, owner, название, summary, тип, режим старта, статус, locale, active stage, timestamps.
- **ProjectMember:** project, user, project role, status.
- **StageDefinition:** версия шаблона этапа, правила входа/выхода, допустимые зависимости.
- **ProjectStage:** project, stage definition/version, dependencies, order, status, required flag, checklist, review policy, assignee.
- **StageMember:** участник и stage-level role.
- **Requirement:** category, title, text, priority, status, source type/reference, author, timestamps.
- **Decision:** category (product/technology/code-storage/deployment/integration/operations), вопрос и простой summary влияния, варианты, recommended option + rationale, выбранная опция, status (proposed/accepted/rejected/deferred), `requires_explicit_confirmation`, provenance, связанный этап и actor/timestamp.
- **Artifact:** stage, тип, заголовок, ссылка на current version, visibility.
- **ArtifactVersion:** immutable object reference, номер, checksum, creator, source, metadata.
- **CommentThread и Comment:** stage/artifact version, parent, author, state, timestamps.
- **ReviewRequest:** конкретная версия артефакта, snapshot policy/reviewers, status.
- **Approval:** review request, reviewer, decision, comment, exact artifact version, timestamp.
- **ActivityEvent:** actor, project, channel, action, entity reference, safe metadata, timestamp.
- **FileObject:** object key, size, MIME, checksum, malware scan state, uploader, retention.
- **BuildJob:** job type, state, idempotency key, attempt, limits, log reference, result.
- **PreviewEnvironment:** build, opaque preview ID, URL, expiry, state.
- **ExternalConnection:** workspace/provider, encrypted credential reference, scopes, state.
- **Deployment:** project, build, provider, target, status, URL, confirmation actor, timestamps.
- **TelegramIdentity:** account, Telegram user ID, private chat ID, linked/revoked timestamps; unique active Telegram user.
- **TelegramLinkToken:** hash only, user, expiry, consumed timestamp.
- **NotificationPreference:** user, project scope, event, channel, enable, quiet hours.
- **Notification:** recipient, event, channel, delivery state, retry state.
- **AIInteraction:** provider key, task type, model ID, artifact references and safe usage metadata.
- **QualityProfile:** project, versioned baseline, risk classification, applicable checks, thresholds and approved exceptions.
- **QualityCheckRun:** project/build/artifact version, profile version, runner/tool versions, environment, lifecycle state, timestamps and summary.
- **QualityFinding:** check run, stable rule ID, severity, status, evidence reference, affected resource, remediation guidance and optional exception record.

### 4.2 Data invariants

- Artifact versions are immutable; edits create a new version.
- An approval is valid only for the exact version that was reviewed.
- Active review requests retain a snapshot of the required reviewers and policy.
- A project has a versioned route; dependency edges define stage order.
- User removal revokes future access but preserves audit history as required.
- Provider credentials are encrypted and never returned to the client.
- Every state-changing request creates an activity event transactionally.

## 5. State machines and workflow rules

### Project states

draft → active ↔ paused → completed → archived. Archived is read-only except export/deletion. Completing requires all required stages to be approved or explicitly waived when the route allows it.

### Stage states

locked → ready → in_progress → in_review → approved.

Additional transitions: in_review → changes_requested → in_progress; ready/in_progress → blocked; ready/in_progress → skipped only when allowed and with a reason. An approved stage may reopen by authorized action with a recorded reason. Dependent artifacts become stale and affected stages are flagged for review.

### Approval policies

- ONE_OF_ASSIGNED
- ALL_ASSIGNED
- QUORUM with required count
- NAMED_APPROVERS with explicit member IDs

Validate eligible reviewers before starting a review. A separation-of-duties rule may prohibit contributors from approving their own submitted version. If the policy or membership changes during active review, cancel and reissue the review explicitly.

The stage approval transition is transactional: validate required artifacts and checklist, no open blocking comments, approval threshold met, then save approved state and activity event. Handoff is either automatic under the configured rule or requires a separate authorized action.

### Readiness

Return structured data: required stages approved/total, omitted optional stages, blockers, pending approvals, launch checks passed/failed/unknown, next action and owner. A readiness percent may use disclosed route weights. Unknown or failed checks never count as passed. Readiness logic must be deterministic domain code.

## 6. Guided discovery and AI boundary

Define typed AI tasks: intake extraction, follow-up suggestions, type recommendation, requirement/business-rule draft, option comparison, technical-plan proposal, site generation/change proposal and review summary.

Every AI response is schema-validated and length-limited. Invalid output may be retried once, then surfaced as a recoverable error. Provider failures do not mutate project state.

AI cannot confirm requirements, change permissions, approve a stage, skip required checks, access provider credentials or publish. AI returns a proposal/diff. An authorized user explicitly accepts changes. Forms and manually entered structured data remain available if the provider is unavailable.

For each requirement, preserve provenance: user answer, imported material, deterministic validation or AI proposal. Keep confirmed requirements distinct from assumptions and open questions.

### Product intent and implementation decision policy

Treat the user's goal, audience, business rules and explicit constraints as the source of truth. A user may provide an existing technical specification, upload it or state preferred technologies; this is optional. When absent, derive an implementation plan from the confirmed requirements and supported platform capabilities.

Classify each implementation decision by impact. Low-impact, reversible technical details may use the supported recommended default and be recorded for transparency. Decisions with material consequences (recurring charges, provider account/credential, public exposure, data/control ownership, portability, external integration, or irreversible action) must be presented as plain-language options with tradeoffs, recommendation and an explicit user choice. If the user delegates the choice, follow their stored preferences and recommendation policy; still require confirmation before billing, connecting an account, granting external access or publishing to production.

Maintain a decision ledger separately from requirements: user constraint, system proposal, recommendation, chosen option, deferral and exception must be distinguishable. The deterministic workflow engine stores and enforces decisions; AI may explain or propose but cannot commit one. If a constraint conflicts with security, supported deployment capabilities or another requirement, surface the conflict and alternatives rather than silently overriding it.

Expose current supported code custody/export and deployment choices from capability registries, not from hardcoded AI claims. Each option includes supported status, prerequisites, ownership/access implications, portability, potential cost, required secrets, supported regions and next action. Never request provider credentials until the user selects that integration and starts connection.


## 7. REST API

All endpoints require authorization except health, registration/login and signed provider callbacks. Use typed DTOs, validation, pagination, idempotency keys for job creation and consistent problem responses.

### Auth and account

- POST /api/v1/auth/register
- POST /api/v1/auth/login
- POST /api/v1/auth/logout
- POST /api/v1/auth/verify-email
- POST /api/v1/auth/password/reset
- GET /api/v1/me

### Workspaces and projects

- POST /api/v1/workspaces
- GET /api/v1/workspaces/:workspaceId
- POST /api/v1/workspaces/:workspaceId/invitations
- POST /api/v1/invitations/:token/accept
- POST /api/v1/projects
- GET /api/v1/projects?workspaceId=...
- GET /api/v1/projects/:projectId
- PATCH /api/v1/projects/:projectId
- POST /api/v1/projects/:projectId/pause
- POST /api/v1/projects/:projectId/resume
- POST /api/v1/projects/:projectId/members
- DELETE /api/v1/projects/:projectId/members/:memberId

### Intake and requirements

- POST /api/v1/projects/:projectId/intake/messages
- POST /api/v1/projects/:projectId/intake/answers
- GET /api/v1/projects/:projectId/requirements
- PATCH /api/v1/projects/:projectId/requirements/:requirementId
- POST /api/v1/projects/:projectId/requirements/:requirementId/confirm
- GET /api/v1/projects/:projectId/route
- POST /api/v1/projects/:projectId/route/proposals
- POST /api/v1/projects/:projectId/route/accept
- POST /api/v1/projects/:projectId/technical-plan/proposals
- GET /api/v1/projects/:projectId/technical-plan
- GET /api/v1/projects/:projectId/decisions
- POST /api/v1/projects/:projectId/decisions/:decisionId/resolve
- GET /api/v1/projects/:projectId/delivery-options (code custody/export and deployment capabilities with prerequisites)

### Stages and reviews

- GET /api/v1/projects/:projectId/stages
- GET /api/v1/projects/:projectId/stages/:stageId
- PATCH /api/v1/projects/:projectId/stages/:stageId
- POST /api/v1/projects/:projectId/stages/:stageId/assignments
- POST /api/v1/projects/:projectId/stages/:stageId/review-requests
- POST /api/v1/review-requests/:reviewId/decisions
- POST /api/v1/projects/:projectId/stages/:stageId/handoffs
- GET /api/v1/projects/:projectId/activity
- GET /api/v1/projects/:projectId/readiness

### Upload, site generation and delivery

- POST /api/v1/projects/:projectId/files/upload-intents
- POST /api/v1/projects/:projectId/files/:fileId/complete
- POST /api/v1/projects/:projectId/imports
- POST /api/v1/projects/:projectId/site-generations
- POST /api/v1/projects/:projectId/site-changes
- POST /api/v1/projects/:projectId/builds
- GET /api/v1/builds/:buildId
- POST /api/v1/projects/:projectId/previews
- POST /api/v1/projects/:projectId/deployments
- GET /api/v1/deployments/:deploymentId
- GET /api/v1/projects/:projectId/quality/profile
- PATCH /api/v1/projects/:projectId/quality/profile (authorized, audited exceptions only)
- POST /api/v1/projects/:projectId/quality/check-runs
- GET /api/v1/projects/:projectId/quality/check-runs/:runId
- GET /api/v1/projects/:projectId/quality/findings
- POST /api/v1/projects/:projectId/quality/findings/:findingId/exceptions
- POST /api/v1/projects/:projectId/exports

### Telegram and notifications

- POST /api/v1/telegram/link-tokens
- DELETE /api/v1/telegram/connection
- POST /api/v1/webhooks/telegram
- GET /api/v1/notifications
- PATCH /api/v1/notification-preferences

The concrete DTOs and response schemas are the OpenAPI contracts; arbitrary untyped JSON is not allowed.

## 8. Code generation and build isolation

- Accept only configured file types; validate extension, MIME and magic bytes; cap file size.
- Keep uploads private, scan before processing, use short-lived signed URLs.
- Never run project code in the API or web process.
- Execute each build in a disposable sandbox with non-root UID, bounded CPU/RAM/disk/time, read-only base image, no host socket or host mounts.
- Restrict network egress by default; block cloud metadata and internal service ranges; allow only explicitly required package registries and integrations through a controlled proxy.
- Keep production secrets out of generated build jobs.
- Use separate preview origins and restrictive CSP. Never forward application cookies to previews.
- Redact and cap build logs; associate logs with job IDs.
- Automatically delete expired workspaces and temporary artifacts according to retention policy.
- Production hostile-code boundary must use a dedicated isolation runtime such as gVisor, Firecracker or equivalent. Docker alone is development-only.

## 9. Security and privacy

- Use secure, HTTP-only, SameSite browser cookies and CSRF protection for mutations.
- Hash passwords with a modern adaptive password hash, or use a verified passwordless flow. Verify email before provider connections.
- Check tenant, project and stage permissions on every API operation; the UI is not an authorization boundary.
- Telegram link token is random, single-use, hashed at rest, short-lived and revocable.
- Validate Telegram webhook secret and Mini App init data server-side using Telegram's signature rules.
- Initial bot mode is private chat. Group use is optional later and needs explicit project configuration plus per-user verification.
- Encrypt external provider credentials with a managed key. Do not put tokens in logs, build variables, source files or client responses.
- Rate-limit auth, AI, uploads, URL imports, builds, deploy and invitations.
- Account export and deletion, data retention, backup and restore behavior are documented before production.

### URL import SSRF defenses

Resolve and validate DNS on every redirect, block private/loopback/link-local/metadata ranges for IPv4 and IPv6, cap redirects and response size, restrict schemes to HTTP/HTTPS, set timeouts and use an egress proxy. Do not fetch arbitrary user-supplied URLs from privileged API networks.

## 10. Job processing and errors

Job states: queued, running, succeeded, failed, cancelled. Jobs have idempotency key, attempt count, progress, correlation ID, safe error code and related artifact. Use bounded exponential retries only for retryable errors. Distinguish provider timeout/quota, invalid output, failed build, unsafe file, failed preview, deploy rejection and unknown remote state.

The user sees a concise cause, what was preserved and the next available action. Never display a false success state when a webhook or provider result is unknown.

## 11. Configuration and operations

- Validate environment variables at startup and provide a complete .env.example.
- Include Docker Compose for local database, Redis, storage emulator and mail testing.
- Document migrations, Telegram webhook setup, worker concurrency and provider setup.
- Production uses HTTPS, managed backups, isolated workers, authenticated Redis, private object storage and separate preview origin.
- Maintain health/readiness endpoints, structured errors and job metrics.
- Document backup restore, failed deployment rollback, retention and incident response.

## 12. Quality engineering policy

### Policy selection

Create a versioned quality profile for every project. The deterministic policy engine derives applicable checks from project type, public/private visibility, data classification, authentication, payments, external integrations and deployment target. Users can see the defaults in ordinary language; only advanced users need to inspect technical controls. A profile change is versioned and cannot erase earlier check results.

### Baseline targets

- **Security:** use OWASP ASVS 5.0 Level 1 as the default verification checklist for web applications. Increase required controls and add expert review for elevated-risk projects. Run secret detection, dependency/license checks, static analysis, permission-boundary tests and applicable dynamic checks. Never claim security certification from automated scans.
- **Accessibility:** target WCAG 2.2 AA for generated user interfaces. Combine automated analysis with keyboard navigation, focus visibility, zoom/reflow and screen-reader sampling. Keep automated and manual coverage separate.
- **Performance:** optimize for mobile and slow connections. Track lab metrics per build and, after launch, field Core Web Vitals (LCP, INP, CLS). Initial good targets are LCP ≤ 2.5 s, INP ≤ 200 ms and CLS ≤ 0.1 at the 75th percentile. Do not mark field targets met without sufficient real-user data.
- **SEO:** enable technical SEO checks only for public indexable pages: crawlability, canonical URLs, metadata, sitemap/robots behavior, semantic headings/links and valid structured data. Private routes must not leak into an index. Rankings are outside the product's guarantee.
- **Reliability and maintainability:** validate types, formatting, lint, critical route behavior, link integrity, supported browser/mobile layouts, migration safety, backup/restore steps, deployment health checks and rollback availability.
- **Privacy:** minimize collected data; document purpose, retention, export/deletion and third-party transfer for each sensitive data class. Prevent personal data, credentials and source contents from appearing in telemetry unless explicitly required and protected.

Standards and tools are versioned in the profile. The initial references are OWASP ASVS 5.0.0, W3C WCAG 2.2, Google Search Essentials and current Core Web Vitals definitions. Keep standards identifiers and URLs in the profile registry so policy updates are reviewed rather than silently changing old project results.

Reference sources: [OWASP ASVS 5.0.0](https://asvs.owasp.org/downloads/), [W3C WCAG 2.2](https://www.w3.org/TR/WCAG22/), [Google Search Essentials](https://developers.google.com/search/docs/essentials), [Core Web Vitals](https://web.dev/articles/vitals).

### Check result and release gate

Each finding has severity (critical/high/medium/low/info), status (open/fixed/accepted/not-applicable/not-verified), evidence reference, affected version/resource and actionable remediation. Check runs are immutable records of the exact artifact/build, policy version, tool versions and environment. The public API exposes a summary and safe evidence only; secret values, raw credentials, sensitive source snippets and exploitable exploit details are redacted or access-controlled.

Before preview publication or production deployment, evaluate the policy against the exact build. Block release by default on unresolved critical/high security findings, failed critical user journeys, or mandatory checks that are not verified. Warnings require a visible acknowledgement; exceptions require a role-authorized actor, a reason, expiry/review date where applicable, and an audit event. A new artifact version invalidates relevant earlier results and triggers only the affected checks again. External audits, legal compliance and guarantees that a site is vulnerability-free are never inferred from a passing automated pipeline.

Quality execution runs as bounded background jobs outside API processes and outside the untrusted build sandbox where appropriate. Use isolated runners, timeouts, resource limits, fixed tool images and redacted logs. Store machine-readable reports by access-controlled object reference; escape/sanitize report data before rendering it in the dashboard.

## 13. Verification required during implementation

- Domain unit tests for stage transitions, approval policy and readiness math.
- PostgreSQL/Redis API integration tests.
- Authorization matrix tests for workspace/project/stage roles.
- Idempotency/retry tests for long-running jobs.
- Telegram linking, revocation, webhook authentication and action authorization.
- Upload validation, URL import SSRF and preview-origin isolation checks.
- Mobile end-to-end tests for intake, explained options, preview, revision, pause/resume and deployment.
- Accessibility checks for keyboard, screen reader, contrast and reduced motion.
- Sandbox tests using valid, malformed and intentionally hostile project packages.

A check with no recorded run result is unknown, not passed.
