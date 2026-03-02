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

## Critical Reading

Before making changes to **infodancer/auth** or **infodancer/webauth**:

**Read [oidc-federation-design.md](docs/oidc-federation-design.md) first.**

It documents use cases, the correct security controls, and — explicitly — decisions
that were made, reversed, and must not be repeated.
