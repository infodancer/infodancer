# Infodancer

Infodancer is a small open-source organization building mail server infrastructure for people who prefer to run their own.

## Philosophy

Email is one of the internet's foundational technologies — federated, open, and owned by no one. We think it should stay that way. The goal here is to make self-hosting mail practical for small operators: individuals, families, small organizations, and anyone else who would rather not hand their communication over to a large provider.

We write Go. We prefer simple, auditable code over clever code. Security is a design requirement, not a feature. We use standard protocols where they serve us and aren't afraid to question them where they don't.

## Projects

| Project | Description |
|---|---|
| [smtpd](https://github.com/infodancer/smtpd) | SMTP server for receiving mail |
| [pop3d](https://github.com/infodancer/pop3d) | POP3 server for mail retrieval |
| [imapd](https://github.com/infodancer/imapd) | IMAP server for mail access |
| [msgstore](https://github.com/infodancer/msgstore) | Message storage library shared by all daemons |
| [auth](https://github.com/infodancer/auth) | Shared authentication library |
| [webadmin](https://github.com/infodancer/webadmin) | Web administration interface |

## Design Goals

- **Correct before clever.** Implement the protocol. Test it. Then optimize if necessary.
- **Reject early.** Validate during the protocol conversation. Never generate bounces after the fact.
- **Small operator first.** Configuration should be simple. Operational burden should be low.
- **Secure by default.** TLS required, sane defaults, no legacy footguns enabled without explicit configuration.

## Status

Active development. Not yet production-ready for general use.

## Contributing

Each project has its own repository with a CLAUDE.md and CONVENTIONS.md describing its architecture and coding standards. Issues and pull requests welcome.

## Research

The `docs/` directory contains design research and protocol proposals, including longer-term thinking about where email infrastructure should go.
