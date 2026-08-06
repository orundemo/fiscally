# fiscally — Risks and open questions

## Risks

### Product and technical

**A week-5 contractual date on a fixed price, with twenty client-grade days of
work.** There is no slack. A single unscoped change request eats the buffer that
does not exist. *Mitigation:* get the "locked scope baseline" document **in hand
before signing** — their whole pitch is that the scope is locked, so ask for it
and hold them to it; a written scope-freeze clause; and the milestone structure
in the plan so a slip is visible in week two rather than week five.

**The isolation review is pass/fail and they will actually run it.** A reviewer
who finds one cross-tenant leak will not look at anything else. *Mitigation:*
FS1 lands before features; the denial suite enumerates every route rather than
sampling; the second scoping grain is enforced in the policy path and the
repository signature, not in handlers; and the isolation proof page lets them
run the review themselves rather than trusting a claim.

**"Never store X" degrades under deadline pressure.** Week four, a bug needs
diagnosing, and someone logs the payload "just temporarily". *Mitigation:* the
schema has nowhere to put it, so storing it requires a migration and therefore a
review; the memory/log test in FS4 fails the build if the payload escapes; and
the `dropped:[…]` audit event makes the property continuously observable rather
than a one-time assertion.

**A hallucinated financial claim is a regulatory problem, not a bug report.**
This is consumer finance; "you should apply for X" or an invented number is a
different class of failure from a bad marketing email. *Mitigation:* the model
receives bands rather than values, so it has no numbers to leak; the supervisor
blocks regulated-advice patterns; action-plan items reference reviewed templates
instead of free prose; blocked generations store nothing and are not billed.
Even so — see open question 1 on who signs off the template set.

**Charge-on-success is where money bugs live.** Double charges destroy trust
with the operator *and* the consumer; missed charges destroy the unit economics
quietly. *Mitigation:* the durable ledger with a deterministic unique
idempotency key, the conditional `pending → charging` claim so only one attempt
proceeds, the same key passed to the provider, and concurrency tests that drive
duplicate successes in parallel.

**Two external providers on the critical path of signup.** An e-sign or IDV
outage stalls every new consumer. *Mitigation:* the adapter seam means a second
provider is configuration; applications stay resumable across an outage rather
than failing; provider callbacks are treated as hints with authoritative re-read,
so a provider's own retry storm cannot corrupt state.

**The credit-data widget is a third party inside our page.** Its payload
provenance must be verified or the whole ephemeral pipeline is theatre.
*Mitigation:* signature/provenance verification before derivation; the digest
recorded so a disputed derivation can be traced to a specific payload without
retaining it.

**Two workers is more surface than one.** Deliberate (see `design.md` §2), but it
doubles the wiring, the fixtures, and the deployment units. *Mitigation:* both
follow the same anatomy as existing workers, so nothing about the shape is novel
to a reviewer or to us.

### Delivery and commercial

**Middleman margin compression.** We deliver under the partner agency's brand;
they keep the client relationship and PM. Our $4,000 is a slice of what their
client pays. *Mitigation:* take the MVP at their number — the value is the
channel, not the invoice — and attach a retainer proposal for the maintenance
tail and their next client build.

**Delivering under someone else's brand means no public reference.** We cannot
show this work in the next proposal. *Mitigation:* negotiate a named reference
or a case-study right at contract time, even if anonymised; and note that the
demo (Fiscally, ours) *is* showable regardless.

**They have been burned by a vibe-coded submission.** That is why isolation
"will be reviewed" and why the constraints are phrased as hard. It also means
scepticism is the default posture and evidence is the only currency.
*Mitigation:* answer their three screening questions literally and in order; lead
with live URLs because they said videos will not do; make the isolation proof
clickable.

**Payment and channel risk.** A dev shop subcontracting can also stop
subcontracting. *Mitigation:* milestone escrow; small first milestone; do not
build the retainer assumption into the pricing of the MVP.

**Docs that contradict the product.** The known failure mode from the reference
build. *Mitigation:* FS10 re-runs `08-docs` after the domain lands, and the
overview is read fresh before any workspace link is shared.

**Repo hygiene.** `.orun/` is generated, gitignored, and can reach hundreds of
megabytes — never in a client clone. Composition contracts stay pinned to an OCI
tag.

## Open questions

| # | Question | Owner | Needed by |
|---|----------|-------|-----------|
| 1 | **Who signs off the reviewed rationale-template set and the supervisor's regulated-advice rules?** This is a compliance decision, not an engineering one, and it gates FS5. The partner agency's client almost certainly has a view. | client / compliance | FS5 |
| 2 | Which model provider — a Workers-native binding or an external API with the key in environment configuration? Affects latency, per-turn cost, and the exit story. | eng | FS5 |
| 3 | Which e-sign and IDV providers, and do their contracts sit with us or with the end client? Strong preference: the end client, so credentials and retention obligations are theirs. | client | FS3 |
| 4 | What exactly triggers charge-on-success? `onboarding_approved` is the obvious one; is there a second (plan milestone reached)? Pricing depends on the answer. | client | FS6 |
| 5 | Is the consumer subscription billed to the consumer, or to the operator on the consumer's behalf? B2B2C makes this genuinely ambiguous and it changes the billing wiring. | client | FS6 |
| 6 | Profile staleness horizon — how old is too old before a derived profile stops being usable for coaching? | product | FS4 |
| 7 | Does branding include a custom domain per operator, or only theming? Custom domains add a Terraform surface per tenant. | client | FS7 |
| 8 | Their "per-view component map" — do we receive component names we must match exactly, or a visual spec we implement? This materially changes the FS9 estimate. | client | FS9 |
| 9 | Data retention and deletion: what happens to `coach` history and `onboarding` records on consumer deletion? A right-to-erasure path is likely required and is not currently scoped. | client / compliance | FS10 (client-grade) |
| 10 | Is their existing "finished skeleton" something we are expected to migrate from, or purely a spec artifact? The epic assumes the latter; assuming wrong is a week. | client | before signing |
