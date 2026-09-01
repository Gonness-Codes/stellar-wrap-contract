# Architecture Decision: In-Guest Ed25519 Signature Verification vs Host Crypto Primitive

## Context
Issue #721 requested a comprehensive performance and architectural re-evaluation of in-guest Ed25519 signature verification (`ed25519-dalek`) in `src/signature.rs` against the Soroban WASM bytecode size budget (200 KB / 204,800 bytes).

In `src/signature.rs`, the contract currently embeds `ed25519-dalek` to verify off-chain administrator signatures on mint and batch-mint operations. This was originally implemented so that signature rejection errors map to a structured guest error code (`ContractError::InvalidSignature`), avoiding uncatchable VM host errors from `soroban_sdk::Env::crypto().ed25519_verify`.

However, embedding cryptographic curves and arithmetic in the guest contract incurs substantial bytecode overhead. This document records empirical size measurements, toolchain configurations, operational trade-offs, and dependency pin analysis.

---

## Toolchain & Build Environment

The measurements were collected using the official Soroban compilation target on the active repository toolchain:

- **Rust Compiler**: `rustc 1.94.1 (e408947bf 2026-03-25)`
- **Compilation Target**: `wasm32v1-none` (official Soroban Protocol 22 / SDK 27 target)
- **Soroban SDK**: `soroban-sdk = "=27.0.3"`, `soroban-env-host = "=27.0.1"`, `soroban-env-common = "=27.0.1"`
- **Release Profile Flags**:
  ```toml
  [profile.release]
  opt-level = "z"
  lto = true
  panic = "abort"
  codegen-units = 1
  ```
- **Standard Soroban Bytecode Budget**: 200 KB (`204,800 bytes`)

---

## Empirical Measurements

Both implementations were compiled under identical optimization flags and measured:

| Configuration | WASM Size (Bytes) | WASM Size (KB) | % of 200 KB Budget | Budget Status | Size Delta vs In-Guest |
|:---|:---:|:---:|:---:|:---:|:---:|
| **In-Guest `ed25519-dalek`** (`verify_strict`) | **238,040 B** | **232.46 KB** | **116.23%** | ❌ **Exceeds Budget (+33,240 B / +16.23%)** | Baseline |
| **Host Primitive `e.crypto().ed25519_verify`** | **192,533 B** | **188.02 KB** | **94.01%** | ✅ **Within Budget (-12,267 B / -5.99% headroom)** | **-45,507 B (-44.44 KB / -19.12%)** |

### Breakdown of In-Guest Overhead
- `ed25519-dalek` + `curve25519-dalek` contributes **45,507 bytes** (**44.44 KB**) of compiled WASM bytecode.
- In-guest signature verification represents **19.12%** of the entire contract binary size.

---

## Trade-off Analysis

### 1. In-Guest Verification (`ed25519-dalek`)
- **Advantages**:
  - **Structured Guest Error Handling**: All rejection modes (malformed public key, invalid signature, corrupted payload, altered domain separator) return `Err(ContractError::InvalidSignature)`.
  - **Contract-Level Interoperability**: Caller contracts and downstream client SDKs can pattern-match on `ContractError::InvalidSignature` rather than handling a fatal transaction trap.
- **Disadvantages**:
  - **Exceeds 200 KB Size Limit**: At 238,040 bytes, the contract binary exceeds the standard Soroban 200 KB budget by 33,240 bytes.
  - **Dependency Maintenance**: Requires managing `ed25519-dalek`, `curve25519-dalek`, and `subtle` with explicit `default-features = false` to prevent pulling in `std` on `no_std` targets.

### 2. Host Crypto Primitive (`e.crypto().ed25519_verify`)
- **Advantages**:
  - **Significant Size Savings**: Shrinks the binary by 45,507 bytes (to 192,533 bytes), keeping the contract safely below the 200 KB limit with ~12.2 KB of headroom.
  - **Zero Guest Crypto Dependency**: Offloads elliptic curve operations and signature verification to host-native C++/Rust implementations in `soroban-env-host`.
  - **Deterministic Gas Costs**: Host crypto operations are metered according to calibrated host cost models rather than guest WASM instruction metering.
- **Disadvantages**:
  - **Host Error on Failure**: On an invalid signature or invalid key, `soroban_sdk::Env::crypto().ed25519_verify` halts execution with `HostError: Error(Crypto, InvalidInput)`. The guest does not regain control to emit a custom `ContractError`.

---

## Dependency Pin Re-Verification (`ed25519-dalek`)

The previous comment in `Cargo.toml` stated:
> *"Pin ed25519-dalek to match soroban-env-host's internal requirement for unification across the dependency tree... ed25519-dalek = { version = "=3.0.0", features = ["rand_core"] }"*

### Re-verification Findings:
1. **`soroban-env-host v27.0.1` Dependency**:
   - `soroban-env-host 27.0.1` depends on `ed25519-dalek v2.2.0` (with `ed25519 v2.2.3`), rather than `v3.0.0`.
2. **`no_std` Target Requirement (`wasm32v1-none`)**:
   - For `wasm32v1-none` (the standard Soroban target in Rust 1.84+), `default-features = false` MUST be explicitly declared:
     ```toml
     ed25519-dalek = { version = "=2.2.0", default-features = false, features = ["rand_core"] }
     ```
   - Omitting `default-features = false` causes Cargo to resolve default features (`std`), failing compilation on `no_std` bare-metal targets.

**Conclusion**: Because `soroban-env-host v27.0.1` depends on `ed25519-dalek v2.2.0`, the previous pin to `=3.0.0` is no longer grounded in dependency unification. Since the in-guest path is retained behind the `in-guest-ed25519` feature flag, the pin will be updated to `=2.2.0` to match the host and avoid unnecessary version divergence.

---

## Decision & Path Forward

**Decision**: Based on the measurements above, the in-guest `ed25519-dalek` dependency adds 45,507 bytes (19.12% of binary size) and pushes the contract over the 200 KB budget (238,040 B > 204,800 B). We therefore **switch the default verification path to the host primitive** `e.crypto().ed25519_verify`. The host error on invalid signatures (`Error(Crypto, InvalidInput)`) will be accepted and documented as the canonical failure signal; client SDKs should map it to an `InvalidSignature`-equivalent error. Pre-validation of signature shape is not required because the host primitive already checks all inputs deterministically.

1. **Default Implementation**:
   - The host primitive `e.crypto().ed25519_verify` is now the default verification method, ensuring the binary fits within the 200 KB budget with 12,267 bytes of headroom.
   - The in-guest `ed25519-dalek` path is retained behind the `in-guest-ed25519` feature flag for contracts that require structured guest errors and have a relaxed size budget.
2. **Manifest Fixes**:
   - Updated `Cargo.toml` to enforce `default-features = false` on `ed25519-dalek`, ensuring reliable `wasm32v1-none` builds across all modern toolchains.
   - Documented the exact `soroban-env-host` dependency relation in `Cargo.toml`.
3. **Continuous WASM Size Monitoring**:
   - Implemented `scripts/check_wasm_size.sh` and a Makefile target `check-wasm-size`.
   - Integrated automated WASM size tracking into GitHub Actions CI (`.github/workflows/ci.yml`) to track binary growth and alert on budget thresholds.
