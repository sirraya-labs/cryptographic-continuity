

# Cryptographic Continuity: Patterns for Long-Term Trust Across Cryptographic Transitions

## A Container-Agnostic Architectural Framework

**Status:** Community Group Draft  
**Editors:** Amir Hameed Mir (Sirraya Lab)  
**Repository:** w3c-cg/cryptographic-continuity  
**Latest Draft:** [https://w3c-cg.github.io/cryptographic-continuity](https://w3c-cg.github.io/cryptographic-continuity)

---

## Abstract

Cryptographic systems are not permanent. Algorithms weaken, implementations break, and organizations must migrate to new assumptions while continuing to trust material signed years or decades earlier. This document defines **Cryptographic Continuity**—an architectural discipline for keeping records verifiable, auditable, and trustworthy across cryptographic generations.

It is not a standard for any single container format. It is a universal pattern language that applies whether the protected object is a JSON document, a CBOR-encoded credential, an X.509 certificate chain, a DNSSEC zone, a firmware image, a transparency log entry, or a Bluetooth handshake. It specifies *what* properties a system must exhibit to survive cryptographic transitions, and *how* those properties manifest in specific ecosystems—but it does not mandate *which* ecosystem to use.

This document does not define new cryptographic algorithms, does not modify existing container specifications, and does not mandate specific suites. It provides the missing operational layer: when to dual-proof, how verifiers should behave during transitions, how to preserve evidence when algorithms break, and what to do when multiple assumptions fail simultaneously.


## Table of Contents

1. [Introduction](#introduction)
2. [Terminology](#terminology)
3. [Design Principles](#design-principles)
4. [The Universal Pattern Language](#the-universal-pattern-language)
   - 4.1 Parallel Independent Proofs
   - 4.2 Verifier Policy During Transitions
   - 4.3 Re-Anchoring
   - 4.4 Evidence Preservation Beyond Signatures
   - 4.5 Catastrophic Dual Break Response
5. [Container Bindings](#container-bindings)
   - 5.1 W3C Verifiable Credentials Data Integrity
   - 5.2 JOSE / COSE
   - 5.3 X.509 and PKIX
   - 5.4 Web Authentication (WebAuthn)
   - 5.5 Offline and Proximity Protocols
   - 5.6 DNSSEC
   - 5.7 Firmware and Software Signing
   - 5.8 Transparency Logs and Verifiable Data Structures
6. [Shared Infrastructure](#shared-infrastructure)
   - 6.1 Algorithm and Suite Status Registry
   - 6.2 Canonicalization Conformance
   - 6.3 Test Vectors
7. [Threat Model](#threat-model)
8. [Operational Deployment Considerations](#operational-deployment-considerations)
9. [Stakeholder Guidance](#stakeholder-guidance)
10. [Living Document Process](#living-document-process)
11. [Security and Privacy Considerations](#security-and-privacy-considerations)
12. [Acknowledgements](#acknowledgements)


## 1. Introduction

A diploma issued today may need to remain verifiable for a professional lifetime. A land title may need to outlast multiple generations of cryptographic standards. An audit log protecting public-interest data may be subpoenaed decades after the algorithms that signed it are retired. A firmware image in a medical device may need to be authenticated long after its manufacturer has ceased operations.

Each of these scenarios shares a common property: the cryptographic tools used to establish trust at issuance time will not be the tools available at verification time. The question this document addresses is not "which algorithm should we use"—it is: **how does a system keep old records trustworthy while actively changing the tools used to establish trust in new ones?**

This property—**cryptographic continuity**—is an engineering discipline that sits above any individual cryptosuite, container format, or protocol. It is not solved by choosing a "better" algorithm, because the definition of "better" changes over time. It is solved by architectural patterns that make cryptographic assumptions replaceable without breaking the records they protect.

### 1.1 Scope

This document covers:

- Universal patterns for maintaining verifiability across cryptographic transitions
- Container-specific bindings showing how those patterns manifest in major ecosystems
- Verifier behavior specification during transition windows
- Evidence preservation strategies that survive algorithm breaks
- The catastrophic dual-break case and graceful degradation
- Shared infrastructure: algorithm status registries, test vectors, conformance guidance

### 1.2 Out of Scope

- Defining new cryptographic algorithms or primitives
- Replacing or modifying existing container specifications
- Mandating specific algorithms or establishing regulatory requirements
- Encryption continuity (harvest-now-decrypt-later for confidentiality is related but addressed through analogous patterns in encryption-specific documents)

### 1.3 How to Read This Document

- **Architects and policy makers** should start with the Design Principles (Section 3) and Universal Pattern Language (Section 4)
- **Implementers** should start with their ecosystem's Container Binding (Section 5) and the Shared Infrastructure (Section 6)
- **Auditors and compliance officers** should focus on the Verifier Policy (Section 4.2), Evidence Preservation (Section 4.4), and Stakeholder Guidance (Section 9)
- **Everyone** should read the Catastrophic Dual Break Response (Section 4.5)



## 2. Terminology

**Cryptographic Continuity**
: The property that a system preserves the verifiability, integrity, and evidentiary value of protected objects across changes in the cryptographic algorithms, suites, or key material used to secure them.

**Protected Object**
: Any digital artifact whose authenticity or integrity is established through cryptographic means. A Verifiable Credential, a JWT, an X.509 certificate, a DNSSEC-signed zone file, a signed firmware image, a transparency log entry, a WebAuthn assertion, a COSE-signed message.

**Proof**
: A cryptographic assertion of authenticity and/or integrity over a protected object. The term is used generically across ecosystems: a Data Integrity proof in W3C, a JWS signature in JOSE, a Certificate signature in X.509, an assertion signature in WebAuthn.

**Proof Set**
: Two or more independent proofs over the same protected object, each using a different cryptosuite or key, allowing the object to be verifiable under multiple assumptions simultaneously. The container-specific term varies: `proof` array in W3C Data Integrity, `signatures` array in JOSE, multiple certificates in X.509 cross-certification, multiple assertion signatures in WebAuthn.

**Dual-Proof**
: The specific case of a proof set containing exactly one classical and one post-quantum proof. The recommended default posture for new long-lived protected objects during the current post-quantum transition.

**Proof Chain**
: An ordered sequence of proofs in which each proof (after the first) also commits to the preceding proof, expressing sequential re-attestation over time. Container-specific: W3C proof chains, countersignatures in JOSE/COSE, certificate chains in X.509.

**Re-Anchoring**
: The act of producing a new proof (or proof set) over a previously issued protected object using a currently-trusted suite, before the suite originally used is retired, so that the object's verifiability does not depend solely on a suite scheduled for deprecation.

**Suite Status**
: A lifecycle state assigned to a cryptographic algorithm or suite:
- `active`: Currently recommended for use. No known weaknesses that reduce its security below the intended level.
- `monitor`: Under active cryptanalysis. Known results exist that warrant attention but do not yet constitute a practical break. Migration planning should begin.
- `deprecated`: Superseded by a stronger suite but not known to be broken. New objects SHOULD NOT use this suite; existing objects SHOULD be re-anchored.
- `broken`: No longer provides the security property it was relied upon for. Proofs using this suite MUST NOT be treated as establishing authenticity.

**Catastrophic Dual Break**
: A scenario in which both a classical assumption and a post-quantum assumption protecting the same object fail—whether simultaneously, in close succession, or by a single technique effective against both.

**Composite / Hybrid Proof**
: A proof construction that combines two or more algorithms into a single atomic value (e.g., Composite ML-DSA in X.509), distinct from a proof set, which keeps proofs structurally separate. Both patterns have valid use cases; this document addresses both and specifies when each is appropriate.

**Harvest-Now-Decrypt-Later (HNDL)**
: An adversary strategy of recording protected material today for future attack once sufficient capability becomes available. The signature analogue is "harvest-now-forge-later": recording signed objects today to forge signatures once the algorithm is broken.



## 3. Design Principles

These principles are intended to hold for the current post-quantum transition and for every subsequent transition, decades from now, driven by causes not yet known.

**P1 — Agility is the default, not the exception.**
Systems MUST be designed assuming that the cryptosuite in use today will not be the cryptosuite in use at the end of a protected object's useful life. The ability to add, remove, and replace proofs without invalidating the protected object is a baseline architectural requirement, not an optional feature.

**P2 — No single mathematical assumption should be a single point of failure for long-lived, high-consequence records.**
Where a protected object's lifetime exceeds the expected lifetime of a cryptographic assumption, the object SHOULD be protected by at least one proof based on a structurally independent assumption. The current landscape offers: algebraic (elliptic curve, finite field), lattice-based (ML-DSA, ML-KEM), and hash-based (SLH-DSA, LMS/XMSS) assumptions. A long-lived object protected by two lattice-based proofs has diversification within an assumption family but not across families.

**P3 — Evidence should outlive the algorithm that produced it.**
The purpose of a proof is to serve as evidence of authorship and integrity at a point in time. Once that evidence has been independently witnessed, timestamped, or logged, its evidentiary value SHOULD degrade gracefully rather than vanishing the instant the originating algorithm is broken. This requires separating "is this proof cryptographically valid?" from "does independent evidence corroborate this object's existence and content at this time?"

**P4 — Verifier behavior during a transition must be specified, not improvised.**
"It depends what the verifier does" is not an acceptable answer for what happens when one proof in a set validates and another does not. That decision MUST be documented, versioned, and consistent across implementations of the same policy. This document provides a decision matrix (Section 4.2) as a baseline.

**P5 — Migration must be reversible, incremental, and auditable.**
No organization should face a forced choice between "stay on a weakening suite" and "flag-day cutover with no rollback." Dual-proof patterns, re-anchoring, and log-based fallbacks exist precisely to eliminate that false choice. Every migration step SHOULD be independently auditable and SHOULD preserve the ability to verify the object under the *previous* configuration until the new configuration is confirmed stable.

**P6 — Canonicalization is a load-bearing invariant.**
A proof is only as durable as the byte-for-byte reconstruction of what was signed. Any transformation between the protected object's storage form and its signing form is a potential continuity failure point. Systems MUST document and test their canonicalization pipeline as rigorously as their cryptographic operations.

**P7 — The framework is container-agnostic; the bindings are container-specific.**
The patterns described in this document apply regardless of the container format. The specific syntax, data model, and protocol details vary by ecosystem. This document treats all major containers as first-class citizens and actively solicits binding specifications from their respective communities.



## 4. The Universal Pattern Language

This section defines the architectural patterns that constitute cryptographic continuity, expressed independently of any container format. Section 5 provides the container-specific bindings.

### 4.1 Parallel Independent Proofs

**What it is:** Multiple self-contained proofs over the same protected object, each independently verifiable under its own cryptosuite and key material. No proof depends on another for its cryptographic validity, though verifier policy may require a threshold of valid proofs.

**Why it matters:** This is the foundational pattern for all cryptographic continuity. If one proof's algorithm is broken, the others remain independently valid. The protected object does not need to be modified; verifiers simply evaluate the proofs they trust.

**Key properties:**
- Each proof is structurally independent and can be validated in isolation
- Adding or removing a proof does not invalidate other proofs
- Proofs MAY use different canonicalization schemes, but same-canonicalization is preferred where possible (see Section 6.2)
- The order of proofs MAY convey policy intent but does not affect cryptographic validity

**When to use:**
- As the default issuance posture for any protected object with an expected lifetime exceeding 2-3 years
- Whenever an organization is adding a new cryptosuite while still needing backwards compatibility with verifiers that only understand the old one
- When multiple relying parties have different algorithm acceptance policies

**When not to use:**
- In severely size-constrained environments where even the overhead of a second proof is prohibitive (in such cases, composite/hybrid proofs may be more appropriate, or a single proof with aggressive re-anchoring schedules)
- When the container format does not natively support multiple proofs (see Section 5 for workarounds)

### 4.2 Verifier Policy During Transitions

**The core question:** When a protected object carries multiple proofs and not all of them validate, what should the verifier do?

**The answer depends on context,** but the decision must be explicit and documented. This document defines a baseline decision matrix that implementations SHOULD adopt as a starting point, adapting it to their specific threat model and regulatory environment.

| Scenario | Verifier Action |
|----------|-----------------|
| All proofs validate; all suites `active` | **Accept.** Full assurance. |
| At least one proof validates and uses an `active` suite; other proofs fail or use `deprecated` suites | **Accept with reduced assurance.** Record the degraded state in verification output. Downstream systems MAY impose additional requirements. |
| All validating proofs use `deprecated` suites; no proof uses `active` or `broken` | **Accept with warning.** The object's authenticity is cryptographically established but the protecting suites are no longer recommended. Re-anchoring SHOULD be requested. |
| At least one proof uses a `broken` suite; other proofs validate and use `active` suites | **Accept.** The `broken` proof is ignored. The valid active proof(s) establish authenticity. |
| All proofs fail, or all validating proofs use `broken` suites | **Reject for authenticity purposes.** Fall back to independent evidence (Section 4.4) if available. |
| No proof can be parsed or the proof format is unrecognized | **Implementation-dependent.** Legacy verifiers MAY ignore unrecognized proofs and fall back to recognized ones. Security-critical verifiers SHOULD reject if no recognized active proof is present. |

**Default posture:** Security-critical contexts (financial transactions, legal identity, access control) SHOULD default to **fail-closed**: reject unless an active proof validates. Lower-stakes contexts MAY adopt **fail-open-with-warning** but MUST log the decision.

**Policy versioning:** Verifier policies MUST be versioned. When a suite transitions between statuses (e.g., `active` to `deprecated`), the verifier policy SHOULD be updated and the version recorded in verification logs alongside the decision.

### 4.3 Re-Anchoring

**What it is:** Issuing a new proof (or proof set) over a previously protected object, using a currently-trusted suite, before the original suite is retired or broken.

**Why it matters:** Re-anchoring is the operational mechanism that prevents an object from depending indefinitely on an aging cryptosuite. It converts the abstract principle of agility into a scheduled operational practice.

**When to re-anchor:**
- When the original suite transitions from `active` to `monitor`: begin planning
- When the original suite transitions to `deprecated`: re-anchor before the object's next use in a high-stakes context
- When the original suite is scheduled for `broken`: re-anchor immediately, regardless of object usage
- When a new, structurally independent suite becomes available and the object is high-value: consider proactive re-anchoring even if the current suite is still `active`

**What re-anchoring produces:**
- A new proof (or proof set) over the *same* protected object bytes, using the *new* suite and key material
- The original proof(s) remain attached to the object; they are not removed
- The object's semantic content does not change; only the proofs protecting it change

**Proof chains for re-anchoring history:**
When an object is re-anchored multiple times, the sequence of proofs forms a chain of custody. Each re-anchoring proof SHOULD reference the previous proof(s) to establish temporal ordering. This is distinct from a single proof set: a proof set says "these proofs were all applied to this object at roughly the same time"; a proof chain says "this object was re-attested at time T1, then re-attested again at T2, and so on."

### 4.4 Evidence Preservation Beyond Signatures

**The fundamental separation:** Two questions are often conflated:

1. **Authenticity:** Was this object issued by the party it claims, in the form it now has? — Answered by cryptographic proof validation.
2. **Existence:** Did this object exist, in this form, at this time? — Answered by independent witnessing, which survives algorithm breaks.

**Independent witnessing mechanisms:**
- Transparency logs (e.g., Certificate Transparency, Key Transparency, general verifiable logs)
- Timestamping authorities (RFC 3161, ANSI X9.95)
- Blockchain/DLT anchoring (where the consensus mechanism itself has continuity properties)
- Institutional notarization and archival attestation
- Multi-party witnessing (multiple independent entities attest to having seen the object at time T)

**The pattern:**
At or near issuance time, the issuer (or holder, or a designated witness) submits a cryptographic commitment to the protected object—typically a hash—to one or more independent witnessing systems. The witness returns a receipt that can be verified independently of the object's original proof(s). If the original proof(s) later become invalid, the witness receipt still establishes that the object existed in this form at that time.

**What this does not do:**
Witnessing does not replace cryptographic proofs for establishing *who* issued the object. A witness receipt says "this object existed at time T," not "Alice signed this object." If the signing key is compromised and the signature algorithm is broken, witnessing establishes the object's existence but not its authorship. Systems relying on witnessing as a fallback MUST distinguish these two properties in their verification output.

### 4.5 Catastrophic Dual Break Response

**The hard case, stated without euphemism:** There is no cryptographic construction unconditionally safe against every future mathematical or computational advance. A design that assumes otherwise is not more secure—only less honest about its assumptions. When both the classical and post-quantum assumptions protecting an object fail, the object loses provable cryptographic authenticity. The question is not how to prevent this (that is not achievable by any architectural document) but how to ensure the failure is *graceful* rather than *catastrophic*.

**What "both broken" means:**
- **Sequential break:** The classical assumption breaks after the post-quantum assumption in use has already been broken and not yet replaced
- **Simultaneous break:** A single technique (e.g., an unanticipated structural attack with implications for both assumption families, or a supply-chain compromise affecting shared implementation components) invalidates both at once
- **Partial break with uncertainty:** Credible cryptanalytic results exist against both, but practical exploitation is not yet demonstrated. Operationally, this is often the most dangerous case because it is litigated in real time

**The response pattern:**

1. **Fall back to independent evidence, not to another signature algorithm chosen in a hurry.**
   If both protecting assumptions are compromised, the immediate fallback should be evidence that was independent of both *before* the break: transparency log entries, third-party timestamps, institutional witnessing collected at or near issuance time. This is why Section 4.4 recommends log-based anchoring as routine practice now, not as emergency design under pressure.

2. **Prefer hash-based proofs as the structurally orthogonal fallback for re-anchoring.**
   Hash-based signature schemes (SLH-DSA, LMS, XMSS) rest on assumptions about hash function properties that are structurally independent of both algebraic and lattice-based hardness assumptions. A break of both an algebraic and a lattice-based assumption does not, on current understanding, imply a break of a well-chosen hash function. For high-value, long-lived objects, maintaining at least one hash-based proof provides a hedge: the object can be re-anchored under a new suite using the still-valid hash-based proof as the trust anchor.

3. **Re-anchor under institutional, not purely cryptographic, chain of custody.**
   When no cryptosuite is trusted for re-signing, organizations SHOULD fall back to an explicit, disclosed, auditable institutional attestation process—a named responsible party asserting "we hold records showing X was issued by Y at time T," backed by organizational accountability rather than a cryptographic signature. This is a deliberate, temporary, clearly-labeled degradation, not a silent failure. The institutional attestation MUST be versioned, logged, and accompanied by a statement of what cryptographic evidence formerly existed and why it is no longer relied upon.

4. **Treat the break as a coordinated-disclosure event with a pre-prepared runbook.**
   Organizations SHOULD maintain an emergency response process structurally analogous to vulnerability disclosure: notification channels, a triage procedure for identifying affected objects, a communication template for relying parties, and a pre-agreed decision authority for declaring suites `broken`. Designing this process during the emergency is itself a continuity failure.

5. **Distinguish "no longer cryptographically provable" from "presumed false."**
   An object whose proofs are broken has lost a specific property—provable non-repudiation going forward—but this does not retroactively make its contents false, nor does it erase independent evidence collected about it. Verifier policy and legal/operational process MUST treat these as distinct outcomes. Logging systems MUST record the distinction explicitly.

**An explicit limit:**
This pattern does not prevent harm from a dual break—it cannot. Objects that depended *solely* on cryptographic authenticity, with no independent witnessing and no hash-based fallback, will lose that property if both assumptions fail, and no process can retroactively restore it. What this pattern achieves is narrower: the *cost* of a dual break is substantially reduced, and the *response* is substantially de-panicked, by building independent-evidence and hash-based-fallback habits before they are needed, as routine operational practice.



## 5. Container Bindings

This section specifies how the universal patterns manifest in each major container ecosystem. Each binding is maintained in coordination with the relevant standards community. The bindings are intended to be equally authoritative: no container is privileged as the "primary" binding.

### 5.1 W3C Verifiable Credentials Data Integrity

**Container:** Verifiable Credentials secured using Data Integrity proofs, as specified in [[VC-DATA-INTEGRITY]].

**Parallel Independent Proofs (Section 4.1):**
Implemented as a Data Integrity `proof` array containing multiple `DataIntegrityProof` objects, each with its own `cryptosuite`, `verificationMethod`, and `proofValue`. Each proof is independently validatable using the procedure defined for its cryptosuite.

```json
{
  "@context": ["https://www.w3.org/ns/credentials/v2"],
  "type": ["VerifiableCredential"],
  "proof": [
    {
      "type": "DataIntegrityProof",
      "cryptosuite": "eddsa-jcs-2022",
      "verificationMethod": "did:example:issuer#classical-key",
      "proofPurpose": "assertionMethod",
      "proofValue": "z..."
    },
    {
      "type": "DataIntegrityProof",
      "cryptosuite": "mldsa87-jcs-2024",
      "verificationMethod": "did:example:issuer#pq-key",
      "proofPurpose": "assertionMethod",
      "proofValue": "z..."
    }
  ]
}
```

**Canonicalization considerations:**
Data Integrity supports multiple canonicalization schemes, primarily JCS ([[RFC8785]]) and RDFC-1.0 ([[RDF-CANON]]). Same-canonicalization proof sets (e.g., both proofs using JCS) are the recommended baseline: both proofs canonicalize the same document with the same deterministic algorithm, eliminating canonicalization divergence as a failure mode. Mixed-canonicalization proof sets (e.g., one JCS proof, one RDFC proof) are permitted but carry additional risk (see Section 6.2).

**Re-Anchoring (Section 4.3):**
Implemented as issuing a new proof (or proof set) over the same credential, with the new proof(s) referencing the prior proof(s) to form a proof chain. The W3C Data Integrity proof chain mechanism is defined in [[VC-DATA-INTEGRITY]].

**Existing specifications:**
- [[VC-DATA-INTEGRITY]]: Proof sets and proof chains
- [[VC-DI-EDDSA]]: Classical EdDSA cryptosuite
- [[VC-DI-ECDSA]]: Classical ECDSA cryptosuite
- [[VC-DI-QUANTUM-RESISTANT]]: ML-DSA and SLH-DSA cryptosuites

### 5.2 JOSE / COSE

**Container:** JSON Object Signing and Encryption (JOSE) [[RFC7515]] and CBOR Object Signing and Encryption (COSE) [[RFC9052]].

**Parallel Independent Proofs (Section 4.1):**
JOSE and COSE support multiple signatures through their respective multi-signature constructs. In JOSE, a JWS JSON Serialization with multiple signatures in the `signatures` array. In COSE, a `COSE_Sign` structure with multiple `COSE_Signature` objects. Each signature is independently verifiable with its own algorithm identifier and key reference.

```json
{
  "payload": "...",
  "signatures": [
    {
      "protected": "...",
      "header": { "alg": "EdDSA", "kid": "classical-key" },
      "signature": "..."
    },
    {
      "protected": "...",
      "header": { "alg": "ML-DSA-87", "kid": "pq-key" },
      "signature": "..."
    }
  ]
}
```

**Composite/Hybrid alternative:**
JOSE and COSE are also standardizing composite/hybrid algorithms ([[I-D.ietf-jose-pq-composite-sigs]]) that combine classical and post-quantum algorithms into a single atomic signature value. Composite signatures are appropriate when the protocol context is not multi-signature-aware—for example, when a `kid` field is expected to map to a single algorithm, or when constrained token formats (JWS Compact Serialization) do not support multiple signatures. This document treats parallel proofs and composite signatures as complementary, not competing: parallel proofs provide separability and independent verifiability; composite signatures provide drop-in compatibility with single-signature protocols.

**Canonicalization considerations:**
JOSE and COSE signatures are over a well-defined signing input (the JWS Signing Input or COSE Sig_structure), which serves the role of canonicalization. Divergence risk is lower than in container formats that require separate canonicalization steps, but implementers must still ensure identical reconstruction of the signing input at verification time.

### 5.3 X.509 and PKIX

**Container:** X.509 certificates [[RFC5280]] and related PKIX structures.

**Parallel Independent Proofs (Section 4.1):**
X.509 does not natively support multiple signatures on a single certificate in the way that Data Integrity or JOSE do. Parallel independent proofs are achieved through:
- **Multiple certificates for the same key pair but different algorithms:** The same subject public key is certified under both a classical and a post-quantum CA certificate. The relying party validates whichever chain it trusts.
- **Cross-certification:** Two parallel certificate chains using different algorithms, both attesting to the same subject.
- **Composite certificates:** A single certificate whose signature is a composite of classical and post-quantum algorithms, as standardized in [[I-D.ietf-lamps-pq-composite-sigs]].

**Recommendation for the post-quantum transition:**
Where the PKI ecosystem supports parallel certificate issuance, dual-certificate deployment is preferred for its separability. Where protocol or operational constraints require a single certificate, composite certificates provide the continuity guarantee within the single-signature constraint.

**Log-based anchoring:**
Certificate Transparency [[RFC6962]] provides log-based witnessing natively: every certificate is logged. This creates an independent evidence trail that survives algorithm breaks in the certificate's own signature, provided the log's own continuity is maintained.

### 5.4 Web Authentication (WebAuthn)

**Container:** WebAuthn assertions [[WebAuthn-3]].

**Parallel Independent Proofs (Section 4.1):**
WebAuthn supports multiple signatures on a single assertion through the `attObj.authData` extensions mechanism. An authenticator can produce an assertion with multiple `sig` values, each corresponding to a different algorithm, within the extension data. Relying parties validate the signatures they recognize and ignore the rest.

**Transition model:**
During credential registration, an authenticator registers multiple public keys (classical + post-quantum) under a single credential ID. During assertion, the authenticator produces both signatures. Legacy relying parties validate the classical signature; post-quantum-ready relying parties validate both.

**Offline and proximity use:**
WebAuthn's CTAP2 protocol operates over USB, NFC, and BLE, making it directly applicable to the offline scenarios described in Section 5.5.

### 5.5 Offline and Proximity Protocols

**Container:** Protocols operating over local transport (NFC, Bluetooth, QR code, local Wi-Fi) where network-based verification is unavailable or undesirable. Examples include ISO 18013-5 (mobile driver's license), ISO 23220 (mobile identity), FIDO CTAP, and proprietary offline attestation formats.

**Parallel Independent Proofs (Section 4.1):**
The protected object—typically a CBOR-encoded credential or attestation bundle—carries multiple signatures in a COSE-like multi-signature structure. The reader device validates the signatures it trusts based on locally-stored trust anchors and algorithm policies, updated during occasional online synchronization.

**Key considerations:**
- **Size constraints:** NFC transactions are severely size-limited. Post-quantum signatures (ML-DSA-87: ~4.6KB; SLH-DSA: larger) push against these limits. Implementations MAY use a single algorithm with aggressive re-anchoring and short validity periods if dual-proofing exceeds the transport budget, but this trade-off MUST be explicitly documented and the validity period MUST be short enough that a suite transition can be completed before the algorithm weakens.
- **Trust anchor updates:** Offline verifiers must receive updated trust anchor and suite status information during connectivity windows. The verifier's local policy SHOULD degrade gracefully (defaulting to `reject-with-warning` rather than `accept` when trust anchor data is stale beyond a configured threshold).
- **Holder-mediated re-anchoring:** A holder device with intermittent connectivity can request re-anchored credentials from the issuer when online, then present the updated proofs to offline verifiers. This is a distinct re-anchoring model from the issuer-initiated pattern: the holder acts as the re-anchoring trigger.

### 5.6 DNSSEC

**Container:** DNSSEC-signed zones [[RFC4033]], [[RFC4034]], [[RFC4035]].

**Parallel Independent Proofs (Section 4.1):**
DNSSEC supports multiple signatures on a single resource record set through the `RRSIG` records. A zone owner can sign with both a classical algorithm (e.g., ECDSA P-256, algorithm 13) and a post-quantum algorithm (when standardized and assigned an algorithm number), publishing both `RRSIG` records. Validating resolvers validate whichever algorithm(s) their policy recognizes.

**Transition model:**
Algorithm rollover in DNSSEC already follows a multi-step process (add new algorithm, publish both, wait for TTL expiration, remove old algorithm). This existing operational practice maps directly to the re-anchoring pattern in Section 4.3. The post-quantum transition is an instance of this already-understood rollover process, not a new operational challenge.

### 5.7 Firmware and Software Signing

**Container:** Signed firmware images, software packages, and update manifests (UEFI Secure Boot, TUF, Android Verified Boot, iOS Secure Boot, Sigstore, Binary Authorization).

**Parallel Independent Proofs (Section 4.1):**
Multiple signatures on firmware images can be implemented as:
- A signature catalog embedded in the image metadata, each entry containing an algorithm identifier, key reference, and signature value
- A detached signature file with multiple signature blocks
- TUF's multi-signature delegation model, where multiple roles (with different keys and potentially different algorithms) must sign the same target

**Key considerations:**
- **Boot-time verification:** The first-stage bootloader has severe size and complexity constraints. It may only understand one algorithm. In this case, the first-stage bootloader validates the classical signature, and the second-stage bootloader (less constrained) validates the full proof set. This is a layered verification model, not a dual-proof model in the strict sense, but it achieves the same continuity property.
- **Long-term storage:** Firmware images are often archived for decades (medical devices, industrial control systems, automotive ECUs). Log-based anchoring (Section 4.4) is particularly important: the firmware image's hash should be logged at release time so that, decades later, the image's integrity can be verified even if all signature algorithms used at release time are broken.
- **Supply chain integrity:** The Sigstore model (signing certificates with transparency log inclusion proofs) already implements a form of log-based evidence preservation. Extending Sigstore to support post-quantum signature algorithms and dual-proof artifacts is a natural continuity measure.

### 5.8 Transparency Logs and Verifiable Data Structures

**Container:** Append-only verifiable logs (Certificate Transparency, Key Transparency, Trillian, Rekor, general verifiable logs), Merkle trees, and verifiable data structures.

**Parallel Independent Proofs (Section 4.1):**
A transparency log entry can carry multiple proofs. The log's own signature (typically on the Signed Tree Head or equivalent) can be a proof set. The entries within the log can also carry proofs (e.g., a Rekor entry containing a dual-proof artifact).

**The log as continuity infrastructure:**
Transparency logs are both a consumer of continuity patterns (the log itself must maintain continuity across algorithm transitions) and a provider of continuity infrastructure (the log serves as the independent witness for other protected objects, as described in Section 4.4). This dual role means that log operators have a heightened responsibility: if the log itself suffers a continuity failure, every protected object that depended on it for witnessing loses its fallback evidence.

**Log migration:**
When a log transitions algorithms, it follows the same patterns as other containers. The log SHOULD maintain parallel Signed Tree Heads (old algorithm + new algorithm) during the transition window. Log clients SHOULD verify both. Historical log entries SHOULD be re-anchored by including them in new log entries signed under the new algorithm, creating a chain of log-based custody.


## 6. Shared Infrastructure

### 6.1 Algorithm and Suite Status Registry

This document defines the structure and governance requirements for an algorithm/suite status registry. The registry itself is a separate, versioned resource maintained independently of this document's prose, but normatively referenced by it.

**Registry entries MUST include:**
- **Suite identifier:** Uniquely identifying the algorithm or suite (e.g., `eddsa-jcs-2022`, `ML-DSA-87`, `ES256`, `Composite-ML-DSA-87-ECC-P256`)
- **Container binding:** Which container(s) this suite is defined for (may be multiple)
- **Status:** One of `active`, `monitor`, `deprecated`, `broken`
- **Status effective date:** When the current status was assigned
- **Status rationale:** Narrative explaining the reason for the current status, including references to relevant cryptanalytic results or standards body decisions
- **Recommended replacement:** If `deprecated` or `broken`, the suite(s) recommended to replace it
- **Expected transition timeline:** If `monitor`, the anticipated timeline for potential status change

**Governance requirements:**
- The registry MUST be publicly accessible and version-controlled
- Changes to status MUST require a documented decision from a recognized authority (e.g., a standards body, a national cryptographic authority, or a multi-stakeholder governance body)
- Status changes MUST be announced with sufficient lead time for relying parties to update their configurations
- The emergency transition from any status to `broken` MUST have a pre-defined fast-track process

**Relationship to existing registries:**
Multiple algorithm registries already exist (NIST algorithm catalog, IANA JOSE algorithm registry, W3C cryptosuite registry, IETF PKIX algorithm registry). This document does not propose replacing them. It defines a *status overlay*: an additional property—the lifecycle status—that can be associated with entries in any of these registries, maintained in a format that is queryable by automated systems across all container ecosystems.

### 6.2 Canonicalization Conformance

Canonicalization is the most frequent silent failure point in cryptographic continuity. Two implementations of the same canonicalization specification may produce different byte sequences for the same input—and the failure may only manifest for specific edge-case inputs.

**This document recommends:**
- **Shared test vector corpus:** A publicly maintained, versioned corpus of test vectors for each canonicalization scheme, contributed by multiple independent implementations. Each test vector includes the input (in its native format), the expected canonical output bytes, and a description of any edge case exercised.
- **Adversarial test fixtures:** Inputs specifically designed to trigger known divergence points: whole-number floating-point values, very large integers, Unicode normalization boundaries, key ordering edge cases, empty/absent optional fields, and input-length boundary conditions.
- **Dual canonicalization at issuance:** For systems using mixed-canonicalization proof sets, the issuer SHOULD independently canonicalize under both schemes using at least two independent implementations before signing, diffing the results. This catches divergence at issuance time rather than at verification time.
- **Canonicalization disclosure:** A canonicalization mismatch between issuer and verifier SHOULD be treated as a continuity defect, tracked and disclosed with severity proportional to the divergence's impact, even though no key material is compromised.

### 6.3 Test Vectors

This document maintains, as a companion deliverable, a shared corpus of test vectors covering:

- **Parallel proof examples:** Same-object-multiple-proof test vectors for each container binding (Section 5), including both same-canonicalization and mixed-canonicalization cases where applicable
- **Verifier policy test cases:** Test vectors exercising every row of the verifier decision matrix (Section 4.2) for each container binding, including edge cases (proof that parses but uses unknown algorithm, proof with valid signature but expired key, proof set where one proof has been stripped by an intermediary)
- **Re-anchoring sequences:** Multi-step proof chains across multiple suite generations, testing that verifiers correctly validate chains where intermediate links use deprecated suites
- **Catastrophic dual break scenarios:** Simulated dual-break test cases to validate that verifier fallback logic (Section 4.5) functions correctly
- **Canonicalization edge cases:** As described in Section 6.2

Test vectors are accepted from any implementation, regardless of ecosystem or organization, subject to review for correctness and relevance. A test vector contribution that exercises a previously unaddressed edge case is treated as a first-class contribution to this document.


## 7. Threat Model

This document treats the following threats as within scope for continuity planning:

| Threat | Description | Primary Mitigation |
|--------|-------------|-------------------|
| Harvest-now-forge-later | Adversary records signed objects now for forgery once algorithm is broken | Early dual-proofing of long-lived objects |
| Gradual cryptanalytic erosion | Algorithm security margin shrinks incrementally over years | Suite status `monitor`; scheduled re-anchoring |
| Sudden algorithm break | Practical attack published against previously-trusted suite | Verifier policy for partial-validity proof sets; emergency suite deprecation |
| Canonicalization divergence | Independent implementations produce different bytes for same input | Conformance test suites; dual-implementation validation at issuance |
| Implementation compromise | Library or hardware broken even though mathematical assumption is not | Suite status scoped to implementation+version, not only algorithm |
| Key compromise | Signing key exfiltrated or misused | Existing key rotation/revocation; orthogonal to continuity but interacts during migration |
| Format/registry obsolescence | Infrastructure to parse old proof (contexts, registries, method resolution) unavailable | Archival preservation alongside cryptographic continuity |
| Catastrophic dual break | Both classical and post-quantum assumptions fail simultaneously | Independent evidence fallback; hash-based proof hedging; institutional attestation (Section 4.5) |
| Supply chain compromise | Shared implementation component used by multiple suites compromised | Diversification of implementation dependencies; independent implementation verification |


## 8. Operational Deployment Considerations

**Size and performance:**
Post-quantum signatures and keys are substantially larger than their classical counterparts. A proof set combining classical, lattice-based, and hash-based proofs multiplies object size and verification cost accordingly. Deployments MUST budget for this explicitly: model the size of protected objects with the full proof set, measure verification latency under expected load, and test in constrained environments (NFC, embedded bootloaders, high-throughput verification services) before production deployment.

**Key management and hardware constraints:**
PQC key sizes strain existing HSM, smart card, and secure element form factors. Organizations SHOULD inventory hardware constraints as part of migration planning. Where hardware cannot store or process PQC keys, software-based PQC signing with hardware-based classical signing as a co-requirement is an acceptable intermediate posture, with the understanding that the classical hardware provides defense-in-depth while the PQC software provides the post-quantum guarantee.

**Backwards compatibility:**
Legacy verifiers that do not understand multi-proof objects or unfamiliar algorithm identifiers need a defined, safe failure mode. This document recommends: legacy verifiers SHOULD ignore unrecognized proofs and validate only recognized ones (fail-open for unrecognized proof types, fail-closed for recognized proof types with validation failures). Verifier documentation MUST state which behavior is implemented.

**Storage and archival cost:**
Long-term evidence preservation—logs, timestamps, multiple proof generations across re-anchoring events—has non-trivial and growing storage cost. Organizations SHOULD model this cost over the protected object's expected lifetime, not just at issuance. Storage cost modeling SHOULD account for redundancy, geographic distribution, and format migration (the storage format itself may evolve).

**Operational metrics:**
Deployments SHOULD monitor and alert on:
- Percentage of protected objects with only `deprecated` proofs (re-anchoring backlog)
- Verifier-side failures due to unrecognized algorithms (adoption lag)
- Suite status changes in the registry (trigger for re-anchoring campaigns)
- Canonicalization mismatches (potential latent divergence)
- Proof set size approaching transport budget limits


## 9. Stakeholder Guidance

This section provides role-specific guidance, independent of container ecosystem.

| Stakeholder | Primary Concern | Guidance |
|-------------|-----------------|----------|
| **Issuers / Signers** | Producing objects that remain verifiable for their full intended lifetime | Default to dual-proof for long-lived objects; log hashes independently of signing; schedule re-anchoring before suite status reaches `monitor`; maintain an inventory of all long-lived protected objects and their current suite status |
| **Verifiers / Relying Parties** | Consistent, defensible accept/reject decisions during transitions | Adopt an explicit, versioned verifier policy per Section 4.2; log decisions involving non-`active` suites; subscribe to the suite status registry; test with multi-proof objects before they appear in production |
| **Holders / Intermediaries** | Presenting objects that remain acceptable as suite status changes | Support requesting re-anchored proofs; surface suite status to end users when relevant to transaction stakes; preserve witness receipts alongside protected objects |
| **Auditors / Compliance** | Demonstrating that historical records remain defensible evidence | Require documented re-anchoring schedules and suite status registry subscriptions as audit scope; verify that evidence preservation (Section 4.4) is in place for high-value records; test catastrophic-break response plans |
| **Public-sector / Regulators** | Multi-decade trust obligations for civil records | Mandate hash-based fallback proofs and independent log anchoring for records with multi-decade retention; publish suite status expectations aligned with national cryptographic policy; participate in registry governance |
| **Log Operators** | Maintaining continuity for the witnessing infrastructure itself | Follow Section 5.8; maintain parallel Signed Tree Heads during transitions; publish continuity plan including own catastrophic-break response |
| **Library / Platform Maintainers** | Enabling adoption through tooling | Support multi-proof objects natively; surface suite status from registry automatically; provide re-anchoring utilities; publish canonicalization conformance test results |
| **Standards Bodies** | Ensuring ecosystem-wide interoperability | Coordinate algorithm identifiers across containers where possible; align suite status registries; publish container binding specifications; contribute test vectors |


## 10. Living Document Process

This document is expected to evolve continuously rather than being finalized once.

**Update triggers:**
- A new post-quantum algorithm is standardized (e.g., FIPS 206 Falcon)
- A credible cryptanalytic result against a suite currently marked `active` or `monitor` is published
- A new container binding is contributed and reviewed
- Implementation experience reveals a pattern not covered by the current document
- A real-world dual-break or near-miss event occurs and lessons are extracted

**Review cadence:**
The document SHOULD be reviewed at least annually, with additional reviews triggered by material events as listed above.

**Contribution model:**
Contributions of container bindings, test vectors, case studies, and stakeholder guidance are explicitly welcomed from any organization or individual. Contributions are reviewed by the document editors and the broader community. This document's authority derives from the breadth and quality of its contributions, not from the authority of any single standards body.

**Coordination:**
This document coordinates with, but is independent of:
- W3C Verifiable Credentials Working Group (Data Integrity bindings)
- IETF JOSE/COSE/LAMPS working groups (JOSE, COSE, X.509 bindings)
- W3C Web Authentication Working Group (WebAuthn binding)
- ISO/IEC JTC 1/SC 17 and SC 27 (offline protocol bindings)
- NIST Cryptographic Technology Group (algorithm status)
- IETF DNS Operations Working Group (DNSSEC binding)
- Cloud Native Computing Foundation / Sigstore (firmware signing, transparency log bindings)



## 11. Security and Privacy Considerations

**Linkability through proof sets:**
Multi-proof objects carry more metadata (multiple key references, algorithm identifiers, proof values) than single-proof objects. This increases the linkable surface area of the protected object. In privacy-sensitive contexts (selective disclosure, unlinkable presentation), the additional linkability MUST be weighed against the continuity benefit.

**Key reuse across contexts:**
Keys used in a single-algorithm context MUST NOT be reused in a composite/hybrid context without explicit analysis, due to stripping-attack risk (an adversary strips the composite signature and presents the remaining single-algorithm signature as valid in a context that does not expect the composite). This guidance aligns with [[I-D.ietf-lamps-pq-composite-sigs]].

**Log/witness infrastructure dependencies:**
Independent witnessing (Section 4.4) introduces additional trust dependencies: the log operator, the timestamping authority, the consensus mechanism. These dependencies MUST be documented and their own continuity properties assessed. A witness whose own cryptographic continuity is weaker than the object it is witnessing provides a false sense of security.

**Registry integrity:**
The suite status registry (Section 6.1) is a trust root. Its integrity, availability, and governance—including who may mark a suite `broken` and under what emergency conditions—deserve the same scrutiny as any other trust root in the system. Verifier policies SHOULD include a configurable "maximum registry staleness" beyond which the verifier treats all suites conservatively (e.g., as if downgraded one status level).

**Algorithm announcement timing:**
Public announcement of a `broken` suite status may accelerate attacks against objects that have not yet been re-anchored. The coordinated-disclosure model (Section 4.5, item 4) SHOULD account for a window during which major relying parties are notified before public announcement, consistent with standard vulnerability disclosure practices.


## 12. Acknowledgements

This document was informed by public discussion on the W3C Credentials Community Group mailing list and by contributions from implementers, cryptographers, and standards engineers across multiple ecosystems.

The editor thanks: Tokachi Kamimura for migration semantics and operational patterns; Sylvain Cormier for canonical-byte stability analysis; Greg Bernstein for proof-set examples; Stephen Curran for log-based verification perspective; Jori Lehtinen for algorithm-agnostic architecture input; Manu Sporny for Data Integrity binding details; and the did:trail contributors for dual-proof deployment experience.

Additional contributors from JOSE, COSE, X.509, WebAuthn, DNSSEC, firmware signing, and transparency log communities are explicitly sought to expand the container bindings in Section 5 and the test vectors in Section 6.3.



## References

### Normative References

- [[VC-DATA-INTEGRITY]]: Verifiable Credential Data Integrity 1.0
- [[RFC7515]]: JSON Web Signature (JWS)
- [[RFC9052]]: CBOR Object Signing and Encryption (COSE)
- [[RFC5280]]: Internet X.509 Public Key Infrastructure Certificate and CRL Profile
- [[WebAuthn-3]]: Web Authentication: An API for accessing Public Key Credentials Level 3
- [[RFC4033]], [[RFC4034]], [[RFC4035]]: DNS Security Introduction, Resource Records, Protocol Modifications
- [[RFC8785]]: JSON Canonicalization Scheme (JCS)
- [[RDF-CANON]]: RDF Dataset Canonicalization

### Informative References

- [[NIST-IR-8547]]: Transition to Post-Quantum Cryptography Standards
- [[FIPS203]]: Module-Lattice-Based Key-Encapsulation Mechanism Standard
- [[FIPS204]]: Module-Lattice-Based Digital Signature Standard
- [[FIPS205]]: Stateless Hash-Based Digital Signature Standard
- [[I-D.ietf-lamps-pq-composite-sigs]]: Composite ML-DSA Signatures for X.509
- [[I-D.ietf-jose-pq-composite-sigs]]: Composite ML-DSA Signatures for JOSE and COSE
- [[VC-DI-EDDSA]]: Data Integrity EdDSA Cryptosuites
- [[VC-DI-ECDSA]]: Data Integrity ECDSA Cryptosuites
- [[VC-DI-QUANTUM-RESISTANT]]: Data Integrity Quantum-Resistant Cryptosuites
- [[PQT-HYBRID-TERM]]: Terminology for Post-Quantum Traditional Hybrid Schemes
- [[RFC6962]]: Certificate Transparency
- [[VC-FORGERY-DEFENSE]]: Verifiable Credentials Forgery Defense
- [[DID-TRAIL]]: did:trail DID Method

