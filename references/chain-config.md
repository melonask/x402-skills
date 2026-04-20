# Chain Configuration: Networks, Tokens, and RPC Setup

This reference covers all supported blockchain networks, token addresses, RPC configuration, feature flags, and environment variables for each chain.

## Supported Networks

### EVM Chains (EIP-155)

| Network           | Chain ID | CAIP-2 ID      | Name in x402-rs          | USDC Address |
| ----------------- | -------- | -------------- | ------------------------ | ------------ |
| Base              | 8453     | `eip155:8453`  | `USDC::base()`           | Built-in     |
| Base Sepolia      | 84532    | `eip155:84532` | `USDC::base_sepolia()`   | Built-in     |
| Polygon           | 137      | `eip155:137`   | `USDC::polygon()`        | Built-in     |
| Polygon Amoy      | 80002    | `eip155:80002` | `USDC::polygon_amoy()`   | Built-in     |
| Avalanche C-Chain | 43114    | `eip155:43114` | `USDC::avalanche()`      | Built-in     |
| Avalanche Fuji    | 43113    | `eip155:43113` | `USDC::avalanche_fuji()` | Built-in     |
| Sei               | 1329     | `eip155:1329`  | `USDC::sei()`            | Built-in     |
| Sei Testnet       | —        | `eip155:*`     | `USDC::sei_testnet()`    | Built-in     |
| XDC Network       | —        | `eip155:*`     | `USDC::xdc()`            | Built-in     |
| XRPL EVM          | —        | `eip155:*`     | `USDC::xrpl_evm()`       | Built-in     |
| Peaq              | —        | `eip155:*`     | `USDC::peaq()`           | Built-in     |
| IoTeX             | —        | `eip155:*`     | `USDC::iotex()`          | Built-in     |
| Celo              | —        | `eip155:*`     | `USDC::celo()`           | Built-in     |
| Celo Sepolia      | —        | `eip155:*`     | `USDC::celo_sepolia()`   | Built-in     |

### Solana

| Network        | CAIP-2 ID                                             | Name in x402-rs         |
| -------------- | ----------------------------------------------------- | ----------------------- |
| Solana Mainnet | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`             | `USDC::solana()`        |
| Solana Devnet  | `solana:EtWTRABZaYq6iM94YHpEqi5Swv8aLxwG3TpvNsw7qjjp` | `USDC::solana_devnet()` |

### Aptos

| Network       | CAIP-2 ID | Name in x402-rs         |
| ------------- | --------- | ----------------------- |
| Aptos Mainnet | `aptos:1` | `USDC::aptos()`         |
| Aptos Testnet | `aptos:2` | `USDC::aptos_testnet()` |

## Using the USDC Convenience Struct

The `USDC` struct (from `x402_types::networks`) provides pre-configured token deployments:

```rust
use x402_types::networks::USDC;
use x402_chain_eip155::V2Eip155Exact;

// Get USDC deployment info for a specific network
let usdc_base = USDC::base();           // Returns Eip155TokenDeployment for Base Mainnet
let usdc_sepolia = USDC::base_sepolia(); // Returns Eip155TokenDeployment for Base Sepolia

// Get the amount in smallest units
let amount = usdc_base.amount(1_000_000u64); // 1.0 USDC (6 decimals)

// Parse from decimal string
let amount = usdc_sepolia.parse("0.01").unwrap(); // 10000 smallest units

// Use in a price tag
let price_tag = V2Eip155Exact::price_tag(
    address!("0xYourWallet"),
    usdc_base.amount(100),
);
```

### Solana USDC

```rust
use x402_types::networks::USDC;
use x402_chain_solana::V2SolanaExact;

let usdc_sol = USDC::solana();
let price_tag = V2SolanaExact::price_tag(
    "9WzDXwBbmkg8ZTbNMqUxvQRAyrZzDsGYdLVL9zYtAWWM".to_string(), // pay-to address
    usdc_sol.amount(1_000_000), // 1.0 USDC
);
```

## Using Any Token (Not Just USDC)

The protocol supports **any ERC-20 token**. Pass the token contract address directly:

```rust
use x402_chain_eip155::V2Eip155Exact;
use alloy_primitives::address;

// Accept DAI payments on Base Sepolia
let price_tag = V2Eip155Exact::price_tag(
    address!("0xDA898..."), // Your wallet address (pay_to)
    "1000000000000000000".to_string(), // 1 DAI (18 decimals)
);
```

For this to work, the server must include the token contract address in the payment requirements. On the client side, the facilitator will handle the specific transfer mechanism based on what the token supports (EIP-3009 or Permit2).

## Chain-Specific Config for Facilitator

### EVM Chain Config

```json
{
  "eip155:8453": {
    "eip1559": true,
    "flashblocks": false,
    "receipt_timeout_secs": 30,
    "signers": ["$FACILITATOR_KEY"],
    "rpc": [
      {
        "http": "https://mainnet.base.org",
        "rate_limit": 100
      },
      {
        "http": "$BACKUP_RPC",
        "rate_limit": 50
      }
    ]
  }
}
```

Parameters:

- `eip1559` — Use EIP-1559 gas model (default: `true`)
- `flashblocks` — Enable flashblocks for faster confirmation (default: `false`)
- `receipt_timeout_secs` — Max wait for transaction receipt (default: `30`)
- `signers` — Array of hex-encoded private keys. Multiple signers enable round-robin load distribution.
- `rpc` — Array of RPC configs with `http` URL and optional `rate_limit` (requests per second)

### Solana Chain Config

```json
{
  "solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp": {
    "signers": ["$SOLANA_PRIVATE_KEY"],
    "rpc": [{ "http": "https://api.mainnet-beta.solana.com" }],
    "pubsub": "wss://api.mainnet-beta.solana.com"
  }
}
```

Parameters:

- `signers` — Array of base58-encoded 64-byte Solana private keys
- `rpc` — Array of HTTP RPC endpoints
- `pubsub` — Optional WebSocket endpoint for faster transaction confirmations via signature subscriptions

### Aptos Chain Config

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

Parameters:

- `sponsor_gas` — Whether facilitator pays gas (default: `false`)
- `signer` — Hex-encoded Ed25519 private key
- `rpc` — Aptos REST API endpoint
- `api_key` — Optional API key for rate-limited endpoints

## Environment Variable Resolution

Config values support transparent environment variable references:

```json
{
  "http": "https://mainnet.base.org", // Literal string
  "http": "$RPC_URL", // Simple env var reference
  "http": "${RPC_URL}", // Braced env var reference
  "signers": ["$FACILITATOR_PRIVATE_KEY"] // Resolved from environment
}
```

The `LiteralOrEnv<T>` wrapper:

- Stores the original variable name alongside the resolved value
- `Display` reconstructs `$VAR_NAME` syntax (prevents leaking secrets in logs)
- Supports config round-tripping (save and reload without exposing values)

## Top-Level Environment Variables

| Variable                      | Description             | Default       |
| ----------------------------- | ----------------------- | ------------- |
| `HOST`                        | Server bind address     | `0.0.0.0`     |
| `PORT`                        | Server port             | `8080`        |
| `CONFIG`                      | Path to config file     | `config.json` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OpenTelemetry collector | —             |
| `OTEL_SERVICE_NAME`           | Service name            | —             |

## Feature Flags by Crate

### x402-chain-eip155

| Feature       | Description                      |
| ------------- | -------------------------------- |
| `server`      | Server-side price tag generation |
| `client`      | Client-side payment signing      |
| `facilitator` | Facilitator-side verify/settle   |
| `telemetry`   | OpenTelemetry tracing            |

### x402-chain-solana

| Feature       | Description                      |
| ------------- | -------------------------------- |
| `server`      | Server-side price tag generation |
| `client`      | Client-side payment signing      |
| `facilitator` | Facilitator-side verify/settle   |
| `telemetry`   | OpenTelemetry tracing            |

### x402-chain-aptos

| Feature       | Description                    |
| ------------- | ------------------------------ |
| `facilitator` | Facilitator-side verify/settle |
| `telemetry`   | OpenTelemetry tracing          |

Note: Aptos requires git-only dependencies and must include patches in `Cargo.toml`.

### x402-types

| Feature     | Description                   |
| ----------- | ----------------------------- |
| `cli`       | CLI argument parsing via clap |
| `telemetry` | Tracing instrumentation       |

## Workspace Dependencies

The root `Cargo.toml` defines shared versions (workspace v1.4.6, Rust edition 2024, MSRV 1.88.0):

```toml
[workspace.dependencies]
x402-types = { version = "1.0", path = "crates/x402-types" }
x402-axum = { version = "1.0", path = "crates/x402-axum" }
x402-reqwest = { version = "1.0", path = "crates/x402-reqwest" }
x402-facilitator-local = { version = "1.0", path = "crates/x402-facilitator-local" }
x402-chain-eip155 = { version = "1.0", path = "crates/chains/x402-chain-eip155" }
x402-chain-solana = { version = "1.0", path = "crates/chains/x402-chain-solana" }
x402-chain-aptos = { version = "1.0", path = "crates/chains/x402-chain-aptos" }
```

Key dependency versions:

- `alloy-primitives` 1.4.1 — EVM types and token amounts
- `axum` 0.8 — HTTP framework
- `reqwest` 0.13 — HTTP client
- `tokio` 1.35 — Async runtime
- `serde` 1.0 — Serialization
- `tower` 0.5 — Service trait and layers
