# x402-skills

A comprehensive skill for the LLM that enables building solutions with the [x402 protocol](https://www.x402.org) and the [x402-rs](https://github.com/x402-rs/x402-rs) Rust library. The x402 protocol activates the long-dormant HTTP `402 Payment Required` status code to enable blockchain payments directly through HTTP — the client never touches the blockchain, and the facilitator handles all on-chain operations.

## Overview

With this skill, the LLM developer can:

- **Create paid API services** — Protect Axum routes behind blockchain micropayments using `x402-axum` middleware. Accept any ERC-20 token, SPL token, or Aptos coin across EVM, Solana, and Aptos blockchains.
- **Build paying agents** — Create `reqwest` clients that transparently handle x402 payments when accessing paid data endpoints, with automatic signature generation and retry logic.
- **Run and customize facilitators** — Deploy production facilitators via Docker or from source, or build fully custom facilitators with programmatic verify/settle access.
- **Implement gasless payments** — Leverage the full gasless stack: EIP-3009 (preferred), Permit2 (universal fallback), and EIP-2612 gas sponsoring.
- **Support smart wallets** — Handle EOA, EIP-1271 (deployed smart wallets), and EIP-6492 (counterfactual wallets) seamlessly.
- **Work with any token** — Not limited to USDC. Any ERC-20 token, SPL token, or native coin can be used as payment.

## Installation

```bash
npx skills add melonask/x402-skills
```

## File Structure

```text
x402/
├── SKILL.md                          # Main entry point (336 lines)
│   ├── Protocol overview & 3-actor diagram
│   ├── Crate architecture table
│   ├── Quick-start routing guide
│   ├── Minimal working examples (server, client, multi-chain)
│   ├── Key concepts (any token, gasless, V1 vs V2, price tag types)
│   ├── Installation patterns (server, client, facilitator)
│   ├── Common patterns (dynamic pricing, multi-chain, settlement timing)
│   └── Reference file routing
│
└── references/
    ├── server-guide.md                # Axum middleware — protecting routes (294 lines)
    ├── client-guide.md                # Reqwest middleware — auto-paying agents (253 lines)
    ├── facilitator-guide.md           # Facilitator setup & custom builds (358 lines)
    ├── chain-config.md                # Networks, tokens, RPC, feature flags (250 lines)
    ├── protocol-details.md            # V1/V2, gasless stack, smart wallets (365 lines)
    └── custom-scheme-guide.md         # Custom schemes & EIP-7702 implementation (150 lines)
```

The skill uses **progressive disclosure**: the `SKILL.md` main file stays concise (under 500 lines) while detailed guides are organized into domain-specific reference files that are loaded on demand.

## Supported Blockchains

| Chain             | Networks                                                                  | Protocol Versions | Payment Methods                     |
| ----------------- | ------------------------------------------------------------------------- | ----------------- | ----------------------------------- |
| **EVM (EIP-155)** | Base, Polygon, Avalanche, Sei, Celo, XDC, XRPL EVM, Peaq, IoTeX, and more | V1 + V2           | EIP-3009, Permit2                   |
| **Solana**        | Mainnet, Devnet                                                           | V1 + V2           | SPL Token pre-signed transfer       |
| **Aptos**         | Mainnet, Testnet                                                          | V2                | Fungible asset transfer (sponsored) |

## Quick Examples

### Protect a Route (Server)

```rust
use x402_axum::X402Middleware;
use x402_chain_eip155::{V2Eip155Exact, KnownNetworkEip155};
use x402_types::networks::USDC;

let x402 = X402Middleware::new("https://facilitator.x402.rs");

let app = Router::new().route(
    "/paid-content",
    get(handler).layer(
        x402.with_price_tag(V2Eip155Exact::price_tag(
            address!("0xYourWallet"),
            USDC::base_sepolia().amount(10u64),
        ))
    ),
);
```

### Auto-Pay for Data (Client)

```rust
use x402_reqwest::{X402Client, ReqwestWithPayments, ReqwestWithPaymentsBuild};
use x402_chain_eip155::V2Eip155ExactClient;

let client = Client::new()
    .with_payments(
        X402Client::new()
            .register(V2Eip155ExactClient::new(signer))
    )
    .build();

// Payments handled transparently
let res = client.get("https://api.example.com/protected").send().await?;
```

### Run a Facilitator (Docker)

```bash
docker run -v $(pwd)/config.json:/app/config.json -p 8080:8080 \
  ghcr.io/x402-rs/x402-facilitator
```

## x402-rs Crate Overview

| Crate                    | Published | Purpose                                                               |
| ------------------------ | --------- | --------------------------------------------------------------------- |
| `x402-types`             | crates.io | Core types, facilitator traits, CAIP-2 chain IDs, wire format (V1/V2) |
| `x402-axum`              | crates.io | Axum middleware for protecting routes with payments                   |
| `x402-reqwest`           | crates.io | Reqwest middleware for transparent payment handling                   |
| `x402-facilitator-local` | crates.io | Local facilitator implementation                                      |
| `x402-chain-eip155`      | crates.io | EVM/EIP-155 chain support                                             |
| `x402-chain-solana`      | crates.io | Solana chain support                                                  |
| `x402-chain-aptos`       | git-only  | Aptos chain support                                                   |
| `x402-facilitator`       | git-only  | Production facilitator binary                                         |

## Protocol Highlights

- **Gasless for clients** — The client only produces an off-chain cryptographic signature. The facilitator pays gas.
- **Trust-minimized** — The facilitator cannot modify the payment amount or destination (locked by the client's signature).
- **Any token** — Works with any ERC-20 token, SPL token, or fungible asset, not just USDC.
- **Zero protocol fees** — Only nominal payment network fees.
- **V1 and V2** — Full backward compatibility; V2 adds CAIP-2 chain IDs, multi-chain, and extensible schemes.

## References

- [x402 Protocol Specification](https://www.x402.org)
- [x402-rs GitHub Repository](https://github.com/x402-rs/x402-rs)
- [Coinbase x402 Documentation](https://docs.cdp.coinbase.com/x402/docs/overview)
- [CAIP-2 Chain IDs](https://chainagnostic.org/CAIPs/caip-2)
- [EIP-3009: transferWithAuthorization](https://eips.ethereum.org/EIPS/eip-3009)
- [Permit2 by Uniswap](https://github.com/Uniswap/permit2)

## License

This skill documentation is provided for use with the x402 protocol ecosystem. The x402-rs library itself is licensed under Apache-2.0.
