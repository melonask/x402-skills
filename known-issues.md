# Known Issues with x402-rs SKILL.md (April 2026)

This file documents real-world compilation and runtime issues discovered while testing every code example from the `x402-rs` skill documentation against the latest published crates (`1.4.6`). All issues were reproduced and fixed in a production workspace using real dependencies (not mocked).

**Environment:** `rustc 1.95.0`, `cargo 1.95.0`, Linux x86_64.  
**Crates tested:** `x402-axum 1.4.6`, `x402-reqwest 1.4.6`, `x402-chain-eip155 1.4.6`, `x402-types 1.4.6`, `x402-facilitator-local 1.4.6` (published 2026-04-14).

---

## Issue 1: Server example missing `Router` type annotation

**Location:** `SKILL.md` lines 75–97, `references/server-guide.md` lines 24–44  
**Severity:** Compilation error  
**Status:** Fixed with explicit type in real project

### The Bug
The skill shows:
```rust
let app = Router::new().route(
    "/paid-content",
    get(handler).layer(
        x402.with_price_tag(...)
    ),
);
```
This fails because `Router::new()` returns `Router<S>` where `S` is the state type, and the chained `.route(...).layer(...)` doesn't allow inference. Compiler error:
```
error[E0283]: type annotations needed for `Router<_>`
```

### The Real Fix
Add an explicit `Router` type annotation:
```rust
let app: Router = Router::new().route(
    "/paid-content",
    get(handler).layer(
        x402.with_price_tag(V2Eip155Exact::price_tag(
            address!("0xBAc675C310721717Cd4A37F6cbeA1F081b1C2a07"),
            USDC::base_sepolia().amount(10u64),
        ))
    ),
);
```
**Verified:** `cargo run` succeeds for `x402-production-server`.

---

## Issue 2: Missing `KnownNetworkEip155` import for `USDC::base_sepolia()`

**Location:** `references/chain-config.md` lines 45–65 (and many server/client snippets)  
**Severity:** Compilation error  
**Status:** Fixed by adding the trait import

### The Bug
Calling `USDC::base_sepolia()` or any `USDC::*` method fails unless the trait `KnownNetworkEip155` is in scope:
```
error[E0599]: no function or associated item named `base_sepolia` found for struct `USDC`
```

### The Real Fix
Always import:
```rust
use x402_chain_eip155::KnownNetworkEip155;
use x402_types::networks::USDC;
```
**Verified:** Required in `x402-production-server` and `x402-production-e2e`.

---

## Issue 3: `X402Middleware` config methods live on `X402LayerBuilder`, not `X402Middleware`

**Location:** `SKILL.md` lines 295–300 (settle timing is correct, `.with_resource`/`.with_mime_type`/`.with_description` are not)  
**Severity:** Compilation error  
**Status:** Fixed – use `X402LayerBuilder` methods

### The Bug
The skill shows:
```rust
let x402 = X402Middleware::new("https://facilitator.x402.rs")
    .with_base_url(Url::parse("https://api.example.com").unwrap())    // correct
    .with_resource(Url::parse("https://api.example.com/premium").unwrap()) // NOT on X402Middleware
    .with_description("Premium API access")                                // NOT on X402Middleware
    .with_mime_type("application/json");                                 // NOT on X402Middleware
```
These methods do not exist on `X402Middleware`; they exist only on `X402LayerBuilder`, which is returned by `with_price_tag(...)` or `with_dynamic_price(...)`.

### The Real Fix
Chain resource/description/mime type on the LayerBuilder:
```rust
let app: Router = Router::new().route(
    "/custom-config",
    get(handler).layer(
        x402.with_price_tag(V2Eip155Exact::price_tag(
            address!("..."),
            USDC::base_sepolia().amount(10u64),
        ))
        .with_description("Premium API access".to_string())
        .with_mime_type("application/json".to_string())
    ),
);
```
**Verified:** compiles successfully in `x402-production-server`.

---

## Issue 4: `price_tag` does NOT accept a plain `String` for amount

**Location:** `SKILL.md` lines 168–173, `references/server-guide.md` lines 71–76  
**Severity:** Compilation error  
**Status:** `price_tag` requires `DeployedTokenAmount`

### The Bug
The skill claims:
```rust
V2Eip155Exact::price_tag(
    address!("0xTokenContractAddress"),
    "1000000".to_string(), // amount in smallest token unit
)
```
This produces:
```
error[E0308]: arguments to this function are incorrect
expected `DeployedTokenAmount<Uint<256, 4>, Eip155TokenDeployment>`, found `String`
```
The second argument must be a `DeployedTokenAmount`, obtained only from:
- `USDC::base_sepolia().amount(1000000u64)`
- `USDC::base_sepolia().parse("0.01").unwrap()`

### The Real Fix
Use `amount()` or `parse()` on the `Eip155TokenDeployment` type from `USDC`:
```rust
V2Eip155Exact::price_tag(
    address!("0x..."),
    USDC::base_sepolia().amount(1000000u64),
)
```
**Verified:** correct signature confirmed by inspecting `x402-chain-eip155/src/v2_eip155_exact/server.rs:44`.

---

## Issue 5: `PaymentSelector` trait has different signatures and `PaymentCandidate` fields are public

**Location:** `references/client-guide.md` lines 147–173  
**Severity:** Compilation error  
**Status:** Fixed in production client

### The Bugs
1. `select` takes a lifetime: `fn select<'a>(&self, candidates: &'a [PaymentCandidate]) -> Option<&'a PaymentCandidate>` – the skill omits lifetimes.
2. `amount` and `chain_id` are **public fields**, not methods. Calling `c.amount()` or `c.network()` fails:
   ```
   error[E0599]: no method named `amount` found for reference `&&PaymentCandidate`
   error[E0599]: no method named `network` found for reference `&&PaymentCandidate`
   ```

### The Real Fix
```rust
struct CheapestSelector;
impl PaymentSelector for CheapestSelector {
    fn select<'a>(&self, candidates: &'a [PaymentCandidate]) -> Option<&'a PaymentCandidate> {
        candidates.iter().min_by_key(|c| c.amount) // field, not method
    }
}

struct ChainPreferenceSelector { preferred_chain: String }
impl PaymentSelector for ChainPreferenceSelector {
    fn select<'a>(&self, candidates: &'a [PaymentCandidate]) -> Option<&'a PaymentCandidate> {
        candidates.iter()
            .find(|c| c.chain_id.to_string().contains(&self.preferred_chain))
            .or_else(|| candidates.first())
    }
}
```
**Verified:** all client tests pass in `x402-production-client`.

---

## Issue 6: `SchemeRegistry::build` signature differs from skill example

**Location:** `references/facilitator-guide.md` lines 199–246  
**Severity:** Compilation error  
**Status:** Fixed in production facilitator

### The Bugs
1. `SchemeRegistry::build` is **not** fallible (returns `Self`, not `Result<Self, ...>`);
2. It needs a `ChainRegistry<P>` where `P: ChainProviderOps` — using `()` will not satisfy this bound unless you provide a real provider like `Eip155ChainProvider`.
3. `SchemeConfig::chains` is `ChainIdPattern`, parsed with `"eip155:*".parse()?`.
4. `SchemeRegistry` has no `::new()`; use `Default::default()` for an empty registry, or `SchemeRegistry::build(...)` for a populated one.

### The Real Fix (minimal compilation test)
```rust
use x402_types::chain::ChainRegistry;
use x402_types::scheme::SchemeRegistry;
use x402_chain_eip155::Eip155ChainProvider;

let chain_registry: ChainRegistry<Eip155ChainProvider> = ChainRegistry::new(HashMap::new());
let blueprints = SchemeBlueprints::new()
    .and_register(V1Eip155Exact)
    .and_register(V2Eip155Exact);
let config: Vec<SchemeConfig> = vec![];
let scheme_registry = SchemeRegistry::build(chain_registry, blueprints, &config);
```
**Verified:** `x402-production-facilitator` compiles.

---

## Issue 7: Solana dependency conflict breaks compilation

**Location:** `SKILL.md` lines 131–153 (multi-chain with Solana)  
**Severity:** Compilation error  
**Status:** Confirmed upstream issue; workaround documented

### The Bug
Adding `x402-chain-solana` pulls `spl-token-2022 v10.0.0`, which has an incompatible API with `solana-native-token v0.1.0`:
```
error[E0308]: expected `&OptionalNonZeroPubkey`, found `&MaybeNull<Pubkey>`
   --> spl-token-2022-10.0.0/src/extension/token_group/processor.rs
```

### The Real Fix
The ecosystem conflict exists on crates.io as of April 2026. Workarounds:
- Use the upstream GitHub repo directly (`git = "https://github.com/x402-rs/x402-rs"`) and its pinned `Cargo.lock`.
- In standalone crates.io projects, omit Solana support and document the limitation.

**Verified:** EVM-only multi-chain client works perfectly.

---

## Issue 8: Skill lists crate version `"1.0"` when actual latest is `1.4.6`

**Location:** All Cargo.toml snippets across the skill  
**Severity:** Non-breaking (semver compatible)  
**Status:** Noted for accuracy

The skill uses `"1.0"` which resolves to `1.4.6` because of semver compatibility. However, for documentation accuracy, the latest published versions as of April 2026 are:

| Crate | Latest |
|-------|--------|
| `x402-axum` | `1.4.6` |
| `x402-reqwest` | `1.4.6` |
| `x402-chain-eip155` | `1.4.6` |
| `x402-chain-solana` | `1.4.6` |
| `x402-chain-aptos` | `1.4.6` (git-only) |
| `x402-types` | `1.4.6` |
| `x402-facilitator-local` | `1.4.6` |
| `x402-facilitator` | not on crates.io |

---

## Verified Working Patterns (tested in production workspace)

| Pattern | Crate / Feature | Status |
|---------|----------------|--------|
| Static pricing server | `x402-axum` + `x402-chain-eip155/server` | Works |
| Dynamic pricing (`with_dynamic_price`) | `x402-axum` + `x402-chain-eip155/server` | Works |
| Conditional free access (empty vec) | `x402-axum` | Works |
| `settle_before_execution()` | `x402-axum` | Works |
| `settle_after_execution()` | `x402-axum` | Works (default) |
| `with_supported_cache_ttl` on middleware | `x402-axum` | Works |
| `with_description`, `with_mime_type` on LayerBuilder | `x402-axum` | Works |
| `USDC::base_sepolia().amount(u64)` | `x402-chain-eip155` + `KnownNetworkEip155` | Works |
| `USDC::base_sepolia().parse("0.01")` | `x402-chain-eip155` + `KnownNetworkEip155` | Works |
| `USDC::base()`, `polygon()`, `avalanche()`, `sei()`, `xdc()` etc. | `x402-chain-eip155` + `KnownNetworkEip155` | Works |
| Single-chain EVM client | `x402-reqwest` + `x402-chain-eip155/client` | Works |
| Multi-chain EVM client (V1 + V2) | `x402-reqwest` + `x402-chain-eip155/client` | Works |
| Custom `PaymentSelector` implementation | `x402-reqwest` / `x402-types` | Works (with lifetime annotation + fields) |
| FacilitatorLocal creation | `x402-facilitator-local` | Works |
| `handlers::routes()` | `x402-facilitator-local` | Works |
| Docker facilitator runtime | `ghcr.io/x402-rs/x402-facilitator:latest` | Works – tested live |
| E2E: server returns 402 + `Payment-Required` header | Real facilitator over internet | Verified |
| E2E: paying client signs and retries | Real facilitator over internet | Verified (rejected due to 0 USDC balance, which is expected) |

---

*All tests were executed in a real workspace with actual crate dependencies from crates.io, running real binaries that talk to the live public facilitator at `https://facilitator.x402.rs`.*
