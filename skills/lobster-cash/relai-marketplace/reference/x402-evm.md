# x402 Payment Flow — EVM (SKALE, Base, Avalanche...)

> **Status: Not yet supported.** Waiting for an EVM-compatible wallet plugin (e.g. lobster.cash EVM support). This document describes the expected flow for when EVM support becomes available.

## Overview

EVM x402 payments use EIP-3009 `transferWithAuthorization` for gasless USDC transfers. The client signs an off-chain authorization, and the facilitator broadcasts it on-chain.

For full protocol details, see the [x402 protocol skill](https://github.com/TheGreatAxios/agent-skills/tree/main/x402).

## Networks (CAIP-2 format)

| Network | CAIP-2 ID |
|---|---|
| Base mainnet | `eip155:8453` |
| Base Sepolia | `eip155:84532` |
| SKALE Europa | `eip155:2046399126` |
| Avalanche mainnet | `eip155:43114` |

## Expected flow

### Step 1 — Get 402 challenge

Same as Solana: `curl` the relay endpoint, get HTTP 402.

### Step 2 — Parse 402 response

The `payment-required` header contains EVM-specific fields:

```json
{
  "x402Version": 2,
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:2046399126",
      "maxAmountRequired": "10000",
      "asset": "0x...",
      "payTo": "0x...",
      "maxTimeoutSeconds": 60
    }
  ],
  "resource": {
    "url": "https://api.relai.fi/relay/{apiId}{path}"
  }
}
```

Key differences from Solana:
- `network` uses CAIP-2 format (`eip155:*`)
- `maxAmountRequired` instead of `amount`
- `maxTimeoutSeconds` defines the payment window

### Step 3 — Sign EIP-3009 authorization

The client must sign an EIP-712 typed message for `transferWithAuthorization`:

```json
{
  "from": "0x...",
  "to": "0x...",
  "value": "10000",
  "validAfter": "1740672089",
  "validBefore": "1740672154",
  "nonce": "0x..."
}
```

This requires an EVM wallet capable of EIP-712 signing.

### Step 4 — Build X-PAYMENT header

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "exact",
    "network": "eip155:2046399126",
    "maxAmountRequired": "10000",
    "asset": "0x...",
    "payTo": "0x..."
  },
  "payload": {
    "signature": "0x...",
    "authorization": {
      "from": "0x...",
      "to": "0x...",
      "value": "10000",
      "validAfter": "...",
      "validBefore": "...",
      "nonce": "0x..."
    }
  }
}
```

Base64-encode and send as `X-PAYMENT` header.

### Step 5 — Retry with payment

Same as Solana: `curl` the `resource.url` with `X-PAYMENT` header.

The facilitator calls `transferWithAuthorization` on the ERC-20 contract to settle the payment on-chain, then returns the API response.

## What's needed to activate

- An EVM wallet plugin for openclaw that can:
  - Hold ERC-20 tokens (USDC) on target chains
  - Sign EIP-712 typed data
  - Report balances
- Update SKILL.md step 3 to route EVM networks to the new wallet
- Remove the "not yet supported" guard in the workflow
