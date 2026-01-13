# Quick Reference Card

> One-page reference for common Polymarket bot operations

## 🚀 Essential Setup

### 1. Install SDK
```bash
pip install py-clob-client python-dotenv
```

### 2. Environment Variables
```env
POLYMARKET_PRIVATE_KEY=0x...
POLYMARKET_SIGNATURE_TYPE=0
POLYMARKET_API_KEY=...
POLYMARKET_API_SECRET=...
POLYMARKET_API_PASSPHRASE=...
```

### 3. Initialize Client
```python
from py_clob_client.client import ClobClient
import os
from dotenv import load_dotenv

load_dotenv()

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

---

## 📊 Common Operations

### Get Markets
```python
markets = client.get_markets()
for m in markets[:5]:
    print(f"{m['question'][:50]}... | Ends: {m['end_date_iso']}")
```

### Get Order Book
```python
book = client.get_order_book(token_id)
best_bid = float(book['bids'][0]['price'])
best_ask = float(book['asks'][0]['price'])
print(f"Bid: ${best_bid:.4f} | Ask: ${best_ask:.4f}")
```

### Check Balance
```python
balance = client.get_balance_allowance()
print(f"Balance: ${balance['balance']}")
```

### Place Order
```python
order = client.create_and_post_order({
    "token_id": "0x...",
    "price": 0.65,
    "size": 10,
    "side": "BUY",
    "order_type": "FOK"
})
```

### Cancel Order
```python
client.cancel_order(order_id)
```

### Get Positions
```python
positions = client.get_positions()
```

---

## 🎯 Arbitrage Pattern

```python
# Find opportunity
def find_arb(market):
    tokens = market['tokens']
    yes_book = client.get_order_book(tokens[0]['token_id'])
    no_book = client.get_order_book(tokens[1]['token_id'])
    
    yes_price = float(yes_book['asks'][0]['price'])
    no_price = float(no_book['asks'][0]['price'])
    
    if yes_price + no_price < 0.99:
        return {
            'yes': {'token': tokens[0], 'price': yes_price},
            'no': {'token': tokens[1], 'price': no_price},
            'profit': 1.0 - (yes_price + no_price)
        }
    return None

# Execute arbitrage
def execute_arb(opp, size=10):
    yes_order = client.create_and_post_order({
        "token_id": opp['yes']['token']['token_id'],
        "price": opp['yes']['price'],
        "size": size,
        "side": "BUY",
        "order_type": "FOK"
    })
    
    no_order = client.create_and_post_order({
        "token_id": opp['no']['token']['token_id'],
        "price": opp['no']['price'],
        "size": size,
        "side": "BUY",
        "order_type": "FOK"
    })
    
    return yes_order, no_order
```

---

## 📈 Market Maker Pattern

```python
def place_quotes(token_id, spread=0.02, size=10):
    book = client.get_order_book(token_id)
    
    best_bid = float(book['bids'][0]['price'])
    best_ask = float(book['asks'][0]['price'])
    fair = (best_bid + best_ask) / 2
    
    # Place orders
    buy = client.create_and_post_order({
        "token_id": token_id,
        "price": fair - spread/2,
        "size": size,
        "side": "BUY",
        "order_type": "GTC"
    })
    
    sell = client.create_and_post_order({
        "token_id": token_id,
        "price": fair + spread/2,
        "size": size,
        "side": "SELL",
        "order_type": "GTC"
    })
    
    return buy, sell
```

---

## 🔑 Key Concepts

### Signature Types
| Type | Value | Use Case |
|------|-------|----------|
| EOA | 0 | MetaMask, hardware wallets |
| POLY_PROXY | 1 | Magic.link (email login) |
| POLY_GNOSIS_SAFE | 2 | Multi-sig wallets |

### Order Types
| Type | Behavior |
|------|----------|
| FOK | All or nothing (use for arbitrage) |
| GTC | Stays until filled (use for market making) |
| FAK | Partial fills OK (use for quick execution) |

### Market States
```python
def get_state(market):
    if market.get('closed'):
        return "CLOSED"
    from datetime import datetime
    end = datetime.fromisoformat(market['end_date_iso'].replace('Z', '+00:00'))
    if datetime.now(end.tzinfo) >= end:
        return "ENDED"
    return "ACTIVE"
```

---

## ⚡ Pro Tips

### 1. Always Use Dry Run First
```python
DRY_RUN = True

if not DRY_RUN:
    execute_real_order()
else:
    print("Would execute:", order_params)
```

### 2. Verify Both Legs in Arbitrage
```python
# After placing both orders
yes_status = client.get_order(yes_order['id'])
no_status = client.get_order(no_order['id'])

if yes_status['status'] != 'FILLED' or no_status['status'] != 'FILLED':
    print("⚠️ Partial fill! Unwinding...")
    # Cancel and exit positions
```

### 3. Set NEG_RISK for Multi-Outcome Markets
```python
if market.get('neg_risk'):
    order_params['neg_risk'] = True
```

### 4. Handle Errors Gracefully
```python
try:
    order = client.create_and_post_order(params)
except Exception as e:
    print(f"Order failed: {e}")
    # Log and continue, don't crash
```

### 5. Monitor Performance
```python
class Stats:
    def __init__(self):
        self.trades = 0
        self.wins = 0
        self.total_pnl = 0.0
    
    def record(self, pnl):
        self.trades += 1
        if pnl > 0:
            self.wins += 1
        self.total_pnl += pnl
    
    def summary(self):
        win_rate = (self.wins / self.trades * 100) if self.trades > 0 else 0
        avg_pnl = self.total_pnl / self.trades if self.trades > 0 else 0
        return f"Trades: {self.trades} | Win Rate: {win_rate:.1f}% | Avg P&L: ${avg_pnl:.2f}"
```

---

## 🔗 Smart Contract Addresses (Polygon)

```python
CTF_EXCHANGE = "0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E"
CONDITIONAL_TOKENS = "0x4D97DCd97eC945f40cF65F87097ACe5EA0476045"
NEG_RISK_ADAPTER = "0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296"
USDC = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"
```

---

## 📞 Quick Help

- **Docs**: [docs/polymarket-developer-guide/README.md](./README.md)
- **Auth Issues**: [01-authentication.md](./01-authentication.md)
- **Trading**: [06-trading-orders.md](./06-trading-orders.md)
- **Strategies**: [07-bot-strategies.md](./07-bot-strategies.md)
- **Glossary**: [11-resources.md](./11-resources.md)

---

**Print this for quick reference while coding!** 🚀
