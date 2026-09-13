# infodancer

Home of the **infodancer** organization: the cross-cutting design documents
behind our self-hosted, federated mail and identity stack.

For the project overview and the active repositories, see the
[organization profile](https://github.com/infodancer). The mail server suite
lives in [maildancer](https://github.com/infodancer/maildancer).

## Design documents

Living design docs -- the reasoning behind the architecture, not only its
current state. Read the relevant one before changing the matching subsystem.

| Document | What it covers |
|---|---|
| [mail-security-model](docs/mail-security-model.md) | Process separation, privilege model, uid/gid allocation, the pipe protocol |
| [encryption-design](docs/encryption-design.md) | At-rest encryption: key model, delivery and retrieval points, search without a decrypted cache |
| [keyring-design](docs/keyring-design.md) | Client keyring and key-encryption-key (KEK): client-generated keys, wrap-slots, and the off/on/escrow trust postures |
| [oidc-federation-design](docs/oidc-federation-design.md) | Auth stack design and security model (read before touching auth or webauth) |
| [federated-identity-stack](docs/federated-identity-stack.md) | Identity federation overview |
| [next-gen-messaging-protocol](docs/next-gen-messaging-protocol.md) | SCMP/SDMP design: end-to-end encryption, sender stores and recipient pulls |
| [protocol-outlines](docs/protocol-outlines.md) | Open protocol problems and options, ahead of specification |
| [session-manager-design](docs/session-manager-design.md) | Session manager architecture |
| [mail-session-deliver-merge](docs/mail-session-deliver-merge.md) | Merging the mail-session and delivery agents |
| [queue-design](docs/queue-design.md) | Unified on-disk outbound queue |
| [outbound-transport-routing](docs/outbound-transport-routing.md) | Per-sender-domain delivery transport routing |
| [rcpt-rejection-design](docs/rcpt-rejection-design.md) | Recipient rejection strategy |
| [dsn-bounce-design](docs/dsn-bounce-design.md) | DSN and bounce handling |
| [uid-persistence-design](docs/uid-persistence-design.md) | UID allocation and persistence |
| [deployment-filesystem](docs/deployment-filesystem.md) | Config and data filesystem split, Docker volume strategy |
| [web-portfolio-architecture](docs/web-portfolio-architecture.md) | Reusable Go web modules and the shared UI layer |

## Contributing

Issues and pull requests welcome. Docs use ASCII punctuation only (no smart
quotes, em-dashes, or ellipsis glyphs). After cloning, enable the local guard
with `git config core.hooksPath .githooks`; CI enforces the same rule on push
and pull requests via the shared `ascii-punctuation` action.
