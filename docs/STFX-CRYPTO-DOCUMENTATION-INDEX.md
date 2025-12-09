# 📑 STFX Crypto Documentation Index

## Quick Navigation

### 🎯 Start Here
- **[stfx-crypto.md](./stfx-crypto.md)** ← Current spec (trait-only approach)
  - Focused traits: Signer, Verifier, AEAD/HPKE, Hasher, KDF
  - Rationale: ISP-compliant, no bundled layers or profiles
- **[00-RESOLUTION-OVER-MODULARIZATION.md](./00-RESOLUTION-OVER-MODULARIZATION.md)**
  - Decision: Keep core crypto trait-only; three-layer exploration retained as historical record
  - Research summary and rationale

---

## 📚 Documentation Set

### Current (Trait-Only Baseline)

1. **[stfx-crypto.md](./stfx-crypto.md)**
   - Core crypto traits and rationale
   - ISP-focused design, no bundled profiles
   - Usage examples (direct trait wiring)
   - SOLID compliance verification

2. **[stfx-crypto-api-analysis.md](./stfx-crypto-api-analysis.md)**
   - Evidence from real-world crypto APIs (TSP, ssi, Aries)
   - Why traits remain the stable core

### Historical (Three-Layer Exploration)

3. **[STFX-CRYPTO-DESIGN-SUMMARY.md](./STFX-CRYPTO-DESIGN-SUMMARY.md)**
   - Archived three-layer summary (Layer 1 traits, Layer 2 enum, Layer 3 bundle)
   - Kept for reference only

4. **[stfx-crypto-implementation-layers.md](./stfx-crypto-implementation-layers.md)**
   - Archived code examples for the three-layer experiment
   - Not the current recommended approach

5. **[STFX-CRYPTO-VISUAL-SUMMARY.md](./STFX-CRYPTO-VISUAL-SUMMARY.md)**
   - Archived visuals/diagrams of the three-layer approach

---
## 🗺️ Reading Paths

### Path A: "I want the current spec"
1. stfx-crypto.md
2. stfx-crypto-api-analysis.md (optional evidence)

### Path B: "I want the historical three-layer exploration"
1. STFX-CRYPTO-DESIGN-SUMMARY.md (archived)
2. stfx-crypto-implementation-layers.md (archived examples)
3. STFX-CRYPTO-VISUAL-SUMMARY.md (archived diagrams)

---

## 📊 Related Documentation

### Architecture & Principles
- **[architecture.md](./architecture.md)** - Overall STFX architecture
- **[arch_DIP_SOLID.md](./arch_DIP_SOLID.md)** - DIP and SOLID principles analysis
- **[architecural_principles.md](./architecural_principles.md)** - Core principles

### Other STFX Modules
- **[stfx-transport.md](./stfx-transport.md)** - Transport layer (message delivery)
- **[stfx-runtime.md](./stfx-runtime.md)** - Runtime execution engine
- **[stfx-runtime_decision_analysis.md](./stfx-runtime_decision_analysis.md)** - Runtime design decisions
- **[VID-IMPLEMENTATION-ANALYSIS.md](./VID-IMPLEMENTATION-ANALYSIS.md)** - Verified Identifier implementation

### Identity
- **[STF-IDENTITY/canonicaldid.md](./STF-IDENTITY/canonicaldid.md)** - Canonical DID specification

---

## 📝 Document Metadata

| Document | Purpose | Current/History |
|----------|---------|-----------------|
| stfx-crypto.md | Core trait-only spec and usage | Current |
| stfx-crypto-api-analysis.md | Evidence from production systems (historical 3-layer analysis) | Historical |
| 00-RESOLUTION-OVER-MODULARIZATION.md | Decision record (trait-only chosen) | Current |
| STFX-CRYPTO-DESIGN-SUMMARY.md | Archived three-layer summary | Historical |
| stfx-crypto-implementation-layers.md | Archived three-layer examples | Historical |
| STFX-CRYPTO-VISUAL-SUMMARY.md | Archived three-layer diagrams | Historical |

---

## 🎯 Status

**Analysis:** ✅ COMPLETE
- Real-world systems analyzed
- Evidence collected and documented
- Design patterns identified
- Recommendation formulated

**Documentation:** ✅ COMPLETE
- 5 comprehensive documents created
- Code examples provided
- Decision trees included
- Architecture verified against SOLID

**Next Steps:** 
- ⏳ Review and validation
- ⏳ Implementation of Layer 1 trait crates
- ⏳ Implementation of Layer 2 Algorithm enum
- ⏳ Implementation of Layer 3 convenience types

---

## 📞 Quick Reference Links

- **Main crypto docs:** [stfx-crypto.md](./stfx-crypto.md)
- **Three-layer guide:** [00-RESOLUTION-OVER-MODULARIZATION.md](./00-RESOLUTION-OVER-MODULARIZATION.md)
- **Implementation examples:** [stfx-crypto-implementation-layers.md](./stfx-crypto-implementation-layers.md)
- **Evidence & analysis:** [stfx-crypto-api-analysis.md](./stfx-crypto-api-analysis.md)
- **Executive summary:** [STFX-CRYPTO-DESIGN-SUMMARY.md](./STFX-CRYPTO-DESIGN-SUMMARY.md)

