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
