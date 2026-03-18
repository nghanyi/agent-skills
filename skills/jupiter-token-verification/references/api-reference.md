# API Reference — Jupiter Token Verification

> **Base URL**: `https://token-verification-dev-api.jup.ag`

## Authentication

Some endpoints require an API key passed via the `x-api-key` header:

| Endpoint | Auth Required |
| -------- | ------------- |
| `GET /combined/express/check-eligibility` | None |
| `GET /combined/basic/check-eligibility` | None |
| `POST /basic/submit` | API key |
| `GET /payments/express/craft-txn` | API key |
| `POST /payments/express/execute` | API key |

**Rate limit:** 2 requests/day per API key for submission endpoints.

```
x-api-key: your-api-key-here
```

---

## Check Express Eligibility

**`GET /combined/express/check-eligibility`**

```
GET https://token-verification-dev-api.jup.ag/combined/express/check-eligibility?tokenId={tokenId}
```

| Param     | Type   | Required | Notes                    |
| --------- | ------ | -------- | ------------------------ |
| `tokenId` | string | **Yes**  | Solana token mint address |

**Response:**

```json
{
  "canVerify": true,
  "canMetadata": true,
  "verificationError": null,
  "metadataError": null
}
```

- `canVerify: true` → token is eligible for express verification
- `canVerify: false` → check `verificationError` for the reason
- `canMetadata: true` → token metadata can be submitted alongside verification
- `canMetadata: false` → check `metadataError` for the reason

---

## Check Basic Eligibility

**`GET /combined/basic/check-eligibility`**

```
GET https://token-verification-dev-api.jup.ag/combined/basic/check-eligibility?tokenId={tokenId}
```

| Param     | Type   | Required | Notes                    |
| --------- | ------ | -------- | ------------------------ |
| `tokenId` | string | **Yes**  | Solana token mint address |

**Response:**

```json
{
  "canVerify": true,
  "canMetadata": false,
  "verificationError": null,
  "metadataError": "Token metadata already exists"
}
```

Same response schema as express eligibility.

---

## Submit Basic Verification

**`POST /basic/submit`**

```
POST https://token-verification-dev-api.jup.ag/basic/submit
Content-Type: application/json
x-api-key: your-api-key-here
```

**Request body:**

```json
{
  "tokenId": "So11111111111111111111111111111111111111112",
  "walletAddress": "8xDr...",
  "submitVerification": true,
  "twitterHandle": "https://x.com/jupiterexchange",
  "senderTwitterHandle": "https://x.com/sender_handle",
  "description": "Official wrapped SOL token",
  "tokenMetadata": {
    "name": "Wrapped SOL",
    "symbol": "SOL"
  }
}
```

| Field                 | Type    | Required | Notes                                                              |
| --------------------- | ------- | -------- | ------------------------------------------------------------------ |
| `tokenId`             | string  | **Yes**  | Solana token mint address                                          |
| `walletAddress`       | string  | **Yes**  | Requester's wallet address (valid Solana address)                  |
| `submitVerification`  | boolean | No       | Set to `true` to submit verification request                       |
| `twitterHandle`       | string  | No       | Token's Twitter — must be a valid `x.com` or `twitter.com` URL     |
| `senderTwitterHandle` | string  | No       | Requester's Twitter — must be a valid `x.com` or `twitter.com` URL |
| `description`         | string  | No       | Description of the token                                           |
| `tokenMetadata`       | object  | No       | Optional token metadata to set alongside verification              |

**Response:**

```json
{
  "success": true,
  "data": {
    "verificationCreated": true,
    "metadataCreated": false
  }
}
```

> For **basic** verification, you're done here. The express payment flow is only needed for **express** verification (paid with JUP).

---

## Craft Payment Transaction (Express Only)

**`GET /payments/express/craft-txn`**

```
GET https://token-verification-dev-api.jup.ag/payments/express/craft-txn?senderAddress={walletAddress}
x-api-key: your-api-key-here
```

| Param           | Type   | Required | Notes                        |
| --------------- | ------ | -------- | ---------------------------- |
| `senderAddress` | string | **Yes**  | Wallet that will pay 1 JUP |

**Response:**

```json
{
  "receiverAddress": "VRFD...",
  "mint": "JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN",
  "amount": "1000000",
  "tokenDecimals": 6,
  "feeLamports": 5000,
  "feeMint": "So11111111111111111111111111111111111111112",
  "feeTokenDecimals": 9,
  "feeAmount": 5000,
  "transaction": "<base64-encoded-unsigned-transaction>",
  "requestId": "req_abc123",
  "totalTime": "150ms",
  "expireAt": "2025-06-01T12:05:00Z",
  "code": 0,
  "gasless": false
}
```

The `transaction` field is a **base64-encoded unsigned transaction**. The user must sign it before proceeding to the execute step.

---

## Sign and Execute Payment (Express Only)

The user signs the transaction from the craft step client-side, then submits it to the execute endpoint. The server co-signs with its verification wallet before broadcasting.

**`POST /payments/express/execute`**

```
POST https://token-verification-dev-api.jup.ag/payments/express/execute
Content-Type: application/json
x-api-key: your-api-key-here
```

**Request body:**

```json
{
  "transaction": "<base64-signed-transaction>",
  "requestId": "req_abc123",
  "senderAddress": "8xDr...",
  "tokenId": "So11111111111111111111111111111111111111112",
  "twitterHandle": "https://x.com/jupiterexchange",
  "senderTwitterHandle": "https://x.com/sender_handle",
  "description": "Official wrapped SOL token",
  "tokenMetadata": {
    "name": "Jupiter Exchange",
    "symbol": "JUP"
  }
}
```

| Field                 | Type   | Required | Notes                                      |
| --------------------- | ------ | -------- | ------------------------------------------ |
| `transaction`         | string | **Yes**  | Base64 user-signed transaction from craft   |
| `requestId`           | string | **Yes**  | From `craft-txn` response                   |
| `senderAddress`       | string | **Yes**  | Wallet that signed the transaction         |
| `tokenId`             | string | **Yes**  | Token mint being verified                  |
| `twitterHandle`       | string | **Yes**  | Token's Twitter URL                        |
| `senderTwitterHandle` | string | No       | Requester's Twitter URL                    |
| `description`         | string | **Yes**  | Description of the token                   |
| `tokenMetadata`       | object | No       | Optional token metadata to set alongside verification |

**Response:**

```json
{
  "status": "Success",
  "signature": "5tG8...",
  "verificationCreated": true,
  "metadataCreated": false,
  "totalTime": 2500
}
```

On success, the server automatically creates a **express** verification request for the token. The `verificationCreated` and `metadataCreated` fields indicate what was created.

---

## Data Types

### Verification Tiers

| Tier      | Cost    | Description                                                           |
| --------- | ------- | --------------------------------------------------------------------- |
| `basic`   | Free    | Standard verification — submit via `POST /basic/submit`               |
| `express` | 1 JUP | Paid verification — requires payment via `craft-txn` + `execute` flow. API value: `"premium"` |

### Verification Statuses

| Status     | Meaning                                            |
| ---------- | -------------------------------------------------- |
| `pending`  | Awaiting review                                    |
| `verified` | Approved                                           |
| `rejected` | Denied (check `rejectCategory` and `rejectReason`) |

### Reject Categories

`below_thresholds`, `insufficient_trading_activity`, `ticker_conflict`, `duplicate_token`, `others`, `NA`

### Twitter Handle Format

Twitter handles must be full URLs from `x.com` or `twitter.com`:

- `https://x.com/jupiterexchange` — valid
- `https://twitter.com/jupiterexchange` — valid
- `@jupiterexchange` — **invalid** (will be rejected)
- `jupiterexchange` — **invalid** (will be rejected)

Usernames must match `^[a-zA-Z0-9_]{1,15}$` (1–15 characters, alphanumeric + underscore).
