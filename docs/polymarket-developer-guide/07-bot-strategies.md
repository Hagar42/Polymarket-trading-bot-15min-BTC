# Bot Development Strategies

> Comprehensive guide to building trading bots on Polymarket

## Overview

This guide covers proven trading strategies for Polymarket bots:

1. **Arbitrage** - Exploit price inefficiencies
2. **Market Making** - Provide liquidity for profit
3. **Trend Following** - Trade based on price momentum
4. **Oracle Front-Running** - React to external events
5. **Statistical Arbitrage** - Cross-market opportunities

---

## 1. Arbitrage Strategies

### Intra-Market Arbitrage (Most Common)

**Concept**: In binary markets, YES + NO should always equal $1.00. When it's less, buy both sides for guaranteed profit.

**Example:**
```
YES price: $0.48
NO price:  $0.51
Total:     $0.99

Profit: $1.00 - $0.99 = $0.01 (1.01% return)
```

### Python Implementation

```python
from py_clob_client.client import ClobClient
from py_clob_client.clob_types import OrderArgs
import time

class ArbitrageBot:
    def __init__(self, client, threshold=0.99):
        self.client = client
        self.threshold = threshold  # Max total cost for arbitrage
        
    def find_arbitrage_opportunity(self, market):
        """Check if market has arbitrage opportunity"""
        tokens = market.get('tokens', [])
        if len(tokens) != 2:
            return None
        
        # Get best ask prices for both sides
        prices = {}
        for token in tokens:
            token_id = token['token_id']
            outcome = token['outcome']
            
            try:
                book = self.client.get_order_book(token_id)
                asks = book.get('asks', [])
                
                if not asks:
                    return None
                
                # Best ask (lowest sell price = where we can buy)
                best_ask = float(asks[0]['price'])
                size_available = float(asks[0]['size'])
                
                prices[outcome] = {
                    'price': best_ask,
                    'size': size_available,
                    'token_id': token_id
                }
            except Exception as e:
                print(f"Error fetching book for {outcome}: {e}")
                return None
        
        if len(prices) != 2:
            return None
        
        # Calculate total cost
        outcomes = list(prices.keys())
        price_yes = prices[outcomes[0]]['price']
        price_no = prices[outcomes[1]]['price']
        total_cost = price_yes + price_no
        
        # Check for arbitrage
        if total_cost < self.threshold:
            profit = 1.0 - total_cost
            profit_pct = (profit / total_cost) * 100
            
            # Calculate max shares we can buy
            max_shares = min(
                prices[outcomes[0]]['size'],
                prices[outcomes[1]]['size']
            )
            
            return {
                'market': market,
                'total_cost': total_cost,
                'profit': profit,
                'profit_pct': profit_pct,
                'max_shares': max_shares,
                'sides': prices
            }
        
        return None
    
    def execute_arbitrage(self, opportunity, shares=10, dry_run=True):
        """Execute arbitrage trade"""
        if dry_run:
            print(f"\n🎯 ARBITRAGE OPPORTUNITY (DRY RUN)")
            print(f"Market: {opportunity['market']['question']}")
            print(f"Total cost: ${opportunity['total_cost']:.4f}")
            print(f"Profit: ${opportunity['profit']:.4f} ({opportunity['profit_pct']:.2f}%)")
            print(f"Shares: {shares}")
            print(f"Total investment: ${opportunity['total_cost'] * shares:.2f}")
            print(f"Expected profit: ${opportunity['profit'] * shares:.2f}")
            return None
        
        # Place orders on both sides
        orders = []
        for outcome, data in opportunity['sides'].items():
            try:
                order = self.client.create_and_post_order({
                    "token_id": data['token_id'],
                    "price": data['price'],
                    "size": shares,
                    "side": "BUY",
                    "order_type": "FOK",  # Fill-or-kill (all or nothing)
                    "neg_risk": opportunity['market'].get('neg_risk', False)
                })
                orders.append(order)
                print(f"✅ Bought {shares} {outcome} @ ${data['price']}")
            except Exception as e:
                print(f"❌ Error placing order for {outcome}: {e}")
                # Try to cancel other orders if partial fill
                self.cancel_pending_orders()
                return None
        
        return orders
    
    def scan_markets(self, market_filter=None):
        """Scan all markets for arbitrage opportunities"""
        markets = self.client.get_markets()
        
        if market_filter:
            markets = [m for m in markets if market_filter(m)]
        
        opportunities = []
        for market in markets:
            if market.get('closed'):
                continue
            
            opp = self.find_arbitrage_opportunity(market)
            if opp:
                opportunities.append(opp)
        
        return opportunities
    
    def run(self, interval=10, market_filter=None, max_shares=10, dry_run=True):
        """Main bot loop"""
        print(f"\n🤖 Arbitrage Bot Started")
        print(f"Threshold: ${self.threshold}")
        print(f"Max shares: {max_shares}")
        print(f"Dry run: {dry_run}")
        print(f"Scan interval: {interval}s")
        print("=" * 60)
        
        try:
            while True:
                opportunities = self.scan_markets(market_filter)
                
                if opportunities:
                    print(f"\n✨ Found {len(opportunities)} opportunities!")
                    
                    for opp in opportunities:
                        self.execute_arbitrage(opp, max_shares, dry_run)
                else:
                    print(".", end="", flush=True)
                
                time.sleep(interval)
                
        except KeyboardInterrupt:
            print("\n\n👋 Bot stopped")

# Usage example
if __name__ == "__main__":
    from dotenv import load_dotenv
    import os
    
    load_dotenv()
    
    client = ClobClient(
        host="https://clob.polymarket.com",
        key=os.getenv("POLYMARKET_PRIVATE_KEY"),
        chain_id=137
    )
    client.set_api_creds({
        "apiKey": os.getenv("POLYMARKET_API_KEY"),
        "secret": os.getenv("POLYMARKET_API_SECRET"),
        "passphrase": os.getenv("POLYMARKET_API_PASSPHRASE")
    })
    
    # Create bot
    bot = ArbitrageBot(client, threshold=0.99)
    
    # Filter for BTC 15-min markets
    def btc_15min_filter(market):
        return 'bitcoin' in market['question'].lower() and '15' in market['question']
    
    # Run bot
    bot.run(
        interval=5,
        market_filter=btc_15min_filter,
        max_shares=10,
        dry_run=True  # Set to False for real trading
    )
```

### Advanced Arbitrage: Depth-Aware Execution

```python
def calculate_average_fill_price(order_book_side, target_size):
    """Calculate average price to fill target_size shares"""
    total_cost = 0.0
    filled_size = 0.0
    
    for level in order_book_side:
        level_price = float(level['price'])
        level_size = float(level['size'])
        
        if filled_size + level_size >= target_size:
            # This level completes the order
            remaining = target_size - filled_size
            total_cost += remaining * level_price
            filled_size = target_size
            break
        else:
            # Take entire level
            total_cost += level_size * level_price
            filled_size += level_size
    
    if filled_size < target_size:
        return None  # Not enough liquidity
    
    return total_cost / target_size

# Use in arbitrage check
def find_arbitrage_with_depth(self, market, target_shares=100):
    tokens = market.get('tokens', [])
    if len(tokens) != 2:
        return None
    
    avg_prices = {}
    for token in tokens:
        book = self.client.get_order_book(token['token_id'])
        asks = book.get('asks', [])
        
        avg_price = calculate_average_fill_price(asks, target_shares)
        if not avg_price:
            return None
        
        avg_prices[token['outcome']] = avg_price
    
    total_cost = sum(avg_prices.values())
    
    if total_cost < self.threshold:
        return {
            'market': market,
            'total_cost': total_cost,
            'profit': 1.0 - total_cost,
            'shares': target_shares,
            'avg_prices': avg_prices
        }
    
    return None
```

---

## 2. Market Making Strategy

**Concept**: Provide liquidity by placing limit orders on both sides of the market, profiting from the spread.

### Simple Market Maker

```python
class MarketMaker:
    def __init__(self, client, spread=0.02):
        self.client = client
        self.spread = spread  # Minimum spread to maintain
        
    def calculate_fair_price(self, token_id):
        """Calculate fair price based on order book midpoint"""
        book = self.client.get_order_book(token_id)
        
        bids = book.get('bids', [])
        asks = book.get('asks', [])
        
        if not bids or not asks:
            return None
        
        best_bid = float(bids[0]['price'])
        best_ask = float(asks[0]['price'])
        
        # Midpoint
        fair_price = (best_bid + best_ask) / 2.0
        
        return fair_price
    
    def place_quotes(self, token_id, size=10, dry_run=True):
        """Place bid and ask quotes"""
        fair_price = self.calculate_fair_price(token_id)
        
        if not fair_price:
            return None
        
        # Calculate our bid/ask prices
        our_bid = fair_price - (self.spread / 2)
        our_ask = fair_price + (self.spread / 2)
        
        # Ensure prices are valid (0.01 to 0.99)
        our_bid = max(0.01, min(0.99, our_bid))
        our_ask = max(0.01, min(0.99, our_ask))
        
        if dry_run:
            print(f"\n📊 Market Making Quote (DRY RUN)")
            print(f"Fair price: ${fair_price:.4f}")
            print(f"Our bid: ${our_bid:.4f} (buy {size})")
            print(f"Our ask: ${our_ask:.4f} (sell {size})")
            print(f"Spread: ${our_ask - our_bid:.4f}")
            return None
        
        # Place orders
        try:
            bid_order = self.client.create_and_post_order({
                "token_id": token_id,
                "price": our_bid,
                "size": size,
                "side": "BUY",
                "order_type": "GTC"  # Good-til-cancelled
            })
            
            ask_order = self.client.create_and_post_order({
                "token_id": token_id,
                "price": our_ask,
                "size": size,
                "side": "SELL",
                "order_type": "GTC"
            })
            
            return {'bid': bid_order, 'ask': ask_order}
            
        except Exception as e:
            print(f"❌ Error placing quotes: {e}")
            return None
```

---

## 3. Statistical Arbitrage

**Concept**: Compare prices across different platforms or correlated markets.

### Cross-Platform Arbitrage

```python
class CrossPlatformArbitrage:
    def __init__(self, polymarket_client, other_platform_client):
        self.poly = polymarket_client
        self.other = other_platform_client
    
    def find_cross_platform_opportunity(self, poly_market_id, other_market_id):
        """Find arbitrage between Polymarket and another platform"""
        # Get Polymarket price
        poly_book = self.poly.get_order_book(poly_market_id)
        poly_best_ask = float(poly_book['asks'][0]['price'])
        
        # Get other platform price
        other_price = self.other.get_price(other_market_id)
        
        # Check for arbitrage
        if other_price > poly_best_ask + 0.02:  # 2% threshold
            return {
                'poly_price': poly_best_ask,
                'other_price': other_price,
                'spread': other_price - poly_best_ask,
                'action': 'BUY_POLY_SELL_OTHER'
            }
        elif poly_best_ask > other_price + 0.02:
            return {
                'poly_price': poly_best_ask,
                'other_price': other_price,
                'spread': poly_best_ask - other_price,
                'action': 'BUY_OTHER_SELL_POLY'
            }
        
        return None
```

---

## 4. 15-Minute Crypto Strategy

**Specialized strategy for BTC/ETH/SOL 15-minute markets**

```python
import requests
from datetime import datetime

class Crypto15MinBot:
    def __init__(self, client, crypto='BTC'):
        self.client = client
        self.crypto = crypto
        
    def get_spot_price(self):
        """Get current spot price from Coinbase"""
        response = requests.get(
            f'https://api.coinbase.com/v2/prices/{self.crypto}-USD/spot'
        )
        data = response.json()
        return float(data['data']['amount'])
    
    def find_15min_market(self):
        """Find active 15-minute market"""
        markets = self.client.get_markets()
        
        for market in markets:
            question = market['question'].lower()
            if self.crypto.lower() in question and '15' in question:
                # Check if market is about to close soon
                end_time = datetime.fromisoformat(
                    market['end_date_iso'].replace('Z', '+00:00')
                )
                now = datetime.now(end_time.tzinfo)
                minutes_remaining = (end_time - now).total_seconds() / 60
                
                if 0 < minutes_remaining < 15:
                    return market
        
        return None
    
    def predict_direction(self, market):
        """Predict if price will go UP or DOWN"""
        # Get current spot price
        current_price = self.get_spot_price()
        
        # Extract target price from market question
        # Example: "Will Bitcoin be above $50,000 in 15 minutes?"
        # (This is simplified - real implementation needs better parsing)
        
        # Compare with market prices
        tokens = market['tokens']
        
        up_token = next(t for t in tokens if 'up' in t['outcome'].lower())
        down_token = next(t for t in tokens if 'down' in t['outcome'].lower())
        
        up_book = self.client.get_order_book(up_token['token_id'])
        down_book = self.client.get_order_book(down_token['token_id'])
        
        up_price = float(up_book['asks'][0]['price'])
        down_price = float(down_book['asks'][0]['price'])
        
        # Simple momentum strategy
        # If UP is underpriced relative to DOWN, buy UP
        if up_price < down_price - 0.05:
            return 'UP', up_token, up_price
        elif down_price < up_price - 0.05:
            return 'DOWN', down_token, down_price
        
        return None, None, None
```

---

## 5. Risk Management

### Position Limits

```python
class RiskManager:
    def __init__(self, max_position_size=100, max_daily_loss=50):
        self.max_position_size = max_position_size
        self.max_daily_loss = max_daily_loss
        self.daily_pnl = 0.0
        self.positions = {}
        
    def can_trade(self, size, cost):
        """Check if trade is within risk limits"""
        # Check position size
        if cost > self.max_position_size:
            print(f"❌ Trade blocked: Exceeds max position size")
            return False
        
        # Check daily loss
        if self.daily_pnl < -self.max_daily_loss:
            print(f"❌ Trading blocked: Max daily loss reached")
            return False
        
        return True
    
    def record_trade(self, pnl):
        """Record trade P&L"""
        self.daily_pnl += pnl
        
    def reset_daily(self):
        """Reset daily stats (call at start of day)"""
        self.daily_pnl = 0.0
```

---

## Best Practices

### 1. Start Small
```python
# ✅ Good: Start with minimum sizes
ORDER_SIZE = 5  # Minimum on Polymarket

# ❌ Bad: Start with large positions
ORDER_SIZE = 1000
```

### 2. Use Simulation Mode
```python
# Always test first
DRY_RUN = True

if not DRY_RUN:
    # Only execute real trades after thorough testing
    execute_trade()
```

### 3. Implement Logging
```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('bot.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)
logger.info("Trade executed: ...")
```

### 4. Handle Errors Gracefully
```python
try:
    order = client.create_and_post_order(...)
except Exception as e:
    logger.error(f"Order failed: {e}")
    # Don't crash - continue monitoring
    pass
```

### 5. Monitor Performance
```python
class PerformanceTracker:
    def __init__(self):
        self.trades = []
        self.total_pnl = 0.0
        
    def record_trade(self, trade):
        self.trades.append(trade)
        self.total_pnl += trade['pnl']
        
    def get_stats(self):
        if not self.trades:
            return None
        
        win_rate = sum(1 for t in self.trades if t['pnl'] > 0) / len(self.trades)
        avg_pnl = self.total_pnl / len(self.trades)
        
        return {
            'total_trades': len(self.trades),
            'win_rate': win_rate,
            'avg_pnl': avg_pnl,
            'total_pnl': self.total_pnl
        }
```

---

## Next Steps

- **[WebSocket Streaming](./05-websocket-streaming.md)** - Get real-time data
- **[Risk Management](./08-risk-management.md)** - Advanced risk controls
- **[Code Examples](./09-code-examples.md)** - More code samples

---

**Ready to build?** Start with the [Quick Start](./02-quickstart.md) guide!
