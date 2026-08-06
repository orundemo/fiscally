# fiscally — Design

Status: Ready for implementation. This is the technical design for the
`onboarding` and `coach` product bounded contexts and the commercial wiring that
joins them.

## 1. The shape of the problem

The brief describes a white-label B2B2C financial-wellness platform: operators
(businesses) serve consumers under the operator's brand; consumers onboard with
e-signature and identity verification, receive coaching from an LLM, and are
billed on a mixed model. Tenant isolation "will be reviewed". Certain data must
never be stored. MVP acceptance is contractually due in week 5.

Decomposed:

| Capability | Owner |
|---|---|
| Tenants, users, four role types, audit | platform — shipped |
| Consumer subscriptions, per-seat plans, metered add-ons | platform — shipped |
| **Prove isolation to a reviewer's satisfaction** | **new — and first** |
| **Resumable onboarding through two external providers** | **new** — `onboarding` |
| **Analyse credit data without persisting it** | **new** — the hard constraint |
| **Coach a consumer with an LLM, safely** | **new** — `coach` |
| **Charge only when the outcome succeeded** | **new** — a saga |
| **Two tenants, one deployment, different brands** | **new** — configuration |

Two of these are unusual enough to shape the whole design.

**The isolation review.** Most projects treat tenant isolation as an invariant
they assert. This client will audit it. So it becomes a *deliverable with
artifacts*: an automated denial suite that runs in CI, a second scoping grain
below the tenant, and a page in the product that demonstrates a denied
cross-tenant read arriving in the audit trail. FS1 lands this before any feature,
because it is the thing being bought.

**The never-store rule.** A rule enforced by discipline degrades the moment
someone is in a hurry. A rule enforced by schema cannot. The `coach` tables have
no column able to hold a provider payload — no `raw`, no `response`, no
untyped blob wide enough. Storing the payload would require a migration, which
would require review. That is the difference between a promise and a property.

## 2. Bounded contexts

### Why two workers

The signalpilot epic put one product context in one worker and argued against
splitting. The rule is *one worker per bounded context*, and here there are two:

| | `onboarding` | `coach` |
|---|---|---|
| Owns | applications, steps, signature requests, verification checks | derived profiles, sessions, messages, action plans |
| External dependencies | e-sign provider, IDV provider | model provider, credit-data widget |
| Failure mode | a provider stalls → applications queue | a model is slow → coaching degrades |
| Cadence | bursty, human-paced, minutes-to-days | interactive, sub-second expectations |
| Data sensitivity | identity evidence (references only) | financial derivations (bucketed only) |
| Retention | tied to the application lifecycle | tied to the consumer relationship |

They share only `subject_user_id`, an opaque identity reference. A stalled IDV
provider must not add latency to a coaching turn, and a model outage must not
block someone finishing their signup. One worker would couple both.

### Anatomy

```
apps/onboarding-worker/            apps/coach-worker/
  component.yaml                     component.yaml
    dependsOn: [membership, policy,     dependsOn: [membership, policy,
                events, notifications]              billing, metering, events]
  src/                               src/
    index.ts router.ts env.ts ...      index.ts router.ts env.ts ...
    membership-client.ts               membership-client.ts
    policy-client.ts                   policy-client.ts
    events-client.ts                   billing-client.ts  metering-client.ts
    adapters/                          adapters/
      esign/{sandbox,provider}.ts        model/{sandbox,provider}.ts
      idv/{sandbox,provider}.ts          creditdata/{sandbox,provider}.ts
    handlers/*.ts                      handlers/*.ts
    engine/                            engine/
      flow.ts      the state machine     derive.ts     payload → features
      redact.ts    provider → refs       supervisor.ts the guardrail harness
tests/onboarding-worker/           tests/coach-worker/
tests/tenancy/                     ← the shared cross-tenant denial suite (FS1)
```

`tests/tenancy` is its own test package because it spans both workers and the
platform: it is the isolation review, expressed as code.

## 3. The two grains of scope

Every `onboarding` and `coach` row carries **two** scoping columns:

- `org_id` — the tenant (an operator business, or the platform's own consumer
  tenant)
- `subject_user_id` — the consumer the record is about

Authorization resolves a **scope** rather than a boolean:

| Role | Scope |
|---|---|
| `consumer` | `org_id = theirs AND subject_user_id = self` |
| `operator` | `org_id = theirs AND subject_user_id IN (their assigned consumers)` |
| `operator_admin` | `org_id = theirs` |
| `platform_admin` | any — through `admin-worker`, audited, never through the ordinary path |

The repository layer takes the resolved scope as a required argument; there is
no repository method that accepts an id without one. A consumer reading another
consumer's session in the *same* tenant is as much a denial as a cross-tenant
read, and both are tested.

## 4. Data model

Three migrations. The platform's last is
`190_integrations_delivery_attribution`, so the product block starts at `200`.

### `200_onboarding_core`

#### `onboarding.applications`

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `apl_<hex>` |
| `org_id` | `UUID NOT NULL` | tenant |
| `subject_user_id` | `UUID NOT NULL` | the consumer |
| `flow_version` | `INT NOT NULL` | which step graph applies |
| `current_step` | `TEXT NOT NULL` | step key |
| `status` | `TEXT NOT NULL DEFAULT 'in_progress'` | `CHECK IN ('in_progress','pending_review','approved','rejected','expired','abandoned')` |
| `started_at` / `completed_at` / `expires_at` | `TIMESTAMPTZ` | |
| `assigned_operator_id` | `UUID` | which operator owns this consumer |

Partial unique `(org_id, subject_user_id) WHERE status = 'in_progress'`. Index
`(org_id, status, started_at DESC)`.

#### `onboarding.steps` — append-only

`id` `stp_<hex>`, `org_id`, `subject_user_id`, `application_id`, `step_key`,
`outcome` (`completed`/`skipped`/`failed`), `payload_digest` (sha256 of what the
client submitted — **not** the submission), `duration_ms`, `created_at`.

Append-only: a re-attempted step is a new row. "Where did people drop out" is a
query, and resumption reads the latest row per step.

#### `onboarding.signature_requests`

`id` `sig_<hex>`, `org_id`, `subject_user_id`, `application_id`, `provider`,
`provider_ref`, `document_template`, `document_version`, `envelope_status`
(`created`/`sent`/`viewed`/`signed`/`declined`/`expired`), `signed_at`,
`evidence_digest`. **The executed document is not stored here** — it lives with
the provider and is referenced by `provider_ref`; `evidence_digest` pins what
was signed.

#### `onboarding.verification_checks`

`id` `ver_<hex>`, `org_id`, `subject_user_id`, `application_id`, `provider`,
`provider_ref`, `check_type` (`document`/`liveness`/`address`/`sanctions`),
`outcome` (`pass`/`refer`/`fail`), `reason_codes` `TEXT[]`, `source_digest`,
`checked_at`.

**No images, no extracted PII, no raw provider response.** Outcome plus reason
codes is what the product needs; everything else stays with the provider under
their retention policy, which is also the compliance answer.

### `210_coach_core`

#### `coach.profiles` — the derived view of a consumer's finances

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `cpr_<hex>` |
| `org_id` / `subject_user_id` | `UUID NOT NULL` | both grains |
| `features` | `JSONB NOT NULL` | **bucketed derivations only** — e.g. `{"utilisation_band":"high","delinquencies_12m":0,"file_age_band":"3-5y","score_band":"fair"}` |
| `features_version` | `INT NOT NULL` | the derivation function version |
| `source_digest` | `TEXT NOT NULL` | sha256 of the payload the derivation came from |
| `source_provider` | `TEXT NOT NULL` | |
| `computed_at` / `expires_at` | `TIMESTAMPTZ` | staleness horizon |

Partial unique `(org_id, subject_user_id) WHERE expires_at > now()`.

**Bands, not values.** `utilisation_band: "high"` rather than `utilisation:
0.83`; `score_band: "fair"` rather than a score. The coach does not need the
number, and a band is not a credit report. This is the schema-level expression
of the never-store rule — and a schema test asserts every `features` key ends in
`_band`, is a bounded enum, or is a small integer count.

#### `coach.sessions`

`id` `ses_<hex>`, `org_id`, `subject_user_id`, `topic`, `status`
(`active`/`ended`), `profile_id`, `started_at`, `ended_at`.

#### `coach.messages` — append-only

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `msg_<hex>` |
| `org_id` / `subject_user_id` | `UUID NOT NULL` | |
| `session_id` | `UUID NOT NULL` | |
| `role` | `TEXT NOT NULL` | `CHECK IN ('user','assistant')` |
| `content` | `TEXT NOT NULL` | |
| `model` / `prompt_version` | `TEXT` / `INT` | provenance, assistant rows only |
| `guardrail_verdict` | `TEXT` | `CHECK IN ('pass','revised','blocked')` |
| `guardrail_notes` | `JSONB` | which checks fired |
| `input_digest` | `TEXT` | cache key |
| `created_at` | `TIMESTAMPTZ` | |

#### `coach.action_plans`

`id` `pln_<hex>`, `org_id`, `subject_user_id`, `session_id`, `items` JSONB
(each `{id, title, rationale_ref, target_band, status}`), `status`, `created_at`,
`completed_at`. `rationale_ref` points at a reviewed template, not free text —
see §6.

### `220_fiscally_commercial`

#### `fiscally.seats`

`id` `seat_<hex>`, `org_id`, `user_id`, `plan_code`, `assigned_at`,
`released_at`, `assigned_by`. Partial unique
`(org_id, user_id) WHERE released_at IS NULL`.

A seat is distinct from a membership: a member can exist without a paid seat
(e.g. a read-only observer). Per-seat billing counts seats, not members.

#### `fiscally.success_events` — the charge-on-success ledger

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `suc_<hex>` |
| `org_id` / `subject_user_id` | `UUID NOT NULL` | |
| `kind` | `TEXT NOT NULL` | which success condition — e.g. `onboarding_approved`, `plan_milestone_reached` |
| `evidence` | `JSONB NOT NULL` | ids of the records that prove it |
| `charge_state` | `TEXT NOT NULL DEFAULT 'pending'` | `CHECK IN ('pending','charging','charged','failed','waived')` |
| `charge_ref` | `TEXT` | provider reference |
| `idempotency_key` | `TEXT NOT NULL UNIQUE` | derived from `(org_id, subject_user_id, kind, evidence digest)` |
| `attempts` | `INT NOT NULL DEFAULT 0` | |
| `occurred_at` / `charged_at` | `TIMESTAMPTZ` | |

The unique idempotency key is the whole safety property — see §7.

Each migration adds a `packages/db/src/manifest.ts` entry with its sha256; the
runner refuses an unlisted or drifted file.

## 5. Onboarding

### The flow engine

`engine/flow.ts` is a pure state machine over a versioned step graph:

```
account → profile → disclosures → esign → idv_document → idv_liveness → review → approved
                                     │           │              │           │
                                     └───────────┴──────────────┴─── refer ─┘ (pending_review)
```

`nextStep(flowVersion, completedSteps, outcomes) → stepKey | 'complete'`. No
clock, no I/O. Resumption is therefore trivial and correct: read the step rows,
call the engine, return the step. A consumer who abandons at `idv_liveness`
returns to `idv_liveness`, not to `account`.

`flow_version` is pinned on the application at creation, so a graph change never
strands someone mid-flow.

### The provider adapters

```ts
interface EsignAdapter {
  readonly id: string
  createEnvelope(input: EnvelopeInput, ctx): Promise<{ providerRef: string; url: string }>
  status(providerRef: string, ctx): Promise<EnvelopeStatus>
}

interface IdvAdapter {
  readonly id: string
  createCheck(input: CheckInput, ctx): Promise<{ providerRef: string; url?: string }>
  result(providerRef: string, ctx): Promise<{ outcome: Outcome; reasonCodes: string[]; sourceDigest: string }>
}
```

Note what the interfaces *cannot* return: no document bytes, no extracted
fields, no images. `engine/redact.ts` sits between the raw provider response and
the adapter's return value, and it is unit-tested with recorded provider
fixtures to prove nothing but outcome, reason codes, and a digest escapes.

v1 ships a deterministic `sandbox` adapter for each (powering the demo and every
test) plus one real provider each.

Provider callbacks arrive at a dedicated edge route with signature verification,
and are treated as *hints*: the worker always re-reads authoritative status from
the provider before writing. A forged callback therefore cannot approve an
application.

## 6. The coach

### The ephemeral pipeline

The credit-data widget embeds in the console and returns a payload to the
worker. Within a single request:

1. verify the payload's provenance and signature
2. `engine/derive.ts` → bucketed features (pure, versioned, unit-tested)
3. compute `source_digest` = sha256 of the payload
4. write `coach.profiles` with features + digest
5. **drop the payload** — it is never assigned to a variable that outlives the
   request, never logged, never sent to the model
6. emit `coach.profile.derived` carrying `{features_version, source_digest,
   dropped: ["raw_report","tradelines","inquiries","identifiers"]}`

Step 6 is the compliance artifact: an append-only audit record stating **what
was not stored**. When the reviewer asks how they can trust the rule, the answer
is a query against their own audit log, plus a schema with nowhere to put the
data.

### The supervisor harness

The model receives the bucketed features, the session history, and a reviewed
system prompt — nothing else. Every generation passes `engine/supervisor.ts`
before it is stored or returned:

| Check | Rule |
|-------|------|
| Grounding | every claim maps to a feature present in the profile; unmapped claims are stripped |
| No numbers it wasn't given | the model may not state a score, a balance, or a rate — it has bands, and bands are what it may say |
| No regulated advice | no product recommendations, no "you should apply for", no guarantees of outcome; a reviewed template space bounds what an action-plan item may assert |
| No PII echo | no identifiers, addresses, or account numbers may appear |
| Bounds | length, tone, banned phrases (no false urgency, no invented testimonials) |

Verdict is stored on the message: `pass`, `revised` (checks stripped content), or
`blocked` (unsalvageable — nothing stored, typed error returned, **not billed**).
Action-plan items carry a `rationale_ref` into the reviewed template set rather
than free-form model prose, so the advice surface is bounded by construction.

Generations are digest-cached and metered
(`fiscally.coach.messages` on `metering-worker`), and the entitlement check runs
**before** the model call.

## 7. The charge-on-success saga

Charging when an outcome succeeds is where naive implementations double-charge
or silently under-charge. The design is a three-phase saga with a durable
ledger:

1. **Record.** The success condition — onboarding approved, plan milestone
   reached — writes a `success_events` row with `charge_state = 'pending'` and a
   deterministic `idempotency_key` derived from the tenant, subject, kind, and a
   digest of the evidence. The unique constraint means the same success recorded
   twice is one row.
2. **Charge.** A worker transitions `pending → charging` (a conditional update,
   so only one attempt can claim it), calls `billing-worker` with the same
   idempotency key, then writes `charged` + `charge_ref`, or `failed` +
   increments `attempts`.
3. **Reconcile.** A scheduled sweep retries `failed` rows with backoff up to a
   bound, then raises them for operator attention. A row stuck in `charging`
   past a timeout is re-driven — safe, because the provider call is idempotent
   on the same key.

`waived` exists as an explicit operator action with an audit record, because
"we're not charging this one" is a real business decision and should not be
achieved by deleting a row.

**No charge without a durable success row; no second charge on any retry path.**
Both properties are tested by driving concurrent duplicate successes and
injecting provider failures.

## 8. Commercial surface

| Shape | Mechanism |
|---|---|
| Consumer subscription | `billing-worker` plan on the consumer's org |
| Per-seat operator plan | seat count from `fiscally.seats`, synced to the subscription quantity on assign/release |
| Metered add-ons | `metering-worker` meters: `fiscally.coach.messages`, `fiscally.idv.checks`, `fiscally.esign.envelopes` |
| Charge-on-success | §7 |

Entitlement checks run at request time via `billing-client`, mirroring
`projects-worker`. Exhaustion returns a typed `quota_exhausted` error carrying
meter, limit, and reset date, and emits `fiscally.quota.exhausted`.

**Events** (`events-worker` + `webhooks-worker`):
`onboarding.application.started` · `.step_completed` · `.submitted` ·
`.approved` · `.rejected` · `onboarding.verification.completed` ·
`onboarding.signature.completed` · `coach.profile.derived` ·
`coach.message.generated` · `coach.plan.created` ·
`fiscally.success.recorded` · `.charged` · `.charge_failed` ·
`fiscally.seat.assigned` · `.released` · `fiscally.quota.exhausted`

**Notifications:** application approved/rejected (consumer), verification
referred (operator), plan milestone reached (both), charge failed (operator
admin), seat assigned (the new seat holder).

## 9. White-label theming

Branding resolves from `config-worker` settings per org: name, logo URL, colour
tokens, typography scale, support contact, legal entity, custom domain. The
console reads the resolved theme in its root layout and applies it as CSS custom
properties; emails read the same settings through `notifications-worker`
templates.

**One deployment, many brands.** There is no build-time variant, no per-tenant
fork, and no separate deploy. The demo proves it by running two visibly
different tenants side by side on the same URL family — which is also the
cheapest possible refutation of "did they just theme it once for the demo".

A tenant with no branding configured falls back to platform defaults, so a new
operator is never broken, only unstyled.

## 10. API surface

Under `/v1/organizations/:orgId/`, all behind the standard gate plus scope
resolution (§3), deny-as-404.

**Onboarding:** `POST /applications` · `GET /applications` ·
`GET /applications/:id` · `POST /applications/:id/steps` ·
`POST /applications/:id/signature` · `GET /applications/:id/signature` ·
`POST /applications/:id/verification` · `GET /applications/:id/verification` ·
`POST /applications/:id/submit` · `POST /applications/:id/decision` (operator
review) · plus the provider callback route at the edge.

**Coach:** `POST /profiles/derive` · `GET /profiles/current` ·
`POST /sessions` · `GET /sessions` · `GET /sessions/:id/messages` ·
`POST /sessions/:id/messages` · `GET`/`POST /plans` ·
`PATCH /plans/:id/items/:itemId`.

**Commercial:** `GET`/`POST /seats` · `DELETE /seats/:id` ·
`GET /success-events` · `POST /success-events/:id/waive` ·
`GET`/`PUT /branding`.

**Edge.** Two facades — `onboarding-facade.ts` and `coach-facade.ts` — plus the
signed provider-callback route. Stricter rate-limit classes on
`POST /sessions/:id/messages` (model cost), `POST /applications/:id/verification`
(provider cost), and the callback route.

## 11. RBAC

| Action | consumer | operator | operator_admin |
|---|:--:|:--:|:--:|
| `organization.application.read` | own | assigned | all |
| `organization.application.start` / `.step` | own | — | — |
| `organization.application.review` / `.decide` | — | assigned | all |
| `organization.verification.read` | own (outcome only) | assigned | all |
| `organization.coach.session.start` / `.message` | own | — | — |
| `organization.coach.session.read` | own | assigned | all |
| `organization.coach.plan.read` | own | assigned | all |
| `organization.coach.plan.write` | own | assigned | all |
| `organization.seat.list` | — | — | ✓ |
| `organization.seat.assign` / `.release` | — | — | ✓ |
| `organization.branding.read` | ✓ | ✓ | ✓ |
| `organization.branding.write` | — | — | ✓ |
| `organization.success_event.read` / `.waive` | — | read | ✓ |

`platform_admin` reaches everything through `admin-worker`, audited. Note that
`consumer` rows in this matrix mean *own subject scope*, not org-wide — the
second grain from §3 is enforced in the policy path, not in handlers.

## 12. Console

Four role surfaces on their design system, ~30+ views:

| Surface | Content |
|---|---|
| Consumer | onboarding wizard (resumable), profile summary in bands, coach chat with guardrail badge, action plan, billing |
| Operator | assigned consumer list, application queue with referred verifications, per-consumer coach history (read), plan progress |
| Operator admin | everything an operator sees, plus seats, branding, plans and usage, success-event ledger with waive |
| Platform admin | tenant list, the isolation proof page, audit search, support actions |

**The isolation proof page** is a first-class product surface, not a demo hack:
it runs a live cross-tenant read attempt against a fixture record in another
tenant, shows the 404, and links to the resulting audit entry. The reviewer runs
the review themselves.

## 13. Non-goals (v1)

- Lending, credit decisioning, or any regulated financial advice output.
- Storing raw bureau payloads, KYC document images, or extracted PII — under any
  circumstance, including "temporarily for debugging".
- Direct bureau contracts; the widget embed is the boundary.
- Mobile applications.
- Multi-currency.
- Migrating the partner's existing skeleton codebase (see
  `risks-and-open-questions.md`).

## 14. Verification bar

| Layer | How it is verified |
|-------|--------------------|
| `tests/tenancy` | cross-tenant *and* cross-subject denial for every route in §10, each asserting 404 **and** an audit row — runs in CI on every PR |
| Schema | an automated test asserts no `coach`/`onboarding` table has a column able to hold a raw payload, and every `features` key is a band, enum, or small count |
| `engine/derive.ts` | unit tests mapping recorded fixtures to bands; same input, same features, always |
| `engine/redact.ts` | recorded provider fixtures in, only outcome + reason codes + digest out |
| `engine/supervisor.ts` | adversarial generations killed for each check in §6 |
| `engine/flow.ts` | resumption from every step, including after a `refer` outcome |
| Charge saga | concurrent duplicate successes produce one charge; injected provider failures retry and never double-charge |
| Theming | two tenants render visibly differently from one deployment; an unconfigured tenant falls back cleanly |
| Stage | authenticated CLI walkthrough: onboard → verify → derive → coach → plan → success → charge |
| Console | Playwright walkthrough across both branded tenants, four logins, screenshots |

"Implemented locally" is not a completion state.
