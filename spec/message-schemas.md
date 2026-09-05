# AIP Message Schemas v0.1 (normative)

JSON Schema (2020-12). These schemas are the normative wire format; prose in AIP-v0.1.md is
explanatory.

## Envelope

All AIP messages validate against `Envelope`:

```json
{
  "$id": "https://teloplex.ai/aip/v0.1/envelope",
  "type": "object",
  "required": ["aip_version", "sender_bid", "recipient_bid", "message_id", "timestamp", "signature", "body"],
  "properties": {
    "aip_version": { "const": "0.1" },
    "sender_bid": { "type": "string", "pattern": "^aip:[a-z0-9.-]+:[a-z0-9-]+$" },
    "recipient_bid": { "type": "string", "pattern": "^aip:[a-z0-9.-]+:[a-z0-9-]+$" },
    "session_id": { "type": "string" },
    "message_id": { "type": "string", "format": "uuid" },
    "timestamp": { "type": "string", "format": "date-time" },
    "signature": { "type": "string", "description": "Detached Ed25519 signature over canonical form" },
    "body": { "type": "object" }
  },
  "maxBytes": 32768
}
```

## Body types

`body.type` discriminates:

| type | payload |
|---|---|
| `session.hello` | capability manifest (spec §3) |
| `session.grant_propose` | grant object (spec §4), unsigned |
| `session.grant_accept` | grant object, countersigned |
| `query` | query payload (spec §5) |
| `response` | response payload (spec §6) |
| `session.revoke` | `{ "grant_id": string, "reason": string }` |
| `session.end` | `{ "reason": string? }` |

## Query payload

```json
{
  "type": "object",
  "required": ["query_class", "grant_id", "parameters"],
  "properties": {
    "query_class": { "type": "string", "pattern": "^[a-z0-9_.]+\\.v[0-9]+$" },
    "grant_id": { "type": "string" },
    "parameters": {
      "type": "object",
      "additionalProperties": {
        "oneOf": [
          { "type": "string", "maxLength": 1024 },
          { "type": "number" },
          { "type": "boolean" },
          { "type": "array", "items": { "oneOf": [ { "type": "string", "maxLength": 1024 }, { "type": "number" }, { "type": "boolean" } ] } }
        ]
      }
    },
    "response_constraints": {
      "type": "object",
      "properties": {
        "fidelity": { "enum": ["boolean", "summary", "structured"] },
        "max_tokens": { "type": "integer", "minimum": 1, "maximum": 500 },
        "schema_id": { "type": "string" }
      }
    }
  }
}
```

## Response payload

```json
{
  "type": "object",
  "required": ["answering", "status", "provenance", "audit_ref"],
  "properties": {
    "answering": { "type": "string" },
    "status": { "enum": ["answered", "refused_out_of_scope", "refused_rate_limit", "refused_grant_expired", "refused_unresolvable_subject", "refused_policy"] },
    "fidelity": { "enum": ["boolean", "summary", "structured"] },
    "answer": { "description": "string (summary) | object (structured) | boolean (boolean)" },
    "confidence": { "enum": ["high", "medium", "low", "unknown"] },
    "provenance": {
      "type": "object",
      "required": ["as_of"],
      "properties": {
        "as_of": { "type": "string", "format": "date-time" },
        "source_classes": { "type": "array", "items": { "type": "string" } },
        "disclosure_policy": { "type": "string" }
      }
    },
    "audit_ref": { "type": "string" }
  }
}
```

## Conformance rule

An implementation MUST validate every inbound envelope against these schemas before any other
processing, and MUST drop non-conforming messages with a `refused_policy` response (when a
session exists) or silence (when none does).
