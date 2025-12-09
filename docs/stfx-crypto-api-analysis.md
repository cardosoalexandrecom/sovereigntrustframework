# 🔍 STFX Crypto Module: Real-World API Burden Analysis (Historical)

**Note:** This analysis documents the prior three-layer exploration. The current decision is a trait-only core; see `stfx-crypto.md` and `00-RESOLUTION-OVER-MODULARIZATION.md` for the active approach.

## Executive Summary

**The question (historical):** Is the hyper-modular stfx-crypto design (4 separate traits) a practical burden on users, or does it match what real SSI projects actually need?

**Findings from analyzing TSP SDK, Spruce SSI, and Hyperledger Aries:**
- **TSP SDK** uses minimal trait polymorphism: crypto is mostly generic over AEAD/KDF/KEM types, not trait objects
- **Spruce SSI** splits crypto across 100+ modules but uses a different strategy: concrete Algorithm enum instead of traits
- **Hyperledger Aries** (askar) provides mid-level composability with Builder patterns
- **ISP absolutely matters** - but there are better ways to achieve it than with maximum trait count

---

## Part 1: Real-World Crypto APIs

### TSP SDK Pattern (OpenWallet Foundation)

**Architecture:**
```rust
// Top-level interface: generic over crypto primitives, NOT trait objects
pub fn seal_and_hash<A, Kdf, Kem>(
    sender: &dyn PrivateVid,
    receiver: &dyn VerifiedVid,
    payload: Payload<&[u8]>,
    digest: Option<&mut Digest>,
) -> Result<TSPMessage, CryptoError>
where
    A: aead::Aead,          // Trait bound, but...
    Kdf: kdf::Kdf,          // ...NOT used as &dyn trait
    Kem: kem::Kem,          // ...used at COMPILE TIME
```

**Key insight:** **Generics, not trait objects!**

The algorithm selection happens at **compile time**:
```rust
pub fn seal_and_hash(...) {
    let msg = match receiver.encryption_key_type() {
        VidEncryptionKeyType::X25519 => 
            tsp_hpke::seal::<Aead, Kdf, kem::X25519HkdfSha256>(sender, receiver, ...),
        #[cfg(feature = "pq")]
        VidEncryptionKeyType::X25519Kyber768Draft00 => 
            tsp_hpke::seal::<Aead, Kdf, kem::X25519Kyber768Draft00>(sender, receiver, ...),
    };
}
```

**User burden:** LOW
- High-level API: `seal(sender, receiver, payload)` - ONE function
- Generics chosen at compile-time based on VID key type
- No trait objects, no complexity leakage to users

**What this tells us:**
- ✅ TSP doesn't need separate Signer/SignatureVerifier traits
- ✅ Compile-time polymorphism via generics is "free" (no perf cost)
- ✅ Users don't see the generics - they call simple functions

---

### Spruce SSI Pattern (Production Implementation)

**Architecture:**
```rust
// Rather than multiple traits, use concrete Algorithm enum
pub enum Algorithm {
    EdDSA,
    ES256K,
    ES256,
    ES384,
    ES256KR,
    ESBlake2b,
    EdBlake2b,
    RS256,
    PS256,
    Bbs(BbsInstance),
    // ... 30+ algorithms
}

impl AlgorithmInstance {
    pub fn sign(&self, key: &SecretKey, bytes: &[u8]) -> Result<Vec<u8>> {
        match self {
            Self::EdDSA => { /* ed25519 signing */ },
            Self::ES256K => { /* secp256k1 signing */ },
            // ... pattern match all variants
        }
    }

    pub fn verify(&self, key: &PublicKey, bytes: &[u8], sig: &[u8]) -> Result<bool> {
        match self {
            Self::EdDSA => { /* ed25519 verify */ },
            Self::ES256K => { /* secp256k1 verify */ },
            // ... pattern match all variants
        }
    }
}
```

**User burden:** MEDIUM
- Simple API: `algorithm.sign(key, bytes)` - ONE call
- But: library has 100+ modules (`ssi-crypto`, `ssi-jwk`, `ssi-claims`, etc.)
- The complexity is pushed into the library, not exposed to users

**What this tells us:**
```rust
// User code is actually simple:
let sig = algorithm.sign(&key, &message)?;
let valid = algorithm.verify(&pubkey, &message, &sig)?;

// Algorithm comes from JWK or verification method:
let algorithm: Algorithm = verification_method.algorithm()?;
```

---

### Hyperledger Aries Pattern (Battle-Tested)

**Architecture (askar):**
```rust
// Builder pattern for key selection
let key = SecretKey::generate(SecretKeyType::Ed25519)?;
let sig = key.sign_message(message)?;

// Or with explicit algorithm:
let sig = key.sign_bytes(Algorithm::EdDSA, message)?;
```

**User burden:** LOW
- Builder selects algorithm at construction, not at use-time
- No runtime polymorphism unless explicitly chosen
- Type-safe: `SecretKey<Ed25519>` vs `SecretKey<Secp256k1>`

---

## Part 2: Crypto Needs Analysis Across Scenarios

### Scenario 1: TSP Protocol Implementation

**What it needs:**
- ✅ Ed25519 signing/verification (mandatory)
- ✅ HPKE-Auth encryption (mandatory)
- ✅ SHA256/BLAKE2b hashing (optional)
- ❌ Plugin signers, decryption, key derivation during setup
- ❌ Hardware accelerators, TPM integration

**API Burden with Proposed stfx-crypto (4 traits):**
```rust
pub struct TSPApp<S, SV, HS, HO, H>
where
    S: Signer,              // trait #1
    SV: SignatureVerifier,  // trait #2
    HS: HPKESealer,         // trait #3
    HO: HPKEOpener,         // trait #4
    H: Hasher,              // trait #5
{
    // ...
}
```
**Result:** 5 generic type parameters for TSP. **BURDENSOME!**

**Better approach (TSP-focused):**
```rust
pub struct TSPCrypto {
    signer: Ed25519Signer,
    verifier: Ed25519Verifier,
    sealer: HpkeAuthSealer,
    opener: HpkeAuthOpener,
    hasher: Sha256,  // or Blake2b
}

pub struct TSPApp {
    crypto: TSPCrypto,  // ONE concrete struct, no generics
}
```
**Result:** No generics, users call `app.crypto.sign()` directly. **CLEAN!**

---

### Scenario 2: DIDComm v2 / W3C VC Stack

**What it needs:**
- ✅ Multiple signing algorithms (Ed25519, secp256k1, P-256, P-384, BLS)
- ✅ JWE encryption (AES-256-GCM, ChaCha20Poly1305)
- ✅ JSON canonicalization + hashing (SHA256, BLAKE2b, BLAKE3)
- ✅ Key derivation (HKDF, PBKDF2, Argon2)
- ✅ Algorithm selection at runtime (from JWK/proof metadata)
- ❌ Individual trait per signing algorithm (impossible!)

**TSP's approach:** **Will NOT work!** Can't have `Signer` for each algorithm.

**SSI Library's approach (better):**
```rust
pub enum Algorithm {
    EdDSA,
    ES256K,
    ES256,
    // ... 30+ algorithms
}

// Usage:
let algorithm = Algorithm::from_jwk(&key)?;
let sig = algorithm.sign(&key, message)?;  // Runtime polymorphism
```
**Result:** One trait bound (Algorithm enum), handles all cases. **SCALABLE!**

---

### Scenario 3: Hardware Crypto / TPM / Edge Devices

**What it needs:**
- ✅ Signer only (no encryption, no hashing)
- ✅ Verifier only (most deployment nodes)
- ✅ Hardware accelerators (Ed25519 in secure enclave)
- ❌ Forced to implement unused traits

**With hyper-modular design (ISP achieved!):**
```rust
// TPM can implement JUST the Signer trait
pub struct TpmSigner { /* ... */ }

impl Signer for TpmSigner {
    async fn sign(&self, message: &[u8]) -> Result<Vec<u8>> { /* HSM call */ }
    fn public_key(&self) -> Result<Vec<u8>> { /* Get from TPM */ }
}

// No need to implement Cipher, Hasher, KDF - ISP perfect!
```

**With Algorithm enum:**
```rust
pub enum Algorithm { /* ... */ }

// TPM can't just implement signing - has to match on all algorithms
// even though it only supports Ed25519
impl Algorithm {
    pub fn sign(&self, key: &SecretKey, msg: &[u8]) -> Result<Vec<u8>> {
        match self {
            Algorithm::EdDSA => { /* TPM */ },
            Algorithm::ES256K => { /* ??? TPM doesn't support this */ },
            // Forced to handle all variants
        }
    }
}
```

**Result:** ISP matters for hardware crypto! **Trait-based wins here.**

---

## Part 3: The Real Tension

| Design | TSP SDK | DIDComm/VC | Hardware | Maintenance |
|--------|---------|-----------|----------|-------------|
| **Hyper-modular traits** | ❌ 5+ params | ❌ Can't support | ✅ Perfect ISP | ⚠️ Complex |
| **Algorithm enum** | ✅ Simple | ✅ Scalable | ❌ Forced match arms | ✅ Easy |
| **Hybrid (Aries)** | ✅ OK | ✅ OK | ✅ OK | ✅ Great |
| **Generic+Builder** | ✅ Great | ⚠️ Limited | ✅ OK | ✅ Good |

---

## Part 4: Alternative Designs

### Option A: **Hybrid Trait + Algorithm Enum** (RECOMMENDED)

```rust
// Core traits for extensibility (ISP respected)
pub trait Signer: Send + Sync {
    async fn sign(&self, message: &[u8]) -> Result<Vec<u8>>;
    fn public_key(&self) -> Result<Vec<u8>>;
    fn algorithm(&self) -> &str;
}

pub trait SignatureVerifier: Send + Sync {
    async fn verify(&self, message: &[u8], sig: &[u8], pubkey: &[u8]) -> Result<bool>;
    fn algorithm(&self) -> &str;
}

// For users wanting multiple algorithms: use enum
pub enum Algorithm {
    EdDSA,
    ES256K,
    ES256,
    // ... etc
}

impl Algorithm {
    pub async fn sign(&self, key: &SecretKey, msg: &[u8]) -> Result<Vec<u8>> {
        match self {
            Self::EdDSA => /* use Ed25519Signer */,
            Self::ES256K => /* use Secp256k1Signer */,
        }
    }
    
    pub async fn verify(&self, key: &PublicKey, msg: &[u8], sig: &[u8]) -> Result<bool> {
        match self {
            Self::EdDSA => /* use Ed25519Verifier */,
            Self::ES256K => /* use Secp256k1Verifier */,
        }
    }
}
```

**Benefits:**
- ✅ TSP can use `Algorithm::EdDSA` directly (no generics)
- ✅ DIDComm can use `Algorithm` enum (handles 30+ algorithms)
- ✅ Hardware can implement `Signer` trait only (ISP!!)
- ✅ Simple cases don't see complexity

**User code for TSP:**
```rust
pub struct TSPCrypto {
    sign_alg: Algorithm,       // Enum, not trait
    verify_alg: Algorithm,
    // ... rest
}

// Just use it:
let sig = self.sign_alg.sign(&key, msg)?;
```

**User code for DIDComm:**
```rust
pub struct DIDCommCrypto {
    algorithms: HashMap<String, Box<dyn SignatureVerifier>>,
}

// Flexible, supports any algorithm
let verifier = self.algorithms.get("EdDSA")?;
verifier.verify(msg, sig, pubkey)?;
```

---

### Option B: **Monolithic CryptoProvider Trait** (SIMPLE, ISP COST)

```rust
pub trait CryptoProvider: Send + Sync {
    // Signing
    async fn sign(&self, key_id: &str, message: &[u8]) -> Result<Vec<u8>>;
    async fn verify(&self, key_id: &str, message: &[u8], sig: &[u8]) -> Result<bool>;
    
    // Encryption
    async fn encrypt(&self, key_id: &str, plaintext: &[u8]) -> Result<Vec<u8>>;
    async fn decrypt(&self, key_id: &str, ciphertext: &[u8]) -> Result<Vec<u8>>;
    
    // Hashing
    fn hash(&self, data: &[u8]) -> Vec<u8>;
}
```

**Benefits:**
- ✅ Minimal trait count (1)
- ✅ TSP integration is trivial
- ✅ User generics are gone

**Costs:**
- ❌ ISP violated: TPM must implement decrypt() even though unsupported
- ❌ Returns Err(Unsupported) for operations not available
- ❌ Less type-safe

---

### Option C: **Your Current Design** (MAXIMIZED ISP, HIGH BURDEN)

```rust
pub trait Signer { /* ... */ }
pub trait SignatureVerifier { /* ... */ }
pub trait AEAD { /* ... */ }
pub trait HPKESealer { /* ... */ }
pub trait HPKEOpener { /* ... */ }
pub trait Hasher { /* ... */ }
pub trait KeyDerivation { /* ... */ }
```

**Benefits:**
- ✅ ISP perfect: implement only what you need
- ✅ Hardware crypto: signer-only TPM module
- ✅ Clean separation of concerns

**Costs:**
- ❌ TSP becomes generic nightmare: 7 type params
- ❌ Documentation burden: must explain trait composition
- ❌ Testing: need mock implementations for 7 traits
- ❌ User confusion: "Which traits do I actually need?"

---

## Part 5: Recommendation

**HYBRID APPROACH (Option A):**

1. **Keep focused traits** for extensibility (Signer, SignatureVerifier, AEAD, Hasher, KDF)
2. **Add Algorithm enum** for common cases
3. **Provide StandardTSPCrypto** concrete type for TSP
4. **Document clear migration path**: users start with concrete types, graduate to traits if needed

### Implementation structure:

```
stfx-crypto-signer/              # Signer, SignatureVerifier traits + Ed25519 impl
    ├── Signer trait
    ├── SignatureVerifier trait
    └── stfx-crypto-signer-ed25519/ # Ed25519 concrete

stfx-crypto-cipher/               # Cipher traits + HPKE impl
    └── stfx-crypto-cipher-hpke/ # HPKE concrete

stfx-crypto-hasher/              # Hasher trait + SHA256 impl
    └── stfx-crypto-hasher-sha256/ # SHA256 concrete

stfx-crypto-provider/            # Algorithm enum (NEW!)
    └── Combines all above into Algorithm enum for convenience

stfx-crypto-standard-tsp/        # (NEW!) Pre-composed TSP crypto
    └── One concrete type: StandardTSPCrypto
```

### TSP Code becomes simple:

```rust
use stfx_crypto_standard_tsp::StandardTSPCrypto;

pub struct TSPApp {
    crypto: StandardTSPCrypto,  // That's it!
}

impl TSPApp {
    pub async fn send_message(&self, ...) {
        let sig = self.crypto.sign(message)?;  // Direct method
        // ...
    }
}
```

### DIDComm code stays flexible:

```rust
use stfx_crypto_provider::Algorithm;

pub struct DIDCommApp {
    algorithms: HashMap<String, Algorithm>,
}

impl DIDCommApp {
    pub async fn verify(&self, alg_name: &str, msg: &[u8], sig: &[u8]) {
        let alg = self.algorithms.get(alg_name)?;
        alg.verify(msg, sig, pubkey)?;
    }
}
```

### Hardware signer still works:

```rust
use stfx_crypto_signer::Signer;

pub struct TpmSigner;

impl Signer for TpmSigner {
    async fn sign(&self, msg: &[u8]) -> Result<Vec<u8>> {
        // TPM call
    }
}

// Can be used anywhere Signer is expected!
```

---

## Conclusion

**Your question:** "Aren't we over-modularizing?"

**Answer:** The hyper-modular traits ARE the "right architecture" from a SOLID standpoint. BUT they're **under-utilized** by TSP, which doesn't need that granularity.

**The solution:**
- Keep the trait structure (it's correct)
- Add convenience wrappers (Algorithm enum, StandardTSPCrypto)
- Document layers: "Use StandardTSPCrypto for TSP. Use Algorithm for DIDComm. Use individual traits for hardware."
- This way: Simple projects stay simple, complex projects stay powerful.

**Cost:** 10% more code (the wrapper layer). **Benefit:** 0% cognitive burden on users who don't need modularity.

