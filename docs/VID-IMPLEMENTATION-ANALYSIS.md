# Verified Identifier (VID) Implementation Analysis

## Executive Summary

This document synthesizes current state research on Verified Identifier (VID) implementations and standards in the Self-Sovereign Identity (SSI) ecosystem. The analysis addresses six key architectural questions and provides evidence-based guidance for system designers implementing identity infrastructure aligned with Trust Over IP (ToIP) Layer 2 standards.

**Key Finding:** VID is an abstract category encompassing multiple identifier types (DIDs, AIDs) bound to cryptographic trust roots. TSP (Trust Spanning Protocol) serves as the universal Layer 2 interoperability mechanism, enabling diverse VID types to communicate securely while preserving trust properties.

---

## Question 1: Main VID Types - Differences (AID, DID, VID)

### VID (Verified Identifier) - The Umbrella Category

**Definition (TSP Spec):** A Verified Identifier is a category of digital identifier that meets cryptographic verification and governance assessment requirements. It is **not itself a digital identifier scheme** but rather a category encompassing identifiers that:

1. Support public-key cryptography with verifiable trust roots
2. Can be verified cryptographically (through signature verification)
3. Can be assessed for governance (through associated support systems)
4. May be centralized, federated, or decentralized

**Key Characteristics:**
- Format requirement: Must be compliant with DID format or URN format (RFC8141)
- Cryptographic non-correlation: Multiple VIDs controlled by same endpoint must not reveal correlation
- Support mandatory operations: address resolution, mapping to keys, verification
- Can be used in public, well-known, or nested scenarios

### DID (Decentralized Identifier) - W3C Standard

**Definition (W3C v1.0):** A globally unique persistent identifier that does not require a centralized registration authority.

**Syntax Format:** `did:method:method-specific-id[/path][?query][#fragment]`

**Core Properties:**
- **Verification Methods:** Public key material (Ed25519, RSA, secp256k1, etc.) with:
  - Format support: JSON Web Key (JWK) and Multibase encodings
  - Controller: Who controls the key
  - Type: Cryptographic algorithm identifier
  - Purposes: Authentication, assertion, key agreement, capability operations

- **Verification Relationships:** Define purposes for verification methods:
  - `authentication`: For authenticating the controller/subject
  - `assertionMethod`: For asserting claims about the subject
  - `keyAgreement`: For encrypted communications
  - `capabilityInvocation`: For invoking capabilities
  - `capabilityDelegation`: For delegating capabilities

- **Services:** Enable discovery and interaction patterns
  - Service endpoints with types (e.g., messaging, credential repository)
  - Transport addresses and protocols

- **Metadata:** 
  - `created`, `updated`, `deactivated` timestamps
  - `equivalentId`, `canonicalId` for identity relationships
  - `versionId`, `nextUpdate` for state management

**Core Operations (Method-dependent):**
1. **Create:** Method-specific process to generate new DID
2. **Resolve:** `resolve(did, options) → (metadata, DID document, document metadata)`
3. **Update:** Modify DID document properties (method-specific)
4. **Deactivate:** Mark DID as no longer active

**Supported DID Methods in Production:**
- `did:peer` - Peer DIDs (public-key based, no registration required)
- `did:web` - Web-hosted DIDs (HTTP-based resolution)
- `did:webvh` - Web with verifiable history
- `did:key` - Cryptographic material embedded in identifier
- DID methods in registry at did-method-registry.github.io

**Resolution Process:**
```
Input: DID + resolution options
     ↓
1. Parse DID identifier
2. Route to appropriate DID method resolver
3. Retrieve DID document (from blockchain, web server, etc.)
4. Validate signature/cryptographic proof
     ↓
Output: (resolution metadata, DID document, document metadata)
```

### AID (Autonomic Identifier) - KERI-Based

**Emerging Standard:** Key Event Receipt Infrastructure (KERI) defines AIDs with autonomic key management.

**Core Philosophy:**
- Autonomic: Self-managed, no external registry required
- Event-based: Keys managed through verifiable event logs (key event receipt logs)
- Non-correlatable: Distinct keys for different contexts

**Key Characteristics:**
- Identifier format: Derived from inception event
- Key rotation: Native support through event log
- Verification: Through key event history, not just current keys
- Trust root: Event log itself (KEL - Key Event Log)
- Use cases: High-security applications requiring key event history

**Status:** Active development, IETF standardization in progress (Web of Trust community)

### Comparative Analysis

| Characteristic | DID | AID | VID (Category) |
|---|---|---|---|
| **Standardization** | W3C v1.0 complete | IETF draft (KERI) | ToIP framework (abstract) |
| **Trust Root** | Method-dependent | Event log (KEL) | Any cryptographic foundation |
| **Key Management** | Method-dependent | Autonomic (event-based) | Method-dependent |
| **Registration Required** | Method-dependent | No | No |
| **Identifier Syntax** | Strict DID format | URN-compatible | DID or URN format |
| **Verification Method** | Cryptographic keys | Key event history | Keys + support system assessment |
| **Primary Use Cases** | Broad SSI | High-security, key-rotatable | Universal Layer 2 interoperability |
| **Governance Assessment** | Out of scope | Through event log | Required (via support system) |

---

## Question 2: Implementation in Major Projects

### Trust Spanning Protocol (TSP) - OpenWallet Foundation Labs

**Project:** Rust-based TSP SDK (openwallet-foundation-labs/tsp)

**Status:** Early prototype/experimental (v0.0.1), interfaces subject to change

**Supported VID Types:**
- `did:peer` - Self-contained peer DIDs (no resolution required)
- `did:web` - Web-hosted DIDs
- `did:webvh` - Web with verifiable history

**Implementation Architecture:**

```
TSP SDK Workspace (Cargo)
├── tsp_sdk/              # Core TSP implementation (5 crates)
│   ├── cesr/             # Composable Event Streaming Representation
│   ├── crypto/           # Cryptographic primitives
│   ├── definitions/      # Common data structures, traits
│   ├── transport/        # tokio-based transport layer
│   └── vid/              # Verified identifier handling
├── examples/
├── python/               # Python language bindings
├── javascript/           # JavaScript/Node.js bindings
└── fuzzing/              # Fuzz testing
```

**Core Cryptographic Primitives:**

1. **Signatures (Authentication):**
   - Algorithm: Ed25519 (mandatory)
   - Format: 64-byte signature using Curve25519 + SHA2-512
   - Security: SUF-CMA (Strong UnForgeability under Chosen Message Attack)

2. **Encryption (Confidentiality):**
   - **HPKE-Auth (Mandatory):**
     - KEM: DHKEM(X25519, HKDF-SHA256)
     - KDF: HKDF-SHA256
     - AEAD: ChaCha20Poly1305
     - Includes sender authentication via ephemeral keys
   - **HPKE-Base (Mandatory):**
     - Same algorithms, non-authenticating variant
     - Used when sender must be in encrypted payload
   - **Libsodium Sealed Box (Optional):**
     - X25519 DH + XSalsa20-Poly1305
     - Legacy support, deprecated in favor of HPKE

3. **Hashing:**
   - SHA2-256
   - BLAKE2b (256-bit)

**Message Structure:**
```
TSP_Message = {
    TSP_Envelope {
        TSP_Tag,           # Message start marker
        TSP_Version,       # Protocol version
        VID_sender,        # Sender's VID
        VID_receiver       # Receiver's VID (or NULL for nested)
    },
    TSP_Payload {
        Type,              # Message type (RFI, RFA, RFD, etc.)
        ControlFields,     # TSP operation fields
        Data,              # Application payload (optional)
    },
    TSP_Signature          # Ed25519 signature over envelope + payload
}
```

**Encoding:** CESR (Composable Event Streaming Representation) for all message parts

**CLI Example (TSP):**
```bash
# Create identity
tsp create --type web --alias bob bob

# Verify VID
tsp verify --alias alice <did-url>

# Direct message
tsp message --from alice --to bob "Hello"

# Routed message
tsp message --from alice --to bob --via <intermediary> "Hello"
```

**Key VID Operations Supported:**

1. **Create:** CLI command `tsp create --type [peer|web|webvh]`
2. **Resolve:** Automatic during relationship forming (VID_rcvr resolution)
3. **Verify:** TSP_VERIFY operation in relationship forming protocol
4. **Rotate:** Out-of-band only (not yet fully specified in TSP)
5. **Discover:** Via OOBI (Out-Of-Band Introduction) mechanism

**VID Mapping Operations (Required by TSP):**
```
VID.PK_e      # Public key for encryption (HPKE)
VID.SK_e      # Private key for encryption
VID.SK_s      # Private key(s) for signing
VID.PK_s      # Public key for signature verification
VID.VERIFY    # Verify endpoint has access to SK_s
VID.RESOLVEADDRESS  # Map to transport address
```

### DIDKit (Spruce Systems)

**Status:** Production-ready reference implementation

**Supported Features:**
- DID method implementations (did:key, did:web, did:peer, etc.)
- Credential issuance and verification
- Presentation generation and verification

**Architecture:** Rust core with language bindings (JavaScript, Python, WASM)

**VID Operations Support:**
- Create/resolve DIDs
- Verify signatures
- Manage verification methods
- Full W3C DID Core compliance

### Hyperledger Indy / Sovrin

**Status:** Production SSI infrastructure

**Identifier Type:** DIDs (did:indy, did:sov)

**Key Features:**
- Distributed ledger-based identity
- Credential ecosystem
- Trust registry for governance

**VID Operations:**
- On-ledger DID registration
- Credential-based authentication
- Governance framework integration

---

## Question 3: Supported Operations

### Universal VID Operations (TSP Requirements)

TSP defines abstract operations that all VID types must support:

#### Mandatory Operations

**1. VID.RESOLVEADDRESS(VID) → Transport Address**
- **Purpose:** Map VID to network address for message delivery
- **Usage:** Required for public VIDs; optional for nested VIDs
- **Implementation:** VID type-specific
- **Example (did:web):** Resolve to HTTPS endpoint
- **Example (did:peer):** Bypass (embedded in relationship)

**2. VID.MAPPING_TO_KEYS(VID) → {PK_e, SK_e, SK_s, PK_s}**
- **By controller (sender side):**
  - `VID.PK_e` - Public key for HPKE encryption
  - `VID.SK_e` - Private key for HPKE
  - `VID.SK_s` / `VID.SK_s_i` - Private keys for signature (can be multiple)

- **By assessing endpoint (receiver side):**
  - `VID.PK_s` - Public key for signature verification
  - `VID.PK_e` - Public key for HPKE

- **Implementation:** VID type-specific resolution process

**3. VID.VERIFY(VID, signature) → BOOL**
- **Purpose:** Verify that endpoint controls the VID (has access to secret key)
- **Method:** PKC signature verification
- **Additional info:** May use additional assessment information from support system
- **Simplified for nested:** Can be simpler for VIDs in already-verified relationships

#### DID-Specific Operations (W3C Standard)

**1. DID Resolution**
```
resolve(did, resolutionOptions) → {
    resolutionMetadata: {
        contentType,
        retrieved,
        duration,
        error
    },
    didDocument: {
        @context,
        id,
        controller,
        verificationMethod,
        authentication,
        assertionMethod,
        keyAgreement,
        capabilityInvocation,
        capabilityDelegation,
        service,
        created,
        updated,
        deactivated,
        versionId,
        nextUpdate
    },
    didDocumentMetadata: {
        created,
        updated,
        deactivated,
        nextUpdate,
        equivalentId,
        canonicalId
    }
}
```

**2. DID Create** (Method-specific)
- Generate DID identifier
- Create initial DID document
- Establish verification methods
- Register with support system (blockchain, web server, etc.)

**3. DID Update** (Method-specific)
- Modify verification methods
- Add/remove services
- Update controller
- Rotate keys

**4. DID Deactivate** (Method-specific)
- Mark DID as inactive
- Preserve DID document history
- Prevent further updates

#### AID-Specific Operations (KERI)

**1. AID Creation**
- Generate from inception event
- Create initial key event
- Establish event log (KEL - Key Event Log)

**2. AID Key Rotation**
- Add rotation event to KEL
- Update valid signing keys
- Maintain event history

**3. AID Verification**
- Verify against event log
- Check key validity at event time
- Validate event signatures

**4. AID Recovery**
- Recover from compromised keys via rotation
- Update valid key set atomically
- Maintain cryptographic proof of recovery

### Operation Mapping by VID Type

| Operation | DID | AID | TSP Support |
|---|---|---|---|
| **Create** | Method-specific | KERI inception | Via support system |
| **Resolve** | Method-specific | Event log lookup | Mandatory for public VIDs |
| **Verify** | Signature on VID | Event log + signature | Mandatory |
| **Rotate Keys** | Method-specific | KERI rotation event | Out-of-band (TBD) |
| **Discover** | Out-of-band | Out-of-band | OOBI mechanism |
| **Revoke/Deactivate** | Method-specific | Not applicable | Out-of-band |
| **Update Metadata** | Method-specific | Inception event (static) | Via support system |

---

## Question 4: Protocol Relationships (TRQP, ACDC, KERI/AID)

### Trust Registry Query Protocol (TRQP) v2.0

**Status:** Implementers Draft (ToIP)

**Purpose:** Enable VID holders to query trust registries for governance information

**Relationship to VIDs:**
- **Operates at ToIP Layer 2/3 boundary**
- Queries registry for governance metadata associated with VIDs
- Returns trust framework policies, compliance status, revocation status
- Used after VID verification to assess governance alignment

**Typical Flow:**
```
1. TSP endpoint A verifies VID_b (via TSP)
   ↓
2. A queries TRQP registry about VID_b governance
   ↓
3. Registry returns: governance framework, policies, accreditation status
   ↓
4. A assesses trust based on returned information + cryptographic trust from TSP
```

### Authentic Chained Data Container (ACDC)

**Status:** Public Review Phase (ToIP)

**Purpose:** Cryptographically-bound, portable credential/attestation format

**Relationship to VIDs:**
- **Issued by VID controller**
- **About VID subject** (or other subjects)
- **Cryptographically linked to issuer's VID** through signature
- **Contains chain of custody** (issuer → holder → others)

**Structure (Abstract):**
```
ACDC {
    issuer: VID,           # Verified identifier of issuer
    subject: VID,          # Subject identifier (may be another VID)
    claims: {},            # Credential claims
    rules: {},             # Usage rules
    signatures: [],        # Issuer signatures
    previousHash: SAID     # Link to previous event
}
```

**VID Relationship Patterns:**

1. **Self-Issued Credential:**
   - Issuer VID == Subject VID
   - Self-attestation use cases

2. **Third-Party Credential:**
   - Issuer VID != Subject VID
   - Verifier can establish chain: Issuer VID → ACDC → Subject VID

3. **Delegated Credential:**
   - Issuer delegates to delegatee
   - Chain: Original issuer VID → delegation ACDC → delegatee VID → final ACDC

**Integration with TSP:**
- ACDC can be carried in TSP message payloads
- TSP provides authenticity/confidentiality channel for ACDC transmission
- ACDC provides long-term, revocation-resistant attestations

### KERI / Autonomic Identifier (AID)

**Status:** IETF Standardization (Web of Trust)

**Purpose:** Self-sovereign identifier with autonomic key management

**Relationship to VIDs in TSP:**
- **AID as VID type:** Can be used as VID in TSP relationships
- **Event-based trust root:** TSP verifies against KEL, not static keys
- **Inherent key rotation:** Via event log, enables key agility
- **Non-correlatable:** Multiple AIDs per controller without key reuse

**Key Architectural Differences from DID:**

| Aspect | DID | AID |
|---|---|---|
| **Trust Root** | Registry (blockchain, web server, etc.) | Event log (KEL) |
| **Key Storage** | In DID document | In KEL events |
| **Key Rotation** | Registry update | Event log rotation entry |
| **Non-correlation** | Different DIDs with different keys | Different AIDs, inherent non-correlation |
| **Verifier Role** | Resolves DID document | Verifies against KEL |

**KERI Event Structure:**
```
KeyEvent {
    version: "KERI1",
    kind: "icp|rot|dip|drt|ixn|rct|...",  # Event type
    serialNumber: <n>,
    aidPrefix: <aid>,           # AID identifier
    sequenceNumber: <sn>,
    flags: <digest algorithm>,
    witnesses: [<wits>],        # Witness KEL endorsements
    keyConfig: {
        keys: [<keys>],         # Current signing keys
        threshold: <t>
    },
    nextKeys: [<keys>],         # Keys for next rotation
    timestamp: <ts>,
    signatures: [<sigs>]        # Endorsing signatures
}
```

**Witness Architecture:**
- External systems that endorse (sign) key events
- Enable distributed verification without requiring all parties to maintain full KEL
- Can be validators in blockchain or dedicated witness servers

### Protocol Integration Architecture

```
TSP Layer 2 (Trust Spanning)
│
├─ Direct VID Verification
│  └─ DID resolution + signature verification
│  └─ AID verification against KEL
│
├─ Governance Assessment (Layer 3)
│  └─ TRQP queries for trust registry
│  └─ Policy evaluation
│
└─ Credential Exchange
   └─ ACDC transmission in TSP payloads
   └─ Chain of custody verification
```

**Typical Conversation Flow:**
```
Endpoint A ──TSP Message──> Endpoint B (TSP Layer 2: authenticity)
           │
           ├─ VID verification (DID resolve or AID KEL check)
           │
           ├─ Governance check (TRQP query)
           │
           └─ If ACDC in payload: verify issuer signature against issuer VID
```

---

## Question 5: Architectural Patterns in Existing Implementations

### Pattern 1: Relationship Table Architecture (TSP)

**Core Concept:** Endpoints maintain directional relationship tables

```
Endpoint A's Relationship Table:
┌──────────────────────────────┐
│ <VID_a0, VID_b0> : metadata  │  Uni-directional relationship
│ (VID_a1, VID_b1) : metadata  │  Bi-directional relationship
│ (VID_a0, VID_c0) : metadata  │  To different endpoint
└──────────────────────────────┘
```

**Characteristics:**
- **Directional by default:** <VID_local, VID_remote>
- **Bi-directional when mutual:** (VID_local, VID_remote) notation
- **Per-relationship state:** Each relationship is independent
- **Non-correlation:** Different VID pairs even from same endpoint
- **Nested relationships:** Can create new relationships inside existing ones

**Implementation Benefit:** Enables metadata privacy by using different VIDs for different contexts while maintaining cryptographic non-correlation.

### Pattern 2: Support System Abstraction (TSP/DID)

**Core Concept:** Abstract interface between identifier and its support infrastructure

```
Endpoint ──Interface──> Support System
         │
         ├─ VID.RESOLVEADDRESS()     → Transport address
         ├─ VID.MAPPING_TO_KEYS()    → Cryptographic keys
         └─ VID.VERIFY()             → Verification info
         │
         └─ Implementation
            ├─ Blockchain-based (did:indy)
            ├─ Web-based (did:web)
            ├─ Self-contained (did:peer)
            ├─ Event log-based (AID/KERI)
            └─ Custom (organizational)
```

**Benefits:**
- VID type agnostic at TSP layer
- Support system selection per VID
- Decoupled identifier from infrastructure
- Enables governance assessment hooks

### Pattern 3: Nested VID Architecture

**Core Concept:** Hide inner VID identifiers within encrypted outer TSP message

```
Outer Layer (Public):
[VID_a1, VID_p1, [ENCRYPTED]]  ← Visible intermediaries

Inner Layer (Hidden):
           [VID_a2, VID_b2, payload] ← Only visible to intended endpoints
```

**Use Cases:**
- Hide endpoint-to-endpoint relationships from intermediaries
- Multiple relationship contexts with same physical endpoint
- Progressive trust establishment (public → private VIDs)

**Architectural Benefit:** Enables metadata privacy (correlation resistance) without requiring new VID types.

### Pattern 4: Event-Log-Based Trust (KERI)

**Core Concept:** Replace registry with cryptographically-linked event history

```
Inception Event
    ↓ (signed)
Rotation Event 1 (new keys, references previous)
    ↓ (signed with old keys, attests new keys)
Rotation Event 2
    ↓
... Event Log (KEL) ...

Verification: Replay events from inception, verify chains
```

**Characteristics:**
- **Immutable history:** Cannot revise past events
- **Key agility:** Frequent key rotation without registration
- **Non-repudiation:** Event signatures prove historical state
- **Distributed:** Can be endorsed by witnesses
- **Portable:** Full KEL can be transmitted (or just recent events)

**Contrast with DID:**
- DID: Current state in registry (point-in-time)
- AID: Full history in event log (continuous)

### Pattern 5: Multi-Level Relationship Formation (TSP)

**Stages of Relationship Establishment:**

```
Stage 1: Out-of-Band Introduction (OOBI)
         └─ Discover at least one VID of other endpoint
         └─ No security guarantees at this stage

Stage 2: Relationship Forming Invite (TSP_RFI)
         └─ Send cryptographic challenge
         └─ Contains: VID_sender, digest, nonce

Stage 3: Relationship Forming Accept (TSP_RFA)
         └─ Prove possession of private key
         └─ Create uni-directional relationship

Stage 4: Bi-directional Confirmation
         └─ Both endpoints verify each other
         └─ Relationship becomes bi-directional
         └─ Can establish nested or routed variants

Stage 5: Subsequent Messages
         └─ Use established relationship table
         └─ No re-verification overhead
```

**Pattern Benefit:** Separates initial trust establishment (expensive verification) from subsequent operations (cached relationship).

---

## Question 6: Gaps in Current VID Abstractions

### Gap 1: Key Rotation Specification

**Current State:**
- DID: Method-dependent (no standard approach)
- AID/KERI: Well-specified via event log
- TSP: Key update marked "TBD" (Issue #7)

**Impact:**
- Implementations diverge on rotation semantics
- No standard way to negotiate key updates in TSP relationships
- Unclear how rotated keys affect existing relationships

**Architectural Implication:**
- TSP needs explicit key update control message
- Should support both "hard" rotation (revoke old keys) and "soft" rotation (phase out)
- Need mechanism to signal key compromise vs. scheduled rotation

**Recommendation:**
```
Proposed TSP Control Message:
TSP_KEY_UPDATE {
    VID: <VID>,
    newKeys: <public keys>,
    rotationType: ["scheduled" | "compromise"],
    effectiveTime: <timestamp>,
    retirementTime: <timestamp or NULL>
}
```

### Gap 2: Governance Assessment Integration

**Current State:**
- TSP: Defines VID verification (cryptography), governance assessment separate
- TRQP: Specifies governance query protocol (draft)
- No standard integration between TSP verification and governance assessment

**Impact:**
- Endpoints must implement custom logic to integrate TRQP queries
- No standard response to "VID cryptographically valid but governance unknown"
- Unclear responsibility split (TSP layer vs. application layer)

**Architectural Implication:**
- Need standard policy attachment to VID verification results
- TRQP should be tightly integrated with TSP Layer 2/3 boundary
- Policy enforcement should be explicit in relationship table metadata

**Recommendation:**
```
Enhanced Relationship Table Entry:
RelationshipEntry {
    relationship: <VID_local, VID_remote>,
    cryptoVerificationStatus: "verified" | "failed" | "expired",
    governanceStatus: "compliant" | "unknown" | "non-compliant",
    policyId: <policy URI>,
    trustScore: <0-100>,
    metadata: {
        verifiedAt: <timestamp>,
        governanceCheckedAt: <timestamp>,
        expiresAt: <timestamp>
    }
}
```

### Gap 3: VID Lifecycle Management

**Current State:**
- DID: Deactivate defined, revocation mechanism unclear
- AID: No explicit deactivation (event log is immutable)
- TSP: No standard revocation or expiration mechanism

**Impact:**
- Endpoints may continue using compromised VIDs
- No standard for VID retirement
- Unclear interaction between VID deactivation and established relationships

**Architectural Implication:**
- Need standardized revocation checking (similar to certificate revocation lists)
- Revocation authority unclear (issuer, trusted third party, distributed)
- Relationship invalidation semantics undefined

**Recommendation:**
```
VID Lifecycle States:
- ACTIVE: Can be used for new relationships
- ROTATING: Keys rotating, relationships still valid
- SUSPENDED: Compromised, existing relationships must check revocation
- RETIRED: No longer used, can be removed from relationship table
- DEACTIVATED: Explicitly marked inactive, may be reactivated later

Revocation Checking:
- Inline with VID resolution
- Cached with TTL
- Network failure handling policy
```

### Gap 4: Cross-Method VID Relationships

**Current State:**
- TSP supports `did:peer`, `did:web`, `did:webvh` concurrently
- No specification for mixed DID/AID relationships
- No standardized method for negotiating supported VID types

**Impact:**
- Interoperability uncertain between different VID type ecosystems
- Cannot determine if endpoints can communicate until relationship established
- No mechanism to signal VID type preferences

**Architectural Implication:**
- Need capability negotiation protocol before relationship forming
- Support system compatibility must be verified
- Fallback strategies for incompatible VID types undefined

**Recommendation:**
```
Enhanced OOBI (Out-of-Band Introduction):
OOBI {
    primaryVID: <VID>,
    supportedMethods: ["did:peer", "did:web", "AID"],
    requiredCapabilities: {
        "encryption": "HPKE-Auth",
        "signing": "Ed25519",
        "metadata_privacy": boolean
    },
    fallbackVIDs: [<VID2>, <VID3>]
}
```

### Gap 5: Nested Relationship Limits

**Current State:**
- TSP supports arbitrary nesting depth
- No practical guidance on nesting levels
- Performance implications undefined

**Impact:**
- Implementers uncertain about reasonable nesting depths
- No standards for when nesting is appropriate
- Potential for performance degradation with deep nesting

**Architectural Implication:**
- Need guidance on nesting semantics (e.g., max 3-4 levels)
- Caching strategies for nested relationship tables
- Key agility through deep nesting not well-studied

### Gap 6: Support System Interoperability

**Current State:**
- TSP defines abstract interface to support system
- No standardized support system API
- Different VID methods use different backends

**Impact:**
- Each VID type requires custom support system integration
- Cannot implement generic "VID resolver"
- Difficult to add new VID types to existing implementations

**Architectural Implication:**
- Need common support system interface (HTTP REST? GraphQL? Custom?)
- Standard error handling and caching semantics
- Discovery mechanism for support systems

**Recommendation:**
```
Standard Support System Interface (Proposed):

GET /did-resolver/{method}/{id}
  → {didDocument, metadata, timestamp}

GET /vid-verify/{method}/{id}/{signature}
  → {valid: boolean, timestamp: <ts>, proof: <verification data>}

GET /governance/{method}/{id}
  → {policies: [...], compliance: {...}, revocationStatus: ...}
```

### Gap 7: Metadata Privacy Guidance

**Current State:**
- TSP defines nested and routed message mechanisms
- No clear guidance on when to use each
- No performance/security tradeoff analysis

**Impact:**
- Implementers uncertain about metadata privacy strategy
- Potential unnecessary performance overhead
- Threat model not fully articulated

**Architectural Implication:**
- Need metadata privacy policy language
- Clear threat model per nesting strategy
- Performance impact benchmarks

---

## Synthesis: Architectural Design Decisions

### Decision 1: VID Type Selection

**When to use each type:**

**DIDs (W3C Standard)** - Recommended for:
- ✅ Public/discoverable identities
- ✅ Interoperability with W3C ecosystem
- ✅ Governance framework integration (TRQP-compatible)
- ✅ Credential issuance (ACDC integration)
- ✅ When support system (blockchain, web) is already available

**AIDs (KERI-based)** - Recommended for:
- ✅ High-security applications requiring key event history
- ✅ Self-sovereign scenarios (no external registry needed)
- ✅ Frequent key rotation required
- ✅ When non-correlation is critical
- ✅ Research/advanced deployments

**Decision Rule:**
```
IF system requires external governance framework
  USE DID (did:web, did:indy, etc.)
ELSE IF key event history required
  USE AID (KERI)
ELSE IF peer-to-peer, self-contained
  USE DID (did:peer)
```

### Decision 2: TSP Relationship Architecture

**Recommended implementation:**

1. **Use directional relationships as default**
   - Implement relationship table: `<VID_local, VID_remote> → metadata`
   - Track verification status per relationship
   - Separate governance assessment from cryptographic verification

2. **Implement nested relationships for sensitive contexts**
   - Use outer relationship for intermediary routes
   - Use inner relationship for endpoint-to-endpoint (hidden from intermediaries)
   - Cache inner relationships per outer relationship

3. **Implement three trust levels:**
   ```
   CRYPTOGRAPHIC_VERIFIED  (TSP verification passed)
   GOVERNANCE_COMPLIANT    (TRQP assessment passed)
   APPLICATION_TRUSTED     (Application policy satisfied)
   ```

### Decision 3: Key Rotation Strategy

**Recommended approach:**

1. **Implement soft rotation pattern:**
   - Announce new keys in advance (grace period)
   - Accept signatures from both old and new keys during transition
   - Explicitly retire old keys after grace period

2. **Separate key rotation from relationship establishment:**
   - Key rotation does NOT invalidate existing relationships
   - Implement TSP_KEY_UPDATE control message (proposed above)
   - Allow policy-based key rotation (scheduled vs. emergency)

3. **Cache key information with TTL:**
   - Cache support system responses (DID resolution, AID KEL)
   - Set TTL based on key rotation frequency
   - Implement cache invalidation on key rotation notification

### Decision 4: Metadata Privacy Strategy

**Recommended policies:**

| Scenario | Pattern | Rationale |
|---|---|---|
| **Public Communication** | Direct mode, public VIDs | No privacy needed, minimal overhead |
| **Organization-internal** | Nested mode, private VIDs | Hide from external observers |
| **Cross-organization** | Routed mode, public→private | Hide from intermediaries |
| **High-privacy** | Nested + routed, multi-level | Maximum privacy (performance cost) |

**Implementation:**
- Make nested/routed mode decision explicit in policy
- Default to direct mode (minimum overhead)
- Document privacy threat model for each level

### Decision 5: Support System Integration

**Recommended architecture:**

```
Application Layer
       ↓
VID Manager (local implementation)
  ├─ Relationship table
  ├─ Local VID management
  ├─ TSP message handling
       ↓
Support System Adapter Layer (pluggable)
  ├─ DID Resolver (for did:*)
  ├─ KERI Verifier (for AID)
  ├─ TRQP Client (for governance)
  ├─ Cache layer (TTL-based)
       ↓
External Support Systems
  ├─ Blockchains (Indy, web3)
  ├─ Web servers (did:web)
  ├─ KERI witness network
  └─ Trust registries (TRQP)
```

**Interface design:**
- Standardize on HTTP/REST with JSON payloads
- Implement circuit breaker for support system unavailability
- Local fallback for cached VIDs
- Graceful degradation when support system down

---

## STFX Architecture Decision: VID Module Design

### The SOLID Challenge

When designing `stfx-vid` following STFX principles (Reusability, Modularity, SOLID compliance), a critical design question emerged:

**Should stfx-vid expose ONE trait (like `stfx-transport` and `stfx-runtime`) or MULTIPLE traits?**

### Analysis

VID operations naturally cluster into three distinct concerns:

1. **Resolution/Verification** (read path): `resolve()`, `verify()`
   - Used by: TSP message handlers, credential verifiers, signature validators
   - Examples: did:web resolver, did:key resolver
   - Characteristic: Read-only, stateless, universal support

2. **Creation** (write path): `create()`
   - Used by: Onboarding systems, wallet initialization, agent setup
   - Examples: AID creator (KERI), did:peer generator
   - Characteristic: Write-only, stateful, method-specific

3. **Discovery** (search path): `discover()`
   - Used by: Directory services, TRQP queries, relationship discovery
   - Examples: TRQP trust registry, well-known endpoint discovery
   - Characteristic: Query-based, optional, governance-integrated

**Critical observation:** Unlike `Transport` (where ALL implementations need open/send/receive/close) and `Runtime` (where ALL implementations need spawn/join/cancel/sleep/timeout), VID operations are **not universally required together**:

- `did:key` resolver: only needs `resolve()` + `verify()` (keys embedded, no creation)
- `did:web` resolver: only needs `resolve()` + `verify()` (read-only HTTP)
- AID creator: needs `create()` (KERI inception event generation)
- TRQP client: needs `discover()` (governance queries)

### Decision

**❌ REJECTED: Single `VIDProvider` trait**
```rust
pub trait VIDProvider {
    async fn resolve(&self, id: &str) -> Result<VIDDocument>;
    async fn verify(&self, id: &str, msg: &[u8], sig: &[u8]) -> Result<bool>;
    async fn create(&self, config: VIDConfig) -> Result<CreatedVID>;
    async fn discover(&self, query: &Query) -> Result<Vec<String>>;
}
```
**Problem:** Violates Interface Segregation Principle (ISP)
- Forces did:key to implement `create()` → returns `Err(Unsupported)`
- Forces did:web to implement `discover()` → returns `Err(Unsupported)`
- Clients depend on methods they don't use

**✅ ADOPTED: Three separate trait modules**

Following SOLID principles over pattern consistency:

```
stfx-vid-resolver/     # Trait: VIDResolver (resolve, verify)
stfx-vid-creator/      # Trait: VIDCreator (create)
stfx-vid-discovery/    # Trait: VIDDiscovery (discover)
```

Each trait is its own STFX module with focused responsibilities:
- `stfx-vid-resolver`: Core read path, universally needed
- `stfx-vid-creator`: Optional write path, onboarding systems
- `stfx-vid-discovery`: Optional search path, directory services

**Rationale:**
1. **ISP compliance**: Clients depend only on interfaces they use
2. **SRP compliance**: Each trait has single, focused responsibility
3. **DIP compliance**: TSP depends on `trait VIDResolver`, not concrete implementations
4. **Modularity**: Can implement resolver without creator
5. **Real-world alignment**: Matches actual deployment patterns (many systems only resolve, few create)

### Implication for stfx-crypto

This analysis reveals that **stfx-crypto will face identical challenges**:

Crypto operations cluster into distinct concerns:
- **Signing** (authentication): `sign()`, `verify_signature()`
- **Encryption** (confidentiality): `encrypt()`, `decrypt()`
- **Hashing** (integrity): `hash()`, `verify_hash()`
- **Key derivation** (key management): `derive_key()`, `rotate_key()`

**Question for stfx-crypto design:** Should these be separate trait modules (`stfx-crypto-signer`, `stfx-crypto-cipher`, `stfx-crypto-hasher`, `stfx-crypto-kdf`) or combined into one trait?

**Answer (following VID precedent):** Separate traits following ISP, as not all implementations need all operations:
- Ed25519 signer: only signing/verification
- ChaCha20Poly1305 cipher: only encryption/decryption
- BLAKE2b hasher: only hashing
- HKDF deriver: only key derivation

---

## Conclusion

Verified Identifiers are a unifying abstraction enabling diverse trust models (DID, AID, future types) to operate together through TSP. Key architectural insights:

1. **VID is category, not type:** Encompasses DIDs, AIDs, and potential future identifier schemes
2. **TSP is universal Layer 2:** Enables trustworthy communication across VID types
3. **Relationships, not registries:** TSP relationship table pattern replaces global registry
4. **Trust is multi-faceted:** Cryptographic + governance + application policies
5. **Gaps remain:** Key rotation, governance integration, lifecycle management need standardization
6. **STFX design principle:** When operations are not universally co-required, separate into focused trait modules (ISP over pattern consistency)

**For system architects:** Start with DIDs for initial deployment, prepare for AID integration, implement extensible support system adapters, keep governance assessment separate from cryptographic verification.

---

## References

- **W3C DID Core v1.0:** https://www.w3.org/TR/did-1.0/
- **Trust Spanning Protocol Specification:** https://trustoverip.github.io/tswg-tsp-specification/
- **TSP SDK (Rust):** https://github.com/openwallet-foundation-labs/tsp
- **KERI Specification:** https://trustoverip.github.io/keri/
- **ToIP Technology Architecture:** https://github.com/trustoverip/TechArch/blob/main/spec.md
- **TRQP (Trust Registry Query Protocol):** https://github.com/trustoverip/tswg-trqp-specification
