# 🔐 STFX Crypto Layer

## 📖 Overview

**stfx-crypto** provides cryptographic abstractions for signatures, encryption, hashing, and key derivation in decentralized trust systems. Following STFX principles (Reusability, Modularity, SOLID), the crypto layer is intentionally **trait-only**: consumers compose the traits they need and supply concrete implementations.

This design recognizes that cryptographic operations serve **distinct purposes**:
- **Signing** (authentication, non-repudiation)
- **Encryption** (confidentiality, key agreement)
- **Hashing** (integrity, content addressing)
- **Key Derivation** (key management, entropy expansion)

Real-world implementations (TSP SDK, Spruce/ssi, Hyperledger Aries) show trait-based abstraction is a solid foundation; profiles and enums can be layered on top by downstream projects when needed, but STFX keeps the core surface minimal and focused on traits.

---

## 🎯 Design Philosophy

### The Problem We're Solving

**❌ ANTI-PATTERN 1: Monolithic Crypto Trait (violates ISP)**
```rust
pub trait Crypto {
    fn sign(&self, msg: &[u8]) -> Result<Vec<u8>>;
    fn verify(&self, msg: &[u8], sig: &[u8]) -> Result<bool>;
    fn encrypt(&self, plaintext: &[u8]) -> Result<Vec<u8>>;
    fn decrypt(&self, ciphertext: &[u8]) -> Result<Vec<u8>>;
    fn hash(&self, data: &[u8]) -> Vec<u8>;
}
```
**Problems:**
- Forces Ed25519 signer to implement unused `encrypt()`, `decrypt()`
- Impossible for TPM (signing-only device) to implement fully
- ISP violated: clients depend on unused methods

**❌ ANTI-PATTERN 2: Over-Generics (burdens implementers)**
```rust
pub struct TSPApp<S, SV, C, H, KD>
where
    S: Signer,
    SV: SignatureVerifier,
    C: Cipher,
    H: Hasher,
    KD: KeyDerivation,
{ /* 5+ type parameters! */ }
```
**Problems:**
- TSP has fixed algorithms (Ed25519, HPKE, SHA256) - doesn't need flexibility
- Generic bloat makes code hard to read
- Every instantiation requires specifying all 5 types
- Testing requires 5 mock implementations

### The Trait-Only Approach

```
stfx-crypto-signer/             # Traits: Signer, SignatureVerifier
    └── stfx-crypto-signer-ed25519/    # Concrete: Ed25519Signer, Ed25519Verifier
stfx-crypto-cipher/             # Traits: AEAD, HPKESealer, HPKEOpener
    └── stfx-crypto-cipher-hpke/       # Concrete: HpkeSealer, HpkeOpener
stfx-crypto-hasher/             # Trait: Hasher
    └── stfx-crypto-hasher-sha256/     # Concrete: Sha256Hasher, Blake2bHasher
stfx-crypto-kdf/                # Trait: KeyDerivation
    └── stfx-crypto-kdf-hkdf/          # Concrete: HkdfKeyDerivation
```

**Key idea:** STFX publishes only small, ISP-compliant traits. Integrators (including TSP) compose or wrap them as they see fit—either direct wiring or project-specific profiles—without STFX prescribing higher-level layers.

---

## 📚 Usage Guide (Trait-Only)

The only STFX surface is the set of focused traits. Downstream projects (including TSP) choose how to assemble them:

```rust
use stfx_crypto_signer::Signer;
use stfx_crypto_signer_ed25519::Ed25519Signer;

pub struct TSPRuntime<S: Signer> {
    signer: S,
    // ... other fields
}

let signer = Ed25519Signer::new(keypair);
let runtime = TSPRuntime { signer };

// Later, swap to TPM without changing call sites
let tpm_signer = TpmSigner::new();
let runtime = TSPRuntime { signer: tpm_signer };
```

Profiles, enums, or convenience bundles can be built on top by integrators, but they are intentionally **not** part of the core STFX API.

```
Are you implementing a custom protocol?
├─ Yes → Use Layer 1: Traits (maximum flexibility, ISP compliance)
│        pub struct MyProtocol<S: Signer, C: Cipher> { ... }
│
└─ No ─→ Do you need multiple algorithms at runtime?
         ├─ Yes → Use Layer 2: Algorithm Enum (DIDComm, W3C VC)
         │        pub struct MyApp { algorithms: HashMap<String, Algorithm> }
         │
         └─ No ─→ Use Layer 3: Convenience Type (TSP)
                  pub struct MyApp { crypto: StandardTSPCrypto }
```

---

## 🔌 Module 1: stfx-crypto-signer

### Purpose
Digital signature creation and verification for authentication and non-repudiation.

### Core Traits

```rust
use async_trait::async_trait;
use std::error::Error;

/// Trait for creating digital signatures
#[async_trait]
pub trait Signer: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    
    /// Signs a message with the private key
    async fn sign(&self, message: &[u8]) -> Result<Vec<u8>, Self::Error>;
    
    /// Returns the public key material for verification
    fn public_key(&self) -> Result<Vec<u8>, Self::Error>;
    
    /// Returns the algorithm identifier (e.g., "Ed25519", "ES256K")
    fn algorithm(&self) -> &str;
}

/// Trait for verifying digital signatures
#[async_trait]
pub trait SignatureVerifier: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    
    /// Verifies a signature against a message and public key
    async fn verify(&self, message: &[u8], signature: &[u8], public_key: &[u8]) 
        -> Result<bool, Self::Error>;
    
    /// Returns the algorithm identifier this verifier supports
    fn algorithm(&self) -> &str;
}
```

### Why Two Traits?

**Signer** and **SignatureVerifier** are separated because:
1. **Security isolation**: Signing requires private keys (HSM, secure enclave), verification is public operation
2. **Deployment patterns**: Many systems only verify signatures (relying parties), few create signatures (issuers)
3. **Performance**: Verification can be optimized differently (batch verification, caching)
4. **ISP compliance**: TSP message receivers only need `SignatureVerifier`, not `Signer`

### Implementation Example: Ed25519

```rust
// In stfx-crypto-signer-ed25519 crate
use ed25519_dalek::{Keypair, Signature, PublicKey};
use stfx_crypto_signer::{Signer, SignatureVerifier};

pub struct Ed25519Signer {
    keypair: Keypair,
}

#[async_trait]
impl Signer for Ed25519Signer {
    type Error = Ed25519Error;
    
    async fn sign(&self, message: &[u8]) -> Result<Vec<u8>, Self::Error> {
        use ed25519_dalek::Signer as DalekSigner;
        let signature: Signature = self.keypair.sign(message);
        Ok(signature.to_bytes().to_vec())
    }
    
    fn public_key(&self) -> Result<Vec<u8>, Self::Error> {
        Ok(self.keypair.public.to_bytes().to_vec())
    }
    
    fn algorithm(&self) -> &str {
        "Ed25519"
    }
}

pub struct Ed25519Verifier;

#[async_trait]
impl SignatureVerifier for Ed25519Verifier {
    type Error = Ed25519Error;
    
    async fn verify(&self, message: &[u8], signature: &[u8], public_key: &[u8]) 
        -> Result<bool, Self::Error> {
        use ed25519_dalek::Verifier;
        
        let public_key = PublicKey::from_bytes(public_key)?;
        let signature = Signature::from_bytes(signature)?;
        
        Ok(public_key.verify(message, &signature).is_ok())
    }
    
    fn algorithm(&self) -> &str {
        "Ed25519"
    }
}
```

### TSP Integration Example

```rust
use stfx_crypto_signer::{Signer, SignatureVerifier};

pub struct TSPMessageHandler<S, V>
where
    S: Signer,
    V: SignatureVerifier,
{
    signer: S,
    verifier: V,
}

impl<S, V> TSPMessageHandler<S, V>
where
    S: Signer,
    V: SignatureVerifier,
{
    pub async fn send_message(&self, envelope: &[u8], payload: &[u8]) 
        -> Result<TSPMessage, Box<dyn Error>> {
        // Concatenate envelope + payload
        let msg = [envelope, payload].concat();
        
        // Sign using injected signer
        let signature = self.signer.sign(&msg).await?;
        
        Ok(TSPMessage {
            envelope: envelope.to_vec(),
            payload: payload.to_vec(),
            signature,
        })
    }
    
    pub async fn verify_message(&self, msg: &TSPMessage, sender_pubkey: &[u8]) 
        -> Result<bool, Box<dyn Error>> {
        let msg_bytes = [&msg.envelope[..], &msg.payload[..]].concat();
        
        // Verify using injected verifier
        Ok(self.verifier.verify(&msg_bytes, &msg.signature, sender_pubkey).await?)
    }
}
```

### SOLID Compliance Analysis

**Single Responsibility (SRP)** ✅
- `Signer`: Only signature creation
- `SignatureVerifier`: Only signature verification
- Each trait has one reason to change (algorithm requirements)

**Open/Closed (OCP)** ✅
- New signature algorithms (ES256K, BLS) extend via new implementations
- Core traits never change
- Example: `stfx-crypto-signer-secp256k1`, `stfx-crypto-signer-bls`

**Liskov Substitution (LSP)** ✅
- Any `impl Signer` can be swapped (Ed25519 → secp256k1)
- TSP SDK accepts `&dyn Signer` regardless of algorithm
- Behavioral contract: `sign(msg)` always produces verifiable signature

**Interface Segregation (ISP)** ⭐ ✅
- **Signature creators** (wallets) only implement `Signer`
- **Signature verifiers** (relying parties) only implement `SignatureVerifier`
- **Hardware signers** (TPM) can implement `Signer` without `SignatureVerifier`
- No client forced to implement methods they don't need

**Dependency Inversion (DIP)** ✅
- TSP SDK depends on `trait Signer`, not `Ed25519Signer`
- Implementations injected at startup
- High-level policy (TSP) never imports low-level details (ed25519-dalek)

---

## 🔌 Module 2: stfx-crypto-cipher

### Purpose
Symmetric and asymmetric encryption for confidentiality and authenticated encryption.

### Core Traits

```rust
use async_trait::async_trait;
use std::error::Error;

/// Trait for authenticated encryption with associated data (AEAD)
#[async_trait]
pub trait AEAD: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    
    /// Encrypts plaintext with authenticated additional data
    async fn encrypt(&self, plaintext: &[u8], aad: &[u8]) 
        -> Result<Vec<u8>, Self::Error>;
    
    /// Decrypts ciphertext and verifies additional data
    async fn decrypt(&self, ciphertext: &[u8], aad: &[u8]) 
        -> Result<Vec<u8>, Self::Error>;
    
    /// Returns the algorithm identifier (e.g., "ChaCha20Poly1305", "AES-256-GCM")
    fn algorithm(&self) -> &str;
}

/// Trait for Hybrid Public Key Encryption (HPKE)
#[async_trait]
pub trait HPKESealer: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    
    /// Seals (encrypts) plaintext to recipient's public key
    async fn seal(&self, plaintext: &[u8], recipient_pubkey: &[u8], info: &[u8]) 
        -> Result<Vec<u8>, Self::Error>;
    
    /// Returns the ephemeral public key for this operation
    fn ephemeral_public_key(&self) -> Result<Vec<u8>, Self::Error>;
}

/// Trait for HPKE decryption
#[async_trait]
pub trait HPKEOpener: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    
    /// Opens (decrypts) ciphertext using private key
    async fn open(&self, ciphertext: &[u8], ephemeral_pubkey: &[u8], info: &[u8]) 
        -> Result<Vec<u8>, Self::Error>;
}
```

### Why Separate AEAD and HPKE?

1. **Different use cases**: 
   - AEAD: Symmetric encryption after key exchange (session keys)
   - HPKE: Asymmetric encryption for initial message encryption

2. **Key material**:
   - AEAD: Pre-shared or derived symmetric key
   - HPKE: Public/private keypairs

3. **Protocol alignment**:
   - TSP uses HPKE for message confidentiality
   - DIDComm uses HPKE for encrypted envelopes
   - Both may use AEAD internally

### Implementation Example: HPKE with X25519 + ChaCha20Poly1305

```rust
// In stfx-crypto-cipher-hpke crate
use hpke::{Hpke, HpkeMode, HpkeSuite};
use stfx_crypto_cipher::{HPKESealer, HPKEOpener};

pub struct HpkeAuthSealer {
    sender_keypair: (Vec<u8>, Vec<u8>), // (private, public)
}

#[async_trait]
impl HPKESealer for HpkeAuthSealer {
    type Error = HpkeError;
    
    async fn seal(&self, plaintext: &[u8], recipient_pubkey: &[u8], info: &[u8]) 
        -> Result<Vec<u8>, Self::Error> {
        // HPKE-Auth mode: includes sender authentication
        let suite = HpkeSuite::new(
            KemAlgorithm::DhKemX25519HkdfSha256,
            KdfAlgorithm::HkdfSha256,
            AeadAlgorithm::ChaCha20Poly1305,
        );
        
        let (encap, ciphertext) = hpke::single_shot_seal_auth(
            &suite,
            recipient_pubkey,
            info,
            plaintext,
            &self.sender_keypair.0, // Sender private key
        )?;
        
        // Prepend encapsulated key to ciphertext
        Ok([encap, ciphertext].concat())
    }
    
    fn ephemeral_public_key(&self) -> Result<Vec<u8>, Self::Error> {
        Ok(self.sender_keypair.1.clone())
    }
}

pub struct HpkeAuthOpener {
    recipient_keypair: (Vec<u8>, Vec<u8>), // (private, public)
}

#[async_trait]
impl HPKEOpener for HpkeAuthOpener {
    type Error = HpkeError;
    
    async fn open(&self, ciphertext: &[u8], ephemeral_pubkey: &[u8], info: &[u8]) 
        -> Result<Vec<u8>, Self::Error> {
        let suite = HpkeSuite::new(
            KemAlgorithm::DhKemX25519HkdfSha256,
            KdfAlgorithm::HkdfSha256,
            AeadAlgorithm::ChaCha20Poly1305,
        );
        
        // Extract encapsulated key (first 32 bytes) and ciphertext
        let (encap, ct) = ciphertext.split_at(32);
        
        let plaintext = hpke::single_shot_open_auth(
            &suite,
            &self.recipient_keypair.0, // Recipient private key
            encap,
            info,
            ct,
            ephemeral_pubkey, // Sender public key for authentication
        )?;
        
        Ok(plaintext)
    }
}
```

### TSP Integration Example

```rust
use stfx_crypto_cipher::{HPKESealer, HPKEOpener};

pub struct TSPEncryptedMessage<S, O>
where
    S: HPKESealer,
    O: HPKEOpener,
{
    sealer: S,
    opener: O,
}

impl<S, O> TSPEncryptedMessage<S, O>
where
    S: HPKESealer,
    O: HPKEOpener,
{
    pub async fn encrypt_payload(&self, payload: &[u8], recipient_pubkey: &[u8]) 
        -> Result<Vec<u8>, Box<dyn Error>> {
        let info = b"TSP-v1-HPKE-Auth";
        Ok(self.sealer.seal(payload, recipient_pubkey, info).await?)
    }
    
    pub async fn decrypt_payload(&self, ciphertext: &[u8], sender_pubkey: &[u8]) 
        -> Result<Vec<u8>, Box<dyn Error>> {
        let info = b"TSP-v1-HPKE-Auth";
        Ok(self.opener.open(ciphertext, sender_pubkey, info).await?)
    }
}
```

### SOLID Compliance Analysis

**SRP** ✅ - Each trait handles one encryption mode
**OCP** ✅ - New cipher suites extend via new implementations
**LSP** ✅ - Any HPKE implementation is interchangeable
**ISP** ✅ - Sealers and openers are separate (asymmetric operation)
**DIP** ✅ - TSP depends on traits, not concrete implementations

---

## 🔌 Module 3: stfx-crypto-hasher

### Purpose
Cryptographic hashing for integrity verification and content addressing.

### Core Trait

```rust
use std::error::Error;

/// Trait for cryptographic hash functions
pub trait Hasher: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    
    /// Computes the hash of input data
    fn hash(&self, data: &[u8]) -> Result<Vec<u8>, Self::Error>;
    
    /// Returns the algorithm identifier (e.g., "SHA256", "BLAKE2b-256")
    fn algorithm(&self) -> &str;
    
    /// Returns the output size in bytes
    fn output_size(&self) -> usize;
}
```

### Why Single Trait?

Unlike signing/verification or sealing/opening, hashing is **symmetric and stateless**:
- No public/private key distinction
- Same operation for all participants
- Pure function: `hash(data) → digest`

### Implementation Example: BLAKE2b

```rust
// In stfx-crypto-hasher-blake2b crate
use blake2::{Blake2b512, Digest};
use stfx_crypto_hasher::Hasher;

pub struct Blake2bHasher256;

impl Hasher for Blake2bHasher256 {
    type Error = std::convert::Infallible;
    
    fn hash(&self, data: &[u8]) -> Result<Vec<u8>, Self::Error> {
        let mut hasher = Blake2b512::new();
        hasher.update(data);
        let result = hasher.finalize();
        
        // Truncate to 256 bits (32 bytes)
        Ok(result[..32].to_vec())
    }
    
    fn algorithm(&self) -> &str {
        "BLAKE2b-256"
    }
    
    fn output_size(&self) -> usize {
        32 // 256 bits
    }
}
```

### TSP Integration Example

```rust
use stfx_crypto_hasher::Hasher;

pub struct TSPContentAddressing<H: Hasher> {
    hasher: H,
}

impl<H: Hasher> TSPContentAddressing<H> {
    pub fn compute_message_id(&self, envelope: &[u8], payload: &[u8]) 
        -> Result<String, Box<dyn Error>> {
        let msg = [envelope, payload].concat();
        let digest = self.hasher.hash(&msg)?;
        
        // Convert to base64url for TSP message ID
        Ok(base64::encode_config(digest, base64::URL_SAFE_NO_PAD))
    }
}
```

### SOLID Compliance Analysis

**SRP** ✅ - Only hashing responsibility
**OCP** ✅ - New hash algorithms extend via new implementations
**LSP** ✅ - Any hasher is interchangeable
**ISP** ✅ - Single focused interface (no unused methods)
**DIP** ✅ - TSP depends on trait, not concrete hasher

---

## 🔌 Module 4: stfx-crypto-kdf

### Purpose
Key derivation for entropy expansion, key stretching, and hierarchical key generation.

### Core Trait

```rust
use async_trait::async_trait;
use std::error::Error;

/// Trait for key derivation functions
#[async_trait]
pub trait KeyDerivation: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    
    /// Derives a key from input key material
    async fn derive(&self, ikm: &[u8], salt: Option<&[u8]>, info: &[u8], output_len: usize) 
        -> Result<Vec<u8>, Self::Error>;
    
    /// Returns the algorithm identifier (e.g., "HKDF-SHA256")
    fn algorithm(&self) -> &str;
}
```

### Implementation Example: HKDF-SHA256

```rust
// In stfx-crypto-kdf-hkdf crate
use hkdf::Hkdf;
use sha2::Sha256;
use stfx_crypto_kdf::KeyDerivation;

pub struct HkdfSha256;

#[async_trait]
impl KeyDerivation for HkdfSha256 {
    type Error = hkdf::InvalidLength;
    
    async fn derive(&self, ikm: &[u8], salt: Option<&[u8]>, info: &[u8], output_len: usize) 
        -> Result<Vec<u8>, Self::Error> {
        let hkdf = Hkdf::<Sha256>::new(salt, ikm);
        
        let mut okm = vec![0u8; output_len];
        hkdf.expand(info, &mut okm)?;
        
        Ok(okm)
    }
    
    fn algorithm(&self) -> &str {
        "HKDF-SHA256"
    }
}
```

### TSP Integration Example

```rust
use stfx_crypto_kdf::KeyDerivation;

pub struct TSPSessionKeys<K: KeyDerivation> {
    kdf: K,
}

impl<K: KeyDerivation> TSPSessionKeys<K> {
    pub async fn derive_session_key(&self, shared_secret: &[u8], context: &[u8]) 
        -> Result<Vec<u8>, Box<dyn Error>> {
        // Derive 32-byte session key from ECDH shared secret
        let key = self.kdf.derive(
            shared_secret,
            None, // No salt
            context,
            32, // AES-256 key size
        ).await?;
        
        Ok(key)
    }
}
```

### SOLID Compliance Analysis

**SRP** ✅ - Only key derivation responsibility
**OCP** ✅ - New KDFs (PBKDF2, Argon2) extend via new implementations
**LSP** ✅ - Any KDF is interchangeable
**ISP** ✅ - Single focused interface
**DIP** ✅ - TSP depends on trait, not concrete KDF

---

## 📦 Implementation Structure

```
stfx-crypto-signer/           # Core signing traits
    ├── src/lib.rs            # Signer, SignatureVerifier traits
    └── Cargo.toml

stfx-crypto-signer-ed25519/   # Ed25519 implementation
stfx-crypto-signer-secp256k1/ # secp256k1 (ES256K) implementation
stfx-crypto-signer-bls/       # BLS signatures (future)

stfx-crypto-cipher/           # Core encryption traits
    ├── src/lib.rs            # AEAD, HPKESealer, HPKEOpener traits
    └── Cargo.toml

stfx-crypto-cipher-hpke/      # HPKE implementation
stfx-crypto-cipher-chacha/    # ChaCha20Poly1305 AEAD
stfx-crypto-cipher-aes/       # AES-GCM AEAD

stfx-crypto-hasher/           # Core hashing trait
    ├── src/lib.rs            # Hasher trait
    └── Cargo.toml

stfx-crypto-hasher-sha256/    # SHA2-256 implementation
stfx-crypto-hasher-blake2b/   # BLAKE2b implementation

stfx-crypto-kdf/              # Core KDF trait
    ├── src/lib.rs            # KeyDerivation trait
    └── Cargo.toml

stfx-crypto-kdf-hkdf/         # HKDF implementation
stfx-crypto-kdf-pbkdf2/       # PBKDF2 (future)
```

---

## 🎯 Protocol Support Matrix

### TSP Requirements

| Operation | Module | Implementation | Status |
|-----------|--------|----------------|--------|
| Sign messages | `stfx-crypto-signer` | Ed25519 | ✅ Mandatory |
| Verify signatures | `stfx-crypto-signer` | Ed25519 | ✅ Mandatory |
| Encrypt payloads | `stfx-crypto-cipher` | HPKE-Auth (X25519+ChaCha20Poly1305) | ✅ Mandatory |
| Decrypt payloads | `stfx-crypto-cipher` | HPKE-Auth | ✅ Mandatory |
| Hash messages | `stfx-crypto-hasher` | SHA256, BLAKE2b | ✅ Mandatory |
| Derive session keys | `stfx-crypto-kdf` | HKDF-SHA256 | ⚠️ Optional |

### DIDComm v2 Requirements

| Operation | Module | Implementation | Status |
|-----------|--------|----------------|--------|
| Sign JWS | `stfx-crypto-signer` | Ed25519, ES256K | ✅ Required |
| Verify JWS | `stfx-crypto-signer` | Ed25519, ES256K | ✅ Required |
| Encrypt JWE | `stfx-crypto-cipher` | HPKE, ECDH-ES+A256KW | ✅ Required |
| Content encryption | `stfx-crypto-cipher` | AES-256-GCM, XChaCha20Poly1305 | ✅ Required |

### KERI (AIDs) Requirements

| Operation | Module | Implementation | Status |
|-----------|--------|----------------|--------|
| Sign events | `stfx-crypto-signer` | Ed25519, secp256k1, BLS | ⚠️ Ed25519 only (Phase 1) |
| Verify events | `stfx-crypto-signer` | Ed25519, secp256k1, BLS | ⚠️ Ed25519 only (Phase 1) |
| Hash events | `stfx-crypto-hasher` | BLAKE3, SHA256 | ⚠️ SHA256 only (Phase 1) |

---

## 🔗 Full Stack Integration Example

```rust
use stfx_transport::Transport;
use stfx_runtime::Runtime;
use stfx_crypto_signer::{Signer, SignatureVerifier};
use stfx_crypto_cipher::{HPKESealer, HPKEOpener};
use stfx_crypto_hasher::Hasher;

pub struct TSPSecureChannel<T, R, S, SV, HS, HO, H>
where
    T: Transport,
    R: Runtime,
    S: Signer,
    SV: SignatureVerifier,
    HS: HPKESealer,
    HO: HPKEOpener,
    H: Hasher,
{
    transport: T,
    runtime: R,
    signer: S,
    verifier: SV,
    sealer: HS,
    opener: HO,
    hasher: H,
}

impl<T, R, S, SV, HS, HO, H> TSPSecureChannel<T, R, S, SV, HS, HO, H>
where
    T: Transport,
    R: Runtime,
    S: Signer,
    SV: SignatureVerifier,
    HS: HPKESealer,
    HO: HPKEOpener,
    H: Hasher,
{
    pub async fn send_secure_message(
        &self, 
        payload: &[u8], 
        recipient_pubkey: &[u8]
    ) -> Result<(), Box<dyn Error>> {
        // 1. Hash payload for integrity
        let payload_hash = self.hasher.hash(payload)?;
        
        // 2. Encrypt payload with HPKE
        let ciphertext = self.sealer.seal(
            payload, 
            recipient_pubkey, 
            b"TSP-v1"
        ).await?;
        
        // 3. Sign encrypted payload
        let signature = self.signer.sign(&ciphertext).await?;
        
        // 4. Construct TSP message
        let msg = TSPMessage {
            ciphertext,
            signature,
            payload_hash,
        };
        
        // 5. Send over transport
        let msg_bytes = bincode::serialize(&msg)?;
        self.transport.send(&msg_bytes).await?;
        
        Ok(())
    }
    
    pub async fn receive_secure_message(
        &self,
        sender_pubkey: &[u8]
    ) -> Result<Vec<u8>, Box<dyn Error>> {
        // 1. Receive from transport
        let mut buf = vec![0u8; 4096];
        let n = self.transport.receive(&mut buf).await?;
        let msg: TSPMessage = bincode::deserialize(&buf[..n])?;
        
        // 2. Verify signature
        let valid = self.verifier.verify(
            &msg.ciphertext, 
            &msg.signature, 
            sender_pubkey
        ).await?;
        
        if !valid {
            return Err("Invalid signature".into());
        }
        
        // 3. Decrypt payload
        let plaintext = self.opener.open(
            &msg.ciphertext, 
            sender_pubkey, 
            b"TSP-v1"
        ).await?;
        
        // 4. Verify integrity hash
        let computed_hash = self.hasher.hash(&plaintext)?;
        if computed_hash != msg.payload_hash {
            return Err("Integrity check failed".into());
        }
        
        Ok(plaintext)
    }
}
```

---

## 🚀 Phased Rollout Plan

### Phase 1: TSP Core Requirements (Now)

**Modules:**
- ✅ `stfx-crypto-signer` (trait definitions)
- ✅ `stfx-crypto-signer-ed25519` (TSP mandatory)
- ✅ `stfx-crypto-cipher` (trait definitions)
- ✅ `stfx-crypto-cipher-hpke` (X25519+ChaCha20Poly1305)
- ✅ `stfx-crypto-hasher` (trait definitions)
- ✅ `stfx-crypto-hasher-sha256` (TSP mandatory)
- ✅ `stfx-crypto-hasher-blake2b` (TSP optional)

**Goal:** Enable TSP SDK to be fully algorithm-agnostic

### Phase 2: DIDComm/W3C VC Support (Q1 2026)

**Modules:**
- ⚠️ `stfx-crypto-signer-secp256k1` (ES256K for Bitcoin/Ethereum DIDs)
- ⚠️ `stfx-crypto-cipher-aes` (AES-256-GCM for JWE)
- ⚠️ `stfx-crypto-kdf` (trait definitions)
- ⚠️ `stfx-crypto-kdf-hkdf` (session key derivation)

**Goal:** Enable DIDComm v2 and W3C VC issuance/verification

### Phase 3: KERI/AID Support (Q2 2026)

**Modules:**
- ⚠️ `stfx-crypto-signer-bls` (BLS signatures for KERI multi-sig)
- ⚠️ `stfx-crypto-hasher-blake3` (KERI event hashing)
- ⚠️ `stfx-crypto-kdf-pbkdf2` (password-based key derivation)

**Goal:** Enable KERI AIDs and event log verification

---

## ✅ Summary: Why This Design Works

### Respects STFX Principles

1. **Reusability** ✅
   - Each crypto module usable independently
   - Ed25519 signer works without HPKE cipher
   - TSP, DIDComm, KERI can share implementations

2. **Modularity** ✅
   - Loose coupling via traits
   - Pluggable implementations (software, hardware, mock)
   - No monolithic crypto library dependency

3. **SOLID Compliance** ✅
   - **SRP**: Each trait has single cryptographic responsibility
   - **OCP**: New algorithms extend without modifying traits
   - **LSP**: Any implementation substitutable
   - **ISP**: Clients only implement methods they need
   - **DIP**: High-level code depends on abstractions

### Enables Real-World Deployments

- **Cloud/Server**: Software implementations (RustCrypto, ring, sodiumoxide)
- **Embedded/IoT**: Hardware crypto accelerators (TPM, secure element)
- **Testing**: Mock implementations for deterministic tests
- **WASM**: Pure Rust implementations without platform dependencies

### Aligns with Ecosystem

- **TSP SDK**: All mandatory algorithms supported
- **DIDComm v2**: JWS/JWE compatibility via trait implementations
- **KERI**: Event signing/verification via pluggable signers
- **W3C VCs**: LD-Proofs and JWT-VCs via common signer trait

---

## 🔗 Related Documentation

- [STFX Architectural Principles](./architecural_principles.md)
- [Dependency Inversion Principle (DIP)](./arch_DIP_SOLID.md)
- [STFX Transport Layer](./stfx-transport.md)
- [STFX Runtime Layer](./stfx-runtime.md)
- [VID Implementation Analysis](./VID-IMPLEMENTATION-ANALYSIS.md) (ISP rationale)
