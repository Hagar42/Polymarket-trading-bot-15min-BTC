# Authentication Guide

> Learn how to authenticate with the Polymarket CLOB API and generate API credentials

## Overview

Polymarket uses a two-layer authentication system:

1. **Wallet Signature** - Uses your private key to sign messages (EIP-712)
2. **API Credentials** - API Key, Secret, and Passphrase derived from your wallet

This guide covers all authentication methods supported by Polymarket.

---

## Wallet Types & Signature Types

Polymarket supports three signature types:

| Signature Type | Value | Description | Use Case |
|----------------|-------|-------------|----------|
| **EOA** | `0` | Externally Owned Account | MetaMask, hardware wallets, standard private keys |
| **POLY_PROXY** | `1` | Polymarket Proxy | Magic.link email login accounts |
| **POLY_GNOSIS_SAFE** | `2` | Gnosis Safe | Multi-sig wallets |

### How to Determine Your Signature Type

- **Using MetaMask or a hardware wallet?** → Use `signature_type=0` (EOA)
- **Login with email on Polymarket?** → Use `signature_type=1` (POLY_PROXY)
- **Using a Gnosis Safe multi-sig?** → Use `signature_type=2` (POLY_GNOSIS_SAFE)

---

## Authentication Flow

```
┌──────────────────┐
│  Private Key     │
│  (Your Wallet)   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Derive/Create    │
│ API Credentials  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  API Key         │
│  API Secret      │
│  Passphrase      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Initialize      │
│  CLOB Client     │
└──────────────────┘
```

---

## Python Implementation

### 1. Basic Authentication (EOA Wallet)

```python
from py_clob_client.client import ClobClient

# Configuration
HOST = "https://clob.polymarket.com"
CHAIN_ID = 137  # Polygon mainnet
PRIVATE_KEY = "0xYOUR_PRIVATE_KEY_HERE"

# Create client
client = ClobClient(
    host=HOST,
    key=PRIVATE_KEY,
    chain_id=CHAIN_ID
)

# Generate API credentials
api_creds = client.create_or_derive_api_creds()

# Configure client with API credentials
client.set_api_creds(api_creds)

# Test connection
print(client.get_ok())  # Should return {'status': 'ok'}
```

### 2. Magic.link / Proxy Wallet Authentication

For Magic.link accounts (email login), you need to specify:
- `signature_type=1`
- Your **funder address** (proxy wallet address)

```python
from py_clob_client.client import ClobClient

HOST = "https://clob.polymarket.com"
CHAIN_ID = 137
PRIVATE_KEY = "0xYOUR_SIGNER_PRIVATE_KEY"
FUNDER = "0xYOUR_PROXY_WALLET_ADDRESS"  # Important!

client = ClobClient(
    host=HOST,
    key=PRIVATE_KEY,
    chain_id=CHAIN_ID,
    signature_type=1,  # Magic.link/Proxy
    funder=FUNDER
)

api_creds = client.create_or_derive_api_creds()
client.set_api_creds(api_creds)
```

#### Finding Your Proxy Wallet Address (FUNDER)

1. Go to your Polymarket profile: `https://polymarket.com/@YOUR_USERNAME`
2. Click **"Copy address"** button next to your balance
3. This is your `FUNDER` address (different from your signer address)

**Common mistake:** Using your Polygon wallet address instead of Polymarket proxy address causes `"invalid signature"` errors.

### 3. Using Environment Variables (Recommended)

```python
import os
from py_clob_client.client import ClobClient
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

HOST = "https://clob.polymarket.com"
CHAIN_ID = 137

client = ClobClient(
    host=HOST,
    key=os.getenv("POLYMARKET_PRIVATE_KEY"),
    chain_id=CHAIN_ID,
    signature_type=int(os.getenv("POLYMARKET_SIGNATURE_TYPE", "0")),
    funder=os.getenv("POLYMARKET_FUNDER", None)
)

# Restore existing credentials or create new ones
api_key = os.getenv("POLYMARKET_API_KEY")
api_secret = os.getenv("POLYMARKET_API_SECRET")
api_passphrase = os.getenv("POLYMARKET_API_PASSPHRASE")

if api_key and api_secret and api_passphrase:
    # Use existing credentials
    client.set_api_creds({
        "apiKey": api_key,
        "secret": api_secret,
        "passphrase": api_passphrase
    })
else:
    # Generate new credentials
    api_creds = client.create_or_derive_api_creds()
    client.set_api_creds(api_creds)
    
    # Print to save in .env file
    print(f"POLYMARKET_API_KEY={api_creds['apiKey']}")
    print(f"POLYMARKET_API_SECRET={api_creds['secret']}")
    print(f"POLYMARKET_API_PASSPHRASE={api_creds['passphrase']}")
```

### 4. Read-Only Access (No Authentication)

For public data (markets, order books, prices), you don't need authentication:

```python
from py_clob_client.client import ClobClient

client = ClobClient("https://clob.polymarket.com")

# Public endpoints work without auth
ok = client.get_ok()
server_time = client.get_server_time()
markets = client.get_markets()
```

---

## TypeScript Implementation

### 1. Basic Authentication (EOA Wallet)

```typescript
import { ClobClient } from "@polymarket/clob-client";
import { Wallet } from "ethers";

const HOST = "https://clob.polymarket.com";
const CHAIN_ID = 137;
const privateKey = process.env.PRIVATE_KEY!;

// Create signer
const signer = new Wallet(privateKey);

// Create temporary client to generate credentials
const tempClient = new ClobClient(HOST, CHAIN_ID, signer);
const apiCreds = await tempClient.createOrDeriveApiKey();

// Create authenticated client
const client = new ClobClient(
  HOST,
  CHAIN_ID,
  signer,
  apiCreds,
  0  // signature_type: 0 = EOA
);

// Test connection
const ok = await client.getOk();
console.log(ok);
```

### 2. Magic.link Authentication

```typescript
import { ClobClient } from "@polymarket/clob-client";
import { Wallet } from "ethers";

const HOST = "https://clob.polymarket.com";
const CHAIN_ID = 137;
const privateKey = process.env.PRIVATE_KEY!;
const funder = process.env.FUNDER_ADDRESS!;

const signer = new Wallet(privateKey);
const tempClient = new ClobClient(HOST, CHAIN_ID, signer);
const apiCreds = await tempClient.createOrDeriveApiKey();

const client = new ClobClient(
  HOST,
  CHAIN_ID,
  signer,
  apiCreds,
  1,  // signature_type: 1 = Magic.link
  funder
);
```

---

## API Credentials Management

### Generate New Credentials

**Python:**
```python
api_creds = client.create_or_derive_api_creds()
print(api_creds)
# {
#   'apiKey': 'abc123...',
#   'secret': 'xyz789...',
#   'passphrase': 'mypassphrase'
# }
```

**TypeScript:**
```typescript
const apiCreds = await client.createOrDeriveApiKey();
console.log(apiCreds);
```

### Store Credentials Securely

**Best practices:**
1. ✅ Store in environment variables (`.env` file)
2. ✅ Never commit credentials to Git
3. ✅ Use secrets management (AWS Secrets Manager, HashiCorp Vault, etc.)
4. ✅ Rotate credentials periodically
5. ❌ Don't hardcode in source code
6. ❌ Don't share credentials

**Example `.env` file:**
```env
POLYMARKET_PRIVATE_KEY=0x...
POLYMARKET_SIGNATURE_TYPE=1
POLYMARKET_FUNDER=0x...
POLYMARKET_API_KEY=abc123...
POLYMARKET_API_SECRET=xyz789...
POLYMARKET_API_PASSPHRASE=mypassphrase
```

### Credential Regeneration

⚠️ **Important:** API credentials are deterministically derived from your private key. If you:
- Change your private key → Must regenerate credentials
- Change signature_type → Must regenerate credentials
- Change funder address → Must regenerate credentials

---

## Testing Your Authentication

### Python Test Script

```python
from py_clob_client.client import ClobClient
import os
from dotenv import load_dotenv

load_dotenv()

def test_authentication():
    """Test Polymarket authentication"""
    print("Testing Polymarket Authentication...")
    print("=" * 50)
    
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
    
    # Test 1: Basic connectivity
    print("\n1. Testing connectivity...")
    ok = client.get_ok()
    print(f"   Status: {ok}")
    
    # Test 2: Get wallet address
    print("\n2. Getting wallet address...")
    address = client.get_address()
    print(f"   Address: {address}")
    
    # Test 3: Get balance
    print("\n3. Getting USDC balance...")
    balance = client.get_balance_allowance()
    print(f"   Balance: ${balance['balance']}")
    print(f"   Allowance: ${balance['allowance']}")
    
    print("\n" + "=" * 50)
    print("✅ Authentication successful!")
    
if __name__ == "__main__":
    test_authentication()
```

---

## Common Issues & Solutions

### Issue 1: "Invalid signature" error

**Symptoms:**
```
Error: invalid signature
```

**Solutions:**
1. ✅ Verify `signature_type` matches your wallet type
2. ✅ For Magic.link (type=1): ensure `FUNDER` is set correctly
3. ✅ Regenerate API credentials with correct settings
4. ✅ Check that private key matches the wallet you're using

### Issue 2: "API key not found" error

**Symptoms:**
```
Error: API key not found
```

**Solutions:**
1. ✅ Call `create_or_derive_api_creds()` before trading
2. ✅ Call `set_api_creds()` with the credentials
3. ✅ Check API credentials are correctly formatted

### Issue 3: Balance shows $0 but I have funds

**Symptoms:**
- API shows balance: $0
- Polymarket website shows balance: $100

**Solutions:**
1. ✅ For Magic.link: verify `FUNDER` address is your Polymarket proxy address
2. ✅ Check you're using the correct private key
3. ✅ Ensure funds are on Polygon (not Ethereum mainnet)
4. ✅ Verify USDC is approved for the CTF Exchange contract

### Issue 4: Rate limiting

**Symptoms:**
```
Error: Too many requests
```

**Solutions:**
1. ✅ Reduce request frequency
2. ✅ Implement exponential backoff
3. ✅ Use WebSocket for real-time data instead of polling
4. ✅ Rate limits: 100 req/min (public), 60 orders/min (trading)

---

## Security Best Practices

### 1. Private Key Security

```python
# ✅ Good: Use environment variables
private_key = os.getenv("POLYMARKET_PRIVATE_KEY")

# ❌ Bad: Hardcode in source
private_key = "0x1234..."  # Never do this!
```

### 2. Credential Rotation

```python
# Rotate credentials periodically
def rotate_credentials(client):
    """Generate new API credentials"""
    new_creds = client.create_or_derive_api_creds()
    client.set_api_creds(new_creds)
    
    # Save new credentials securely
    # update_env_file(new_creds)
    
    return new_creds
```

### 3. Separate Wallets for Trading

- Use a dedicated wallet for bot trading
- Don't use your main wallet with large holdings
- Start with small amounts for testing

### 4. API Credential Scope

- API credentials have full trading access
- Treat them like passwords
- Don't share or expose them

---

## Next Steps

Now that you're authenticated, you can:

1. **[Quick Start](./02-quickstart.md)** - Place your first order
2. **[API Reference](./04-api-reference.md)** - Explore all API endpoints
3. **[Trading Guide](./06-trading-orders.md)** - Learn about order types

---

**Questions?** See [Troubleshooting](./10-troubleshooting.md) for more help.
