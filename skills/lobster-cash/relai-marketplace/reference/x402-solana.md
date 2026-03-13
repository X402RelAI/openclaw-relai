# x402 Payment Flow

The x402 protocol uses HTTP 402 challenge-response for micropayments.

## Overview

```
Client                     Relay                    Facilitator        Blockchain
  |-- request (no payment) -->|                          |                  |
  |<-- 402 + payment-required-|                          |                  |
  |                           |                          |                  |
  |-- pay via wallet -------->|------------------------->|----------------->|
  |<-- tx confirmed ----------|                          |                  |
  |                           |                          |                  |
  |-- request + X-PAYMENT --->|-- verify payment ------->|-- check chain -->|
  |<-- API response ----------|<-- settlement OK --------|                  |
```

## Step 1 — Get 402 challenge

Call the relay endpoint without payment headers.

```bash
curl -s -w "\n---HTTP_STATUS:%{http_code}---" \
  "${RELAI_API_URL}/relay/{apiId}{endpointPath}" \
  -X {METHOD} \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{REQUEST_BODY}'
```

For GET requests, omit `-d`.

Expected: HTTP 402 with `payment-required` response header.

## Step 2 — Parse 402 response

The `payment-required` header is base64-encoded JSON:

```json
{
  "x402Version": 2,
  "accepts": [
    {
      "scheme": "exact",
      "network": "solana",
      "amount": "10000",
      "asset": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "payTo": "DnoU...Mh1",
      "extra": {
        "feePayer": "DEXV...kNV"
      }
    }
  ],
  "resource": {
    "url": "https://api.relai.fi/relay/nshield/v1/health"
  }
}
```

### Key fields

| Field | Description |
|---|---|
| `accepts[0].network` | Payment network (must be `solana`) |
| `accepts[0].amount` | Amount in token atomic units |
| `accepts[0].asset` | Token mint address (USDC on Solana) |
| `accepts[0].payTo` | Recipient wallet address |
| `accepts[0].extra` | Additional parameters (e.g. feePayer for gas sponsoring) |
| `resource.url` | URL to retry with payment proof |

### Amount conversion

USDC on Solana has 6 decimals:
- `10000` atomic = `0.01` USDC
- `100000` atomic = `0.10` USDC
- `1000000` atomic = `1.00` USDC

## Step 3 — Pay via lobster.cash

Send the required amount to the `payTo` address. Use lobster.cash's send functionality and wait for on-chain confirmation. The confirmed transaction hash is needed for the next step.

## Step 4 — Build X-PAYMENT header

Construct payment proof as base64-encoded JSON:

```bash
PAYMENT_HEADER=$(echo -n '{
  "x402Version": 2,
  "accepted": {
    "scheme": "exact",
    "network": "{NETWORK}",
    "amount": "{AMOUNT}",
    "payTo": "{PAY_TO}",
    "asset": "{ASSET}"
  },
  "payload": {
    "txId": "{TX_HASH}"
  }
}' | base64 -w 0)
```

Replace placeholders:
- `{NETWORK}` — from `accepts[0].network`
- `{AMOUNT}` — from `accepts[0].amount` (atomic units, as string)
- `{PAY_TO}` — from `accepts[0].payTo`
- `{ASSET}` — from `accepts[0].asset`
- `{TX_HASH}` — confirmed transaction hash from lobster.cash

## Step 5 — Retry with payment

```bash
curl -s "${RESOURCE_URL}" \
  -X {METHOD} \
  -H "X-PAYMENT: ${PAYMENT_HEADER}" \
  -H "Content-Type: application/json" \
  -d '{REQUEST_BODY}'
```

- Use `resource.url` from step 2
- Request body must be identical to step 1
- The facilitator verifies the on-chain payment and returns the API response

## Error cases

| HTTP Status | Meaning |
|---|---|
| 200 | Success — API response returned |
| 402 | Payment required (expected on first call) |
| 400 | Bad request or invalid payment header |
| 401 | Payment verification failed |
| 404 | API or endpoint not found |
| 5xx | Upstream API or facilitator error |
