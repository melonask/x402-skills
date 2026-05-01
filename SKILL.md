---
name: x402
description: "Comprehensive guide for building solutions with the x402-rs Rust library and the x402 HTTP 402 payment protocol. Use this skill whenever the user wants to: create API services that accept x402 payments and return results; build agents that automatically pay via the x402 protocol to access paid data; implement payment verification on their own facilitator; configure x402 facilitators for EVM/Solana/Aptos blockchains; use any ERC-20 token or native coin as payment (not just USDC); build custom payment schemes (like EIP-7702 delegation); implement gasless payments via EIP-3009, Permit2, or EIP-2612; protect Axum routes behind blockchain micropayments; create reqwest clients that transparently handle x402 payments; or work with the x402 protocol V1 or V2 wire formats. Make sure to use this skill when the user mentions HTTP 402, x402, blockchain payments, crypto payments, micropayments over HTTP, pay-per-request APIs, or facilitator setup."
---

# x402 Protocol & x402-rs Development Guide

The x402 protocol activates the long-dormant HTTP `402 Payment Required` status code to enable blockchain payments directly through HTTP. The client never touches the blockchain directly — it produces a cryptographic signature, and a **facilitator** verifies and settles the payment on-chain.

## Three Actors

```
  Client                    Resource Server              Facilitator
    |                             |                          |
    | 1. GET /paid-content        |                          |
    |---------------------------->|                          |
    | 2. 402 Payment Required     |                          |
    |    Payment-Required: <b64>  |                          |
    |<----------------------------|                          |
    | 3. Sign payment (off-chain) |                          |
    | 4. GET /paid-content        |                          |
    |    Payment-Signature: <b64> |                          |
    |---------------------------->|                          |
    |                             | 5. POST /verify          |
    |                             |------------------------->|
    |                             | 6. Valid                 |
    |                             |<-------------------------|
    |                             | 7. Execute handler       |
    |                             | 8. POST /settle          |
    |                             |------------------------->|
    |                             | 9. On-chain tx           |
    |                             |<-------------------------|
    | 11. 200 OK + content        |                          |
    |<----------------------------|                          |
```

## Crate Architecture

| Crate                    | Purpose                                                                                |
| ------------------------ | -------------------------------------------------------------------------------------- |
| `x402-types`             | Core protocol types, facilitator traits, CAIP-2 chain IDs, config, wire format (V1/V2) |
| `x402-axum`              | Axum middleware — protect routes with x402 payments (server-side)                      |
| `x402-reqwest`           | Reqwest middleware — transparent x402 payment handling (client-side)                   |
| `x402-facilitator-local` | Local facilitator — verify and settle payments                                         |
| `x402-chain-eip155`      | EVM chain support (Ethereum, Base, Polygon, Avalanche, Sei, Celo, etc.)                |
| `x402-chain-solana`      | Solana chain support                                                                   |
| `x402-chain-aptos`       | Aptos chain support (git-only dependency, not on crates.io)                            |
| `x402-facilitator`       | Production facilitator binary (not on crates.io)                                       |

## Quick Start: Which Guide Do I Need?

The user's task determines which guide to read next:

- **Creating a paid API/protecting routes** -> Read `references/server-guide.md`
- **Building an agent that pays for data** -> Read `references/client-guide.md`
- **Running a facilitator / verifying payments** -> Read `references/facilitator-guide.md`
- **Chain-specific config, tokens, networks** -> Read `references/chain-config.md`
- **V1 vs V2 differences, schemes, gasless internals** -> Read `references/protocol-details.md`
- **Building custom payment schemes (e.g., EIP-7702)** -> Read `references/custom-scheme-guide.md`

## Minimal Working Examples

### Server: Protect a Route (Axum)

```toml
# Cargo.toml
[dependencies]
x402-axum = "1.0"
x402-chain-eip155 = { version = "1.0", features = ["server"] }
x402-types = "1.0"
alloy-primitives = "1.4"
axum = "0.8"
tokio = { version = "1", features = ["full"] }
```

```rust
use alloy_primitives::address;
use axum::{Router, routing::get, response::IntoResponse, http::StatusCode};
use x402_axum::X402Middleware;
use x402_chain_eip155::{V2Eip155Exact, KnownNetworkEip155};
use x402_types::networks::USDC;

let x402 = X402Middleware::new("https://facilitator.x402.rs");

let app = Router::new().route(
    "/paid-content",
    get(handler).layer(
        x402.with_price_tag(V2Eip155Exact::price_tag(
            address!("0xYourWalletAddress"),
            USDC::base_sepolia().amount(10u64), // 10 USDC (6 decimals)
        ))
    ),
);

async fn handler() -> impl IntoResponse {
    (StatusCode::OK, "Paid content here!")
}
```

### Client: Auto-Pay for Data (Reqwest)

```toml
# Cargo.toml
[dependencies]
x402-reqwest = { version = "1.0", features = ["json"] }
x402-chain-eip155 = { version = "1.0", features = ["client"] }
alloy-signer-local = "1.4"
reqwest = { version = "0.13", features = ["json"] }
tokio = { version = "1", features = ["full"] }
```

```rust
use x402_reqwest::{ReqwestWithPayments, ReqwestWithPaymentsBuild, X402Client};
use x402_chain_eip155::V2Eip155ExactClient;
use alloy_signer_local::PrivateKeySigner;
use std::sync::Arc;
use reqwest::Client;

let signer: Arc<PrivateKeySigner> = Arc::new("0x...private_key...".parse()?);

let x402_client = X402Client::new()
    .register(V2Eip155ExactClient::new(signer));

let client = Client::new()
    .with_payments(x402_client)
    .build();

// Payments are handled transparently
let res = client.get("https://api.example.com/protected").send().await?;
```

### Multi-Chain Client (EVM + Solana)

```rust
use x402_reqwest::{X402Client, ReqwestWithPayments, ReqwestWithPaymentsBuild};
use x402_chain_eip155::{V1Eip155ExactClient, V2Eip155ExactClient};
use x402_chain_solana::{V1SolanaExactClient, V2SolanaExactClient};
use alloy_signer_local::PrivateKeySigner;
use solana_keypair::Keypair;
use solana_client::nonblocking::rpc_client::RpcClient;
use std::sync::Arc;

let evm_signer: Arc<PrivateKeySigner> = Arc::new("0x...".parse()?);
let sol_kp = Arc::new(Keypair::from_base58_string("..."));
let sol_rpc = Arc::new(RpcClient::new("https://api.mainnet-beta.solana.com"));

let x402_client = X402Client::new()
    .register(V1Eip155ExactClient::new(evm_signer.clone()))
    .register(V2Eip155ExactClient::new(evm_signer))
    .register(V1SolanaExactClient::new(sol_kp.clone(), sol_rpc.clone()))
    .register(V2SolanaExactClient::new(sol_kp, sol_rpc));

let client = Client::new().with_payments(x402_client).build();
```

## Key Concepts at a Glance

### Any Token, Any Chain

x402 supports **any ERC-20 token** (not just USDC). The `price_tag` method accepts any token contract address:

```rust
use x402_chain_eip155::KnownNetworkEip155;
use x402_types::networks::USDC;

// USDC (convenience helper)
USDC::base_sepolia().amount(10u64)

// Any ERC-20 token directly
V2Eip155Exact::price_tag(
    address!("0xTokenContractAddress"),
    "1000000".to_string(), // amount in smallest token unit
)
```

### Gasless Payments

The client **never pays gas**. Three mechanisms exist for EVM chains:

1. **EIP-3009** (`transferWithAuthorization`) — preferred when token supports it (e.g., USDC). Client signs one message, facilitator calls the transfer function.
2. **Permit2** (Uniswap universal proxy) — fallback for any ERC-20. Requires one-time `approve(Permit2)` setup by the user (gasless options exist).
3. **EIP-2612 gas sponsoring** — facilitator can sponsor the Permit2 approval itself if the token supports `permit()`.

### V1 vs V2 Protocol

| Aspect              | V1                               | V2                                  |
| ------------------- | -------------------------------- | ----------------------------------- |
| Network IDs         | Network names (`"base-sepolia"`) | CAIP-2 chain IDs (`"eip155:84532"`) |
| Payment header      | `X-PAYMENT`                      | `Payment-Signature`                 |
| Requirements header | JSON body                        | `Payment-Required` header (base64)  |
| Token support       | USDC-focused                     | Any ERC-20/SPL/Aptos token          |

Both are supported simultaneously by x402-rs. Prefer V2 for new projects.

### Price Tag Types

| Type                         | Chain  | Protocol | Scheme |
| ---------------------------- | ------ | -------- | ------ |
| `V1Eip155Exact::price_tag()` | EVM    | V1       | exact  |
| `V2Eip155Exact::price_tag()` | EVM    | V2       | exact  |
| `V1SolanaExact::price_tag()` | Solana | V1       | exact  |
| `V2SolanaExact::price_tag()` | Solana | V2       | exact  |
| `V2AptosExact::price_tag()`  | Aptos  | V2       | exact  |
| `V2Eip155Upto::price_tag()`  | EVM    | V2       | upto   |

### Custom Schemes & Extensibility

If built-in schemes like `exact` or `upto` don't fit, `x402-rs` allows you to implement custom schemes via the `X402SchemeId` and `X402SchemeFacilitator` traits. This is highly useful for implementing bleeding-edge blockchain patterns—like **EIP-7702 delegation**—where the scheme dictates unique signature payloads and Type 4 transaction settlements, while keeping the HTTP middleware layers unchanged.

## Installation Patterns

### Server (accept payments)

```toml
[dependencies]
x402-axum = "1.0"
x402-chain-eip155 = { version = "1.0", features = ["server"] }
x402-chain-solana = { version = "1.0", features = ["server"] }
x402-types = "1.0"
```

### Client (make payments)

```toml
[dependencies]
x402-reqwest = { version = "1.0", features = ["json"] }
x402-chain-eip155 = { version = "1.0", features = ["client"] }
x402-chain-solana = { version = "1.0", features = ["client"] }
```

### Facilitator (verify/settle)

```toml
[dependencies]
x402-facilitator-local = "1.0"
x402-chain-eip155 = { version = "1.0", features = ["facilitator"] }
x402-chain-solana = { version = "1.0", features = ["facilitator"] }
```

### Full custom facilitator server

```toml
[dependencies]
x402-facilitator-local = { version = "1.0", features = ["telemetry"] }
x402-chain-eip155 = { version = "1.0", features = ["facilitator"] }
x402-chain-solana = { version = "1.0", features = ["facilitator"] }
x402-types = { version = "1.0", features = ["cli"] }
```

## Common Patterns

### Dynamic Pricing

Compute price per-request based on headers, query params, or auth:

```rust
x402.with_dynamic_price(|headers, uri, _base_url| {
    let has_discount = uri.query().map(|q| q.contains("discount")).unwrap_or(false);
    let amount = if has_discount { 50u64 } else { 100u64 };
    async move {
        vec![V2Eip155Exact::price_tag(
            address!("0x..."),
            USDC::base_sepolia().amount(amount),
        )]
    }
})
```

Return `vec![]` from dynamic pricing to bypass payment entirely (conditional free access).

### Multi-Chain Payment Acceptance (Server)

Accept payments on multiple chains simultaneously:

```rust
x402
    .with_price_tag(V2Eip155Exact::price_tag(
        address!("0x..."),
        USDC::base_sepolia().amount(10u64),
    ))
    .with_price_tag(V2SolanaExact::price_tag(
        "EGBQqKn968sVv5cQh5Cr72pSTHfxsuzq7o7asqYB5uEV".to_string(),
        USDC::solana().amount(10u64),
    ))
```

### Settlement Timing

```rust
// Default: settle after handler (content served even if settlement fails mid-flight)
x402.settle_after_execution();

// Settle before handler (guarantees payment, but slight latency)
x402.settle_before_execution();
```

### Custom Facilitator Endpoints

```rust
x402.with_base_url(Url::parse("https://api.example.com").unwrap())
    .with_resource(Url::parse("https://api.example.com/premium").unwrap())
    .with_description("Premium API access")
    .with_mime_type("application/json")
    .with_supported_cache_ttl(Duration::from_secs(300))
```

## Running a Facilitator

### Docker (simplest)

```bash
docker run -v $(pwd)/config.json:/app/config.json -p 8080:8080 \
  ghcr.io/x402-rs/x402-facilitator
```

### From Source

```bash
# All chains
cargo run --package x402-facilitator --features full

# Specific chains only
cargo run --package x402-facilitator --features chain-eip155,chain-solana
```

See `references/facilitator-guide.md` for full facilitator configuration.

## Reference Files

For detailed information on any topic, read the appropriate reference file:

- `references/server-guide.md` — Complete server-side guide: static/dynamic pricing, multi-chain, custom schemes, error handling, the `PaygateProtocol` trait
- `references/client-guide.md` — Complete client-side guide: scheme registration, payment selection, multi-chain clients, custom selectors, V1/V2 handling
- `references/facilitator-guide.md` — Facilitator setup, configuration, custom facilitator implementation, scheme registry, OpenTelemetry, graceful shutdown
- `references/chain-config.md` — Per-chain config (EVM, Solana, Aptos), all known networks, USDC addresses, RPC setup, feature flags, environment variables, `LiteralOrEnv`
- `references/protocol-details.md` — V1 vs V2 wire format, EIP-3009 flow, Permit2 flow, EIP-2612 gas sponsoring, smart wallet support (EIP-1271/EIP-6492), `upto` scheme, custom scheme implementation
- `references/custom-scheme-guide.md` — Creating custom schemes, extending the protocol, and full-stack EIP-7702 implementation.
