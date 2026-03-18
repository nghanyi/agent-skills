# Complete Working Example

> Copy-paste-ready TypeScript showing the full express verification flow.
>
> This example wraps the payment script from [Payment Execution](payment-execution.md) with status checking and keypair loading. The payment script template in `payment-execution.md` is the **single source of truth** for transaction crafting, verification, signing, and execution — refer to it for implementation details.

```typescript
// SECURITY: Private key is used ONLY for local signing.
// Only the signed transaction is sent to the Jupiter API.
// The private key NEVER leaves this machine.

import { Keypair, PublicKey } from "@solana/web3.js";
import fs from "fs";
import path from "path";

const BASE_URL = "https://token-verification-dev-api.jup.ag";
const KEYPAIR_PATH = "/path/to/your/keypair.json";

// Example values — replace with your actual token details
const TOKEN_ID = "YourTokenMintAddress111111111111111111111111";
const TWITTER_HANDLE = "https://x.com/your_token";
const SENDER_TWITTER = "https://x.com/your_wallet";
const DESCRIPTION = "My awesome Solana token";

function loadKeypair(keypairPath: string): Keypair {
  const fullPath = path.resolve(keypairPath);
  const secret = JSON.parse(fs.readFileSync(fullPath, "utf8"));
  return Keypair.fromSecretKey(new Uint8Array(secret));
}

async function main() {
  // SECURITY: Wallet address is derived from the local keypair — no external lookup needed
  const wallet = loadKeypair(KEYPAIR_PATH);
  const senderAddress = wallet.publicKey.toBase58();

  // ── Step 1: Check existing verification status ──
  const statusRes = await fetch(`${BASE_URL}/verifications/token/${TOKEN_ID}`);
  const statusData = await statusRes.json();

  if (statusData.data?.status === "verified") {
    console.log("Token is already verified.");
    return;
  }

  if (statusData.data?.status === "pending") {
    console.log("Verification already pending — skipping resubmission.");
    return;
  }

  // ── Step 2: Run the payment script ──
  // For the full payment flow (craft → verify → sign → execute),
  // see the template script in payment-execution.md (steps 7b–7e).
  //
  // In short:
  //   1. Write a config.json with { tokenId, twitterHandle, senderTwitterHandle, description }
  //   2. Write the pay.ts script from payment-execution.md
  //   3. Run: KEYPAIR_PATH="/path/to/keypair.json" npx tsx pay.ts
  //
  // The script handles: crafting the unsigned transaction, verifying its
  // contents (SPL Token program, amount ≤ 1 JUP, correct destination ATA),
  // signing locally, and calling the execute endpoint.

  console.log("Token is not yet verified. Ready to submit express verification.");
  console.log(`Wallet: ${senderAddress}`);
  console.log(`Token:  ${TOKEN_ID}`);
  console.log("Run the payment script from payment-execution.md to proceed.");
}

main().catch(console.error);
```

## What the payment script does

For reference, the payment script in [Payment Execution](payment-execution.md) performs these steps:

1. **Craft** — calls `GET /payments/transfer/craft-txn` to get an unsigned transaction
2. **Verify** — decodes the transaction and checks:
   - Exactly one SPL Token transfer instruction (plus optional compute budget)
   - Amount matches the API response and does not exceed 1 JUP (1,000,000 base units)
   - Destination matches the expected receiver ATA
3. **Sign** — signs locally with `VersionedTransaction.sign([keypair])`
4. **Execute** — calls `POST /payments/transfer/execute` with the signed transaction

See [Payment Execution](payment-execution.md) for the complete implementation.
