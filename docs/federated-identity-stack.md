# Federated Identity & Email Stack
*Speculative architecture document — vision, not roadmap*
*Last updated: 2026-03-01*

> This document describes the long-term identity and deployment vision for the infodancer stack.
> For the future messaging protocol design, see [next-gen-messaging-protocol.md](next-gen-messaging-protocol.md).
> For the current mail security and process model, see [mail-security-model.md](mail-security-model.md).

---

## The Vision

A self-hosted, turnkey stack where pulling a set of Docker images and logging in once gives you:

- Hosted email domains with competently managed PKI
- OIDC identity for those domains, auto-discoverable by standard clients
- Web authentication for sites you build, with single sign-on across all of them
- Federation with any other instance of this stack (or any standards-compliant system) — automatically, via DNS and OIDC discovery, no registration or approval required

The core principle: **your email address is your identity, and the domain in your address is the authority for that identity.** This is what OAuth and OIDC were designed to enable. We implement it properly.

---

## Why This Matters

The current state:

- Identity is effectively controlled by three companies (Google, Microsoft, Apple)
- "Login with X" outsources authentication to whoever has the biggest OAuth button
- Self-hosted email is technically possible but operationally punishing (PKI, DKIM, DMARC, deliverability)
- Running your own identity provider requires Keycloak or Zitadel — serious infrastructure for a non-trivial operational burden
- Federated identity exists as a standard but not as a usable product for small operators

What we're building changes the unit of operation: a single `docker compose up` that produces a fully functional, standards-compliant node in a decentralized identity and email network.

---

## Stack Components

```
┌─────────────────────────────────────────────────────────┐
│                      Admin UI                           │
│         (single pane of glass across all services)      │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼──────────────────┐
        │                │                  │
┌───────▼──────┐  ┌──────▼──────┐  ┌───────▼──────┐
│  webauth     │  │infodancer/  │  │   PKI mgr    │
│  (OIDC IdP + │  │  auth       │  │  (ACME/LE,   │
│   web broker)│  │  (domain    │  │   DKIM, key  │
│              │  │   auth +    │  │   rotation)  │
│              │  │   OIDC)     │  │              │
└───────┬──────┘  └──────┬──────┘  └───────┬──────┘
        │                │                  │
        │         ┌──────┴──────┐           │
        │         │   smtpd     │           │
        │         │   pop3d     │           │
        │         │   imapd     │           │
        │         │   msgstore  │           │
        │         └─────────────┘           │
        │                                   │
        └──────────────DNS ────────────────-┘
                  (SPF, DKIM, DMARC,
                   MX, WebFinger,
                   OIDC discovery hints)
```

### Components and Status

| Component | Purpose | Status |
|---|---|---|
| **smtpd** | SMTP receive + relay | exists |
| **pop3d** | POP3 mail retrieval | exists |
| **imapd** | IMAP mail access | exists |
| **msgstore** | Encrypted message storage | exists |
| **infodancer/auth** | Domain credential auth | exists |
| **infodancer/auth OIDC** | OIDC server for managed domains | planned |
| **webauth** | OIDC identity broker + web SSO | planned |
| **PKI manager** | Cert + key lifecycle | not started |
| **Admin UI** | Unified management interface | not started |
| **Docker stack** | Turnkey deployment | not started |

---

## The Identity Model

A user's identity is `localpart@domain`. The domain is the authority. This resolves naturally:

- `matthew@infodancer.net` → infodancer/auth's OIDC endpoint is authoritative
- `alice@company.com` → company.com's OIDC endpoint if it publishes one, local password fallback otherwise
- `bob@gmail.com` → Google's OIDC endpoint (they're just another domain)

No hardcoded provider list. No "login with X" buttons. One field: email address.

Federation is automatic via standard OIDC discovery (`/.well-known/openid-configuration`) and WebFinger (`/.well-known/webfinger`). Any two instances of this stack that can reach each other over HTTPS federate without configuration.

---

## PKI Management

This is the operationally hard part of self-hosted email and identity. The PKI manager handles:

### TLS Certificates
- ACME/Let's Encrypt for all domains and subdomains
- Auto-renewal before expiry
- Certificate provisioned at domain onboarding, renewed silently

### DKIM Keys
- Per-domain Ed25519 (or RSA-2048 minimum) DKIM signing keys
- Generated at domain onboarding
- Published to DNS automatically (if DNS is managed by the stack) or with guided instructions
- Rotation on a configurable schedule with overlap period (old key valid during DNS TTL propagation)

### OIDC Signing Keys
- Per-domain RS256 keypairs for JWT signing
- Published via JWKS endpoints
- Rotation with overlap (new key in JWKS before old key retired from signing)

### Audit Trail
- Key generation, rotation, and retirement events logged
- Immutable log for compliance

---

## Domain Onboarding Flow

What "add a domain" looks like in the finished stack:

1. Admin adds domain `example.com` in the admin UI
2. Stack generates: TLS cert (ACME), DKIM keypair, OIDC signing keypair
3. Admin UI presents DNS records to add (or auto-applies them if DNS is managed locally):
   - MX record → smtpd
   - SPF TXT record
   - DKIM TXT record (`mail._domainkey.example.com`)
   - DMARC TXT record
   - WebFinger redirect or hosting
   - OIDC discovery hosting (served by webauth/infodancer-auth)
4. Stack verifies DNS propagation
5. Domain is live: mail works, OIDC works, users can be provisioned

---

## User Provisioning Flow

Admin creates `alice@example.com`:
- Account created in infodancer/auth (mail credentials)
- Account created in webauth (OIDC identity, same credential or linked)
- Maildir provisioned in msgstore
- Welcome email sent

Self-service registration (if enabled per domain):
- User registers on a webauth-hosted page or any app using webauth
- Email verification sent to their address
- After verification: mail account + OIDC identity provisioned together

---

## Federation

Two instances of this stack federate automatically:

1. `alice@instance-a.net` tries to log into an app protected by `instance-b.net`'s webauth
2. webauth at instance-b discovers instance-a's OIDC endpoint via WebFinger
3. Standard OIDC authorization code flow — no prior arrangement required
4. alice authenticates at her home instance, instance-b gets a verified identity claim

The trust model is DNS: if you control the domain, you control the OIDC endpoint for that domain. Domain ownership is the root of trust, not a central registry.

---

## What This Is Not

- Not a replacement for existing large-scale mail providers for people who don't want to self-host
- Not trying to federate with Google, Microsoft, or Apple — they can discover us if they choose to implement standard OIDC discovery; we're not pursuing integration
- Not ActivityPub / Mastodon (though the identity model is philosophically aligned)
- Not a commercial product — this is infrastructure for people who want to own their stack

---

## Open Problems

**Deliverability:** IP reputation for outbound mail remains a real problem independent of PKI correctness. New IP addresses start with no reputation. The PKI manager gets SPF/DKIM/DMARC right, which is necessary but not sufficient. No clean solution exists for this without time in the market or using a relay with established reputation.

**Dynamic upstream clients:** When webauth federates with an external OIDC provider, it needs to be registered as a client at that provider. For other instances of this stack, RFC 7591 dynamic client registration handles this automatically. For arbitrary external providers, manual registration is required. This is a fundamental property of the OIDC trust model, not a gap in our design.

**Key distribution for E2E encrypted mail:** msgstore uses per-user encryption keys. Publishing public keys for E2E encrypted mail (e.g., via autocrypt, WKD) so external senders can encrypt to our users is a separate problem from identity. Tracked in msgstore's encryption design.

**DNS management:** The stack can generate correct DNS records but can't apply them unless DNS is also managed locally (e.g., running PowerDNS or CoreDNS). For externally managed DNS (Cloudflare, Route53), the admin UI presents the records to copy. Full automation requires DNS provider API integration — a rabbit hole best deferred.
