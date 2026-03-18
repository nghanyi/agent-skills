# Complete Working Example

> Copy-paste-ready TypeScript showing the full express verification flow. Install: `npm install @solana/web3.js @solana/spl-token`
>
> In agent-generated scripts, token details are read from a `config.json` file (never interpolated into source code). This example uses hardcoded constants for readability — replace with your actual values.

```typescript
// SECURITY: Private key is used ONLY for local signing.
// Only the signed transaction is sent to the Jupiter API.
// The private key NEVER leaves this machine.

import { Keypair, PublicKey, Transaction } from "@solana/web3.js";
import {
  TOKEN_PROGRAM_ID,
  getAssociatedTokenAddressSync,
} from "@solana/spl-token";
import fs from "fs";
import path from "path";

const BASE_URL = "https://token-verification-dev-api.jup.ag";
const JUP_MINT = new PublicKey("JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN");
const COMPUTE_BUDGET_PROGRAM_ID = new PublicKey(
  "ComputeBudget111111111111111111111111111111"
);
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

  // ── Step 3: Craft payment transaction ──
  // (Step 2 — POST /verifications — is not needed for express; execute auto-creates it)
  const craftRes = await fetch(
    `${BASE_URL}/payments/transfer/craft-txn?senderAddress=${senderAddress}`
  );

  if (!craftRes.ok) {
    throw new Error(`Failed to craft transaction: ${craftRes.statusText}`);
  }

  const craftData = await craftRes.json();
  const { transaction: unsignedTxBase64, requestId } = craftData;

  console.log(`Payment: ${craftData.amount} base units of JUP`);
  console.log(`Request ID: ${requestId}`);

  // ── Verify transaction contents before signing ──
  // SECURITY: Never blindly sign a server-provided transaction
  const txBuffer = Buffer.from(unsignedTxBase64, "base64");
  const transaction = Transaction.from(txBuffer);

  const mainIxs = transaction.instructions.filter(
    (ix) => !ix.programId.equals(COMPUTE_BUDGET_PROGRAM_ID)
  );

  if (mainIxs.length !== 1) {
    throw new Error(
      `Expected 1 main instruction, found ${mainIxs.length}`
    );
  }

  const ix = mainIxs[0];

  if (!ix.programId.equals(TOKEN_PROGRAM_ID)) {
    throw new Error(
      `Unexpected program ${ix.programId.toBase58()}, expected SPL Token`
    );
  }

  const opcode = ix.data[0];
  let transferAmount: bigint;
  if ((opcode === 3 || opcode === 12) && ix.data.length >= 9) {
    transferAmount = ix.data.readBigUInt64LE(1);
  } else {
    throw new Error(`Not a Transfer instruction (opcode=${opcode})`);
  }

  const expectedAmount = BigInt(craftData.amount);
  const MAX_AMOUNT = BigInt(1_000_000);
  if (transferAmount !== expectedAmount) {
    throw new Error(
      `Amount mismatch — transaction has ${transferAmount}, API said ${expectedAmount}`
    );
  }
  if (transferAmount > MAX_AMOUNT) {
    throw new Error(`Amount ${transferAmount} exceeds maximum ${MAX_AMOUNT}`);
  }

  const receiverWallet = new PublicKey(craftData.receiverAddress);
  const expectedReceiverATA = getAssociatedTokenAddressSync(
    JUP_MINT,
    receiverWallet
  );
  const destinationAccount = ix.keys[1].pubkey;
  if (!destinationAccount.equals(expectedReceiverATA)) {
    throw new Error(
      `Destination ${destinationAccount.toBase58()} does not match expected receiver ATA ${expectedReceiverATA.toBase58()}`
    );
  }

  console.log("Transaction verified: single SPL token transfer of 1 JUP ✓");

  // ── Sign the transaction locally ──
  // SECURITY: Only the signed transaction is sent to the server, NOT the private key
  transaction.partialSign(wallet);

  const signedTxBase64 = Buffer.from(
    transaction.serialize({ requireAllSignatures: false })
  ).toString("base64");

  // ── Step 4: Execute payment (server co-signs and broadcasts) ──
  const executeRes = await fetch(`${BASE_URL}/payments/transfer/execute`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      transaction: signedTxBase64,
      requestId,
      senderAddress,
      tokenId: TOKEN_ID,
      twitterHandle: TWITTER_HANDLE,
      senderTwitterHandle: SENDER_TWITTER,
      description: DESCRIPTION,
    }),
  });

  const executeData = await executeRes.json();

  if (executeData.status === "Success") {
    console.log(`Express verification submitted!`);
    console.log(`Transaction signature: ${executeData.signature}`);
  } else {
    console.error("Execution failed:", executeData.error);
  }
}

main().catch(console.error);
```
