# AIP Observability Profile (draft for v0.2)

**Status:** draft — proposed addition for AIP v0.2. Not normative for v0.1
implementations. Sections marked MUST/SHOULD/MAY below become normative if and
when this profile is merged into v0.2.

**What this makes concrete:** AIP-v0.1 §6 requires every response to carry an
`audit_ref` that "resolves, for the counterparty, to a verification record
proving the answer was generated under the cited grant... existence and
conformance, not content." v0.1 leaves the shape of that record unspecified.
This profile defines it, plus the local audit-log mechanics that produce it, so
that **counterparty auditability is verifiable from either party's logs alone**
(design principle 4) rather than asserted.

## 1. Local audit log (normative shape)

Every AIP participant maintains, on its own infrastructure, an append-only audit
log of every request and response it sends or receives.

- **Storage:** one newline-delimited JSON (JSONL) file per UTC day. Local file
  permissions SHOULD restrict read access to the participant's own operator
  (e.g. mode 0600 on POSIX). The log is write-ahead: an entry is fsynced before
  the protocol action it describes is treated as committed.
- **No raw knowledge.** Log entries record metadata and summaries only:
  message class, counterparty BID, session id, message id, size band, response
  status, and the grant id under which the interaction ran. The `fidelity` cap
  of the grant governs what summary fields MAY be recorded — never the answer
  payload, never document/person/system names behind `source_classes`.
- **Logs never leave the brain they are written on.** Nothing in this profile
  requires or permits shipping full audit logs to any counterparty, aggregator,
  or vendor. Only the derived artifacts of §3–§5 cross a trust boundary.

## 2. Tamper-evident chaining (normative shape)

The log is hash-structured so that silent after-the-fact edits are detectable
by the participant itself (and provable to a counterparty via §4).

- **Intra-day chain:** each entry carries `seq` (1-based, per day),
  `prev_hash` (hash of the prior entry, or the prior day's Merkle root for the
  first entry of a day), and `hash = SHA-256(canonical(entry minus hash) ‖ prev_hash)`.
  Canonical form: UTF-8 JSON, sorted keys, compact separators.
- **Inter-day seal:** at day close, the day's entry hashes are folded into a
  single Merkle root; day N's first entry chains to day N−1's root. A missing or
  reordered entry breaks verification deterministically.
- **Self-verification:** participants SHOULD expose an operator command that
  verifies the full chain (intra-day + day-to-day) and reports the earliest
  divergence, plus a command that prints a day's root. Detection is local;
  what an operator does about a detected tamper is out of scope.

## 3. `audit_ref` resolution record (normative shape)

When a response cites `audit_ref`, the record it resolves to MUST contain, at
metadata level only:

| Field | Meaning |
|---|---|
| `log_day` | UTC day the interaction was logged |
| `seq` | entry sequence number within that day |
| `day_root` | Merkle root of that day (present if the day is sealed) |
| `grant_id` | the grant the interaction ran under |
| `status` | response status (`answered` / `refused_*`) |
| `classes` | message class(es) involved |

The record proves **existence and conformance** — that a given message id was
logged, under the cited grant, with the cited status — and nothing about
content. A counterparty MAY cross-check the record against its own log of the
same interaction (message id + timestamp + class); disagreement between the
two parties' records for the same message id is a security event both SHOULD
treat as reportable.

## 4. Interchange proofs (optional MAY)

A participant MAY, upon verified counterparty request (itself an AIP message
under an audit-class grant), disclose a **proof excerpt** rather than a log:
the single entry's hash-chain siblings up to the day root. This lets a
counterparty verify one interaction was logged without receiving any other
participant's entries. Proof excerpts never include payload bytes.

## 5. Summary reporting (`aip-report.v1`, draft)

Operators of multiple participants (a fleet) need health telemetry without
reading anyone's log. A participant MAY expose a summary report derived from
the audit log:

```json
{
  "aip_version": "0.2",
  "report_class": "aip-report.v1",
  "window": { "since": "...", "until": "..." },
  "counts": { "answered": 0, "refused_out_of_scope": 0, "refused_grant_expired": 0,
              "refused_rate_limit": 0, "refused_policy": 0,
              "refused_unresolvable_subject": 0 },
  "by_class": {},
  "chain": { "selfcheck": "pass|fail", "days_sealed": 0 },
  "anomalies": []
}
```

- Counts and classes only — no counterparty-identifiable content beyond
  aggregate numbers; `by_class` MAY name message classes, never BIDs of
  individual counterparties unless the operator is party to those interactions.
- `chain.selfcheck = fail` is the tamper signal (§2); `anomalies` carries
  operator-defined flags (e.g. refusal-rate spikes — see the security
  considerations in the threat model).
- Pull-based: an aggregator polls summaries; the participant never pushes logs
  or summaries to any party not already holding an audit-class grant.

## 6. Why this shape

- **Sovereignty:** every byte of log stays on the participant; only derived,
  metadata-level records and counts cross boundaries, and only on request.
- **Dispute resolution from either side (principle 4):** two honest
  participants' logs agree on message ids, classes, statuses, and grants; the
  chain + roots make each side's record independently tamper-evident.
- **Bounded disclosure:** the record format cannot leak content because it has
  no content-bearing fields — the same "structure over free-form" rule the
  envelope enforces on the wire.
- **Implementable now:** this profile is extracted from a working reference
  implementation (append-only daily JSONL, SHA-256 entry chaining, day-seal
  Merkle roots, self-verify + root CLI), not designed on paper.

## 7. Open questions for v0.2

- Should day roots be externally anchored (e.g. periodic transparency-log
  style checkpoint among fleet participants) or is local chaining sufficient
  for the threat model?
- Multi-party sessions (if added to v0.2) need a defined fan-out of one
  interaction into N log entries — interaction id vs message id.
- Minimum retention window, if any, before an operator may prune sealed days.
