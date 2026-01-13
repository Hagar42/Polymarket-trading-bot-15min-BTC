# Resources & Glossary

> Quick reference guide to Polymarket terminology, links, and resources

## Official Resources

### Documentation
- **Polymarket Docs**: https://docs.polymarket.com/developers
- **CLOB Introduction**: https://docs.polymarket.com/developers/CLOB/introduction
- **NEG_RISK Overview**: https://docs.polymarket.com/developers/neg-risk/overview
- **CTF Overview**: https://docs.polymarket.com/developers/CTF/overview

### GitHub Repositories
- **py-clob-client** (Python SDK): https://github.com/Polymarket/py-clob-client
- **clob-client** (TypeScript SDK): https://github.com/Polymarket/clob-client
- **NEG_RISK Adapter**: https://github.com/Polymarket/neg-risk-ctf-adapter
- **CTF Exchange**: https://github.com/Polymarket/ctf-exchange

### API Endpoints
- **CLOB API**: https://clob.polymarket.com
- **WebSocket**: wss://ws-subscriptions-clob.polymarket.com/ws/market
- **Polygon RPC**: https://polygon-rpc.com

### Community
- **Discord**: https://discord.gg/polymarket
- **Twitter**: https://twitter.com/Polymarket
- **Telegram**: Various trading communities

---

## Glossary

### A

**Arbitrage**  
Trading strategy that exploits price differences to guarantee profit. On Polymarket, buying both YES and NO when their combined cost is less than $1.00.

**Ask**  
The lowest price at which someone is willing to sell (the price you pay when buying).

**API (Application Programming Interface)**  
Interface that allows programs to interact with Polymarket programmatically.

**API Credentials**  
API Key, Secret, and Passphrase used to authenticate with the Polymarket CLOB.

### B

**Bid**  
The highest price at which someone is willing to buy (the price you receive when selling).

**Binary Market**  
A market with exactly two possible outcomes (e.g., YES/NO, UP/DOWN).

**Book** (Order Book)  
List of all buy and sell orders for a specific outcome token, organized by price.

### C

**CLOB (Central Limit Order Book)**  
The order matching system used by Polymarket. Orders are matched based on price-time priority.

**Collateral**  
The asset used to back outcome tokens. On Polymarket, this is USDC.

**Condition**  
The event or question being predicted. Each market has a unique condition ID.

**Conditional Token Framework (CTF)**  
Ethereum standard (ERC-1155) for creating prediction market tokens. Developed by Gnosis.

**CTF Exchange**  
Smart contract on Polygon that handles token splitting, merging, and redemption.

### D

**Dry Run**  
Simulation mode where bot logic runs without placing real orders.

### E

**EOA (Externally Owned Account)**  
Standard Ethereum wallet controlled by a private key (e.g., MetaMask).

**EIP-712**  
Ethereum standard for signing typed data. Used by Polymarket for order signatures.

**ERC-1155**  
Ethereum token standard that supports multiple token types in a single contract.

### F

**FAK (Fill-and-Kill)**  
Order type that fills as much as possible immediately and cancels the rest. Also called IOC.

**FOK (Fill-or-Kill)**  
Order type that must fill completely immediately or be cancelled entirely.

**Funder**  
For proxy wallets (Magic.link), the address where your funds are actually held. Different from your signer address.

### G

**GTC (Good-Til-Cancelled)**  
Order type that remains active in the order book until filled or manually cancelled.

**Gnosis**  
Organization that created the Conditional Token Framework.

### I

**IOC (Immediate-or-Cancel)**  
Same as FAK. Fills what it can immediately, cancels the rest.

### L

**Limit Order**  
Order with a specified price. Will only execute at that price or better.

**Liquidity**  
The availability of shares to buy or sell. High liquidity means many orders at various prices.

### M

**Maker**  
Trader who adds liquidity to the order book by placing limit orders. On 15-min crypto markets, makers earn rebates.

**Market Maker**  
Strategy of placing both buy and sell orders to profit from the spread while providing liquidity.

**Market Order**  
Order that executes immediately at the best available price.

**Merging**  
Converting a complete set of outcome tokens (YES + NO) back into collateral (USDC) before settlement.

**Midpoint**  
Average of best bid and best ask prices: `(best_bid + best_ask) / 2`

### N

**NEG_RISK (Negative Risk)**  
Advanced market structure for mutually exclusive multi-outcome events. Allows conversion between NO tokens across markets.

**NegRiskAdapter**  
Smart contract that handles NEG_RISK conversions and position management.

### O

**Oracle**  
Entity or system that reports the outcome of a market for settlement.

**Order Book**  
List of all buy (bid) and sell (ask) orders for a token, sorted by price.

**Outcome Token**  
ERC-1155 token representing a specific outcome of a market (e.g., YES, NO).

### P

**Polygon**  
Layer 2 Ethereum scaling solution where Polymarket is deployed. Chain ID: 137.

**Position**  
Your holdings in a market. Number of outcome tokens you own.

**Proxy Wallet**  
Wallet implementation used by Magic.link and browser wallets where funds are held in a separate contract.

### R

**Redemption**  
Converting winning outcome tokens to USDC after market settlement.

**Resolution**  
The process of determining and reporting the winning outcome of a market.

### S

**Settlement**  
Final resolution of a market where winning tokens become redeemable for $1.00 each.

**Signature Type**  
Parameter specifying wallet type: 0 = EOA, 1 = Proxy (Magic.link), 2 = Gnosis Safe.

**Signer**  
The wallet/address that signs transactions and orders.

**Slippage**  
Difference between expected price and actual execution price, usually due to market movement.

**Splitting**  
Converting USDC collateral into a complete set of outcome tokens (YES + NO).

**Spread**  
Difference between best bid and best ask: `best_ask - best_bid`

### T

**Taker**  
Trader who removes liquidity by executing against existing orders. On 15-min crypto markets, takers pay fees.

**Token ID**  
Unique identifier for an outcome token (ERC-1155 token ID).

**Time in Force**  
How long an order remains active (FOK, GTC, FAK/IOC).

### U

**USDC**  
USD Coin, the stablecoin used as collateral on Polymarket.

### W

**WebSocket**  
Protocol for real-time bidirectional communication. Used for streaming market data.

---

## Code Quick Reference

### Initialize Client

```python
from py_clob_client.client import ClobClient
import os

client = ClobClient(
    host="https://clob.polymarket.com",
    key=os.getenv("POLYMARKET_PRIVATE_KEY"),
    chain_id=137,
    signature_type=int(os.getenv("POLYMARKET_SIGNATURE_TYPE", "0")),
    funder=os.getenv("POLYMARKET_FUNDER")
)

client.set_api_creds({
    "apiKey": os.getenv("POLYMARKET_API_KEY"),
    "secret": os.getenv("POLYMARKET_API_SECRET"),
    "passphrase": os.getenv("POLYMARKET_API_PASSPHRASE")
})
```

### Common Operations

```python
# Get markets
markets = client.get_markets()

# Get order book
book = client.get_order_book(token_id)

# Get balance
balance = client.get_balance_allowance()

# Place order
order = client.create_and_post_order({
    "token_id": "0x...",
    "price": 0.65,
    "size": 10,
    "side": "BUY",
    "order_type": "FOK"
})

# Cancel order
client.cancel_order(order_id)

# Get positions
positions = client.get_positions()
```

---

## Smart Contract Addresses (Polygon Mainnet)

```python
CONTRACTS = {
    "CTF_EXCHANGE": "0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E",
    "CONDITIONAL_TOKENS": "0x4D97DCd97eC945f40cF65F87097ACe5EA0476045",
    "NEG_RISK_ADAPTER": "0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296",
    "NEG_RISK_CTF_EXCHANGE": "0xC5d563A36AE78145C45a50134d48A1215220f80a",
    "USDC": "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"
}
```

---

## Useful Links

### Trading & Analysis Tools
- **PolyTrack HQ**: https://www.polytrackhq.app/
- **Polymarket Dashboard**: https://polymarket.com/activity
- **Crypto 15-min Markets**: https://polymarket.com/crypto/15M

### Educational Content
- **Polymarket Blog**: https://polymarket.com/blog
- **YouTube Tutorials**: Search "Polymarket trading bot tutorial"
- **Medium Articles**: Various community guides

### Development Tools
- **Polygon Explorer**: https://polygonscan.com/
- **Web3.py Docs**: https://web3py.readthedocs.io/
- **Ethers.js Docs**: https://docs.ethers.org/

### Market Data
- **Polymarket API**: Direct API for market data
- **The Graph**: Subgraph for historical data
- **Dune Analytics**: Community dashboards

---

## Rate Limits

| Endpoint Type | Limit | Notes |
|--------------|-------|-------|
| Public (read) | 100 req/min | Market data, order books |
| Trading (write) | 60 orders/min | Order placement, cancellation |
| WebSocket | N/A | No strict limit, connection-based |

---

## Common Error Codes

| Error | Meaning | Solution |
|-------|---------|----------|
| `invalid signature` | Authentication failed | Verify signature_type and funder |
| `insufficient balance` | Not enough USDC | Deposit more USDC |
| `invalid price` | Price out of range | Use 0.01 to 0.99 |
| `invalid size` | Order size too small | Minimum 5 shares |
| `rate limit exceeded` | Too many requests | Slow down requests |
| `market closed` | Market no longer active | Find active market |

---

## Environment Variables Template

```env
# Wallet Configuration
POLYMARKET_PRIVATE_KEY=0x...
POLYMARKET_SIGNATURE_TYPE=0
POLYMARKET_FUNDER=

# API Credentials (generate with create_or_derive_api_creds)
POLYMARKET_API_KEY=
POLYMARKET_API_SECRET=
POLYMARKET_API_PASSPHRASE=

# Trading Configuration
DRY_RUN=true
ORDER_SIZE=5
TARGET_PAIR_COST=0.99
ORDER_TYPE=FOK
COOLDOWN_SECONDS=10

# Risk Management
MAX_DAILY_LOSS=50.0
MAX_POSITION_SIZE=100.0
MIN_BALANCE_REQUIRED=10.0
```

---

## Support & Help

### Official Support
- **Email**: support@polymarket.com
- **Discord**: #dev-chat channel

### Community Resources
- **GitHub Issues**: Report bugs in SDK repos
- **Stack Overflow**: Tag questions with `polymarket`
- **Reddit**: r/Polymarket

### This Bot Repository
- **Telegram**: [@terauss](https://t.me/terauss)
- **GitHub Issues**: Report problems or ask questions

---

## Next Steps

Ready to start building? Go back to:
- **[README](./README.md)** - Documentation index
- **[Quick Start](./02-quickstart.md)** - Build your first bot
- **[Bot Strategies](./07-bot-strategies.md)** - Trading strategies

---

*Last updated: 2026-01-13*
