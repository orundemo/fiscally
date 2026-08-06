# fiscally — Market context

Why this epic exists, what the buyer is actually buying, and what has to be in
the envelope alongside the code.

> Sourced from the shortlist analysis in `~/sourceplane/orun-upwork-top5.md`
> (prepared 3 Aug 2026). Upwork disallows automated fetching, so the figures
> below are as recorded at that time and should be re-checked in the browser
> before a proposal is sent.

## The originating job

**"Multi-Tenant Fintech SaaS"** — a white-label B2B2C build delivered under a
partner agency's branding. Ranked #4 of five.

- **Link:** https://www.upwork.com/jobs/Multi-Tenant-Fintech-span-class-highlight-SaaS-span_~022079584437221826762/
- **Budget:** $4,000 fixed · **Featured**
- **Posted:** ~20 Jul 2026 · 20–50 proposals · **Interviewing: 0** · **last viewed 2 weeks ago**
- **Client:** India; Tech & IT; mid-sized company (10–99); payment + phone verified; 8 jobs posted, **0% hire rate**; member since Jun 2025
- **Connects:** 19

**What they say they want:** turn a finished skeleton into a production
application. Locked scope, a per-view component map, an interactive UI demo of
30+ views, flow diagrams, a written source of truth. Next.js App Router +
TypeScript + Tailwind/shadcn; **Supabase Postgres + RLS multi-tenant backend
where "tenant isolation is architecturally critical and will be reviewed"**;
RBAC across four user types; e-signature + IDV onboarding; **Stripe with consumer
subscriptions, per-seat operator plans, metered add-ons, and charge-on-success**;
an LLM coach with a supervisor/guardrail layer; a credit-data widget embed with a
**never-persist-raw-payloads** rule. MVP acceptance contractually due **week 5**.

**What they actually want.** This is a **dev shop subcontracting a client
build** — we would deliver under their brand while they keep PM and the client
calls. Three signals matter:

1. ***"We'll ask for live URLs, not videos."*** They are explicitly running the
   exact evaluation our strategy is built for. The demo is the submission.
2. ***"Tenant isolation … will be reviewed"*** and ***"'never store X' rules are
   hard constraints."*** They have been burned by a vibe-coded submission and are
   now screening for architectural discipline. Everything in this epic's FS1 and
   decision 5 exists because of those two sentences.
3. **Interviewing: 0, last viewed two weeks ago.** The post is stale — which is
   *good news*. A strong late application lands in an empty room rather than a
   queue of forty.

**The strategic value is not the $4,000.** It is that an agency with a client
pipeline, who has failed to fill this role for two weeks, is a **recurring
revenue channel**. Win this and we are their platform bench.

## Why this shape wins

**Coverage: ~70%** — the lowest of the shortlist, and honestly so. Multi-tenancy,
four-role RBAC, per-seat plus metered billing, and audit are shipped. Genuinely
new: the onboarding/e-sign/IDV flow, the guardrailed coach, the ephemeral credit
pipeline, and the charge-on-success saga.

| Their requirement | Status |
|---|---|
| Multi-tenant Postgres, tenant isolation | shipped — and provable |
| RBAC across four user types | shipped |
| Consumer subs + per-seat + metered add-ons | shipped (billing + metering, live) |
| Audit trail | shipped |
| Console shell, org switching, settings, billing UI | shipped |
| E-signature + IDV onboarding | **this epic** |
| LLM coach with supervisor layer | **this epic** |
| Credit-data widget, never-persist-raw | **this epic** |
| Charge-on-success | **this epic** |
| White-label per-tenant branding | **this epic** |

## The demo

**Fiscally** → `fiscally.orun.dev`. **Four logins, one per user type**, and
**two tenants with visibly different branding on the same deployment** — which
is also the cheapest possible refutation of "did they just theme it once".

**Make the tenant-isolation proof the headline.** The demo ships a page that
performs a live cross-tenant read attempt, shows the 404, and links to the audit
entry it produced. They told us they will review isolation — hand them the
review, running, rather than a paragraph claiming it works.

Second-strongest artifact: the `coach.profile.derived` audit event that names
what was **not** stored. A hard constraint they can verify in their own audit
console beats any assurance in a proposal.

## Proposal angle

Answer their three screening questions **literally and in their order** — two
live URLs; Stripe per-seat/metered experience; AI-assisted development setup and
how you stay inside a spec. Then:

> *"Week 5 acceptance is the reason I'm a fit, not a risk: my platform layer —
> auth, four-role RBAC, per-seat + metered Stripe, audit — is already live.
> Week 1 is your design system on top of a running multi-tenant app, not
> scaffolding. You said isolation will be reviewed: here's the review, running —
> [URL], platform-admin login, the isolation page runs a live cross-tenant denial
> and links to the audit entry."*

The four artefacts, as with every proposal:

1. **The live product** — URL and four logins **in the proposal body**. One line
   on what to try first: *run the isolation proof.*
2. **The workspace** — a read-only Orun Cloud login at **app.orun.dev**: the
   component catalog, real deployment history, docs pinned to a commit. For an
   agency evaluating whether we can be their bench, the delivery record matters
   more than the product.
3. **The onboarding pack** — `docs/{overview,architecture,runbook}.md`, generated
   `ai/context/*`, plus `ONBOARDING.md` and `HANDOVER.md`/`EXIT.md`.
4. **The cover letter** — live URL in paragraph one; Upwork truncates previews at
   about two lines.

## Pricing

Take the **$4,000 as posted** for the MVP *if the scope is genuinely locked* —
the value here is the channel, not the margin. Attach a **retainer proposal** for
the maintenance tail and for their next client build, and price that properly.

**Before signing:** get the locked-scope baseline document in hand. Their entire
pitch is that the scope is locked, so asking for it is not a difficult
conversation — and twenty client-grade days against a five-week contractual date
has no room for a discovery phase.

## Risk

Middleman margin compression, and a week-5 contractual date on a fixed price.
Mitigate with a written scope-freeze clause and by treating the retainer, not the
MVP fee, as the actual commercial objective.
