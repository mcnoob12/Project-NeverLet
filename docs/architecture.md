# KalTex Architecture and Operating Constraints

Status: V1 reference architecture

Last updated: 2026-09-02

For the implementation-grade design, exact authentication flow, domain schemas, route contract, recovery behavior, and required technical spikes, read `system-design.md`.

## Goal

Serve roughly 1,000 to 10,000 active researchers with no managed servers, minimal fixed cost, and automatic scaling. The principle is not literally zero spend. It is predictable, low spend that grows slowly with useful research activity rather than traffic egress or idle infrastructure.

The supplied MongoDB credits are a strategic advantage, but they do not justify adding database complexity where a Cloudflare primitive is better suited.

## Chosen architecture

```text
Public reader or researcher
        |
        v
Cloudflare CDN, WAF, Turnstile, and Workers Static Assets
        |
        v
KalTex application Worker (TypeScript / React Router framework mode)
   |             |                 |                  |
   v             v                 v                  v
MongoDB       Cloudflare R2     Cloudflare Queues     Resend
Atlas Flex    private PDFs      short async jobs      transactional email
metadata      and file streaming and retries
   |
   v
Atlas Search for title, abstract, author, and topic search

Browser -> PostHog Cloud only for consented, privacy-conscious product analytics
```

Use one Cloudflare Worker deployment with Static Assets for the web application and API routes. This is preferable to splitting the V1 application between Cloudflare Pages and a separate Worker: it retains serverless static hosting while making authentication, public page rendering, and API behavior one deployable unit. React Router framework mode provides SSR for indexable public paper/profile pages and an app-like experience for the signed-in product.

Do not introduce a VPS, container cluster, self-hosted queue, self-hosted search engine, or self-hosted observability stack for V1.

## Services and their jobs

| Service | Decision | Responsibility | Explicit non-use |
| --- | --- | --- | --- |
| Cloudflare Workers + Static Assets | Use | Public rendering, API routes, auth/session handling, moderation UI, signed upload issuance, edge caching | Heavy PDF conversion or scientific analysis |
| Cloudflare WAF + Turnstile | Use | Bot resistance, rate limiting, sign-up and submission abuse prevention | Identity verification or content correctness |
| Cloudflare R2 | Use | PDF storage and global delivery | Relational metadata or search |
| MongoDB Atlas Flex | Use with existing credits | Accounts, profiles, papers, versions, review records, references, follows, saves, aggregate stats | Raw clickstream or PDF binaries |
| MongoDB Atlas Search | Use | Search titles, abstracts, author names, tags, and topics | A separate search vendor in V1 |
| Cloudflare Queues | Use sparingly | Retryable email dispatch, stats batching, reference resolution, review notifications | Long-running document analysis |
| Resend | Use | Account, publishing, and security email | Marketing automation or approval by email reply |
| PostHog Cloud | Use selectively | Aggregated product funnels and feature adoption | Paper-reader surveillance or sole system monitoring |
| Cloudflare Logs/Analytics | Use | Service errors, latency, Worker/R2 usage, basic operational diagnostics | Long-term full-fidelity log warehouse |

## Why this is a better fit than the original Pages plus Workers phrasing

Cloudflare's current Workers platform supports Static Assets and provides a migration path from Pages to Workers. A unified Worker retains the cost and operations advantages of Pages-style static delivery, with fewer boundaries around SSR and APIs. It also avoids adding Next.js plus an adapter before there is a product reason to need it.

Use React, TypeScript, and React Router framework mode. Do not choose Next.js by default. A future team may choose OpenNext only if a concrete Next.js capability outweighs the extra runtime and deployment complexity.

## Storage layout

Use two private R2 buckets:

- `kaltex-submissions`: private quarantine for incomplete, submitted, rejected, and not-yet-published PDFs.
- `kaltex-published-papers`: immutable published PDFs, delivered through a Worker route on `files.kaltex.org`.

Object-key convention:

```text
submissions/{submissionId}/source.pdf
papers/{paperId}/v{versionNumber}/paper.pdf
papers/{paperId}/v{versionNumber}/manifest.json
```

The Worker issues short-lived, one-purpose, size-bound signed upload URLs after creating a `draft` or `submitted` record. The browser uploads directly to the quarantine bucket; it never streams an uploaded PDF through the application Worker. On publication, a privileged Worker copies the exact reviewed object to the private published bucket, records its SHA-256 checksum, and writes the immutable version record.

The `files.kaltex.org` route checks publication state, streams the private R2 object without buffering it, and uses a cache-friendly immutable version URL. A new version gets a new object key and does not invalidate the old object.

## Request flows

### Public paper reading

```text
Reader -> /p/{kaltexId} -> Worker cache -> MongoDB metadata -> SSR response
Reader -> files.kaltex.org/content/p/{kaltexId}/v/{version}.pdf -> Worker state check -> R2 stream/CDN
```

The Worker caches anonymous public metadata responses aggressively but purges or revalidates on new publication, revision, withdrawal, or major metadata correction. PDFs are streamed from private R2 through a lightweight state-checking route. This costs a Worker request but nearly no file-processing CPU and no Cloudflare egress.

### Submission

```text
Researcher -> authenticated submission draft -> MongoDB
Researcher -> Turnstile-validated request for upload URL -> Worker
Researcher -> direct PDF upload -> private R2 quarantine
Researcher -> finalize submission -> Worker preflight -> MongoDB intake-checking
Worker -> queue -> triage and reviewer notification
Reviewer decision -> Worker copies file to private published R2 + publishes MongoDB version
Worker -> queue -> Resend notice, search indexing, citation resolution, cache invalidation
```

### Email and account flow

```text
User requests sign-in -> rate-limited Worker -> hashed short-lived token in MongoDB
Worker/queue -> Resend -> signed magic-link URL
User opens link -> Worker validates once -> signed secure session cookie
```

Use email only as a notification and authentication transport. All approval or moderation actions occur in the authenticated dashboard and are written to the audit log.

## MongoDB model

The source of truth for all durable relational metadata is MongoDB Atlas Flex. Use explicit schemas, database-level validation where practical, and repository-level TypeScript validators. Do not allow unbounded arbitrary documents in public user fields.

### Essential collections

| Collection | Holds | High-value indexes |
| --- | --- | --- |
| `users` | account, email state, roles, public profile state | normalized email unique, handle unique, external identifiers sparse unique |
| `sessions` | revocable user sessions | userId, expiresAt TTL |
| `auth_tokens` | hashed magic-link and verification tokens | tokenHash unique, expiresAt TTL |
| `papers` | stable paper record and latest version pointer | kaltexId unique, authorIds, status, publishedAt, topic ids |
| `paper_versions` | immutable metadata and R2 object reference for each version | paperId plus versionNumber unique, checksum |
| `submissions` | mutable pre-publication draft and intake state | ownerId, status, createdAt |
| `references` | normalized references and internal resolution result | citingVersionId, resolvedPaperId |
| `follows` | reader follows researcher or topic | followerId plus target unique, targetId |
| `saves` | private reader library items | userId plus paperId unique |
| `review_cases` | review assignment, rationale, decisions | status, priority, assignedTo, createdAt |
| `reports` | public concerns and abuse reports | target type/id, status, createdAt |
| `paper_stats_daily` | aggregate daily paper metrics | paperId plus date unique |
| `audit_log` | security-sensitive decisions and admin actions | actorId, target, createdAt |

### Search

Create Atlas Search indexes for title, abstract, author display names, topics, disciplines, and keywords. Keep a denormalized, search-safe author and latest-version projection on `papers` so public search does not require joins over many collections.

Search results return published papers only. Drafts, review records, email addresses, IP-derived data, moderation notes, and unlisted identities are never searchable.

### MongoDB connection risk and required spike

Cloudflare Workers can make TCP connections to database wire protocols, including MongoDB, but Worker invocations do not provide traditional long-lived server connection pooling. Before building product features, implement and benchmark a minimal Worker-to-Atlas Flex proof of connectivity in the intended region, under realistic concurrency.

The spike must verify:

1. TLS connection reliability with the current official MongoDB Node driver and current Workers Node compatibility.
2. Median and tail latency for common metadata reads and writes.
3. Correct handling of Atlas network-access policy without exposing administrative credentials.
4. A disciplined connection strategy, timeouts, retry policy, and a low `maxPoolSize` appropriate to the runtime.
5. Atlas Search behavior and cost under representative queries.

Do not invent a permanent server to hide a failed proof of concept. If the Worker-to-Atlas path is unsuitable, make an explicit architecture decision before proceeding: either use a managed serverless API boundary or reconsider the database choice. The product architecture assumes MongoDB first because the available credits make it economically sensible, but the connection spike is a real go/no-go gate.

## Security baseline

### Edge and account safety

- Turnstile on sign-up, magic-link requests, passwordless recovery, and upload initiation.
- Cloudflare rate limits by route, account, and risk signals. Stricter limits apply to unauthenticated endpoints.
- Secure, HttpOnly, SameSite session cookies; CSRF protection for state-changing browser requests.
- Secrets live only in Cloudflare secrets and the deployment environment, never in source code or client bundles.
- Least-privilege MongoDB application user and separate credentials per environment.
- Separate reviewer/admin roles, server-side authorization, and immutable audit-log writes.
- Idempotency keys for publication actions, email callbacks, R2 promotion, and payment-free but consequential moderation actions.

### File safety

- PDF only, with strict MIME and magic-byte validation.
- Reject encrypted, password-protected, corrupted, oversized, and unsupported PDFs.
- Quarantine uploads until publication; no public R2 path before a successful decision.
- Store checksum, byte size, page count, and uploader identity with every version.
- Treat all extracted PDF content as untrusted; escape it before display.
- Never execute uploaded content or render arbitrary source files on the platform.

Virus scanning and deep document analysis are not assumed to be free or reliable enough for the V1 request path. If adopted, they must run as isolated asynchronous jobs and have a defined privacy policy. They cannot block the Worker route for a long period.

## Review operations that stay small

KalTex cannot promise that no humans will ever moderate research. The goal is to concentrate human attention on uncertainty.

Use a risk-based queue:

- Trusted identity, clean history, complete metadata, valid PDF, and no risk flags: eligible for expedited publication after automated intake and periodic sampling.
- First-time, incomplete, duplicate-looking, heavily flagged, or impersonation-risk submissions: human review required.
- Reported content: stays public unless a moderator determines immediate restriction is necessary under published policy.

The reviewer UI needs only a queue, submission metadata, file preview/download, check results, decision buttons, reason templates, internal notes, and audit history. It does not need an elaborate workflow engine.

## Observability and analytics decision

Start with PostHog Cloud for product analytics and feature flags only. Track low-cardinality events such as account created, profile completed, submission started, submission finalized, paper published, paper opened, PDF downloaded, paper saved, researcher followed, and search performed. Do not send paper abstract text, email addresses, IP addresses, raw query text, or moderation data to PostHog.

Use Cloudflare's native Worker observability, error logs, usage reporting, and billing alerts for operational awareness. Keep structured logs concise and free of sensitive data.

Do not adopt Datadog or Grafana in V1. Datadog would create a significant recurring cost and Grafana introduces either a hosted bill or a system to run. Reconsider one of them only after there is a concrete operational question that Cloudflare logs and PostHog cannot answer. Cloudflare can export OpenTelemetry data to Grafana Cloud later if needed.

Essential alerts:

- Worker error rate and latency budget breach.
- R2 storage/operation anomaly.
- MongoDB Atlas credit or monthly-cost threshold.
- Submission queue age above the published target.
- Resend bounce/complaint anomaly.
- Search failure rate.

## Cost guardrails

Costs must be designed around public readership, not merely researcher count. A successful paper can attract many downloads, and that must remain safe.

### Known platform baselines, verified 2026-09-02

- Cloudflare Workers Free allows 100,000 requests per day; Workers Paid has a $5 monthly minimum and includes 10 million requests plus 30 million CPU milliseconds per month.
- R2 Standard is $0.015 per GB-month after the first 10 GB-month free; it has zero Internet egress fees. The free tier includes 1 million Class A and 10 million Class B operations monthly.
- Atlas Serverless is discontinued. Atlas Flex is the low-operations replacement; it ranges from $8 to $30 per month before credits, includes 5 GB storage and is capped at $30 monthly.
- Resend's Free tier provides 3,000 emails per month with a 100-email/day cap; its Pro tier is $20 per month for 50,000 emails. Verify current pricing before enabling a paid plan.

These are planning inputs, not a promise. Service pricing changes. Billing alerts and quotas are mandatory before inviting users.

### Conservative monthly planning envelope

| Item | Early launch | 10,000 active researchers | Control |
| --- | --- | --- | --- |
| Cloudflare Worker and static hosting | $0 to $5 | typically $5 plus usage | cache public pages, cap CPU, keep static assets static |
| R2 PDFs | near $0 | usually low single digits for tens of GB | PDF-only, direct delivery, object limits |
| MongoDB Atlas Flex | covered by credits where available | $8 to $30 before credits | indexes, aggregates, budget alert |
| Transactional email | $0 under Free cap | $0 to $20 depending on notices | only necessary email, digest preference later |
| Analytics/observability | $0 initially | grow only with explicit need | PostHog event minimization; no Datadog/Grafana V1 |

The original 20 GB estimate assumes 10,000 total 2 MB PDFs. Plan for a wider range: research PDFs may be 5 to 20 MB, authors may revise, and readers may download repeatedly. R2's absence of egress charges is the decisive protection; operation counts and storage still require alerts.

### Hard budget controls

- Configure Cloudflare usage notifications and CPU limits per Worker.
- Put an explicit monthly spend alert and a lower warning threshold on Atlas.
- Set R2 object-size limits, write-rate limits, and lifecycle policy for abandoned quarantine uploads.
- Retain `draft`/abandoned quarantine files for a short documented window, then delete automatically.
- Queue email and use idempotency to prevent duplicate sends during retries.
- Make expensive external citation imports and content-analysis tools opt-in later work with their own budget.
- Add a circuit breaker for expensive endpoints such as export generation or search abuse.

## Deployment and environments

Use three environments: local development, preview, and production. Each gets distinct Cloudflare bindings, R2 buckets, MongoDB database, Resend keys/domains where practical, and analytics keys.

Production deploys from a protected main branch after automated tests. Preview deploys are used for every change that affects public pages, publishing state, authentication, or moderation.

Infrastructure configuration should be represented in version-controlled Cloudflare and environment configuration, with secrets injected separately. Do not make undocumented dashboard-only production changes.

## Ranking service boundary

`kaltex-ranking` is a separate repository because recommendation logic needs offline evaluation, explainable score components, and independent iteration. Do not deploy it as a separate always-on service for V1.

V1 integration can be a versioned package called by the application Worker against a bounded candidate set from MongoDB. Later, if candidate selection or offline computation becomes too expensive, introduce a queue-fed materialized feed service with a measured reason and budget.

## Implementation sequence

1. Perform the Cloudflare Worker to MongoDB Atlas Flex connectivity spike and record results.
2. Establish environments, secrets, R2 buckets, Turnstile, and billing alerts.
3. Build public paper/profile data model, SSR routes, SEO metadata, and direct PDF delivery.
4. Build account creation, magic links, optional ORCID linking, and profile editing.
5. Build draft/upload/preflight/review/publish state machine with an audit log.
6. Build explore/search, follows/saves, and internal citation edges.
7. Add aggregate statistics and the chronological feed.
8. Integrate the first version of `kaltex-ranking` for the optional ranked feed.
9. Add moderation reporting, withdrawal flow, alerts, and operational runbooks before broad invitation.

## Primary references

- [Cloudflare Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/)
- [Cloudflare database connections](https://developers.cloudflare.com/workers/databases/connecting-to-databases/)
- [MongoDB Atlas Flex pricing](https://www.mongodb.com/docs/atlas/billing/atlas-flex-costs/)
- [MongoDB Serverless to Flex migration](https://www.mongodb.com/docs/atlas/flex-migration/)
- [Resend pricing](https://resend.com/pricing)
- [Resend with Cloudflare Workers](https://resend.com/docs/send-with-cloudflare-workers)
