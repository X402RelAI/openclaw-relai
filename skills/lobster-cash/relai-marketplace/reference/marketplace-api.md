# RelAI Marketplace API

Base URL: `${RELAI_API_URL}`

## List all APIs

```
GET /marketplace
```

Response: array of API objects.

```json
[
  {
    "apiId": "nshield",
    "name": "NovaShield",
    "description": "Security engine API",
    "supportedNetworks": ["solana"],
    "zAuthEnabled": false
  }
]
```

### Key fields

| Field | Type | Description |
|---|---|---|
| `apiId` | string | Unique API identifier, used in relay URL |
| `name` | string | Display name |
| `description` | string | What the API does |
| `supportedNetworks` | string[] | Blockchains supported for payment |
| `zAuthEnabled` | boolean | Whether zAuth is required (not supported by this skill) |

## Get API details

```
GET /marketplace/{apiId}
```

Response: API object with endpoints.

```json
{
  "apiId": "nshield",
  "name": "NovaShield",
  "network": "solana",
  "zAuthEnabled": false,
  "endpoints": [
    {
      "path": "/v1/health",
      "method": "GET",
      "summary": "Health check",
      "usdPrice": 0.01,
      "enabled": true
    }
  ]
}
```

### Endpoint fields

| Field | Type | Description |
|---|---|---|
| `path` | string | Endpoint path (appended to relay URL) |
| `method` | string | HTTP method (GET, POST, etc.) |
| `summary` | string | What the endpoint does |
| `usdPrice` | number | Cost per call in USD |
| `enabled` | boolean | Whether the endpoint is active |

## Pre-fetch pricing (x402 details)

```
GET /marketplace/{apiId}/x402?method={METHOD}&path={PATH}
```

Returns x402 payment requirements without triggering a 402 challenge.

## Relay URL pattern

```
{RELAI_API_URL}/relay/{apiId}{endpointPath}
```

Example: `https://api.relai.fi/relay/nshield/v1/health`
