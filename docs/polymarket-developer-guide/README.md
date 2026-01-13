# Polymarket Developer Guide

> **Comprehensive documentation for building trading bots on Polymarket**  
> This guide provides everything you need to create automated trading bots using the Polymarket CLOB (Central Limit Order Book) API.

## 📚 Table of Contents

### Getting Started
1. **[Overview](#overview)** - Introduction to Polymarket and its ecosystem
2. **[Authentication](./01-authentication.md)** - How to authenticate and generate API credentials
3. **[Quick Start](./02-quickstart.md)** - Get your first bot running in 5 minutes

### Core Concepts
4. **[Market Types & Structures](./03-market-types.md)** - Understanding binary markets, NEG_RISK, and CTF
5. **[API Reference](./04-api-reference.md)** - Complete API endpoints documentation
6. **[WebSocket Streaming](./05-websocket-streaming.md)** - Real-time market data feeds
7. **[Trading & Orders](./06-trading-orders.md)** - Order types, execution, and fees

### Advanced Topics
8. **[Bot Development Strategies](./07-bot-strategies.md)** - Arbitrage, market making, and other strategies
9. **[Risk Management](./08-risk-management.md)** - Best practices for managing risk
10. **[Code Examples](./09-code-examples.md)** - Practical code snippets and examples

### Reference
11. **[Troubleshooting](./10-troubleshooting.md)** - Common issues and solutions
12. **[Resources & Links](./11-resources.md)** - External documentation and tools
13. **[Glossary](./12-glossary.md)** - Terms and definitions

---

## Overview

### What is Polymarket?

Polymarket is a decentralized prediction market platform built on Polygon that allows users to trade on the outcome of real-world events. The platform uses:

- **Blockchain**: Polygon (Layer 2 Ethereum)
- **Collateral**: USDC stablecoin
- **Token Standard**: ERC-1155 (Conditional Tokens Framework)
- **Trading Mechanism**: Central Limit Order Book (CLOB)

### Why Build Bots on Polymarket?

1. **Arbitrage Opportunities**: Price inefficiencies between YES/NO sides
2. **High Liquidity**: Especially on popular markets (politics, crypto, sports)
3. **Fast Markets**: 15-minute crypto markets create frequent opportunities
4. **Public API**: Full programmatic access to all trading functions
5. **Zero Fees**: Most markets are fee-free (except 15-min crypto markets)

### Key Features for Developers

- ✅ **REST API** - Complete access to markets, order books, trading
- ✅ **WebSocket API** - Real-time market data and order updates
- ✅ **Python & TypeScript SDKs** - Official client libraries
- ✅ **EIP-712 Signatures** - Secure order signing
- ✅ **Proxy Wallet Support** - Magic.link, browser wallets
- ✅ **NEG_RISK Markets** - Advanced multi-outcome markets

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Your Trading Bot                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Strategy   │  │     Risk     │  │   Execution  │  │
│  │    Logic     │  │  Management  │  │    Engine    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │     Polymarket CLOB Client SDK       │
        │  (py-clob-client / @polymarket)      │
        └──────────────────────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          ▼                                  ▼
    ┌──────────┐                      ┌──────────┐
    │ REST API │                      │ WebSocket│
    │          │                      │   API    │
    └──────────┘                      └──────────┘
          │                                  │
          └────────────────┬─────────────────┘
                           ▼
              ┌────────────────────────┐
              │  Polymarket CLOB       │
              │  (Off-chain Matching)  │
              └────────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │  CTF Exchange Contract │
              │  (On-chain Settlement) │
              │  Polygon Blockchain    │
              └────────────────────────┘
```

---

## Quick Navigation

### I want to...

- **Get started quickly** → See [Quick Start Guide](./02-quickstart.md)
- **Understand authentication** → See [Authentication Guide](./01-authentication.md)
- **Learn about markets** → See [Market Types](./03-market-types.md)
- **Use the API** → See [API Reference](./04-api-reference.md)
- **Build an arbitrage bot** → See [Bot Strategies](./07-bot-strategies.md)
- **Stream real-time data** → See [WebSocket Streaming](./05-websocket-streaming.md)
- **Fix an issue** → See [Troubleshooting](./10-troubleshooting.md)

---

## Prerequisites

Before you start, make sure you have:

- ✅ **Python 3.8+** or **Node.js 14+**
- ✅ **Polymarket account** (create at https://polymarket.com)
- ✅ **USDC funds** in your Polymarket wallet (for live trading)
- ✅ **Wallet private key** (MetaMask, hardware wallet, or Magic.link)
- ✅ **Basic understanding** of REST APIs and async programming

---

## Language Support

This guide covers both **Python** and **TypeScript/JavaScript** implementations:

### Python (py-clob-client)
```bash
pip install py-clob-client
```

### TypeScript/JavaScript (@polymarket/clob-client)
```bash
npm install @polymarket/clob-client ethers
```

Most examples in this guide use **Python** since it's commonly used for trading bots, but TypeScript equivalents are provided where relevant.

---

## Documentation Structure

Each guide follows this format:

1. **Overview** - What you'll learn
2. **Concepts** - Key terminology and theory
3. **Implementation** - Step-by-step code examples
4. **Best Practices** - Tips and recommendations
5. **Common Issues** - Troubleshooting

---

## Safety First ⚠️

Before building bots for Polymarket:

- ⚠️ **Never share your private key** with anyone
- ⚠️ **Start with simulation mode** (dry run) to test strategies
- ⚠️ **Use small amounts** when testing live trading
- ⚠️ **Implement proper error handling** and logging
- ⚠️ **Set up risk limits** (max loss, position size, etc.)
- ⚠️ **Monitor your bots** - don't leave them unattended
- ⚠️ **Understand market risks** - arbitrage isn't always risk-free
- ⚠️ **Comply with regulations** in your jurisdiction

---

## Support & Community

- **Official Docs**: https://docs.polymarket.com/developers
- **GitHub**: 
  - Python SDK: https://github.com/Polymarket/py-clob-client
  - TypeScript SDK: https://github.com/Polymarket/clob-client
- **Discord**: https://discord.gg/polymarket
- **Telegram**: Various trading communities

---

## Contributing

Found an error or want to improve this guide? Contributions are welcome!

---

**Ready to start?** Begin with the [Authentication Guide](./01-authentication.md) →

---

*Last updated: 2026-01-13*  
*Guide maintained by the Polymarket bot developer community*
