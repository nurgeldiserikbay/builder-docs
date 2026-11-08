# BuildPath design conformance and site review

## Purpose

The design conformance workspace lets a project owner compare the working website with its requirements and design, page by page. It supports a public or preview URL, uploaded designs, and a connected Figma file. Review can be manual, automated, or both. The tool produces a versioned report and does not modify the website.

This is a product capability specification. It does not claim that an implementation exists yet.

## User journey

1. The user starts a review from a project or chooses **Analyze only**. They provide a website URL, a private preview URL, a ZIP of page designs, or a Figma file. A project can combine these sources.
2. BuildPath discovers public pages from same-origin links and sitemap, then presents a page list. The user can add, remove, or rename routes and explicitly pair each site route with a Figma frame or uploaded image.
3. For every page, the user selects the viewport (mobile, tablet, desktop), state (for example, menu open or form error), and evidence to compare. Each comparison is tied to a specific URL, design frame/file version, viewport, and captured timestamp.
4. The workspace shows the rendered page and design together. The user can switch between side-by-side, synchronized overlay with opacity slider, and a draggable wipe. Zoom and scroll stay synchronized where possible. Differences can be pinned to a location as an annotated finding.
5. BuildPath checks the page against mapped requirements, design, observed behavior, API/backend evidence, and quality checks. AI explains likely mismatches and suggests fixes with evidence. The user can accept, dismiss, edit, or assign findings.
6. The report shows coverage and gaps by page and by source. The user can export PDF/HTML/JSON or turn selected findings into project tasks. Fixes require a separate explicit action and create a new version to review.

## Scope and evidence

Each finding must identify its source and evidence. Supported comparison dimensions:

- **Requirements:** requirement ID, expected behavior, observed result, and pass/fail/unknown.
- **Design:** Figma frame or uploaded design, viewport, image diff regions, layout and content observations.
- **Backend/API:** documented API contract or connected test environment, request/response status, and relevant sanitized evidence. BuildPath must not claim backend correctness from a screenshot alone.
- **Working site:** URL, route, viewport, capture time, browser/device profile, console/network errors when available.
- **Quality:** accessibility, responsive layout, performance, SEO, and security checks applicable to the project.

AI results are suggestions, not approvals. Every AI finding includes a confidence label, a short explanation, the evidence used, and limitations. Unknown or inaccessible checks remain **not verified**; they are never silently counted as passed. Human review can change status, with author and timestamp recorded.

## Page and state mapping

A review has a source snapshot and a set of page comparisons. A page mapping contains:
- normalized same-origin route and optional query/state recipe;
- design source type (Figma frame or uploaded asset) and immutable source version;
- target viewport dimensions and device scale;
- optional requirement IDs and API operations;
- capture reference and comparison status.

If a Figma frame changes, the old comparison remains reproducible and is marked stale until recaptured against the new design version. URL discovery never crawls outside the explicitly allowed origin. Authenticated pages require a user-authorized test session; credentials and cookies are not included in reports.

## MVP behavior

MVP supports:
- entering a public or reachable preview URL and discovering same-origin pages from sitemap and links;
- manually adding a route when discovery misses it;
- uploading PNG/JPEG designs per page;
- page-to-design mapping and side-by-side, overlay, and wipe comparison;
- screenshot capture at agreed viewport sizes, manual annotations, and a downloadable report;
- requirement checklist mapping and AI-assisted visual summary of the captured evidence.

Figma OAuth, interactive-state scripts, authenticated browser sessions, live backend contract execution, and automated repair are later capabilities. The MVP must show them as unavailable instead of implying they ran.

## Security and privacy requirements

- URL analysis is an SSRF-sensitive operation. Resolve and validate every redirect and DNS result; reject loopback, private, link-local, multicast, reserved, and metadata-service IPs; allow only HTTP(S); limit redirect count, response size, content type, page count, depth, and execution time. Revalidate DNS at connection time and block access to internal networks.
- Use an isolated, disposable browser worker with no platform secrets, no host filesystem access, restricted egress, resource limits, and a clean profile per review. Do not execute uploaded HTML/JS as trusted content.
- Figma authorization uses least-privilege read access and an explicit file selection. Store tokens encrypted, redact them from logs, support disconnect/revoke, and fetch only selected files/frames.
- Uploaded files are scanned, type-sniffed, size-limited, stored privately, and served through short-lived authorized links. Escape user-provided labels and comments.
- Reports inherit project access controls. Never expose private previews, cookies, source files, or raw API payloads to other project members without permission.
- AI receives only the minimum necessary, sanitized evidence. Treat page text, designs, and API responses as untrusted content; they cannot override system policy or instructions. Do not send secrets or credentials to AI.
- Analysis is read-only. Any requested code change, backend call with side effects, deployment, or publishing requires a separate action and the project's existing approval flow.

## Data model additions

- ReviewRun: project, owner, status, source snapshot, started/finished timestamps, configuration, and tool versions.
- ReviewPage: review run, route, viewport, state recipe, capture asset/version, and status.
- DesignSource: type, provider file/frame or uploaded asset, immutable version identifier, access metadata.
- PageDesignMapping: page, design source, requirement IDs, API operation IDs, and mapping author.
- ReviewFinding: category, severity, status, explanation, confidence, evidence references, page coordinates, assigned user, and audit timestamps.
- ReviewReport: immutable version, coverage summary, limitations, generated formats, and source snapshot references.

All entities are tenant/project scoped and use existing authorization/audit rules. Large captures and uploads live in private object storage; relational rows contain metadata and checksums.

## API additions

The OpenAPI contract is in builder-back at apps/api/openapi.yaml. Its server base is /api/v1; the following examples show the full path:

- POST /api/v1/projects/{projectId}/reviews — create a read-only review run.
- POST /api/v1/reviews/{reviewId}/sources/url — register URL after validation.
- POST /api/v1/reviews/{reviewId}/sources/design-upload — request a signed upload and register the verified asset.
- POST /api/v1/reviews/{reviewId}/sources/figma — later: attach an explicitly authorized Figma file.
- POST /api/v1/reviews/{reviewId}/sources/design-upload/{assetId}/complete — verify the uploaded asset before use.
- POST /api/v1/reviews/{reviewId}/pages/discover — enqueue bounded same-origin discovery.
- PUT /api/v1/reviews/{reviewId}/page-mappings/{pageId} — pair route, design, viewport, requirement IDs, and API operation IDs.
- POST /api/v1/reviews/{reviewId}/captures — enqueue a capture for a mapped page and viewport.
- GET /api/v1/reviews/{reviewId} and GET /api/v1/reviews/{reviewId}/report — retrieve status and versioned report.
- PATCH /api/v1/reviews/{reviewId}/findings/{findingId} — update status, comment, or assignment with audit history.
- POST /api/v1/reviews/{reviewId}/exports — generate authorized PDF, HTML, or JSON report.

Long-running discovery, browser capture, image comparison, and AI review are asynchronous jobs with idempotency keys, bounded retries, progress events, and explicit terminal failure reasons. Every endpoint checks project membership and permission. No endpoint accepts arbitrary browser commands or unrestricted network targets.

## Acceptance criteria

- A user can analyze a URL without creating a development workflow, map at least two pages to uploaded designs, compare at mobile and desktop sizes, annotate a difference, and export a report.
- A report identifies exact route, design version, viewport, capture time, evidence, test coverage, limitations, and review status.
- The same run is reproducible from its immutable source snapshot; stale designs are flagged.
- Private or dangerous network targets are rejected, including redirects to private IPs and DNS rebinding attempts.
- Analysis cannot change source code, production data, or deployment state.
- The interface works on a phone: page list, compare modes, annotations, and report remain usable without horizontal page scrolling.
- Unavailable checks and low-confidence AI suggestions are clearly labelled and cannot be counted as passing acceptance criteria.

## Implementation order

1. Add review entities and OpenAPI contracts; implement authorization and read-only job state.
2. Implement URL validation, bounded page discovery, private uploads, and immutable snapshots.
3. Build the responsive page mapping and visual comparison workspace.
4. Add deterministic report generation and evidence-linked manual findings.
5. Add AI summaries behind feature flags and only after sanitization, privacy, and confidence handling are in place.
6. Add Figma read integration and authenticated-state capture after the MVP's isolation and consent flows are verified.


## Базовые security и performance доказательства

Каждый захват публичной страницы фиксирует HTTPS, наличие выбранных заголовков защиты (HSTS, CSP, X-Frame-Options или frame-ancestors, nosniff, Referrer-Policy, Permissions-Policy), browser navigation time, TTFB, объём переданных ресурсов и их число. Quality gate учитывает эти данные и помечает отсутствующие обязательные захваты как непроверенные.

Это диагностический baseline по одному анонимному захвату. Он не выполняет поиск уязвимостей приложения/зависимостей, тесты TLS-конфигурации, полный Lighthouse/Core Web Vitals audit или проверку авторизованных страниц. Положительный результат отчёта не разрешает публикацию.
