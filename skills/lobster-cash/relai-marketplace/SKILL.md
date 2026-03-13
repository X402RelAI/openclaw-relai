---
name: relai
version: 1.1.0
description: "Browse and call paid APIs on the RelAI marketplace (relai.fi) using x402 micropayments. Supports Solana (via lobster.cash) with EVM support planned. Use when: searching for an API, calling a paid endpoint, or checking pricing. NOT for: free/public APIs."
metadata: {"openclaw": {"emoji": "🔌", "requires": {"bins": ["curl", "jq"]}}}
---

# RelAI — x402 Paid API Client

Call paid APIs on the RelAI marketplace. Handles API discovery and x402 payment protocol across multiple chains.

## Supported networks

| Network | Status | Wallet | Reference |
|---|---|---|---|
| Solana | Active | lobster.cash | [reference/x402-solana.md](reference/x402-solana.md) |
| EVM (SKALE, Base...) | Planned | TBD | [reference/x402-evm.md](reference/x402-evm.md) |

## Constants

```
RELAI_API_URL = https://api.relai.fi
```

Override: set `RELAI_API_URL` env var to use a different instance.

## Inputs needed

- A configured wallet with USDC balance on the target network

## Workflow

### Step 0 — Validate prerequisites

1) Set `RELAI_API_URL` to `https://api.relai.fi` unless the env var is already set (user override).
2) Verify a wallet is configured for the target network. Currently only lobster.cash (Solana) is supported.

### Step 1 — Discover APIs

See [reference/marketplace-api.md](reference/marketplace-api.md) for full endpoint details.

```bash
curl -s "${RELAI_API_URL}/marketplace" | jq '.[] | {apiId, name, description, supportedNetworks, zAuthEnabled}'
```

Filter rules — skip and warn the user if any apply:
- `zAuthEnabled` is `true` (zAuth not supported)

### Step 2 — Explore endpoints and pricing

```bash
curl -s "${RELAI_API_URL}/marketplace/{apiId}" | jq '{
  apiId, name, description, network, zAuthEnabled,
  endpoints: [.endpoints[] | {path, method, summary, usdPrice, enabled}]
}'
```

Only endpoints with `enabled: true` and a `usdPrice` can be called.

### Step 3 — Identify network and wallet

From the API details, determine the payment network:
- `solana` → use lobster.cash. See [reference/x402-solana.md](reference/x402-solana.md).
- `eip155:*` (Base, SKALE, etc.) → EVM wallet required. See [reference/x402-evm.md](reference/x402-evm.md). **Not yet supported — inform the user and stop.**

### Step 4 — Wallet precheck

1) Check wallet balance on the target network.
2) If balance < endpoint `usdPrice`, inform the user and stop.
3) Show the user the endpoint name, price, network, and ask for confirmation before proceeding.

### Step 5 — Call the API (x402 payment flow)

1) **Get 402 challenge**: `curl` the relay endpoint without payment. Expect HTTP 402.
2) **Parse**: decode `payment-required` header (base64 JSON). Extract `network`, `amount`, `asset`, `payTo`, `resource.url`.
3) **Pay**: route to the network-specific flow:
   - **Solana**: send amount to `payTo` via lobster.cash, wait for confirmation, get `txId`. See [reference/x402-solana.md](reference/x402-solana.md).
   - **EVM**: not yet supported.
4) **Build X-PAYMENT**: construct header with payment proof (format varies by network, see reference docs).
5) **Retry**: `curl` the `resource.url` with `X-PAYMENT` header. Same request body as step 1.

## Output format

- **API discovered**: name, apiId, price, network
- **API response**: raw JSON from the endpoint
- **Payment summary**: amount paid, network, transaction hash, wallet balance after

## Guardrails

- Never call a paid endpoint without confirming the price with the user first.
- Never fabricate API responses or transaction hashes.
- Never create a new wallet if one already exists.
- Do not retry a failed payment automatically — show the error and let the user decide.
- Do not call APIs with `zAuthEnabled: true`.
- Do not attempt payment on an unsupported network — inform the user.

## Error handling

| Situation | Action |
|---|---|
| `RELAI_API_URL` override invalid | Warn and fall back to default |
| Wallet not configured | Stop, ask user to set up wallet for that network |
| Insufficient balance | Show required amount, stop |
| `zAuthEnabled: true` | Inform user, skip this API |
| Unsupported network | Inform user which network is needed, stop |
| Non-402 response | Show the response and status code |
| Payment failed or timed out | Show error, stop |
| API returned 4xx/5xx after payment | Show the raw error response |
