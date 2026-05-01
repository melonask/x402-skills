# Custom Scheme Guide: Building Extensions (e.g., EIP-7702)

The x402 protocol is heavily extensible. If the built-in `exact` or `upto` schemes do not fit your use case, you can define **Custom Schemes**. A scheme dictates the lifecycle of a payment: how the server declares the price, how the client authorizes it, and how the facilitator verifies and settles it on-chain.

This guide walks through building a complete custom scheme: **`v2-eip155-eip7702`**. This scheme leverages EIP-7702 to allow a user's Externally Owned Account (EOA) to temporarily act as a smart contract (delegate) and execute a gasless transaction sponsored by the Facilitator. Smart contract `Delegate.sol` the exact same address (`0xD064939e706dC03699dB7Fe58bB0553afDF39fDd`) deployed across Ethereum Mainnet, L2s (Base, Optimism, Arbitrum), and alternative L1s (Polygon, BNB, Avalanche).

## 1. Scheme Definition & Types

First, define the scheme identifier and the wire-format types for the `Requirements` (what the server asks for) and the `Payload` (what the client sends back).

```rust
use alloy_primitives::{Address, Bytes, U256};
use alloy_eips::eip7702::SignedAuthorization;
use serde::{Deserialize, Serialize};
use x402_types::scheme::X402SchemeId;

pub struct V2Eip155Eip7702;

impl X402SchemeId for V2Eip155Eip7702 {
    fn x402_version(&self) -> u8 { 2 }
    fn namespace(&self) -> &str { "eip155" }
    fn scheme(&self) -> &str { "eip7702" }
}

/// Server-side: Declared in the 402 Payment Required header
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Eip7702Requirements {
    pub token: Address,
    pub amount: U256,
    pub payee: Address,
    pub required_delegate: Address, // The EIP-7702 contract to delegate to
}

/// Client-side: Attached to the retried request
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Eip7702Payload {
    pub sender: Address,
    pub authorization: SignedAuthorization,
    pub calldata: Bytes, // Execution intent (e.g., transfer(payee, amount))
}
```

## 2. Server-Side: Generating the Price Tag

To protect Axum routes with your new scheme, implement a helper that generates a `v2::PriceTag`. The server embeds your custom `Eip7702Requirements` into the JSON wire format.

```rust
use x402_types::proto::v2::{PriceTag, PaymentRequirements};

impl V2Eip155Eip7702 {
    pub fn price_tag(
        network: String, // e.g., "eip155:84532"
        token: Address,
        amount: U256,
        payee: Address,
        required_delegate: Address,
    ) -> PriceTag {
        let reqs = Eip7702Requirements { token, amount, payee, required_delegate };

        PriceTag {
            requirements: PaymentRequirements {
                scheme: "eip7702".to_string(),
                network: network.parse().unwrap(),
                amount: amount.to_string(),
                asset: token.to_string(),
                pay_to: payee.to_string(),
                max_timeout_seconds: 300,
                // Embed custom scheme constraints
                extra: Some(serde_json::to_value(reqs).unwrap()),
            },
            enricher: None,
        }
    }
}
```

## 3. Facilitator-Side: Verify and Settle

The facilitator acts as the trustless verifier and gas sponsor. Implement `X402SchemeFacilitator` to handle the off-chain verification and the Type 4 (EIP-7702) on-chain settlement.

```rust
use x402_types::facilitator::{FacilitatorError, X402SchemeFacilitator};
use alloy_rpc_types::TransactionRequest;
use alloy_provider::Provider;
use async_trait::async_trait;

pub struct Eip7702Facilitator<P> {
    pub provider: P,
}

#[async_trait]
impl<P: Provider + Send + Sync> X402SchemeFacilitator for Eip7702Facilitator<P> {
    type Requirements = Eip7702Requirements;
    type Payload = Eip7702Payload;

    async fn verify(&self, reqs: &Self::Requirements, payload: &Self::Payload) -> Result<(), FacilitatorError> {
        // 1. Verify EIP-7702 delegation target matches the merchant's requirement
        if payload.authorization.address() != &reqs.required_delegate {
            return Err(FacilitatorError::VerificationFailed("Invalid delegate contract".into()));
        }

        // 2. Recover the signature to ensure the sender authorized the delegation
        let recovered = payload.authorization.recover_authority()
            .map_err(|_| FacilitatorError::VerificationFailed("Invalid signature".into()))?;

        if recovered != payload.sender {
            return Err(FacilitatorError::VerificationFailed("Signer mismatch".into()));
        }

        Ok(())
    }

    async fn settle(&self, _reqs: &Self::Requirements, payload: &Self::Payload) -> Result<String, FacilitatorError> {
        // Construct the EIP-7702 Type 4 transaction
        // The facilitator pays gas, executing the calldata against the delegated EOA
        let tx = TransactionRequest::default()
            .with_to(payload.sender)
            .with_call_data(payload.calldata.clone())
            .with_authorization_list(vec![payload.authorization.clone()]);

        let pending = self.provider.send_transaction(tx).await
            .map_err(|e| FacilitatorError::SettlementFailed(e.to_string()))?;

        let receipt = pending.get_receipt().await
            .map_err(|e| FacilitatorError::SettlementFailed(e.to_string()))?;

        Ok(receipt.transaction_hash.to_string())
    }
}
```

_Note: You must also implement `X402SchemeFacilitatorBuilder` to register this with the `SchemeRegistry` (see `references/facilitator-guide.md`)._

## 4. Client-Side: Automatically Paying (Reqwest)

To make your Rust AI agents capable of paying via your custom scheme, implement the `PaymentClient` trait. The client intercepts the requirements, uses Alloy to sign the EIP-7702 authorization tuple, and builds the payload.

```rust
use x402_types::scheme::client::{PaymentClient, PaymentCandidate};
use x402_types::proto::v2::{PaymentRequired, PaymentPayload, PayloadData};
use alloy_signer::Signer;
use alloy_network::Ethereum;

pub struct V2Eip155Eip7702Client<S> {
    signer: S,
}

impl<S: Signer + Send + Sync> PaymentClient for V2Eip155Eip7702Client<S> {
    fn accept(&self, required: &PaymentRequired) -> Vec<PaymentCandidate> {
        let mut candidates = Vec::new();

        for req in &required.payment_requirements {
            if req.scheme == "eip7702" && req.network.namespace() == "eip155" {
                // Parse the extra EIP-7702 requirements
                let custom_reqs: Eip7702Requirements = serde_json::from_value(req.extra.clone().unwrap()).unwrap();

                // Sign the EIP-7702 Authorization tuple (mock async context)
                // In a real client, this logic runs inside the async payload builder
                let auth = self.signer.sign_auth_tuple(custom_reqs.required_delegate).await.unwrap();

                // Build execution calldata (e.g., triggering the transfer)
                let calldata = build_calldata(&custom_reqs);

                let payload = Eip7702Payload {
                    sender: self.signer.address(),
                    authorization: auth,
                    calldata,
                };

                let payment_payload = PaymentPayload {
                    x402_version: 2,
                    resource: required.resource.clone(),
                    accepted: req.clone(),
                    payload: PayloadData::Custom(serde_json::to_value(payload).unwrap()),
                };

                candidates.push(PaymentCandidate::new(req.clone(), payment_payload));
            }
        }
        candidates
    }
}
```

## Summary

By breaking out the logic into these four components (`Types`, `PriceTag`, `Facilitator`, `Client`), you can drop absolutely any blockchain interaction logic into the `x402-rs` ecosystem. The Axum and Reqwest middlewares will automatically handle the HTTP lifecycle, allowing you to focus entirely on the cryptographic scheme.
