# Identity Anchor Protocol (IAP)

Identity Anchor Protocol (IAP) is a cryptographic standard for establishing
persistent, verifiable identity and continuity for autonomous agents.

IAP defines certificate types, signature rules, and verification requirements.
It does not define behavior, intelligence, governance, or evaluation criteria.

The protocol enables independent verification of agent identity and continuity
without requiring registry availability.

---

## Purpose

Autonomous systems increasingly act across sessions, environments, and
organizations. IAP provides a minimal cryptographic primitive for:

- Binding an `agent_id` to a public key
- Anchoring state continuity across time
- Declaring canonical identity lineage

IAP does not assess intelligence or capability.
It certifies cryptographic facts.

---

## Design Principles

- **Cryptographic First** — All attestations are signed using Ed25519.
- **Offline Verifiable** — Verification MUST NOT require registry availability.
- **Minimal Surface Area** — Identity and continuity primitives only.
- **Versioned Protocol** — Certificate types are explicitly versioned.
- **Deterministic Serialization** — Canonical JSON required for verification.

---

## Certificate Types (IAP-0.1)

### Identity Anchor (`IAP-Identity-0.1`)

Binds a deterministic `agent_id` to an Ed25519 public key at a defined time.

Required before issuing any other certificate type.

---

### Continuity (`IAP-Continuity-0.2`)

Attests that a specific `memory_root` is bound to an identity
under a strictly monotonic `ledger_sequence`.

Enables verification of persistent identity across sessions / time.

---

### Lineage (`IAP-Lineage-0.1`)

Declares canonical parent or fork relationships between identities.

Enables cryptographic traceability of derived agents.

---

## Verification Model

All certificates:

- Are signed by the Registry private key
- Include a `registry_signature_b64`
- Must be verified using canonical JSON serialization
- Exclude the `registry_signature_b64` field from the signed payload
- Are verifiable using the published registry public key

Verification does not require contacting the registry.

---

## Registry Role

The Registry acts as a Certificate Authority (CA):

- Verifies protocol conditions
- Verifies agent signatures where required
- Issues signed certificates
- Publishes its public key

The Registry does not control agents.
It does not evaluate intelligence or behavior.
It certifies identity-bound cryptographic facts.

---

## Specification

The formal specification is located at:

- [`IAP-Protocol-Specification-v0.1.md`](./IAP-Protocol-Specification-v0.1.md)

Future protocol revisions will be versioned.

---

## Status

IAP-0.1 is an initial protocol release focused on identity anchoring and
continuity primitives.

Backward compatibility is not guaranteed between major protocol versions.

---

## Contributing

Protocol discussions, issues, and revision proposals are welcome.

Changes to certificate formats, signature rules, or verification requirements
should include:

- Clear security reasoning
- Backward compatibility analysis
- Migration considerations

---

## License

Creative Commons Attribution 4.0 (CC BY 4.0).
