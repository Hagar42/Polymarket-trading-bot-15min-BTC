# Quick Start Guide

> Get your first Polymarket bot running in 5 minutes

## Overview

This guide will walk you through creating a simple bot that:
1. Connects to Polymarket
2. Fetches market data
3. Places a test order (in simulation mode)

---

## Prerequisites

- Python 3.8+ or Node.js 14+
- Polymarket account
- Private key from your wallet

---

## Python Quick Start

### Step 1: Install Dependencies

```bash
pip install py-clob-client python-dotenv
```

### Step 2: Create Configuration File

Create a `.env` file:

```env
# Wallet Configuration
POLYMARKET_PRIVATE_KEY=0xYOUR_PRIVATE_KEY_HERE
POLYMARKET_SIGNATURE_TYPE=0

# Leave empty for EOA wallets, set for Magic.link
POLYMARKET_FUNDER=

# Will be generated in next step
POLYMARKET_API_KEY=
POLYMARKET_API_SECRET=
POLYMARKET_API_PASSPHRASE=
```

### Step 3: Generate API Credentials

Create `generate_credentials.py`:

```python
from py_clob_client.client import ClobClient
import os
from dotenv import load_dotenv

load_dotenv()

# Initialize client
client = ClobClient(
    host="https://clob.polymarket.com",
    key=os.getenv("POLYMARKET_PRIVATE_KEY"),
    chain_id=137
)

# Generate API credentials
api_creds = client.create_or_derive_api_creds()

print("\n" + "=" * 50)
print("API CREDENTIALS GENERATED")
print("=" * 50)
print(f"API Key:      {api_creds['apiKey']}")
print(f"API Secret:   {api_creds['secret']}")
print(f"Passphrase:   {api_creds['passphrase']}")
print("=" * 50)
print("\nAdd these to your .env file:")
print(f"POLYMARKET_API_KEY={api_creds['apiKey']}")
print(f"POLYMARKET_API_SECRET={api_creds['secret']}")
print(f"POLYMARKET_API_PASSPHRASE={api_creds['passphrase']}")
```

Run it:
```bash
python generate_credentials.py
```

Copy the output to your `.env` file.

### Step 4: Test Your Connection

Create `test_connection.py`:

```python
from py_clob_client.client import ClobClient
import os
from dotenv import load_dotenv

load_dotenv()

def test_connection():
    """Test connection to Polymarket"""
    
    # Initialize client
    client = ClobClient(
        host="https://clob.polymarket.com",
        key=os.getenv("POLYMARKET_PRIVATE_KEY"),
        chain_id=137,
        signature_type=int(os.getenv("POLYMARKET_SIGNATURE_TYPE", "0")),
        funder=os.getenv("POLYMARKET_FUNDER")
    )
    
    # Set API credentials
    client.set_api_creds({
        "apiKey": os.getenv("POLYMARKET_API_KEY"),
        "secret": os.getenv("POLYMARKET_API_SECRET"),
        "passphrase": os.getenv("POLYMARKET_API_PASSPHRASE")
    })
    
    print("\n🔍 Testing Polymarket Connection...")
    print("=" * 60)
    
    # Test 1: Server status
    print("\n1. Server Status:")
    ok = client.get_ok()
    print(f"   ✅ {ok}")
    
    # Test 2: Your wallet address
    print("\n2. Wallet Address:")
    address = client.get_address()
    print(f"   📍 {address}")
    
    # Test 3: USDC Balance
    print("\n3. USDC Balance:")
    balance_info = client.get_balance_allowance()
    print(f"   💰 Balance: ${balance_info['balance']}")
    print(f"   ✓ Allowance: ${balance_info['allowance']}")
    
    # Test 4: Get some markets
    print("\n4. Sample Markets:")
    markets = client.get_markets()
    for i, market in enumerate(markets[:3], 1):
        print(f"   {i}. {market.get('question', 'N/A')[:50]}")
    
    print("\n" + "=" * 60)
    print("✅ All tests passed! You're ready to trade.")
    print("=" * 60 + "\n")

if __name__ == "__main__":
    test_connection()
```

Run it:
```bash
python test_connection.py
```

### Step 5: Your First Bot - Market Data Fetcher

Create `simple_bot.py`:

```python
from py_clob_client.client import ClobClient
import os
from dotenv import load_dotenv
import time

load_dotenv()

def initialize_client():
    """Initialize and authenticate Polymarket client"""
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
    
    return client

def get_market_info(client, market_slug):
    """Get information about a specific market"""
    # Get market data
    markets = client.get_markets()
    
    # Find market by slug or question
    for market in markets:
        if market_slug.lower() in market.get('question', '').lower():
            return market
    
    return None

def get_order_book(client, token_id):
    """Get order book for a token"""
    book = client.get_order_book(token_id)
    return book

def display_market_data(client, market_slug="bitcoin"):
    """Display market data for a specific market"""
    
    print("\n" + "=" * 70)
    print(f"🔍 Searching for markets matching: '{market_slug}'")
    print("=" * 70)
    
    market = get_market_info(client, market_slug)
    
    if not market:
        print(f"❌ No market found matching '{market_slug}'")
        return
    
    print(f"\n📊 Market: {market.get('question', 'N/A')}")
    print(f"📍 Market ID: {market.get('condition_id', 'N/A')}")
    
    # Get outcome tokens
    tokens = market.get('tokens', [])
    
    for token in tokens[:2]:  # Usually YES/NO
        token_id = token.get('token_id')
        outcome = token.get('outcome', 'N/A')
        
        print(f"\n{'─' * 70}")
        print(f"🎯 Outcome: {outcome}")
        print(f"Token ID: {token_id}")
        
        # Get order book
        try:
            book = get_order_book(client, token_id)
            
            # Best bid/ask
            bids = book.get('bids', [])
            asks = book.get('asks', [])
            
            if asks:
                best_ask = asks[0]
                print(f"📈 Best Ask: ${float(best_ask['price']):.4f} (Size: {best_ask['size']})")
            
            if bids:
                best_bid = bids[0]
                print(f"📉 Best Bid: ${float(best_bid['price']):.4f} (Size: {best_bid['size']})")
            
            # Calculate spread
            if asks and bids:
                spread = float(asks[0]['price']) - float(bids[0]['price'])
                print(f"📊 Spread: ${spread:.4f}")
                
        except Exception as e:
            print(f"⚠️  Error fetching order book: {e}")
    
    print("\n" + "=" * 70 + "\n")

def main():
    """Main bot loop"""
    print("\n🤖 Simple Polymarket Data Bot")
    print("=" * 70)
    
    # Initialize
    client = initialize_client()
    print("✅ Connected to Polymarket")
    
    # Continuous monitoring
    try:
        while True:
            display_market_data(client, "bitcoin")
            
            # Wait before next update
            print("⏳ Waiting 30 seconds before next update...")
            time.sleep(30)
            
    except KeyboardInterrupt:
        print("\n\n👋 Bot stopped by user")
        print("=" * 70 + "\n")

if __name__ == "__main__":
    main()
```

Run it:
```bash
python simple_bot.py
```

---

## TypeScript Quick Start

### Step 1: Install Dependencies

```bash
npm install @polymarket/clob-client ethers dotenv
```

### Step 2: Create Configuration File

Create `.env`:

```env
PRIVATE_KEY=0xYOUR_PRIVATE_KEY_HERE
SIGNATURE_TYPE=0
FUNDER_ADDRESS=
```

### Step 3: Generate API Credentials

Create `generateCredentials.ts`:

```typescript
import { ClobClient } from "@polymarket/clob-client";
import { Wallet } from "ethers";
import * as dotenv from "dotenv";

dotenv.config();

async function generateCredentials() {
  const HOST = "https://clob.polymarket.com";
  const CHAIN_ID = 137;
  
  const signer = new Wallet(process.env.PRIVATE_KEY!);
  const client = new ClobClient(HOST, CHAIN_ID, signer);
  
  const apiCreds = await client.createOrDeriveApiKey();
  
  console.log("\n" + "=".repeat(50));
  console.log("API CREDENTIALS GENERATED");
  console.log("=".repeat(50));
  console.log("API Key:     ", apiCreds.apiKey);
  console.log("API Secret:  ", apiCreds.secret);
  console.log("Passphrase:  ", apiCreds.passphrase);
  console.log("=".repeat(50));
}

generateCredentials();
```

### Step 4: Simple Market Data Bot

Create `simpleBot.ts`:

```typescript
import { ClobClient } from "@polymarket/clob-client";
import { Wallet } from "ethers";
import * as dotenv from "dotenv";

dotenv.config();

async function main() {
  const HOST = "https://clob.polymarket.com";
  const CHAIN_ID = 137;
  
  // Initialize client
  const signer = new Wallet(process.env.PRIVATE_KEY!);
  const tempClient = new ClobClient(HOST, CHAIN_ID, signer);
  const apiCreds = await tempClient.createOrDeriveApiKey();
  
  const client = new ClobClient(
    HOST,
    CHAIN_ID,
    signer,
    apiCreds,
    parseInt(process.env.SIGNATURE_TYPE || "0")
  );
  
  console.log("\n🤖 Simple Polymarket Data Bot");
  console.log("=".repeat(60));
  
  // Test connection
  const ok = await client.getOk();
  console.log("✅ Connected:", ok);
  
  // Get markets
  const markets = await client.getMarkets();
  console.log(`\n📊 Found ${markets.length} markets`);
  
  // Display first 5 markets
  console.log("\nTop 5 Markets:");
  markets.slice(0, 5).forEach((market: any, i: number) => {
    console.log(`${i + 1}. ${market.question}`);
  });
  
  console.log("\n" + "=".repeat(60) + "\n");
}

main().catch(console.error);
```

Run it:
```bash
npx ts-node simpleBot.ts
```

---

## Understanding the Output

When you run the market data bot, you'll see:

```
🔍 Searching for markets matching: 'bitcoin'
======================================================================

📊 Market: Will Bitcoin go up or down in the next 15 minutes?
📍 Market ID: 0x1234...

──────────────────────────────────────────────────────────────────────
🎯 Outcome: Up
Token ID: 5678...
📈 Best Ask: $0.5100 (Size: 100)
📉 Best Bid: $0.4900 (Size: 50)
📊 Spread: $0.0200

──────────────────────────────────────────────────────────────────────
🎯 Outcome: Down
Token ID: 9012...
📈 Best Ask: $0.4900 (Size: 75)
📉 Best Bid: $0.4700 (Size: 100)
📊 Spread: $0.0200

======================================================================
```

**What this means:**
- **Best Ask**: Lowest price someone will sell at (you can BUY here)
- **Best Bid**: Highest price someone will buy at (you can SELL here)
- **Spread**: Difference between best bid and ask
- **Size**: Number of shares available at that price

---

## Next Steps: Simple Arbitrage Detector

Now let's detect arbitrage opportunities (when YES + NO < $1.00):

```python
def check_arbitrage(client, market_slug="bitcoin"):
    """Check for arbitrage opportunities"""
    
    market = get_market_info(client, market_slug)
    if not market:
        return
    
    tokens = market.get('tokens', [])
    if len(tokens) < 2:
        return
    
    # Get prices for both outcomes
    prices = {}
    for token in tokens[:2]:
        token_id = token.get('token_id')
        outcome = token.get('outcome')
        
        try:
            book = get_order_book(client, token_id)
            asks = book.get('asks', [])
            
            if asks:
                prices[outcome] = float(asks[0]['price'])
        except:
            pass
    
    # Check for arbitrage
    if len(prices) == 2:
        total_cost = sum(prices.values())
        
        print(f"\n{'─' * 60}")
        print(f"Market: {market.get('question', 'N/A')[:50]}")
        
        for outcome, price in prices.items():
            print(f"  {outcome}: ${price:.4f}")
        
        print(f"  Total: ${total_cost:.4f}")
        
        if total_cost < 1.0:
            profit = 1.0 - total_cost
            profit_pct = (profit / total_cost) * 100
            print(f"\n  🎯 ARBITRAGE OPPORTUNITY!")
            print(f"  💰 Profit: ${profit:.4f} ({profit_pct:.2f}%)")
        else:
            print(f"  ❌ No arbitrage (needs < $1.00)")
        
        print(f"{'─' * 60}\n")

# Add to main loop:
while True:
    check_arbitrage(client, "bitcoin")
    time.sleep(30)
```

---

## What You've Learned

✅ How to install and configure Polymarket client  
✅ How to generate API credentials  
✅ How to connect and test your setup  
✅ How to fetch market data  
✅ How to read order books  
✅ How to detect arbitrage opportunities  

---

## Next Steps

1. **[Market Types](./03-market-types.md)** - Understand different market structures
2. **[Trading Guide](./06-trading-orders.md)** - Learn how to place orders
3. **[Bot Strategies](./07-bot-strategies.md)** - Build more advanced bots
4. **[WebSocket Streaming](./05-websocket-streaming.md)** - Get real-time data

---

## Common Issues

### "Module not found" error
```bash
pip install py-clob-client python-dotenv
```

### "Private key invalid" error
- Check your private key starts with `0x`
- Ensure there are no spaces or quotes

### "API credentials not set" error
- Run `generate_credentials.py` first
- Add credentials to `.env` file
- Call `client.set_api_creds()` before trading

---

**Ready to trade?** See [Trading & Orders Guide](./06-trading-orders.md) →
