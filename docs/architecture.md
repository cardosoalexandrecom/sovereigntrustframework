---
title: Sovereign Trust Framework Architecture
description: Comprehensive architecture of STFX, SOLID principles, modular design, trait-based abstraction layers, ToIP alignment, and protocol integration capabilities.
---

# Sovereign Trust Framework (STFX) — Architecture

The **Sovereign Trust Framework (STFX)** is a modular, trait-driven architecture for building interoperable, policy-driven trust systems that remain privacy-preserving, transport-agnostic, and governance-neutral. STFX provides an extensible foundation enabling seamless integration with current and future trust protocols and specifications.

---

## Table of Contents

1. [Design Philosophy & SOLID Principles](#design-philosophy--solid-principles)
2. [Two-Layer Architecture Pattern](#two-layer-architecture-pattern)
3. [STFX Core Modules](#stfx-core-modules)
4. [Integration Kits & Protocol Adapters](#integration-kits--protocol-adapters)
5. [ToIP Stack Alignment](#toip-stack-alignment)
6. [Architectural Principles](#architectural-principles)

---

## Design Philosophy & SOLID Principles

STFX follows **SOLID design principles** to ensure maintainability, extensibility, and reusability:

### Single Responsibility Principle (SRP)
Each trait and module has **one reason to change**:
- `Signer` handles only message signing, not verification
- `Encryptor` and `Decryptor` are separate (not monolithic AEAD)
- `Hasher` is independent of key derivation
- `HPKESealer` and `HPKEOpener` are distinct operations

### Open/Closed Principle (OCP)
- **Open for extension**: New cryptographic algorithms via trait implementations
- **Closed for modification**: Base traits remain stable; add adapters, not changes
- Example: Add `ChaCha20Poly1305Encryptor` without modifying existing code

### Liskov Substitution Principle (LSP)
Any trait implementation is **interchangeable** within its interface:
```rust
// Both work identically via the Signer trait
let signer: Box<dyn Signer> = Box::new(Ed25519Signer::new(keypair));
let signer: Box<dyn Signer> = Box::new(ECDSASigner::new(keypair));
```

### Interface Segregation Principle (ISP)
Traits are **minimal and focused**:
- Don't force clients to depend on methods they don't use
- `Verifier` doesn't need signing capability
- `Wallet` doesn't need sealing (only decryption/opening)
- Each role gets **exactly** what it needs

### Dependency Inversion Principle (DIP)
- **Depend on abstractions (traits), not concrete implementations**
- `VerifyService` accepts `dyn SignatureVerifier`, not `Ed25519Verifier`
- Enables swappable crypto backends without refactoring business logic

---

## Two-Layer Architecture Pattern

STFX employs a **two-layer pattern** across all modules:

### Layer 1: Base Traits (Abstraction)
Minimal, **ISP-compliant trait interfaces** defining the contract for each cryptographic or functional capability. Traits are:
- **Single-purpose** (one operation per trait)
- **Async-aware** (using `#[async_trait]`)
- **Error-transparent** (using `Result<T>`)
- **Implementation-agnostic** (no knowledge of algorithm details)

Example from `stfx-crypto`:
```rust
#[async_trait]
pub trait Signer {
    async fn sign(&self, message: &[u8]) -> Result<Vec<u8>>;
    fn public_key(&self) -> Result<Vec<u8>>;
    fn algorithm(&self) -> &str;
}

pub trait SignatureVerifier {
    fn verify(&self, message: &[u8], signature: &[u8]) -> Result<()>;
    fn algorithm(&self) -> &str;
}

pub trait Encryptor {
    async fn encrypt(&self, plaintext: &[u8]) -> Result<Vec<u8>>;
    fn algorithm(&self) -> &str;
}

pub trait Decryptor {
    async fn decrypt(&self, ciphertext: &[u8]) -> Result<Vec<u8>>;
    fn algorithm(&self) -> &str;
}
```

### Layer 2: Helper Wrappers & Composition (Convenience)
**Concrete structs** that combine related traits for convenience, enabling:
- **Simplified delegation** to multiple trait implementations
- **State sharing** across operations
- **API ergonomics** for common patterns

Example helper wrappers:
```rust
// Symmetric cipher combining Encryptor + Decryptor
pub struct SymmetricCipher<E: Encryptor, D: Decryptor> {
    encryptor: E,
    decryptor: D,
}

// Signing kit combining Signer + Verifier
pub struct SigningKit<S: Signer, V: SignatureVerifier> {
    signer: S,
    verifier: V,
}

// HPKE kit combining Sealer + Opener
pub struct HpkeKit<Sealer: HPKESealer, Opener: HPKEOpener> {
    sealer: Sealer,
    opener: Opener,
}
```

### Benefits of Two-Layer Pattern

| Aspect | Layer 1 (Traits) | Layer 2 (Wrappers) |
|--------|------------------|-------------------|
| **Modularity** | Maximum; single concern | Composable combinations |
| **Reusability** | Core abstraction reused everywhere | Convenient for common scenarios |
| **Testing** | Easy mocking via trait bounds | Integration testing of combinations |
| **Deployment** | Flexible algorithm selection | Pre-composed for typical use cases |
| **SOLID Compliance** | SRP + ISP enforced | OCP + DIP facilitated |

---

## STFX Core Modules

STFX is organized into functional modules, each following the two-layer pattern. Each module has:
1. **Trait definitions** (`stfx-xxx/`) — Base Layer 1 abstractions
2. **Helper wrappers** (`stfx-xxx/`) — Layer 2 convenience structs
3. **Concrete implementations** (`stfx-xxx-yyy/`) — Algorithm-specific crates
4. **Integration kits** (protocol repos or `integration-kits/`) — Protocol-specific adapters

### 1. stfx-crypto — Cryptographic Operations

**Purpose**: Core cryptographic trait abstractions for signing, verification, encryption, decryption, hashing, key derivation, and hybrid encryption.

**Repository**: [docs/stfx-crypto.md](./stfx-crypto.md)

**Layer 1 Traits**:
- `Signer` — Sign messages
- `SignatureVerifier` — Verify signatures
- `Encryptor` — Encrypt plaintext
- `Decryptor` — Decrypt ciphertext
- `HPKESealer` — Hybrid Public Key Encryption (seal)
- `HPKEOpener` — Hybrid Public Key Encryption (open)
- `Hasher` — Cryptographic hashing
- `KeyDerivation` — Key derivation functions (KDF)

**Layer 2 Wrappers**:
- `SymmetricCipher<E, D>` — Combined encryption/decryption
- `SigningKit<S, V>` — Combined signing/verification
- `HpkeKit<Sealer, Opener>` — Combined HPKE sealing/opening

**Concrete Implementation Crates**:
- `stfx-crypto-signer-ed25519` — Ed25519 signing
- `stfx-crypto-verifier-ed25519` — Ed25519 verification
- `stfx-crypto-cipher-chacha20` — ChaCha20Poly1305 encryption/decryption
- `stfx-crypto-hpke-x25519` — X25519 HPKE
- `stfx-crypto-hasher-sha256` — SHA-256 hashing
- `stfx-crypto-kdf-blake3` — BLAKE3 key derivation

**Example Integration Kit** (in protocol repos):
```rust
// tsp/stfx-crypto-tsp/src/lib.rs
pub struct TSPCryptoBundle<S, V, E, D, Sealer, Opener, H, KDF> {
    signer: S,
    verifier: V,
    encryptor: E,
    decryptor: D,
    sealer: Sealer,
    opener: Opener,
    hasher: H,
    kdf: KDF,
}
```

---

### 2. stfx-vid — Verifiable Identity

**Purpose**: Identity abstraction traits for creating, resolving, discovering, and verifying identity representations across DIDs, VIDs, and other identity formats.

**Layer 1 Traits** (Proposed):
- `VIDCreator` — Create verifiable identities
- `VIDResolver` — Resolve identity metadata
- `VIDDiscoverer` — Discover identity credentials/attestations
- `VIDVerifier` — Verify identity claims and attestations

**Layer 2 Wrappers**:
- `FullVIDKit<Creator, Resolver, Discoverer, Verifier>` — Complete identity operations
- `ResolverKit<Resolver, Discoverer>` — Read-only identity discovery

**Concrete Implementation Crates**:
- `stfx-vid-didcore` — W3C DID Core compliance
- `stfx-vid-tsp` — TSP VID implementation
- `stfx-vid-resolver-http` — HTTP-based resolution
- `stfx-vid-discoverer-dht` — DHT-based discovery

---

### 3. stfx-runtime — Async Execution & Orchestration

**Purpose**: Runtime-agnostic abstraction for task spawning, coordination, and timing across different async executors (Tokio, async-std, Embassy) and deployment environments.

**Layer 1 Traits**:
- `Runtime` — Core async task management (spawn, join, cancel, sleep, timeout)

**Layer 2 Wrappers** (Proposed):
- Interval managers
- Task pools
- Timing utilities

**Concrete Implementation Crates**:
- `stfx-runtime-tokio` — Tokio-based runtime
- `stfx-runtime-embassy` — Embassy for embedded systems
- `stfx-runtime-async-std` — async-std runtime

---

### 4. stfx-transport — Communication Layer

**Purpose**: Transport-agnostic message delivery abstractions enabling protocol bridging.

**Layer 1 Traits** (Likely already implemented):
- `TransportClient` — Send messages
- `TransportServer` — Receive messages
- `MessageSerializer` — Encode/decode messages
- `ChannelEstablisher` — Set up communication channels

**Concrete Bridges**:
- `stfx-transport-tsp` — TSP message protocol
- `stfx-transport-didcomm` — DIDComm protocol
- `stfx-transport-http` — HTTP/HTTPS
- `stfx-transport-p2p-ble` — Bluetooth Low Energy

---

### 5. Trust Decision Engines (Layer 3 — Future Component)

**Purpose**: Provide declarative, language-agnostic evaluation frameworks for trust decisions. This layer uses `stfx-runtime` for async execution and `stfx-crypto` + `stfx-vid` for cryptographic validation.

**Future Concrete Implementations** (TBD):
- Rego-based evaluation engine (OPA-compatible)
- Core evaluation graph processing
- Custom decision engines as needed

**Note**: This is a planned Layer 3 component with better naming TBD.

---

## Integration Kits & Protocol Adapters

**Integration kits** are **protocol-specific bundles** combining STFX traits and implementations for seamless integration with external trust protocols. They eliminate impedance mismatch between STFX abstractions and protocol requirements.

### Architecture

```
┌─────────────────────────────────────────────────┐
│          Protocol Repositories                  │
│  (tsp/, didcomm/, openid/, ...)                │
│  ┌─────────────────────────────────────┐       │
│  │ integration-kits/stfx-crypto-tsp/   │       │
│  │ integration-kits/stfx-vid-tsp/      │       │
│  │ integration-kits/stfx-runtime-tsp/  │       │
│  └─────────────────────────────────────┘       │
└─────────────────────────────────────────────────┘
                      ↑
        ┌─────────────┴────────────────┐
        │                              │
┌───────────────────┐      ┌──────────────────────┐
│ STFX Repository   │      │ STFX Repository      │
│ (Core Traits)     │      │ (Reference Impls)    │
├───────────────────┤      ├──────────────────────┤
│ stfx-crypto/      │      │ integration-kits/    │
│ stfx-vid/         │      │ ├─tsp/               │
│ stfx-runtime/     │      │ ├─didcomm/           │
│ stfx-transport/   │      │ └─openid/            │
└───────────────────┘      └──────────────────────┘
```

### Examples

#### TSPCrypto Integration Kit
```rust
// integration-kits/stfx-crypto-tsp/src/lib.rs
use stfx_crypto::{Signer, SignatureVerifier, Encryptor, Decryptor};
use stfx_crypto_signer_ed25519::Ed25519Signer;
use stfx_crypto_cipher_chacha20::ChaCha20Poly1305;

pub struct TSPCrypto {
    signer: Ed25519Signer,
    verifier: Ed25519Verifier,
    encryptor: ChaCha20Poly1305,
    decryptor: ChaCha20Poly1305,
}

impl TSPCrypto {
    pub fn sign_tsp_message(&self, msg: &TSPMessage) -> Result<Vec<u8>> {
        self.signer.sign(&msg.serialize()).await
    }
    
    pub fn verify_tsp_message(&self, msg: &TSPMessage, sig: &[u8]) -> Result<()> {
        self.verifier.verify(&msg.serialize(), sig)
    }
}
```

#### DIDCommCrypto Integration Kit
```rust
// integration-kits/stfx-crypto-didcomm/src/lib.rs
pub struct DIDCommCrypto {
    signer: Ed25519Signer,
    encryptor: ChaCha20Poly1305,
    decryptor: ChaCha20Poly1305,
    hpke: HpkeKit<X25519Sealer, X25519Opener>,
}

impl DIDCommCrypto {
    pub fn pack(&self, plaintext: &[u8], recipient_key: &[u8]) -> Result<Vec<u8>> {
        // DIDComm message packing with HPKE + ChaCha20
        self.hpke.sealer.seal(plaintext, recipient_key)
    }
    
    pub fn unpack(&self, ciphertext: &[u8]) -> Result<Vec<u8>> {
        // DIDComm message unpacking
        self.hpke.opener.open(ciphertext)
    }
}
```

### Governance

| Location | Responsibility | When |
|----------|-----------------|------|
| **Protocol repos** | Full integration; native protocol support | Always (primary) |
| **STFX `integration-kits/`** | Reference implementations; templates | When protocol is stable and widely used |
| **Downstream projects** | Fork/customize for specific needs | For domain-specific variants |

---

## ToIP Stack Alignment

STFX is designed as a **complete implementation of the Trust Over IP (ToIP) technology stack**. The ToIP Foundation defines a four-layer model for decentralized digital trust:

### ToIP 4-Layer Model

**Source**: [ToIP Technology Architecture Specification V1.0](https://trustoverip.org/our-work/technical-architecture/)

```
┌────────────────────────────────────────────────┐
│  Layer 4: Trust Applications                   │
│  (User-facing apps: wallets, KYC, verifiers)  │
└────────────────────────────────────────────────┘
                       ↑
┌────────────────────────────────────────────────┐
│  Layer 3: Trust Tasks                          │
│  (Credential exchange, proof, verification)    │
└────────────────────────────────────────────────┘
                       ↑
┌────────────────────────────────────────────────┐
│  Layer 2: Trust Spanning (Messaging)           │
│  (Universal interoperable messaging protocols) │
└────────────────────────────────────────────────┘
                       ↑
┌────────────────────────────────────────────────┐
│  Layer 1: Trust Support                        │
│  (Cryptography, keys, identity binding)        │
└────────────────────────────────────────────────┘
                       ↑
┌────────────────────────────────────────────────┐
│  Layer 0: Foundation                           │
│  (Async runtime + transport mechanisms)        │
└────────────────────────────────────────────────┘
```

| ToIP Layer | Description | STFX Module |
|-----------|-------------|-------------|
| **Layer 0: Foundation** | Runtime-agnostic async execution + transport mechanisms | `stfx-runtime`, `stfx-transport` |
| **Layer 1: Trust Support** | Cryptographic primitives, key management, identity infrastructure | `stfx-crypto`, `stfx-vid` |
| **Layer 2: Trust Spanning** | Universal interoperable messaging protocols (TSP, DIDComm) | Integration kits combining `stfx-crypto` + `stfx-vid` |
| **Layer 3: Trust Tasks** | Credential exchange, proof presentation, claim verification, trust decisions | Trust Decision Engines (TBD naming) |
| **Layer 4: Trust Applications** | User-facing applications leveraging trust infrastructure | Built on top of STFX layers 1-3 |

### STFX-to-ToIP Mapping

```
STFX Architecture          ToIP Architecture
─────────────────────────  ──────────────────────────
stfx-runtime              →    ToIP Layer 0: Async Execution
stfx-transport            →    ToIP Layer 0: Transport Foundation
stfx-crypto               →    ToIP Layer 1: Cryptography
stfx-vid                  →    ToIP Layer 1: Identity
integration-kits/ (TSP)   →    ToIP Layer 2: Messaging (Trust Spanning)
Trust Decision Engines    →    ToIP Layer 3: Tasks
(applications)            →    ToIP Layer 4: Apps
```

---

## Architectural Principles

### 1. Modularity Through Traits

Every significant behavior is abstracted as a trait. Implementations are separate crates:
- **Trait definition** in `stfx-xxx/` (Layer 1)
- **Algorithm implementations** in `stfx-xxx-yyy/` (concrete)
- **Helpers and wrappers** in `stfx-xxx/` (Layer 2)

This ensures:
- Swappable algorithm backends
- No monolithic dependencies
- Clear separation of concerns

### 2. Composable Cryptography

Rather than large, opinionated crypto suites, STFX combines **minimal, composable traits**:

```rust
// Instead of: MonolithicCrypto { sign, verify, encrypt, decrypt }
// Use: Compose traits as needed
pub struct CryptoKit<S, V, E, D> {
    signer: S,
    verifier: V,
    encryptor: E,
    decryptor: D,
}
```

### 3. Transport Agnosticism

`stfx-transport` abstracts all message delivery:
- **Primary protocol**: TSP (Trust Spanning Protocol)
- **Alternative protocols**: DIDComm, HTTP/WS, BLE/NFC
- **Bridge layer**: Integration kits adapt between protocols

No layer above Layer 2 depends on a specific transport.

### 4. Trust Decision Evaluation (Future Layer 3)

A separate **trust decision evaluation layer** (to be designed, built on top of `stfx-runtime` for execution) will provide:
- Language-agnostic decision frameworks (e.g., Rego for OPA)
- Type-safe claim graphs
- Multi-source evidence processing
- Deterministic, verifiable results

Applications will make trust decisions based on evaluation execution, not hard-coded logic.

### 5. Privacy-by-Design

- **cDID model** (canonical DID) with ephemeral identifiers
- **Anti-correlation** through salted derivation
- **Metadata stripping** at message boundaries
- **Minimal disclosure** through selective disclosure protocols

### 6. Protocol-Agnostic, Protocol-Friendly

STFX is:
- **Not opinionated** about which protocols you use
- **Helper-friendly** through integration kits
- **Composable** — pick TSP + DIDComm + HTTP as needed
- **Future-proof** — add new protocols via new integration kits

---

## Module Dependency Graph

```
┌─────────────────────────────────┐
│   Applications / Use Cases      │  (Layer 4)
└────────────┬────────────────────┘
             │
┌────────────▼────────────────────┐
│   Trust Decision Engines        │  (Layer 3)
├─────────────────────────────────┤
│ Depends: stfx-vid, stfx-crypto  │
│          stfx-runtime           │
└────────────┬────────────────────┘
             │
┌────────────▼────────────────────┐
│      stfx-transport / Bridges   │  (Layer 2)
├─────────────────────────────────┤
│ Depends: stfx-crypto, stfx-vid  │
│          stfx-transport         │
└────────────┬────────────────────┘
             │
┌────────────▼────────────────────┐
│  stfx-crypto / stfx-vid         │  (Layer 1)
├─────────────────────────────────┤
│ Depends: stfx-runtime           │
│          stfx-transport         │
└────────────┬────────────────────┘
             │
┌────────────▼────────────────────┐
│  stfx-runtime / stfx-transport  │  (Layer 0)
├─────────────────────────────────┤
│ No STFX dependencies (only ext) │
└─────────────────────────────────┘
```

---

## Design Rationale: Why Two Layers?

### The Problem
- **Too few traits**: Monolithic trait designs (e.g., single `Crypto` trait) violate ISP. Wallet implementations don't need signing; signers don't need verification.
- **Too many traits**: Without composition helpers, each application must wire dozens of trait bounds together.

### The Solution
- **Layer 1 (Base Traits)**: Minimal, ISP-compliant abstractions. Each trait does exactly one thing.
- **Layer 2 (Helpers)**: Pre-composed wrappers for common patterns, reducing boilerplate and improving ergonomics.

### Benefits
| Challenge | Layer 1 Solution | Layer 2 Solution |
|-----------|------------------|------------------|
| ISP violations | Separate traits per operation | N/A (Layer 1 enforces ISP) |
| Boilerplate | Precise, minimal interfaces | Pre-composed combinations |
| Reusability | Universal across protocols | Reusable within protocol families |
| Testing | Easy mocking via trait bounds | Integration testing of combinations |

---

## Example: Implementing a Verifier

```rust
// Layer 1: Use only the traits you need
use stfx_crypto::{SignatureVerifier, HPKEOpener};
use stfx_vid::VIDVerifier;

pub struct Verifier<V: SignatureVerifier, O: HPKEOpener, VV: VIDVerifier> {
    sig_verifier: V,
    opener: O,
    vid_verifier: VV,
}

impl<V, O, VV> Verifier<V, O, VV> {
    pub fn verify_credential(&self, cred: &Credential) -> Result<()> {
        // Use only the traits we need; no Signer, no Encryptor dependency
        self.sig_verifier.verify(&cred.data, &cred.signature)?;
        self.vid_verifier.verify_identity(&cred.issuer)?;
        Ok(())
    }
}

// Layer 2: Use convenient helper wrappers for common scenarios
use stfx_crypto::VerificationKit;

let verifier = VerificationKit::default() // Ed25519 + X25519 HPKE
    .with_vid_verifier(MyVIDVerifier);
```

---

## Next Steps

1. **Finalize Layer 1 traits** for `stfx-vid` and `stfx-runtime`
2. **Implement Layer 2 helper wrappers** across all modules
3. **Create integration kit templates** for TSP, DIDComm, OpenID
4. **Document ISP corrections** (Encryptor/Decryptor split)
5. **Establish governance model** for protocol-specific kits

---

## References

- [Trust Over IP Foundation](https://trustoverip.org/)
- [ToIP Technology Architecture Specification V1.0](https://trustoverip.org/our-work/technical-architecture/)
- [SOLID Principles](https://en.wikipedia.org/wiki/SOLID)
- [stfx-crypto Module](./stfx-crypto.md)
- [STFX Cryptographic API Analysis](./stfx-crypto-api-analysis.md)
- [STFX Implementation Layers](./stfx-crypto-implementation-layers.md)  

