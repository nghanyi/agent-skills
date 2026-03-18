# API Reference — Jupiter Token Verification

> **Base URL**: `https://token-verification-dev-api.jup.ag`

## Check Existing Status

**`GET /verifications/token/:tokenId`**

```
GET https://token-verification-dev-api.jup.ag/verifications/token/{tokenId}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": 123,
    "tokenId": "So11111111111111111111111111111111111111112",
    "walletAddress": "8xDr...",
    "twitterHandle": "jupiterexchange",
    "senderTwitterHandle": "sender_handle",
    "verifiedAt": "2025-01-15T10:30:00Z",
    "verificationTier": "premium",
    "status": "verified",
    "smartFollowers": 1500,
    "evaluationCount": 2,
    "lastEvaluationAt": "2025-01-15T10:30:00Z",
    "createdAt": "2025-01-10T08:00:00Z",
    "updatedAt": "2025-01-15T10:30:00Z",
    "description": "Official wrapped SOL token",
    "rejectCategory": null,
    "rejectReason": null
  }
}
```

If `data` is `null`, no verification exists yet. If `status` is `"verified"`, the token is already verified — do not resubmit.

---

## Submit Verification Request

**`POST /verifications`**

```
POST https://token-verification-dev-api.jup.ag/verifications
Content-Type: application/json
```

**Request body:**

```json
{
  "tokenId": "So11111111111111111111111111111111111111112",
  "walletAddress": "8xDr...",
  "twitterHandle": "https://x.com/jupiterexchange",
  "senderTwitterHandle": "https://x.com/sender_handle",
  "verificationTier": "basic",
  "description": "Official wrapped SOL token"
}
```

| Field                 | Type   | Required | Notes                                                              |
| --------------------- | ------ | -------- | ------------------------------------------------------------------ |
| `tokenId`             | string | **Yes**  | Solana token mint address                                          |
| `walletAddress`       | string | No       | Requester's wallet address (valid Solana address)                  |
| `twitterHandle`       | string | No       | Token's Twitter — must be a valid `x.com` or `twitter.com` URL     |
| `senderTwitterHandle` | string | No       | Requester's Twitter — must be a valid `x.com` or `twitter.com` URL |
| `verificationTier`    | string | No       | `"basic"` (default) or `"premium"` (express)                       |
| `description`         | string | No       | Description of the token                                           |

**Response:**

```json
{
  "success": true,
  "data": {
    "id": 456,
    "tokenId": "So11111111111111111111111111111111111111112",
    "status": "pending",
    "verificationTier": "basic",
    "createdAt": "2025-06-01T12:00:00Z",
    "updatedAt": "2025-06-01T12:00:00Z"
  }
}
```

> For **basic** verification, you're done here. Steps 3–4 are only needed for **express** verification (paid with JUP).

---

## Craft Payment Transaction (Express Only)

**`GET /payments/transfer/craft-txn`**

```
GET https://token-verification-dev-api.jup.ag/payments/transfer/craft-txn?senderAddress={walletAddress}
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

**`POST /payments/transfer/execute`**

```
POST https://token-verification-dev-api.jup.ag/payments/transfer/execute
Content-Type: application/json
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
  "description": "Official wrapped SOL token"
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

**Response:**

```json
{
  "status": "Success",
  "signature": "5tG8...",
  "totalTime": 2500
}
```

On success, the server automatically creates a **express** verification request for the token.

---

## Data Types

### Verification Tiers

| Tier      | Cost    | Description                                                           |
| --------- | ------- | --------------------------------------------------------------------- |
| `basic`   | Free    | Standard verification — submit via `POST /verifications`              |
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
