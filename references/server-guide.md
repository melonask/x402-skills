# Server-Side Guide: Protecting Routes with x402 Payments

This guide covers using `x402-axum` to gate Axum routes behind blockchain micropayments. The middleware is a `tower::Layer` that intercepts requests, validates payment headers via a facilitator, and settles payments.

## Installation

```toml
[dependencies]
x402-axum = "1.0"
x402-chain-eip155 = { version = "1.0", features = ["server"] }
x402-chain-solana = { version = "1.0", features = ["server"] }
x402-types = "1.0"
alloy-primitives = "1.4"
axum = "0.8"
tokio = { version = "1", features = ["full"] }
```

Enable the `telemetry` feature for OpenTelemetry tracing:

```toml
x402-axum = { version = "1.0", features = ["telemetry"] }
```

## Basic Usage: Static Pricing

The simplest way to protect a route is with a static price tag:

```rust
use alloy_primitives::address;
use axum::{Router, routing::get, response::IntoResponse, http::StatusCode};
use x402_axum::X402Middleware;
use x402_chain_eip155::V2Eip155Exact;
use x402_types::networks::USDC;

let x402 = X402Middleware::new("https://facilitator.x402.rs");

let app = Router::new().route(
    "/paid-content",
    get(handler).layer(
        x402.with_price_tag(V2Eip155Exact::price_tag(
            address!("0xBAc675C310721717Cd4A37F6cbeA1F081b1C2a07"),
            USDC::base_sepolia().parse("0.01").unwrap(),
        ))
    ),
);

async fn handler() -> impl IntoResponse {
    (StatusCode::OK, "This is VIP content!")
}
```

When a request arrives without valid payment, the middleware returns `402 Payment Required` with the `Payment-Required` header (base64-encoded JSON describing accepted payment).

### Understanding `price_tag` Parameters

The `price_tag` method takes two arguments:

1. **`pay_to`** — The wallet address that receives the payment (as a string or `Address` type)
2. **`amount`** — The payment amount in the token's smallest unit

For USDC (6 decimals), `"0.01"` USDC = `10000` smallest units. Use the `USDC` convenience struct for known networks:

```rust
USDC::base_sepolia().amount(10000u64)  // 0.01 USDC on Base Sepolia
USDC::base().amount(1000000u64)        // 1.0 USDC on Base Mainnet
USDC::solana().amount(1000000u64)       // 1.0 USDC on Solana
```

Or use any ERC-20 token address directly:

```rust
V2Eip155Exact::price_tag(
    address!("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"), // USDC on Ethereum
    "1000000".to_string(), // 1.0 USDC (6 decimals)
)
```

## Dynamic Pricing

Use `with_dynamic_price` to compute prices per-request:

```rust
x402.with_dynamic_price(|headers, uri, _base_url| {
    let has_discount = uri.query()
        .map(|q| q.contains("discount"))
        .unwrap_or(false);
    let amount = if has_discount { 50 } else { 100 };
    async move {
        vec![V2Eip155Exact::price_tag(
            address!("0x..."),
            USDC::base_sepolia().amount(amount),
        )]
    }
})
```

The callback receives:

- `headers` — the request headers (`&HeaderMap`)
- `uri` — the request URI (`&Uri`)
- `base_url` — the configured base URL (`Option<&Url>`)

It returns `Vec<PriceTag>` — a list of accepted payment options. Return `vec![]` to **bypass payment entirely** (conditional free access).

### Conditional Free Access Example

```rust
x402.with_dynamic_price(|headers, uri, _base_url| {
    let is_free = uri.query()
        .map(|q| q.contains("free"))
        .unwrap_or(false);
    async move {
        if is_free {
            vec![]  // No payment required
        } else {
            vec![V2Eip155Exact::price_tag(
                address!("0x..."),
                USDC::base_sepolia().amount(100),
            )]
        }
    }
})
```

With this: `GET /api/data` returns 402, but `GET /api/data?free` serves content directly.

## Multi-Chain Payment Acceptance

Accept payments from multiple blockchains on the same endpoint:

```rust
use x402_chain_eip155::V2Eip155Exact;
use x402_chain_solana::V2SolanaExact;
use x402_chain_aptos::V2AptosExact;
use solana_pubkey::pubkey;

x402
    // Accept USDC on Base Sepolia
    .with_price_tag(V2Eip155Exact::price_tag(
        address!("0xBAc675C310721717Cd4A37F6cbeA1F081b1C2a07"),
        USDC::base_sepolia().parse("0.01").unwrap(),
    ))
    // Accept USDC on Solana
    .with_price_tag(V2SolanaExact::price_tag(
        pubkey!("EGBQqKn968sVv5cQh5Cr72pSTHfxsuzq7o7asqYB5uEV").to_string(),
        USDC::solana().amount(100),
    ))
```

The client picks whichever chain they prefer. The facilitator handles verification and settlement for the chosen chain.

## Multi-Token Acceptance

Accept different tokens on the same chain:

```rust
x402
    .with_price_tag(V2Eip155Exact::price_tag(
        address!("0xYourWallet"),
        "1000000".to_string(), // 1 USDC (6 decimals)
    ))
```

Each `price_tag` can specify a different token contract address and amount. The client selects which to pay.

## Settlement Timing

Control when on-chain settlement happens:

```rust
// Default: settle after handler execution
let x402 = X402Middleware::new("https://facilitator.x402.rs")
    .settle_after_execution();

// Alternative: settle before handler execution
let x402 = X402Middleware::new("https://facilitator.x402.rs")
    .settle_before_execution();
```

- **`settle_after_execution`** (default): Content is served first, then settlement happens. If settlement fails, content was already served.
- **`settle_before_execution`**: Settlement completes before the handler runs. Guarantees payment, but adds latency.

## Configuration Options

```rust
let x402 = X402Middleware::new("https://facilitator.x402.rs")
    .with_base_url(Url::parse("https://api.example.com").unwrap())
    .with_resource(Url::parse("https://api.example.com/premium").unwrap())
    .with_description("Premium API access")
    .with_mime_type("application/json")
    .with_supported_cache_ttl(Duration::from_secs(300));
```

- **`with_base_url`**: Base URL for computing resource URLs dynamically. Defaults to `http://localhost/`.
- **`with_resource`**: Explicit full URI of the protected resource (recommended in production).
- **`with_description`**: Human-readable description of what is being paid for.
- **`with_mime_type`**: MIME type of the protected resource (default: `application/json`).
- **`with_supported_cache_ttl`**: Cache TTL for facilitator capability queries. `Duration::from_secs(0)` disables caching.

## Error Handling

The middleware returns appropriate 402 responses with error details:

- `VerificationError::PaymentHeaderRequired` — Missing `Payment-Signature` header
- `VerificationError::InvalidPaymentHeader` — Malformed payment header
- `VerificationError::NoPaymentMatching` — Payment doesn't match any accepted requirements
- `VerificationError::VerificationFailed` — Facilitator rejected the payment
- `PaygateError::Settlement` — On-chain settlement failed

## Custom Schemes

Implement the `PaygateProtocol` trait to support custom payment methods:

```rust
use x402_axum::paygate::PaygateProtocol;
use x402_types::proto::v2;

pub struct MyCustomScheme;

impl MyCustomScheme {
    pub fn price_tag(
        pay_to: String,
        asset: String,
        amount: u64,
    ) -> v2::PriceTag {
        v2::PriceTag {
            requirements: v2::PaymentRequirements {
                scheme: "my-custom-scheme".to_string(),
                pay_to,
                asset,
                network: "eip155:8453".parse().unwrap(),
                amount: amount.to_string(),
                max_timeout_seconds: 300,
                extra: None,
            },
            enricher: None,
        }
    }
}
```

See `references/protocol-details.md` for the full custom scheme implementation guide.

## Telemetry

Enable the `telemetry` feature for structured tracing spans:

- `x402.handle_request`
- `x402.verify_payment`
- `x402.settle_payment`

Connect to OpenTelemetry exporters (Jaeger, Tempo, etc.) via standard `tracing` subscribers.

## HTTP Behavior

### V2 Protocol (preferred)

When no valid payment is provided:

```
HTTP/1.1 402 Payment Required
Payment-Required: <base64-encoded PaymentRequired>
```

The `Payment-Required` header contains base64-encoded JSON:

```json
{
  "x402Version": "2",
  "resource": { "url": "...", "description": "...", "mimeType": "..." },
  "paymentRequirements": [
    {
      "scheme": "exact",
      "network": "eip155:84532",
      "amount": "10000",
      "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
      "payTo": "0x...",
      "maxTimeoutSeconds": 60,
      "extra": {
        "assetTransferMethod": "eip3009",
        "name": "USDC",
        "version": "2"
      }
    }
  ]
}
```

### V1 Protocol

```
HTTP/1.1 402 Payment Required
Content-Type: application/json

{
  "error": "X-PAYMENT header is required",
  "accepts": [...],
  "x402Version": "1"
}
```
