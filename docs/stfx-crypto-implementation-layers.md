# 💻 STFX Crypto: Three-Layer Implementation Examples (Historical)

**Status:** Archived for reference. The current approach is trait-only; see `stfx-crypto.md` and `00-RESOLUTION-OVER-MODULARIZATION.md` for the active spec.

This document provides concrete code examples for each layer of the historical three-layer exploration.

---

## Layer 1: Traits (Maximum Extensibility)

### Module Structure

```
stfx-crypto-signer/
├── Cargo.toml
├── src/
│   ├── lib.rs          # Re-export traits
│   ├── signer.rs       # trait Signer, SignatureVerifier
│   └── error.rs        # Error types

stfx-crypto-signer-ed25519/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── signer.rs       # impl Signer for Ed25519Signer
│   └── verifier.rs     # impl SignatureVerifier for Ed25519Verifier
```

### Core Trait Definition (stfx-crypto-signer/src/signer.rs)

```rust
use async_trait::async_trait;
use std::error::Error;

/// Digital signature creation
#[async_trait]
pub trait Signer: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    
    async fn sign(&self, message: &[u8]) -> Result<Vec<u8>, Self::Error>;
    
    fn public_key(&self) -> Result<Vec<u8>, Self::Error>;
    
    fn algorithm(&self) -> &str;
}

/// Digital signature verification
#[async_trait]
pub trait SignatureVerifier: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    
    async fn verify(
        &self,
        message: &[u8],
        signature: &[u8],
        public_key: &[u8],
    ) -> Result<bool, Self::Error>;
    
    fn algorithm(&self) -> &str;
}
```

### Implementation Example (stfx-crypto-signer-ed25519/src/signer.rs)

```rust
use async_trait::async_trait;
use ed25519_dalek::{Keypair, Signature, SigningKey};
use stfx_crypto_signer::{Signer, SignatureVerifier};
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Ed25519Error {
    #[error("Invalid key")]
    InvalidKey,
    #[error("Signature error: {0}")]
    SignatureError(String),
}

pub struct Ed25519Signer {
    keypair: Keypair,
}

impl Ed25519Signer {
    pub fn new(keypair: Keypair) -> Self {
        Self { keypair }
    }
    
    pub fn from_seed(seed: &[u8; 32]) -> Self {
        let signing_key = SigningKey::from_bytes(seed);
        let keypair = Keypair {
            secret: signing_key,
            public: signing_key.verifying_key(),
        };
        Self { keypair }
    }
}

#[async_trait]
impl Signer for Ed25519Signer {
    type Error = Ed25519Error;
    
    async fn sign(&self, message: &[u8]) -> Result<Vec<u8>, Self::Error> {
        use ed25519_dalek::Signer as DalekSigner;
        let signature = self.keypair.sign(message);
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
    
    async fn verify(
        &self,
        message: &[u8],
        signature: &[u8],
        public_key: &[u8],
    ) -> Result<bool, Self::Error> {
        use ed25519_dalek::Verifier;
        
        let public_key = ed25519_dalek::VerifyingKey::from_bytes(
            public_key.try_into()
                .map_err(|_| Ed25519Error::InvalidKey)?
        ).map_err(|_| Ed25519Error::InvalidKey)?;
        
        let sig_bytes: [u8; 64] = signature.try_into()
            .map_err(|_| Ed25519Error::InvalidKey)?;
        let signature = Signature::from_bytes(&sig_bytes);
        
        Ok(public_key.verify(message, &signature).is_ok())
    }
    
    fn algorithm(&self) -> &str {
        "Ed25519"
    }
}

// Hardware TPM example - impl Signer only
#[cfg(feature = "tpm")]
pub struct TpmSigner {
    key_handle: u32,
}

#[cfg(feature = "tpm")]
#[async_trait]
impl Signer for TpmSigner {
    type Error = Ed25519Error;
    
    async fn sign(&self, message: &[u8]) -> Result<Vec<u8>, Self::Error> {
        // Call TPM hardware
        todo!("TPM sign operation")
    }
    
    fn public_key(&self) -> Result<Vec<u8>, Self::Error> {
        // Call TPM to get public key
        todo!("TPM get public key")
    }
    
    fn algorithm(&self) -> &str {
        "Ed25519"
    }
}
// Note: TpmSigner does NOT implement SignatureVerifier (ISP!)
// Verification is a client-side operation; TPM only signs
```

### Layer 1 Usage Pattern

```rust
use stfx_crypto_signer::{Signer, SignatureVerifier};
use stfx_crypto_signer_ed25519::{Ed25519Signer, Ed25519Verifier};

/// Protocol that works with ANY signer implementation
pub struct TSPMessageSigner<S: Signer> {
    signer: S,
}

impl<S: Signer> TSPMessageSigner<S> {
    pub async fn sign_message(&self, msg: &[u8]) -> Result<Vec<u8>> {
        self.signer.sign(msg).await.map_err(Into::into)
    }
}

// Usage: works with any Signer implementation
let ed25519_signer = Ed25519Signer::new(keypair);
let handler = TSPMessageSigner { signer: ed25519_signer };

// Later: swap in TPM without changing code
#[cfg(feature = "tpm")]
let tpm_signer = TpmSigner { key_handle: 0x1001 };
#[cfg(feature = "tpm")]
let handler = TSPMessageSigner { signer: tpm_signer };
```

---

## Layer 2: Algorithm Enum (Runtime Flexibility)

### Module Structure

```
stfx-crypto-provider/
├── Cargo.toml
├── src/
│   ├── lib.rs           # Re-export Algorithm enum
│   ├── algorithm.rs     # enum Algorithm
│   ├── signing.rs       # Algorithm::sign() implementation
│   ├── verification.rs  # Algorithm::verify() implementation
│   └── error.rs
```

### Algorithm Enum Definition (stfx-crypto-provider/src/algorithm.rs)

```rust
use async_trait::async_trait;
use crate::error::CryptoError;

/// Enumeration of supported signature algorithms
#[derive(Debug, Clone)]
pub enum Algorithm {
    EdDSA,
    ES256,      // NIST P-256
    ES256K,     // secp256k1
    ES384,      // NIST P-384
    ES512,      // NIST P-521
    #[cfg(feature = "pq")]
    MlDsa65,    // Post-quantum (NIST FIPS 204)
    #[cfg(feature = "bbs")]
    BBS,        // BBS signatures
}

impl Algorithm {
    /// Get the algorithm name for JWK/proof metadata
    pub fn name(&self) -> &str {
        match self {
            Algorithm::EdDSA => "EdDSA",
            Algorithm::ES256 => "ES256",
            Algorithm::ES256K => "ES256K",
            Algorithm::ES384 => "ES384",
            Algorithm::ES512 => "ES512",
            #[cfg(feature = "pq")]
            Algorithm::MlDsa65 => "MlDsa65",
            #[cfg(feature = "bbs")]
            Algorithm::BBS => "BBS",
        }
    }
    
    /// Parse from JWK "alg" field
    pub fn from_jwk_alg(alg: &str) -> Result<Self, CryptoError> {
        match alg {
            "EdDSA" => Ok(Algorithm::EdDSA),
            "ES256" => Ok(Algorithm::ES256),
            "ES256K" => Ok(Algorithm::ES256K),
            "ES384" => Ok(Algorithm::ES384),
            "ES512" => Ok(Algorithm::ES512),
            #[cfg(feature = "pq")]
            "MlDsa65" => Ok(Algorithm::MlDsa65),
            #[cfg(feature = "bbs")]
            "BBS" => Ok(Algorithm::BBS),
            other => Err(CryptoError::UnsupportedAlgorithm(other.to_string())),
        }
    }
}

#[async_trait]
pub trait SigningCapability {
    async fn sign(&self, key: &[u8], message: &[u8]) -> Result<Vec<u8>, CryptoError>;
}

#[async_trait]
pub trait VerificationCapability {
    async fn verify(
        &self,
        public_key: &[u8],
        message: &[u8],
        signature: &[u8],
    ) -> Result<bool, CryptoError>;
}
```

### Signing Implementation (stfx-crypto-provider/src/signing.rs)

```rust
use async_trait::async_trait;
use crate::{Algorithm, SigningCapability};
use crate::error::CryptoError;

#[async_trait]
impl SigningCapability for Algorithm {
    async fn sign(&self, key: &[u8], message: &[u8]) -> Result<Vec<u8>, CryptoError> {
        match self {
            Algorithm::EdDSA => {
                // Use stfx-crypto-signer-ed25519
                let signing_key = ed25519_dalek::SigningKey::from_bytes(
                    key.try_into().map_err(|_| CryptoError::InvalidKey)?
                );
                let keypair = ed25519_dalek::Keypair {
                    secret: signing_key,
                    public: signing_key.verifying_key(),
                };
                use ed25519_dalek::Signer as DalekSigner;
                Ok(keypair.sign(message).to_bytes().to_vec())
            }
            Algorithm::ES256K => {
                // Use stfx-crypto-signer-secp256k1
                let signing_key = secp256k1::SecretKey::from_slice(key)
                    .map_err(|_| CryptoError::InvalidKey)?;
                let msg = secp256k1::Message::from_digest_slice(message)
                    .map_err(|_| CryptoError::SigningFailed)?;
                let sig = secp256k1::SECP256K1.sign_recoverable(&msg, &signing_key);
                Ok(sig.serialize_compact().1.to_vec())
            }
            Algorithm::ES256 => {
                // Use stfx-crypto-signer-p256
                todo!("P-256 signing")
            }
            #[cfg(feature = "pq")]
            Algorithm::MlDsa65 => {
                todo!("ML-DSA-65 signing")
            }
            #[cfg(feature = "bbs")]
            Algorithm::BBS => {
                todo!("BBS signing")
            }
            other => Err(CryptoError::UnsupportedAlgorithm(other.name().to_string())),
        }
    }
}
```

### Verification Implementation (stfx-crypto-provider/src/verification.rs)

```rust
use async_trait::async_trait;
use crate::{Algorithm, VerificationCapability};
use crate::error::CryptoError;

#[async_trait]
impl VerificationCapability for Algorithm {
    async fn verify(
        &self,
        public_key: &[u8],
        message: &[u8],
        signature: &[u8],
    ) -> Result<bool, CryptoError> {
        match self {
            Algorithm::EdDSA => {
                // Use stfx-crypto-signer-ed25519
                let verify_key = ed25519_dalek::VerifyingKey::from_bytes(
                    public_key.try_into().map_err(|_| CryptoError::InvalidKey)?
                ).map_err(|_| CryptoError::InvalidKey)?;
                
                let sig_bytes: [u8; 64] = signature.try_into()
                    .map_err(|_| CryptoError::InvalidSignature)?;
                let signature = ed25519_dalek::Signature::from_bytes(&sig_bytes);
                
                Ok(verify_key.verify(message, &signature).is_ok())
            }
            Algorithm::ES256K => {
                // Use stfx-crypto-signer-secp256k1
                let public_key = secp256k1::PublicKey::from_slice(public_key)
                    .map_err(|_| CryptoError::InvalidKey)?;
                let msg = secp256k1::Message::from_digest_slice(message)
                    .map_err(|_| CryptoError::VerificationFailed)?;
                let signature = secp256k1::RecoverableSignature::from_compact(
                    signature,
                    secp256k1::RecoveryId::Zero, // This is a simplification
                ).map_err(|_| CryptoError::InvalidSignature)?;
                
                Ok(secp256k1::SECP256K1.verify(&msg, &signature.to_standard(), &public_key).is_ok())
            }
            _ => Err(CryptoError::UnsupportedAlgorithm(self.name().to_string())),
        }
    }
}
```

### Layer 2 Usage Pattern

```rust
use stfx_crypto_provider::Algorithm;
use std::collections::HashMap;

pub struct DIDCommVerifier {
    algorithms: HashMap<String, Algorithm>,
}

impl DIDCommVerifier {
    pub fn new() -> Self {
        let mut algorithms = HashMap::new();
        algorithms.insert("EdDSA".to_string(), Algorithm::EdDSA);
        algorithms.insert("ES256K".to_string(), Algorithm::ES256K);
        algorithms.insert("ES256".to_string(), Algorithm::ES256);
        
        Self { algorithms }
    }
    
    pub async fn verify_proof(
        &self,
        proof: &Proof,
        public_key: &[u8],
    ) -> Result<bool> {
        // Get algorithm from proof metadata (from JWK, etc.)
        let algorithm = self.algorithms
            .get(&proof.algorithm)
            .ok_or(CryptoError::UnsupportedAlgorithm(proof.algorithm.clone()))?;
        
        // One line, no generics!
        algorithm.verify(public_key, &proof.message, &proof.signature).await
    }
}

// Usage: add algorithms at runtime
let mut verifier = DIDCommVerifier::new();
verifier.algorithms.insert(
    "ES384".to_string(),
    Algorithm::ES384,
);

// Works with any supported algorithm
let is_valid = verifier.verify_proof(&proof, &pubkey).await?;
```

---

## Layer 3: Convenience Type (Simplicity)

### Module Structure

```
stfx-crypto-standard-tsp/
├── Cargo.toml
├── src/
│   ├── lib.rs          # Re-export StandardTSPCrypto
│   ├── crypto.rs       # struct StandardTSPCrypto
│   └── error.rs
```

### Concrete Type Definition (stfx-crypto-standard-tsp/src/crypto.rs)

```rust
use async_trait::async_trait;
use stfx_crypto_signer_ed25519::{Ed25519Signer, Ed25519Verifier};
use stfx_crypto_cipher_hpke::HpkeSealer;
use stfx_crypto_hasher_sha256::Sha256Hasher;
use crate::error::TSPCryptoError;

/// All cryptographic operations needed for TSP in one type
/// Zero generic parameters, zero thinking required
pub struct StandardTSPCrypto {
    signer: Ed25519Signer,
    verifier: Ed25519Verifier,
    sealer: HpkeSealer,
    hasher: Sha256Hasher,
}

impl StandardTSPCrypto {
    /// Create with a specific Ed25519 keypair
    pub fn new(keypair: ed25519_dalek::Keypair) -> Self {
        Self {
            signer: Ed25519Signer::new(keypair),
            verifier: Ed25519Verifier,
            sealer: HpkeSealer::default(),
            hasher: Sha256Hasher,
        }
    }
    
    /// Create from a random seed
    pub fn from_seed(seed: &[u8; 32]) -> Self {
        let signer = Ed25519Signer::from_seed(seed);
        Self {
            signer,
            verifier: Ed25519Verifier,
            sealer: HpkeSealer::default(),
            hasher: Sha256Hasher,
        }
    }
    
    /// Sign a message (TSP: [envelope || payload])
    pub async fn sign(&self, message: &[u8]) -> Result<Vec<u8>, TSPCryptoError> {
        self.signer.sign(message)
            .await
            .map_err(TSPCryptoError::SigningFailed)
    }
    
    /// Verify a signature
    pub async fn verify(
        &self,
        message: &[u8],
        signature: &[u8],
        public_key: &[u8],
    ) -> Result<bool, TSPCryptoError> {
        self.verifier.verify(message, signature, public_key)
            .await
            .map_err(TSPCryptoError::VerificationFailed)
    }
    
    /// Seal (encrypt) a message to recipient
    pub async fn seal(
        &self,
        plaintext: &[u8],
        recipient_public_key: &[u8],
    ) -> Result<Vec<u8>, TSPCryptoError> {
        self.sealer.seal(plaintext, recipient_public_key, &[])
            .await
            .map_err(TSPCryptoError::EncryptionFailed)
    }
    
    /// Open (decrypt) a sealed message
    pub async fn open(
        &self,
        ciphertext: &[u8],
        ephemeral_public_key: &[u8],
    ) -> Result<Vec<u8>, TSPCryptoError> {
        // Implementation would use internal opener
        todo!()
    }
    
    /// Hash data with SHA-256
    pub fn hash(&self, data: &[u8]) -> Vec<u8> {
        self.hasher.hash(data)
    }
    
    /// Get the signer's public key
    pub fn public_key(&self) -> Result<Vec<u8>, TSPCryptoError> {
        self.signer.public_key()
            .map_err(TSPCryptoError::KeyError)
    }
}

impl Default for StandardTSPCrypto {
    fn default() -> Self {
        let keypair = ed25519_dalek::SigningKey::generate(rand::thread_rng());
        let keypair = ed25519_dalek::Keypair {
            secret: keypair,
            public: keypair.verifying_key(),
        };
        Self::new(keypair)
    }
}
```

### Layer 3 Usage Pattern

```rust
use stfx_crypto_standard_tsp::StandardTSPCrypto;

pub struct TSPApp {
    crypto: StandardTSPCrypto,  // That's it! No generics!
}

impl TSPApp {
    pub async fn send_message(
        &self,
        envelope: &[u8],
        payload: &[u8],
    ) -> Result<TSPMessage> {
        // Concatenate
        let msg = [envelope, payload].concat();
        
        // Sign - direct method, no trait bounds
        let sig = self.crypto.sign(&msg).await?;
        
        // Encrypt
        let encrypted = self.crypto.seal(payload, &RECIPIENT_PUBKEY).await?;
        
        Ok(TSPMessage {
            envelope: envelope.to_vec(),
            payload: payload.to_vec(),
            signature: sig,
            encrypted_payload: encrypted,
        })
    }
    
    pub async fn receive_message(
        &self,
        msg: &TSPMessage,
        sender_pubkey: &[u8],
    ) -> Result<bool> {
        let msg_bytes = [&msg.envelope[..], &msg.payload[..]].concat();
        
        // Verify - direct method, no trait bounds
        self.crypto.verify(&msg_bytes, &msg.signature, sender_pubkey).await
    }
}

// Usage: create and use - zero complexity
let app = TSPApp {
    crypto: StandardTSPCrypto::from_seed(&SEED),
};

app.send_message(&envelope, &payload).await?;
```

---

## Summary: Choosing Your Layer

```
┌─────────────────────────────────────┐
│ Need pluggable crypto impls?        │ → Use Layer 1: Traits
│ (Hardware, custom protocols)        │
└─────────────────────────────────────┘
         │ No
         ↓
┌─────────────────────────────────────┐
│ Need multiple algorithms?           │ → Use Layer 2: Algorithm Enum
│ (DIDComm, W3C VC)                   │
└─────────────────────────────────────┘
         │ No
         ↓
┌─────────────────────────────────────┐
│ Fixed algorithm protocol? (TSP)     │ → Use Layer 3: Concrete Type
│ Minimal dependencies?               │
└─────────────────────────────────────┘
```

All layers are **equally valid**. Users choose based on their actual needs, not theoretical perfection.

