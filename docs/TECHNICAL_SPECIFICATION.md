# BuildPath — техническая спецификация

**Статус:** базовая архитектура для реализации.  
**Стиль:** модульный монолит с асинхронными worker-процессами; микросервисы для первой версии не требуются.

## 1. Технологический стек

- **Репозитории:** три отдельных GitHub-репозитория: `builder-front`, `builder-back` и `builder-admin`. Версионированный API-контракт хранится в `builder-back/apps/api/openapi.yaml`; frontend и admin генерируют клиентские типы из опубликованной версии контракта.
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

## 2. Структура репозиториев

Реализация ведётся в трёх репозиториях с независимыми сборками и релизами. Контракты связывают части системы; копии доменной логики между ними не создаются.

- **builder-back:** NestJS REST API, OpenAPI source of truth, workers, Prisma schema/migrations, domain rules, provider adapters и серверные интеграции.
- **builder-front:** Next.js App Router, мобильный веб, PWA и Telegram Mini App; использует типизированный клиент, сгенерированный из версии OpenAPI.
- **builder-admin:** отдельное Next.js приложение для внутренней панели; использует тот же версионированный API-контракт и отдельную staff authorization policy.
- Shared DTO/client code публикуется или генерируется из контракта `builder-back`; ручное копирование request/response схем запрещено.
- Изменение API выполняется в таком порядке: изменить и проверить OpenAPI, реализовать сервер, выпустить совместимую версию контракта, обновить front/admin clients. Нельзя считать API доступным только потому, что он описан в UI.
- Правила домена остаются в `builder-back`; их нельзя дублировать в React-компонентах, Telegram handlers или контроллерах.

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
- **Project:** workspace, owner, название, summary, тип, режим старта (`build`, `improve`, `design_implementation`, `analysis_only`, ...), status, `resumeState` для паузы, locale, active stage, выбранный stopping outcome, timestamps.
- **ProjectMember:** project, user, project role, status.
- **StageDefinition:** версия шаблона этапа, правила входа/выхода, допустимые зависимости.
- **ProjectStage:** project, stage definition/version, dependencies, order, status, required flag, checklist, review policy, assignee, blocked reason and resume state.
- **StageMember:** участник и stage-level role.
- **Requirement:** category, title, text, priority, status (`proposed`, `confirmed`, `assumption`, `open_question`, `rejected`), source type/reference, author, `confirmedBy/At`, `supersedesId`, timestamps. Only confirmed requirements drive generation; assumptions must remain visible.
- **Decision:** category (product/technology/code-storage/deployment/integration/operations), вопрос и простой summary влияния, варианты, recommended option + rationale, выбранная опция, status (proposed/accepted/rejected/deferred), `requires_explicit_confirmation`, provenance, связанный этап и actor/timestamp.
- **DecisionPreference:** user/project scope, categories/providers/options the user prefers, disallowed options, applicable limits and expiry/revocation. A preference helps choose defaults; it never authorizes charging, external account access, code handoff or production deployment.
- **Artifact:** stage, тип, заголовок, ссылка на current version, visibility.
- **ArtifactVersion:** immutable object reference, version number, checksum, creator, source, metadata. A generated `SiteSpec`/page model is stored as a typed artifact version and names its schema and capability-manifest versions.
- **CommentThread и Comment:** stage/artifact version, parent, author, state, timestamps.
- **ReviewRequest:** конкретная версия артефакта, snapshot policy/reviewers, status.
- **Approval:** review request, reviewer, decision, comment, exact artifact version, timestamp.
- **ActivityEvent:** actor, project, channel, action, entity reference, safe metadata, timestamp.
- **FileObject:** object key, size, MIME, checksum, malware scan state, uploader, retention, owning project/stage, visibility.
- **UserAIInstruction:** project, owner, source file/text reference, extracted-text checksum, version, scope (project/stage/task), status, findings, validator/profile version, enabledAt/by, disabledAt/by, timestamps.
- **StageMaterialPolicy:** stage template/version, allowed input/output artifact types and formats, size/count limits, required/optional flags, validation and export rules.
- **BuildJob:** job type, state, idempotency key, attempt, limits, log reference, result.
- **PreviewEnvironment:** build, opaque preview ID, URL, expiry, state.
- **ExternalConnection:** workspace/provider, encrypted credential reference, scopes, state.
- **Deployment:** project, build, provider, target, status, URL, confirmation actor, timestamps.
- **TelegramIdentity:** account, Telegram user ID, private chat ID, linked/revoked timestamps; unique active Telegram user.
- **TelegramLinkToken:** hash only, user, expiry, consumed timestamp.
- **NotificationPreference:** user, project scope, event, channel, enable, quiet hours.
- **Notification:** recipient, event, channel, delivery state, retry state.
- **AIInteraction:** provider key, task type, model ID, artifact references and safe usage metadata.
- **CapabilityCatalogVersion:** immutable manifest revision defining project/platform/features marked supported, limited, docs-only or unsupported, and compatible generators, checks and delivery adapters.
- **QualityProfile:** project, versioned baseline, risk classification, applicable checks, thresholds and approved exceptions.
- **QualityCheckRun:** project/build/artifact version, profile version, runner/tool versions, environment, lifecycle state, timestamps and summary.
- **QualityFinding:** check run, stable rule ID, severity, status, evidence reference, affected resource, remediation guidance and optional exception record.
- **AnalysisRun:** project, requested scope, exact input artifact/URL/repository snapshot revision, selected analysis profile/version, status, checks/tools versions, started/finished timestamps, summary, limitations.
- **AnalysisReport:** analysis run, immutable artifact/version, format, findings and recommendations, evidence references, unverified areas, exporter version.

### 4.2 Data invariants

- Artifact versions are immutable; edits create a new version.
- An approval is valid only for the exact version that was reviewed.
- Active review requests retain a snapshot of the required reviewers and policy.
- A project has a versioned route; dependency edges define stage order.
- User removal revokes future access but preserves audit history as required.
- Provider credentials are encrypted and never returned to the client.
- Project requirement/decision changes increment a revision; generated plans and artifacts record the exact input revision and capability manifest version.
- Decision preferences can only narrow supported options and steer defaults; product safety policy and explicit confirmations for external side effects always take precedence.
- Every state-changing request creates an activity event transactionally.
- An AnalysisRun is read-only and binds every finding to an immutable input snapshot/version; starting analysis cannot mutate source files, requirements, repository branches or deployment state.
- A file or artifact is always scoped to a project and, when applicable, a stage; membership alone does not grant read/download permission.
- UserAIInstruction is never sent to a model unless that exact version passed checks, is enabled by an authorized project owner/manager, and is within its declared scope.

## 5. State machines and workflow rules

### Project states

- `draft → active` when the owner starts the guided project.
- Any nonterminal state (`draft` or `active`) may transition to `paused`, saving `resumeState`; `paused → resumeState` restores the previous project state and preserves each stage's prior state.
- `active → completed` only when the chosen stopping outcome is satisfied and all required stages for that outcome are approved or explicitly waived where the route permits it.
- `completed → archived`; archived is read-only except authorized export/deletion. Restore returns to `completed`; resuming work then requires the reopen flow.
- Reopening a completed project requires owner/admin permission, a reason and a new route revision.

### Stage states

locked → ready → in_progress → in_review → approved.

Additional transitions: `in_review → changes_requested → in_progress`; `ready/in_progress → blocked → prior resumable state` after resolving the blocker; `ready → skipped` only when optional and route policy permits it, with a reason. An approved stage may reopen by authorized action with a recorded reason. Dependent artifacts become stale and affected stages are flagged for review. A route must be an acyclic dependency graph; validate this when accepting or revising a route.

### Approval policies

- ONE_OF_ASSIGNED
- ALL_ASSIGNED
- QUORUM with required count
- NAMED_APPROVERS with explicit member IDs

Validate eligible reviewers before starting a review. By default, an author cannot approve their own submitted version; a project policy may relax this only through an explicit owner choice and an audit record. If the policy or membership changes during active review, cancel and reissue the review explicitly.

The stage approval transition is transactional: validate required artifacts and checklist, no open blocking comments, approval threshold met, then save approved state and activity event. Handoff is either automatic under the configured rule or requires a separate authorized action.

### Readiness

Return structured data: required stages approved/total, omitted optional stages, blockers, pending approvals, launch checks passed/failed/unknown, next action and owner. A readiness percent may use disclosed route weights. Unknown or failed checks never count as passed. Readiness logic must be deterministic domain code.

## 6. Guided discovery and AI boundary

Define typed AI tasks: intake extraction, follow-up suggestions, type recommendation, requirement/business-rule draft, option comparison, technical-plan proposal, site generation/change proposal and review summary.

Every AI response is schema-validated and length-limited. Invalid output may be retried once, then surfaced as a recoverable error. Provider failures do not mutate project state.

AI cannot confirm requirements, change permissions, approve a stage, skip required checks, access provider credentials or publish. A user's explicit create/revise request authorizes generating a candidate artifact/code version for isolated validation and preview against the accepted requirements; it does not authorize changing confirmed requirements or making external changes. AI returns a typed proposal/change set; deterministic validators and project policy decide whether it can become a preview candidate. No per-library/framework approval is required when it follows the agreed technical plan. External account connection, paid action, code handoff and production deployment always require separate user confirmation. Forms and manually entered structured data remain available if the provider is unavailable.

#### User-provided AI instruction pipeline

Use typed states: `uploaded`, `scanning`, `needs_changes`, `rejected`, `approved`, `active`, `disabled`, `deleted`. `approved` is not active until an authorized user explicitly enables that exact version. A replacement creates a new version; an active prior version is disabled while the new one is checked.

The pipeline is deterministic and fail-closed:
1. Check project permission, extension/MIME/magic bytes, configured size/page limits, checksum, malware and archive/decompression limits. Keep the object private and quarantine it from parsers/models until checks pass.
2. Extract plain text in a constrained parser without executing embedded code, macros, links or attachments. Apply encoding, page-count, token/character and content-length limits; retain source provenance.
3. Run versioned deterministic pattern/policy checks plus a separately identified classifier for prompt injection, requests for secrets/data exfiltration, unsafe code/commands, permission changes, policy bypass or harmful instructions. Classifier failure must not skip deterministic checks; uncertain/high-risk results remain inactive and require editing/re-upload or human review under an explicit policy.
4. Return stable finding codes, severity, safe excerpt/position, explanation and remediation. Do not expose another project's data or echo sensitive content into logs. Allow the owner to edit, remove the flagged passage, upload a new version or delete it. Do not silently strip text and activate a modified instruction.
5. An authorized owner/manager explicitly enables an approved version and chooses project, stage or task scope. For every AI call, inject it in a labeled untrusted-context field after system/product instructions; never concatenate it as an instruction with equal/higher priority. Revalidate scope and status at use time. It cannot change requirements/decisions, permissions, safety policy, quality gates or authorize side effects.
6. Record version/checker revision, status changes, actor, scope and safe finding metadata in the audit log. Provide disable/revoke/delete and project/account retention/export behavior. Redact content from telemetry and provider logs where configurable.

Instruction checks detect known patterns and reduce risk; they do not certify arbitrary text as harmless. The UI must say so plainly. A failed/unavailable scanner or classifier never grants approval; users can continue without custom instructions using the normal BuildPath flow.

#### Stage material upload and export

Every StageDefinition version declares input/output artifact types, allowed formats, maximum count/size, required/optional status, validation checks and export formats. A stage can start without optional materials; required inputs block only that stage and explain how to supply them. Users can upload or download materials for any stage they have permission to access. Each upload is private, scanned before parsing, checksummed and versioned; each export is generated from a specific artifact version, access-checked, audited and delivered using a short-lived URL. Never overwrite a prior version silently. The UI must state when a file is only attached for human reference versus actually parsed or used to build.

For each requirement, preserve provenance: user answer, imported material, deterministic validation or AI proposal. Keep confirmed requirements distinct from assumptions and open questions.

### Product intent and implementation decision policy

Treat the user's goal, audience, business rules and explicit constraints as the source of truth. A user may provide an existing technical specification, upload it or state preferred technologies; this is optional. When absent, derive an implementation plan from the confirmed requirements and supported platform capabilities.

Classify each implementation decision by impact. Low-impact, reversible technical details may use the supported recommended default and be recorded for transparency. Decisions with material consequences (recurring charges, provider account/credential, public exposure, data/control ownership, portability, external integration, or irreversible action) must be presented as plain-language options with tradeoffs, recommendation and an explicit user choice. If the user delegates the choice, follow their stored preferences and recommendation policy; still require confirmation before billing, connecting an account, granting external access or publishing to production.

Maintain a decision ledger separately from requirements: user constraint, system proposal, recommendation, chosen option, deferral and exception must be distinguishable. The deterministic workflow engine stores and enforces decisions; AI may explain or propose but cannot commit one. If a constraint conflicts with security, supported deployment capabilities or another requirement, surface the conflict and alternatives rather than silently overriding it.

Expose current supported code custody/export and deployment choices from capability registries, not from hardcoded AI claims. Each option includes supported status, prerequisites, ownership/access implications, portability, potential cost, required secrets, supported regions and next action. Never request provider credentials until the user selects that integration and starts connection.


### Capability manifest and MVP scope

The capability catalog is a versioned, deterministic manifest, not an AI-generated claim. Each project type/feature combination is one of: `supported`, `limited`, `docs_only`, `unsupported`. A supported entry names its validated input schema, page/component model, generator/template version, required quality checks, known limits, and available preview/export/deploy adapters. Include the manifest revision in the route, generated artifacts and quality reports.

Initial automatic code generation is limited to responsive public websites assembled from vetted page templates: landing, portfolio, small business/professional site, event/product promo, and informational menu/catalog. Initial feature modules are content pages/navigation, responsive sections/media and contact/lead forms only through a configured supported handler. Native iOS/Android/desktop apps, payments, complex accounts, marketplaces and sensitive/high-risk workflows remain documentation/prototype/developer-handoff until a capability entry explicitly supports their complete generation and verification path.

The stack in section 1 is the stack used to build the BuildPath platform; it is not inherited automatically by customer projects. The first generated-site profile emits a static-first Next.js App Router + TypeScript project from the versioned SiteSpec. It does not add a customer backend/database unless a supported feature profile requires one. If a customer-provided stack is not in the verified generator manifest, preserve it as a requirement and offer a developer handoff or an explicit switch to a supported generator; never silently claim or substitute stack compliance.

Unsupported feature requests still produce requirements, business logic, architecture, UX and implementation handoff. Do not create a fake functional button or mark an unsupported runtime feature complete. The UI must show the capability status and why code generation/deployment is unavailable.

### Deterministic site model and AI-independent baseline

Canonical generated content is a typed, versioned `SiteSpec`/page model composed from allowlisted page types, components, properties, interactions and feature modules. A deterministic renderer/compiler maps that model to the supported site package. AI may propose a schema-validated `SiteChangeSet`; deterministic validation checks references, allowed components, required inputs, route behavior and capability limits before applying it as a new candidate version. Never execute model-returned shell commands or accept arbitrary runtime code as if it had passed the typed model.

When AI providers are unavailable, a user can still create supported sites by filling structured forms, choosing approved templates/components, editing content and configuring available modules. Existing requirements, review, quality, export and project continuation continue to work. For requests outside the deterministic builder's capability, explain the limit and create a developer handoff package instead of fabricating completion.

## 7. REST API

All endpoints require authorization except health, registration/login and signed provider callbacks. Use typed DTOs, validation, pagination, idempotency keys for job creation and consistent problem responses.

The list below is the endpoint inventory, not a substitute for schemas. The OpenAPI document at `apps/api/openapi.yaml` is the source of truth and must define every operation's request/response DTO, validation, authorization scope, side effects and error codes before that endpoint is implemented. Generate client types from it.

- Success body: `{ data, meta? }`; `meta` may include request ID and cursor pagination fields.
- Errors use RFC 9457 Problem Details with stable application `code`, `requestId` and field-level validation details; never return stack traces or secrets.
- Collection endpoints use bounded cursor pagination; validate sort/filter fields against allowlists.
- Job-creating mutations require an `Idempotency-Key`; retrying the same key and body returns the original job, and the same key with a different body is a conflict.
- Every route documents required role/scope and whether it creates an audit event. OpenAPI schemas must not use untyped `any`/arbitrary JSON for domain inputs.


### Auth and account

- POST /api/v1/auth/register
- POST /api/v1/auth/login
- POST /api/v1/auth/logout
- POST /api/v1/auth/verify-email
- POST /api/v1/auth/password/reset
- GET /api/v1/me
- POST /api/v1/me/exports
- POST /api/v1/me/deletion-requests

### Workspaces and projects

- POST /api/v1/workspaces
- GET /api/v1/workspaces/:workspaceId
- GET /api/v1/workspaces/:workspaceId/members
- PATCH /api/v1/workspaces/:workspaceId/members/:memberId/role
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
- POST /api/v1/workspaces/:workspaceId/connections/:provider/authorization
- GET /api/v1/workspaces/:workspaceId/connections
- DELETE /api/v1/workspaces/:workspaceId/connections/:connectionId
- POST /api/v1/webhooks/providers/:provider

### Intake and requirements

- POST /api/v1/projects/:projectId/intake/messages
- POST /api/v1/projects/:projectId/intake/answers
- GET /api/v1/projects/:projectId/requirements
- PATCH /api/v1/projects/:projectId/requirements/:requirementId
- POST /api/v1/projects/:projectId/requirements/:requirementId/confirm
- GET /api/v1/capabilities?projectType=...
- GET /api/v1/projects/:projectId/capabilities
- GET /api/v1/projects/:projectId/route
- POST /api/v1/projects/:projectId/route/proposals
- POST /api/v1/projects/:projectId/route/accept
- POST /api/v1/projects/:projectId/technical-plan/proposals
- GET /api/v1/projects/:projectId/technical-plan
- GET /api/v1/projects/:projectId/decisions
- GET /api/v1/projects/:projectId/decision-preferences
- PATCH /api/v1/projects/:projectId/decision-preferences
- POST /api/v1/projects/:projectId/decisions/:decisionId/resolve
- GET /api/v1/projects/:projectId/delivery-options (code custody/export and deployment capabilities with prerequisites)

### Analysis-only reports

Support `analysis_only` as a project start mode without generating a mandatory development route. Inputs are uploaded documents/designs/source archives, a public URL fetched only through the SSRF-safe importer, or a repository connection limited to explicit read-only scopes. The user chooses categories or accepts a recommended profile. Validate the requested source and each check against the versioned capability manifest; show unsupported/unavailable areas before starting.

A run is asynchronous and idempotent. Its status is `queued | running | succeeded | partial | failed | cancelled`. Each finding includes stable ID, category, severity, confidence/status, evidence pointer to an exact file/page/line/version, plain-language explanation and remediation. The report records tools/profile/input revisions, checks run, checks skipped, access errors, limitations and unverified areas. AI may summarize or explain evidence but cannot invent findings or mark an unrun check as passed. A report is not a security certification or a guarantee that no issue exists.

Analysis is read-only by default. It does not edit the input, create code changes, open a PR, deploy, approve a stage or alter confirmed requirements. Turning recommendations into work items or requesting fixes is a separate user action and follows normal stage, permission, review and quality-gate rules. Store report as an immutable artifact version and export only through access-checked short-lived links in supported formats (PDF, Markdown, JSON, HTML according to capability).

### Stages and reviews

- POST /api/v1/projects/:projectId/analysis-runs
- GET /api/v1/projects/:projectId/analysis-runs
- GET /api/v1/projects/:projectId/analysis-runs/:runId
- GET /api/v1/projects/:projectId/analysis-runs/:runId/report
- POST /api/v1/projects/:projectId/analysis-runs/:runId/exports
  - Implemented first version (STAGE_WORK_MODEL.md §40): sources `file` (ZIP stage file), `repository`, `public_repo` (GitHub); export is `GET …/analysis-runs/:runId/export?format=md|json` with the session instead of a short-lived link.

- GET /api/v1/projects/:projectId/stages
- GET /api/v1/projects/:projectId/stages/:stageId
- PATCH /api/v1/projects/:projectId/stages/:stageId (metadata only; never directly changes workflow state)
- POST /api/v1/projects/:projectId/stages/:stageId/transitions (validated start, block/unblock, reopen or permitted-skip command; review submission and decisions use their dedicated endpoints, and handoff remains a separate operation)
- POST /api/v1/projects/:projectId/stages/:stageId/assignments
- POST /api/v1/projects/:projectId/stages/:stageId/review-requests
- POST /api/v1/review-requests/:reviewId/decisions
- POST /api/v1/projects/:projectId/stages/:stageId/handoffs
- GET /api/v1/projects/:projectId/activity
- GET /api/v1/projects/:projectId/readiness

### Upload, site generation and delivery

- POST /api/v1/projects/:projectId/files/upload-intents
- POST /api/v1/projects/:projectId/files/:fileId/complete
- GET /api/v1/projects/:projectId/stages/:stageId/materials (authorized list and stage policy)
- POST /api/v1/projects/:projectId/stages/:stageId/materials/upload-intents
- POST /api/v1/projects/:projectId/stages/:stageId/materials/:materialId/complete
  - Implemented first version (STAGE_WORK_MODEL.md §40): `GET|POST /projects/:projectId/stages/:stageKey/files` (multipart, bytes stored in the database like project images, scanned before storing), `GET|DELETE …/files/:fileId`. Upload intents with object storage remain the target for large files.
- POST /api/v1/projects/:projectId/stages/:stageId/exports (artifact/version and requested format)
- GET /api/v1/exports/:exportId (authorized status and short-lived download URL)
- POST /api/v1/projects/:projectId/ai-instructions (create from validated text or completed upload)
- GET /api/v1/projects/:projectId/ai-instructions (metadata/status only; content requires authorized scope)
- POST /api/v1/projects/:projectId/ai-instructions/:instructionId/enable
- POST /api/v1/projects/:projectId/ai-instructions/:instructionId/disable
- DELETE /api/v1/projects/:projectId/ai-instructions/:instructionId
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
- Keep uploads private, scan before processing, use short-lived signed URLs. User AI instructions require the separate staged validation pipeline in §6; file completion alone never makes text available to a model.
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
- Analysis input access is read-only and least-privilege. Never clone/fetch a private repository until the user authorizes the exact repository and read-only scope; never use repo credentials in AI context or build containers.
- Authorization is deny-by-default and object-scoped: active account + project membership/delegated project access + stage scope + action permission. Workspace membership or ownership alone does not reveal private project contents; stage membership cannot grant workspace/admin powers. Use the permission matrix in PRODUCT_REQUIREMENTS.md as the policy source.
- Telegram link token is random, single-use, hashed at rest, short-lived and revocable.
- Validate Telegram webhook secret and Mini App init data server-side using Telegram's signature rules.
- Initial bot mode is private chat. Group use is optional later and needs explicit project configuration plus per-user verification.
- Encrypt external provider credentials with a managed key. Do not put tokens in logs, build variables, source files or client responses.
- Provider OAuth uses single-use state, PKCE where supported, exact allowlisted redirect URIs, minimal scopes and explicit user consent. Validate callback state, bind the resulting connection to the initiating workspace, support revocation, and never pass connection tokens to AI or build containers.
- Rate-limit auth, AI, uploads, URL imports, builds, deploy and invitations. Apply tenant/project/stage authorization, configured size/count limits, retention and audit rules to every file and export. Do not log custom instruction bodies.
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

Evaluate every check against the exact immutable build and capability/profile version.

- **Candidate build:** preserve source and reports when a build/check fails; never call it a preview or deployment success.
- **Private iteration preview:** allow only after the build starts successfully and preview-isolation checks pass (malware/secret scan, isolated origin with no application cookies, short-lived unguessable URL, restrictive CSP and bounded network). It remains private, unindexed and visibly marked as an unverified preview. Production-only checks may still be pending; never label it production-ready.
- **Shareable/public preview and production:** require every mandatory release check to be verified, no unresolved critical/high security finding, no failed critical user journey, and any provider-specific prerequisites. Critical/high findings are non-waivable by default. Warnings need visible acknowledgement. Any allowed exception must be within the configured policy, by a role-authorized person, with reason, expiry/review date and audit event. Production release also requires a separate user confirmation.
- **Version changes:** a new artifact version invalidates relevant earlier results and triggers only affected checks again. Unknown and not-applicable statuses remain distinct.
- **Truthful claims:** external audits, legal compliance and guarantees that a site is vulnerability-free are never inferred from a passing automated pipeline.

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

## Сверка дизайна и рабочего сайта

См. [DESIGN_CONFORMANCE.md](DESIGN_CONFORMANCE.md) для модели ReviewRun/ReviewPage/DesignSource/PageDesignMapping/ReviewFinding/ReviewReport, API-контрактов, асинхронных заданий сравнения, SSRF-защиты, браузерной изоляции и требований к воспроизводимым доказательствам. Сначала обновить OpenAPI и миграции; обработчики не писать до фиксации схем. Сравнение backend допускается только при наличии связанного контракта или тестового окружения и санитизированного запроса/ответа.
