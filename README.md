

# Cryptographic Continuity: A Reference Architecture for Long-Term Trust Across Cryptographic Transitions

## W3C Community Group Note

**Status:** CG-DRAFT  
**Editors:** Amir Hameed Mir (Sirraya Lab)  
**Repository:** w3c-cg/cryptographic-continuity  
**Latest Draft:** https://w3c-cg.github.io/cryptographic-continuity

---

## Abstract

Confidentiality, integrity, authenticity, availability, and non-repudiation are commonly treated as enduring security properties. But for systems expected to operate across decades—land titles, medical device firmware, audit logs, civil identity records—there is another property that becomes equally important: **the ability to preserve trust as the cryptographic assumptions protecting that trust evolve.** This document names that property **cryptographic continuity**, provides a reference architecture for achieving it, and demonstrates through container-specific bindings that this architecture is not a proposal for new mechanisms but a formalization of patterns already emerging independently across deployed systems.

Existing specifications define how to produce proofs. This document defines the architectural patterns governing how proofs evolve across time—and provides an evaluation framework for assessing whether a system exhibits the property.

It does not define new cryptographic algorithms. It does not modify existing container specifications. It does not mandate specific suites. It provides the missing architectural layer between algorithm selection and operational procedure.

---

## 1. Introduction

### 1.1 The Property Problem

A diploma issued today may need to remain verifiable for a professional lifetime. A land title may need to outlast multiple generations of cryptographic standards. An audit log protecting public-interest data may be subpoenaed decades after the algorithms that signed it are retired. A firmware image in a medical device may need to be authenticated long after its manufacturer has ceased operations.

These scenarios share a structural property: the cryptographic tools used to establish trust at issuance will not be the tools available at verification time. Confidentiality, integrity, authenticity, availability, and non-repudiation describe what a system guarantees *at a point in time*. They say nothing about what happens *as time passes and the cryptographic assumptions underlying those guarantees evolve.*

**Cryptographic continuity** is the property that a system preserves verifiability, integrity, and evidentiary value across changes in cryptographic algorithms, suites, or key material. It is not a replacement for authenticity or integrity. It is a meta-property that governs how authenticity and integrity are maintained over time horizons that exceed algorithm lifetimes.

### 1.2 The Architectural Thesis

This document advances a specific claim:

> Cryptographic continuity is not a feature that any single standard designed for. It is an architectural property that can be observed emerging independently across W3C Data Integrity, JOSE, COSE, PKIX, DNSSEC, firmware signing, Sigstore, and transparency log ecosystems—each converging on similar responses to the same underlying constraint: **algorithms age faster than the records they protect.**

This document identifies the recurring architectural patterns that constitute this convergence, formalizes them as a reference architecture, and provides an evaluation framework for assessing whether a given system exhibits the property.

It does not invent new mechanisms. It names and structures what already exists, implicitly, across independent ecosystems.

### 1.3 Scope

This document covers:

- Cryptographic continuity as a first-class architectural property
- Universal patterns for maintaining verifiability across cryptographic transitions
- Container-specific bindings showing how those patterns manifest in major ecosystems
- An evaluation framework for assessing continuity
- Verifier behavior specification during transition windows
- Evidence preservation strategies that survive algorithm breaks
- The catastrophic dual-break case and graceful degradation
- Shared infrastructure: algorithm status registries, test vectors, conformance guidance

### 1.4 Out of Scope

- Defining new cryptographic algorithms or primitives
- Replacing or modifying existing container specifications
- Mandating specific algorithms or establishing regulatory requirements
- Encryption continuity (harvest-now-decrypt-later for confidentiality is addressed through analogous patterns in encryption-specific documents)

### 1.5 How to Read This Document

- **Architects and policy makers:** Design Principles (Section 3), Universal Patterns (Section 4), Evaluation Framework (Section 5)
- **Implementers:** Your ecosystem's Container Binding (Section 6), Shared Infrastructure (Section 7)
- **Auditors and compliance officers:** Verifier Policy (Section 4.2), Evidence Preservation (Section 4.4), Evaluation Framework (Section 5), Stakeholder Guidance (Section 10)
- **Everyone:** Catastrophic Dual Break Response (Section 4.5)

---

## 2. Terminology

### 2.1 Core Concepts

**Cryptographic Continuity**
: The property that a system preserves the verifiability, integrity, and evidentiary value of protected objects across changes in cryptographic algorithms, suites, or key material.

**Protected Object**
: Any digital artifact whose authenticity or integrity is established through cryptographic means. Examples: a Verifiable Credential, a JWT, an X.509 certificate, a DNSSEC-signed zone, a signed firmware image, a transparency log entry, a WebAuthn assertion, a COSE-signed message.

**Cryptographic Proof**
: A cryptographic assertion of authenticity and/or integrity over a protected object. Used generically across ecosystems: a Data Integrity proof in W3C, a JWS signature in JOSE, a Certificate signature in X.509, an assertion signature in WebAuthn. This document uses "proof" to mean "cryptographic proof" throughout; it does not address zero-knowledge proofs.

### 2.2 Cryptographic Primitives and Suites

**Algorithm**
: A mathematical primitive for signing or verification (e.g., Ed25519, ML-DSA-87, ECDSA P-256).

**Scheme**
: An algorithm plus its operational parameters and encoding rules (e.g., EdDSA with Ed25519 curve and RFC 8032 encoding).

**Cryptosuite**
: A registered combination of canonicalization algorithm, hash function, and signature scheme as a deployable unit (e.g., `eddsa-jcs-2022`, `mldsa87-jcs-2024`, `ES256` in JOSE). This is the granularity at which suite status is assigned.

### 2.3 Proof Structures

**Proof Set**
: Two or more independent cryptographic proofs over the same protected object, each using a different cryptosuite or key, allowing the object to be verifiable under multiple assumptions simultaneously. Container-specific equivalents: `proof` array in W3C Data Integrity, `signatures` array in JOSE JSON Serialization, multiple certificates in X.509 cross-certification.

**Dual-Proof**
: A proof set containing exactly one classical and one post-quantum proof. The recommended default posture for new long-lived protected objects during the current post-quantum transition.

**Proof Chain**
: An ordered sequence of proofs in which each proof (after the first) commits to the preceding proof, expressing sequential re-attestation over time. Container-specific: W3C proof chains, countersignatures in JOSE/COSE, certificate chains in X.509.

**Composite / Hybrid Proof**
: A proof combining two or more algorithms into a single atomic value (e.g., Composite ML-DSA in X.509). Distinct from a proof set: composite proofs are not separable after issuance. Both patterns are addressed in this document.

### 2.4 Lifecycle States

**Suite Status**
: A lifecycle state assigned to a cryptosuite:

| Status | Meaning |
|--------|---------|
| `active` | Currently recommended. No known weaknesses reducing security below intended level. |
| `monitor` | Under active cryptanalysis. Results warrant attention but do not constitute a practical break. Migration planning SHOULD begin. |
| `deprecated` | Superseded but not known broken. New objects SHOULD NOT use this suite. Existing objects SHOULD be re-anchored. |
| `broken` | No longer provides the security property it was relied upon for. Proofs using this suite MUST NOT be treated as establishing authenticity. |

**Re-Anchoring**
: Producing a new proof (or proof set) over a previously issued protected object using a currently-trusted suite, before the original suite is retired or broken, so the object's verifiability does not depend solely on a suite scheduled for deprecation.

### 2.5 Threat-Specific Terms

**Harvest-Now-Decrypt-Later (HNDL)**
: An adversary strategy of recording protected material today for future attack. The signature analogue—harvest-now-forge-later—is the recording of signed objects today to forge signatures once the algorithm is broken.

**Catastrophic Dual Break**
: A scenario in which both a classical assumption and a post-quantum assumption protecting the same object fail—whether simultaneously, in close succession, or by a single technique effective against both.

---

## 3. Design Principles

These principles hold for the current post-quantum transition and for every subsequent transition, driven by causes not yet known.

**P1 — Agility is the default, not the exception.**
Systems MUST be designed assuming the cryptosuite in use today will not be the cryptosuite in use at the end of a protected object's useful life. The ability to add, remove, and replace proofs without invalidating the protected object is a baseline architectural requirement.

**P2 — No single mathematical assumption should be a single point of failure for long-lived, high-consequence records.**
Where a protected object's lifetime exceeds the expected lifetime of a cryptographic assumption, the object SHOULD be protected by at least one proof based on a structurally independent assumption. Current assumption families: algebraic (elliptic curve, finite field), lattice-based (ML-DSA, ML-KEM), and hash-based (SLH-DSA, LMS/XMSS). Diversification within a family (e.g., two lattice-based proofs) provides partial but not full independence.

**P3 — Evidence should outlive the algorithm that produced it.**
Once a proof has been independently witnessed, timestamped, or logged, its evidentiary value SHOULD degrade gracefully rather than vanishing when the originating algorithm is broken. Systems MUST separate "is this proof cryptographically valid?" from "does independent evidence corroborate this object's existence and content at this time?"

**P4 — Verifier behavior during a transition must be specified, not improvised.**
The decision of what happens when one proof in a set validates and another does not MUST be documented, versioned, and consistent across implementations of the same policy. Section 4.2 provides a baseline decision matrix.

**P5 — Migration must be reversible, incremental, and auditable.**
No organization should face a forced choice between "stay on a weakening suite" and "flag-day cutover with no rollback." Every migration step SHOULD be independently auditable and SHOULD preserve the ability to verify under the previous configuration until the new configuration is confirmed stable.

**P6 — Canonicalization is a load-bearing invariant.**
A proof is only as durable as the byte-for-byte reconstruction of what was signed. Systems MUST document and test their canonicalization pipeline as rigorously as their cryptographic operations.

**P7 — The reference architecture is container-agnostic; the bindings are container-specific.**
The patterns described in this document apply regardless of container format. Specific syntax and protocol details vary by ecosystem. This document treats all major containers as first-class citizens and actively solicits binding specifications from their communities.

---

## 4. Universal Patterns

This section defines the architectural patterns that constitute cryptographic continuity, expressed independently of any container format. Section 6 provides ecosystem-specific bindings.

### 4.1 Parallel Independent Proofs

Multiple self-contained cryptographic proofs over the same protected object, each independently verifiable under its own cryptosuite and key material. If one proof's algorithm is broken, the others remain valid. The protected object does not need modification; verifiers evaluate the proofs they trust.

**Key properties:**
- Each proof is structurally independent and can be validated in isolation
- Adding or removing a proof does not invalidate other proofs
- Proofs MAY use different canonicalization schemes, but same-canonicalization is preferred (see Section 7.2)
- Proof order MAY convey policy intent but does not affect cryptographic validity

**When to use:**
- Default issuance posture for protected objects with expected lifetimes exceeding 2–3 years
- When adding a new cryptosuite while maintaining backwards compatibility
- When multiple relying parties have different algorithm acceptance policies

**When composite/hybrid signatures may be more appropriate:**
- Severely size-constrained environments where a second full proof is prohibitive
- Container formats that do not natively support multiple proofs (single-signature protocols)

### 4.2 Verifier Policy During Transitions

When a protected object carries multiple proofs and not all validate, the verifier's decision must be explicit, documented, and versioned. The following matrix is a **baseline for adaptation**—not a universal mandate. Deployments will justifiably choose different points based on their threat model and regulatory environment.

| Scenario | Verifier Action |
|----------|-----------------|
| All proofs validate; all suites `active` | **Accept.** Full assurance. |
| At least one proof validates and uses an `active` suite; other proofs fail or use `deprecated` suites | **Accept with reduced assurance.** Record degraded state in verification output. Downstream systems MAY impose additional requirements. |
| All validating proofs use `deprecated` suites; no proof uses `active` or `broken` | **Accept with warning.** Authenticity is cryptographically established but suites are no longer recommended. Re-anchoring SHOULD be requested. |
| At least one proof uses a `broken` suite; other proofs validate and use `active` suites | **Accept.** The `broken` proof is ignored. Valid active proofs establish authenticity. |
| All proofs fail, or all validating proofs use `broken` suites | **Reject for authenticity purposes.** Fall back to independent evidence (Section 4.4) if available. |
| No proof can be parsed or proof format is unrecognized | **Implementation-dependent.** Legacy verifiers MAY ignore unrecognized proofs and fall back to recognized ones. Security-critical verifiers SHOULD reject if no recognized active proof is present. |

**Default posture:** Security-critical contexts (financial transactions, legal identity, access control) SHOULD default to **fail-closed**: reject unless an active proof validates. Lower-stakes contexts MAY adopt **fail-open-with-warning** but MUST log the decision.

**Policy versioning:** Verifier policies MUST be versioned. When a suite transitions between statuses, the policy SHOULD be updated and the version recorded in verification logs alongside the decision.

### 4.3 Re-Anchoring

Issuing a new proof (or proof set) over a previously protected object using a currently-trusted suite, before the original suite is retired or broken. Re-anchoring converts the abstract principle of agility into a scheduled operational practice.

**Re-anchoring triggers:**

| Original Suite Status | Action |
|----------------------|--------|
| Transitions to `monitor` | Begin migration planning |
| Transitions to `deprecated` | Re-anchor before next high-stakes use |
| Scheduled for `broken` | Re-anchor immediately, regardless of object usage |
| New structurally independent suite available, object is high-value | Consider proactive re-anchoring even if current suite is `active` |

**What re-anchoring produces:**
- A new proof over the *same* protected object bytes, using the *new* suite and key material
- Original proofs remain attached; they are not removed
- The object's semantic content does not change; only the proofs protecting it change

**Proof chains for re-anchoring history:**
When an object is re-anchored multiple times, the sequence forms a chain of custody. Each re-anchoring proof SHOULD reference the previous proof to establish temporal ordering. A proof set says "these proofs were applied at roughly the same time"; a proof chain says "this object was re-attested at T1, T2, and T3."

### 4.4 Evidence Preservation Beyond Signatures

Two questions are often conflated and must be separated:

1. **Authenticity:** Was this object issued by the party it claims, in the form it now has? Answered by cryptographic proof validation.
2. **Existence:** Did this object exist, in this form, at this time? Answered by independent witnessing, which survives algorithm breaks.

**Independent witnessing mechanisms:**
- Transparency logs (Certificate Transparency, Key Transparency, Rekor, general verifiable logs)
- Timestamping authorities (RFC 3161, ANSI X9.95)
- Blockchain/DLT anchoring (where the consensus mechanism has continuity properties)
- Institutional notarization and archival attestation
- Multi-party witnessing

**The pattern:**
At or near issuance time, a cryptographic commitment to the protected object (typically a hash) is submitted to one or more independent witnessing systems. The witness returns a receipt verifiable independently of the object's original proofs. If the original proofs later become invalid, the witness receipt still establishes the object's existence and content at that time.

**What witnessing does not do:**
A witness receipt says "this object existed at time T," not "Alice signed this object." If the signing key is compromised and the signature algorithm is broken, witnessing establishes existence but not authorship. Systems relying on witnessing as a fallback MUST distinguish these properties in verification output.

### 4.5 Catastrophic Dual Break Response

There is no cryptographic construction unconditionally safe against every future mathematical advance. A design assuming otherwise is not more secure—only less honest about its assumptions. When both the classical and post-quantum assumptions protecting an object fail, the object loses provable cryptographic authenticity. The question is not how to prevent this—that is not architecturally achievable—but how to ensure the failure is *graceful* rather than *catastrophic*.

**What "both broken" means:**
- **Sequential break:** The classical assumption breaks after the post-quantum assumption in use has already been broken and not yet replaced
- **Simultaneous break:** A single technique invalidates both at once (e.g., an unanticipated structural attack spanning assumption families, or a supply-chain compromise of a shared implementation component)
- **Partial break with uncertainty:** Credible results exist against both, but practical exploitation is not yet demonstrated. Operationally, this is often the most dangerous case because it is litigated in real time

**The response pattern:**

**1. Fall back to independent evidence, not to a signature algorithm chosen in a hurry.**
The immediate fallback should be evidence independent of both broken assumptions *before* the break: transparency log entries, third-party timestamps, institutional witnessing collected at or near issuance. This is why Section 4.4 recommends log-based anchoring as routine practice, not emergency design.

**2. Use hash-based proofs as the structurally orthogonal fallback for re-anchoring.**
SLH-DSA, LMS, and XMSS rest on hash function properties structurally independent of algebraic and lattice-based hardness assumptions. A break of both algebraic and lattice assumptions does not, on current understanding, imply a break of a well-chosen hash function. For high-value, long-lived objects, maintaining at least one hash-based proof provides a hedge: the object can be re-anchored using the still-valid hash-based proof as the trust anchor. This is one example of assumption-diverse fallback design; it should not be interpreted as a claim that hash-based signatures are unconditionally permanent.

**3. Re-anchor under institutional chain of custody when cryptographic re-signing is impossible.**
When no cryptosuite is trusted, organizations SHOULD fall back to an explicit, disclosed, auditable institutional attestation—a named responsible party asserting "we hold records showing X was issued by Y at time T," backed by organizational accountability. This is a deliberate, temporary, clearly-labeled degradation. The attestation MUST be versioned, logged, and accompanied by a statement of what cryptographic evidence formerly existed and why it is no longer relied upon.

**4. Treat the break as a coordinated-disclosure event with a pre-prepared runbook.**
Organizations SHOULD maintain an emergency response process structurally analogous to vulnerability disclosure: notification channels, triage procedure for identifying affected objects, communication template for relying parties, and a pre-agreed decision authority for declaring suites `broken`. Designing this process during the emergency is itself a continuity failure.

**5. Distinguish "no longer cryptographically provable" from "presumed false."**
An object whose proofs are broken has lost provable non-repudiation going forward—but this does not retroactively make its contents false, nor does it erase independent evidence. Verifier policy and legal/operational process MUST treat these as distinct outcomes. Logging systems MUST record the distinction.

**An explicit limit:**
This pattern does not prevent harm from a dual break—it cannot. Objects depending solely on cryptographic authenticity, with no independent witnessing and no hash-based fallback, will lose that property if both assumptions fail. What this pattern achieves is narrower: the *cost* is substantially reduced, and the *response* is substantially de-panicked, by building independent-evidence and hash-based-fallback habits before they are needed, as routine operational practice.

---

## 5. Continuity Evaluation Framework

If cryptographic continuity is a first-class architectural property, it should be evaluable. This section provides a framework for assessing whether a system—or a deployment of a system—exhibits the property. The framework does not mandate a minimum score; it mandates awareness of the score and explicit acknowledgment of gaps.

### 5.1 Evaluation Dimensions

| # | Dimension | Question |
|---|-----------|----------|
| 1 | **Parallel Proofs** | Can the system attach multiple independent cryptographic proofs to the same protected object? |
| 2 | **Assumption Diversity** | Can proofs rest on structurally independent mathematical assumptions (algebraic, lattice-based, hash-based)? |
| 3 | **Re-Anchoring** | Can the system issue new proofs over existing objects without changing the objects' semantic content? |
| 4 | **Evidence Preservation** | Does the system support independent witnessing that survives algorithm breaks? |
| 5 | **Verifier Policy** | Is verifier behavior during partial-validity scenarios explicitly specified, versioned, and auditable? |
| 6 | **Suite Lifecycle Management** | Is there a defined process for transitioning suites through active/monitor/deprecated/broken states? |
| 7 | **Catastrophic Recovery** | Is there a pre-defined, tested response when all protecting assumptions fail? |
| 8 | **Canonicalization Stability** | Is byte-for-byte reconstruction of what was signed guaranteed across implementations and time? |
| 9 | **Key and Implementation Diversity** | Are protections in place against single-implementation or single-hardware compromise? |

### 5.2 Scoring Guidance

Each dimension is assessed as **satisfied**, **partially satisfied**, or **not satisfied** based on the system's architecture and deployed configuration. A system scoring "not satisfied" on a dimension has an acknowledged continuity gap. Whether that gap is acceptable depends on the protected object's value, lifetime, and threat model.

The framework does not prescribe a passing threshold. It requires that:
- The score is documented
- Gaps are explicitly acknowledged
- The rationale for accepting each gap is recorded
- The score is reviewed when the threat environment changes

### 5.3 Example Assessment

*This section will be populated with representative assessments of deployed systems (Sigstore, DNSSEC, W3C Data Integrity deployments) as community contributions are received.*

---

## 6. Container Bindings

This section demonstrates how the universal patterns manifest in each major ecosystem. Each binding is maintained in coordination with the relevant standards community. No container is privileged as the "primary" binding.

### 6.1 W3C Verifiable Credentials Data Integrity

**Parallel Proofs:** Data Integrity `proof` array containing multiple `DataIntegrityProof` objects, each with independent `cryptosuite`, `verificationMethod`, and `proofValue`. Each proof is independently validatable.

**Canonicalization:** Supports JCS ([[RFC8785]]) and RDFC-1.0 ([[RDF-CANON]]). Same-canonicalization proof sets (e.g., both JCS) are the recommended baseline. Mixed-canonicalization sets are permitted but carry additional risk (see Section 7.2).

**Re-Anchoring:** Implemented via proof chains referencing prior proofs.

**Existing specifications:** [[VC-DATA-INTEGRITY]], [[VC-DI-EDDSA]], [[VC-DI-ECDSA]], [[VC-DI-QUANTUM-RESISTANT]].

**Evaluation notes:** Parallel proofs, re-anchoring, and assumption diversity are natively supported. Evidence preservation requires integration with external witnessing infrastructure.

### 6.2 JOSE / COSE

**Parallel Proofs:** JWS JSON Serialization with multiple `signatures` entries; COSE_Sign with multiple COSE_Signature objects. Compact serializations do not support multiple signatures, creating a constraint where composite signatures ([[I-D.ietf-jose-pq-composite-sigs]]) provide the continuity guarantee within single-signature protocols.

**Canonicalization:** JWS Signing Input and COSE Sig_structure serve as canonicalization. Divergence risk is lower than in formats requiring separate canonicalization, but signing input reconstruction must be identical at verification time.

**Composite/Hybrid:** Treated as complementary to parallel proofs—composite for drop-in compatibility, parallel for separability and independent verifiability.

**Existing specifications:** [[RFC7515]], [[RFC9052]], [[I-D.ietf-jose-pq-composite-sigs]].

**Evaluation notes:** Parallel proofs supported in JSON/CBOR serializations but not compact. Composite signatures bridge the gap. Evidence preservation requires external witnessing.

### 6.3 X.509 and PKIX

**Parallel Proofs:** X.509 does not natively support multiple signatures on a single certificate. Parallel trust is achieved through multiple certificates for the same key pair under different algorithms, cross-certification, or composite certificates ([[I-D.ietf-lamps-pq-composite-sigs]]).

**Recommendation:** Dual-certificate deployment preferred for separability where PKI operations support it. Composite certificates provide continuity within single-certificate constraints.

**Log-based anchoring:** Certificate Transparency [[RFC6962]] provides native log-based witnessing.

**Existing specifications:** [[RFC5280]], [[RFC6962]], [[I-D.ietf-lamps-pq-composite-sigs]].

**Evaluation notes:** Native log anchoring is a strength. Parallel proofs require operational workarounds. Composite signatures fill the single-certificate gap.

### 6.4 Web Authentication (WebAuthn)

**Parallel Proofs (illustrative):** A WebAuthn authenticator *could* produce an assertion with multiple signature values through the extensions mechanism. As of this writing, this pattern is not standardized in [[WebAuthn-3]]. It is included as input to the WebAuthn Working Group's post-quantum planning.

**Transition model (illustrative):** During registration, an authenticator registers multiple public keys under a single credential ID. During assertion, it produces both signatures. Legacy relying parties validate the classical signature; post-quantum-ready relying parties validate both.

**Offline applicability:** CTAP2 operates over USB, NFC, and BLE, making it directly applicable to offline scenarios (Section 6.5).

**Existing specifications:** [[WebAuthn-3]].

**Evaluation notes:** Patterns are largely illustrative pending standardization.

### 6.5 Offline and Proximity Protocols

**Container:** ISO 18013-5 (mDL), ISO 23220, FIDO CTAP, proprietary offline attestation formats over NFC, Bluetooth, QR code.

**Parallel Proofs:** CBOR-encoded credential carrying multiple signatures in a COSE-like structure. Reader devices validate based on locally-stored trust anchors and algorithm policies, updated during occasional online synchronization.

**Key considerations:** NFC size constraints severely limit multi-proof payloads. Implementations MAY use single-algorithm with aggressive re-anchoring and short validity periods if dual-proofing exceeds transport budget. This trade-off MUST be documented; the validity period MUST be short enough that suite transition can complete before algorithm weakening. Holder-mediated re-anchoring—where the holder requests updated proofs from the issuer when online—is a distinct model from issuer-initiated re-anchoring.

### 6.6 DNSSEC

**Parallel Proofs:** Multiple `RRSIG` records over the same RRset, each with a different algorithm. Validating resolvers validate whichever algorithms their policy recognizes.

**Transition model:** Algorithm rollover already follows a multi-step process (add new, publish both, wait TTL, remove old) that maps directly to re-anchoring. The post-quantum transition is an instance of this already-understood process.

**Existing specifications:** [[RFC4033]], [[RFC4034]], [[RFC4035]].

**Evaluation notes:** DNSSEC's existing algorithm rollover is one of the most mature implementations of the re-anchoring pattern in deployed infrastructure.

### 6.7 Firmware and Software Signing

**Container:** UEFI Secure Boot, TUF, Android Verified Boot, iOS Secure Boot, Sigstore, Binary Authorization.

**Parallel Proofs:** Signature catalogs in image metadata, detached signature files with multiple blocks, TUF multi-signature delegations. Boot-time verification may use layered verification: first-stage bootloader validates classical signature; second-stage validates full proof set.

**Sigstore:** The Sigstore model already implements log-based evidence preservation (Rekor inclusion proofs). Extending to post-quantum signatures and dual-proof artifacts is a natural continuity measure. Sigstore is the strongest deployed example of multiple continuity patterns operating simultaneously: short-lived certificates, transparency anchoring, multi-signature support, and active post-quantum migration work.

**Long-term storage:** Firmware images archived for decades (medical, industrial, automotive) particularly benefit from log-based anchoring at release time.

### 6.8 Transparency Logs and Verifiable Data Structures

**Dual role:** Transparency logs are both consumers of continuity patterns (the log itself must maintain continuity) and providers of continuity infrastructure (the log serves as independent witness for other protected objects). This dual role imposes heightened responsibility: if the log suffers a continuity failure, every object depending on it for witnessing loses fallback evidence.

**Parallel Proofs:** Log entries can carry multiple proofs. Signed Tree Heads can be proof sets.

**Log migration:** Maintain parallel STHs during transitions. Log clients verify both. Historical entries re-anchored through inclusion in new log entries under new algorithms.

**Existing specifications:** [[RFC6962]], Rekor, Trillian.

---

## 7. Shared Infrastructure

### 7.1 Algorithm and Suite Status Registry

A *status overlay* for existing algorithm registries (NIST catalog, IANA JOSE registry, W3C cryptosuite registry, IETF PKIX registry). It does not replace them; it adds lifecycle status queryable across all container ecosystems.

**Registry entries include:** suite identifier, container binding(s), status, status effective date, rationale with cryptanalytic references, recommended replacement if deprecated/broken, expected transition timeline if monitor.

**Governance:** Publicly accessible, version-controlled. Status changes require documented decision from a recognized authority. Changes announced with lead time. Emergency transition to `broken` has a pre-defined fast-track process.

**Verifier integration:** Verifier policies SHOULD include a configurable "maximum registry staleness" beyond which the verifier treats all suites conservatively (as if downgraded one status level).

### 7.2 Canonicalization Conformance

The most frequent silent failure point in cryptographic continuity. Two implementations of the same canonicalization specification may produce different byte sequences for specific edge-case inputs.

**Recommended practices:**
- Shared, versioned test vector corpus with adversarial fixtures targeting known divergence points: whole-number floats, large integers, Unicode normalization boundaries, key ordering, empty/absent optional fields
- Dual-implementation canonicalization at issuance for mixed-canonicalization proof sets: canonicalize independently under both schemes using at least two implementations, diff results before signing
- Canonicalization mismatch treated as a continuity defect with tracked disclosure, even though no key material is compromised
- Test fixture versioning alongside cryptosuite registrations

### 7.3 Test Vectors

A companion deliverable to this document, accepting contributions from any implementation regardless of ecosystem:

- Parallel proof examples for each container binding
- Verifier policy test cases covering every row of the decision matrix (Section 4.2), including edge cases
- Re-anchoring sequences across multiple suite generations
- Catastrophic dual break simulation cases
- Canonicalization edge cases per Section 7.2

A test vector exercising a previously unaddressed edge case is a first-class contribution.

---

## 8. Threat Model

| Threat | Description | Primary Mitigation |
|--------|-------------|-------------------|
| Harvest-now-forge-later | Recording signed objects now for forgery once algorithm is broken | Early dual-proofing of long-lived objects |
| Gradual cryptanalytic erosion | Security margin shrinks incrementally | Suite status `monitor`; scheduled re-anchoring |
| Sudden algorithm break | Practical attack published | Verifier policy for partial-validity proof sets; emergency deprecation |
| Canonicalization divergence | Implementations produce different bytes for same input | Conformance test suites; dual-implementation validation at issuance |
| Implementation compromise | Library/hardware broken, mathematical assumption intact | Suite status scoped to implementation+version |
| Key compromise | Signing key exfiltrated | Existing key rotation/revocation; interacts with continuity during migration |
| Format/registry obsolescence | Infrastructure to parse old proofs unavailable | Archival preservation alongside cryptographic continuity |
| Catastrophic dual break | Both classical and post-quantum assumptions fail | Independent evidence fallback; hash-based hedging; institutional attestation (Section 4.5) |
| Supply chain compromise | Shared component used by multiple suites compromised | Diversification of implementation dependencies |

---

## 9. Operational Deployment Considerations

**Size and performance:** Post-quantum signatures and keys are substantially larger than classical counterparts. Model protected object size with full proof set, measure verification latency under expected load, test in constrained environments before production.

**Key management and hardware:** Inventory hardware constraints during migration planning. Where hardware cannot process PQC keys, software-based PQC with hardware-based classical as co-requirement is an acceptable intermediate posture.

**Backwards compatibility:** Legacy verifiers SHOULD ignore unrecognized proofs and validate only recognized ones (fail-open for unrecognized types, fail-closed for recognized types with validation failures). Document which behavior is implemented.

**Storage and archival:** Model preservation cost over the object's expected lifetime, accounting for redundancy, geographic distribution, and format migration.

**Operational metrics:** Monitor: percentage of objects with only `deprecated` proofs, verifier failures due to unrecognized algorithms, suite status registry changes, canonicalization mismatches, proof set size approaching transport limits.

---

## 10. Stakeholder Guidance

| Stakeholder | Primary Concern | Guidance |
|-------------|-----------------|----------|
| **Issuers / Signers** | Objects verifiable for full lifetime | Default to dual-proof for long-lived objects; log hashes independently; schedule re-anchoring before `monitor`; maintain inventory of long-lived objects and suite status |
| **Verifiers / Relying Parties** | Defensible decisions during transitions | Adopt versioned verifier policy (Section 4.2); log non-`active` decisions; subscribe to suite status registry; test multi-proof objects before production |
| **Holders / Intermediaries** | Objects acceptable as suites change | Support re-anchoring requests; surface suite status to end users; preserve witness receipts |
| **Auditors / Compliance** | Historical records as defensible evidence | Require documented re-anchoring schedules; verify evidence preservation for high-value records; test catastrophic-break response |
| **Public-sector / Regulators** | Multi-decade trust for civil records | Mandate hash-based fallbacks and log anchoring for multi-decade records; publish suite status expectations; participate in registry governance |
| **Log Operators** | Witnessing infrastructure continuity | Maintain parallel STHs during transitions; publish own continuity plan including catastrophic-break response |
| **Library / Platform Maintainers** | Adoption through tooling | Support multi-proof objects natively; surface suite status automatically; provide re-anchoring utilities; publish canonicalization conformance results |
| **Standards Bodies** | Ecosystem-wide interoperability | Coordinate algorithm identifiers across containers; align suite status registries; publish container binding specifications; contribute test vectors |

---

## 11. Living Document Process

This document evolves continuously rather than being finalized once.

**Update triggers:** New PQC algorithm standardized; credible cryptanalytic result against an `active` or `monitor` suite; new container binding contributed; implementation experience reveals uncovered pattern; real-world dual-break or near-miss event.

**Review cadence:** At least annually, with additional reviews triggered by material events.

**Contribution model:** Container bindings, test vectors, case studies, and stakeholder guidance welcomed from any organization or individual. Authority derives from breadth and quality of contributions, not from any single standards body.

**Coordination:** Independent of but coordinated with W3C VCWG, IETF JOSE/COSE/LAMPS, W3C WebAuthn WG, ISO/IEC JTC 1/SC 17 and SC 27, NIST Cryptographic Technology Group, IETF DNSOP, CNCF/Sigstore.

---

## 12. Security and Privacy Considerations

**Linkability:** Multi-proof objects carry more metadata, increasing linkable surface area. In privacy-sensitive contexts, weigh this against the continuity benefit.

**Key reuse:** Keys used in single-algorithm contexts MUST NOT be reused in composite contexts without explicit analysis, due to stripping-attack risk. Aligns with [[I-D.ietf-lamps-pq-composite-sigs]].

**Log/witness dependencies:** Witness infrastructure introduces trust dependencies. A witness with weaker continuity than the object it witnesses provides false security. Document and assess the continuity properties of witness systems.

**Registry integrity:** The suite status registry is a trust root. Its integrity, availability, and governance deserve scrutiny commensurate with other trust roots.

**Announcement timing:** Public announcement of `broken` status may accelerate attacks. Coordinated-disclosure model SHOULD allow a notification window for major relying parties before public announcement.

---

## 13. Acknowledgements

This document was informed by public discussion on the W3C Credentials Community Group mailing list and by contributions from implementers, cryptographers, and standards engineers across multiple ecosystems.

The editor thanks: Tokachi Kamimura for migration semantics and operational patterns; Sylvain Cormier for canonical-byte stability analysis; Greg Bernstein for proof-set examples; Stephen Curran for log-based verification perspective; Jori Lehtinen for algorithm-agnostic architecture input; Manu Sporny for Data Integrity binding details; and the did:trail contributors for dual-proof deployment experience.

Additional contributors from JOSE, COSE, X.509, WebAuthn, DNSSEC, firmware signing, and transparency log communities are explicitly sought to expand the container bindings (Section 6) and test vectors (Section 7.3).

---

## References

### Normative References

- [[VC-DATA-INTEGRITY]]: Verifiable Credential Data Integrity 1.0
- [[RFC7515]]: JSON Web Signature (JWS)
- [[RFC9052]]: CBOR Object Signing and Encryption (COSE)
- [[RFC5280]]: Internet X.509 Public Key Infrastructure Certificate and CRL Profile
- [[WebAuthn-3]]: Web Authentication Level 3
- [[RFC4033]], [[RFC4034]], [[RFC4035]]: DNSSEC
- [[RFC8785]]: JSON Canonicalization Scheme (JCS)
- [[RDF-CANON]]: RDF Dataset Canonicalization
- [[RFC6962]]: Certificate Transparency

### Informative References

- [[NIST-IR-8547]]: Transition to Post-Quantum Cryptography Standards
- [[FIPS203]]: ML-KEM Standard
- [[FIPS204]]: ML-DSA Standard
- [[FIPS205]]: SLH-DSA Standard
- [[I-D.ietf-lamps-pq-composite-sigs]]: Composite ML-DSA for X.509
- [[I-D.ietf-jose-pq-composite-sigs]]: Composite ML-DSA for JOSE/COSE
- [[VC-DI-EDDSA]]: Data Integrity EdDSA Cryptosuites
- [[VC-DI-ECDSA]]: Data Integrity ECDSA Cryptosuites
- [[VC-DI-QUANTUM-RESISTANT]]: Data Integrity Quantum-Resistant Cryptosuites
- [[PQT-HYBRID-TERM]]: Terminology for Post-Quantum Traditional Hybrid Schemes
- [[VC-FORGERY-DEFENSE]]: Verifiable Credentials Forgery Defense
- [[DID-TRAIL]]: did:trail DID Method

---

## Appendix A: Diagrams

*Placeholder for architectural diagrams:*
- *A.1: Architectural Overview — relationship between universal patterns, container bindings, and shared infrastructure*
- *A.2: Verifier Decision Flowchart*
- *A.3: Proof Evolution Timeline — object from issuance through multiple re-anchorings to log-based verification*
- *A.4: Re-Anchoring Sequence Diagram*
- *A.5: Catastrophic Break Response Swimlane*