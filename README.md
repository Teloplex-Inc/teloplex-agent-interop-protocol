# Agent Interop Protocol (AIP)

**Status:** v0.1 draft — open standard
**Authors:** Teloplex, Inc.
**License:** The specification text is licensed CC BY 4.0. Normative schemas, examples, and
any reference code are licensed Apache 2.0 (express patent grant for implementers).
Implementations of this specification may be proprietary.

## What this is

AIP is a protocol for **brains** — organization-owned intelligence layers — to communicate with
each other across trust boundaries. It defines how one organization's brain can safely answer
queries from, and issue queries to, another organization's brain, without either party granting
the other access to its underlying knowledge.

AIP is not a data-sharing protocol, a prompt-passing convention, or an API wrapper. It is a
**trust contract**: every interaction is scoped, structured, logged, and revocable by design.

## Design principles

1. **Brains answer queries; they do not expose store.** No participant ever receives raw
   knowledge, embeddings, document full-text, or credentials from another participant.
2. **Structure over free-form.** Cross-brain communication uses typed contract classes, never
   raw natural-language prompts. This eliminates prompt-injection as a cross-org attack vector.
3. **Least privilege by default.** Every grant is: query-class-limited, time-boxed, rate-limited,
   and revocable within seconds.
4. **Full auditability.** Every request and response carries a provenance chain; both sides log
   every interaction. Disputes must be resolvable from either party's logs alone.
5. **Fail closed.** Any malformed, out-of-scope, or unverifiable request is refused. Silence is
   never interpreted as consent.

## Documents

- [`spec/AIP-v0.1.md`](spec/AIP-v0.1.md) — the protocol specification
- [`spec/threat-model.md`](spec/threat-model.md) — adversarial analysis and mitigations
- [`spec/message-schemas.md`](spec/message-schemas.md) — normative message schemas

## Why open

Teloplex builds proprietary implementations of this protocol. The specification itself is open:
inter-brain trust only works if the counterparty can independently audit the contract it is
signing up to. Standards are shared; implementations are moats.

## Versioning

AIP uses semantic versioning. v0.x is exploratory — any normative statement may change. v1.0
will be the first stability commitment.
