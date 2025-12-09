---
title: Canonical DID (cDID)
description: The canonical identity model used inside STF Layer 1 (STF-IDENTITY) for privacy-preserving identity abstraction.
---

# Canonical DID (cDID)

The **canonical DID (cDID)** is the internal, never-exposed, privacy-preserving identity anchor used by the **Sovereign Trust Framework (STF)**.  
It is the foundation of **STF-IDENTITY**, providing a common internal identity abstraction independent of DID methods, transport protocols, or VID formats.

The cDID provides:

- A **stable identity anchor** for local trust and reputation  
- **Strong unlinkability** across sessions and relationships  
- **Mapping** between external identifiers (DIDs, VIDs, others)  
- Integration with STF-POLICY (VSTP) and STF-COMM layers  
- A **unifying identity representation** for all STF components  

The cDID is **never shared over the network** and **never written to a ledger**.

---

# 1. Purpose of the Canonical DID

STF is identity-agnostic: DIDs, VIDs, peer DIDs, ephemeral identifiers, and transport-native IDs may all appear in Layer 2.

However, STF requires **one stable, internal identifier** for:

- Policy evaluation (STF-POL)  
- Reputation aggregation (STF-REPUTE)  
- Attestation weighting (STF-ATT)  
- Message verification context (STF-Message)  
- Local identity indexing (STF-IDX)  

Thus the cDID serves as:

**“A canonical internal representation of an external identity, derived in a privacy-preserving way.”**

---

# 2. What the cDID Is *Not*

To clarify its role:

❌ It is **not** a public DID  
❌ It is **not** resolvable  
❌ It is **not** a fixed identifier shared between devices  
❌ It is **not** method-specific  
❌ It is **not** used for routing or transport  

The cDID is **local-only** and **device-bound**.

---

# 3. How cDID Fits into STF Identity Layer

```
external identifiers → eDID (ephemeral) → cDID (canonical, internal)
```

Supported external identifiers:

- DID (all methods)
- DID:peer
- TSP VID
- Derived session identifiers
- Application-native identifiers (optional)
- Phone/email (optional mapping, privacy-safe if hashed)
- QR-scanned contact handles
- Local contact book entries

The canonical DID becomes the **internal identity anchor**.

---

# 4. Formal Definition of cDID

A **cDID** is a deterministic, salted, device-local identifier generated from an input identity:

```
cDID = hash( normalize(external_identifier) + salt_epoch + device_secret )
```

Characteristics:

- **Deterministic within an epoch**  
- **Different across devices**  
- **Different across epochs**  
- **Impossible to reconstruct externally**  
- **Impossible to link across epochs**  

The normalization step ensures input format neutrality.

---

# 5. Normalization Function

The input identifier must be canonicalized before hashing.

```
normalized_id = Normalize(external_identifier)
```

Normalization rules:

1. Trim whitespace.
2. Lowercase unless case-sensitive DID spec requires otherwise.
3. Collapse equivalent encodings (`did:key:z...` vs multibase raw key).
4. For VIDs, turn them into UTF-8 safe binary form.
5. For QR-based identifiers or phone/email:
   - Normalize email: lowercase, remove dots for Gmail-like systems.
   - Normalize phone: E.164 format.

This ensures stable results even if identifiers come in multiple formats.

---

# 6. Derivation Algorithm (Reference Implementation)

```
function derive_cdid(external_identifier):
    normalized = Normalize(external_identifier)
    epoch_salt = current_salt()
    input = normalized || epoch_salt || device_secret
    return H(input)
```

Where:

- `H` = SHA-256 / BLAKE3 (configurable)
- `device_secret` = secret key only stored locally
- `epoch_salt` = rotating salt (see section below)

Output:

- 256-bit canonical identity string
- Represented as: `cdid:z<multibase-encoded-value>`

Example:

```
cdid:z4Bx9Kk2vWuF1g8qZdD9tP9yLN8oGfP7F2kzMhZq9mSs
```

---

# 7. Salt Rotation

The cDID must evolve over time to preserve unlinkability.

**Salt rotation schedule:**

- Rotate every **N days** (default: 30 days)
- Rotation triggered when:
  - time threshold reached  
  - device recovers a backup  
  - user rotates keys manually  

**Salt storage:**

- Each salt epoch stored locally:
  - `salt_2025-01`
  - `salt_2025-02`
- Keeps a finite window (e.g., last 3 epochs) to maintain backward compatibility.

**Important:**  
Old cDIDs remain functional for reputation but cannot be linked to new ones externally.

---

# 8. Storage Model

All cDID data resides under **STF-STORAGE**, protected via:

- OS secure storage  
- hardware secure module (TPM/SE/Keystore) where available  
- encrypted local database  

Storage structure:

```
/identity/
    device_secret
    epoch_salts/
        salt_2025-01
        salt_2025-02
        ...
    mapping/
        cdid_index.db   (stores mapping external→cDID)
        reversed_mapping.db (optional, encrypted, prevents leaks)
```

Privacy protections:

- No external identifier is stored in plaintext
- External identifiers hashed and salted with a different salt
- Mapping is one-way unless device_secret is known

---

# 9. How cDID Interacts with Other STF Components

## 9.1 With STF-Message (L2)
- When a message arrives with DID/VID:
  - normalize and derive cDID  
  - load associated state (session keys, policies, reputation, etc.)  
- Outgoing messages **never contain the cDID**.

## 9.2 With STF-Transport (L2)
- VID → cDID mapping performed before message envelope processing  
- DIDComm route resolution uses the mapped cDID for session context  

## 9.3 With STF-POLICY (L3)
- Policy evaluation uses cDID as the **subject**  
- Canonical claims attach to the cDID  
- Rego rules refer to cDID instead of DID  

Example:

```
allow_response {
   trust_score[cDID] > 75
}
```

## 9.4 With STF-REPUTE (L3)
- Reputation is stored per-cDID  
- Attestations apply to canonical identities  

## 9.5 With STF-ATT (L3)
- Attestations are bound to cDID locally  
- Device enforces: "Issued-to" cDID must match derived value  

---

# 10. When cDID is Recomputed vs. Reused

| Scenario | Uses new cDID? | Notes |
|----------|----------------|-------|
| Same user, same epoch | ❌ | Stable identity anchor |
| New epoch (salt rotates) | ✔ | Preserves unlinkability |
| Device restored from backup | ❌ (if device_secret preserved) | Stable identity |
| Multi-device use | ✔ | No cross-device linking |
| User resets identity | ✔ | New device_secret |

---

# 11. Should cDID Use Its Own DID Method?

cDID can optionally be defined as a private DID method:

```
did:stf:<encoded_value>
```

But this is only recommended for:

- internal debugging
- optional internal API traces
- representing cDID-like objects in logs

**IMPORTANT:**  
This DID method MUST NEVER be transmitted externally.

---

# 12. Example cDID Lifecycle

```
1. Bob receives Alice’s DID or VID
2. STF performs Normalize(DID)
3. Epoch salt is applied
4. cDID derived
5. Trust score and attestations linked to this cDID
6. Next month salt rotates
7. Old cDID archived as "epoch-1"
8. New cDID used going forward
```

---

# 13. Security & Privacy Guarantees

- **Unlinkability:** No two parties can correlate Alice’s identities.
- **Local-only:** No network exposure.
- **Sybil mitigation:** Combined with STF-ATT and STF-REPUTE.
- **No ledger dependency.**
- **Minimal attack surface.**

---

# 14. Future Extensions

- Multi-device canonical identity synchronization with privacy guarantees  
- Multi-party blinded attestations  
- cDID versioning for DIDComm 2.0 compatibility  
- Selective correlation via user-consented export  

---

# 15. Summary

The **canonical DID (cDID)** is the foundational identity abstraction mechanism of STF Layer 1.

It:

- unifies all external identifiers  
- ensures privacy and unlinkability  
- anchors policy and reputation  
- integrates deeply across STF layers  
- is never exposed  
- is derived using deterministic, rotating, privacy-preserving methods  

This mechanism ensures STF remains **privacy-first**, **transport-agnostic**, **DID-method-agnostic**, and **future-proof**.

