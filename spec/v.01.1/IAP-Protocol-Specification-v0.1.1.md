# IAP Protocol Specification v0.1.1

---

# 1. Introduction

## 1.1 Purpose

The Identity Anchor Protocol (IAP) defines a cryptographic standard for establishing and verifying persistent identity and continuity of autonomous software agents.

The protocol provides:

* Deterministic identity derivation
* Canonicalized memory state anchoring (via AMCS-0.1)
* Ordered continuity attestations
* Cryptographically verifiable certificates
* Offline verification capability

IAP defines identity and continuity primitives only.
It does not define agent intelligence, behavior, or evaluation criteria.

---

## 1.2 Redline Summary (v0.1.1)

This version clarifies and tightens the prior draft to match current implementation behavior:

* AMCS event envelope fields updated to AMCS-0.1 canonical structure.
* `memory_root` derivation updated to Merkle root of ordered AMCS event hashes.
* Continuity `ledger_sequence` baseline updated to start at `1`.
* Certificate versioning clarified: protocol certificate version remains `IAP-0.1` in v0.1.x.
* Key Rotation certificate type added to normative certificate taxonomy.
* Verification section split into required baseline checks and optional strict checks.

---

# 2. Terminology

**Agent**  
A software system capable of autonomous or semi-autonomous operation.

**Agent Keypair**  
An Ed25519 public/private keypair controlled by the agent runtime.

**agent_id**  
Deterministic identifier derived from the agent public key.

**Registry**  
An implementation of an IAP Certificate Authority issuing signed certificates.

**Certificate**  
A structured, signed document conforming to this specification.

**Continuity Event**  
An AMCS-compliant memory ledger event entry.

**AMCS-0.1**  
Agent Memory Canonicalization Standard v0.1 defining event logging and memory-root derivation.

---

# 3. Identity Derivation

## 3.1 Key Algorithm

Agents MUST use:

* Ed25519 keypairs

Private keys MUST remain agent-controlled and MUST NOT be transmitted to registries.

---

## 3.2 agent_id Derivation

```
agent_id = "ed25519:" + base32(sha256(public_key_bytes))[0:32]
```

Rules:

* SHA-256 MUST be used.
* Output MUST be lowercase base32.
* Truncation MUST use the first 32 characters.
* Derivation MUST be deterministic.

This creates a self-certifying identity.

---

# 4. Canonicalization Rules

All signed payloads MUST:

* Use UTF-8 encoding
* Use canonical JSON
* Sort keys lexicographically
* Exclude insignificant whitespace
* Reject non-finite numbers
* Reject floats unless a profile explicitly allows them

Canonical bytes are the exact input for signing.

---

# 5. AMCS-0.1 Memory Ledger (Normative Binding)

IAP Continuity Certificates MUST reference a `memory_root` derived using AMCS-0.1.

---

## 5.1 Event Structure (AMCS-0.1)

Each AMCS event envelope MUST contain:

* `amcs_version` (string, MUST equal `AMCS-0.1`)
* `event_id` (UUIDv4 string)
* `agent_id` (string)
* `sequence` (positive integer; assigned on append)
* `event_type` (string)
* `timestamp` (RFC3339 UTC)
* `scope` (`private` | `shared` | `public`)
* `payload` (object)
* `tags` (string array)

Each event MUST be canonicalized before hashing.

---

## 5.2 Event Hashing

For each event:

```
event_hash = sha256(canonical_event_bytes)
```

Rules:

* Hash MUST be lowercase hex.
* SHA-256 is mandatory for v0.1.x.

---

## 5.3 Ledger Ordering

Events MUST be:

* Strictly append-only
* Sequentially ordered by `sequence`
* Persisted with immutable hash history semantics

Mutation of past events invalidates integrity verification.

---

## 5.4 memory_root Derivation

In v0.1.x, `memory_root` MUST be the Merkle root of ordered event hashes:

```
memory_root = merkle_root_hex([event_hash_1, event_hash_2, ..., event_hash_n])
```

Rules:

* Leaf order MUST follow ascending `sequence`.
* Merkle pair hashing MUST use SHA-256 over raw hash bytes (`left || right`).
* For odd node counts, the last node MUST be duplicated at that tree level.
* Result MUST be lowercase hex.

---

# 6. Certificate Authority Model

* Registries sign certificates.
* Multiple registries MAY exist.
* Inter-registry trust is out of scope.
* Verification MUST NOT require registry availability.

---

# 7. Certificate Structure

All certificates MUST include:

* `certificate_version` (v0.1.x value: `IAP-0.1`)
* `certificate_type`
* `agent_id`
* `issued_at` (RFC3339 UTC)
* `registry_id`
* `registry_signature_b64`

Optional:

* `metadata`

Metadata MUST NOT influence identity derivation.

Certificates are immutable once issued.

---

# 8. Certificate Types

---

## 8.1 Identity Anchor Certificate

`certificate_type = "IAP-Identity-0.1"`

Required:

* `agent_public_key_b64`

Establishes binding between `agent_id` and public key.

---

## 8.2 Continuity Certificate

`certificate_type = "IAP-Continuity-0.2"`

Required:

* `memory_root`
* `ledger_sequence`

Rules:

* `ledger_sequence` MUST start at `1`.
* `ledger_sequence` MUST strictly increase per `agent_id`.
* Registry MUST verify:
  * Agent signature
  * Correct `agent_id` derivation
  * Monotonic `ledger_sequence`

Note: Full AMCS ledger replay/root recomputation is verifier-dependent and MAY be performed when ledger data is available.

---

## 8.3 Lineage Certificate

`certificate_type = "IAP-Lineage-0.1"`

Optional:

* `parent_agent_id`
* `fork_event_hash`

Used to declare canonical lineage claims.

---

## 8.4 Key Rotation Certificate

`certificate_type = "IAP-KeyRotation-0.1"`

Required:

* `old_agent_id`
* `new_agent_id`
* `old_agent_public_key_b64`
* `new_agent_public_key_b64`

Used to attest approved continuity transition from old to new identity keys.

---

# 9. Signing Rules

---

## 9.1 Agent-Side Signing

Signed issuance requests MUST include:

* Canonical payload
* Agent signature field(s)
* `nonce`
* `created_at`
* Request intent discriminator (for domain separation)

Nonce rules:

* MUST be unique per request for a given `agent_id`
* MUST be verified by registry
* MUST prevent replay attacks

Replay protection is REQUIRED.

---

## 9.2 Registry Signing

Registry MUST:

* Canonicalize certificate excluding `registry_signature_b64`
* Sign canonical bytes using Ed25519 private key

Signature MUST cover all remaining fields.

---

# 10. Verification Procedure

Baseline verification (required):

1. Canonicalize certificate excluding `registry_signature_b64`.
2. Verify registry signature with published registry public key.
3. Validate `certificate_version` and `certificate_type`.
4. Verify `agent_id` derivation from referenced public key material.

Continuity-specific checks:

1. Verify `ledger_sequence` monotonicity against prior continuity context when available.
2. If AMCS ledger data is available, independently verify AMCS event chain and recompute `memory_root`.

Verification MUST be possible offline for baseline cryptographic checks.

---

# 11. Derived Properties

Agent age:

```
now - first Identity Anchor issued_at
```

Continuity count:

```
highest ledger_sequence
```

---

# 12. Security Considerations

* Private keys MUST remain agent-controlled.
* Registry keys SHOULD be hardware-protected in production.
* Replay protection via nonce is REQUIRED.
* Clock-skew tolerance SHOULD be defined by registry policy.
* Canonicalization errors invalidate signatures.
* Offline verification MUST remain possible for baseline checks.

---

# 13. Extension Rules

Future versions MUST:

* Introduce new `certificate_type` identifiers for new semantics
* Preserve backward compatibility where feasible
* Avoid mutation of existing field semantics
* Increment protocol version for breaking cryptographic or semantic changes

---

# 14. Scope Clarification

IAP v0.1.x provides:

* Identity binding
* Ordered continuity attestations
* Tamper-evident memory anchoring

It does NOT provide:

* Behavioral evaluation
* Reputation scoring
* Governance enforcement
* Mandatory blockchain anchoring

---

# Summary Positioning

IAP v0.1.1 with AMCS-0.1 defines a minimal, deterministic, cryptographically verifiable continuity layer for autonomous agents, with offline-verifiable certificate trust.
