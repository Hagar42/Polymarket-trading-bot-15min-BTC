# Trading & Orders Guide

> Complete guide to placing orders and understanding Polymarket trading mechanics

## Overview

This guide covers:
- Order types and parameters
- Order execution
- Trading fees
- Order book mechanics
- Best practices

---

## Order Types

### 1. Fill-or-Kill (FOK)

**Behavior**: Order must fill completely immediately or be cancelled entirely.

**Use Case**: Arbitrage - ensures both legs fill or neither fills.

```python
order = client.create_and_post_order({
    "token_id": "0x...",
    "price": 0.65,
    "size": 10,
    "side": "BUY",
    "order_type": "FOK"  # Fill-or-Kill
})
```

**Pros**:
- ✅ No partial fills
- ✅ All-or-nothing execution
- ✅ Perfect for arbitrage

**Cons**:
- ❌ May not fill if insufficient liquidity
- ❌ Higher rejection rate

### 2. Good-Til-Cancelled (GTC)

**Behavior**: Order stays in book until filled or manually cancelled.

**Use Case**: Market making, limit orders.

```python
order = client.create_and_post_order({
    "token_id": "0x...",
    "price": 0.65,
    "size": 10,
    "side": "BUY",
    "order_type": "GTC"  # Good-Til-Cancelled
})
```

**Pros**:
- ✅ Captures future price movements
- ✅ Provides liquidity
- ✅ Better fill rate

**Cons**:
- ❌ May leave open positions
- ❌ Requires monitoring

### 3. Immediate-or-Cancel (IOC) / Fill-and-Kill (FAK)

**Behavior**: Fill what you can immediately, cancel the rest.

**Use Case**: Quick execution with some flexibility.

```python
order = client.create_and_post_order({
    "token_id": "0x...",
    "price": 0.65,
    "size": 10,
    "side": "BUY",
    "order_type": "FAK"  # Fill-and-Kill
})
```

**Pros**:
- ✅ Partial fills allowed
- ✅ No lingering orders
- ✅ Fast execution

**Cons**:
- ❌ May only partially fill
- ❌ Less control over fill size

---

## Order Parameters

### Required Parameters

```python
{
    "token_id": "0x...",        # Token to trade
    "price": 0.65,              # Price per share (0.01 to 0.99)
    "size": 10,                 # Number of shares (min 5)
    "side": "BUY",              # "BUY" or "SELL"
}
```

### Optional Parameters

```python
{
    "order_type": "FOK",        # FOK, GTC, or FAK (default: GTC)
    "neg_risk": False,          # True for NEG_RISK markets
    "expiration": 1234567890,   # Unix timestamp (optional)
}
```

### Complete Example

```python
from py_clob_client.client import ClobClient
import time

client = ClobClient(...)
client.set_api_creds(...)

# Place a buy order
order = client.create_and_post_order({
    "token_id": "0x1234...",
    "price": 0.65,
    "size": 10,
    "side": "BUY",
    "order_type": "FOK",
    "neg_risk": False
})

print(f"Order ID: {order['id']}")
print(f"Status: {order['status']}")
```

---

## Trading Fees

### Fee-Free Markets

Most Polymarket markets have **zero fees**:
- Political markets
- Sports markets  
- Long-term crypto markets
- Entertainment markets

### 15-Minute Crypto Markets (Taker Fees Only)

**Fee Structure:**
- **Makers**: 0% fee + **rebates** (receive taker fees)
- **Takers**: Variable fee based on probability

**Fee Formula:**
```
Fee = Order Value × Fee Rate × (p × (1-p))^exponent

Where:
- p = outcome probability (price)
- exponent = curve parameter
- Max fee ~1.56% at p=0.50
- Min fee ~0% at p close to 0 or 1
```

**Examples:**

| Price | Size | Order Value | Taker Fee | Net Cost |
|-------|------|-------------|-----------|----------|
| $0.50 | 100  | $50.00      | ~$0.78    | $50.78   |
| $0.30 | 100  | $30.00      | ~$0.30    | $30.30   |
| $0.70 | 100  | $70.00      | ~$0.30    | $70.30   |
| $0.90 | 100  | $90.00      | ~$0.08    | $90.08   |

### Maker Rebates Program

All taker fees are redistributed daily to makers (liquidity providers):

```python
# Check your maker rebates
rebates = client.get_maker_rebates()
print(f"Earned rebates: ${rebates['total']}")
```

---

## Order Execution Flow

### 1. Create Order

```python
order = client.create_and_post_order({...})
```

### 2. Order States

```
PENDING → PROCESSING → FILLED / CANCELLED / REJECTED
```

### 3. Check Order Status

```python
# Get order by ID
order_status = client.get_order(order['id'])
print(order_status['status'])

# Possible statuses:
# - PENDING: Submitted but not processed
# - LIVE: In order book
# - FILLED: Completely filled
# - PARTIALLY_FILLED: Partially filled (GTC/FAK only)
# - CANCELLED: Cancelled by user or system
# - REJECTED: Invalid or failed
```

### 4. Cancel Order

```python
# Cancel specific order
client.cancel_order(order['id'])

# Cancel all orders
client.cancel_all()

# Cancel orders for specific market
client.cancel_market_orders(market_id)
```

---

## Order Book Mechanics

### Understanding the Order Book

```python
book = client.get_order_book(token_id)

print("BIDS (Buy orders):")
for bid in book['bids'][:5]:
    print(f"  ${bid['price']:.4f} × {bid['size']}")

print("\nASKS (Sell orders):")
for ask in book['asks'][:5]:
    print(f"  ${ask['price']:.4f} × {ask['size']}")
```

**Output:**
```
BIDS (Buy orders):
  $0.6500 × 100
  $0.6490 × 50
  $0.6480 × 200
  $0.6470 × 75
  $0.6460 × 150

ASKS (Sell orders):
  $0.6520 × 80
  $0.6530 × 120
  $0.6540 × 60
  $0.6550 × 200
  $0.6560 × 90
```

### Spread

```python
best_bid = float(book['bids'][0]['price'])
best_ask = float(book['asks'][0]['price'])
spread = best_ask - best_bid

print(f"Spread: ${spread:.4f}")
```

### Market vs. Limit Orders

**Limit Order (Price Specified)**:
```python
# This will sit in book at $0.65 until filled
order = client.create_and_post_order({
    "token_id": "...",
    "price": 0.65,  # Your limit price
    "size": 10,
    "side": "BUY",
    "order_type": "GTC"
})
```

**Market Order (Take Best Available)**:
```python
# Get best ask price
book = client.get_order_book(token_id)
best_ask = float(book['asks'][0]['price'])

# Place at best ask (instant fill)
order = client.create_and_post_order({
    "token_id": "...",
    "price": best_ask,
    "size": 10,
    "side": "BUY",
    "order_type": "FOK"  # Immediate execution
})
```

---

## Trading Scenarios

### Scenario 1: Buy a YES Outcome

```python
# You believe YES will win, buy YES shares
token_id = "0x..."  # YES token ID
book = client.get_order_book(token_id)
best_ask = float(book['asks'][0]['price'])

order = client.create_and_post_order({
    "token_id": token_id,
    "price": best_ask,
    "size": 100,
    "side": "BUY",
    "order_type": "FOK"
})

# If YES wins, you get $100 (100 shares × $1.00)
# Cost: ~$65 (100 shares × $0.65)
# Profit: $35
```

### Scenario 2: Arbitrage Both Sides

```python
# Buy both YES and NO when total < $1.00
yes_token = "0x..."
no_token = "0x..."

yes_book = client.get_order_book(yes_token)
no_book = client.get_order_book(no_token)

yes_price = float(yes_book['asks'][0]['price'])
no_price = float(no_book['asks'][0]['price'])

if yes_price + no_price < 0.99:
    # Buy both sides
    yes_order = client.create_and_post_order({
        "token_id": yes_token,
        "price": yes_price,
        "size": 10,
        "side": "BUY",
        "order_type": "FOK"
    })
    
    no_order = client.create_and_post_order({
        "token_id": no_token,
        "price": no_price,
        "size": 10,
        "side": "BUY",
        "order_type": "FOK"
    })
    
    # Guaranteed profit when one side wins
```

### Scenario 3: Provide Liquidity (Market Making)

```python
# Place limit orders on both sides
token_id = "0x..."
book = client.get_order_book(token_id)

best_bid = float(book['bids'][0]['price'])
best_ask = float(book['asks'][0]['price'])

# Place bid slightly above current best
our_bid = best_bid + 0.01

buy_order = client.create_and_post_order({
    "token_id": token_id,
    "price": our_bid,
    "size": 50,
    "side": "BUY",
    "order_type": "GTC"
})

# Place ask slightly below current best
our_ask = best_ask - 0.01

sell_order = client.create_and_post_order({
    "token_id": token_id,
    "price": our_ask,
    "size": 50,
    "side": "SELL",
    "order_type": "GTC"
})

# Profit from spread when both fill
```

---

## Position Management

### Check Your Positions

```python
# Get all your positions
positions = client.get_positions()

for position in positions:
    print(f"Market: {position['market']}")
    print(f"Outcome: {position['outcome']}")
    print(f"Size: {position['size']}")
    print(f"Avg Price: ${position['avg_price']:.4f}")
    print(f"Current Value: ${position['value']:.2f}")
    print()
```

### Exit Position

```python
# Sell your position
def exit_position(client, token_id, size):
    """Sell position at market price"""
    book = client.get_order_book(token_id)
    
    if not book['bids']:
        print("No buyers available")
        return None
    
    best_bid = float(book['bids'][0]['price'])
    
    # Sell at best bid
    order = client.create_and_post_order({
        "token_id": token_id,
        "price": best_bid,
        "size": size,
        "side": "SELL",
        "order_type": "FOK"
    })
    
    return order
```

---

## Best Practices

### 1. Validate Before Trading

```python
def validate_order_params(order_params):
    """Validate order parameters"""
    # Check price range
    if not (0.01 <= order_params['price'] <= 0.99):
        raise ValueError("Price must be between 0.01 and 0.99")
    
    # Check minimum size
    if order_params['size'] < 5:
        raise ValueError("Minimum order size is 5 shares")
    
    # Check side
    if order_params['side'] not in ['BUY', 'SELL']:
        raise ValueError("Side must be BUY or SELL")
    
    return True
```

### 2. Handle Partial Fills

```python
def handle_order_result(client, order):
    """Check order status and handle partial fills"""
    status = client.get_order(order['id'])
    
    if status['status'] == 'FILLED':
        print(f"✅ Order fully filled: {status['size_matched']} shares")
        return 'FILLED'
    
    elif status['status'] == 'PARTIALLY_FILLED':
        filled = float(status['size_matched'])
        remaining = float(status['size']) - filled
        print(f"⚠️  Partial fill: {filled} / {status['size']} shares")
        
        # Cancel remaining
        client.cancel_order(order['id'])
        return 'PARTIAL'
    
    elif status['status'] == 'CANCELLED':
        print(f"❌ Order cancelled")
        return 'CANCELLED'
```

### 3. Implement Retry Logic

```python
import time

def place_order_with_retry(client, order_params, max_retries=3):
    """Place order with retry logic"""
    for attempt in range(max_retries):
        try:
            order = client.create_and_post_order(order_params)
            return order
        except Exception as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

### 4. Monitor Order Book Depth

```python
def check_liquidity(client, token_id, target_size):
    """Check if sufficient liquidity for target size"""
    book = client.get_order_book(token_id)
    asks = book.get('asks', [])
    
    total_size = sum(float(ask['size']) for ask in asks[:5])
    
    if total_size < target_size:
        print(f"⚠️  Insufficient liquidity: {total_size} < {target_size}")
        return False
    
    return True
```

---

## Troubleshooting

### "Invalid price" Error

```python
# ❌ Bad: Price out of range
order = client.create_and_post_order({
    "price": 1.05  # Over 0.99 limit
})

# ✅ Good: Valid price range
order = client.create_and_post_order({
    "price": 0.65  # Between 0.01 and 0.99
})
```

### "Insufficient balance" Error

```python
# Check balance before trading
balance = client.get_balance_allowance()
required = order_size * price

if float(balance['balance']) < required:
    print(f"Insufficient funds: {balance['balance']} < {required}")
```

### Order Not Filling

```python
# Check if price is competitive
book = client.get_order_book(token_id)
best_ask = float(book['asks'][0]['price'])

if your_bid < best_ask - 0.05:
    print(f"Bid too low: {your_bid} vs {best_ask}")
    # Consider increasing bid price
```

---

## Next Steps

- **[Bot Strategies](./07-bot-strategies.md)** - Learn trading strategies
- **[Risk Management](./08-risk-management.md)** - Manage your risk
- **[WebSocket](./05-websocket-streaming.md)** - Real-time order updates

---

**Need help?** See [Troubleshooting](./10-troubleshooting.md)
