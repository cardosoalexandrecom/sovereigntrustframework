---
title: Sovereign Trust Framework Home Page
description: Home Page
---
# Sovereign Trust Framework (STF)

**Trait-Driven, Modular Foundation for Trust Over IP (ToIP) and Self-Sovereign Identity (SSI)**

---

## Introduction to the Sovereign Trust Framework (STF)

<div style="float: right; width: 300px; margin: 0 0 1rem 1rem;">
  <video width="100%" controls autoplay muted style="border-radius: 4px;">
    <source src="videos/stf_bg.mp4" type="video/mp4">
    Your browser does not support the video tag.
  </video>
</div>

The **Sovereign Trust Framework (STF)** is an innovative orchestration layer designed to enhance **Self-Sovereign Identity (SSI)** systems by integrating and extending components from the [Trust Over IP (ToIP)](https://trustoverip.org/) Foundation's technology stack. STF focuses on providing modular, trait-driven abstractions for building interoperable, policy-driven trust systems that enable secure, privacy-preserving interactions in decentralized ecosystems.

STF contributes to ToIP by providing:
- **Modular trait-based interfaces** for cryptography, identity, runtime, and transport
- **Protocol-agnostic foundations** supporting TSP, DIDComm, HTTP, BLE, and future protocols
- **Maximum reusability** of existing battle-tested Rust crates
- **Deployment flexibility** across servers, cloud, embedded devices, and WASM
- **Clean abstractions** that facilitate adoption without competing with existing standards

### ToIP Foundation and the "Hourglass" Model

STFX primarily focuses on ToIP's foundational layers (Layer 0 and Layer 1), respecting the **"hourglass" philosophy** adopted by ToIP. This hourglass approach makes **Layer 2 (Trust Spanning Layer)** the central convergence point of the ToIP stack.

**Key ToIP Requirement:** A ToIP endpoint system MUST communicate with another ToIP endpoint system using the **Trust Spanning Protocol (TSP)** [REQ L2.1].

This design creates:
- **Diversity below Layer 2**: Many cryptographic algorithms, transport mechanisms, and identity formats
- **Convergence at Layer 2**: Universal messaging protocol (TSP) as the narrow waist
- **Diversity above Layer 2**: Many applications, use cases, trust frameworks, and governance models

STF provides the foundational abstractions enabling this architecture without enforcing specific implementations.

---

## STF High-Level Vision

STF addresses ToIP's requirement [REQ A.2]: *"In a ToIP endpoint system, the higher layers of the ToIP protocol stack MUST communicate with the lower layers via defined interfaces."*

**Key Observations:**
- ToIP/SSI is a rapidly evolving area with diverse developments across applications, prototypes, and specifications
- Current implementations (DIDComm, various DID methods) must be supported while enabling new approaches
- ToIP Layer 2 centers on TSP, but existing implementations like DIDComm remain important
- Well-defined layer interfaces are critical for portability, modularity, and evolution

**STF Approach:**
- Define **trait-based interfaces** for Layer 0 (runtime/transport) and Layer 1 (crypto/identity)
- Enable Layer 2/3/4 implementations to remain **self-contained, modular, and abstract**
- Support TSP as primary Layer 2 protocol while accommodating DIDComm and future protocols
- Maximize **reusability** of existing Rust crates rather than reimplementation

Since ToIP has no official layer interfaces, STF defines foundational trait interfaces based on TSP requirements and SSI best practices.

### Implementation: STFX Crates

STF is implemented through a collection of **STFX** (STF eXtensions) Rust crates that provide concrete trait definitions and implementations following the two-layer pattern.

---

## Core Design Principles

STF follows **SOLID principles** with a focus on:

- **Single Responsibility**: Each trait has one clear purpose
- **Interface Segregation**: Clients depend only on traits they use (e.g., `Signer` ≠ `SignatureVerifier`)
- **Dependency Inversion**: Depend on traits, not concrete implementations

### Two-Layer Pattern
- **Layer 1 (Base Traits)**: Minimal, ISP-compliant abstractions
- **Layer 2 (Helper Wrappers)**: Convenience structs combining traits

---

## STF Modules (STFX Crates)

STF provides **five foundational modules** implemented as STFX crates:

| Module | Purpose | Documentation |
|--------|---------|---------------|
| **stfx-crypto** | Cryptographic operations (signing, encryption, hashing, KDF) | [stfx-crypto.md](./stfx-crypto.md) |
| **stfx-vid** | Verifiable identity (creation, resolution, verification) | [VID Analysis](./VID-IMPLEMENTATION-ANALYSIS.md) |
| **stfx-runtime** | Async execution abstraction (Tokio, Embassy, async-std) | [stfx-runtime.md](./stfx-runtime.md) |
| **stfx-transport** | Transport mechanisms (HTTP, BLE, NFC, WebSockets) | [stfx-transport.md](./stfx-transport.md) |
| **Trust Decision Engines** | Trust evaluation frameworks (future component) | TBD |

---

## ToIP Alignment

STF implements the complete ToIP technology stack foundation:

| ToIP Layer | STF Module (STFX Crate) |
|-----------|-------------|
| **Layer 0: Foundation** | `stfx-runtime`, `stfx-transport` |
| **Layer 1: Trust Support** | `stfx-crypto`, `stfx-vid` |
| **Layer 2: Trust Spanning** | Integration kits (TSP, DIDComm) |
| **Layer 3: Trust Tasks** | Trust Decision Engines (future) |
| **Layer 4: Applications** | Built on STF Layers 0-3 |

### ToIP "Hourglass" Philosophy

STF respects ToIP's **hourglass model** where **Layer 2 (Trust Spanning)** is the narrow waist:
- **Below Layer 2**: Maximum diversity (many crypto algorithms, transports, identity formats)
- **Layer 2**: Convergence on universal messaging — ToIP requires endpoints communicate via **Trust Spanning Protocol (TSP)** [REQ L2.1]
- **Above Layer 2**: Application diversity (many use cases, trust frameworks, governance models)

### ToIP Layer 1: Trust Support Functions

ToIP Layer 1 provides "trust support functions" that STF abstracts through `stfx-crypto` and `stfx-vid` crates:

**Machine-to-Machine Trust:**
- Cryptographic hardware modules for key material generation
- Secure storage of secrets and cryptographic materials
- Secure computing environments
- Communication functions for deployment environments

**Human-to-Human Trust:**
- Identity binding mechanisms (biometrics, hardware attestation)
- Trusted Platform Modules (TPM) and confidential computing
- Hardware-based trust attestation systems

STF provides trait abstractions enabling these functions while remaining implementation-agnostic.

---

## Integration Kits

**Integration kits** combine STF traits (from STFX crates) into protocol-specific bundles:
- **TSPCrypto**: Combines signing, verification, encryption for TSP
- **DIDCommCrypto**: Combines HPKE + ChaCha20 for DIDComm
- **TSPVIDHandler**: Combines identity traits for TSP

Integration kits live primarily in **protocol repositories** with reference implementations in STF when appropriate.

---

## Architecture

For comprehensive architectural details, including:
- SOLID principles application
- Two-layer pattern specifications
- Module dependency graph
- Complete ToIP stack mapping
- Integration kit patterns and governance

See the **[STF Architecture](./architecture.md)** page.

---

## Technical Documentation

### Cryptographic Layer
- **[stfx-crypto](./stfx-crypto.md)** — Trait specifications and design
- **[Crypto API Analysis](./stfx-crypto-api-analysis.md)** — Design decisions
- **[Implementation Layers](./stfx-crypto-implementation-layers.md)** — Layer analysis

### Identity Layer
- **[VID Implementation Analysis](./VID-IMPLEMENTATION-ANALYSIS.md)** — Identity abstraction design

### Runtime & Transport
- **[stfx-runtime](./stfx-runtime.md)** — Async execution abstraction
- **[stfx-transport](./stfx-transport.md)** — Transport mechanisms

### Architectural Decisions
- **[Architecture Overview](./architecture.md)** — Complete architecture
- **[Architectural Principles](./architecural_principles.md)** — Design principles
- **[SOLID & DIP](./arch_DIP_SOLID.md)** — SOLID principles application

## References

A list of relevant references can be found in the [References](./references.md) page.

## Ongoing Work
Visit the GitHub repository at:

**[https://github.com/sovereigntrustframework](https://github.com/sovereigntrustframework)**

## Author & License
Author: **[Alexandre Cardoso](www.cardosoalexandre.com)** 

License: **Apache 2.0**
