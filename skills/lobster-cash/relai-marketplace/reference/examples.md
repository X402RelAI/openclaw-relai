# Examples

## Discover APIs

```
User: "What APIs are available on RelAI?"

1. curl marketplace -> list APIs
2. Filter: zAuthEnabled=false, supportedNetworks includes solana
3. Present list to user

Output:
- NovaShield (nshield) — Security engine, $0.01/call, Solana
- Xona (xona) — Data API, $0.005/call, Solana
```

## Call a paid endpoint

```
User: "Check the NovaShield health endpoint"

0. Prerequisites: RELAI_API_URL set, wallet configured
1. Discover: find NovaShield (zAuthEnabled: false, Solana)
2. Explore: GET /v1/health, $0.01/call
3. Wallet precheck: 9.95 USDC, sufficient. Confirm with user.
4. x402 flow:
   - curl relay -> HTTP 402
   - Parse: 10000 atomic = 0.01 USDC, payTo=DnoU...Mh1, network=solana
   - lobster.cash send 0.01 USDC -> confirmed, tx hash = 5xK...
   - Build X-PAYMENT with txId
   - curl with X-PAYMENT -> {"status":"healthy","service":"NovaShield Security Engine"}

Output:
- API: NovaShield /v1/health (GET)
- Response: {"status":"healthy","service":"NovaShield Security Engine"}
- Payment: 0.01 USDC, tx 5xK..., balance: 9.94 USDC
```

## Check pricing without calling

```
User: "How much does NovaShield health check cost?"

1. curl /marketplace/nshield -> list endpoints
2. Find /v1/health: $0.01/call, GET, enabled

Output:
- NovaShield /v1/health (GET): $0.01 per call, Solana network
```

## Unsupported API (zAuth)

```
User: "Call the TGM API"

1. Discover: find TGM -> zAuthEnabled: true
2. Inform user: "TGM requires zero-knowledge authentication which is not supported by this skill."
```

## Unsupported network

```
User: "Call the Weather API"

1. Discover: find Weather -> supportedNetworks: ["base"]
2. Inform user: "Weather API requires Base network. This skill only supports Solana via lobster.cash."
```
