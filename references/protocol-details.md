# Protocol Details: V1 vs V2, Schemes, and Gasless Payments

This reference covers the internal workings of the x402 protocol, the differences between V1 and V2, the gasless payment stack (EIP-3009, Permit2, EIP-2612), smart wallet support, and custom scheme implementation.

## V1 vs V2 Protocol

### V1 Protocol

Uses network names (e.g., `"base-sepolia"`) and JSON-based communication:

- **402 response**: JSON body with `accepts` array and `x402Version: "1"`
- **Payment header**: `X-PAYMENT: <base64>`
- **Payment response**: JSON body
- **Network identification**: String names like `"base"`, `"polygon"`
- **Scope**: Single-call, exact payments, USDC-focused

### V2 Protocol

Uses CAIP-2 chain IDs and HTTP headers:

- **402 response**: `Payment-Required: <base64>` header
- **Payment header**: `Payment-Signature: <base64>`
- **Payment response**: `Payment-Response: <base64>` header
- **Network identification**: CAIP-2 format like `"eip155:84532"`, `"solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp"`
- **Scope**: Multi-chain, extensible schemes, any token, dynamic pricing

### V2 PaymentRequired Structure

```json
{
  "x402Version": "2",
  "resource": {
    "url": "https://api.example.com/premium",
    "description": "Premium API access",
    "mimeType": "application/json"
  },
  "paymentRequirements": [
    {
      "scheme": "exact",
      "network": "eip155:84532",
      "amount": "10000",
      "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
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

### V2 PaymentPayload Structure

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://api.example.com/premium-data",
    "description": "Access to premium market data",
    "mimeType": "application/json"
  },
  "accepted": {
    "scheme": "exact",
    "network": "eip155:84532",
    "amount": "10000",
    "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
    "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
    "maxTimeoutSeconds": 60,
    "extra": {
      "assetTransferMethod": "eip3009"
    }
  },
  "payload": {
    "signature": "0x2d6a7588...148b571c",
    "authorization": {
      "from": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
      "to": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "value": "10000",
      "validAfter": "1740672089",
      "validBefore": "1740672154",
      "nonce": "0xf3746613..."
    }
  }
}
```

Both V1 and V2 are supported simultaneously by x402-rs. The client automatically detects which protocol the server uses.

## Payment Schemes

### exact Scheme

The primary scheme. Transfers a specific fixed amount. "Pay exactly $0.01 to read this article."

**Built-in implementations:**

| Implementation  | Protocol | Chain  | Transfer Method                      |
| --------------- | -------- | ------ | ------------------------------------ |
| `V1Eip155Exact` | V1       | EVM    | EIP-3009 `transferWithAuthorization` |
| `V2Eip155Exact` | V2       | EVM    | EIP-3009 or Permit2                  |
| `V1SolanaExact` | V1       | Solana | SPL Token pre-signed transfer        |
| `V2SolanaExact` | V2       | Solana | SPL Token pre-signed transfer        |
| `V2AptosExact`  | V2       | Aptos  | Fungible asset transfer              |

### upto Scheme (Planned)

Transfers up to a specified amount based on resources consumed. "Pay up to $0.05 for this LLM response."

Currently available for EIP-155 chains only:

- `V2Eip155Upto` — V2 EVM upto payment (experimental)

### deferred Scheme (Planned)

Supports deferred settlement and payment scheduling.

## Gasless Payment Stack (EVM)

The x402 protocol provides a layered approach to gasless payments on EVM chains. The client never pays gas — the facilitator does.

### Layer 1: EIP-3009 (Preferred)

**Use when the token supports it** (e.g., USDC on Base, Ethereum, Polygon).

EIP-3009 defines `transferWithAuthorization`, allowing a token holder to authorize a transfer with an off-chain signature:

```solidity
function transferWithAuthorization(
    address from,
    address to,
    uint256 value,
    uint256 validAfter,
    uint256 validBefore,
    bytes32 nonce,
    bytes memory signature
) external
```

**Flow:**

1. Client signs an EIP-712 typed `TransferWithAuthorization` message off-chain
2. Sends signature in `Payment-Signature` header
3. Facilitator calls `token.transferWithAuthorization(from, to, value, validAfter, validBefore, nonce, signature)` on-chain
4. One-time use — each nonce can only be used once

**Why preferred:** Simplest flow, truly gasless for the client, single on-chain call.

### Layer 2: Permit2 (Universal Fallback)

**Use for any ERC-20 token** that doesn't support EIP-3009.

Uses Uniswap's Permit2 contract (`0x000000000022D473030F116dDEE9F6B43aC78BA3`) with the `X402ExactPermit2Proxy` (`0x4020615294c913F045dc10f0a5cdEbd86c280001`).

**Phase 1: One-Time Setup**

The user must approve Permit2 to spend their tokens. Three options:

- **Option A**: User submits `approve(Permit2)` on-chain themselves (pays own gas)
- **Option B**: `erc20ApprovalGasSponsoring` — Facilitator sponsors the approval transaction
- **Option C**: `eip2612GasSponsoring` — If token supports EIP-2612 `permit()`, user signs a permit, facilitator calls `x402ExactPermit2Proxy.settleWithPermit()`

**Phase 2: Payment**

1. Client signs a `permitWitnessTransferFrom` EIP-712 message with the `payTo` address as a "witness"
2. The witness prevents the spender (Permit2 proxy) from routing funds elsewhere
3. Facilitator verifies the signature (supports EIP-6492 and EIP-1271)
4. Facilitator validates allowance, balance, and constraints
5. Facilitator settles via `X402ExactPermit2Proxy`

### Layer 3: EIP-2612 (Gas Sponsorship Helper)

Not a standalone payment method. Used as an extension to make the Permit2 approval step gasless:

- Token implements `permit(owner, spender, value, deadline, v, r, s)` (EIP-2612)
- User signs a permit message off-chain
- Facilitator submits the permit + approval in one call
- User never pays gas

**In x402-rs:** The `Permit2PaymentPayloadExt` trait provides `eip2612_gas_sponsoring()` for unified EIP-2612 handling.

### Priority Order

When `assetTransferMethod` is not specified in the payment requirements:

1. **EIP-3009** — if the token supports it (simplest, truly gasless)
2. **Permit2** — universal fallback for any ERC-20

When `assetTransferMethod` is specified:

- `"eip3009"` — Force EIP-3009 path
- `"permit2"` — Force Permit2 path

## Smart Wallet Support

x402-rs supports smart contract wallets (not just EOAs):

### EIP-1271 (Deployed Smart Wallets)

Smart wallets that implement `isValidSignature(address, bytes32, bytes)` can produce signatures that the facilitator verifies via contract call.

### EIP-6492 (Counterfactual Smart Wallets)

For smart wallets that are **not yet deployed** on-chain. Uses a magic byte suffix:

```
<signature><32-byte-magic:0x6492649264926492649264926492649264926492649264926492649264926492><validator_address>
```

The facilitator detects this format and:

1. Tries to call `isValidSignature` on the wallet (if deployed)
2. If reverted with `EIP6492` error, deploys the wallet first, then verifies
3. Uses the `Validator6492` contract (`0x82c6D79a13E6F42A8C7A3Ea24A4628B1eD9c5E34`) for validation

### Signature Detection in x402-rs

The facilitator intelligently dispatches based on signature format:

- **EOA signatures** (64-65 bytes): Parsed as `(r, s, v)`, dispatched to standard EIP-3009 function
- **EIP-1271 signatures**: Passed as full bytes for contract wallet verification
- **EIP-6492 signatures**: Detected by 32-byte magic suffix, validated via Validator6492

## CAIP-2 Chain IDs (V2)

The V2 protocol uses CAIP-2 (Chain Agnostic Improvement Proposal 2) identifiers:

```
{namespace}:{reference}
```

Examples:

- `eip155:8453` — Base Mainnet
- `eip155:84532` — Base Sepolia
- `eip155:137` — Polygon
- `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` — Solana Mainnet
- `aptos:1` — Aptos Mainnet

**Parsing:**

```rust
use x402_types::chain::ChainId;

let chain_id: ChainId = "eip155:8453".parse().unwrap();
assert_eq!(chain_id.namespace(), "eip155");
assert_eq!(chain_id.reference(), "8453");

// V1 compatibility
let chain_id = ChainId::from_network_name("base").unwrap();
```

V2 is designed to work with **any** CAIP-2 chain ID without requiring a predefined registry.

## Solana Payment Flow

Solana payments use pre-signed `VersionedTransaction`:

1. Client receives `PaymentRequired` response with price tags
2. Client selects Solana payment option
3. Client creates a `VersionedTransaction` containing:
   - SPL Token transfer instruction
   - Compute budget instructions (compute unit limit + price)
4. Client signs with their keypair
5. Client serializes and base64-encodes the transaction
6. Facilitator deserializes, validates, simulates, checks balance
7. Facilitator adds its signature as fee payer and submits on-chain
8. Waits for confirmation

## Aptos Payment Flow

Aptos payments use sponsored transactions with BCS-encoded payloads:

1. Client creates a transaction calling `0x1::primary_fungible_store::transfer`
2. Client signs with their Ed25519 key
3. Client BCS-encodes and base64-encodes the transaction
4. Facilitator deserializes, validates, simulates, checks balance
5. If `sponsor_gas` is enabled, facilitator adds fee payer signature
6. Facilitator submits the dual-signed transaction on-chain

## Custom Scheme Implementation

Create a custom payment scheme by implementing the `X402SchemeId` trait and either the client or facilitator trait:

### Scheme ID

```rust
use x402_types::scheme::X402SchemeId;
use x402_types::proto::{v1, v2};

pub struct MyCustomScheme;

impl X402SchemeId for MyCustomScheme {
    fn x402_version(&self) -> u8 { 2 }
    fn namespace(&self) -> &str { "eip155" }
    fn scheme(&self) -> &str { "my-custom" }
}
```

### Facilitator Builder

```rust
use x402_types::scheme::X402SchemeFacilitatorBuilder;

impl<P> X402SchemeFacilitatorBuilder<P> for MyCustomScheme {
    fn build(&self, provider: P, config: Option<&serde_json::Value>)
        -> Result<Box<dyn x402_types::facilitator::Facilitator>, Box<dyn std::error::Error>>
    {
        // Return your facilitator implementation
        todo!()
    }
}
```

### Client-Side

Implement `accept` to sign payments and return `PaymentPayload`:

```rust
impl PaymentClient for MyCustomClient {
    fn accept(&self, payment_required: &PaymentRequired) -> Vec<PaymentCandidate> {
        // Check if this client can handle the payment requirements
        // Return candidates with signed payloads
        todo!()
    }
}
```

For the complete guide on writing schemes, see the x402-rs docs at `docs/how-to-write-a-scheme.md`.

## Wire Format Types

### x402-types::proto::v1

V1 protocol types using network names.

### x402-types::proto::v2

V2 protocol types using CAIP-2 chain IDs:

- `PaymentRequired` — 402 response structure
- `PaymentRequirements` — Individual payment option
- `PaymentPayload` — Client's signed payment
- `PriceTag` — Server-side payment configuration
- `VerifyRequest` / `VerifyResponse` — Facilitator verify
- `SettleRequest` / `SettleResponse` — Facilitator settle
- `SupportedResponse` — Facilitator capabilities

## Key Facilitator Traits

```rust
// Core facilitator trait
#[async_trait]
pub trait Facilitator: Send + Sync {
    async fn verify(&self, request: VerifyRequest) -> Result<VerifyResponse, FacilitatorError>;
    async fn settle(&self, request: SettleRequest) -> Result<SettleResponse, FacilitatorError>;
}

// Scheme ID trait
pub trait X402SchemeId: Send + Sync {
    fn x402_version(&self) -> u8;
    fn namespace(&self) -> &str;
    fn scheme(&self) -> &str;
}

// Scheme facilitator builder
pub trait X402SchemeFacilitatorBuilder<P>: X402SchemeId {
    fn build(&self, provider: P, config: Option<&Value>)
        -> Result<Box<dyn Facilitator>, Box<dyn Error>>;
}
```
