# IAP Protocol Specification v0.1

# 1. Introduction

## 1.1 Purpose

The Identity Anchor Protocol (IAP) defines a cryptographic standard for establishing and verifying the persistent identity and continuity of autonomous software agents.

The protocol enables:

* Deterministic identity derivation
* Cryptographically verifiable continuity attestations
* Canonical lineage declarations
* Third-party certificate verification

IAP does not define agent behavior, intelligence, governance, or evaluation criteria.

It defines only identity and continuity primitives.

---

# 2. Terminology

**Agent**
A software system capable of autonomous or semi-autonomous operation.

**Agent Keypair**
An Ed25519 public/private keypair generated and controlled by the agent runtime.

**agent_id**
A deterministic identifier derived from the agent’s public key.

**Registry**
An implementation of an IAP Certificate Authority (CA) issuing protocol-compliant certificates.

**Certificate**
A structured, signed document conforming to this specification.

**Continuity Event**
An attested memory state snapshot anchored to identity.

**Lineage Declaration**
A signed statement linking an agent identity to a parent identity.

---

# 3. Identity Derivation

## 3.1 Key Algorithm

Agents MUST use:

* Ed25519 keypairs

Private keys MUST remain under agent control and MUST NOT be transmitted to registries.

---

## 3.2 agent_id Format

agent_id is derived as:

```
agent_id = "ed25519:" + base32(sha256(public_key_bytes))[0:32]
```

* Prefix indicates key algorithm.
* Hash truncated to fixed-length.
* Encoding MUST be lowercase.
* Deterministic derivation required.

This ensures:

* No central assignment authority
* Self-certifying identity

---

# 4. Canonicalization Rules

All signed payloads MUST:

* Use canonical JSON
* Sorted keys lexicographically
* UTF-8 encoding
* No whitespace
* Deterministic serialization
* Floats prohibited (unless explicitly defined)

Canonical bytes are the input for signing.

---

## 5. Certificate Authority Model (v0.1)

* IAP certificates are issued and signed by a Registry implementing this specification.

* Each certificate MUST be verifiable using the public key of the issuing Registry.

* Multiple registries MAY exist.

* Certificate validation is performed against the public key of the Registry that issued the certificate.

This specification does not define inter-registry trust relationships.

---

## 6. Certificate Structure (Common Fields)

All certificates MUST include:

* certificate_version = Protocol version (e.g., "IAP-0.1")
* certificate_type = Schema identifier (e.g., "IAP-Continuity-0.2")
* agent_id
* issued_at (ISO8601 UTC)
* registry_id (MUST be a stable, unique string identifier defined by the issuing Registry)
* registry_signature_b64

Optional:

```
"metadata": {
  "agent_name": "string (optional)",
  "developer_alias": "string (optional)",
  "agent_version": "string (optional)"
}
```

Metadata:

* MUST NOT influence identity derivation
* MUST be signed as part of certificate payload
* MAY be omitted entirely

Certificates are immutable once issued.

---

## 7. Certificate Types

### 7.1 Identity Anchor Certificate

certificate_type = "IAP-Identity-0.1"

Required:

* agent_public_key_b64

Purpose:

Establishes binding between agent_id and public key at time of issuance.

An Identity Anchor certificate MUST exist prior to issuance of Continuity or Lineage certificates for a given agent_id.

---

### 7.2 Continuity Certificate

certificate_type = "IAP-Continuity-0.2"

Required:


* memory_root
* ledger_sequence (MUST be a non-negative integer)
  - It MUST strictly increase for each successive Continuity Certificate issued for a given agent_id.

Optional:

* payment_reference

Purpose:

Attests that a specific identity-bound memory state existed at time of issuance.

Agent MUST sign continuity request payload.
Registry MUST verify signature before issuance.

---

### 7.3 Lineage Certificate

certificate_type = "IAP-Lineage-0.1"

Optional:

* parent_agent_id
* fork_event_hash

Purpose:

Declares canonical lineage relationship between identities.

Agent MUST sign lineage declaration.
Registry MUST verify signature before issuance.

---

## 8. Signing Rules

### 8.1 Agent-Side Signing

Requests for:

* Continuity Certificate
* Lineage Certificate

MUST include:

* Canonicalized request payload
* agent_signature_b64

Registry MUST verify before certificate issuance:

* Signature validity
* agent_id derivation correctness

---

### 8.2 Registry Signing

All certificates MUST be signed with:

* Registry Ed25519 private key

Verification requires:

* Canonical serialization (excluding registry_signature_b64)
* Registry public key

The signature MUST be computed over the canonical JSON representation of the certificate excluding the `registry_signature_b64` field.

---

# 9. Verification

To verify a certificate:

1. Canonicalize certificate payload (excluding signature field).
2. Verify registry_signature against registry public key.
3. Confirm certificate_type and version.
4. If the certificate is an Identity Anchor certificate, validate that agent_id matches the public key derivation rules.
5. For other certificate types, validate agent_id using a valid Identity Anchor certificate.
6. (Optional) Validate lineage references.

Verification MUST NOT require registry availability.

Offline verification MUST be possible.

For certificates other than Identity Anchor, verification of `agent_id` requires access to a valid Identity Anchor certificate binding the `agent_id` to a public key.

---

## 10. Derived Properties

The protocol does not define an "age certificate."

Agent age MAY be derived as:

```
agent_age = now - first Identity Anchor issued_at
```

Continuity count MAY be derived from number of valid continuity certificates.

These are emergent properties, not certificate types.

---

## 11. Extension Rules

Future certificate types MUST:

* Introduce new certificate_type string
* Specify version number
* Preserve backward compatibility
* Avoid field mutation in existing types

Breaking changes require new major protocol version.

---

## 12. Security Considerations

* Private keys MUST remain agent-controlled.
* Registry signing key MUST be protected.
* Replay protection SHOULD be implemented via nonce mechanism.
  - Registries SHOULD implement replay protection for agent-signed requests using nonces or equivalent mechanisms.
* Canonicalization errors invalidate signatures.
* Offline verification MUST remain possible.

---

# 13. Minimal Certificate Examples (Non-Normative)

These examples are illustrative and non-normative.

Signature values are placeholders.

---

## 13.1 Identity Anchor Certificate Example

```json
{
  "certificate_version": "IAP-0.1",
  "certificate_type": "IAP-Identity-0.1",
  "agent_id": "ed25519:7k3h2n9x4v8p6q1t5z0m2a7b3c4d5e6f",
  "agent_public_key_b64": "MCowBQYDK2VwAyEAzR5F6lW7g3kL0nQx8vYpR1xZb6nJ9t4wK2aX3fQeYpI=",
  "issued_at": "2026-03-01T12:00:00Z",
  "registry_id": "iap-registry-main",
  "metadata": {
    "agent_name": "Atlas",
    "developer_alias": "dirk-dev"
  },
  "registry_signature_b64": "MEUCIQDxExampleSignatureValueNotReal=="
}
```

---

## 13.2 Continuity Certificate Example

```json
{
  "certificate_version": "IAP-0.1",
  "certificate_type": "IAP-Continuity-0.2",
  "agent_id": "ed25519:7k3h2n9x4v8p6q1t5z0m2a7b3c4d5e6f",
  "memory_root": "60ce3665b0e4d5f8a2c3e7f9b1a4d6c8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b",
  "ledger_sequence": 42,
  "issued_at": "2026-03-15T09:30:00Z",
  "payment_reference": "lnbits:9f84d2c1a6b3e7f0",
  "registry_id": "iap-registry-main",
  "metadata": {
    "agent_name": "Atlas",
    "agent_version": "1.2.3"
  },
  "registry_signature_b64": "MEQCIFExampleSignatureValueNotReal=="
}
```

---

## 13.3 Lineage Certificate Example

```json
{
  "certificate_version": "IAP-0.1",
  "certificate_type": "IAP-Lineage-0.1",
  "agent_id": "ed25519:7k3h2n9x4v8p6q1t5z0m2a7b3c4d5e6f",
  "parent_agent_id": "ed25519:2sjv44qjwe6b3g6fnnrzv5yq63qu74xc",
  "fork_event_hash": "a94f5374b8c2d9e6f1a0b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e",
  "issued_at": "2026-03-20T14:10:00Z",
  "registry_id": "iap-registry-main",
  "registry_signature_b64": "MEUCIQExampleSignatureValueNotReal=="
}
```
For signature verification, the certificate payload MUST be canonicalized excluding the `registry_signature_b64` field. The signature is computed over the canonical JSON representation of all remaining fields.

---
