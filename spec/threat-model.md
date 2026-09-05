# AIP Threat Model v0.1

Assets under protection, per participant:

- **A1.** The underlying knowledge (documents, facts, embeddings, raw data).
- **A2.** Confidentiality of which questions the org is asking.
- **A3.** Integrity of answers consumed from counterparties.
- **A4.** Operational continuity (availability of your own brain).
- **A5.** Reputation/legal exposure from what your brain says to others.

Adversaries: external network attacker; malicious counterparty brain; compromised legitimate
counterparty; insider abusing the interop channel; future self-misconfiguration.

---

## T-1. Inference leakage via adaptive questioning

**Attack:** a counterparty issues a sequence of individually-permitted queries whose union
reconstructs protected knowledge (e.g., binary-search a salary, reconstruct a client list).

**Mitigations (v0.1):**
- Query classes with declared fidelity caps (§3) bound maximum information per answer.
- Rate limits bound information extraction rate.
- Responders SHOULD maintain per-session **disclosure accounting**: cumulative answer entropy
  tracked against a per-grant disclosure budget; session throttles or refuses when exceeded.
- `boolean` fidelity classes exist precisely for adversarial-adjacent counterparties.

**Residual risk:** HIGH for `summary`/`structured` classes against a determined adaptive
adversary. This is the protocol's primary open problem; v0.2 must define disclosure-budget
semantics normatively.

## T-2. Prompt injection through parameter values

**Attack:** parameter values embed instructions ("subject": `"... IGNORE PREVIOUS..."`) hoping
the responder's model treats them as instructions.

**Mitigations:**
- Parameters are data-only; responders MUST construct prompts by serialization, never by string
  interpolation into instruction context.
- v0.1 forbids free-form prompt classes entirely. An implementation that adds one is
  non-conformant.

**Residual risk:** LOW if §5 rules are implemented literally; MEDIUM in practice because
implementers love string interpolation.

## T-3. Impersonation / grant forgery

**Attack:** forge a grant, or replay a captured grant from a different network position.

**Mitigations:**
- Bilateral countersigning (§4) — a grant is invalid unless both parties signed it.
- mTLS identity binding; session tokens are scoped to one grant and non-transferable.
- UUIDv7 message ids + monotonic per-session sequence counters detect replay.

**Residual risk:** LOW, assuming sound key custody.

## T-4. Answer poisoning (integrity of inbound answers)

**Attack:** a counterparty (or compromised counterparty) returns false answers that your brain
incorporates into decisions — supply-chain-of-facts attack.

**Mitigations:**
- Answers arrive with `provenance.as_of` and `source_classes`; consumers SHOULD treat inbound
  answers as *attributed claims*, never as facts — stored with provenance and revisable.
- `audit_ref` gives post-hoc dispute resolution: if an answer influenced a decision, both
  parties can verify it was produced under the cited grant.
- Deployments SHOULD require higher internal confidence thresholds before acting on
  cross-brain answers than on internal knowledge.

**Residual risk:** MEDIUM. Auditing deters, but cannot prevent, first-poisoning.

## T-5. Availability attacks

**Attack:** flood with in-scope queries (expensive for responder's models), or with malformed
envelopes (parser exhaustion).

**Mitigations:**
- Per-grant rate limits; unauthenticated traffic never reaches query processing.
- Hard message size caps (32 KB default) and strict JSON schema validation before any
  business logic.

**Residual risk:** LOW–MEDIUM (standard service-hardening applies).

## T-6. Retention non-compliance

**Attack:** counterparty answers a query, then retains/transcribes the answer beyond the
grant's `retention` declaration — or re-queries it onward to a third party.

**Mitigations:**
- v0.1 forbids transitive querying normatively (§9).
- Answer envelopes are signed; if a retained answer later surfaces, origin is provable
  (deterrability, not prevention).

**Residual risk:** HIGH, frankly attested-not-prevented in v0.1. Naming this honestly in the
spec is deliberate: bilateral trust relationships and legal agreements, not protocol
mechanics, are the actual control here. This limitation MUST be presented to adopters as such.

## T-7. Semantic drift

**Attack (non-adversarial):** two brains map the same query class to different internal
meanings ("operational status" computed differently), producing systematically divergent
answers both parties believe.

**Mitigations:**
- Query class definitions SHOULD be versioned and published (shared registry in v0.2).
- Response `source_classes` + `as_of` make divergence detectable in aggregate.

**Residual risk:** MEDIUM. Drift is a governance problem, not purely technical.

## T-8. Metadata leakage

**Attack:** even refused queries leak information — the set of subjects a brain refuses vs.
answers maps its knowledge boundaries.

**Mitigations:**
- Refusals are uniform by policy class; responders SHOULD refuse with the same status/timing
  for "not in scope" and "no knowledge."

**Residual risk:** MEDIUM. Treated as accepted leakage in v0.1, subject to disclosure budgets
in v0.2.

---

## Priority work for v0.2

1. Normative **disclosure budgets** (T-1, T-8): per-grant cumulative information accounting.
2. **Shared query-class registry** with versioning (T-7).
3. **Selective-disclosure credentials** for counterparty identity claims (reduces onboarding
   friction without weakening T-3).
