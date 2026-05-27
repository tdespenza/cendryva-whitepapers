# Cendryva Advisory Marketplace: Expert Consultation Against Live Operational Data

**Audience:** Operations leaders, COOs, CFOs, founders, and procurement teams evaluating expert-as-a-service platforms; product and engineering leaders designing internal advisor programs.
**Canonical URL:** `/whitepapers/advisory-marketplace-operational-intelligence/`
**Related papers:** Self-hosted ML observability; the 12-Condition Framework; customer-support contact-center AI operations playbook.
**Author:** Cendryva
**Published:** 2026-05-25
**Version:** 1.0
**License:** CC BY 4.0
**Contact:** research@cendryva.com

## Abstract

Most expert consultation marketplaces are content directories. An operator finds a consultant, schedules a video call, exchanges slides, and walks away with notes. The advisor never sees the operator's actual KPIs, the operator never gets a session recording linked to a metric, and the next advisor — three weeks later — starts from zero context. The marketplace owns the *meeting*, not the outcome.

Cendryva Advisory is built on a different premise: a consultation should be a unit of operational change. The advisor is connected to the same management-by-statistics platform the operator runs the business on. The session is recorded, summarized, and persisted alongside the KPIs that triggered the booking. The action items become trackable assignments. The next session has the previous one as context. The marketplace owns the *learning loop*.

This paper describes the operational gap that conventional advisory platforms cannot close, the architectural primitives that connect an expert marketplace to live operational data, and the trust controls — verification, audit, refund policy, payout transparency — that an advisory marketplace needs to be safely operated inside regulated environments and inside the buyer's organization.

## Executive Summary

Expert advisory is a $300B+ global market spread across management consulting, fractional executives, coaching platforms, and one-off expert networks. Adoption inside operationally mature companies is uneven for one reason: consultation outcomes are not measurable. Buyers cannot tell whether a session moved a metric. Sellers cannot prove which sessions produced value. Platforms compete on advisor supply rather than outcome quality.

Cendryva takes a different shape. Because Cendryva is already the operating substrate for KPIs, scoreboards, formulas, accountability, and operational workflows (Pillar 1 — Cendryva Stats), an advisory consultation booked inside Cendryva inherits all of that context. The advisor receives the KPIs attached to the booking, the operator receives a structured AI summary anchored in those KPIs, and the action items emitted from the session become assignments inside the same operations platform. Outcomes are observable across subsequent reporting periods because the session is linked to the metric.

For operations leaders, this turns advisory from a discretionary expense into an instrumented input. For advisors, it turns a marketplace listing into a portfolio of measurable engagements. For platform operators, it produces a defensible data substrate — the marketplace owns not just the supply graph but the outcome graph.

## The Operational Gap

Traditional advisory marketplaces split into three patterns, each with a structural ceiling.

**Expert networks.** GLG, AlphaSights, Third Bridge. High-trust, high-cost, focused on episodic information transfer to investors and strategy teams. Optimized for one-off knowledge access, not iterative operational change. The buyer pays for the advisor's time, not for an outcome the platform can verify.

**Coaching and content platforms.** MasterClass, Reforge, Maven. Optimized for asynchronous education at scale. When live consultation happens, the advisor sees no operational data; the platform stores no operational outcome.

**Calendar-and-call marketplaces.** Clarity, Intro, Superpeer. Lightweight Stripe-plus-Zoom layers around expert directories. Zero operational integration; the platform has no way to differentiate "good session" from "good vibes."

Each pattern wins on a specific axis. None of them produces an outcome substrate. The advisor walks into the meeting blind to the operator's actual statistics. The operator walks out with notes that decay before the next operating review. The platform cannot answer the only question that matters at procurement time: *did booking advisors on this platform produce measurable operational improvement?*

The gap is not advisor quality. It is the absence of an operational substrate underneath the marketplace.

## Why an Operational Substrate Changes the Product

Connecting an advisory consultation to live operational data changes four properties of the session.

**Context becomes free.** When the operator books a session, the booking is attached to specific KPIs and metric IDs from the operator's own scoreboard. The advisor opens the booking and sees the actual numbers — not a deck the operator prepared at 11pm the night before. Preparation time drops, time-in-substance rises, and the conversation starts at signal rather than at setup.

**Outcomes become instrumented.** Action items emitted from the session become assignments inside the same operations platform. Their status — open, in progress, completed — is observable. The KPIs attached to the booking continue to be tracked. When the metric moves, the platform can attribute the move to the consultation with the same fidelity it attributes any other intervention. Advisor reputation stops being a star rating and becomes a track record.

**Trust stops requiring brand.** Buyers no longer have to rely on "ex-McKinsey" as a quality signal. They can look at the advisor's prior engagements on the platform, the metrics those engagements were attached to, the action items completed, and the subsequent KPI trends. The marketplace produces evidence; the buyer makes a procurement-grade decision.

**Continuity replaces churn.** A second booking with the same advisor inherits the first session's summary, action items, and KPI deltas. The platform amortizes the trust-building cost across the entire customer lifetime instead of paying it again every time an advisor changes.

These four properties only emerge when the marketplace lives inside the operations platform, not next to it.

## Cendryva's Approach

Cendryva Advisory is the second of seven product pillars. Pillar 1 (Cendryva Stats) ships the management-by-statistics substrate — formulas, scoreboards, accountability assignments, the 12-Condition Framework, and operational dashboards. Pillar 2 layers an expert marketplace and scheduled video consultation on top of that substrate so the advisor consultations happen with the operator's real numbers in the room.

### Product principles

1. **Sessions are operational units, not content units.** Every booking has a topic, an agenda, attached KPI IDs, and attached metric IDs. The session payload travels with the operational data, not with a generic calendar invite.
2. **The advisor is verified before the buyer pays.** Profiles begin in `pending_verification` and only an admin transition moves them to `active`. Verification evidence is persisted as a JSON blob alongside the row so the customer can wire whichever KYC/AML provider their compliance program requires.
3. **Payments are atomic with bookings.** A booking row is created in `requested` status, a Stripe payment intent is issued against it, and the intent id is stored on the row. There is no booking without a payment intent and no payment intent without a booking.
4. **Conflicts are checked at the data layer.** The Rust data service exposes a `find_booking_conflicts` query that returns active (requested or confirmed) bookings for the advisor whose windows overlap the proposed slot. The booking service blocks the request before it issues a payment intent. Double-bookings are prevented by construction, not by hope.
5. **AI summarization is post-call, not in-call.** A confirmed booking produces a video session via a `VideoProvider` abstraction; recording and transcript URIs are persisted on the booking row when the vendor finishes processing; the `SessionSummaryService` then runs a `SummaryGenerator` (heuristic by default; pluggable to the operator's preferred LLM) to produce a summary, action items, and follow-up topics, all linked back to the booking and discoverable from the KPI scoreboard.
6. **Refund policy is a single configurable cutoff.** The default is full refund for client cancellations made at least 24 hours before the scheduled start; advisor cancellations always refund. The cutoff is the constant `CANCELLATION_REFUND_CUTOFF_MS` so platform operators can shift the policy with a single change reviewed under normal code-change controls.
7. **Every lifecycle event lands in WORM audit.** Booking requests, confirmations, cancellations, and completions emit signed audit events through `ImmutableAuditLogService`. The marketplace inherits the same tamper-evident chain that protects Cendryva's ML decision logs and compliance events.
8. **Vendor surfaces are pluggable, not built-in.** The booking service does not import the Stripe SDK or the Daily/Zoom/Twilio SDKs directly. It depends on three injectable interfaces — `PaymentIntentProvider`, `VideoProvider`, `PayoutProvider`. Cendryva ships logging stubs that exercise the API surface end-to-end without external calls; production deployments wire the real vendor under environment variables documented in the marketplace overview.

## Reference Architecture

```
Buyer (operator)               Advisor                  Platform admin
        │                          │                           │
        ▼                          ▼                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       cendryva-api (TypeScript)                      │
│                                                                      │
│   /api/v1/advisory-marketplace/advisors          (register, search)  │
│   /api/v1/advisory-marketplace/advisors/:id      (profile + rating)  │
│   /api/v1/advisory-marketplace/advisors/:id/availability             │
│   /api/v1/advisory-marketplace/bookings          (book, list, get)   │
│   /api/v1/advisory-marketplace/bookings/:id/confirm                  │
│   /api/v1/advisory-marketplace/bookings/:id/cancel                   │
│   /api/v1/advisory-marketplace/bookings/:id/complete                 │
│   /api/v1/advisory-marketplace/bookings/:id/summary                  │
│   /api/v1/advisory-marketplace/bookings/:id/review                   │
│                                │                                     │
│              ┌─────────────────┼──────────────────┐                  │
│              ▼                 ▼                  ▼                  │
│   AdvisorProfileService  BookingService    SessionSummaryService     │
│   AdvisorSearchService                                               │
│                                │                                     │
│              ┌─────────────────┼──────────────────┐                  │
│              ▼                 ▼                  ▼                  │
│        VideoProvider     PaymentIntentProvider  PayoutProvider       │
│        (Daily / Zoom /   (Stripe)               (Stripe Connect)     │
│         Twilio)                                                      │
│                                │                                     │
│                                ▼ dataClient (HTTP, circuit-broken)   │
└──────────────────────────────────────────────────────────────────────┘
                                 │
┌──────────────────────────────────────────────────────────────────────┐
│                     cendryva-data (Rust, port 4001)                  │
│   advisor_profiles, advisor_availability_windows,                    │
│   advisory_bookings (incl. /conflicts query),                        │
│   advisory_session_summaries, advisor_reviews                        │
│                                 │                                    │
│                                 ▼                                    │
│                              Postgres                                │
└──────────────────────────────────────────────────────────────────────┘

                       Live operational substrate
   ┌──────────────────────────────────────────────────────────────────┐
   │ KPI scoreboards · metric history · 12-Condition assignments ·    │
   │ WORM audit log · BAA program · entitlement gating · RBAC         │
   └──────────────────────────────────────────────────────────────────┘
```

Each booking row references `attached_kpi_ids` and `attached_metric_ids`. The advisor's session UI reads those identifiers and pulls live KPI snapshots from the same data service that powers the operator's everyday scoreboards. Post-session, the AI summary captures the action items as structured JSON so the action-item objects can become workflow assignments via the existing `WorkflowActionEntitiesService`.

## Core Capabilities

### Advisor onboarding and verification

Registration is open to any authenticated user; the row is created in `pending_verification`. The `AdvisorProfileService` enforces a finite-state machine for transitions: only admin actors may move to `active` and only after the customer's verification workflow (Persona, Onfido, Stripe Identity, or a manual review) populates `verification_evidence`. The terminal state is `retired`; suspended advisors can return to `active` through admin review, but retired profiles cannot. The state machine prevents accidental status flips and the audit log records every transition.

### Discovery and ranking

`AdvisorSearchService` returns active advisors enriched with rating and next-available slot. Ranking is deterministic: average rating descending, review count descending, creation date ascending. New advisors with zero reviews still appear at a predictable position in the list, which protects supply-side onboarding. Filters cover category (operations, finance, marketing, sales, leadership, technology, legal strategy, startup scaling, HR/recruiting, productivity), maximum hourly rate, and language. Discovery is auth-only but is not feature-gated — FREE-plan operators may browse the directory; the booking endpoint is gated behind `ADVISORY_MARKETPLACE_ACCESS`, which is granted on STARTER, PROFESSIONAL, and ENTERPRISE plans.

### Booking, conflict prevention, and payment

A booking request runs four checks before money moves: (1) the advisor exists and is `active`; (2) the proposed `scheduled_start` is strictly before `scheduled_end` and `duration_minutes` is positive; (3) no existing requested or confirmed booking for the advisor overlaps the window; (4) the operator's tenant carries the `ADVISORY_MARKETPLACE_ACCESS` entitlement. If any check fails, the row is never created. If all pass, the booking is persisted with status `requested`, the `PaymentIntentProvider` is invoked, the returned intent id is patched onto the row, and a `advisory.booking.requested` audit event is emitted. The advisor's confirmation moves the row to `confirmed` and provisions a video session through the `VideoProvider`; cancellation triggers a refund decision per the published policy and tears down the video session.

### Live operational context in the session

Each booking carries `attached_kpi_ids` and `attached_metric_ids` JSON arrays. The frontend (out of scope for v1; the data substrate is ready) renders the advisor's view of the session as the booking topic + agenda + a side panel of the current KPI values and the trailing-period history for each attached metric. The advisor begins the call already inside the operator's scoreboard. The operator does not have to assemble a "context deck" for the advisor.

### Post-session AI summary

When the advisor marks the booking `completed`, the `SessionSummaryService` invokes the configured `SummaryGenerator` and persists the resulting summary, action items, and follow-up topics in `advisory_session_summaries`. The summary id is patched back onto the booking row. The default `HeuristicSummaryGenerator` produces a deterministic stub; production deployments inject a generator backed by `AssistantLlmService` or another LLM. Recording and transcript URIs are stored on the summary row when the video vendor's webhook reports availability.

### Reviews and rating aggregation

Clients may post a 1-5 review on completed bookings. Reviews are unique per booking. The Rust data service exposes a `rating-summary` query that returns `(advisor_id, review_count, average_rating)` in a single round trip, used by the search service for ranking. Comments are free text and surfaced on the advisor profile.

### Advisor payouts via Stripe Connect

The `PayoutProvider` interface is the seam for Stripe Connect (or any equivalent vendor). The default `LoggingPayoutProvider` records intended transfers without contacting Stripe; the production `StripeConnectPayoutProviderStub` is intentionally left throwing so deployments cannot accidentally move money until the customer implements `stripe.transfers.create` and configures `CENDRYVA_PAYOUT_PROVIDER=stripe_connect`. The platform take rate is a basis-point constant — `DEFAULT_PLATFORM_FEE_BPS = 1500` (15%) — easy to surface during procurement negotiations.

### Audit and compliance

Every booking lifecycle event lands in the same Vault-signed WORM audit chain that protects Cendryva's ML decision logs (see *Self-Hosted ML Observability* paper). For customers operating under HIPAA, advisory consultations that touch PHI inherit the same `business_associate_agreements` and `phi_disclosure` machinery used elsewhere in the platform; the marketplace does not introduce a parallel compliance surface.

## Differentiation

| Capability | Generic advisory marketplace | Cendryva Advisory |
|-----------|------------------------------|--------------------|
| Session context | Operator-prepared deck | Live KPIs + metric history from the operator's own scoreboard |
| Action items | Free-form notes | Structured JSON, optionally promoted to workflow assignments |
| Outcome attribution | Star rating only | KPI deltas observable in subsequent operating periods |
| Verification | Marketplace-side reputation | Pluggable KYC/AML before status `active` |
| Audit trail | Calendar + Zoom logs | Signed WORM chain shared with ML and compliance events |
| Refund policy | Buried in T&Cs | Single code constant; admin-reviewable |
| Payout transparency | Black-box take rate | Explicit basis-point constant; per-booking fee/net breakdown |
| Tenancy | None | Org-scoped bookings, RLS-friendly schema |
| Vendor lock-in | Built on Zoom + Stripe Connect | Pluggable providers under environment-variable contract |

## Use Cases

**SaaS revenue ops review.** A SaaS COO books a 60-minute session attached to `kpi.weekly_pipeline_velocity` and `kpi.deals_lost_no_decision`. The advisor — a former VP of Sales — opens the booking and sees the last 12 weeks of both metrics, the 12-Condition tints for the most recent week, and the linked SOPs. The action items emitted by the AI summary become assignments to the pipeline owner. Three weeks later, the COO books a follow-up; the first summary appears in the context panel automatically.

**Multi-location service operations.** A regional service firm uses Cendryva Stats to roll up daily KPIs from 18 branches. Branch managers may book a 30-minute advisory session against their own scoreboard whenever a specific stat enters DANGER for two consecutive weeks. The platform's `ADVISORY_MARKETPLACE_ACCESS` entitlement plus per-branch booking quota (out of scope for v1; the entitlement substrate is ready) keeps usage governed.

**Investor-mandated improvement plan.** A growth-stage company's board mandates that operating metrics in the bottom quartile receive an external advisor review each month. Cendryva Advisory becomes the receipts platform: every mandated session is bookable from the dashboard, the booking is attached to the underperforming KPI, the AI summary becomes the board-pack attachment, and the WORM audit log is the evidence trail for the investor.

**Internal fractional executive program.** A holding company runs an internal fractional-CXO program across portfolio companies. The portfolio executives are the "advisors" inside Cendryva; the operating company CEOs are the "clients." The marketplace stays inside the holdco's tenant boundary; KYC is satisfied by employment records rather than third-party vendors; payouts run through internal cost-allocation rather than Stripe Connect.

## Implementation Status

This section maps the architectural claims above to the Cendryva codebase as of the publication date in the header.

**Postgres schema** — SHIPPED

- Migration: `migrations/postgres/V010__Billing_And_Marketplace.sql`. Rollback in the matching `*.rollback.sql`.
- Tables: `advisor_profiles`, `advisor_availability_windows`, `advisory_bookings`, `advisory_session_summaries`, `advisor_reviews`.
- Indexes: status, category (GIN), advisor + day (availability), org + status + start (bookings), advisor + start (bookings), advisor + created (reviews); unique on (booking_id) for summaries and reviews; per-booking-uuid CHECK constraints for time ordering and positive duration/price.

**Rust dumb-storage layer** — SHIPPED

- Code: `src/data/advisory_marketplace.rs` (wired into `src/data/mod.rs`).
- Endpoints: advisor profile CRUD; availability window CRUD; booking CRUD plus `/advisory-bookings/conflicts`; summary upsert and read; review create/list/rating-summary.

**TypeScript dataClient** — SHIPPED

- Types: `apps/api/src/shared/cendryva-data/data.types.ts` (AdvisorProfileRow, BookingRow, SessionSummaryRow, AdvisorReviewRow, AvailabilityWindowRow, AdvisorRatingSummary, plus all request DTOs and enums).
- Methods: corresponding `dataClient.createAdvisorProfile`, `dataClient.findBookingConflicts`, `dataClient.getAdvisorRatingSummary`, etc.

**Domain services** — SHIPPED

- Code: `apps/api/src/services/domains/advisory/`:
  - `advisor-profile.service.ts` (finite-state machine, terminal `retired` state).
  - `advisor-search.service.ts` (rating-weighted ranking + heuristic next-available slot).
  - `booking.service.ts` (orchestration, conflict check, refund policy, audit emission).
  - `session-summary.service.ts` (pluggable `SummaryGenerator`).
  - `video-session.service.ts` (`VideoProvider` interface + `LoggingVideoProvider` + vendor stubs).
  - `payout.service.ts` (`PayoutProvider` interface + `LoggingPayoutProvider` + Stripe Connect stub).

**HTTP route** — SHIPPED

- Code: `apps/api/src/routes/advisory-marketplace.ts` mounted at `/api/v1/advisory-marketplace` in `apps/api/src/routes/openapi-mounts.ts`. Registered in the access-control route manifest.

**Entitlement** — SHIPPED

- Feature flag `ADVISORY_MARKETPLACE_ACCESS` granted on STARTER, PROFESSIONAL, ENTERPRISE in `apps/api/src/services/domains/billing/stripe.service.ts`. Wired into the booking endpoint via `requireFeature`.

**Tests** — SHIPPED

- 60+ unit tests in `apps/api/src/services/domains/advisory/*.test.ts` covering the advisor FSM, search ranking, slot computation, booking conflict + entitlement + refund eligibility, video provider env selection, payout fee math.
- E2E happy-path test at `apps/e2e/tests/api/advisory/marketplace-booking.e2e.test.ts` (skipped automatically when the API is not running).

**Documentation** — SHIPPED

- Overview + architecture at `docs/advisory/MARKETPLACE-OVERVIEW.md`.

**Customer-wiring required for production (NOT SHIPPED — by design)**

- Real video provider implementations behind `DailyVideoProviderStub` / `ZoomVideoProviderStub` / `TwilioVideoProviderStub`.
- Real Stripe `PaymentIntentProvider` (logging stub ships).
- Real Stripe Connect `PayoutProvider` (logging + throwing stub ship).
- Real KYC/AML verification provider integration before promoting advisors to `active`.
- LLM-backed `SummaryGenerator` (heuristic ships).
- Recording + transcript webhook handlers that patch `recording_uri` / `transcript_uri` onto the summary row.
- Workflow-assignment promotion of AI-emitted action items (the workflow substrate exists; the wire-up is product work).

## Scope and Limitations

This is a vendor-authored whitepaper from Cendryva. It describes architectural principles and the v1 implementation status for an expert advisory marketplace integrated with an operational intelligence platform. It is not an independent product review, a certification statement, or a procurement endorsement.

**In scope.** The product principles, architecture, data model, lifecycle FSMs, integration interfaces, and v1 implementation status for the Cendryva Advisory marketplace. The conceptual benefits of connecting an advisory marketplace to live KPI substrate.

**Out of scope.** Frontend UX, advisor compensation theory, marketplace pricing strategy, marketing positioning, individual vendor selection (Daily vs Zoom, Persona vs Onfido), tax treatment of international payouts, and content moderation for advisor profiles. This paper does not constitute compliance guidance for any specific regulation. Customers operating with PHI, financial data, or other regulated content should consult counsel before exposing advisors to that data through the platform.

**Not legal or financial advice.** The default refund policy and the 15% platform take rate are illustrative defaults; the right values for any deployment depend on jurisdiction, advisor mix, and the operator's procurement contracts.

**Empirical claims.** Architecture and capabilities described above ship in the codebase as of the publication date and are traceable through the Implementation Status section. Future versions may revise the lifecycle, refund policy, or interface contracts; in-flight customers should pin to a release tag.

**Time-bounded content.** Video conferencing standards, payment processor capabilities, and AI summary generation techniques evolve quickly. Readers should validate the assumptions above against the most recent vendor documentation before designing a production deployment.

## References and Further Reading

### Marketplace design and trust

- Hagiu, Andrei, and Wright, Julian. *Multi-sided platforms*. International Journal of Industrial Organization, 2015. https://www.sciencedirect.com/science/article/abs/pii/S0167718714001167
- Tirole, Jean. *Market Failures and Public Policy* (Nobel lecture). 2014. https://www.nobelprize.org/uploads/2018/06/tirole-lecture.pdf
- Cusumano, Michael, et al. *The Business of Platforms*. HarperBusiness, 2019.

### Operational intelligence and performance management

- Hubbard, L. Ron. *Management By Statistics*. Reference text for the management-by-statistics tradition that Cendryva Stats formalizes.
- Kaplan, Robert S., and Norton, David P. *The Balanced Scorecard*. Harvard Business Review Press, 1996.
- Drucker, Peter F. *The Practice of Management*. Harper & Brothers, 1954.

### Adjacent regulatory and audit frameworks

- NIST. *AI Risk Management Framework (AI RMF 1.0)*. 2023. https://www.nist.gov/itl/ai-risk-management-framework
- AICPA. *Trust Services Criteria (SOC 2)*. https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- HIPAA. *Business Associate Agreement requirements (§164.308(b), §164.314(a))*. https://www.hhs.gov/hipaa/for-professionals/covered-entities/sample-business-associate-agreement-provisions/index.html

### Payments and marketplace infrastructure

- Stripe. *Connect documentation*. https://stripe.com/docs/connect
- Stripe. *PaymentIntents API*. https://stripe.com/docs/api/payment_intents
- Persona / Onfido / Stripe Identity. KYC provider documentation (used for advisor verification flows).

### Video conferencing and recording

- Daily.co. *Prebuilt video API documentation*. https://docs.daily.co/
- Zoom. *Meetings REST API*. https://developers.zoom.us/docs/api/
- Twilio. *Programmable Video documentation*. https://www.twilio.com/docs/video
