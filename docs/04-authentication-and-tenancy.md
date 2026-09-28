# 04 — Authentication, Credentials & Tenancy

Two separate authentication problems, routinely conflated:

1. **Inbound** — how a caller proves it may use the ConConnect API.
2. **Outbound** — how ConConnect proves to a CRM that it may act for a nonprofit.

## Inbound

| Mode | For | Mechanism |
|---|---|---|
| Service API key | ConConnect's own backend services | `Authorization: Bearer ccn_sk_…`, scoped to one tenant, hashed at rest (argon2id), prefix-searchable for rotation |
| OAuth 2.0 client credentials | Third-party partners integrating with us | Short-lived JWT access tokens, scopes per entity+operation (`gifts:write`, `constituents:read`) |
| End-user token | Interactive UIs acting as a signed-in staff member | JWT carrying `tenant_id` + `user_id`; the tenant claim is authoritative and never taken from the request body or a query parameter |

Rules:

- Every request resolves to exactly one `tenant_id` **from the credential**. A
  request that names a tenant in its body gets that value ignored.
- Scopes are enforced at the gateway *and* re-checked in the core. A missing check
  in one layer should not be exploitable.
- Keys are rotatable without downtime: two active keys per tenant, overlapping
  validity, last-used timestamp recorded so dead keys can be retired confidently.

## Outbound: one flow per provider, and they are all different

This is the part that resists abstraction. The connect experience genuinely differs
per provider, and pretending otherwise produces a broken onboarding UI.

### Salesforce — OAuth 2.0, self-service

- **Web server (authorization code + PKCE)** for user-initiated connect. Admin
  clicks connect, authorises our Connected App, we store the refresh token.
- **JWT bearer** for unattended server-to-server, using a certificate uploaded to
  the Connected App. No refresh token, no user session to expire — preferable for
  long-running sync where possible.
- Refresh tokens are long-lived but revocable by the admin, by session-policy
  changes, or by password reset. Treat revocation as expected, not exceptional:
  surface a clear "reconnect required" state rather than retrying forever.
- Must store the returned `instance_url` per connection. Hardcoding
  `login.salesforce.com` breaks sandboxes and My Domain orgs.
- **Onboarding: minutes, self-service.** Good.

### Blackbaud — OAuth 2.0 + a separate subscription key

- Authorization-code flow. Token endpoint authenticates with **HTTP Basic using
  `base64(client_id:client_secret)`**.
- Access tokens last **~60 minutes**; refresh tokens rotate. **VERIFY** the exact
  refresh-token lifetime and whether a returned refresh token invalidates its
  predecessor — if it does, a concurrent refresh race will log the tenant out, so
  refresh must be serialised behind a per-connection lock.
- **Every API call additionally requires a `Bb-Api-Subscription-Key` header.** This
  is *our* developer-account subscription key, not the tenant's — it identifies the
  application, and it is the thing the rate limit is counted against.
- Requires the nonprofit to have the relevant SKY API products enabled on their
  subscription. `describeInstance` should detect this and report it clearly;
  "unauthorized" on a constituent read may mean "not subscribed", not "bad token".
- **Onboarding: self-service OAuth, but gated on their subscription.**

### Bonterra EveryAction / NGP VAN — support-issued API key

- **HTTP Basic**, where the username is the application name and the password is
  the API key, conventionally suffixed with a mode indicator (`|0` vs `|1`
  selecting the voter-file vs campaign/CRM context). **VERIFY** the exact format
  and mode semantics against the API reference before coding.
- Base URL `https://api.securevan.com/v4/`.
- **Keys are requested through the vendor's support system**, scoped to a single
  database, and delivered to a named contact via a one-time link with a four-digit
  key reference. There is no OAuth, no self-service, and no programmatic issuance.
- **Onboarding consequence: days to weeks of lead time, with a human in the loop,
  per customer.** This is a product and go-to-market fact, not just an engineering
  one. The connect UI must model a *pending* connection state where the customer
  has requested a key and is waiting, and onboarding docs must tell them to start
  the request early.
- Because keys are long-lived and non-rotating-by-default, compromise is worse and
  rotation is manual. Store with the same care as a password, alert on failures,
  and provide an explicit "replace key" flow.

### Bonterra Apricot / Impact Management, and ETO

- Apricot API access is **licence-gated to Enterprise and Pro tiers**. A customer
  on a lower tier cannot be integrated at any price of engineering effort. The
  connect flow must detect and explain this rather than failing obscurely.
- ETO authenticates via a REST security endpoint taking a **user email and
  password** (`POST /API/Security.svc/SSOAuthenticate/`) and requires the API
  feature to be enabled on the site. **VERIFY** current mechanism —
  password-based service accounts are a security posture we should push back on,
  and there may be a newer OAuth path via the Bonterra platform portal.
- `developer.bonterra.network` documents an OAuth 2.0 platform API. Whether it
  fronts Apricot/EveryAction data or is a separate newer surface **could not be
  determined** — the portal was unreachable from the authoring environment. This is
  the single biggest documentation gap; see Q1 in [09](09-open-questions.md).

## Credential storage

```
connections
  id, tenant_id, provider, display_name,
  status,                     -- pending | active | needs_reconnect | disabled | error
  instance_url,               -- Salesforce; null elsewhere
  instance_profile jsonb,     -- cached describeInstance output
  credential_ref,             -- pointer into the secrets manager, NOT the secret
  scopes_granted text[],
  connected_by_user_ref,
  expires_at,                 -- access token expiry, for proactive refresh
  last_health_check_at, last_health_status,
  created_at, updated_at
```

Rules:

- **Secrets never live in Postgres.** The database holds a `credential_ref`; the
  material lives in the secrets manager (AWS Secrets Manager / Vault) with
  per-tenant paths and an envelope-encryption key per tenant. A dump of the
  application database must not yield a single working CRM credential.
- **Refresh is serialised** per connection with a distributed lock, and the result
  is written before the old token is discarded. Concurrent refresh is the classic
  way to invalidate a rotating refresh token and lock a customer out.
- **Refresh proactively** at 75% of token lifetime; also refresh reactively on a
  single 401 and retry the call exactly once. A second 401 transitions the
  connection to `needs_reconnect` and raises an alert — it does not retry forever.
- **Never log credentials, or any header containing them.** The HTTP client
  redacts `Authorization`, `Bb-Api-Subscription-Key`, and anything matching known
  key prefixes, at the transport layer so that no call site can leak them by
  accident.
- **Connection status is customer-visible.** A silently dead connection that stops
  syncing donations is discovered at the worst possible moment — usually during a
  year-end appeal. Status changes fire notifications.

## Tenancy isolation

- Every table carries `tenant_id`; every query is scoped by it. Enforce with
  PostgreSQL row-level security in addition to application filters, so a missing
  `WHERE` clause cannot leak across tenants.
- All identifiers in API responses are ConConnect IDs. Provider IDs appear only
  inside `sources[]`, and only for connections belonging to that tenant.
- Background jobs carry the tenant context explicitly; there is no ambient or
  "current tenant" global. A worker processing tenant A's queue cannot construct a
  client for tenant B.
- Provider quota consumption is accounted per tenant even where the provider limit
  is per-application, because that accounting is what makes fair-share scheduling
  and abuse detection possible. See [05](05-sync-engine.md#rate-limit-governance).

## Multiple connections per tenant

Expected, not an edge case: a nonprofit may run Raiser's Edge for major gifts and
EveryAction for advocacy and grassroots fundraising, with overlapping people in both.

- Each connection syncs independently, with its own cursor and its own schedule.
- A canonical record may accumulate several entries in `sources[]`.
- Writes must name a target: `POST /v1/gifts?connection_id=conn_…`. Where the
  tenant has exactly one connection capable of the operation, that one is the
  default; where there are several, omitting `connection_id` is a 400, because
  guessing which donor database receives a gift is not a risk worth taking.
- Linking policy (when are two provider records the same person?) is deliberately
  conservative and documented in [05](05-sync-engine.md#identity-and-linking).
