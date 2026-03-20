# Express Payment Execution

When the user has confirmed an **express** verification request, the agent executes the payment end-to-end. The wallet address was collected in Step 6 of the main flow and must be passed into the local script so it can verify the signer before signing.

## 7a. Locate Private Key Source

**If the private key source was already found in Step 5b** of the main flow, reuse that source — do not re-resolve.

The agent's job is to find **where** the private key is stored — never to read the key value itself. Locate the key source using this priority order:

1. **Check `.env` files** — Look for a `PRIVATE_KEY` or `SOLANA_PRIVATE_KEY` variable name in `.env`, `.env.local`, or similar files in the current project directory. If found, note the **file path and variable name** (e.g., `.env` contains `PRIVATE_KEY`). Do NOT read or extract the actual key value.
2. **Check for keypair file** — Look for a Solana keypair JSON file (e.g., `~/.config/solana/id.json` or a path provided by the user). If found, note the **file path**.
3. **Ask the user** — If no key is found via either method, ask the user to either:
   - Set it in a `.env` file (e.g., `PRIVATE_KEY=<base58-key>`) and provide the file path
   - Provide the path to a Solana keypair JSON file

> **NEVER accept a private key directly in chat.** If a user pastes a raw private key in the conversation, warn them that this is insecure (keys persist in chat history and may be logged) and instruct them to use a `.env` file or keypair file path instead. Do not use the pasted key.

> **NEVER read the contents of `.env` files or keypair files that contain private keys.** The agent must only confirm the file exists and which variable name holds the key. The script itself loads secrets at runtime, so the agent never needs to see the actual values. This prevents secret values from appearing in agent conversation context, tool-call logs, or chat history.

When writing `config.json` in Step 7c, include the **location** of secrets — not the values:
- `"envPath": "/absolute/path/to/.env"` — the `.env` file containing secrets (used for both private key and API key)
- `"envKeyName": "PRIVATE_KEY"` — env var name for the private key (omit if using keypairPath)
- OR `"keypairPath": "/absolute/path/to/keypair.json"` — path to a Solana keypair file (omit if using envPath for private key)
- `"apiKeyEnvName": "JUPITER_API_KEY"` — env var name for the API key (defaults to `JUPITER_API_KEY`)

> **Private Key Security Rules:**
> - The private key MUST stay local. It is NEVER sent to any server, API, or external service.
> - The agent NEVER reads, extracts, or handles the private key value. It only passes file paths to the script.
> - The key is used ONLY to sign the transaction on the user's machine. Only the **signed transaction** (not the key) is sent to the Jupiter API.
> - The script runs entirely locally — it reads the key, signs, and sends only the signed output.
> - The user provides `walletAddress` separately. The script MUST derive the signer wallet locally and abort if it does not match that address.
> - Recommend the user use a dedicated wallet with minimal funds, not their main holdings wallet.
> - If the key is in a `.env` file, remind the user to ensure `.env` is in `.gitignore`.

## 7b. Set Up Execution Environment

Create a temporary directory and install dependencies:

```bash
TMPDIR=$(mktemp -d /tmp/jup-verify-XXXXXX)
cd "$TMPDIR" && npm init -y && npm install @solana/web3.js @solana/spl-token bs58 dotenv
```

## 7c. Write the Payment Script

Write a `pay.ts` script and a separate `config.json` file in the temp directory. The config file contains user-provided parameters and **paths to secret files** (never the secret values themselves). The script loads secrets directly from the user's `.env` or keypair file at runtime — the agent never reads, extracts, or handles secret values. For metadata updates, `tokenMetadata` should include the **full merged object** (existing data from `GET /tokenMetadata/getFromRpcAndSearch/{tokenId}` with user updates applied on top) so unchanged fields are preserved. The script must:

1. Read secret **locations** from `config.json` — the agent writes file paths and env var names so the script can load secrets directly. The agent never reads or handles secret values.
2. Load the private key from the user's `.env` file (via `dotenv`) or keypair file, and the API key from the `.env` file
3. Derive the wallet address from the private key using `Keypair.fromSecretKey`, compare it to the user-provided `walletAddress`, and abort on mismatch
4. Call `GET /payments/express/craft-txn?senderAddress={derived-wallet}` with `x-api-key` header to get the unsigned transaction
5. Deserialize as a `VersionedTransaction` (matching the production UI implementation)
6. Verify the transaction contents before signing: decode the compiled instructions, confirm there is exactly one SPL Token transfer instruction (plus optional compute budget instructions), confirm the amount does not exceed 1 JUP, confirm the destination matches the expected receiver ATA, and reject if any unexpected instructions are present
7. Sign locally with `transaction.sign([keypair])` and serialize
8. Call `POST /payments/express/execute` with `x-api-key` header, the signed transaction, and all verification parameters
9. Print `SUCCESS:<signature>` on success or `ERROR:<code>:<message>` on failure

**Current API quirk:** `POST /payments/express/execute` currently validates `twitterHandle` and `description` as required strings, even for metadata-only requests where no verification data is being submitted. If the user did not provide those fields, send `twitterHandle: ""` and `description: ""` instead of inventing values.

The script MUST include these security comments:

```typescript
// SECURITY: Private key is used ONLY for local signing.
// Only the signed transaction is sent to the Jupiter API.
// The private key NEVER leaves this machine.
// SECURITY: The agent NEVER reads secret values. config.json contains only file paths
// and env var names — the script loads secrets directly from the user's files at runtime.
```

**Template script:**

```typescript
// SECURITY: Private key is used ONLY for local signing.
// Only the signed transaction is sent to the Jupiter API.
// The private key NEVER leaves this machine.
// SECURITY: The agent NEVER reads secret values. config.json contains only file paths
// and env var names — the script loads secrets directly from the user's files at runtime.

import { Keypair, PublicKey, VersionedTransaction } from "@solana/web3.js";
import {
  TOKEN_PROGRAM_ID,
  getAssociatedTokenAddressSync,
} from "@solana/spl-token";
import bs58 from "bs58";
import fs from "fs";
import path from "path";
import dotenv from "dotenv";

const BASE_URL = "https://token-verification-dev-api.jup.ag";
const JUP_MINT = new PublicKey("JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN");
const COMPUTE_BUDGET_PROGRAM_ID = new PublicKey(
  "ComputeBudget111111111111111111111111111111"
);

// Read parameters and secret LOCATIONS from config file.
// config.json contains file paths and env var names — NEVER secret values.
// The script loads secrets directly from the user's .env / keypair files.
// tokenMetadata contains the full merged object (existing data + user updates)
// so unchanged fields are preserved server-side.
const config = JSON.parse(fs.readFileSync("./config.json", "utf8"));
const TOKEN_ID: string = config.tokenId;
const EXPECTED_WALLET_ADDRESS: string = config.walletAddress;
const TWITTER_HANDLE: string = config.twitterHandle ?? "";
const SENDER_TWITTER: string | undefined = config.senderTwitterHandle ?? undefined;
const DESCRIPTION: string = config.description ?? "";
const TOKEN_METADATA: Record<string, unknown> | undefined = config.tokenMetadata ?? undefined;

// Load secrets from the user's .env file if an envPath is provided
if (config.envPath) {
  const envFilePath = path.resolve(config.envPath);
  if (!fs.existsSync(envFilePath)) {
    console.error(`ERROR:ENV_NOT_FOUND:File not found: ${envFilePath}`);
    process.exit(1);
  }
  dotenv.config({ path: envFilePath });
}

// Resolve private key — loaded by the SCRIPT from the user's files, never by the agent
const KEYPAIR_PATH: string | undefined = config.keypairPath;
const ENV_KEY_NAME: string = config.envKeyName ?? "PRIVATE_KEY";
const PRIVATE_KEY = process.env[ENV_KEY_NAME];
if (!PRIVATE_KEY && !KEYPAIR_PATH) {
  console.error(
    `ERROR:NO_KEY:No private key found. Expected env var "${ENV_KEY_NAME}" in ${config.envPath ?? ".env"} or keypairPath in config.json`
  );
  process.exit(1);
}

// Resolve API key — loaded by the SCRIPT from the user's .env, never by the agent
const API_KEY_NAME: string = config.apiKeyEnvName ?? "JUPITER_API_KEY";
const API_KEY = process.env[API_KEY_NAME];
if (!API_KEY) {
  console.error(
    `ERROR:NO_API_KEY:No API key found. Expected env var "${API_KEY_NAME}" in ${config.envPath ?? ".env"}`
  );
  process.exit(1);
}

async function main() {
  // Derive wallet from private key
  let keypair: Keypair;
  if (PRIVATE_KEY) {
    keypair = Keypair.fromSecretKey(bs58.decode(PRIVATE_KEY));
  } else {
    const keypairFilePath = path.resolve(KEYPAIR_PATH!);
    if (!fs.existsSync(keypairFilePath)) {
      console.error(`ERROR:KEYPAIR_NOT_FOUND:File not found: ${keypairFilePath}`);
      process.exit(1);
    }
    const secret = JSON.parse(fs.readFileSync(keypairFilePath, "utf8"));
    keypair = Keypair.fromSecretKey(new Uint8Array(secret));
  }
  const senderAddress = keypair.publicKey.toBase58();
  console.log(`Wallet: ${senderAddress}`);

  let expectedWallet: PublicKey;
  try {
    expectedWallet = new PublicKey(EXPECTED_WALLET_ADDRESS);
  } catch {
    console.error(`ERROR:INVALID_WALLET:Invalid wallet address: ${EXPECTED_WALLET_ADDRESS}`);
    process.exit(1);
  }
  if (!keypair.publicKey.equals(expectedWallet)) {
    console.error(
      `ERROR:WALLET_MISMATCH:Derived wallet ${senderAddress} does not match provided wallet ${expectedWallet.toBase58()}`
    );
    process.exit(1);
  }

  // Step 1: Craft the unsigned payment transaction
  const craftRes = await fetch(
    `${BASE_URL}/payments/express/craft-txn?senderAddress=${senderAddress}`,
    {
      headers: { "x-api-key": API_KEY },
    }
  );

  if (!craftRes.ok) {
    const err = await craftRes.text();
    console.error(`ERROR:CRAFT_FAILED:${craftRes.status} ${err}`);
    process.exit(1);
  }

  const craftData = await craftRes.json();
  const { transaction: unsignedTxBase64, requestId } = craftData;

  if (!unsignedTxBase64 || !requestId) {
    console.error("ERROR:CRAFT_FAILED:Missing transaction or requestId in response");
    process.exit(1);
  }

  console.log(`Request ID: ${requestId}`);
  console.log(`Payment: ${craftData.amount} base units of JUP`);

  // Step 2: Deserialize as VersionedTransaction (matches production UI implementation)
  const txBuffer = Buffer.from(unsignedTxBase64, "base64");
  const transaction = VersionedTransaction.deserialize(txBuffer);

  // Step 3: Verify transaction contents before signing
  // SECURITY: Never blindly sign a server-provided transaction
  const accountKeys = transaction.message.staticAccountKeys;
  const compiledIxs = transaction.message.compiledInstructions;

  // Separate compute budget instructions from main instructions
  const mainIxs = compiledIxs.filter(
    (ix) => !accountKeys[ix.programIdIndex].equals(COMPUTE_BUDGET_PROGRAM_ID)
  );

  // Expect exactly one main instruction (SPL Token Transfer)
  if (mainIxs.length !== 1) {
    console.error(
      `ERROR:TX_VERIFY_FAILED:Expected 1 main instruction, found ${mainIxs.length}`
    );
    process.exit(1);
  }

  const ix = mainIxs[0];
  const programId = accountKeys[ix.programIdIndex];

  // Verify it is an SPL Token program instruction
  if (!programId.equals(TOKEN_PROGRAM_ID)) {
    console.error(
      `ERROR:TX_VERIFY_FAILED:Unexpected program ${programId.toBase58()}, expected SPL Token ${TOKEN_PROGRAM_ID.toBase58()}`
    );
    process.exit(1);
  }

  // Decode SPL Token transfer fields from instruction data
  // Transfer (opcode 3): [u8 opcode] [u64 amount], accounts = [source, destination, owner]
  // TransferChecked (opcode 12): [u8 opcode] [u64 amount] [u8 decimals], accounts = [source, mint, destination, owner]
  const data = Buffer.from(ix.data);
  const opcode = data[0];
  let transferAmount: bigint;
  if ((opcode === 3 || opcode === 12) && data.length >= 9) {
    transferAmount = data.readBigUInt64LE(1);
  } else {
    console.error(
      `ERROR:TX_VERIFY_FAILED:Not a Transfer instruction (opcode=${opcode})`
    );
    process.exit(1);
  }

  // Verify amount matches API response and does not exceed 1 JUP
  const expectedAmount = BigInt(craftData.amount);
  const MAX_AMOUNT = BigInt(1_000_000); // 1 JUP = 1,000,000 base units (6 decimals)
  if (transferAmount !== expectedAmount) {
    console.error(
      `ERROR:TX_VERIFY_FAILED:Amount mismatch — transaction has ${transferAmount}, API said ${expectedAmount}`
    );
    process.exit(1);
  }
  if (transferAmount > MAX_AMOUNT) {
    console.error(
      `ERROR:TX_VERIFY_FAILED:Amount ${transferAmount} exceeds maximum ${MAX_AMOUNT}`
    );
    process.exit(1);
  }

  // Verify destination matches expected receiver ATA
  // TransferChecked (opcode 12): accounts[2] is destination
  // Transfer (opcode 3): accounts[1] is destination
  const destAccountIndex = opcode === 12 ? ix.accountKeyIndexes[2] : ix.accountKeyIndexes[1];
  const destinationAccount = accountKeys[destAccountIndex];
  const receiverWallet = new PublicKey(craftData.receiverAddress);
  const expectedReceiverATA = getAssociatedTokenAddressSync(
    JUP_MINT,
    receiverWallet
  );
  if (!destinationAccount.equals(expectedReceiverATA)) {
    console.error(
      `ERROR:TX_VERIFY_FAILED:Destination ${destinationAccount.toBase58()} does not match expected receiver ATA ${expectedReceiverATA.toBase58()}`
    );
    process.exit(1);
  }

  console.log("Transaction verified: single SPL token transfer of 1 JUP ✓");

  // Step 4: Sign the transaction locally
  // SECURITY: Only the signed transaction leaves this machine, NOT the private key
  transaction.sign([keypair]);
  const signedTxBase64 = Buffer.from(transaction.serialize()).toString("base64");

  // Step 5: Execute — server co-signs and broadcasts
  // `twitterHandle` and `description` are always sent because the current execute schema requires strings.
  // tokenMetadata contains the full merged object so unchanged fields are preserved.
  const executeBody: Record<string, unknown> = {
    transaction: signedTxBase64,
    requestId,
    senderAddress,
    tokenId: TOKEN_ID,
    twitterHandle: TWITTER_HANDLE,
    description: DESCRIPTION,
  };
  if (SENDER_TWITTER) executeBody.senderTwitterHandle = SENDER_TWITTER;
  if (TOKEN_METADATA) executeBody.tokenMetadata = TOKEN_METADATA;

  const executeRes = await fetch(`${BASE_URL}/payments/express/execute`, {
    method: "POST",
    headers: { "Content-Type": "application/json", "x-api-key": API_KEY },
    body: JSON.stringify(executeBody),
  });

  const executeData = await executeRes.json();

  if (executeData.status === "Success") {
    console.log(`SUCCESS:${executeData.signature}`);
    console.log(`Verification created: ${executeData.verificationCreated}`);
    console.log(`Metadata created: ${executeData.metadataCreated}`);
  } else {
    console.error(`ERROR:EXECUTE_FAILED:${JSON.stringify(executeData)}`);
    process.exit(1);
  }
}

main().catch((err) => {
  console.error(`ERROR:EXCEPTION:${err.message}`);
  process.exit(1);
});
```

Write a `config.json` file in the same temp directory with the collected parameters and **paths to secret files** (never the secret values). The script loads secrets directly from the user's files at runtime. For `tokenMetadata`, include the **full merged object** (existing data from `GET /tokenMetadata/getFromRpcAndSearch/{tokenId}` with user updates applied on top) so unchanged fields are preserved:

```json
{
  "envPath": "<absolute path to user's .env file — OMIT if using keypairPath without .env>",
  "envKeyName": "<env var name for private key, e.g. 'PRIVATE_KEY' or 'SOLANA_PRIVATE_KEY' — OMIT if using keypairPath>",
  "keypairPath": "<absolute path to keypair.json — OMIT if using envPath>",
  "apiKeyEnvName": "<env var name for API key, e.g. 'JUPITER_API_KEY' — defaults to 'JUPITER_API_KEY'>",
  "walletAddress": "<user-provided wallet address that must match the signing key>",
  "tokenId": "<collected token mint>",
  "twitterHandle": "<collected twitter URL — OMIT THIS KEY if user skipped>",
  "senderTwitterHandle": "<collected sender twitter URL — OMIT THIS KEY if user skipped>",
  "description": "<collected description — OMIT THIS KEY if user skipped>",
  "tokenMetadata": {
    "tokenId": "<collected token mint>",
    "name": "<existing or updated value>",
    "symbol": "<existing or updated value>",
    "icon": "<existing or updated value>",
    "tokenDescription": "<existing or updated value>",
    "website": "<existing or updated value>",
    "twitter": "<existing or updated value>",
    "...all other fields from existing data with user updates merged in"
  }
}
```

**Example:** If the agent found `PRIVATE_KEY` and `JUPITER_API_KEY` in `/Users/alice/project/.env`:
```json
{
  "envPath": "/Users/alice/project/.env",
  "envKeyName": "PRIVATE_KEY",
  "apiKeyEnvName": "JUPITER_API_KEY",
  "walletAddress": "8xDr...",
  "tokenId": "So11111111111111111111111111111111111111112",
  "twitterHandle": "https://x.com/jupiterexchange",
  "description": "Official wrapped SOL"
}
```

**Example:** If the agent found a keypair file and API key in `.env`:
```json
{
  "envPath": "/Users/alice/project/.env",
  "keypairPath": "/Users/alice/.config/solana/id.json",
  "apiKeyEnvName": "JUPITER_API_KEY",
  "walletAddress": "8xDr...",
  "tokenId": "So11111111111111111111111111111111111111112",
  "twitterHandle": "https://x.com/jupiterexchange",
  "description": "Official wrapped SOL"
}
```

Use `JSON.stringify()` to write this file, ensuring all user-provided values are properly escaped. NEVER embed user input directly into TypeScript source code — this prevents code injection attacks.

> **SECURITY:** `config.json` contains NO secret values — only file paths and env var names. The script loads secrets directly from the user's own files at runtime. The agent must NEVER read the contents of `.env` files or keypair files containing private keys.

## 7d. Execute the Script

Run the script. All secrets are read from `config.json` — no secrets in shell arguments:

```bash
cd "$TMPDIR" && npx tsx pay.ts
```

Fallback if `tsx` is not available:

```bash
cd "$TMPDIR" && npx ts-node pay.ts
```

> **SECURITY:** Never pass secrets as inline shell arguments (e.g., `PRIVATE_KEY="..." npx tsx pay.ts`). This exposes secrets in `ps` output, shell history, and agent tool-call logs. The script loads secrets directly from the user's `.env` and keypair files — `config.json` only contains file paths and env var names, never secret values.

Parse the output:
- `SUCCESS:<signature>` → payment succeeded
- `ERROR:<code>:<message>` → payment failed, see error table in 7e

## 7e. Report Result and Clean Up

**On success:**

> Express verification payment submitted!
>
> | Detail          | Value                                                              |
> | --------------- | ------------------------------------------------------------------ |
> | **Signature**   | `{signature}`                                                      |
> | **Explorer**    | `https://solana.fm/tx/{signature}`                                 |
> | **Token**       | `{tokenId}`                                                        |
> | **Status**      | Express verification pending review                                |

**On failure**, map error patterns to likely causes:

| Error Pattern       | Likely Cause                                         | Suggestion                                              |
| ------------------- | ---------------------------------------------------- | ------------------------------------------------------- |
| `NO_KEY`            | Private key not found in `.env` or keypair file      | Ensure `PRIVATE_KEY` is set in `.env` or provide a valid `keypairPath` |
| `NO_API_KEY`        | API key not found in `.env`                          | Ensure `JUPITER_API_KEY` is set in `.env`                |
| `INVALID_WALLET`    | The provided wallet address is not a valid Solana public key | Check the `walletAddress` value in `config.json`         |
| `WALLET_MISMATCH`   | The provided wallet does not match the signing key   | Verify the wallet address and private key refer to the same wallet |
| `ENV_NOT_FOUND`     | The `.env` file path in config.json does not exist   | Check the `envPath` in config.json points to a valid file |
| `KEYPAIR_NOT_FOUND` | The keypair file path does not exist                 | Check the `keypairPath` in config.json points to a valid file |
| `CRAFT_FAILED:400`  | Invalid wallet or insufficient JUP balance           | Check wallet has ≥1 JUP + small SOL for fees             |
| `CRAFT_FAILED:401`  | Invalid or missing API key                           | Check `JUPITER_API_KEY` is set correctly                  |
| `CRAFT_FAILED:429`  | Rate limited                                         | Wait and retry later, or use a different API key          |
| `TX_VERIFY_FAILED`  | Transaction contents don't match expectations        | Possible API change or malicious response — do not sign   |
| `EXECUTE_FAILED`    | Transaction expired, network error, or co-sign issue | Re-run the script (crafts a fresh transaction)           |
| `EXCEPTION`         | Script crash (network, dependency issue)             | Check Node.js v18+, check network connectivity           |

**Always clean up** — remove the temp directory to ensure no private key material remains on disk:

```bash
rm -rf "$TMPDIR"
```
