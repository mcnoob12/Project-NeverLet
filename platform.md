# KalTex Platform Brief

Status: canonical product direction for the first KalTex release

Last updated: 2026-09-02

## One sentence

KalTex is a radically open, paper-first preprint platform for independent researchers: a public research feed where papers are the posts and a researcher profile is a durable research home.

## Why KalTex exists

Independent researchers can make serious contributions without a university, lab affiliation, institutional email address, or publication network. Existing preprint servers make dissemination possible, but their interfaces often feel like archives rather than places where research can be discovered, understood, and followed.

KalTex changes the experience, not the scholarly record. It gives independent researchers the visibility, profile, discovery, basic statistics, and citation infrastructure that institutions normally supply. It is run by Kalaris Labs as a self-funded mission and community initiative, not as a pay-to-publish business.

The working motto is:

> Preprints for independent researchers.

The product metaphor is deliberately social, but the unit of attention is research. LinkedIn is a useful comparison for the shape of profiles; X is a useful comparison for the speed of a feed. KalTex is neither a professional-networking site nor a general social network.

## The promise

KalTex must make it possible for an independent researcher to:

1. Create a credible, public research identity using a personal email address.
2. Link optional scholarly identities such as ORCID, Google Scholar, ResearcherID, or similar profiles.
3. Upload a preprint in a consistent, durable format.
4. Receive clear submission status and lightweight integrity screening.
5. Publish a citable public landing page with a stable KalTex identifier.
6. Be discovered through a research feed, search, topics, profiles, and direct links.
7. See understandable statistics such as views, downloads, saves, and citations.
8. Revise work without erasing the historical record.

Anyone can read and browse KalTex. Institutional researchers are welcome, but the product should never assume an institutional affiliation, an .edu email address, or access to a lab administrator.

## Principles that decide product questions

### 1. Independent by default

No feature may require institutional email, affiliation, grant number, or a supervisor. Affiliation is optional profile context, never a gate.

### 2. Open access is the floor

Published papers, their public metadata, abstracts, authorship, versions, and citation metadata are readable without an account. The public web is the primary audience, not just logged-in members.

### 3. Open does not mean unmoderated

KalTex is radically open in access and opportunity, while still protecting readers from spam, fraud, plagiarism, misleading claims, and malicious uploads. It is a preprint platform, not a peer-review journal and not an unfiltered file dump.

### 4. The paper outranks the profile

The feed, profile, search experience, and ranking system should all lead a reader toward actual work. Never create an attention economy around personal status updates, follower counts, outrage, or engagement bait.

### 5. Claims must match evidence

The platform may say a paper is "format checked," "identity linked," "similarity screened," or "reproducibility metadata provided" when those statements are true. It must never imply that a paper is peer reviewed, scientifically validated, plagiarism-free, AI-free, or reproducible merely because it passed intake checks.

### 6. A citation is not a like

Citations are scholarly relationships. An internal citation exists when a KalTex paper references another KalTex paper and the relationship has been resolved. It is never created by a click, save, or follower action.

### 7. The scholarly record is durable

Published versions are immutable. A newer version creates a new immutable version; it does not overwrite the prior paper. Withdrawn work leaves a public tombstone and a reason category. Paid plans must never silently remove a citable public version.

### 8. Privacy is compatible with useful metrics

Statistics should help authors understand reach without creating surveillance. Aggregate, rate-limited, bot-filtered metrics are enough for V1. Do not expose reader identities, read histories, or a public list of who viewed a paper.

### 9. Low operations is a feature

Prefer managed, serverless, and edge services. Avoid systems that need a permanent server, manual capacity planning, or an on-call operations team. Spend money only where it improves trust, durability, or discovery.

## What KalTex is and is not

| KalTex is | KalTex is not |
| --- | --- |
| An open preprint server | A journal or peer-review authority |
| A public research discovery feed | A general posting network |
| A durable profile for researchers | A credential marketplace |
| A home for independent researchers | An institution-only repository |
| A lightweight, transparent screening process | A claim that content is scientifically correct |
| An internal citation graph with external import later | A metric that can be purchased or gamed |

## Primary users

### Independent researcher

The core user. May work alone, have a personal email address only, lack formal affiliation, and need a credible public research presence.

### Open institutional researcher

May be university or industry affiliated but shares the need for rapid, readable, citable dissemination. Welcomed without shifting the product toward institutional workflows.

### Reader and collaborator

Browses papers, topics, researcher profiles, citations, and versions. Must not need an account to access research.

### Moderator or reviewer

Uses a deliberately narrow internal queue to resolve submissions that cannot be safely published by automated intake alone, investigate flags, and issue transparent outcomes.

## Success metric

The north-star metric is active independent researchers: people without institutional backing who return to publish, revise, maintain a profile, or follow research on KalTex.

Supporting measures:

- Published independent researchers per month.
- Time from valid submission to decision.
- Percentage of papers with clear identity and screening context.
- Search-to-paper and feed-to-paper open rate.
- Save and citation relationships that indicate real research use.
- Cost per active researcher and cost per published paper.

Do not optimize for raw sign-ups, follower counts, empty profiles, or impressions alone.

## Information architecture and experience

KalTex should feel like a calm, usable research network, not an old archive and not a marketing landing page. The first authenticated screen is the research feed. The first public screen should make it easy to browse current work.

### Main navigation

- Home: personalized paper feed.
- Explore: search, disciplines, topics, trending research, newest research, and independent researchers.
- Publish: submission workflow and author dashboard.
- Library: saved papers and followed topics or researchers.
- Profile: public research home and private stats/settings when viewing one's own profile.

### Feed

Every feed item is a paper card, never a text post. A card contains:

- Author identity, profile image, identity context, and publication date.
- Paper title, short abstract preview, disciplines, and topic tags.
- Trust labels that state only completed checks.
- Views, downloads, saves, and citations in a restrained statistics row.
- Actions to open the paper, save it, cite it, follow the author, or report a concern.

Readers can switch between a chronological Following feed and a ranked For You feed. The chronological feed is always available. The ranked feed must give a short, human explanation such as "Because you follow computational biology" or "Cited by work in your library."

### Paper page

The paper page is the canonical public record. It includes:

- Title, authors, abstract, disciplines, tags, submission and publication dates.
- Stable KalTex identifier and canonical URL.
- View/download PDF action.
- Version selector and a clear latest-version marker.
- Suggested citation in BibTeX, RIS, CSL-JSON, and plain text.
- Trust and review labels with plain-language definitions.
- References parsed or entered by the author, with resolved internal citations when possible.
- "Cited by" for internal citations; external citation import is later work.
- Related research based on topics, references, and responsible ranking.
- A report link and visible withdrawal notice if applicable.

The paper page must emit schema.org ScholarlyArticle and scholar-friendly citation metadata, have a canonical URL, and remain readable without JavaScript where practical. Discoverability by search engines is part of publishing, not an afterthought.

### Researcher profile

A profile is a paper-first public research home. It includes a name, optional bio, optional location and affiliation, linked scholarly identities, topics, research interests, selected work, collections, and paper-based statistics.

Authors may organize their work using pinned papers and named collections. They may not rearrange or suppress the canonical version history of an already published paper.

Followers are a private-to-the-author statistic in V1. Do not make profile popularity visually dominate the research.

### No messaging, comments, or general posts in V1

KalTex V1 is LinkedIn-lite and X-like in layout only. Direct messages, public comments, reposts, general status posts, job boards, funding offers, and communities are explicitly out of scope. Researchers connect through public work, citations, follows, and outbound contact links they choose to show.

## Publishing model

### Scope

KalTex accepts preprints in all sciences. The taxonomy may group work by broad discipline and allow multiple disciplines per paper. Interdisciplinary work is normal and must not be forced into one silo.

### The canonical file format

V1 accepts one canonical public format: PDF.

Reasons:

- It is the most universal format for readers, citations, and preservation.
- It avoids expensive and fragile server-side document conversion.
- It creates a consistent reader and download experience.
- The original uploaded PDF can be stored immutably and checksummed.

The platform may later accept source bundles or datasets as supplementary artifacts, but it must not convert arbitrary documents into the authoritative paper during V1. A valid PDF is the published artifact. Source files are not required.

### Required submission metadata

- Title and author list in display order.
- Abstract.
- One or more disciplines and research topics.
- Keywords.
- Corresponding author and verified account.
- References, where available.
- Explicit publication license, with CC BY 4.0 offered as the default recommendation.
- Funding and conflict-of-interest fields when relevant.
- Reproducibility fields: code, data, materials, preregistration, and ethics links or a clear "not available" explanation.

Metadata must be collected as structured fields, not extracted only from a PDF. This makes search, citation, accessibility, and moderation tractable.

### Submission state machine

```text
draft
  -> submitted
  -> intake-checking
  -> needs-author-action | review-queue | publish-approved
  -> published
  -> revised (creates a new version and returns to intake-checking)
  -> flagged (does not automatically unpublish)
  -> withdrawn (public tombstone remains)
```

Every status change is visible to the author with a specific reason. A submission is not public before it reaches `published`.

### Intake and trust process

The objective is to make abuse expensive and good-faith publishing straightforward. It is not to conduct peer review.

Automated intake can check:

- Verified account, email confirmation, rate limits, and Turnstile challenge.
- PDF signature, MIME type, byte size, page count, encrypted/password-protected files, and obvious corruption.
- Required metadata and author identity consistency.
- Exact duplicate file hash and near-duplicate metadata signals.
- Basic spam and fraud heuristics.

Human review is required for:

- A first publication from an unverified account.
- Risky, incomplete, or suspicious submissions.
- Name or authorship disputes.
- Flags that meet a defined investigation threshold.
- Random quality-control samples from otherwise trusted flows.

An ORCID-linked account and a manual identity check are two valid paths. Neither should be privileged as an academic credential; both are mechanisms for reducing impersonation.

### Integrity language

Use labels such as:

- `Email verified`
- `ORCID linked`
- `Identity manually verified`
- `Format checked`
- `Similarity screening completed`
- `Reproducibility metadata provided`
- `Manual review completed`

Do not use labels like `peer reviewed`, `verified research`, `AI free`, `plagiarism free`, or `reproducible` unless KalTex has completed and can defend that exact standard.

Automated AI-content detectors and plagiarism matches are signals for triage, not proof. The reviewer must be able to override them and record an outcome. A reproducibility link is useful context; it is not a reproduction attempt.

### Revisions, withdrawals, and removals

Each version has an immutable PDF, checksum, date, and citation. The paper landing page points to the latest version while retaining prior versions.

Authors can request a withdrawal for serious errors, legal concerns, duplicate publication, authorship dispute, or other defined reasons. KalTex replaces the landing page with a public withdrawal record while keeping the identifier and version history resolvable. Deletion is reserved for narrowly defined legal, privacy, or safety obligations and must be logged internally.

There is no premium product in V1. Future paid profile curation, if ever considered, may change how work is featured on an author's profile, but cannot erase or paywall the public version history of a published preprint.

## Citation and statistics model

### KalTex identifier

On publication, a paper gets a stable KalTex identifier in a documented, non-editable format such as `KTX:2026.00001`. It is not a DOI. The public record uses a stable landing-page URL, for example `https://kaltex.org/p/KTX:2026.00001`.

DOI minting is out of scope for V1. It needs a formal registration arrangement and an operating policy; it must not be implied by the citation UI.

### Citations

V1 creates an internal citation graph. When an author supplies references or references are extracted and confirmed, KalTex attempts to resolve citations to published KalTex records. Each successful relationship becomes an edge from the citing paper/version to the cited paper record.

External citation counts and imports from scholarly indexes are a later integration. They must be identified by source and update time, never merged into a mysterious single number.

### Statistics definitions

- `View`: a bot-filtered, deduplicated landing-page view using a privacy-preserving rolling identifier.
- `Download`: a successful PDF delivery after basic bot filtering.
- `Save`: a signed-in reader adds a paper to their private library.
- `Internal citations`: published KalTex papers that resolve a reference to this paper.
- `External citations`: later, source-specific imported counts.

Metrics are aggregated and delayed enough to avoid real-time manipulation. Raw event logs must have a short retention period. Never display a list of readers or expose email, IP address, or individual activity to an author.

## Discovery and ranking

The feed should help a reader find relevant research without deciding what is true, prestigious, or worthy of attention.

### V1 feed inputs

- Followed researchers and topics.
- Recency with a bounded time-decay.
- Topic, discipline, and keyword relevance.
- Saved-paper and cited-paper relationships.
- Diversity constraints across authors, disciplines, and familiar versus exploratory work.
- Integrity state and anti-spam penalties.

### Explicit exclusions

- No follower-count boost.
- No pay-for-placement.
- No engagement-bait boost.
- No demographic, institution, country, or email-domain prestige boost.
- No opaque score that can silently bury a compliant paper.

The chronological Following feed is non-negotiable. Search has its own clear relevance ordering. Trending lists use a short time window, bot resistance, and diversity caps so a single author cannot dominate.

The ranking algorithm lives in a separate repository, `kaltex-ranking`, with versioned scoring logic, offline test fixtures, documented features, and a changelog. The main application treats it as a dependency or a narrowly scoped service; it does not duplicate ranking logic ad hoc across pages.

## Identity and account design

### Account requirements

- Personal email address is sufficient.
- Email verification is required before publishing.
- A passwordless magic link is the preferred V1 sign-in path; passkeys can follow.
- Google or ORCID sign-in/linking may be offered as optional convenience, never as a requirement.
- ORCID linking uses the researcher-controlled OAuth flow and shows a plainly worded linked badge.
- Manual verification is available for researchers who prefer not to connect ORCID.

### Roles

- `reader`: can browse without an account; account holders can save and follow.
- `researcher`: verified account that can create a profile and submit work.
- `reviewer`: can assess assigned submissions and flags.
- `moderator`: can apply policy outcomes and withdrawals.
- `admin`: limited platform configuration role.

Roles must be explicit in authorization checks. Moderator access is not a hidden client-side flag.

## Email recommendation

Use Resend for all outbound transactional email, sent from a verified KalTex domain. It fits the serverless Cloudflare stack, has a direct Workers integration, supports webhooks, and starts with a 3,000-email/month free tier and a 100-email/day cap. Email is used for account verification, magic links, submission receipts, decision notices, moderation status, and security alerts. It is not used for bulk engagement mail in V1.

Use Cloudflare Email Routing for inbound aliases such as `support@`, `submissions@`, and `abuse@`, forwarding into the small Kalaris Labs support inbox. Do not approve papers by replying to an email; emails should link to a signed-in review action so decisions are auditable.

Email implementation requirements:

- Verify the sending domain and configure SPF, DKIM, and DMARC before launch.
- Use a dedicated transactional subdomain such as `mail.kaltex.org`.
- Store no email tokens in plaintext; store hashed, single-use, short-lived tokens with a MongoDB TTL index.
- Process Resend delivery, bounce, complaint, and suppression webhooks idempotently.
- Rate-limit magic-link requests and do not reveal whether an email address has an account.
- Keep email templates accessible, plain, and concise.

Postmark is a reasonable later alternative if deliverability support becomes the overriding concern, but it is less cost-friendly at the start. Do not use a large general-purpose marketing platform for account and publishing emails.

## V1 scope

Build:

- Public paper pages, profiles, feeds, exploration, search, and topic browsing.
- Account creation with personal email, magic link, and optional ORCID linking.
- Researcher profiles with bio, external identifiers, pinned work, and collections.
- PDF-only publishing flow, review states, immutable versions, withdrawals, and reporting.
- Manual-review dashboard for a small platform team.
- Internal citation graph and citation exports.
- Privacy-conscious author stats.
- Following, saves, and clear ranking modes.
- Structured metadata and search-engine discoverability.

Do not build in V1:

- Peer review, journal workflows, or reviewer assignment by expertise.
- Direct messages, comments, reposts, general social posts, or chat.
- DOI registration.
- Paid publishing, paid ranking, subscriptions, or a premium plan.
- Native mobile apps.
- Heavy automated scientific validation, a promise of AI detection, or a claim of reproducibility testing.
- A public general-purpose API or bulk data export program.
- Arbitrary source-document conversion, datasets, and rich supplemental artifacts.

## Decision log and open questions

These are settled for V1:

- Name: KalTex, not KalXiv. KalTex is more distinctive and avoids an unnecessarily derivative association with arXiv.
- Audience: open to all researchers; independent researchers receive primary design attention.
- Discipline scope: all sciences.
- Funding: self-funded by Kalaris Labs as a nonprofit-style mission initiative.
- Experience: paper-first feed and profiles; not messaging or comments.
- File format: PDF only.
- Citations: internal graph now, external imports later.
- Versioning: visible, immutable versions and transparent withdrawals.
- Trust: light intake plus risk-based human review; no peer-review claim.
- Ranking: separate repository, chronological alternative always available.

These require a policy decision before public launch:

- Maximum PDF size and page count.
- Exact withdrawal and deletion policy, including legal escalation contact.
- Default license language and available license list.
- Moderation response targets and escalation owner at Kalaris Labs.
- Whether manual identity verification uses a document check, a human interview, or another privacy-preserving method.
- Which external citation sources will be imported, under what terms, and how often.
- Published privacy policy, terms of use, content policy, and accessibility standard.

## Non-negotiable guardrails for future work

1. Do not require an academic institution to register or publish.
2. Do not call intake checks peer review.
3. Do not make published research disappear for commercial convenience.
4. Do not turn citations, identity, or visibility into paid advantages.
5. Do not add a server that somebody must patch, provision, or keep alive merely for the core product to work.
6. Do not introduce a feature that increases administrative burden without a measurable benefit to researcher trust or discovery.
7. Do not collect personal or behavioral data simply because it could help ranking.
8. Keep the public record, profile, and canonical paper URL stable through visual redesigns.

For implementation architecture, operating budget, data model, and service boundaries, read `docs/architecture.md`. For how agents should work in this repository, read `AGENTS.md`.
