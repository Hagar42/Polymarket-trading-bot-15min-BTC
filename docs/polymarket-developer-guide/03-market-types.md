# Market Types & Structures

> Understanding Polymarket's market types, CTF framework, and NEG_RISK markets

## Overview

Polymarket markets are built on the Gnosis Conditional Token Framework (CTF) and support two main architectures:
1. **Standard CTF Markets** - Traditional binary YES/NO markets
2. **NEG_RISK Markets** - Advanced multi-outcome markets with capital efficiency

---

## Conditional Token Framework (CTF)

### What is CTF?

The Conditional Token Framework is an Ethereum standard (ERC-1155) for creating prediction market tokens. Each market outcome is represented as a unique token.

### Key Concepts

```
Event: "Will Bitcoin be above $50k on Dec 31?"
    ├─ Condition ID: 0xabc123... (unique identifier)
    ├─ Collateral: USDC
    ├─ Outcomes:
    │   ├─ YES (Token ID: 0xdef456...)
    │   └─ NO (Token ID: 0xghi789...)
    └─ Settlement: Oracle resolves → Winning side gets $1.00/share
```

### Token Lifecycle

1. **Splitting**: Convert USDC → YES + NO tokens
2. **Trading**: Buy/sell on CLOB
3. **Merging**: Combine YES + NO → USDC (before settlement)
4. **Redemption**: Exchange winning tokens → USDC (after settlement)

### Code Example: Understanding Tokens

```python
from py_clob_client.client import ClobClient

client = ClobClient("https://clob.polymarket.com")

# Get a market
markets = client.get_markets()
market = markets[0]

print(f"Market: {market['question']}")
print(f"Condition ID: {market['condition_id']}")

# Each market has 2 tokens (YES/NO)
for token in market['tokens']:
    print(f"\nOutcome: {token['outcome']}")
    print(f"Token ID: {token['token_id']}")
    print(f"Winner: {token.get('winner', 'Not settled')}")
```

---

## Standard Binary Markets

### Characteristics

- **Outcomes**: 2 (YES/NO, UP/DOWN, etc.)
- **Settlement**: One outcome pays $1.00, other pays $0.00
- **Cost**: YES + NO always equals exactly $1.00 (excluding fees)
- **Use Cases**: Most prediction markets

### Examples

1. **Politics**: "Will candidate X win the election?"
2. **Sports**: "Will Team A win the game?"
3. **Crypto 15-min**: "Will BTC go up in next 15 minutes?"

### Market Structure

```json
{
  "question": "Will Bitcoin be above $50k on Dec 31?",
  "condition_id": "0xabc123...",
  "tokens": [
    {
      "outcome": "Yes",
      "token_id": "0xdef456...",
      "price": "0.65"
    },
    {
      "outcome": "No",
      "token_id": "0xghi789...",
      "price": "0.35"
    }
  ],
  "end_date_iso": "2026-12-31T23:59:59Z"
}
```

### Arbitrage Principle

Since YES + NO must equal $1.00:
- If YES = $0.48 and NO = $0.51
- Total = $0.99 < $1.00
- **Profit = $0.01** guaranteed (1.01% return)

---

## NEG_RISK Markets

### What is NEG_RISK?

NEG_RISK (Negative Risk) markets are designed for **mutually exclusive multi-outcome events** where only ONE outcome can win. They provide:

1. **Capital Efficiency**: Convert NO shares across markets
2. **Higher Liquidity**: Reduced capital requirements
3. **Better UX**: Easier position management

### How It Works

In a standard election with 3 candidates (A, B, C):
- Traditional: You need separate markets for each
- NEG_RISK: All markets are linked

**Key Innovation**: Holding NO on candidates A and B is economically equivalent to holding YES on candidate C.

### Conversion Mechanism

```
Position: 1 NO on A + 1 NO on B
         ↓ (Convert via NegRiskAdapter)
Position: 1 YES on C + USDC collateral
```

### Example: Presidential Election

```
Event: "Who will win the 2026 election?"

Markets:
├─ Candidate A wins: YES/NO
├─ Candidate B wins: YES/NO
├─ Candidate C wins: YES/NO
└─ Other candidate wins: YES/NO

All linked via NEG_RISK adapter
```

### Code Example: Detecting NEG_RISK

```python
from py_clob_client.client import ClobClient

client = ClobClient("https://clob.polymarket.com")

def is_neg_risk_market(market):
    """Check if market uses NEG_RISK"""
    return market.get('neg_risk', False)

# Find NEG_RISK markets
markets = client.get_markets()
neg_risk_markets = [m for m in markets if is_neg_risk_market(m)]

print(f"Found {len(neg_risk_markets)} NEG_RISK markets")

for market in neg_risk_markets[:5]:
    print(f"- {market['question']}")
    print(f"  NEG_RISK: {market.get('neg_risk_market_id', 'N/A')}")
```

### NEG_RISK Market Structure

```json
{
  "question": "Who will win the election?",
  "neg_risk": true,
  "neg_risk_market_id": "0xmarket123...",
  "neg_risk_request_id": "0xrequest456...",
  "tokens": [
    {
      "outcome": "Candidate A",
      "token_id": "0xtoken1..."
    },
    {
      "outcome": "Candidate B", 
      "token_id": "0xtoken2..."
    }
  ]
}
```

### Trading NEG_RISK Markets

**Important:** When trading NEG_RISK markets, you must specify `neg_risk=True` in order parameters:

```python
# Place order on NEG_RISK market
order = client.create_order({
    "token_id": "0xtoken_id...",
    "price": 0.65,
    "size": 10,
    "side": "BUY",
    "neg_risk": True  # Required for NEG_RISK markets!
})
```

---

## Market Categories

### 1. Political Markets
- **Type**: Usually NEG_RISK (multi-candidate)
- **Duration**: Days to months
- **Volume**: Very high during election seasons
- **Examples**: Presidential elections, primaries

### 2. Sports Markets
- **Type**: Standard binary
- **Duration**: Hours to days
- **Volume**: High during games
- **Examples**: Game winners, player performance

### 3. Crypto 15-Minute Markets
- **Type**: Standard binary (UP/DOWN)
- **Duration**: 15 minutes
- **Volume**: Extremely high
- **Special**: Taker fees apply, maker rebates
- **Examples**: "Will BTC go up in next 15 min?"

### 4. Economic Indicators
- **Type**: Binary or NEG_RISK
- **Duration**: Weeks to months
- **Examples**: "Will inflation be above 3%?"

### 5. Entertainment
- **Type**: NEG_RISK (multiple nominees)
- **Duration**: Months
- **Examples**: "Who will win Best Picture Oscar?"

---

## Market Discovery

### Finding Markets by Category

```python
from py_clob_client.client import ClobClient

client = ClobClient("https://clob.polymarket.com")

# Get all markets
markets = client.get_markets()

# Filter by keyword
btc_markets = [
    m for m in markets 
    if 'bitcoin' in m['question'].lower() or 'btc' in m['question'].lower()
]

# Filter by active status
active_markets = [
    m for m in markets
    if not m.get('closed', False)
]

# Filter 15-minute markets
from datetime import datetime, timedelta

def is_15min_market(market):
    """Check if market is a 15-minute market"""
    try:
        end_date = datetime.fromisoformat(market['end_date_iso'].replace('Z', '+00:00'))
        now = datetime.now(end_date.tzinfo)
        duration = (end_date - now).total_seconds() / 60
        return duration <= 20  # Approx 15-min market
    except:
        return False

short_term_markets = [m for m in markets if is_15min_market(m)]
```

### Finding Markets by Slug

```python
def find_market_by_slug(client, slug):
    """Find market by slug or partial match"""
    markets = client.get_markets()
    
    for market in markets:
        # Check market slug
        if market.get('market_slug') == slug:
            return market
        
        # Check question contains keyword
        if slug.lower() in market['question'].lower():
            return market
    
    return None

# Usage
market = find_market_by_slug(client, "bitcoin-15m")
```

---

## Market Lifecycle

### 1. Market Creation
- Oracle creates question
- Condition ID generated
- Tokens minted

### 2. Trading Phase
- Market is active
- Orders matched on CLOB
- Prices reflect probability

### 3. Settlement
- Event occurs
- Oracle reports outcome
- Winning side determined

### 4. Redemption
- Winners claim $1.00/share
- Losers get $0.00

### Market States

```python
def get_market_state(market):
    """Determine market state"""
    if market.get('closed'):
        if market.get('resolved'):
            return "SETTLED"
        return "CLOSED_PENDING_RESOLUTION"
    
    from datetime import datetime
    end_date = datetime.fromisoformat(market['end_date_iso'].replace('Z', '+00:00'))
    now = datetime.now(end_date.tzinfo)
    
    if now >= end_date:
        return "ENDED"
    
    return "ACTIVE"
```

---

## Smart Contract Addresses

### Polygon Mainnet

```python
# CTF Exchange (where trading happens)
CTF_EXCHANGE = "0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E"

# Conditional Tokens (ERC-1155)
CONDITIONAL_TOKENS = "0x4D97DCd97eC945f40cF65F87097ACe5EA0476045"

# NEG_RISK Adapter
NEG_RISK_ADAPTER = "0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296"

# NEG_RISK CTF Exchange
NEG_RISK_CTF_EXCHANGE = "0xC5d563A36AE78145C45a50134d48A1215220f80a"

# USDC (Collateral)
USDC = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"
```

---

## Market Data Fields

### Essential Fields

```python
{
    "condition_id": "0x...",      # Unique market identifier
    "question": "...",             # Market question
    "market_slug": "...",          # URL-friendly slug
    "end_date_iso": "...",         # When market closes
    "game_start_time": "...",      # Event start (sports)
    "tokens": [...],               # Outcome tokens
    "neg_risk": false,             # Is NEG_RISK market?
    "closed": false,               # Is market closed?
    "resolved": false,             # Is outcome decided?
    "active": true,                # Is trading active?
}
```

### Token Fields

```python
{
    "token_id": "0x...",          # Outcome token address
    "outcome": "Yes",              # Outcome name
    "price": "0.65",               # Last trade price (optional)
    "winner": false,               # Is winning outcome? (after settlement)
}
```

---

## Best Practices

### 1. Market Selection
- ✅ Focus on high-liquidity markets
- ✅ Understand the event being traded
- ✅ Check market end time
- ✅ Verify NEG_RISK status before trading

### 2. Risk Management
- ✅ Don't accumulate positions across market close
- ✅ Monitor for oracle issues
- ✅ Understand settlement mechanism

### 3. NEG_RISK Trading
- ✅ Always set `neg_risk=True` parameter
- ✅ Understand conversion mechanics
- ✅ Monitor all related markets

---

## Next Steps

- **[API Reference](./04-api-reference.md)** - Learn API endpoints
- **[Trading Guide](./06-trading-orders.md)** - Place orders
- **[Bot Strategies](./07-bot-strategies.md)** - Build trading strategies

---

**Questions?** See [Troubleshooting](./10-troubleshooting.md)
