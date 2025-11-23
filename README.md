# Self-Custodial Subscription Manager

A decentralized subscription management platform that enables users to control recurring payments directly from their own wallet. Users can track, approve, pause, and fund subscriptions like Netflix, YouTube, Claude, Midjourney, and more, without giving control of their payment method to any centralized service. Payments are executed through programmable x402 flows or via a crypto-card-funded agent to handle Web2 billing.

---

## ✨ Key Features

- Full self-custody: users maintain complete control of funds
- Unified dashboard for managing all subscriptions
- AI-powered agent to automate payment approvals and execution
- Crypto-to-card settlement layer for legacy services
- Price-change protection with spending caps
- Push notifications for approvals and reminders
- Manual subscription entry for unsupported platforms

---

## 🧠 High-Level Workflow

1. User connects a Web3 wallet.
2. User adds subscriptions (auto-detected or manually entered).
3. User funds a Subscription Vault with USDC or stablecoins.
4. When a bill is due:
   - System sends approval request
   - If approved, the agent loads exact funds to a crypto card and pays the provider
   - Or uses x402 to complete an on-chain settlement flow
5. Dashboard updates balance, renewal date, and spend insights.

---

## 🧱 Architecture

```txt
User Wallet (Self-Custody)
    |
    | connect / sign / approve
    v
Subscription Vault Smart Contract
    |
    | transfer on approval
    v
Payment Agent (x402-enabled)
    |
    | load "exact amount"
    v
Crypto Card (e.g., Ready)
    |
    | standard payment rails (Visa/Mastercard)
    v
Service Provider (Netflix, YouTube, Claude, etc.)
```

---

## 🔬 x402 Local Testing Guide

This project includes a lightweight mock x402 endpoint and client-side helpers to test the x402 payment flow locally. Follow these steps to validate the full client flow (client requests -> 402 -> signature -> X-PAYMENT header retry -> server accepts).

Prerequisites
- Node.js (v16+ recommended)
- A dev build of this repo running with Next.js (app router)
- A Privy app ID and environment variable set: `NEXT_PUBLIC_PRIVY_APP_ID`
  - For development you can create a test Privy app or use your existing test credentials.

Files added for local testing
- `app/api/x402/test/route.ts` — mock x402 resource endpoint that responds with 402 when no `X-PAYMENT` header is present and returns mock premium content when the header is present.
- `hooks/useMakePaymentFn.tsx` — client hook that builds a `makePayment` function using Privy’s `wrapFetchWithPayment`.
- `lib/agent/AgentPaymentService.ts` — client-side agent class that accepts `makePaymentFn` and coordinates payment and local transaction recording.
- `context/AgentContext.tsx` — provider that creates the agent and exposes it to components.

Quick local test steps
1. Install and build:
   - Ensure dependencies are installed (pnpm/yarn/npm).
   - Start the dev server in the project root:
     - npm run dev (or pnpm dev / yarn dev)
2. Ensure Privy is configured:
   - Export your Privy app ID: `export NEXT_PUBLIC_PRIVY_APP_ID=your_privy_app_id`
   - The app mounts `PrivyProvider` in `app/client.tsx` so the hooks will be available once the user authenticates.
3. Open the app in your browser:
   - Visit the dashboard page (e.g., `http://localhost:3000/`).
4. Connect an embedded Privy wallet:
   - Use the UI to sign in / connect the embedded wallet. The `useWallets` hook will then expose the connected wallet address used by the test hook.
5. Trigger the x402 flow:
   - The mock test endpoint is at `/api/x402/test`. The client helper (`useMakePaymentFn`) and `FetchWithX402` components call this endpoint via Privy’s `wrapFetchWithPayment`.
   - When the client calls the endpoint, the server will reply with 402 and an `x-payment-required` header containing payment instructions.
   - Privy will prompt the embedded wallet to sign the EIP-712 authorization. After signing, the client will retry the request with an `X-PAYMENT` header. The mock route accepts the header and returns mock premium content.
6. Verify the result:
   - The mock endpoint logs the received `X-PAYMENT` header server-side (check terminal/console where your Next.js server is running).
   - The client-side agent records the transaction in localStorage (see `agent_transactions`) and may surface notifications via the agent context if configured.
7. Optional: Inspect network traffic
   - Use the browser Network tab to inspect the initial 402 response and the subsequent request that includes the `X-PAYMENT` header.

Notes and tips
- The mock `x-payment-required` payload uses placeholders (amount, token, recipient). Replace with appropriate testnet token addresses and amounts before testing on a real testnet.
- `connectWallet()` calls may not synchronously update `useWallets()` state. The client helper performs a short, best-effort re-check after connect. If your Privy SDK returns the connected wallet from `connectWallet`, prefer using that returned value.
- The local mock does not verify signatures onchain; it accepts any `X-PAYMENT` header for development. In production your server must verify the EIP-712 signature, call a facilitator (or verify settlement onchain), and prevent replay/exploit vectors (nonces/expiry).
- For production, integrate with an x402 facilitator (e.g., Pay AI, Corbits) and implement server-side verification and facilitator callbacks.

Troubleshooting
- If the client never receives a 402 from the mock route:
  - Confirm the request is going to `http://localhost:3000/api/x402/test`.
  - Confirm the server is running and the file `app/api/x402/test/route.ts` exists.
- If the wallet signature prompt doesn't appear:
  - Confirm Privy is initialized with `NEXT_PUBLIC_PRIVY_APP_ID`.
  - Ensure the user is signed in and an embedded wallet is available.
- If the signature succeeds but the server still returns 402:
  - Check server logs for the received `X-PAYMENT` header and confirm the client encoded it correctly.
  - Confirm `wrapFetchWithPayment` was used to wrap the fetch call (the hook handles building the typed data and retry).

Next steps (recommended)
1. Replace mock token and recipient with testnet USDC addresses for the network you target (Polygon, Base, or Solana).
2. Integrate a facilitator in the server workflow to validate and settle authorizations onchain.
3. Add server-side verification and webhook handling to update subscription state once settlement completes.
4. Add a minimal UI toast/notification (not in the button) to inform users of payment success or failure.

If you'd like, I can:
- Replace placeholder token/recipient values in the mock with specific testnet addresses,
- Add a short README subsection that shows expected network-specific values (Polygon testnet etc.),
- Or wire the subscription card to call the agent automatically.

Which next small change should I make for you?

---

## x402 Quickstart for Buyers

How to use x402 as a buyer

Learn how to use x402 on Polygon to automatically discover paywalled resources, complete payments in USDC, and retrieve paid data with a single API call.

You will achieve:
- Detect 402 Payment Required responses
- Programmatically complete USDC payments
- Access the unlocked resource using x402-fetch or x402-axios

Security notice
This tutorial uses test credentials and local endpoints for clarity. In production, never expose private keys or facilitator URLs publicly.

### Overview
x402 extends the HTTP standard to enable machine-to-machine payments using the 402 Payment Required status code. It works like a web paywall for APIs — when your request hits a paid endpoint, x402 provides headers that describe how to pay (e.g., cost, token, facilitator).  
Once your wallet client confirms payment, you receive the actual response.

Example: calling GET /weather returns a 402 with payment metadata; the x402 client automatically pays, retries, and fetches the weather data.

### Prerequisites
- Wallet: Metamask, Rabby, or any Polygon-compatible wallet with USDC
- JS env: Node.js 18+ (or your preferred JS runtime)
- Polygon testnet: Amoy RPC (or another testnet you prefer)
- .env: Must include PRIVATE_KEY; optional FACILITATOR_URL, RESOURCE_URL

### Install dependencies
Install an official x402 HTTP client (choose one):

bun install x402-fetch
# or
bun install x402-axios

(If you use npm/yarn/pnpm, replace `bun install` accordingly.)

Also install viem + dotenv (or your preferred Ethereum tooling):

bun install viem dotenv

### Create a wallet client
Set up a Polygon wallet using viem. This wallet signs payments when a 402 challenge is detected.

```js
import { createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { polygonAmoy } from "viem/chains";
import "dotenv/config";

const privateKey = process.env.PRIVATE_KEY;
if (!privateKey) throw new Error("PRIVATE_KEY not set in .env");

const account = privateKeyToAccount(`0x${privateKey}`);
const client = createWalletClient({
  account,
  chain: polygonAmoy,
  transport: http(),
});
console.log("Wallet address:", account.address);
```

Expected output:
```
Wallet address: 0x1234abcd...
```

### Make paid requests automatically
Use `x402-fetch` (or `x402-axios`) to intercept 402 responses and complete the payment automatically.

```js
import { wrapFetchWithPayment, decodeXPaymentResponse } from "x402-fetch";
import { createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { polygonAmoy } from "viem/chains";
import "dotenv/config";

const account = privateKeyToAccount(`0x${process.env.PRIVATE_KEY}`);
const client = createWalletClient({ account, chain: polygonAmoy, transport: http() });

const fetchWithPayment = wrapFetchWithPayment(fetch, client);
const FACILITATOR_URL = process.env.FACILITATOR_URL || "https://x402-amoy.polygon.technology";
const RESOURCE_URL = process.env.RESOURCE_URL || "http://127.0.0.1:4021/weather";

(async () => {
  const response = await fetchWithPayment(RESOURCE_URL, { method: "GET" });
  const body = await response.json();
  console.log("Response body:", body);
  if (body.report) {
    const rawPaymentResponse = response.headers.get("x-payment-response");
    console.log("Raw payment header:", rawPaymentResponse);
    const decoded = decodeXPaymentResponse(rawPaymentResponse);
    console.log("Decoded payment:", decoded);
  }
})();
```

How the client flow works:
1. Sends the request
2. Receives 402 Payment Required
3. Pays in USDC via a facilitator
4. Retries automatically with `X-PAYMENT`
5. Returns the unlocked JSON data

### Schema (env / config)
- PRIVATE_KEY (string, required) — Wallet private key for signing (use test key in dev).
- FACILITATOR_URL (string, optional) — Facilitator relay endpoint.
- RESOURCE_URL (string, required) — Target paid resource URL.
- USDC — Token used for payment (network-dependent).

### Common errors and fixes
- MISSING_CONFIG — Wallet or URL not set. Fix: verify .env keys.
- DUPLICATE_PAYMENT — Payment replay detected. Fix: retry with new request ID or ensure uniqueness.
- BAD_PAYMENT_HEADER — Invalid signature or amount. Fix: make sure client and facilitator are in sync.
- 402_LOOP — API keeps returning 402. Fix: check facilitator health or endpoint config.

### Do / Don't guardrails
Do:
- Use a sandbox facilitator for testing.
- Log decoded payment responses.
- Use `wrapFetchWithPayment` or `x402-axios`.

Don't:
- Hardcode private keys in scripts.
- Parse payment headers manually.
- Re-implement 402 logic yourself without using tested libraries.

---

If you'd like, I can:
- Add concrete example `.env` snippets for the Polygon Amoy testnet,
- Replace placeholder values in the mock endpoint with specific testnet USDC addresses,
- Or add a short script that runs a local x402 test flow end-to-end.

Which of the above would you like me to add next?
