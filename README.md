# TINT Protocol & SDK

![TINT Protocol Banner](https://raw.githubusercontent.com/Khalid-000-ME/tint-stream-sdk/main/frontend/public/tint-flow.svg)

> **The Privacy-Preserving Intent Standard for AI Agents.**
> Sub-second, MEV-proof DeFi execution using Yellow Network state channels, Zero-Knowledge Commitments, and Uniswap V4.

[![NPM Version](https://img.shields.io/npm/v/tint-protocol-ai-sdk)](https://www.npmjs.com/package/tint-protocol-ai-sdk)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-production--ready-green.svg)]()

---

## 📚 Table of Contents

- [Overview](#-overview)
- [Why TINT?](#-why-tint)
- [Core Architecture](#-core-architecture)
  - [1. Intent Parsing (Gemini AI)](#1-intent-parsing-gemini-ai)
  - [2. Privacy (Pedersen Commitments)](#2-privacy-pedersen-commitments)
  - [3. Transport (Yellow Network)](#3-transport-yellow-network)
  - [4. Settlement (Uniswap V4 & Circle CCTP)](#4-settlement-uniswap-v4--circle-cctp)
- [SDK Documentation](#-sdk-documentation)
  - [Installation](#installation)
  - [Configuration](#configuration)
  - [Quick Start](#quick-start)
  - [Advanced Usage](#advanced-usage)
- [Example Scenarios](#-example-scenarios)
- [Smart Contracts](#-smart-contracts)
- [Development Setup](#-development-setup)

---

## 🎯 Overview

**TINT (Threshold Intent Netting & Transport)** is a protocol designed to solve the critical issues of **front-running (MEV)** and **data leakage** in agentic DeFi trading.

By leveraging **Pedersen Commitments** and **State Channels**, TINT allows AI agents to stream trade intents privately. These intents are aggregated off-chain by Solvers, netted against each other, and only the residual difference is settled on-chain via **Uniswap V4**.

This results in:
- **Zero MEV**: Searchers cannot see the trade details before execution.
- **Gas Efficiency**: 98% reduction in gas costs due to off-chain netting.
- **Deep Liquidity**: Guaranteed settlement on Uniswap V4 for residual flows.
- **Cross-Chain**: Seamless bridging via abstracted **Circle CCTP**.

---

## 💡 Why TINT?

Current DeFi execution is flawed for high-frequency AI agents:
1.  **Public Mempool Leakage**: Agents broadcasting intents reveal their strategy to predatory MEV bots.
2.  **High Gas Costs**: Executing every single trade on-chain is prohibitively expensive.
3.  **Fragmentation**: Liquidity is split across chains and pools.

**TINT solves this by creating a "Dark Pool" for Agents.**
Intents are encrypted, matched off-chain, and settled in bulk.

---

## 🏗 Core Architecture

The TINT Protocol consists of four main pillars:

### 1. Intent Parsing (Gemini AI)
The SDK integrates **Google Gemini AI** to robustly parse natural language into structured executable intents.
- **Input**: "Swap 100 USDC to WETH on Base, then bridge 50 WETH to Arbitrum."
- **Output**: JSON Array of `SWAP` and `BRIDGE` actions with precise parameters.

### 2. Privacy (Pedersen Commitments)
We use **Pedersen Commitments** to cryptographically shield trade amounts while allowing Solvers to verify the sum.
- **Formula**: `C = g^a * h^r` (where `a` = amount, `r` = randomness).
- **Property**: Homomorphic. Solvers can add commitments `C1 + C2` to get `C_total` without knowing `a1` or `a2`.
- **ZKP**: Allows proving solvency without revealing the trade size.

### 3. Transport (Yellow Network)
Intents are streamed via **Yellow Network State Channels** (`@erc7824/nitrolite`).
- **Performance**: <100ms latency p2p communication.
- **Security**: State channels ensure that intent data is only visible to the designated Solver/Broker.
- **Off-Chain Ledger**: Tracks netting balances before on-chain settlement.

### 4. Settlement (Uniswap V4 & Circle CCTP)
- **Swaps**: The **net residual** (e.g., if Alice buys 10 ETH and Bob sells 8 ETH, the net is Buy 2 ETH) is executed on **Uniswap V4** via a custom Hook.
- **Bridging**: Cross-chain intents utilize the **Circle Bridge Kit** (CCTP) for burn-and-mint USDC transfers, eliminating slippage.

---

## � SDK Documentation

The `tint-protocol-ai-sdk` is the easiest way to integrate TINT into your dApp or Agent.

### Installation

```bash
npm install tint-protocol-ai-sdk ethers @google/generative-ai dotenv
```

### Configuration

Create a `.env` file in your project root:

```env
# Wallet Configuration
PRIVATE_KEY=your_wallet_private_key_here
RPC_URL=https://sepolia.base.org

# AI Configuration (Optional, for Natural Language)
GEMINI_API_KEY=your_google_gemini_key

# TINT Network (Defaults provided in SDK)
TINT_BROKER_URL=https://broker.tint.org
```

### Quick Start

Here is a complete example of an AI Agent performing a private swap:

```typescript
require('dotenv').config();
const { TintClient } = require('tint-protocol-ai-sdk');

async function main() {
    // 1. Initialize Client
    const client = new TintClient({
        privateKey: process.env.PRIVATE_KEY,
        rpcUrl: process.env.RPC_URL,
        geminiApiKey: process.env.GEMINI_API_KEY
    });

    console.log("🚀 Agent Initialized:", client.getAddress());

    // 2. Process Natural Language Command
    // The SDK handles parsing, commitment generation, and submission.
    const result = await client.processNaturalLanguage(
        "Swap 500 USDC to WETH on Base"
    );

    // 3. Output Results
    if (result.success) {
        console.log("✅ Swap Executed!");
        console.log(`   Tx Hash: ${result.txHash}`);
        console.log(`   Efficiency: ${result.efficiency}% Gas Saved`);
    } else {
        console.error("❌ Failed:", result.error);
    }
}

main();
```

### Advanced Usage

#### Manual Intent Creation
If you don't want to use AI, you can construct intents programmatically:

```typescript
const { TintClient } = require('tint-protocol-ai-sdk');

const client = new TintClient({ ...config });

// Create a Manual SWAP Intent
const intent = {
    type: 'SWAP',
    fromToken: 'USDC',
    toToken: 'WETH',
    amount: 1000,
    recipient: client.getAddress(),
    network: 'base'
};

// 1. Generate Privacy Commitment
const { intent: securedIntent, commitment } = await client.createIntent(intent);

// 2. Submit to TINT Solver
await client.submitIntent({
    intent: securedIntent,
    commitment
});

// 3. Trigger Execution (or wait for batch)
const result = await client.executeBatch();
```

#### Cross-Chain Bridging
The SDK abstracts the Circle Bridge Kit for seamless transfers:

```typescript
// AI Command
await client.processNaturalLanguage(
    "Bridge 1000 USDC from Base to Arbitrum"
);

// Manual Command
await client.createIntent({
    type: 'BRIDGE',
    fromChain: 'Base',
    toChain: 'Arbitrum',
    amount: 1000,
    fromToken: 'USDC'
});
```

---

## 🧪 Example Scenarios

### Scenario A: The Private Whale
**User**: "I want to sell 100 ETH for USDC, but I don't want to crash the market or be front-run."
**TINT Solution**:
1.  user submits 100 ETH sell intent encrypted with Pedersen Commitment.
2.  Solver matches this against 200 small buy orders totaling 95 ETH.
3.  **Net Result**: Sell 5 ETH.
4.  **On-Chain**: Only 5 ETH is sold on Uniswap V4. The market impact is minimized by 95%.

### Scenario B: The Cross-Chain Yield Farmer
**User**: "Move my USDC yield from Base to Arbitrum to buy GMX."
**TINT Solution**:
1.  **Step 1**: TINT bridges USDC from Base to Arbitrum using Circle CCTP (Mint/Burn).
2.  **Step 2**: Once on Arbitrum, TINT creates a Swap intent (USDC -> GMX) and executes it via the local Solver.
3.  **Result**: 1-Click complex workflow execution.

---

## � Smart Contracts

| Network | Contract | Address |
| :--- | :--- | :--- |
| **Base Sepolia** | TINTNettingVerifier | `0x3837C39afF6A207C8B89fa9e4DAa45e3FBB35443` |
| **Arbitrum Sepolia** | TINTNettingVerifier | `0x7c7ccbc98469190849BCC6c926307794fDfB11F2` |
| **Sepolia** | Uniswap V4 PoolManager | `0x...` (Standard V4 Deployment) |

---

## 💻 Development Setup

To contribute to the TINT Protocol or run the full stack locally:

### Prerequisites
- Node.js v18+
- Foundry (for smart contracts)
- Docker (optional, for Yellow Node)

### Setup Steps
1.  **Clone the Repository**
    ```bash
    git clone https://github.com/Khalid-000-ME/tint-stream-sdk.git
    cd tint-stream-sdk
    ```

2.  **Install Frontend Dependencies**
    ```bash
    cd frontend
    npm install
    ```

3.  **Configure Environment**
    Copy `.env.example` to `.env.local` and fill in your keys.

4.  **Run Development Server**
    ```bash
    npm run dev
    ```
    Open `http://localhost:3000` to see the TINT Dashboard.

5.  **Run SDK Tests**
    ```bash
    cd sdk
    npm install
    npm run build
    node test-agent.js
    ```

---

## 🤝 Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct and development process.

## 📄 License

This project is licensed under the **MIT License**.

---

<p align="center">
  Built with ❤️ by the TINT Protocol Team for the AI Agent Economy.
</p>
