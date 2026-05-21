# OIDC Federation Design

**Status:** Authoritative
**Repos affected:** infodancer/auth, infodancer/webauth, infodancer/oidclient, infodancer/herald, the websites (mjh, osg, sf, amy, hunterfamily, tf, …), and homelab (deployment). Any service that authenticates users.

---

## Invariants — read these before changing anything

We have re-derived and re-broken this since March. Three hard rules. If a change
violates one, the change is wrong, not the rule.

1. **auth-oidc is a leaf identity provider. It is NEVER a federation client.**
   Its only job is to authenticate users of *owned mail domains* against their
   *local passwd files*, and to expose that as standard OIDC + webfinger so others
   can discover and use it. It has no upstream, no "login with Google," no
   federation-client / upstream-callback code. A login at `auth.<domain>` is always
   checked against that domain's passwd file — full stop. (If you find upstream/RP
   code in infodancer/auth, it does not belong there.)

2. **webauth is the ONLY federation client, and it federates uniformly.**
   It is the single login UI and the only OIDC relying party in the stack. The user
   always **enters their email**; webauth resolves the domain to an upstream, runs the
   authorization-code flow as an RP, and maps the result to a webauth identity.

   Login is **email-driven, not button-driven.** There are no "Login with X" buttons by
   preference — the user types `alice@gmail.com` and webauth routes by domain, rather
   than presenting a wall of provider buttons. A visible button is a tolerated last
   resort, used only where a specific provider's flow genuinely cannot be initiated
   from the typed email alone.

   Resolving the email domain to an upstream uses one of two mechanisms, both feeding
   the **same** RP path:
   - **Autodiscovery (zero-config):** webfinger → OIDC discovery → RFC 7591 dynamic
     registration. This reaches *auth-hosted owned domains* and any federation-aware
     peer. Owned domains are not special-cased; they are just an upstream whose issuer
     happens to be `auth.<domain>`.
   - **Pre-configured special-case providers:** Gmail, GitHub, and similar consumer
     IdPs do **not** serve webfinger at their email domain and do **not** support
     dynamic registration, so they *cannot* be autodiscovered. Each is reached via a
     manually pre-registered OAuth/OIDC client (a one-time admin setup), still keyed by
     the typed email domain (`@gmail.com` → the configured Google client). This is a
     small, explicit, admin-managed set — not a return to a hardcoded provider list for
     every domain.

   (Case 3 — no OIDC and not a configured provider — falls through to webauth's local DB.)

3. **Websites are dumb relying parties.** They authenticate users either against
   their own local DB or by delegating to webauth via `oidclient`. They never talk to
   auth-oidc, Gmail, or any upstream directly, and they never implement federation.

Everything below elaborates these three rules.

## The Three Use Cases

This stack must handle three distinct authentication scenarios:

| Case | User | Backend | Who handles it |
|------|------|---------|---------------|
| 1 | `alice@infodancer.net` (owned domain) | passwd files via auth-oidc | **auth-oidc** |
| 2 | `alice@gmail.com` (external OIDC domain) | Google, GitHub, any OIDC provider | **webauth** (OIDC RP) |
| 3 | `alice@somedomain.com` (no OIDC support) | webauth's local SQLite | **webauth** (local backend) |

These cases are **not interchangeable** and must not be collapsed into one system.

---

## System Components

### infodancer/auth (auth-oidc)

**Role:** Standalone OIDC provider for owned infodancer mail domains.

- **Authoritative** for users whose domain's passwd file is managed by the infodancer mail stack.
- Exposed publicly at `https://auth.<domain>` (one per domain) via Traefik.
- Serves all standard OIDC endpoints: discovery, JWKS, authorize, token, userinfo, logout.
- Supports RFC 7591 dynamic client registration — **open, no bearer token required**.
- Host-based domain routing: `auth.infodancer.net` → serves the `infodancer.net` domain entry.

**Used by:** webauth (for case 1), and any third party who wants to implement "Login with infodancer.net".

### infodancer/webauth

**Role:** Auth broker for Herald and other infodancer web applications.

- **Not authoritative** over user identity. Credential truth lives in backends.
- Handles all three use cases through a single login UI.
- Acts as an OIDC relying party (RP) to upstream providers for cases 1 and 2.
- Falls through to local SQLite for case 3.
- Issues its own JWT to downstream apps (Herald, etc.) after successful auth.

**Used by:** Herald, and any future infodancer web app that needs user authentication.

---

## Authentication Flow

### Webauth login sequence

```
User submits email at webauth login form
         │
         ▼
Extract domain from email address
         │
         ▼
Probe https://<domain>/.well-known/openid-configuration
         │
    ┌────┴────┐
  Found    Not found
    │          │
    ▼          ▼
RFC 7591   Fall through
register   to local
+ OIDC     SQLite
redirect   credentials
    │
    ▼
User authenticates at upstream OIDC provider
    │
    ▼
Callback: exchange code, verify token
    │
    ▼
Provision/update local user record (sub UUID, roles)
    │
    ▼
Issue webauth JWT to Herald
```

### Discovery probe rules

- HTTPS only — never probe HTTP endpoints for discovery.
- Set a short timeout (5s) to avoid blocking the login form on slow/dead domains.
- Cache the discovery doc per domain (1-hour TTL already implemented in `upstreamoidc/`).
- If the discovery doc is present but `registration_endpoint` is absent, the domain
  supports OIDC but not dynamic registration — fall through to local SQLite.

### Dynamic registration (RFC 7591)

- Webauth POSTs to `registration_endpoint` with its callback URI.
- **No bearer token is required** for auth-oidc or any correctly configured open provider.
- The client_id returned is stable (auth-oidc derives it via HMAC from inputs) — re-registration
  after a server restart returns the same client_id.
- If registration fails (4xx), fall through to local SQLite.

---

## Security Model

### What controls actually matter

**1. Exact redirect_uri matching at authorization time**
This is the primary OIDC security control. A registered client with
`redirect_uri: https://app.example.com/callback` can only receive authorization codes
at that exact URI. Auth-oidc enforces this in `validRedirectURI`. This prevents
code injection and open redirect attacks.

**2. PKCE (S256) required**
Prevents authorization code interception attacks. Plain PKCE is rejected.
No client secrets are issued for dynamic clients — PKCE is the only proof of possession.

**3. HTTPS-only redirect URIs**
Auth-oidc requires HTTPS for all redirect URIs except `localhost`/`127.0.0.1`
(allowed for local development). This prevents credential interception over plaintext.

**4. Exact-match token validation**
Webauth verifies the upstream ID token signature against the upstream JWKS and
validates the issuer against the discovered issuer URL.

### What does NOT need to be a security control

**Registration bearer tokens (DO NOT ADD)**
Auth-oidc previously had a `RegistrationToken` config field that required a bearer
token to call `/register`. This was removed. It is the wrong layer:

- It prevented legitimate third-party federation ("Login with infodancer.net")
- It prevented webauth from autodiscovering and autoregistering
- It added no meaningful security because redirect_uri matching is the real gate
- A rogue client that somehow registers still cannot receive tokens unless a real
  user intentionally authenticates with it

Do not add registration tokens back. If someone proposes it, re-read this section.

**Per-domain redirect URI allowlisting at registration time (DO NOT ADD)**
Auth-oidc previously validated that redirect_uri hosts were subdomains of the mail
domain list at registration time. This was removed. It is the wrong layer:

- It prevented third-party sites from federating (their callback is on their own domain)
- The real protection is exact-match at authorization time, not at registration time
- Domain restriction at registration is security theater given exact-match enforcement

Do not add redirect URI domain allowlisting back at registration time. The check
at authorization time (exact match against registered URIs) is sufficient.

---

## Configuration Reference

### auth-oidc (infodancer/auth)

```toml
[server]
listen           = ":8080"
data_dir         = "/var/lib/auth-oidc"
domain_data_path = "/opt/infodancer/domains"
jwt_ttl_sec      = 3600
session_ttl_sec  = 604800
# No registration_token field — registration is open
```

No per-client or per-domain configuration is needed. The trusted domain list comes
from the directories present in `domain_data_path`.

### webauth (infodancer/webauth)

Most upstreams are not pre-configured: webauth autodiscovers them at login time by
probing the email domain (webfinger → OIDC discovery → RFC 7591). Two things *are*
admin-configured:

- **Special-case providers** that don't support autodiscovery (Gmail, GitHub) are
  reached via manually pre-registered clients, keyed by email domain.
- A **per-tenant federation mode** governs which domains may be used and whether
  accounts may be created: `auto` (probe OIDC; any domain may sign up), `allowlist`
  (only listed domains), `local` (no OIDC; local accounts only), `locked` (no account
  creation — users provisioned out of band).

```toml
[server]
listen   = ":8080"
base_url = "https://webauth.infodancer.net"

[database]
url = "postgres://webauth:…@postgres:5432/webauth?sslmode=disable"

[smtp]
# ... email delivery for verification/reset flows
```

---

## Adding a New Owned Domain

1. Add the domain to the infodancer mail stack (Ansible `mail_domains` list).
2. Provision DNS: `auth.<domain>` → Traefik on docker-web.
3. The auth-oidc config template auto-generates the Traefik route for the new domain.
4. No changes needed in webauth — it will autodiscover `auth.<domain>` on first login.

---

## History / Why We Went Around in Circles

This design was arrived at after multiple iterations. The recurring mistake was adding
security controls at the wrong layer (registration) that belonged at a different layer
(authorization). Each time, the registration controls were removed because they broke
autodiscovery or third-party federation, and the correct controls (exact redirect_uri
matching + PKCE) were confirmed to be sufficient.

**The rule:** if a proposed change gates something at registration time based on who
the client is, stop and ask whether the authorization-time controls already cover
the threat. They almost certainly do.
