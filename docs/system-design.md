# KalTex System Design

Status: implementation-grade V1 system design

Owner: Kalaris Labs

Last updated: 2026-09-02

## Reading order

Read `../platform.md` for the promise and product boundaries. Read `architecture.md` for the short operational overview. This document is the detailed technical design: the exact services, data ownership, authentication, routes, state transitions, security controls, and recovery behavior.

Where this document differs from an earlier overview, this one wins. Its principal refinement is that published PDFs stay in a private R2 bucket and are streamed through a cacheable Worker file route. That gives KalTex a single place to enforce publication/withdrawal state while retaining R2's low-cost delivery.

## 1. System objectives

### Functional objectives

- Public readers can browse papers, profiles, versions, references, citations, topics, search, and feeds without an account.
- A researcher can use a personal email address to create an account, edit a profile, link ORCID, upload a PDF, submit it, revise it, and request withdrawal.
- A published paper has a stable KalTex ID, canonical URL, immutable public versions, citation exports, clear intake labels, and a durable public record.
- Staff can triage submissions and reports, request corrections, publish/reject/restrict work, and leave an auditable decision trail.
- Signed-in readers can save papers, follow researchers/topics, and select chronological or ranked feed views.

### Operational objectives

- No VPS, containers, cluster, self-hosted database, or self-managed search engine.
- Automatic scaling for 1,000 to 10,000 active researchers and potentially far more readers.
- The site remains useful when product analytics, email, ranking, or background work is delayed.
- Cost tracks useful activity without egress surprises; startup credits should cover all core runtime services.
- A small team can operate the platform without a permanent on-call function.

### Non-goals

- Peer review, DOI registration, journal workflows, comments, direct messages, social posts, and native apps.
- A technical claim that a paper is scientifically correct, plagiarism-free, AI-free, fully reproducible, or peer reviewed.
- Full text processing, document conversion, or expensive AI checks in a reader request.
- A bulk public API, open data dump, or external citation import in V1.

## 2. Concrete technology choices

| Concern | V1 choice | Why | Not chosen |
| --- | --- | --- | --- |
| Web UI and API | TypeScript, React Router framework mode, Vite, Cloudflare Workers Static Assets | SSR public pages and one deployable serverless application | Next.js/OpenNext until a concrete need exists |
| Domain validation | Zod and TypeScript domain commands | Explicit, testable boundaries for all untrusted input | Passing request bodies directly into database calls |
| Metadata database | MongoDB Atlas Flex and Atlas Search | Existing startup credits and document-shaped research data | Self-hosted MongoDB, SQL plus a separate search product |
| PDF storage | Cloudflare R2, two private buckets | Cheap immutable blobs, no egress fees, quarantine boundary | MongoDB GridFS, public storage bucket, S3 as source of truth |
| Browser uploads | R2 S3-compatible presigned PUT URLs | Upload bypasses Worker memory/CPU and limits capability to one object | Proxying PDFs through app endpoints |
| Async work | Cloudflare Queues plus MongoDB outbox | Bounded retries and durable side-effect recovery | Cron-only processing, an always-on worker, queues as source of truth |
| Authentication | Custom passwordless magic links, opaque sessions, Resend | No auth SaaS bill, no password surface, direct control of session semantics | Password login or a third-party identity provider for core accounts |
| Identity linking | ORCID OAuth authorization-code flow with PKCE | User-controlled authenticated researcher ID, optional for all | Institutional-email requirement |
| Product analytics | PostHog Cloud with minimal events and feature flags | Startup credits, relevant product feedback loop | Datadog/Grafana/self-hosted observability in V1 |
| Operational telemetry | Cloudflare logs, usage, and alerts | Native service-level data with minimal cost/ops | Separate log warehouse |
| Transactional email | Resend; Cloudflare Email Routing for inbound addresses | Straightforward Workers support and a small support footprint | Marketing suite or email replies as workflow actions |
| Ranking | Separate `kaltex-ranking` package/repository | Independent, explainable, testable scoring | A deployed recommendation microservice |
| AWS credits | Optional future Lambda document analysis only | Bounded, isolated experimentation outside critical path | Core API, storage, database, or CDN duplication |

## 3. Deployment topology

```text
                             +-----------------------------+
                             | PostHog Cloud                |
                             | product events / flags only  |
                             +-------------^---------------+
                                           |
Browser ------------------------------> Cloudflare edge
public reader / researcher                CDN, WAF, Turnstile
                                           |
                                           v
                                KalTex application Worker
                             React Router SSR + resource routes
               +-------------------+----------+--------------------+
               |                   |          |                    |
               v                   v          v                    v
         MongoDB Atlas Flex   R2 submissions  R2 published       Resend API
         metadata + Search    private         private             outbound email
               |                   ^          ^
               v                   |          |
           Mongo outbox       browser PUT     file route streams R2 body
               |
               v
          Cloudflare Queue
         retryable async work

Future, only after an explicit cost/privacy decision:
Cloudflare Queue -> authenticated AWS Lambda inspection job -> MongoDB result
```

The application Worker is the only application runtime. It receives web requests, controls permission/state, and streams file bodies. It does not buffer uploaded PDFs, expose R2 credentials, or run unbounded document analysis.

## 4. Domains and route classes

| Domain | Role | Notes |
| --- | --- | --- |
| `kaltex.org` | Public pages and signed-in product | Canonical site |
| `www.kaltex.org` | Redirect | Permanent redirect to canonical site |
| `files.kaltex.org` | PDF content routes on same Worker | R2 is not publicly mounted here |
| `mail.kaltex.org` | Transactional sender subdomain | DNS/mail only |

| Route class | Examples | Auth | Cache |
| --- | --- | --- | --- |
| Public SSR | `/`, `/p/:kaltexId`, `/r/:handle`, `/explore` | None | Anonymous public responses only |
| Public content | `/content/p/:id/v/:version.pdf`, `/cite/:id.bib`, `/sitemap.xml` | None | Versioned PDFs are immutable; HTML short-lived |
| Researcher workspace | `/publish/*`, `/dashboard`, `/library`, `/settings` | Live session/role | `no-store` |
| Staff workspace | `/ops/*` | Cloudflare Access plus live staff role | `no-store` |
| Provider callbacks | `/webhooks/resend`, `/auth/orcid/callback` | Signed provider data/state | `no-store` |
| JSON commands | `/api/*` | Command-specific | `no-store` by default |

Never place private author state, draft details, review notes, staff identity, or account information in a cacheable response.

## 5. Repository boundaries

Use a pnpm workspace, even if the first deployment is only `apps/web`.

```text
apps/
  web/                      application Worker and React Router routes
    app/routes/             page and resource route modules
    app/features/           UI and server-action feature modules
    worker/                 bindings, middleware, entrypoint
packages/
  domain/                   state machines, permissions, pure commands
  validation/               Zod schemas and normalized primitives
  db/                       Mongo repositories, indexes, migrations
  contracts/                HTTP/queue/email payloads and error codes
  email/                    templates and email send contracts
  test-fixtures/            deterministic papers, researchers, ranking fixtures
docs/
  architecture.md           operating overview
  system-design.md          this technical design
```

`kaltex-ranking` stays in a separate repository. It must export a versioned, deterministic scoring package and fixture suite. The application consumes that package; it must not contain duplicate score formulas in route/UI code.

Code rules:

- Resource routes validate input then call domain commands; they do not contain policy decisions.
- Domain commands receive repositories and a clock/random interface for deterministic tests.
- Only repositories access MongoDB. UI modules never execute a database query.
- All browser, provider, queue, and PDF-derived inputs pass through a Zod schema.
- A side effect occurs through an outbox event unless it is naturally idempotent and documented.

## 6. Cloudflare configuration

### Bindings and secrets

| Name | Type | Responsibility |
| --- | --- | --- |
| `SUBMISSIONS_BUCKET` | R2 binding | Private quarantine, failed/rejected submissions, abandoned drafts |
| `PUBLISHED_BUCKET` | R2 binding | Private immutable published PDFs and manifests |
| `VIEW_DEDUPE` | KV namespace | 36-hour view/download dedupe keys only |
| `ASYNC_EVENTS` | Queue producer/consumer | Typed background events |
| `MONGODB_URI` | Secret | Least-privilege Atlas application connection string |
| `SESSION_PEPPER` | Secret | HMAC session token hashes |
| `MAGIC_LINK_PEPPER` | Secret | HMAC magic-link hashes |
| `ORCID_STATE_PEPPER` | Secret | OAuth state integrity |
| `RESEND_API_KEY` | Secret | Outbound transactional email |
| `RESEND_WEBHOOK_SECRET` | Secret | Provider webhook verification |
| `R2_S3_ACCESS_KEY_ID` | Secret | Narrow presigning credential only |
| `R2_S3_SECRET_ACCESS_KEY` | Secret | Narrow presigning credential only |
| `ORCID_CLIENT_ID` | Environment variable | Registered public OAuth client ID |
| `ORCID_CLIENT_SECRET` | Secret | Registered OAuth client secret |
| `POSTHOG_KEY` | Environment variable | Browser project key, not a secret |

Preview and production use separate Cloudflare bindings, R2 buckets, Atlas databases, Resend keys/sender domains, ORCID redirect URIs, and PostHog projects. A preview must never send real email, use production tokens, or mutate production data.

### Request middleware order

```text
request ID
-> security headers
-> route classification
-> staff Access assertion for /ops/*
-> opaque session lookup when a session is present/required
-> route-specific rate limit and Turnstile verification
-> body-size limit and Zod validation
-> resource relationship + role permission check
-> route command
-> normalized problem response and safe structured log
```

Set a restrictive CSP, `X-Content-Type-Options: nosniff`, HSTS, `Referrer-Policy: strict-origin-when-cross-origin`, and restrictive Permissions Policy. Permit PostHog only after consent and only on the exact required origins.

## 7. Authentication and identity

### 7.1 Account decision

V1 uses passwordless email sign-in. Personal email is sufficient. An account is created on first successful link consumption, and email verification is inherent in that flow. Passwords are intentionally out of scope: KalTex avoids password storage, reset attacks, password policies, and credential stuffing.

### 7.2 Magic link protocol

```text
1. User submits email and a safe local return path.
2. Worker validates syntax, limits requests by IP/email hash/device risk, and invokes Turnstile if needed.
3. Worker always returns the same generic response, whether the account exists or not.
4. Worker generates 32 cryptographically random bytes with Web Crypto.
5. Worker stores only HMAC-SHA-256(token, MAGIC_LINK_PEPPER), purpose, return path,
   request ID, and 15-minute expiration in auth_tokens.
6. Worker writes SendMagicLink to the outbox; queue sends it through Resend.
7. GET /auth/verify displays confirmation but never consumes the token.
8. User performs a same-origin POST to consume the token, preventing mail scanners from spending it.
9. Worker atomically marks token used, creates/updates user, creates session, and redirects
   only to the stored local return path.
```

The generic response is: "If an account can use this address, a sign-in link will arrive shortly." Do not disclose account existence, review state, or user role.

### 7.3 Session protocol

Use opaque server-stored sessions, not browser JWTs.

- Generate 32 random bytes for a session token and store only its HMAC hash.
- Send raw token in `__Host-kaltex_session` with `Secure`, `HttpOnly`, `Path=/`, and `SameSite=Lax`.
- Set absolute expiry at 30 days; renew only inside a 7-day rolling window.
- Lookup the session on state-changing and staff routes; enforce `sessionsRevokedAfter`.
- Rotate session after sign-in and privilege elevation.
- Revoke all sessions on explicit sign-out-all, account restriction, or role change.

The `__Host-` prefix prevents accidental Domain-scoped cookies. Do not set a `Domain` attribute.

### 7.4 ORCID

ORCID is a linkable identity and never a requirement. Use OAuth authorization code with PKCE, a one-time 10-minute server-side state record, exact registered redirects, and the authenticated KalTex session as the link target.

```text
authenticated KalTex user
-> POST /api/identities/orcid/start
-> signed state plus PKCE verifier saved with TTL
-> ORCID authorization screen
-> callback validates state and exchanges code server-side
-> store authenticated ORCID iD + linkedAt + returned name snapshot
-> discard access token unless a future consented sync feature needs it
```

An ORCID iD is unique across active KalTex accounts. A manually typed ORCID/Google Scholar/ResearcherID URL is a self-attested profile link, never an `ORCID linked`/verified badge. Start in ORCID sandbox; move to production only after final domain/redirect configuration is locked.

### 7.5 Manual verification and staff access

Manual identity verification creates a review case with an auditable moderator outcome. Do not collect government identity documents in V1. The public label may say `Identity manually verified`; it never explains private evidence.

Staff routes require two gates: Cloudflare Access restricted to named Kalaris staff identities with MFA, plus a current KalTex session containing `reviewer`, `moderator`, or `admin`. Roles are never self-service and every role change is written to the audit log.

## 8. Authorization

Authorization is server-side and relationship-aware.

| Capability | Reader | Researcher | Reviewer | Moderator | Admin |
| --- | --- | --- | --- | --- | --- |
| Browse public records | Yes | Yes | Yes | Yes | Yes |
| Edit own profile | Account only | Yes | Yes | Yes | Yes |
| Draft/submit own paper | No | Yes | Yes | Yes | Yes |
| See own submission state | No | Yes | Yes | Yes | Yes |
| Review assigned case | No | No | Yes | Yes | Yes |
| Approve/reject/restrict | No | No | No | Yes | Yes |
| Manage roles/config | No | No | No | No | Yes |

Each action validates both role and resource relationship. A reviewer cannot browse arbitrary author material; a researcher cannot edit another author's draft; a moderator decision requires a reason code and audit entry.

## 9. Core identifiers and MongoDB records

Use ULID strings for all internal entities, queue events, requests, and R2 keys. Use `KTX:YYYY.NNNNN` as a stable public paper ID allocated from an atomic yearly counter. Gaps are acceptable. A paper ID is never reused.

### 9.1 Core collection table

| Collection | Responsibility | Essential indexes |
| --- | --- | --- |
| `users` | Private account state, email, role, security state | unique normalized email/hash; roles/status |
| `researcher_profiles` | Public profile projection | unique user ID/handle; public Atlas Search fields |
| `external_identities` | ORCID links and external URLs | unique provider+subject for ORCID |
| `sessions` | Opaque session hashes | unique token hash; user/expiry TTL |
| `auth_tokens` | Magic-link/OAuth state hashes | unique token hash; expiry TTL |
| `papers` | Stable published work record and latest projection | unique KalTex ID; author/status/date; search projection |
| `paper_versions` | Immutable version metadata and private R2 object reference | unique paper+version; document checksum lookup |
| `submissions` | Mutable author draft/intake state | owner/status/date; paper/version revision lookup |
| `review_cases` | Submission, report, identity, and appeal decisions | status/priority/date; subject lookup |
| `references` | Version-bound raw references and resolution result | citing version + ordinal unique |
| `citation_edges` | Resolved paper-to-paper relationships | citing version + cited paper unique |
| `follows`, `saves` | Private reader relationships | user + target unique |
| `feed_events` | First-publication/revision feed candidates | author/date and topic/date |
| `paper_stats_daily` | Aggregate approximate metrics | paper + day unique |
| `outbox_events` | Durable requested side effects | undelivered/date index |
| `audit_log` | Append-only privileged/publication actions | target/date and actor/date |
| `idempotency_keys` | Safe request replay | actor + route + key unique, expiry TTL |

### 9.2 Data-shape rules

- Keep private account data in `users`; never duplicate account email into a public profile.
- `papers` contains only the latest public search/listing projection; authoritative immutable history is in `paper_versions`.
- A `paper_version` contains the exact file checksum, R2 key, license, reproducibility metadata, and screening result for that version.
- Free-text review notes/reports are staff-only and never part of Atlas Search.
- Public search indexes only `papers` where status and visibility are public.
- Published version rows are immutable. Metadata corrections create a new version unless the correction is a narrowly defined non-scholarly display fix recorded in audit.

### 9.3 Publication data invariants

1. One `paper` has one immutable KalTex ID and at least one `paper_version` once published.
2. `latestVersionId` points to a published version with matching paper ID/number.
3. A public content route may stream only when both paper and version are `published`.
4. A citation edge is active only when its citing version is published and not withdrawn/restricted.
5. A trust label is derived from version screening/review data, never hand-typed on the profile.
6. An outbox event is created with every externally visible state change.

## 10. State machines and publication saga

### 10.1 Submission lifecycle

```text
draft -> uploading -> uploaded -> intake_checking
                                  |             |
                                  |             +-> needs_author_action -> uploaded
                                  |             +-> review_queue -> publish_approved
                                  |                                |        |
                                  |                                |        +-> rejected
                                  |                                +-> publication job
                                  +-> cancelled
```

Only named domain commands transition state. Every command checks actor, current state, structured metadata, and preconditions. Invalid transitions return a conflict and do not mutate data.

### 10.2 Paper lifecycle

```text
no paper record
  -> publishing (public ID reserved; version awaits file promotion)
  -> published (public page, content route, search and feed eligible)
  -> withdrawn (public tombstone; file handling per withdrawal policy)
  -> restricted (rare legal/safety/privacy hold)
```

`publishing` is never public. It exists to make a retryable file/database handoff explicit.

### 10.3 Publication saga

R2 and MongoDB cannot share one atomic transaction. Publication is therefore a recoverable saga:

1. Moderator approval runs a MongoDB transaction: reserve KalTex ID if new, create `paper_version` in `publishing`, set paper state `publishing`, and write `PublicationRequested` to `outbox_events`.
2. Queue consumer validates the quarantined PDF and streams it to `PUBLISHED_BUCKET` under a fresh immutable object key.
3. Consumer verifies object size/checksum. A failure is retryable and records no public state.
4. A MongoDB transaction marks the version/paper `published`, updates the latest-version projection, creates a `feed_event`, and writes `PaperPublished` to outbox.
5. Follow-up consumers send author email, resolve supplied references, update citation aggregates, and purge/warm public cache.

If a private R2 copy exists but step 4 fails, it cannot leak: file routes check MongoDB state. A scheduled reconciler either resumes publication or removes orphaned private copies.

### 10.4 Revision and withdrawal rules

- New work begins with `paperId = null`; a revision starts a separate `submission` pointing to the existing paper and next version number.
- Revision approval creates a new immutable version and moves the latest-version projection only after publication completes.
- First publication emits `paper_published`. A revision emits `paper_revised`, visually labelled as an update.
- Author withdrawal is a request, not a direct client-side state change. A moderator executes it with a reason code and audit entry.
- A withdrawn record retains its KalTex ID and tombstone. The exact PDF availability policy must be published before launch; the default should favor record clarity rather than silent disappearance.

## 11. PDF upload, storage, and inspection

### 11.1 Limits

Launch with configuration, never unexplained hardcoding:

```text
MAX_PDF_BYTES = 50 MiB
MAX_ACTIVE_UPLOADS_PER_USER = 3
UPLOAD_URL_TTL = 10 minutes
ABANDONED_DRAFT_RETENTION = 30 days
MAGIC_LINK_TTL = 15 minutes
```

Larger PDF exceptions need a documented staff workflow. Do not increase the platform-wide limit in response to an individual request.

### 11.2 Direct upload flow

```text
researcher creates draft
-> POST upload-grant
-> Worker verifies ownership, state, limits, and Turnstile risk
-> Worker creates random non-user-controlled R2 key + signed PUT URL
-> browser PUTs application/pdf directly to SUBMISSIONS_BUCKET
-> browser calls complete-upload
-> Worker HEADs R2 object, validates actual size/type/header and queues inspection
```

The browser never receives a general bucket credential. A presigned URL is a bearer capability: it expires quickly and covers one random key. The completion command consumes its upload grant; replacing a file creates a new key/grant. Client-declared size is advisory, actual object size is authoritative.

R2 CORS permits only KalTex preview/production origins and only required methods/headers. The R2 S3 credential used for presigning has narrowly scoped permissions and is stored only as a Worker secret.

### 11.3 Inspection policy

The bounded V1 intake worker performs only checks it can defend:

- PDF magic-byte/type and actual byte-size validation.
- Encrypted/password-protected/unreadable file detection and basic page count using a Worker-compatible parser under strict time/CPU limits.
- Required structured metadata, author data, license, and reproducibility URL validation.
- Exact checksum duplicate and title/abstract near-duplicate signals.
- Basic spam/fraud heuristics from account and submission velocity.

If a parser exceeds limits or fails, the platform requests a new file or sends the submission to review. It does not call the research invalid. It never renders arbitrary source files, executes embedded PDF content, or makes an automated AI-detector result into a final decision.

Full-text similarity, PDF repair, OCR, or complex scientific analysis is a future optional AWS Lambda experiment, triggered only after an explicit privacy/cost/accuracy decision. Its output is a triage signal, never a public trust badge or automatic rejection.

### 11.4 Content route

`GET` and `HEAD /content/p/:kaltexId/v/:version.pdf` work as follows:

1. Resolve paper/version from a small public-record cache or MongoDB.
2. Require both paper and version to be `published`.
3. Fetch the immutable private R2 object through `PUBLISHED_BUCKET` binding.
4. Stream the body without buffering it in Worker memory.
5. Respect legitimate Range requests, ETag, last-modified, and `application/pdf` headers.
6. Cache using KalTex ID/version; versioned file content is immutable.
7. Queue one privacy-preserving download metric after an eligible successful response.

This uses a Worker request but nearly no file-processing CPU and retains Cloudflare's no-egress R2 property. It is safer than exposing a public bucket and allows later restriction/withdrawal policy to take effect at one boundary.

## 12. HTTP API contract

All mutation endpoints use JSON, strict Zod validation, origin/CSRF validation, and a command-specific idempotency key. Errors use `application/problem+json` with a stable machine code and a safe human message.

### Authentication

| Route | Command | Important behavior |
| --- | --- | --- |
| `POST /api/auth/magic-link` | RequestMagicLink | Generic response, rate limited |
| `GET /auth/verify` | Render verification page | Does not consume one-time token |
| `POST /api/auth/consume` | ConsumeMagicLink | Creates/rotates session |
| `POST /api/auth/sign-out` | SignOut | Revokes current session |
| `POST /api/auth/sign-out-all` | SignOutAll | Requires recent session confirmation |
| `POST /api/identities/orcid/start` | StartOrcidLink | Authenticated user only |
| `GET /auth/orcid/callback` | FinishOrcidLink | State/PKCE verified; links live session |
| `DELETE /api/identities/:id` | UnlinkIdentity | Audited if it changes public badge |

### Researcher workspace

| Route | Command |
| --- | --- |
| `POST /api/submissions` | CreateDraft |
| `PATCH /api/submissions/:id` | UpdateDraft |
| `POST /api/submissions/:id/upload-grant` | CreateUploadGrant |
| `POST /api/submissions/:id/complete-upload` | CompleteUpload |
| `POST /api/submissions/:id/submit` | SubmitForIntake |
| `POST /api/submissions/:id/cancel` | CancelSubmission |
| `POST /api/papers/:id/revisions` | CreateRevisionDraft |
| `POST /api/papers/:id/withdrawal-request` | RequestWithdrawal |
| `PUT /api/profile` | UpdateProfile |
| `POST /api/library/saves/:id` | SavePaper |
| `DELETE /api/library/saves/:id` | UnsavePaper |
| `POST /api/follows` | FollowTarget |
| `DELETE /api/follows/:type/:id` | UnfollowTarget |

### Staff workspace

| Route | Command |
| --- | --- |
| `GET /api/ops/review-cases` | ListReviewCases |
| `POST /api/ops/review-cases/:id/assign` | AssignReviewCase |
| `POST /api/ops/review-cases/:id/request-changes` | RequestAuthorAction |
| `POST /api/ops/review-cases/:id/approve` | ApprovePublication |
| `POST /api/ops/review-cases/:id/reject` | RejectSubmission |
| `POST /api/ops/reports/:id/restrict` | RestrictPaperOrProfile |
| `POST /api/ops/papers/:id/withdraw` | ExecuteWithdrawal |

Every staff command writes actor, action, target, reason code, request ID, before/after summaries, and timestamp to `audit_log`.

### Resend webhook

`POST /webhooks/resend` verifies the provider signature before parsing the payload. Replayed events are deduplicated by provider event ID. Store actionable delivery/bounce/complaint state only. A bounce does not delete an account or change its scholarly identity; it asks the authenticated user to update their email later.

## 13. Search, citations, feed, and metrics

### Search

Atlas Search operates only on published `papers` projections. It supports title, abstract, author, KalTex ID, topic, and keyword, then filters by discipline, date, author, and trust-label context.

- Max query length: 256 characters; max page size: 50.
- Normalize Unicode/punctuation/case for lookup while preserving original display text.
- Autocomplete: title and author name only, not abstracts.
- No raw query string goes to PostHog or generic logs.
- Apply rate limits to prevent index scraping and surprise Atlas spend.

### Citations

References belong to a citing `paper_version`. Resolve in order: explicit KalTex ID, known DOI/canonical URL, then a high-confidence normalized title/year/first-author match. Leave ambiguous references unresolved. Do not fabricate a citation edge.

Active internal citation count includes only published, non-withdrawn/restricted citing versions. Historical edges remain auditable but inactive if their citing version stops counting. The V1 `Cited by` UI makes clear it is internal KalTex citation data.

### Feed

Chronological Following reads recent `feed_events` for followed researcher/topic IDs. It does not fan out a publication to every follower at write time.

The ranked feed retrieves a bounded candidate set, invokes a pinned `kaltex-ranking` version, enforces author/discipline diversity, and returns a short explanation. Ranking fixtures must prove that institution, follower count, country, email-domain prestige, and paid status do not influence score.

### Privacy-preserving metrics

For an eligible public page view or download, Worker derives `dailyViewerHash = HMAC(rotatingDailyKey, cfConnectingIp + normalizedUserAgent)`, then writes a 36-hour KV key scoped to paper/event/day. Only the first occurrence queues an aggregate metric event. Raw IP/user-agent values are never stored in MongoDB, PostHog, or generic logs.

Views/downloads are explicitly approximate. Saves and citation edges are exact first-party relationships; neither exposes reader identity publicly.

## 14. Outbox and queue

Use MongoDB as the durable source of truth and Cloudflare Queue as delivery machinery.

```ts
type AsyncEvent = {
  id: string;
  type:
    | "SendMagicLink"
    | "SubmissionReceived"
    | "RunIntakeChecks"
    | "PublicationRequested"
    | "PaperPublished"
    | "ResolveReferences"
    | "RecordPaperMetric"
    | "SendPublicationNotice"
    | "PurgePaperCache";
  occurredAt: string;
  requestId: string;
  schemaVersion: 1;
  payload: Record<string, unknown>;
};
```

1. A domain transaction writes the state change plus an `outbox_event`.
2. The request attempts to enqueue it after commit.
3. A scheduled Worker scans undelivered outbox events every minute and retries delivery.
4. Consumers are at-least-once and idempotent by event ID/natural unique key.
5. Terminal failures go to `failed_async_events`, raise an alert, and require a staff retry action.

Do not treat the queue as the source of truth. Never lose a critical activity because a queue send failed after a database update.

## 15. Screening, moderation, and reports

### 15.1 Triage signals

Risk is private triage information, not a public credibility score. Start with explainable signals:

- Email not verified or an untrusted first publication flow.
- Repeated failed upload/intake attempts.
- Required metadata missing or author identity inconsistencies.
- Exact checksum duplication or high title/abstract similarity.
- High account/submission velocity.
- Turnstile/rate-limit risk signals.
- Repeated credible reports.

Never use affiliation, country, language fluency, age, gender, follower count, email-domain prestige, or an automated AI detector as a sole decision factor.

### 15.2 Outcomes

| Outcome | System action | Author/public communication |
| --- | --- | --- |
| `needs_author_action` | Submission remains private, editable | Specific private correction request |
| `publish_approved` | Publication saga queued | Accurate intake labels only |
| `rejected` | Submission stays private, reason recorded | Private reason and appeal path |
| `withdrawn` | Paper tombstone and version policy applied | Public reason category |
| `restricted` | Content route refuses delivery | Staff/policy-defined notice where needed |

The review UI shows author-visible metadata, PDF preview/download, inspection result, duplicate signals, related reports, prior decision history, and a compact policy checklist. It requires a reason code for every final action. It never turns a speculative content detector into a verdict.

### 15.3 Reports and appeals

Reports may be submitted by signed-in users first; anonymous reporting can be added only with stronger abuse controls. A report creates a `review_case` once it meets a deduplication/triage threshold. Reporters are never exposed to authors.

Rejected, withdrawn, or restricted authors can request reconsideration. An appeal creates a new case and preserves the original audit history. Public launch requires a written appeals policy and named escalation owner.

## 16. Cache, consistency, and performance

| Resource | Caching decision | State consistency rule |
| --- | --- | --- |
| PDF version | Edge-cached by immutable ID/version route | Route verifies published status before first stream; object never changes |
| Published paper page | Short anonymous cache with stale-while-revalidate | Publication/revision/withdrawal emits cache work |
| Public profile | Short anonymous cache | Profile command emits cache work |
| Search | No shared cache initially | Atlas Search authoritative projection |
| Feed/library/dashboard | Private/no shared cache | Always session-aware |
| Auth/staff/workspace | `Cache-Control: no-store` | Never cache |

Use short TTL plus purposeful invalidation rather than a complicated globally synchronized cache fabric. For emergency restriction, the content/page route checks an authoritative restriction overlay before using a cached record. The operational runbook must include a Cloudflare emergency route block for legal/safety incidents.

### Performance budgets

- Public cached HTML: target p95 under 500 ms at edge.
- Dynamic metadata page: target p95 under 1.5 s excluding an external Mongo incident.
- File route: begin streaming quickly; never buffer the document.
- Mutation command: return after durable database/outbox commit; do not wait for email, search, stats, or citations.
- Ranking: fixed candidate window and bounded CPU; fall back to chronological/recent results on failure.

## 17. Email design

### Sender/inbound addresses

| Address | Use |
| --- | --- |
| `auth@mail.kaltex.org` | Magic links, account and security messages |
| `publish@mail.kaltex.org` | Submission, author-action, decision, and publication notices |
| `support@kaltex.org` | Cloudflare Email Routing to staff inbox |
| `abuse@kaltex.org` | Cloudflare Email Routing to restricted staff inbox |

Configure SPF, DKIM, and DMARC before user email launches. Start DMARC in monitor mode, verify legitimate traffic, then move toward quarantine/reject. All email links point only to `kaltex.org`. Email replies never execute a staff approval.

### Template rules

- Magic link: expiry, safe-device guidance, no sensitive profile content.
- Submission receipt: title and submission ID, but no promise of publication timing.
- Author action: plain reason, secure dashboard action, no public detail.
- Publication/rejection/withdrawal: outcome, reason category, secure appeal path if eligible.
- Keep templates accessible and minimal. Never put raw moderation evidence in email.

## 18. Privacy and retention

### Data classes

| Class | Examples | Handling |
| --- | --- | --- |
| Durable public scholarly record | Published metadata, authors, version/citation data | Public and stable by design |
| Private account data | Email, sessions, auth tokens | MongoDB only; excluded from analytics/log export |
| Sensitive operations | Review notes, reports, appeals, audit | Staff-only, least privilege, policy retention required |
| Ephemeral metrics | Daily viewer hashes, queue idempotency | Short TTL; no raw IP persistence |
| Product analytics | Coarse product events | PostHog after consent; no paper text/PII |

### Minimum defaults

- Magic-link/OAuth state: 15 minutes, TTL delete.
- Expired/revoked sessions: retain 30 days for security investigation, then TTL delete.
- View/download dedupe key: 36 hours.
- Queue idempotency record: 7 days.
- Abandoned draft PDF: 30 days, then automatic R2 deletion.
- Published record: durable unless a documented legal/privacy/safety exception applies.

Review-note, report, and audit retention needs legal/policy approval before launch. Do not quietly make sensitive data permanent.

## 19. Observability and alerts

### Cloudflare signals

- Worker requests, CPU, errors, tail latency, deployment changes.
- R2 storage, operation count, failed reads/writes, object growth.
- Queue backlog, retries, dead-letter failures.
- WAF, Turnstile, rate-limit, and suspicious traffic signals.
- Credit/billing consumption.

### PostHog event schema

Use only low-cardinality product events that could alter a product decision:

```text
account_magic_link_requested
account_authenticated
profile_completed
submission_draft_created
submission_uploaded
submission_submitted
submission_needs_author_action
paper_published
paper_page_opened
paper_pdf_downloaded
paper_saved
researcher_followed
search_performed              // never raw search terms
feed_mode_selected
```

Use feature flags for reversible changes such as ranked-feed availability, collections, and reviewer UI. Do not feature-flag authorization or content policy in a way that creates unequal rules for equivalent researchers.

### Alerts and response

| Alert | Trigger | Initial response |
| --- | --- | --- |
| Worker errors | Elevated 5xx for 5 minutes | Inspect release correlation; rollback first when clear |
| Queue age | Oldest event over 15 minutes | Inspect queue/consumer/provider and replay from outbox |
| Intake backlog | Beyond published response target | Rebalance reviewer capacity/triage criteria |
| Credit consumption | 50%, 80%, 95% thresholds | Reduce expensive behavior before expiry |
| Email health | Bounce/complaint deviation | Pause nonessential mail, inspect deliverability |
| Atlas failures/latency | Above defined SLO | Inspect indexes, connection behavior, recent query changes |

Do not start with Datadog or Grafana. Cloudflare service signals and PostHog provide enough observability until a particular unanswered operational question justifies new cost/complexity.

## 20. Reliability, backup, and recovery

### Degradation behavior

- Resend unavailable: state commits and queue retries; sign-in may be delayed but not lost.
- Queue unavailable: state commits with durable outbox records; scheduled dispatcher resumes later.
- PostHog unavailable: ignore/short-circuit analytics; never block public pages or commands.
- Ranking unavailable: return chronological/recent results and preserve Following feed.
- Atlas unavailable: dynamic routes fail safely; cached public content may remain readable by cache policy.

### Backups and reconciliation

- Enable the strongest Atlas backup option compatible with the selected credited tier. Export and test metadata backups to a restricted R2 backup bucket on a documented schedule.
- Published PDFs are immutable. Keep a monthly inventory manifest; adopt a second copy only when storage data proves it is worth the added cost.
- A daily reconciliation job verifies each published version's R2 object exists with expected checksum/length and opens an operational alert on mismatch.
- Infrastructure configuration lives in version control; secrets remain in managed secret stores and an encrypted break-glass procedure.

### Initial objectives

- No accepted public-paper record loss.
- Metadata recovery from tested snapshot within one business day.
- Queue recovery by replaying `outbox_events`, never from in-memory logs.
- Any R2/Mongo mismatch receives staff attention before it becomes a reader-visible inconsistency.

## 21. Cost rules

| Provider credit | Use it for | Do not use it for |
| --- | --- | --- |
| Cloudflare Startups | Worker, R2, Queues, CDN, security/rate limits, preview deploys | Unbounded Workers AI/document processing |
| MongoDB for Startups | Atlas Flex, Atlas Search, development/preview databases | Raw clickstream storage or unindexed broad scans |
| PostHog for Startups | Analytics, flags, experiments, error tracking | Scroll/event firehose, raw queries, paper contents, automatic replay on readers |
| AWS credits | Later bounded Lambda inspection experiment | Core API/database/CDN/canonical PDFs |

Enforce cursor pagination and database indexes for all lists. Cap search length/page size. Score a fixed feed candidate set. Size-limit every PDF before granting an upload URL. Do not send email for views/saves/citations. Set spend alerts at 50%, 80%, and 95%; configure Worker CPU limits and PostHog billing limits. AWS budget alerts are not an availability/cost circuit breaker, so AWS must stay off the high-volume critical path.

## 22. Required technical spikes

### A. Atlas Flex from Workers

Prove the current official MongoDB Node driver works from the deployed Cloudflare Worker in the intended region. Measure P50/P95 for session, paper, profile, and search operations under representative concurrency. Confirm TLS, network policy, timeouts, retry behavior, and Atlas Search. This is a go/no-go prerequisite, not a background optimization.

### B. Private R2 direct upload

Test presigned browser PUT, strict CORS, expiry/replay, cancellation, actual byte-size validation, and no secret exposure in preview and production-like environments.

### C. PDF inspection

Benchmark a Worker-compatible parser against ordinary, malformed, encrypted, image-heavy, and deliberately expensive PDFs. Establish strict CPU/memory bounds and decide which cases are automated vs review-only.

### D. Auth/email abuse

Test scanner-safe magic link consumption, expiration/replay, session revocation, generic request responses, rate limits, Turnstile, Resend webhook verification, and SPF/DKIM/DMARC delivery.

### E. Publication recovery

Force failures between every publication-saga step. Prove retry produces exactly one public version, one KalTex ID, correct cache state, correct file authorization, and no duplicated email/citation event.

## 23. Test plan

| Layer | Tooling | Must cover |
| --- | --- | --- |
| Domain unit | Vitest | State machines, permissions, labels, ranking input exclusions |
| Repository integration | Dedicated Atlas dev DB | Indexes, unique constraints, transaction/idempotency, search projection |
| Worker integration | Cloudflare Workers test runtime | Bindings, cache, R2 stream, queue/outbox handlers |
| Browser end-to-end | Playwright | Sign-in mock, upload, submit, staff decision, public read, withdrawal |
| Security | Automated plus manual | IDOR, CSRF, token replay, role escalation, webhook signature, upload abuse |
| Performance/cost | Controlled preview load | Search/feed/paper routes, cache hit rate, queue lag, operation counts |

Every bug involving authorization, publication/withdrawal state, version history, file visibility, citation counts, or token consumption gets a regression test before close.

## 24. Implementation order

1. Complete all five technical spikes and record measured outcomes here.
2. Bootstrap Worker/React Router, bindings, secrets, preview deployments, security headers, and provider budget alerts.
3. Build account/profile schemas, magic links, opaque sessions, ORCID linking, Access staff gate, and Resend webhooks.
4. Build public paper/profile projection, SSR, SEO metadata, sitemap, content file route, and Atlas Search.
5. Build draft/upload/inspection/review/publication saga, outbox, audit log, and staff review UI.
6. Build revisions, withdrawal requests, reports, appeals, profile collections, follows, and saves.
7. Build references/citation edges, daily aggregate metrics, chronological feed, and ranking package integration behind a flag.
8. Publish operations runbooks, content policy, privacy policy, terms, withdrawal policy, retention policy, and appeal policy before broad public invitation.

## 25. Primary references

- [Cloudflare Workers database connections](https://developers.cloudflare.com/workers/databases/connecting-to-databases/)
- [Cloudflare R2 presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [Cloudflare R2 Workers API](https://developers.cloudflare.com/r2/api/workers/workers-api-reference/)
- [MongoDB Atlas Flex](https://www.mongodb.com/docs/atlas/billing/atlas-flex-costs/)
- [ORCID authenticated iD flow](https://info.orcid.org/documentation/api-tutorials/api-tutorial-get-and-authenticated-orcid-id/)
- [ORCID sign-in guidance](https://info.orcid.org/documentation/integration-guide/orcid-oauth-sign-in-guidelines/)
- [Resend with Cloudflare Workers](https://resend.com/docs/send-with-cloudflare-workers)
- [Cloudflare for Startups](https://www.cloudflare.com/startups/)
- [MongoDB for Startups](https://www.mongodb.com/solutions/startups)
- [PostHog for Startups](https://posthog.com/startups)
