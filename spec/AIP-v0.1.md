# AIP — Agent Interop Protocol v0.1 (DRAFT)

Normative keywords (MUST/MUST NOT/SHOULD/MAY) follow RFC 2119.

---

## 1. Model

An AIP participant is a **brain**: a bounded computational service that answers queries against
an organization's owned knowledge. Brains interact via **contract sessions**:

```
Brain A                          Brain B
  |--- 1. Hello / capabilities -->|
  |<-- 2. Capability set + policy |
  |--- 3. Query (typed class) --->|
  |<-- 4. Response + provenance --|
  |--- 5. Revoke / end ---------->|
```

A session is bound to exactly one bilateral **grant** (§4). There are no broadcast or
multi-party sessions in v0.1.

## 2. Transport & identity

- AIP runs over mutually-authenticated TLS (mTLS). Each brain presents an X.509 cert rooted in
  a published trust root or an out-of-band pinned key.
- Each brain has a stable **brain identifier (BID)**: a URI of the form
  `aip:<org-domain>:<brain-name>`, e.g. `aip:teloplex.ai:ops-brain`. BIDs MUST resolve to a
  **capability manifest** (§3) over HTTPS.
- Message envelopes are JSON over HTTP/2 POST. Every envelope carries:
  `aip_version`, `session_id`, `message_id` (UUIDv7), `sender_bid`, `recipient_bid`,
  `timestamp` (RFC 3339 UTC), and a detached Ed25519 `signature`.

## 3. Capability advertisement

Before any query, each brain publishes a **capability manifest**:

```json
{
  "bid": "aip:example.com:legal-brain",
  "aip_version": "0.1",
  "policy": {
    "contact": "interop@example.com",
    "max_session_hours": 24,
    "log_disclosure": "metadata-only"
  },
  "query_classes": [
    {
      "class_id": "operational.status.v1",
      "description": "Point-in-time operational status of a named process or asset",
      "fidelity": "summary",
      "rate_limit": { "per_minute": 30 }
    },
    {
      "class_id": "reference.lookup.v1",
      "description": "Resolve public identifiers to public descriptive records",
      "fidelity": "structured"
    }
  ]
}
```

- **query_class** is the unit of permission. A brain MUST refuse any query that does not map to
  an advertised class.
- **fidelity** ∈ {`boolean`, `summary`, `structured`} declares the maximum resolution of
  answers. `boolean` answers only true/false/unknown; `summary` returns natural-language
  prose capped at 500 tokens; `structured` returns a declared JSON schema. A brain MUST NOT
  emit answers richer than the negotiated fidelity.

## 4. Grants

A **grant** is a signed, bilateral capability statement:

```json
{
  "grant_id": "grt_...",
  "issuer_bid": "aip:example.com:legal-brain",
  "grantee_bid": "aip:teloplex.ai:ops-brain",
  "allowed_classes": ["operational.status.v1"],
  "max_fidelity": "summary",
  "expires_at": "2026-09-11T00:00:00Z",
  "max_queries_per_minute": 10,
  "revocation_endpoint": "https://example.com/aip/revoke"
}
```

- Grants MUST be signed by the issuer and countersigned by the grantee before first use.
- Grants MUST carry an expiry, with a hard maximum of 24 hours for machine-issued grants,
  and 90 days for administratively-issued grants.
- Revocation MUST take effect within 60 seconds. Both sides MUST check their revocation
  endpoints at least every 30 seconds per active session.
- A grant does NOT confer access to anything. It confers the right to *ask*.

## 5. Query messages

A query is a typed envelope:

```json
{
  "aip_version": "0.1",
  "query_class": "operational.status.v1",
  "session_id": "ses_...",
  "grant_id": "grt_...",
  "parameters": {
    "subject": "warehouse-east receiving dock",
    "as_of": "2026-09-04T20:30:00Z"
  },
  "response_constraints": { "fidelity": "summary", "max_tokens": 400 }
}
```

Rules:

- `parameters` MUST be a flat object of scalar values or scalar arrays. No executable
  content, no nested free-text instructions, no embedded markup interpreted by the responder.
- The responder MUST treat parameter values as data, never as instructions.
- Queries referencing identifiers not resolvable to public or explicitly-shared records MUST be
  refused with `unresolvable_subject`.

## 6. Response messages

```json
{
  "aip_version": "0.1",
  "answering": "operational.status.v1",
  "status": "answered",
  "fidelity": "summary",
  "answer": "Receiving dock is operating at normal capacity as of 20:30 UTC.",
  "confidence": "high",
  "provenance": {
    "as_of": "2026-09-04T20:31:02Z",
    "source_classes": ["telemetry", "ops-log"],
    "disclosure_policy": "metadata-only"
  },
  "audit_ref": "aud_..."
}
```

- Every response MUST carry a `provenance.as_of` (the time the underlying knowledge is
  valid *at*, not when the message was sent).
- `source_classes` names the internal knowledge categories consulted — never the documents,
  people, or systems behind them.
- `audit_ref` MUST resolve, for the counterparty, to a verification record proving the answer
  was generated under the cited grant. It discloses existence and conformance, not content.

## 7. Refusals

Refusals are first-class. Status values: `answered`, `refused_out_of_scope`,
`refused_rate_limit`, `refused_grant_expired`, `refused_unresolvable_subject`,
`refused_policy`. A brain SHOULD explain refusals at metadata level only.

## 8. Termination

Either party MAY end a session at any time by sending `session.end` or revoking the grant.
On termination, both parties retain logs; no knowledge acquired during the session may be
retained beyond the grant's scope declaration (`retention: session | query | none`).
Default retention: `query` — an answer may be cached for the duration of that query only.

## 9. Known limitations of v0.1

- No multi-party sessions, no transitive delegation (A→B→C chains MUST NOT form; answers may
  not be re-queried onward).
- No negotiation of new query classes in-session; classes change only by manifest update.
- Retention enforcement is attested, not technically provable. See threat model §T-6.
