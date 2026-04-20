# Facilitator Guide: Verification, Settlement, and Custom Facilitators

The facilitator is the middleware between HTTP and the blockchain. It verifies that a client's payment signature is valid, checks balances, and settles the payment on-chain. This guide covers running a production facilitator and building custom ones.

## What a Facilitator Does

1. **Verify** (`POST /verify`) — Validates the payment signature, checks balances, confirms parameters match requirements
2. **Settle** (`POST /settle`) — Broadcasts the transaction on-chain and waits for confirmation
3. **Supported** (`GET /supported`) — Lists supported (scheme, chain) pairs

The facilitator **cannot modify** the payment amount or destination — these are locked by the client's signature.

## HTTP Endpoints

| Endpoint     | Method | Description                                 |
| ------------ | ------ | ------------------------------------------- |
| `/`          | GET    | Server greeting                             |
| `/verify`    | GET    | Schema information for verify endpoint      |
| `/verify`    | POST   | Verify a payment payload                    |
| `/settle`    | GET    | Schema information for settle endpoint      |
| `/settle`    | POST   | Settle a verified payment on-chain          |
| `/supported` | GET    | List supported payment schemes and networks |
| `/health`    | GET    | Health check (delegates to `/supported`)    |

## Running a Production Facilitator

### Option 1: Docker (Recommended)

Prebuilt images are at GitHub Container Registry:

```bash
docker run -v $(pwd)/config.json:/app/config.json -p 8080:8080 \
  ghcr.io/x402-rs/x402-facilitator
```

### Option 2: Build from Source

```bash
# Clone the repo
git clone https://github.com/x402-rs/x402-rs.git
cd x402-rs

# Run with all chains + telemetry
cargo run --package x402-facilitator --features full

# Run with specific chains only
cargo run --package x402-facilitator --features chain-eip155,chain-solana

# With custom config
cargo run --package x402-facilitator -- --config /path/to/config.json
```

### Feature Flags

| Feature        | Description                       |
| -------------- | --------------------------------- |
| `telemetry`    | OpenTelemetry tracing and metrics |
| `chain-eip155` | EVM/EIP-155 chain support         |
| `chain-solana` | Solana chain support              |
| `chain-aptos`  | Aptos support (requires patches)  |
| `full`         | All features enabled              |

## Facilitator Configuration

### Complete `config.json` Example

```json
{
  "port": 8080,
  "host": "0.0.0.0",
  "chains": {
    "eip155:84532": {
      "_comment": "Base Sepolia",
      "eip1559": true,
      "flashblocks": true,
      "receipt_timeout_secs": 30,
      "signers": ["0xFACILITATOR_PRIVATE_KEY"],
      "rpc": [
        {
          "http": "https://sepolia.base.org",
          "rate_limit": 50
        },
        {
          "http": "https://base-sepolia.g.alchemy.com/v2/YOUR_KEY",
          "rate_limit": 100
        }
      ]
    },
    "eip155:8453": {
      "_comment": "Base Mainnet",
      "eip1559": true,
      "flashblocks": false,
      "signers": ["$FACILITATOR_PRIVATE_KEY"],
      "rpc": [
        {
          "http": "https://mainnet.base.org",
          "rate_limit": 100
        }
      ]
    },
    "solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp": {
      "_comment": "Solana Mainnet",
      "signers": ["$SOLANA_PRIVATE_KEY"],
      "rpc": [
        {
          "http": "https://api.mainnet-beta.solana.com"
        }
      ]
    }
  },
  "schemes": [
    {
      "id": "v1-eip155-exact",
      "chains": "eip155:*"
    },
    {
      "id": "v2-eip155-exact",
      "chains": "eip155:*"
    },
    {
      "id": "v1-solana-exact",
      "chains": "solana:*"
    },
    {
      "id": "v2-solana-exact",
      "chains": "solana:*"
    }
  ]
}
```

### Config Parameters

**Top-level:**

- `port` — Server port (default: `8080`, env: `PORT`)
- `host` — Bind address (default: `0.0.0.0`, env: `HOST`)
- `config` — Path to config file (env: `CONFIG`, default: `config.json`)

**Per-chain (EVM `eip155:*`):**

- `eip1559` — Use EIP-1559 gas model (default: `true`)
- `flashblocks` — Enable flashblocks for faster confirmation (default: `false`)
- `receipt_timeout_secs` — Max seconds to wait for tx receipt (default: `30`)
- `signers` — Array of hex-encoded EVM private keys (first one is primary)
- `rpc` — Array of RPC endpoints with `http` URL and optional `rate_limit`

**Per-chain (Solana `solana:*`):**

- `signers` — Array of base58-encoded Solana private keys
- `rpc` — Array of RPC endpoints
- `pubsub` — Optional WebSocket endpoint for faster confirmations

### Environment Variable Resolution

Config values support `$VAR_NAME` and `${VAR_NAME}` syntax:

```json
{
  "http": "$RPC_URL",
  "signers": ["$FACILITATOR_PRIVATE_KEY"]
}
```

Values are resolved at load time. The `LiteralOrEnv<T>` type stores the original env var name and only resolves it when needed, preventing sensitive values from appearing in logs.

### Multiple RPC Endpoints with Failover

The facilitator supports multiple RPC endpoints per chain with rate limiting:

```json
{
  "eip155:8453": {
    "rpc": [
      { "http": "https://mainnet.base.org", "rate_limit": 100 },
      { "http": "$BACKUP_RPC_URL", "rate_limit": 50 }
    ]
  }
}
```

Multiple signers enable round-robin load distribution for transaction signing.

## Building a Custom Facilitator

For advanced use cases, build your own facilitator using `x402-facilitator-local`:

```toml
[dependencies]
x402-facilitator-local = { version = "1.0", features = ["telemetry"] }
x402-chain-eip155 = { version = "1.0", features = ["facilitator"] }
x402-chain-solana = { version = "1.0", features = ["facilitator"] }
x402-types = { version = "1.0", features = ["cli"] }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
```

### Complete Custom Facilitator Example

```rust
use x402_facilitator_local::{FacilitatorLocal, handlers};
use x402_types::chain::ChainRegistry;
use x402_types::scheme::{SchemeBlueprints, SchemeRegistry};
use x402_chain_eip155::{V1Eip155Exact, V2Eip155Exact};
use x402_chain_solana::{V1SolanaExact, V2SolanaExact};
use std::sync::Arc;
use axum::Router;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Load chain config (from file or env)
    let chains_config = /* ... */;

    // Initialize chain registry with blockchain providers
    let chain_registry = ChainRegistry::from_config(&chains_config).await?;

    // Register supported payment schemes
    let scheme_blueprints = SchemeBlueprints::new()
        .and_register(V1Eip155Exact)
        .and_register(V2Eip155Exact)
        .and_register(V1SolanaExact)
        .and_register(V2SolanaExact);

    // Build the scheme registry from config
    let schemes_config = /* ... */;
    let scheme_registry = SchemeRegistry::build(
        chain_registry,
        scheme_blueprints,
        &schemes_config,
    );

    // Create the facilitator
    let facilitator = FacilitatorLocal::new(scheme_registry);
    let state = Arc::new(facilitator);

    // Create HTTP routes
    let app = Router::new()
        .merge(handlers::routes().with_state(state));

    // Run the server
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

### Facilitator Architecture

```
FacilitatorLocal
       |
  SchemeRegistry  (routes requests to the right handler)
       |
  ┌────┴────┐
  V1Eip155   V2Eip155   V1Solana   V2Solana
  Exact      Exact      Exact      Exact
  (handler)  (handler)  (handler)  (handler)
       |         |         |         |
  Eip155   Eip155    Solana    Solana
  Provider Provider  Provider  Provider
```

1. **ChainRegistry** manages blockchain providers and connections
2. **SchemeBlueprints** defines available payment schemes
3. **SchemeRegistry** combines chains and schemes into executable handlers
4. **FacilitatorLocal** routes `/verify` and `/settle` to the correct handler

### Using the Facilitator Trait Directly

For programmatic use (not HTTP):

```rust
use x402_chain_eip155::{V1Eip155Exact, Eip155ChainProvider};
use x402_types::scheme::X402SchemeFacilitatorBuilder;
use x402_types::proto::v2::{VerifyRequest, SettleRequest};

// Create chain provider from config
let provider = Eip155ChainProvider::from_config(&config).await?;

// Build facilitator for a specific scheme
let facilitator = V1Eip155Exact.build(provider, None)?;

// Verify a payment
let verify_response = facilitator.verify(&verify_request).await?;

// Settle a verified payment
let settle_response = facilitator.settle(&settle_request).await?;
```

## Graceful Shutdown

```rust
use x402_facilitator_local::util::SigDown;

let sig_down = SigDown::try_new()?;
let cancellation_token = sig_down.cancellation_token();

axum::serve(listener, app)
    .with_graceful_shutdown(async move {
        cancellation_token.cancelled().await;
    })
    .await?;
```

Handles `SIGTERM` and `SIGINT` for clean shutdown.

## OpenTelemetry

```rust
use x402_facilitator_local::util::Telemetry;

let telemetry = Telemetry::new()
    .with_name("x402-facilitator")
    .with_version("1.0.0")
    .register();

let tracing_layer = telemetry.http_tracing();

let app = Router::new()
    .merge(handlers::routes().with_state(state))
    .layer(tracing_layer);
```

Set environment variables:

- `OTEL_EXPORTER_OTLP_ENDPOINT` — Collector endpoint
- `OTEL_SERVICE_NAME` — Service name for traces
- `OTEL_SERVICE_VERSION` — Service version

## Aptos Support

Aptos requires git-only dependencies and patches:

```toml
[dependencies]
x402-chain-aptos = { version = "1.0", features = ["facilitator"] }

[patch.crates-io]
merlin = { git = "https://github.com/aptos-labs/merlin" }

[patch."https://github.com/aptos-labs/aptos-core"]
aptos-runtimes = { path = "patches/aptos-runtimes" }
```

Aptos facilitator config:

```json
{
  "aptos:1": {
    "sponsor_gas": true,
    "signer": "$APTOS_FACILITATOR_KEY",
    "rpc": "https://fullnode.mainnet.aptoslabs.com/v1",
    "api_key": "$APTOS_API_KEY"
  }
}
```
