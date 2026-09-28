# 03 — Provider Adapter Contract

Every provider implements this interface. Shown as TypeScript for precision; the
shape holds whichever language we pick (Q4 in [09](09-open-questions.md)).

## The interface

```ts
interface CrmAdapter {
  readonly provider: ProviderId;
  readonly capabilities: CapabilityMatrix;

  // --- lifecycle -------------------------------------------------------
  /** Exchange whatever the connect flow produced for stored credentials. */
  authorize(input: AuthorizeInput): Promise<ConnectionCredentials>;
  /** Refresh before expiry, or after a 401. Idempotent. */
  refresh(creds: ConnectionCredentials): Promise<ConnectionCredentials>;
  /** Cheap authenticated call proving the connection works. Used on connect and hourly. */
  healthCheck(ctx: Ctx): Promise<HealthStatus>;
  /**
   * Inspect the live instance and report what this specific tenant's setup
   * supports. Salesforce needs this to detect NPSP vs Nonprofit Cloud; Blackbaud
   * to detect which API products are subscribed.
   */
  describeInstance(ctx: Ctx): Promise<InstanceProfile>;

  // --- reads -----------------------------------------------------------
  get<E extends EntityName>(ctx: Ctx, entity: E, providerId: string): Promise<Canonical<E> | null>;
  list<E extends EntityName>(ctx: Ctx, entity: E, q: ListQuery): Promise<Page<Canonical<E>>>;
  search<E extends EntityName>(ctx: Ctx, entity: E, q: SearchQuery): Promise<Page<Canonical<E>>>;

  // --- writes ----------------------------------------------------------
  create<E extends EntityName>(ctx: Ctx, entity: E, data: Canonical<E>, idem: string): Promise<WriteResult>;
  update<E extends EntityName>(ctx: Ctx, entity: E, providerId: string,
                               patch: Partial<Canonical<E>>, opts: WriteOpts): Promise<WriteResult>;
  delete<E extends EntityName>(ctx: Ctx, entity: E, providerId: string): Promise<WriteResult>;

  // --- change detection ------------------------------------------------
  /**
   * Return changes since `cursor`. The cursor is an opaque provider-specific
   * token that the core persists verbatim and never interprets. Blackbaud returns
   * a sort_token; EveryAction an export job id; Salesforce a replay id or a
   * SystemModstamp watermark.
   */
  pullChanges(ctx: Ctx, entity: EntityName, cursor: SyncCursor | null): Promise<ChangeBatch>;

  // --- webhooks (optional) ---------------------------------------------
  webhooks?: {
    subscribe(ctx: Ctx, events: WebhookEvent[], deliveryUrl: string): Promise<WebhookSubscription>;
    unsubscribe(ctx: Ctx, subscriptionId: string): Promise<void>;
    /** MUST verify the signature. Returning false discards the delivery. */
    verify(rawBody: Buffer, headers: Record<string, string>, secret: string): boolean;
    /** Normalise a verified delivery into the same shape as pullChanges output. */
    normalize(rawBody: Buffer, headers: Record<string, string>): ChangeBatch;
  };

  // --- reference data ---------------------------------------------------
  /** Funds, campaigns, appeals, gift types, picklist values. Cached aggressively. */
  listReferenceData(ctx: Ctx, kind: ReferenceKind): Promise<ReferenceItem[]>;
}
```

### `Ctx`

```ts
interface Ctx {
  tenantId: string;
  connectionId: string;
  credentials: ConnectionCredentials;   // already refreshed by the core
  instance: InstanceProfile;            // cached result of describeInstance
  http: ThrottledHttpClient;            // MUST be used for all egress
  logger: Logger;                       // request-scoped, correlation id attached
  signal: AbortSignal;
}
```

Adapters **must** issue every outbound request through `ctx.http`. That is what
enforces the global fair-share throttle, backoff, circuit breaking, and audit
logging. An adapter that reaches for a raw HTTP client or a vendor SDK's own
transport bypasses the rate-limit governor and can exhaust a shared provider
budget for every tenant. This is the single most important rule in the codebase
and should be enforced by lint rule and code review, not just convention.

Where a vendor SDK is genuinely worth using (Salesforce's is), it must be
configured with our HTTP transport injected.

## Capability matrix

Declarative, introspectable, and returned to API callers. This is how we keep the
"honest capabilities" promise from [00](00-overview.md).

```ts
type Support = "full" | "partial" | "none";

interface CapabilityMatrix {
  entities: Record<EntityName, {
    read: Support;
    create: Support;
    update: Support;
    delete: Support;
    /** Canonical fields this provider cannot represent at all. */
    unsupportedFields: string[];
    /** Canonical fields that are readable but not writable. */
    readOnlyFields: string[];
    /** Fields the provider computes; writes are ignored. */
    derivedFields: string[];
  }>;
  changeDetection: {
    mechanism: "webhook" | "delta_poll" | "export_job" | "full_scan";
    granularity: "field" | "record";
    /** Does a child change bump the parent's modified timestamp? */
    childChangesPropagate: boolean;
    deletesDetectable: boolean;
    minPollIntervalSeconds: number;
  };
  writes: {
    transactionalBatch: boolean;
    upsertByExternalId: boolean;
    maxBatchSize: number;
    optimisticConcurrency: boolean;   // does it honour If-Match / version checks?
  };
  limits: {
    scope: "per_application" | "per_tenant" | "per_user";
    requestsPerWindow: number;
    windowSeconds: number;
    notes: string;
  };
}
```

### Current matrix (summary)

Full detail per provider in [`providers/`](providers/). Every cell here should be
treated as **VERIFY** until confirmed against a sandbox.

| | Salesforce (NPC) | Salesforce (NPSP) | Blackbaud RE NXT | Bonterra EveryAction | Bonterra Apricot |
|---|---|---|---|---|---|
| Constituent R/W | full / full | full / full | full / full | full / partial | full / partial |
| Gift R/W | full / full | partial¹ / partial¹ | full / full | full / partial | none² |
| Commitment R/W | full / full | partial¹ / partial¹ | partial³ | partial⁴ | none² |
| Soft credits | full | full | full | partial⁴ | none |
| Split gifts | full | partial¹ | full | **VERIFY** | none |
| Designation hierarchy | full | partial | full (fund/campaign/appeal/package) | partial | none |
| Change detection | Pub/Sub CDC | Pub/Sub CDC | `sort_token` poll + webhook (beta) | export job | **VERIFY** |
| Child changes propagate | yes (per-object events) | yes | **no** ⚠ | **VERIFY** | **VERIFY** |
| Transactional batch | yes (Composite) | yes | no | no | no |
| Upsert by external id | yes | yes | no | no | **VERIFY** |
| Rate-limit scope | per tenant org | per tenant org | **per application** ⚠ | per API key | **VERIFY** |

¹ NPSP overloads `Opportunity`; pledges, payments and recurring gifts live in
add-on objects with their own trigger logic. Writing them directly risks bypassing
NPSP's rollup automation. See [`providers/salesforce.md`](providers/salesforce.md).

² Apricot is case management, not fundraising. It has no native gift object to map
to, and its API is licence-gated. See [`providers/bonterra.md`](providers/bonterra.md).

³ Raiser's Edge recurring gifts and pledges exist but are modelled differently from
our Commitment; mapping fidelity needs sandbox confirmation.

⁴ EveryAction has contributions and recurring commitments; write support and
soft-credit fidelity need confirmation against the live API reference.

⚠ = a property that materially changes the architecture, not just the mapping.

## Capability enforcement

The core checks capabilities **before** calling the adapter, and fails with a
precise, actionable error rather than a generic 400:

```jsonc
// POST /v1/gifts  with soft_credits, on a provider that cannot store them
HTTP/1.1 422 Unprocessable Entity
{
  "error": {
    "code": "capability_unsupported",
    "message": "Provider 'bonterra_apricot' cannot represent Gift.soft_credits.",
    "provider": "bonterra_apricot",
    "entity": "gift",
    "unsupported_fields": ["soft_credits"],
    "remedy": "Omit the field, or set X-ConConnect-On-Unsupported: drop to discard it explicitly.",
    "docs": "https://docs.conconnect.example/capabilities/bonterra_apricot#gift"
  }
}
```

Callers who genuinely want lossy behaviour opt in per request:

| `X-ConConnect-On-Unsupported` | Behaviour |
|---|---|
| `fail` (default) | 422 as above. Nothing is written. |
| `drop` | Unsupported fields discarded; response includes a `warnings[]` array naming each dropped field. |
| `custom` | Attempt to store in the provider's custom-field area if the connection has a mapping configured; otherwise 422. |

Defaulting to `fail` is deliberate. The alternative — silently dropping a
`do_not_solicit` flag or an anonymity flag — is a compliance incident, not a bug.

## Adapter testing requirements

Non-negotiable for any adapter to be considered done:

1. **Contract test suite.** One shared suite, run against every adapter, asserting
   canonical round-trips: `create → get → update → get → pullChanges` returns
   consistent shapes, and that every capability the adapter *claims* actually works.
   An adapter declaring `softCredits: full` and failing the soft-credit round-trip
   fails the build.
2. **Recorded fixtures.** Real (scrubbed) provider responses as golden files.
   Hand-written fixtures encode our assumptions rather than the provider's
   behaviour, which is exactly the thing that bites in production.
3. **Error-path coverage.** 401 (expired token → refresh → retry once), 403, 404,
   409/version conflict, 429 with `Retry-After`, 5xx, and provider-specific
   validation errors mapped to canonical codes.
4. **Rate-limit behaviour.** A test that the adapter routes all egress through
   `ctx.http` — assert zero direct socket use.
5. **Sandbox smoke test.** A manual, documented checklist run against a real
   sandbox before enabling the provider for any tenant. Recorded fixtures cannot
   catch a vendor changing behaviour.
