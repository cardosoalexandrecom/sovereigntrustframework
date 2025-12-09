📘 STFX — Sovereign Trust Framework in Rust

STFX is a modular, extensible Rust workspace designed to implement the foundational components of the Sovereign Trust Framework (STF).
Its core purpose is to provide high-assurance, reusable, and interoperable building blocks for identity, trust, cryptography, and ToIP-aligned Layers abstractions.

**STFX follows two guiding principles:**

1. Reusability — every component should be usable independently
2. Modularity — implementations must be loosely coupled behind stable traits

This allows STFX to grow into a flexible ecosystem rather than a monolithic system.

[STFX follows SOLID principles, with special focus on Dependency Inversion Principle - DIP](./arch_DIP_SOLID.md)

**🦀 Why Rust?**

[Rust](https://rust-lang.org/) was chosen as the core implementation language of STFX for the following reasons:

**1. Security Without Garbage Collection**

Identity and trust services are cryptography-heavy and require strict guarantees.
Rust gives:

- memory safety

- race-free concurrency

- deterministic cryptographic execution

This is essential for DID operations, key management, signatures, and trust services.

**2. Performance + Predictability**

Rust performance is near C/C++, enabling fast DID resolution, ledger lookups, and proof verification without compromising safety.

**3. WASM, FFI, and Cross-Platform Flexibility**

Rust can compile to:

- WASM for browser/agent environments

- static binaries for CLI tools

- native libraries with bindings for TypeScript, Go, Python, etc.

A single Rust codebase can serve all STF clients (web, mobile, backend, agent).

**4. A Mature SSI/Crypto Ecosystem**

Rust already powers major decentralized identity and ledger projects:

- DIDKit / SSI (SpruceID)

- ION / Sidetree implementations

- zk-SNARK libraries (Arkworks)

- Blockchain clients (Parity, Solana, IOTA)

STFX builds on shared foundations instead of reinventing them.

**🧱 Architectural Philosophy**

STFX is structured as a Rust workspace composed of independent crates.
Each crate serves one of two roles:

1. **Core crates** — Abstractions Only

These crates define traits, models, and error types.
They never contain concrete implementations.

Examples (Conceptual):

- stfx-did → DID method abstraction trait

- stfx-ledger → layer-1 utility abstraction trait

- stfx-crypto → key + signature trait

- stfx-storage → storage backend trait

These crates:

- have minimal dependencies

- define stable interfaces

- are reusable across many implementations

- make STFX a plugin-friendly system

2. **Implementation crates** — Pluggable Modules

These crates implement the traits defined in the core crates.

Examples (Conceptual):

- stfx-did-web

- stfx-did-key

- stfx-ledger-ethereum

- stfx-crypto-dalek

- stfx-storage-sqlite

Implementation crates may live inside the workspace or externally, meaning:

Any developer can create their own STFX-compatible crate without modifying STFX itself.

This is the key to ecosystem growth.

**🔌 The Plug-In Model**

STFX's identity and trust components are built as plug-in traits, enabling:

- multiple DID methods

- multiple ledger backends

- multiple trust anchor strategies

- multiple cryptography providers

- multiple storage options

Developers can combine them at runtime or compile-time.

Common patterns:
```rust
fn resolve(did: &str, method: &dyn DIDMethod) -> Result<DIDDocument>;
```

Or with generics:

```rust
fn resolve<M: DIDMethod>(method: M, did: &str) -> Result<DIDDocument>;
```


STFX does not force one DID method, one ledger, one crypto provider, or one trust logic.
It provides abstractions, not opinions.

**🧩 Modularity by Design**
**1. Replaceable Components**

Every subsystem can be swapped:

- crypto engine
- DID method
- ledger resolver
- storage engine
- governance module


**2. Independent Versioning**

Each crate can evolve independently, with stable shared interfaces.

**3. Isolation and Testability**

Because each module is a separate crate, it can be:

- tested independently

- benchmarked independently

- improved without breaking others

**4. Extensibility Beyond the Workspace**

A third-party crate (e.g., mycompany-did-customchain) can implement STFX traits and plug into any STFX-based system.

This is fundamental for ToIP compliance and trust ecosystems where many actors exist.

**🧭 Alignment with ToIP**

STFX’s architecture naturally supports:

- ToIP Layer 1 Abstraction
A unified set of traits for DID resolution, registration, updates, and ledger queries.

- Trust Service Providers (TSP)
Pluggable trust anchors, key custody modules, and governance policies.

- Future extensibility
New DID methods, cryptographic primitives, and trust policies can be added without forking STFX.

**📦 Example STFX Workspace Structure**
```text
stfx/
│── stfx-core/         # foundational types + shared models
│── stfx-did/          # DID traits
│── stfx-ledger/       # Layer-1 abstraction traits
│── stfx-crypto/       # cryptographic traits
│── stfx-storage/      # storage abstractions
│
│── implementations/
│   ├── stfx-did-web/
│   ├── stfx-did-key/
│   ├── stfx-crypto-dalek/
│   ├── stfx-storage-sqlite/
│   └── stfx-ledger-ethereum/
│
└── stfx-cli/          # optional CLI tool for testing
```


This is only a reference layout.
The actual repo may evolve based on STF implementation needs.

**🎯 Summary**

STFX is a Rust-based, modular, and reusable framework that:

provides core abstractions for identity and trust

allows modular and interchangeable implementations

aligns with ToIP and STF principles

is designed for long-term ecosystem growth

ensures high security and performance

STFX is not a monolith.
It’s a collection of building blocks that anyone can extend.

**📖 References**

- [Robert Cecil Martin. 2003. Agile Software Development: Principles, Patterns, and Practices. Prentice Hall PTR, USA.](https://dl.acm.org/doi/abs/10.5555/515230)
- 