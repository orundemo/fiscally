# fiscally — Implementation plan

Milestones FS0–FS10. Each is independently shippable and independently
verifiable on `stage`.

**Demo** is the thin-vertical demo build for the proposal. **Client** is the
contracted engagement — and that column is the one that matters here, because
**MVP acceptance is contractually due in week 5**. The week map is at the bottom.

| ID | Milestone | Demo | Client |
|----|-----------|------|--------|
| FS0 | Contract + data model + two workers | 2 h | 1.5 d |
| FS1 | **Isolation proof** | 3 h | 1.5 d |
| FS2 | Onboarding flow engine | 2 h | 1.5 d |
| FS3 | E-sign + IDV adapters | 2 h (sandbox only) | 2 d |
| FS4 | Ephemeral analysis pipeline | 2 h | 1 d |
| FS5 | Coach + supervisor harness | 3 h | 2 d |
| FS6 | Commercial + charge-on-success saga | 2 h | 2 d |
| FS7 | White-label theming | 1 h | 1 d |
| FS8 | Edge + SDK + CLI | 1 h | 1 d |
| FS9 | Console — four surfaces, 30+ views | 4 h | 5 d |
| FS10 | Evidence + two demo tenants | 2 h | 1.5 d |
| | **Total** | **~3 working days** | **~20 working days** |

> Twenty client-grade days against a five-week contractual date is achievable
> only because the platform layer is already live — but it has no slack. The
> scope-freeze clause in `risks-and-open-questions.md` is not boilerplate.

---

## FS0 — Contract, data model, two workers

**Lands:** `packages/contracts/src/onboarding.ts` and `coach.ts`; migrations
`200_onboarding_core`, `210_coach_core`, `220_fiscally_commercial`; three
manifest entries with sha256; `packages/db/src/{onboarding,coach,fiscally}/`
repositories **whose every method requires a resolved scope argument**; the new
actions registered in `@saas/contracts/policy` and `@saas/policy-engine`; both
worker skeletons with health routes.

**Done when:**

- `kiox -- orun validate --intent intent.yaml` passes with both new components
  discovered.
- `db-migrate` plans on the PR and applies on merge; all three manifest entries
  accepted.
- **No repository method accepts a bare id** — a compile-time property, asserted
  by a type test.
- `tests/policy-engine` proves the four-role matrix in `design.md` §11,
  including that `consumer` resolves to own-subject scope rather than org-wide.
- Both workers deploy to `stage` and answer health.

---

## FS1 — The isolation proof

**This lands before any feature, because it is what the client said they will
review.**

**Lands:** the `tests/tenancy` package; scope resolution wired into the policy
path for both new contexts; the schema-shape test; the audit records for denied
reads; the platform-admin isolation proof page (backend + a minimal view).

**Done when:**

- **Cross-tenant:** an `operator_admin` in tenant A receives 404 for every route
  in `design.md` §10 against a tenant B record — enumerated, not sampled.
- **Cross-subject:** a `consumer` receives 404 for another consumer's records
  *within their own tenant*, and an `operator` receives 404 for an unassigned
  consumer.
- Every denial writes an audit entry; a test asserts the entry, not just the
  status code.
- The schema test passes: no table in `onboarding` or `coach` has a column able
  to hold a raw provider payload, and every `coach.profiles.features` key is a
  `_band` suffix, a bounded enum, or a small integer count.
- The suite runs in CI on every PR and **fails the build** — this is a gate, not
  a report.
- The isolation proof page performs a live denied read and links to the audit
  entry it produced.

---

## FS2 — Onboarding flow engine

**Lands:** `engine/flow.ts`; `POST /applications`, `GET /applications`,
`GET /applications/:id`, `POST /applications/:id/steps`; append-only step
records; `flow_version` pinning; `onboarding.application.*` events.

**Done when:**

- The engine is unit-tested for resumption from every step, including after a
  `refer` outcome, with no DB and no clock.
- An abandoned application resumes at the correct step, not the first one.
- A `flow_version` change does not strand in-flight applications — proven by
  advancing an application pinned to v1 after v2 ships.
- Step rows are append-only; a re-attempt adds a row and drop-off is a query.
- The partial unique constraint prevents two concurrent in-progress applications
  for one subject.

---

## FS3 — E-sign and IDV adapters

**Lands:** both adapter interfaces; `engine/redact.ts`; the deterministic
`sandbox` adapters; one real provider each; the signed provider-callback edge
route; `signature_requests` and `verification_checks`; metering on
`fiscally.esign.envelopes` and `fiscally.idv.checks`.

**Done when:**

- `engine/redact.ts` is unit-tested against **recorded real provider fixtures**,
  asserting that only outcome, reason codes, and a digest escape — no images, no
  extracted fields, no raw response.
- A forged callback cannot advance an application: the worker always re-reads
  authoritative status from the provider before writing, proven by a test that
  posts a validly-signed callback claiming `signed` while the provider says
  `sent`.
- Callback signature verification rejects a bad signature with no state change.
- The sandbox adapters are fully deterministic — the demo and the whole test
  suite run with no external calls and no credentials.
- A provider outage leaves the application resumable rather than failed.

---

## FS4 — The ephemeral analysis pipeline

**Lands:** the credit-data widget adapter; `engine/derive.ts`;
`POST /profiles/derive`; `GET /profiles/current`; the
`coach.profile.derived` event carrying the `dropped:[…]` list.

**Done when:**

- Derivation is pure and versioned: the same payload always yields the same
  features, and `features_version` is stamped on every row.
- **A memory/log test proves the payload does not escape the request**: it is
  never logged, never returned, never persisted, never sent to the model.
- The emitted event names what was dropped, and that record is visible in the
  client's own audit console — the compliance artifact.
- Every stored feature is a band, enum, or small count (the FS1 schema test
  covers this and must still pass).
- A stale profile past `expires_at` is not used for coaching; the consumer is
  prompted to refresh.

---

## FS5 — Coach and the supervisor harness

**Lands:** the model adapter; `engine/supervisor.ts`; sessions, messages, action
plans; the reviewed rationale-template set; digest caching; the entitlement
gate; `coach.*` events.

**Done when:**

- Supervisor unit tests kill an adversarial generation for **each** check in
  `design.md` §6 — ungrounded claim, a number it was not given, regulated
  advice, PII echo, banned phrasing.
- A `blocked` verdict stores nothing, returns a typed error, and is **not**
  metered.
- Action-plan items carry a `rationale_ref` into the reviewed template set; a
  test asserts no free-form model prose reaches a plan item.
- The entitlement check happens **before** the model call — verified by driving a
  tenant to its limit and asserting no provider request is made.
- Repeat identical turns hit the digest cache and are not re-metered.

---

## FS6 — Commercial and the charge-on-success saga

**Lands:** consumer subscription plans; `fiscally.seats` with subscription
quantity sync on assign/release; the three metered add-ons; the
`success_events` ledger and the three-phase saga; the reconcile sweep; the
`waive` action; `fiscally.*` commercial events; the quota-exhausted contract.

**Done when:**

- **Concurrent duplicate successes produce exactly one charge** — driven with
  parallel writers against the unique idempotency key.
- An injected provider failure marks `failed`, retries with backoff, and never
  double-charges on the retry path.
- A row stuck in `charging` past the timeout is safely re-driven.
- Seat assign/release syncs subscription quantity; releasing the last seat does
  not cancel the subscription.
- `waive` requires an operator-admin action and writes an audit record.
- A tenant past a metered limit gets the typed `quota_exhausted` payload with
  meter, limit, and reset date, and the console renders the upgrade prompt from
  that payload alone.

---

## FS7 — White-label theming

**Lands:** branding settings schema in `config-worker`; `GET`/`PUT /branding`;
theme resolution in the console root layout as CSS custom properties; themed
notification templates; the platform-default fallback.

**Done when:**

- Two tenants on the **same deployment** render visibly different brands — name,
  logo, colours, typography — with no build-time variant and no redeploy.
- Emails carry the tenant's brand.
- A tenant with no branding configured renders platform defaults and is never
  broken.
- Changing a colour token takes effect on the next request, not the next deploy.

---

## FS8 — Edge, SDK, CLI

**Lands:** `onboarding-facade.ts` and `coach-facade.ts` in the dispatch chain;
the signed provider-callback route; stricter rate-limit classes on the model,
IDV, and callback routes; `packages/sdk` methods; `packages/cli` command groups.

**Done when:**

- Every route in `design.md` §10 is reachable through the public edge with
  tenancy resolution, idempotency, and rate limiting applied.
- The SDK is generated against the contracts modules — no hand-written types.
- One authenticated CLI walkthrough on stage performs: onboard → verify →
  derive → coach → plan → success → charge. That transcript is the evidence.
- Each facade stays under ~100 LOC.

---

## FS9 — Console, four role surfaces

**Lands:** the ~30+ views in `design.md` §12 against **their** design system and
per-view component map.

**Done when:**

- All four surfaces exist, each with empty, loading, error, denied, and
  quota-exhausted states.
- The consumer onboarding wizard resumes correctly on reload and on a new
  device.
- The coach chat shows the guardrail verdict badge on every assistant message.
- Profile figures render as **bands**, never invented precision.
- The isolation proof page is reachable by a platform admin and runs live.
- A Playwright walkthrough passes on stage **across both branded tenants**, with
  screenshots attached.
- No console route calls a worker directly; everything goes through the edge.

---

## FS10 — Evidence and demo tenants

**Lands:** two seeded tenants with distinct branding; **four logins**, one per
user type; consumers at every onboarding stage including one `referred`; derived
profiles with visible bands; a coach session showing one `pass` and one `revised`
verdict; an action plan mid-progress; a populated success-event ledger with one
`charged` and one `waived`; `docs/{overview,architecture,runbook}.md`;
`catalog.entities` enrichment; an `08-docs` re-run.

**Done when:**

- `https://fiscally.orun.dev` serves both tenants, visibly different.
- The four logins are in the proposal body, each landing somewhere immediately
  legible.
- **The isolation proof is the headline**: a reviewer can, in under a minute,
  trigger a denied cross-tenant read and see it in the audit trail. They said
  they will review isolation — this is handing them the review.
- `ai/context/deployment.md` and `operations.md` regenerate against verified live
  state.
- **`docs/overview.md` is read with fresh eyes and describes *this* product** —
  the known failure mode is docs rebranded independently of the domain.

---

## Sequencing and the week-5 map

FS0 blocks everything. **FS1 is deliberately second** — before features —
because it is the reviewed deliverable and because retrofitting a second scoping
grain after five contexts exist is expensive. FS2 → FS3 is a chain; FS4 → FS5 is
a chain; the two chains are independent and can run in parallel. FS6 needs FS2
and FS5 for real success conditions. FS7 is independent throughout. FS8 grows
from FS2 onward — do not defer it, because the CLI walkthrough is how FS2–FS6
get verified. FS9 needs FS8's SDK.

| Week | Lands |
|---|---|
| 1 | FS0, FS1, FS2 — their design system on a running multi-tenant app with the isolation proof already green |
| 2 | FS3, FS4 — both external-provider surfaces and the ephemeral pipeline |
| 3 | FS5, FS6 — coach and the full commercial surface |
| 4 | FS7, FS8, FS9 (start) — theming, edge/SDK/CLI, the bulk of the views |
| 5 | FS9 (finish), FS10 — remaining views, seeded tenants, evidence, **acceptance** |

**Week 1 is the proposal's strongest claim:** *"Week 1 is your design system on
top of a running multi-tenant app, not scaffolding."* It is true only because
FS0–FS2 sit on a platform that is already live — and it is the sentence that
makes a week-5 date credible rather than reckless.

**Never** hand-deploy. A `wrangler deploy` or `terraform apply` by hand is drift
the next plan will fight.
