# Known Issues in x402-rs SKILL.md

Date: April 2026

This document records verified errors discovered while testing the x402-rs skill against real crates on crates.io, along with real-world fixes demonstrated in working test projects.

---

## Issue 1: `USDC::base_sepolia()` requires `KnownNetworkEip155` trait in scope

**Status:** VERIFIED — crate `x402-chain-eip155` 1.4.6 on crates.io

**Skill Error Locations:**

- `SKILL.md` line 89, line 165, line 171, line 254, line 271
- `references/server-guide.md` line 42, line 64, line 91, line 117, line 141, line 230
- `references/chain-config.md` line 55, line 56, line 63
- `README.md` line 64

**Original (broken) code:**

```rust
use x402_types::networks::USDC;
use x402_chain_eip155::V2Eip155Exact;

let price_tag = V2Eip155Exact::price_tag(
    address!("0x..."),
    USDC::base_sepolia().amount(10u64),
);
```

**Actual compiler error:**

```
error[E0599]: no function or associated item named `base_sepolia` found for struct `USDC` in the current scope
   = help: trait `KnownNetworkEip155` which provides `base_sepolia` is implemented but not in scope; perhaps you want to import it
```

**Root Cause:**
`USDC` is a marker struct defined in `x402_types::networks`. The `base_sepolia()` method comes from `impl KnownNetworkEip155<Eip155TokenDeployment> for USDC` in `x402-chain-eip155`. Because `KnownNetworkEip155` is a generic trait, Rust requires it to be in scope to resolve the method on the concrete type.

**Fix demonstrated in `/test-projects/server/src/main.rs`:**

```rust
use x402_chain_eip155::{V2Eip155Exact, KnownNetworkEip155};
use x402_types::networks::USDC;

let price_tag = V2Eip155Exact::price_tag(
    address!("0x..."),
    USDC::base_sepolia().amount(10u64),
);
```

---

## Issue 2: `alloy-signer-local` version mismatch (skill says `0.8`, crate requires `1.4`)

**Status:** VERIFIED — crate `x402-chain-eip155` 1.4.6 on crates.io

**Skill Error Locations:**

- `SKILL.md` line 107
- `references/client-guide.md` line 12
- `README.md` line 74

**Original (broken) Cargo.toml:**

```toml
alloy-signer-local = "0.8"
```

**Actual compiler error:**

```
error: failed to select a version for `c-kzg`.
... required by package `alloy-signer-local v0.8.0`
package `c-kzg` links to the native library `ckzg`, but it conflicts with a previous package
```

**Root Cause:**
`x402-chain-eip155` declares dependency `alloy-signer-local = "1.4"`. Using `0.8` pulls in an incompatible native-linking `c-kzg` version that conflicts with the `alloy` 1.4 dependency graph used by `x402-chain-eip155`.

**Fix demonstrated in `/test-projects/client/Cargo.toml`:**

```toml
alloy-signer-local = "1.4"
```

---

## Issue 3: Solana dependency ecosystem has transitive `spl-token-2022` compilation failure

**Status:** VERIFIED — crates.io as of April 2026

**Skill Error Locations:**

- `SKILL.md` line 138 (`use solana_keypair::Keypair;`)
- `SKILL.md` line 139 (`use solana_client::nonblocking::rpc_client::RpcClient;`)
- `references/client-guide.md` line 72, line 94, line 95
- `references/server-guide.md` line 135 (`use solana_pubkey::pubkey;`)

**Actual compiler error:**

```
error[E0308]: mismatched types
  --> spl-token-2022-10.0.0/src/extension/token_group/processor.rs
   | check_update_authority(group_update_authority_info, &group.update_authority)?;
   | expected `&OptionalNonZeroPubkey`, found `&MaybeNull<Pubkey>`
```

**Root Cause:**
The Solana ecosystem crates (e.g., `solana-client`, `solana-keypair`, `solana-pubkey`) have version conflicts with `spl-token-2022` (transitive dependency of `x402-chain-solana`). The skill's snippet uses `solana_client::nonblocking::rpc_client::RpcClient` and `solana_keypair::Keypair`, but these paths and versions may not align with what `x402-chain-solana` compiled against. Specifically, `x402-chain-solana` 1.4.6 depends on `solana-sdk` 2.x series while `solana-client` 3.1/4.x pulls `spl-token-2022` 10.0.0 which is type-incompatible.

**Workaround demonstrated:**
Avoid mixing `x402-chain-solana` with loose `solana-client` / `solana-keypair` versions. Use exact Solana SDK versions from the `x402-chain-solana` dependency graph, OR gate Solana code behind feature flags in a workspace.

A **minimal correct import** (without compiling the full Solana graph) is to use only the re-exported types from `x402-chain-solana` itself if available, or pin Solana crates to the versions locked by `x402-chain-solana`.

---

## Issue 4: `settle_after_execution` / `settle_before_execution` return different types and require `Clone`

**Status:** PARTIALLY VERIFIED — does not compile as chained methods in skill

**Skill Error Locations:**

- `SKILL.md` line 283–287
- `references/server-guide.md` line 170–178

**Original (problematic) code from skill:**

```rust
let x402 = X402Middleware::new("https://facilitator.x402.rs")
    .settle_after_execution();
```

**Root Cause:**

- `settle_after_execution` returns `Self` (which is `X402Middleware<F>`), but `with_price_tag` is defined on `X402Middleware<TFacilitator>` where `TFacilitator: Clone`.
- If the user chains `.settle_after_execution()` before `.with_price_tag(...)`, the compiler may fail because `F` must satisfy `Clone`, which `Arc<FacilitatorClient>` does, but the builder flow in the skill implies you can call these on the same object after `with_price_tag`, which is impossible because `with_price_tag` returns `X402LayerBuilder`, not `X402Middleware`.
- The **dynamic pricing example** in the skill is broken: it calls `x402.with_dynamic_price(...).with_description(...)` but `with_dynamic_price` returns `X402LayerBuilder`, and `with_description` is a method on `X402LayerBuilder` — correct. However, `with_price_tag` returns `X402LayerBuilder<StaticPriceTags<TPriceTag>, TFacilitator>`, and further `with_price_tag` is only available when `TPriceTag: Clone`.

**Fix demonstrated in `/test-projects/dynamic-pricing/src/main.rs`:**
Call `with_dynamic_price` on `X402Middleware` (which returns a builder) and then chain `with_description` and `with_mime_type` on the builder. This works.

---

## Issue 5: `amount()` in dynamic pricing example requires explicit `u64` type due to `Into<u64>` bound

**Status:** VERIFIED

**Skill Error Locations:**

- `SKILL.md` line 251 (dynamic pricing example)
- `references/server-guide.md` line 87

**Original (broken) code:**

```rust
let amount = if has_discount { 50 } else { 100 };
```

**Actual compiler error:**

```
the trait bound `u64: From<i32>` is not satisfied
```

**Fix demonstrated in `/test-projects/dynamic-pricing/src/main.rs`:**

```rust
let amount = if has_discount { 50u64 } else { 100u64 };
```

---

## Summary of Skill Fixes Applied

| #   | File(s)                                    | Fix                                                                                                                            |
| --- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| 1   | All code examples using `USDC::*`          | Add `use x402_chain_eip155::KnownNetworkEip155;`                                                                               |
| 2   | `SKILL.md`, `client-guide.md`, `README.md` | Change `alloy-signer-local = "0.8"` to `alloy-signer-local = "1.4"`                                                            |
| 3   | `SKILL.md`, `client-guide.md`              | Add note about Solana transitive dep conflicts and exact version pinning                                                       |
| 4   | `SKILL.md`, `server-guide.md`              | Clarify that `settle_before_execution` / `settle_after_execution` are methods on `X402Middleware`, not on the returned builder |
| 5   | `SKILL.md`, `server-guide.md`              | Use `50u64` / `100u64` in dynamic pricing examples                                                                             |
