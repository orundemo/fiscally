# Epic: fiscally

**A white-label B2B2C financial-wellness platform where tenant isolation is the
reviewed deliverable, not an implementation detail.** Consumers onboard through
an e-signature and identity-verification flow, get coached by an LLM that sits
behind a supervisor, and are billed on a mixed model — consumer subscriptions,
per-seat operator plans, metered add-ons, and charge-on-success. Two tenants run
on one deployment and look nothing alike.

Everything below the product line — users, organizations, RBAC, audit,
entitlements, metering, billing, notifications, webhooks, console shell, CI/CD,
IaC, migrations — is already live per environment. This epic adds two bounded
contexts and the commercial wiring that turns them into a product.

## Status

| Field | Value |
|-------|-------|
| Status | **Draft — not started** |
| Cluster | **FS** (FS0–FS10) — the product program for fiscally |
| Owner(s) | `apps/onboarding-worker` (new), `apps/coach-worker` (new), `apps/api-edge`, `apps/config-worker`, `packages/{contracts,policy-engine,db,sdk,cli}`, `apps/web-console-next` |
| Target branch | `epic/fiscally` → `main` |
| Builds on | identity/membership, `policy-worker` + `packages/policy-engine`, `@saas/db` + Hyperdrive, `billing-worker`, `metering-worker`, `events-worker`, `notifications-worker`, `config-worker`, the console foundation |
| Origin | Upwork job **"Multi-Tenant Fintech SaaS"** — white-label build delivered under a partner agency's branding — see [`market-context.md`](./market-context.md) |
| Hard constraint | **MVP acceptance contractually due week 5** |
| Decisions locked | See below |

### Decisions locked

1. **Two new bounded contexts, two workers** — `apps/onboarding-worker` and
   `apps/coach-worker`. This is a deliberate contrast with the signalpilot
   epic's single worker: the rule is *one worker per bounded context*, not one
   per product. These two genuinely are two — different failure modes (a stalled
   IDV provider must not block coaching; a slow model must not stall onboarding),
   different data sensitivity, different retention rules.
2. **Four user types are role bindings on the existing membership model** —
   `consumer`, `operator`, `operator_admin`, `platform_admin`. Not four systems,
   not four schemas.
3. **Tenant isolation is the reviewed artifact, so it ships with its own proof.**
   They said isolation "will be reviewed". FS1 — before any feature work — lands
   an automated cross-tenant denial suite in CI and a console page that
   demonstrates a denied cross-tenant read landing in the audit trail. Hand them
   the review.
4. **A second grain below the tenant: `subject_user_id`.** A consumer sees only
   their own records; an operator sees only assigned consumers. Org-level
   scoping alone is insufficient here, and every `coach` and `onboarding` query
   carries both predicates.
5. **Never-persist-raw is enforced by schema, not by discipline.** The
   credit-data payload is consumed in-request; only bucketed derived features
   and a `source_digest` are written. The `coach` tables have **no column that
   can hold a raw payload** — it cannot be stored even by mistake — and an audit
   event records what was dropped.
6. **The coach sits behind a supervisor/eval harness.** Grounded to the derived
   feature set, bounded to a reviewed template space, no personalised financial
   advice claims. Verdict stored (`pass`/`revised`/`blocked`); blocked
   generations are not billed. Same guardrail pattern as the prospecting epic —
   deliberately, because it is proven.
7. **Charge-on-success is a saga, not a webhook and a hope.** The success
   condition is recorded first, the charge is attempted against that record with
   an idempotency key, and a compensating path handles a failed charge. No
   charge without a durable success row; no double charge on retry.
8. **White-label theming is configuration, not forks.** Per-tenant branding
   resolves from `config-worker` at request time. One deployment, two visibly
   different tenants, zero build-time variants.
9. **Onboarding is resumable by construction** — a server-owned multi-step state
   machine, so a user who abandons at IDV returns to the right step rather than
   the beginning.
10. **E-signature and IDV are provider-neutral adapters** behind one interface
    each. v1 ships a deterministic sandbox adapter plus one real provider per
    capability.

## Thesis

The brief is unusually complete: locked scope, a per-view component map, an
interactive UI demo of 30+ views, flow diagrams, and a written source of truth.
This is not a client discovering what they want — it is an agency subcontracting
a defined build with a contractual week-5 acceptance date.

Read structurally, roughly 70% is platform:

| Brief requirement | Where it lives |
|---|---|
| Multi-tenant Postgres with tenant isolation | the tenancy invariant + `membership` — shipped |
| RBAC across four user types | `policy-worker` + `packages/policy-engine` — shipped |
| Consumer subscriptions | `billing-worker` — shipped, live |
| Per-seat operator plans | `billing-worker` + a seat ledger — mostly shipped |
| Metered add-ons | `metering-worker` — shipped |
| Audit trail | `events-worker` — shipped |
| Transactional email | `notifications-worker` — shipped |
| Console shell, org switching, settings, billing UI | `web-console-next` — shipped |
| **E-signature + IDV onboarding** | **this epic** — `onboarding` |
| **LLM coach with a supervisor layer** | **this epic** — `coach` |
| **Ephemeral credit-data analysis** | **this epic** — the never-store-raw pipeline |
| **Charge-on-success** | **this epic** — the saga |
| **White-label per-tenant branding** | **this epic** — config-resolved theming |

Three signals in the post decide how this epic is shaped:

- ***"We'll ask for live URLs, not videos."*** They are running exactly the
  evaluation our whole strategy is built for. The demo is not supporting
  material; it is the submission.
- ***"Tenant isolation is architecturally critical and will be reviewed"*** and
  ***"'never store X' rules are hard constraints."*** They have been burned by a
  vibe-coded submission and are now screening for architectural discipline. So
  FS1 is the isolation proof, ahead of every feature, and decision 5 makes the
  storage rule structural rather than a code-review promise.
- **Interviewing: 0, last viewed two weeks ago.** The post is stale, which is
  good news: a strong late application lands in an empty room rather than a
  queue of forty.

The strategic value is not the $4,000. It is that an agency with a client
pipeline, who has failed to fill this role for two weeks, becomes a **recurring
channel**. Win this and we are their platform bench.

## Read order

1. `README.md` (this file) — charter and locked decisions.
2. `design.md` — the two contexts, the isolation proof, data model, the
   ephemeral pipeline, the coach harness, the billing saga, theming, non-goals.
3. `implementation-plan.md` — FS0–FS10 with acceptance criteria and the week-5
   mapping.
4. `market-context.md` — the originating job and the channel play.
5. `risks-and-open-questions.md`.
6. `IMPLEMENTATION-STATUS.md` — as-built record (empty until FS0 lands).

## Milestones at a glance

| ID | Milestone | Status |
|----|-----------|--------|
| FS0 | Contract + data model: `@saas/contracts/{onboarding,coach}`, migrations `200`/`210`/`220`, manifest entries, policy actions, two worker skeletons | Draft |
| FS1 | **Isolation proof**: cross-tenant denial suite in CI, the `subject_user_id` grain, the audit-visible demo page | Draft |
| FS2 | Onboarding: resumable multi-step state machine, step records | Draft |
| FS3 | E-sign + IDV adapters: provider-neutral, sandbox + one real each, no raw payloads | Draft |
| FS4 | Ephemeral analysis: in-request derivation, drop-proof schema, an audit record of what was *not* stored | Draft |
| FS5 | Coach: LLM behind the supervisor harness, action plans, metered | Draft |
| FS6 | Commercial: consumer subs, per-seat operator plans, metered add-ons, the charge-on-success saga | Draft |
| FS7 | White-label: config-resolved per-tenant branding, two visibly different tenants | Draft |
| FS8 | Edge + SDK + CLI | Draft |
| FS9 | Console: four role surfaces against their design system | Draft |
| FS10 | Evidence + demo tenants: four logins, two brands, the isolation proof page | Draft |

## Scope boundary

| In scope | Out of scope |
|----------|--------------|
| Two product bounded contexts (`onboarding`, `coach`); resumable onboarding with provider-neutral e-sign and IDV; ephemeral credit-data analysis with structurally-enforced non-persistence; guardrailed LLM coaching with stored verdicts and action plans; consumer subscriptions + per-seat operator plans + metered add-ons + charge-on-success; config-resolved white-label theming; four role surfaces; the cross-tenant isolation proof; a seeded two-brand demo | Lending, credit decisioning, or any regulated advice output; storing raw credit-bureau payloads under any circumstance; document/KYC image retention; direct bureau contracts (the widget embed is the boundary); mobile apps; multi-currency; the partner agency's own client-facing branding beyond what theming covers; migrating their existing skeleton codebase |

## Relationship to the platform

- **`billing-worker` / `metering-worker`** — this epic exercises the widest
  commercial surface of any product so far: three plan shapes at once plus a
  usage-conditional charge. It uses the existing primitives; the saga is new
  application logic, not a billing rewrite.
- **`config-worker`** — becomes the theming source of truth. No schema change;
  branding is settings.
- **`events-worker`** — receives `onboarding.*` and `coach.*` events, including
  the "what was not stored" record that makes decision 5 auditable.
- **`policy-worker`** — receives the new actions and the `subject_user_id`
  predicate. This is where the second grain is enforced.
- **`api-edge`** — two new facades, stricter rate-limit classes on the model and
  IDV routes.

## Verification bar

The supervisor harness, the onboarding state machine, and the feature-derivation
functions are unit-tested with no database and no network. **The headline suite
is cross-tenant denial**, running in CI on every PR: an operator in tenant A
receives 404 for every consumer record in tenant B, a consumer receives 404 for
another consumer in their own tenant, and every denial lands in the audit trail.
A schema test asserts that no `coach` or `onboarding` table has a column capable
of holding a raw provider payload. Backend milestones are verified on `stage` via
an authenticated CLI walkthrough; console milestones with an authenticated
Playwright walkthrough plus screenshots across both branded tenants.

**"Implemented locally" is not a completion state.**
