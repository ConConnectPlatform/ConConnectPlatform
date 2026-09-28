# Adding a New Provider

The test of this architecture is whether CRM #4 is a contained piece of work. This is
the checklist that keeps it so. If a step here requires changing the core or the
public API, that is a signal the abstraction is wrong and should be fixed rather than
worked around.

## 1. Research (before any code)

Produce `docs/providers/<provider>.md` answering:

- **Auth**: which flow? Self-service or human-in-the-loop? Token lifetimes? Does
  refresh rotate and invalidate? What headers does every call need?
- **Data model**: what is a person, a gift, a pledge, a designation? Does it separate
  received from promised money? Are there split gifts, soft credits, tributes?
- **Change detection**: webhooks, delta query, export job, or nothing? Do child
  changes propagate to the parent? Are deletes detectable? What is the resumable
  cursor?
- **Limits**: what is the number, and is it scoped per application, per tenant, or per
  user? **This question determines architecture, not just configuration.**
- **Write semantics**: upsert by external id? Transactional batch? Optimistic
  concurrency? Maximum batch size?
- **Commercial gates**: which licence tiers expose the API at all? What is the
  onboarding lead time?
- **Sandbox**: how do we get one, and how long does it take? Start this immediately —
  it is usually the longest lead-time item.

Every claim gets a citation or a **VERIFY** marker. No exceptions — an unmarked guess
in a provider doc becomes a production incident.

## 2. Capability matrix

Fill in the matrix from [03](../03-provider-adapter-contract.md) **honestly**. An
overstated capability is worse than an absent one: it produces silent data loss where
`capability_unsupported` would have produced a clear error.

Be specific about `partial`. "Partial" with no explanation is not usable by a caller;
name which fields and which operations.

## 3. Field mapping

Add the provider's column to [07](../07-field-mapping.md). For every canonical field,
record: the provider field, the direction (read/write/both), any transform, and
whether it is lossy. Mark unmappable canonical fields explicitly — that list feeds
`unsupportedFields` in the capability matrix, so the two documents must agree.

## 4. Implement the adapter

- Implement `CrmAdapter`. No core changes should be required.
- **All egress through `ctx.http`.** Enforced by lint rule and review. An adapter that
  bypasses it can exhaust a shared rate limit for every tenant
  ([05](../05-sync-engine.md#rate-limit-governance)).
- Map every provider error to a canonical code, preserving the original in
  `provider_error`.
- Implement `describeInstance` properly: detect schema variants, enabled features,
  licence tier, and effective field permissions. A capability matrix derived from
  documentation rather than from the live instance will be wrong for some customers.
- No provider-specific logic anywhere outside the adapter. If something cannot be
  expressed within the adapter boundary, raise it as a design issue rather than
  leaking a special case into the core.

## 5. Test

Per [03](../03-provider-adapter-contract.md#adapter-testing-requirements):

- [ ] Shared contract suite passes
- [ ] Every claimed capability has a passing round-trip test
- [ ] Recorded fixtures from **real** scrubbed responses, not hand-written
- [ ] Error paths: 401→refresh→retry-once, 403, 404, 409, 429 + `Retry-After`, 5xx
- [ ] All egress routed through `ctx.http` (asserted)
- [ ] Rate-limit behaviour under a saturated bucket
- [ ] Idempotency: replayed write does not duplicate
- [ ] Sandbox smoke checklist executed and recorded

## 6. Operational readiness

- [ ] Dashboards include the provider
- [ ] Alerts wired: `needs_reconnect`, quota, delta lag, dead letters
- [ ] Runbook: how to diagnose a stuck sync, how to re-authorise, how to replay
- [ ] Connect flow documented for customers, including lead times and licence
      requirements
- [ ] Support briefed on the failure modes specific to this provider

## 7. Launch

Behind a feature flag, one friendly customer first, with reconciliation drift watched
closely for the first full cycle. Sync bugs are not evenly distributed in time — many
only appear at month-end, at fiscal-year boundaries, or during a year-end appeal when
volume spikes. Plan to still be watching then.

## Design smells

If adding a provider requires any of the following, stop and fix the abstraction:

- Changing the canonical model to accommodate one provider's quirk (use `custom`, or
  add the concept properly if it is genuinely general)
- A `switch` on provider id in the core
- A new public API endpoint named after the provider
- Bypassing the provider gateway "just for this one call"
- A capability marked `full` with a comment explaining when it does not work
