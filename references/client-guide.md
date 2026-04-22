# Client-Side Guide: Building Agents That Pay for Data

This guide covers using `x402-reqwest` to build Rust clients that automatically handle x402 payments. The middleware detects `402 Payment Required` responses, signs payments, and retries — all transparently.

## Installation

```toml
[dependencies]
x402-reqwest = { version = "1.0", features = ["json"] }
x402-chain-eip155 = { version = "1.0", features = ["client"] }
x402-chain-solana = { version = "1.0", features = ["client"] }
alloy-signer-local = "1.4"
reqwest = { version = "0.13", features = ["json"] }
tokio = { version = "1", features = ["full"] }
```

## Basic Usage: Single-Chain Client

```rust
use x402_reqwest::{ReqwestWithPayments, ReqwestWithPaymentsBuild, X402Client};
use x402_chain_eip155::V2Eip155ExactClient;
use alloy_signer_local::PrivateKeySigner;
use std::sync::Arc;
use reqwest::Client;

let signer: Arc<PrivateKeySigner> = Arc::new("0xYOUR_PRIVATE_KEY".parse()?);

let x402_client = X402Client::new()
    .register(V2Eip155ExactClient::new(signer));

let client = Client::new()
    .with_payments(x402_client)
    .build();

// Transparent payment handling
let response = client
    .get("https://api.example.com/protected")
    .send()
    .await?;
```

When the server returns `402 Payment Required`, the middleware:

1. Parses the payment requirements from the response
2. Finds a registered scheme client that can handle the payment
3. Signs the payment payload using the scheme client
4. Retries the request with the `Payment-Signature` header

## Scheme Client Registration

Register scheme clients for each blockchain protocol version you want to support:

### EVM Clients

```rust
use x402_chain_eip155::{V1Eip155ExactClient, V2Eip155ExactClient};

let x402_client = X402Client::new()
    .register(V1Eip155ExactClient::new(evm_signer.clone()))
    .register(V2Eip155ExactClient::new(evm_signer));
```

- `V1Eip155ExactClient` — Handles V1 protocol payments on EVM chains
- `V2Eip155ExactClient` — Handles V2 protocol payments on EVM chains

Both require an `Arc<PrivateKeySigner>` (from `alloy_signer_local`).

### Solana Clients

```rust
use x402_chain_solana::{V1SolanaExactClient, V2SolanaExactClient};
use solana_keypair::Keypair;
use solana_client::nonblocking::rpc_client::RpcClient;

let keypair = Arc::new(Keypair::from_base58_string("YOUR_SOLANA_PRIVATE_KEY"));
let rpc_client = Arc::new(RpcClient::new("https://api.mainnet-beta.solana.com"));

let x402_client = X402Client::new()
    .register(V1SolanaExactClient::new(keypair.clone(), rpc_client.clone()))
    .register(V2SolanaExactClient::new(keypair, rpc_client));
```

Both require an `Arc<Keypair>` and an `Arc<RpcClient>`.

## Multi-Chain Client (EVM + Solana)

Build an agent that can pay on any supported chain:

```rust
use x402_reqwest::{X402Client, ReqwestWithPayments, ReqwestWithPaymentsBuild};
use x402_chain_eip155::{V1Eip155ExactClient, V2Eip155ExactClient};
use x402_chain_solana::{V1SolanaExactClient, V2SolanaExactClient};
use alloy_signer_local::PrivateKeySigner;
use solana_keypair::Keypair;
use solana_client::nonblocking::rpc_client::RpcClient;
use std::sync::Arc;
use reqwest::Client;

// EVM setup
let evm_signer: Arc<PrivateKeySigner> = Arc::new("0x...".parse()?);

// Solana setup
let sol_kp = Arc::new(Keypair::from_base58_string("..."));
let sol_rpc = Arc::new(RpcClient::new("https://api.mainnet-beta.solana.com"));

// Register all scheme clients
let x402_client = X402Client::new()
    .register(V1Eip155ExactClient::new(evm_signer.clone()))
    .register(V2Eip155ExactClient::new(evm_signer))
    .register(V1SolanaExactClient::new(sol_kp.clone(), sol_rpc.clone()))
    .register(V2SolanaExactClient::new(sol_kp, sol_rpc));

// Build the client
let client = Client::new()
    .with_payments(x402_client)
    .build();

// Use normally — x402 is handled transparently
let data: serde_json::Value = client
    .get("https://premium-api.example.com/data")
    .send()
    .await?
    .json()
    .await?;
```

## Payment Selection

When the server offers multiple payment options (e.g., different chains or tokens), `X402Client` uses a `PaymentSelector` to choose which to pay.

### Default: FirstMatch

The default selector picks the first matching registered scheme:

```rust
let x402_client = X402Client::new()
    .register(V2Eip155ExactClient::new(evm_signer))
    .register(V2SolanaExactClient::new(sol_kp, sol_rpc));
// Will try EVM first, then Solana
```

### Custom Payment Selector

Implement the `PaymentSelector` trait for custom logic:

```rust
use x402_reqwest::X402Client;
use x402_types::scheme::client::{PaymentSelector, PaymentCandidate};

struct CheapestSelector;

impl PaymentSelector for CheapestSelector {
    fn select(&self, candidates: &[PaymentCandidate]) -> Option<&PaymentCandidate> {
        // Pick the payment option with the lowest amount
        candidates.iter().min_by_key(|c| c.amount())
    }
}

struct ChainPreferenceSelector {
    preferred_chain: String,
}

impl PaymentSelector for ChainPreferenceSelector {
    fn select(&self, candidates: &[PaymentCandidate]) -> Option<&PaymentCandidate> {
        // Prefer a specific chain
        candidates.iter().find(|c| c.network().contains(&self.preferred_chain))
            .or_else(|| candidates.first())
    }
}

let x402_client = X402Client::new()
    .register(V2Eip155ExactClient::new(evm_signer))
    .with_selector(CheapestSelector);
```

## How It Works: The Payment Flow

1. The client makes a normal HTTP request to a protected endpoint
2. If the response is `402 Payment Required`:
   - The middleware parses the `Payment-Required` header (V2) or response body (V1)
   - It extracts the list of `PaymentRequirements`
   - It iterates through registered scheme clients to find ones that can handle the payment
   - The `PaymentSelector` picks the best option
   - The selected scheme client signs the payment payload
3. The middleware retries the original request with the `Payment-Signature` header
4. If the server verifies the payment, it returns `200 OK` with the content

### V2 Payment Header

```
Payment-Signature: <base64-encoded PaymentPayload>
```

### V1 Payment Header

```
X-PAYMENT: <base64-encoded PaymentPayload>
```

The middleware automatically detects which protocol version the server uses and responds accordingly.

## Building an AI Agent That Pays for Data

Here's a practical example of an agent that fetches paid data:

```rust
use x402_reqwest::{X402Client, ReqwestWithPayments, ReqwestWithPaymentsBuild};
use x402_chain_eip155::V2Eip155ExactClient;
use alloy_signer_local::PrivateKeySigner;
use std::sync::Arc;
use reqwest::Client;

async fn fetch_paid_market_data() -> Result<String, Box<dyn std::error::Error>> {
    let signer: Arc<PrivateKeySigner> = Arc::new("0x...".parse()?);

    let client = Client::new()
        .with_payments(
            X402Client::new()
                .register(V2Eip155ExactClient::new(signer))
        )
        .build();

    // This endpoint requires x402 payment — handled transparently
    let response = client
        .get("https://premium-api.example.com/market-data?symbol=BTC")
        .send()
        .await?;

    let data: serde_json::Value = response.json().await?;
    Ok(serde_json::to_string_pretty(&data)?)
}
```

## Telemetry

Enable the `telemetry` feature for structured tracing:

```toml
x402-reqwest = { version = "1.0", features = ["telemetry", "json"] }
```

Tracing spans emitted:

- `x402.reqwest.handle` — Entire middleware handling (402 detection + retry)
- `x402.reqwest.next` — Underlying HTTP request (initial and retry)
- `x402.reqwest.make_payment_headers` — Payment signing
- `x402.reqwest.parse_payment_required` — 402 response parsing

## Important Notes

- The client **never pays gas**. All on-chain operations are handled by the facilitator.
- The client's private key is only used for signing payment authorizations off-chain.
- EVM payments use EIP-712 typed data signing (for EIP-3009) or Permit2 signatures.
- Solana payments sign a serialized `VersionedTransaction`.
- Payments are one-time use — each nonce can only be used once, preventing replay attacks.
