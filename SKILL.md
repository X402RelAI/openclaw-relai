---
name: relai
description: "Browse and call paid APIs on the RelAI marketplace (relai.fi) using x402 micropayments. Delegates wallet and payment operations to lobster.cash. Use when: (1) searching for an API to accomplish a task, (2) calling a paid API endpoint, (3) checking available APIs and pricing. NOT for: free/public APIs that don't require payment."
metadata:
  {
    "openclaw":
      {
        "emoji": "🔌",
        "primaryEnv": "RELAI_API_URL"
      }
  }
---

# RelAI — x402 Paid API Client

Call any paid API on the RelAI marketplace. This skill handles API discovery and the x402 payment protocol orchestration. Transaction execution and final status are handled by lobster.cash.

## Wallet Compatibility

- **Tested wallet**: lobster.cash
- If wallet context is missing, complete lobster.cash setup first before proceeding

## Environment

- `RELAI_API_URL` — RelAI server base URL (`https://api.relai.fi`)

## When to Use

- User asks to call a paid API (AI, data, SaaS, etc.)
- User wants to find an API for a specific task
- User wants to know pricing for an API endpoint
- User says "use RelAI", "find an API for…", "call the … API"

---

## Step 1 — Discover APIs

List all available APIs:

```bash
curl -s "${RELAI_API_URL}/marketplace" | jq '.[] | {apiId, name, description, supportedNetworks, zAuthEnabled}'
```

Search by keyword:

```bash
curl -s "${RELAI_API_URL}/marketplace" | jq '[.[] | select(.name | test("KEYWORD"; "i"))] | .[] | {apiId, name, description, zAuthEnabled}'
```

**IMPORTANT**: Only use APIs where `zAuthEnabled` is `false`. APIs with `zAuthEnabled: true` require zero-knowledge authentication which is not supported by this skill. Filter them out and warn the user.

## Step 2 — Explore API Endpoints & Pricing

```bash
curl -s "${RELAI_API_URL}/marketplace/{apiId}" | jq '{
  apiId, name, description, network, zAuthEnabled,
  endpoints: [.endpoints[] | {path, method, summary, usdPrice, enabled}]
}'
```

- `usdPrice` — cost per call in USD
- `enabled` — must be `true` to call
- Only endpoints with `enabled: true` have pricing and can be called

## Step 3 — Wallet Precheck

Before any payment:

- If a lobster.cash wallet is already configured, use it. Do not create a new one.
- If no wallet is configured, ask the user to set up lobster.cash first.
- Verify the wallet has sufficient funds to cover the endpoint's `usdPrice`. If insufficient, inform the user and ask them to fund their wallet.

## Step 4 — Call the API (x402 Payment Flow)

The x402 protocol uses a challenge-response flow. This skill orchestrates the protocol; lobster.cash handles payment execution.

### 4a — Initial request (get 402 challenge)

Call the relay endpoint without payment. It will return HTTP 402 with payment requirements.

```bash
curl -s -w "\n---HTTP_STATUS:%{http_code}---" \
  "${RELAI_API_URL}/relay/{apiId}{endpointPath}" \
  -X {METHOD} \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{REQUEST_BODY}'
```

For GET requests, omit `-d`.

### 4b — Parse the 402 response

The 402 response includes a `payment-required` header (base64-encoded JSON). Decode and extract:

- `accepts[0].network` — payment network
- `accepts[0].amount` — amount in token atomic units
- `accepts[0].asset` — token address
- `accepts[0].payTo` — recipient address
- `accepts[0].extra` — additional parameters
- `resource.url` — the URL to call with payment attached

Convert the atomic amount to human-readable (e.g. 10000 atomic USDC = 0.01 USDC, since USDC has 6 decimals).

### 4c — Pay via lobster.cash

Delegate payment to lobster.cash: send the required amount to the `payTo` address. Wait for on-chain confirmation. The confirmed transaction hash is needed for the next step.

### 4d — Build the X-PAYMENT header

Once payment is confirmed, construct the payment proof header using the transaction hash:

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

Replace placeholders with values from the 402 response and the confirmed transaction hash from lobster.cash.

### 4e — Retry with payment

```bash
curl -s "${RESOURCE_URL}" \
  -X {METHOD} \
  -H "X-PAYMENT: ${PAYMENT_HEADER}" \
  -H "Content-Type: application/json" \
  -d '{REQUEST_BODY}'
```

Use `resource.url` from the 402 response. The request body must be identical to step 4a. The facilitator verifies the on-chain payment and returns the API response.

---

## Alternative: Pre-fetch pricing without calling

```bash
curl -s "${RELAI_API_URL}/marketplace/{apiId}/x402?method={METHOD}&path={PATH}" | jq '.'
```

## Error Handling

| Situation | Action |
|---|---|
| Wallet not configured | Ask user to complete lobster.cash setup first |
| Insufficient balance | Show required amount, ask user to fund wallet |
| `zAuthEnabled: true` | Inform user this API requires zAuth (not supported) |
| Payment failed | Display error from lobster.cash, suggest retry |
| Awaiting confirmation | Wait for lobster.cash to report final status before continuing |
| API returned 4xx/5xx after payment | Upstream API error — show the error to the user |

## Full Example

```
User: "Check the NovaShield health endpoint"

1. Discover: find NovaShield (zAuthEnabled: false) ✓
2. Explore: GET /v1/health, $0.01/call, Solana
3. Wallet precheck: lobster.cash configured, 9.95 USDC, sufficient
4a. curl relay → HTTP 402 with payment-required header
4b. Parse: 10000 atomic = 0.01 USDC, payTo=DnoU...Mh1, network=solana
4c. Pay via lobster.cash: send 0.01 USDC to payTo, wait for confirmation → tx hash
4d. Build X-PAYMENT header with txId = confirmed hash
4e. curl with X-PAYMENT → {"status":"healthy","service":"NovaShield Security Engine"}
```

## Notes

- This skill delegates wallet operations to lobster.cash
- This skill owns: API discovery, x402 protocol orchestration, request/response handling
- lobster.cash owns: wallet provisioning, transaction signing/approval/broadcast, transaction state
- The relay URL pattern is `${RELAI_API_URL}/relay/{apiId}{endpointPath}`
