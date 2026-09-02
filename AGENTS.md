# KalTex Agent Guide

Read `platform.md` first. It is the canonical product brief. Read `docs/system-design.md` before changing infrastructure, authentication, uploads, storage, search, observability, ranking, or cost controls. `docs/architecture.md` is the concise operating overview.

## Product posture

KalTex is a radically open, paper-first preprint platform for independent researchers. It is not a journal, a peer-review system, a general social network, or a pay-to-publish product.

The key user is an independent researcher with a personal email address and no institutional affiliation. Never make institutional email, affiliation, or academic prestige a requirement or ranking signal.

## Non-negotiables

- Published paper records and their version history must be durable and public.
- A new version creates a new immutable record. Never overwrite an old published PDF.
- Never describe intake checks as peer review, scientific verification, plagiarism-free, AI-free, or reproducible unless that exact work actually occurred.
- Keep reader access open without an account.
- Do not add messages, comments, general posts, follower-chasing mechanics, or paid visibility to V1.
- Citations are resolved paper-to-paper relationships, never engagement events.
- Never put uploaded PDFs through an app server. Use signed direct browser upload to private R2, then stream published private R2 objects through the state-checking content route.
- Preserve privacy: do not expose reader identities or store unnecessary behavioral data.
- Avoid self-managed servers, databases, search clusters, workers, queues, or monitoring infrastructure.

## User experience rules

- The core screen is a research feed where paper cards are the posts.
- Profiles are paper-first research homes with optional customization through pinned work and collections.
- Paper pages are canonical, indexable public records with stable URLs, version history, citation exports, and clear trust labels.
- Build useful product surfaces first, not a marketing landing page.
- Keep the interface calm, dense enough for repeated research use, responsive, and accessible.
- Prefer explicit labels like `Format checked` over vague trust badges.
- Always retain a chronological Following feed next to any ranked feed.

## Architecture boundaries

- Primary runtime: Cloudflare Worker with Static Assets, TypeScript, React Router framework mode.
- Metadata source of truth: MongoDB Atlas Flex. Atlas Serverless must not be used because it is discontinued.
- PDFs: private R2 quarantine before review; immutable public R2 copies after publication.
- Outbound email: Resend. Inbound aliases: Cloudflare Email Routing.
- Product analytics: minimal PostHog events. Operational signals: Cloudflare native observability.
- Do not add Datadog or Grafana unless an explicit operational gap and cost justification are documented.
- Ranking logic belongs in the separate `kaltex-ranking` repository; do not scatter score formulas across UI code.

## Before implementing a feature

1. Identify whether it touches a settled platform decision in `platform.md`.
2. Check whether it changes a public scholarly record, a trust claim, or a privacy boundary.
3. Prefer a smaller V1 implementation over a new service or workflow.
4. For a significant architecture change, update `docs/architecture.md` and explain the cost/operations impact.
5. For any new external service, document why Cloudflare, MongoDB Atlas, R2, Resend, or PostHog cannot meet the need.

## Data and security rules

- Validate structured submission metadata on the server.
- Enforce authorization server-side, especially reviewer and moderator actions.
- Use idempotency for publication, promotion to public R2, webhook handling, and notification sends.
- Log sensitive administration and moderation actions to an audit trail.
- Escape PDF-extracted text and treat uploaded material as untrusted.
- Keep secrets out of source control and browser bundles.
- Do not write raw emails, tokens, personal identifiers, or paper contents into analytics events or broad logs.

## Documentation obligations

- Update `platform.md` if a product decision changes.
- Update `docs/architecture.md` if infrastructure, cost controls, data flow, or service responsibility changes.
- Record unresolved tradeoffs instead of silently treating them as decided.
- Keep external-service pricing notes dated and linked to primary documentation.
