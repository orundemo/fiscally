# fiscally — Implementation status

As-built record. This file tracks what actually shipped, kept deliberately
distinct from `design.md` (intent) and `implementation-plan.md` (plan).

## Summary

| Field | Value |
|-------|-------|
| Epic status | **Draft — not started** |
| Product repo | not yet bootstrapped |
| Branch | `epic/fiscally` (to be created after `flows/phases/00-all`) |
| Milestones shipped | none |
| Live on `stage` | no |
| Live on `prod` | no |
| Demo URL | `fiscally.orun.dev` (not yet provisioned) |

Nothing has been implemented. This epic carries the charter, the technical
design, and the FS0–FS10 milestone ladder; the first code lands with FS0, after
the product repo is born from the baseline.

## Bootstrap prerequisites

| Step | What it lands | Time |
|---|---|---|
| `flows/phases/00-all` | the full platform layer — 13 workers, console, infra, CI — live on `stage` and `prod` | ~73 min, unattended |
| `tooling/rebrand/rebrand.mjs --values fiscally-brand.json --verify` | repo slug, product name, domain, CLI bin, env prefix, worker names, service bindings | minutes |
| `CONSOLE_CUSTOM_DOMAIN` | `fiscally.orun.dev` on the zone we already control | — |

**Credential-blocked tails to schedule early** (these have lead time and will
stall FS3/FS5/FS6 if left late): e-sign and IDV provider sandbox accounts, the
model provider key, Stripe/Polar products for three plan shapes, and the
credit-data widget's sandbox embed.

## Baseline this epic starts from

Recorded so that "what did the product layer add" stays answerable later. Taken
from the Lumen baseline as of 5 Aug 2026.

| Layer | State at epic open |
|-------|--------------------|
| Workers | 13: `api-edge`, `identity`, `membership`, `projects`, `policy`, `events`, `config`, `metering`, `billing`, `notifications`, `webhooks`, `admin`, `integrations` |
| Console | `web-console-next` live; org surfaces: `api-keys`, `audit`, `billing`, `config`, `invitations`, `members`, `projects`, `settings`, `usage`, `webhooks` |
| Packages | `contracts`, `policy-engine`, `db`, `sdk`, `cli`, `shared`, `testing`, `notifications-client`, `webhook-verifier` |
| Migrations | `000_control` → `190_integrations_delivery_attribution` — product context starts at `200` |
| Policy actions | `ORGANIZATION_ACTIONS` platform-only; scoping is org-level only — **this epic adds the `subject_user_id` grain** |
| Billing | Polar adapter live end to end; per-seat and metered primitives present, no seat ledger yet |
| Theming | platform defaults only — no per-tenant branding |
| Environments | `dev` (verify-only), `stage`, `prod`, converging through Orun |
| Composition stack | `oci://ghcr.io/sourceplane/stack-tectonic:0.18.2` (pinned) |

## Milestone log

| ID | Milestone | Status | PR | Verified on | Notes |
|----|-----------|--------|----|-------------|-------|
| FS0 | Contract + data model + two workers | Draft | — | — | |
| FS1 | Isolation proof | Draft | — | — | the reviewed deliverable |
| FS2 | Onboarding flow engine | Draft | — | — | |
| FS3 | E-sign + IDV adapters | Draft | — | — | provider accounts have lead time |
| FS4 | Ephemeral analysis pipeline | Draft | — | — | |
| FS5 | Coach + supervisor harness | Draft | — | — | blocked on template sign-off (OQ 1) |
| FS6 | Commercial + charge-on-success | Draft | — | — | |
| FS7 | White-label theming | Draft | — | — | |
| FS8 | Edge + SDK + CLI | Draft | — | — | |
| FS9 | Console — four surfaces | Draft | — | — | estimate depends on OQ 8 |
| FS10 | Evidence + two demo tenants | Draft | — | — | |

## Deviations from design

None recorded. Any implementation that diverges from `design.md` is noted here
with the reason, rather than by silently editing the design.
