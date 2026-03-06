# CLAUDE.md

This file provides guidance to Claude Code when working in the infodancer/infodancer repository.

## Repository Purpose

This repo contains cross-cutting documentation for the infodancer mail and web stack.
It is not a runnable service — it holds design docs, protocol specs, and architecture
decisions that span multiple repos.

## Key Design Documents

| Document | Description |
|----------|-------------|
| [mail-security-model.md](docs/mail-security-model.md) | Process separation, privilege model, uid/gid allocation, pipe protocol |
| [oidc-federation-design.md](docs/oidc-federation-design.md) | Auth stack design: auth-oidc + webauth use cases, security model, what must not be changed |
| [federated-identity-stack.md](docs/federated-identity-stack.md) | Identity federation overview |
| [next-gen-messaging-protocol.md](docs/next-gen-messaging-protocol.md) | Messaging protocol design |
| [encryption-design.md](docs/encryption-design.md) | At-rest encryption: key model, delivery/retrieval points, fd key-passing convention |

## Versioning Policy

All infodancer repos follow a unified versioning scheme: `v0.N.X`.

- **The minor version `N` is a human decision.** Never bump it without explicit approval.
- **The patch version `X` is the only thing that gets auto-incremented** when tagging releases.
- All repos in the mail stack should share the same minor version where possible.
  Current target: `v0.1.x` for most repos; `msgstore` is an exception at `v0.2.x`
  (legacy tags cached by the Go module proxy).
- `messagedancer`, `scmp`, and `sdmp` stay at `v0.1.0` until further notice.
- `gotemplate` and `infodancer` (this repo) are not versioned.
- When tagging: use the next available patch for that repo (e.g., if auth is at
  v0.1.12, the next tag is v0.1.13). Never skip numbers.
- All repos stay at `v0.x.y` (pre-1.0) until production-ready.

## Critical Reading

Before making changes to **infodancer/auth** or **infodancer/webauth**:

**Read [oidc-federation-design.md](docs/oidc-federation-design.md) first.**

It documents use cases, the correct security controls, and — explicitly — decisions
that were made, reversed, and must not be repeated.
