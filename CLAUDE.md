# Bitcoin UTXO Explorer - Project Context & Next Steps

## Project Overview
A web application that searches Bitcoin addresses for UTXOs and displays them with links to a local Ord server for rare sat analysis.

## Current Status: ✅ WORKING BASE IMPLEMENTATION

### What Works Now
1. **Address Validation** - Validates Legacy, SegWit, and Taproot Bitcoin addresses
2. **Bitcoin Core RPC Connection** - Connects via CORS proxy to local Bitcoin Core node
3. **UTXO Discovery** - Scans addresses using `scantxoutset` RPC call
4. **Clean UTXO Display** - Shows results in a scrollable table with:
   - UTXO ID (txid:vout)
   - Amount in sats and BTC
   - Clickable links to Ord server web interface

### Infrastructure Setup

#### Local Network Configuration
- **Bitcoin Core Node**: 192.168.0.25:8332
- **Ord Server**: http://192.168.0.25 (runs on default port 80)
- **CORS Proxy**: 192.168.0.25:3001 (required for browser to access Bitcoin Core RPC)

#### Bitcoin Core RPC Credentials
- **User**: claudeRocks
- **Password**: iGgPibddTUnBzq1NjBUMJDOuFvxQYypNDd64ZhFANA0
- **Config**: Set via `rpcauth` in bitcoin.conf

#### Ord Server Startup Command
```bash
sudo -E /home/mclean/bin/ord --index /media/mclean/Nodes/ord/index.redb \
  --index-runes --index-sats --index-transactions server
```

**Note**: Currently does NOT have `--index-addresses` enabled (would require days of re-indexing)

### Required Files

#### 1. bitcoin-cors-proxy.js (on Ubuntu at 192.168.0.25)
Located at: `/home/mclean/bitcoin-cors-proxy.js`

```javascript
const express = require('express');
const cors = require('cors');

const app = express();
const PORT = 3001;

const BITCOIN_RPC_CONFIG = {
    host: 'localhost',
    port: 8332,
    user: 'claudeRocks',
    password: 'iGgPibddTUnBzq1NjBUMJDOuFvxQYypNDd64ZhFANA0'
};

const ORD_CONFIG = {
    host: '192.168.0.25'
};

app.use(cors({
    origin: '*',
    methods: ['GET', 'POST', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization']
}));

app.use(express.json());

app.get('/health', (req, res) => {
    res.json({ 
        status: 'ok', 
        timestamp: new Date().toISOString(),
        bitcoin_rpc: `${BITCOIN_RPC_CONFIG.host}:${BITCOIN_RPC_CONFIG.port}`
    });
});

app.post('/rpc', async (req, res) => {
    try {
        const { method, params = [], id = 1 } = req.body;
        
        if (!method) {
            return res.status(400).json({ error: 'Missing method parameter' });
        }

        console.log(`RPC Call: ${method}`);

        const rpcRequest = {
            jsonrpc: '2.0',
            id: id,
            method: method,
            params: params
        };

        const auth = Buffer.from(`${BITCOIN_RPC_CONFIG.user}:${BITCOIN_RPC_CONFIG.password}`).toString('base64');
        const fetch = (await import('node-fetch')).default;
        
        const response = await fetch(`http://${BITCOIN_RPC_CONFIG.host}:${BITCOIN_RPC_CONFIG.port}`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Basic ${auth}`
            },
            body: JSON.stringify(rpcRequest)
        });

        if (!response.ok) {
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }

        const data = await response.json();
        console.log(`✅ RPC ${method} completed`);
        res.json(data);

    } catch (error) {
        console.error('RPC Error:', error.message);
        res.status(500).json({
            error: { message: error.message, code: -1 }
        });
    }
});

app.listen(PORT, '0.0.0.0', () => {
    console.log(`🚀 Bitcoin CORS Proxy running on port ${PORT}`);
});

process.on('SIGINT', () => {
    console.log('\n👋 Shutting down...');
    process.exit(0);
});
```

**To run the proxy:**
```bash
cd /home/mclean
node bitcoin-cors-proxy.js
```

#### 2. Main HTML Application
The working HTML file (v30) serves the web interface via:
```bash
# From OSX machine in the directory with the HTML file:
python3 -m http.server 8080
# Then access: http://localhost:8080/rare-sats-finder.html
```

**Important**: Must be served via HTTP (not file://) for browser security reasons.

---

## Known Limitations & Issues

### 1. UTXO Discovery is SLOW ⏳
**Problem**: Using `scantxoutset` RPC call scans the entire UTXO set (~150M UTXOs), taking 30-60 seconds per address.

**Why**: Bitcoin Core doesn't index by address by default.

**Solutions** (in order of preference):
- **Option A**: Enable `--index-addresses` on Ord and use Ord's address API (requires days of re-indexing)
- **Option B**: Import addresses as "watch-only" in Bitcoin Core for instant `listunspent` lookups
- **Option C**: Accept the slow scan times (current state)

### 2. Rare Sat Detection NOT Implemented ❌
**Problem**: The app doesn't actually detect rare sats - it only lists UTXOs.

**What Was Attempted**: 
- Tried to use Ord's JSON API endpoints
- Attempted `/output/{txid}:{vout}` endpoint with `Accept: application/json` header
- Endpoints exist but kept returning HTML instead of JSON or we couldn't figure out correct format

**What's Needed**: 
- Correct Ord API usage to get sat ranges and rarity data for each UTXO
- Reference: https://docs.ordinals.com/guides/api.html

**Current Workaround**: Users click the "🔍 View" link to see each UTXO in Ord's web interface manually.

### 3. Ord API Format Confusion 🤔
The Ord documentation says to use `Accept: application/json` header to get JSON responses, but:
- We tried many endpoint variations
- Kept getting HTML responses instead of JSON
- May be a version issue or incorrect endpoint format
- The working curl command was: `curl -s -H "Accept: application/json" http://192.168.0.25:80/output/{txid}:{vout}`

---

## Next Steps for Completion

### Priority 1: Implement Rare Sat Detection 🔥

**Goal**: Automatically detect and display rare sats within each UTXO using Ord's API.

**What to investigate**:
1. **Verify Ord API availability**
   ```bash
   # Test these commands from the Ubuntu machine:
   curl -H "Accept: application/json" http://192.168.0.25/blockheight
   curl -H "Accept: application/json" http://192.168.0.25/output/bc4c30829a9564c0d58e6287195622b53ced54a25711d1b86be7cd3a70ef61ed:0
   ```

2. **Understand the correct API response format**
   - What fields does `/output/{txid}:{vout}` return?
   - How are sat ranges encoded?
   - How to get rarity for individual sats?

3. **Implement in the app**:
   - After getting UTXOs, query each one via Ord API
   - Parse sat ranges from response
   - Determine rarity for sats in those ranges
   - Display rare sats prominently in the results

**Questions to Answer**:
- Does the Ord server's JSON API actually work with the `Accept: application/json` header?
- What does a successful `/output/{txid}:{vout}` JSON response look like?
- How do we map sat ranges to rarity levels (mythic, legendary, epic, rare, uncommon, vintage)?
- Should we call Ord directly from the browser or route through the CORS proxy?

### Priority 2: Improve UTXO Discovery Speed ⚡

**Option 2A**: Use Ord's address API (IF `--index-addresses` gets enabled later)
```javascript
// Would replace slow scantxoutset with:
const response = await fetch(`http://192.168.0.25/address/${address}`, {
    headers: { 'Accept': 'application/json' }
});
const data = await response.json();
// Returns: { outputs: [...], inscriptions: [...], runes_balances: [...] }
```

**Option 2B**: Import addresses as watch-only (can do now)
```javascript
// Before scanning, import the address:
await callBitcoinRPC('importaddress', [address, '', false]);
// Then use fast listunspent instead of scantxoutset
```

**Question**: Which approach is preferred? Trade-offs?

### Priority 3: UI/UX Improvements 🎨

**Potential enhancements**:
- Add loading spinner during slow UTXO scans
- Progress bar for multi-address scans
- Filter/sort UTXOs by value or rarity
- Export results to CSV/JSON
- Save frequent addresses for quick re-scan
- Display inscriptions and runes data alongside UTXOs
- Show estimated time remaining during scans

---

## Development Workflow

### Testing Locally
1. **Start CORS proxy** on Ubuntu: `node bitcoin-cors-proxy.js`
2. **Serve HTML** on OSX: `python3 -m http.server 8080`
3. **Access app**: http://localhost:8080/rare-sats-finder.html
4. **Check console** (F12) for debugging info

### Key Functions to Understand

**Bitcoin Core Integration**:
- `callBitcoinRPC(method, params)` - Makes RPC calls via proxy
- `getUTXOsForAddress(address)` - Gets UTXOs (tries fast then slow method)
- `searchRareSatsForAddresses(addresses)` - Main scan orchestrator

**UI Functions**:
- `validateAddresses()` - Client-side address validation
- `displayUTXOResults(addressUTXOs, nodeInfo)` - Renders the UTXO table
- `showStatus(message, type)` - Updates status display

### Debugging Tips
- **CORS errors**: Proxy not running or wrong URL
- **RPC errors**: Bitcoin Core not running or wrong credentials  
- **Ord errors**: Check Ord server is running and accessible
- **File protocol errors**: Must serve via HTTP, not open file directly
- **HTML instead of JSON**: Check Accept header and endpoint format

---

## Resources

### Documentation
- **Ord API Guide**: https://docs.ordinals.com/guides/api.html
- **Bitcoin Core RPC**: https://developer.bitcoin.org/reference/rpc/
- **Ordinal Theory**: https://docs.ordinals.com/

### Testing Addresses
Sample addresses with known UTXOs:
- `bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh` - SegWit address
- `1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa` - Genesis block address (famous, 57K+ UTXOs)
- `3J98t1WpEZ73CNmQviecrnyiWrnqRhWNLy` - P2SH address

---

## Questions for Claude Code Web Session

When resuming this project, please help me:

1. **Ord API Integration**:
   - Can you help test the Ord API endpoints and understand the response format?
   - How do we properly parse sat ranges and determine rarity?
   - Should we add the `/output` endpoint to the CORS proxy or call Ord directly from browser?
   - Why were we getting HTML instead of JSON from Ord API?

2. **Performance**:
   - What's the best approach to speed up UTXO discovery given our constraints?
   - Should we implement address importing or wait for `--index-addresses`?
   - Can we cache results to avoid re-scanning?

3. **Architecture**:
   - Is the current single-file HTML approach still appropriate, or should we split into separate files?
   - Should we add any caching or persistence (localStorage)?
   - Should we create a proper backend API instead of direct RPC calls?

4. **Next Features**:
   - What should we prioritize: rare sat detection, speed improvements, or UI enhancements?
   - Any other important features we're missing?
   - Should we add support for batch operations?

---

## File Structure

```
project/
├── CLAUDE.md                    # This context file
├── bitcoin-cors-proxy.js        # On Ubuntu at /home/mclean/
├── rare-sats-finder.html        # Main application (current working version v30)
└── README.md                    # (Optional) User-facing documentation
```

---

## Success Criteria

The project will be "complete" when:
- ✅ Users can enter Bitcoin addresses
- ✅ App scans for UTXOs (working, but slow)
- ❌ App automatically detects rare sats in each UTXO (NOT WORKING - TOP PRIORITY)
- ❌ App displays rare sats with rarity levels and details (NOT WORKING)
- ✅ Users can click to view UTXOs in Ord web interface (working)
- ⚠️  Performance is acceptable for typical usage (<10 addresses, <1000 UTXOs)

---

## Development Notes

### What Worked Well ✅
- Incremental testing approach (step by step)
- CORS proxy solution for Bitcoin Core RPC
- Clean separation between Bitcoin Core and Ord server
- Simple single-file HTML for easy deployment
- Clean table UI with direct Ord links

### What Didn't Work ❌
- Multiple attempts to guess Ord API endpoints
- Trying to implement rare sat detection without proper API documentation
- Overly complex rarity detection logic
- Making assumptions about API formats without testing

### Lessons Learned 💡
- **Always verify API endpoints work with curl first** before implementing in code
- **Don't make assumptions** about API formats - test with real responses
- **User can click through to Ord UI** as temporary workaround is actually fine
- **Slow is better than broken** - scantxoutset works, even if it's slow
- **Document everything** - this context file should have been created earlier!

---

## Contact & Context

**User Environment**:
- **Development**: OSX machine
- **Bitcoin Infrastructure**: Ubuntu server at 192.168.0.25
- **User preference**: Incremental development, test each step before moving forward
- **Language**: Canadian English spelling

**User's Expertise**:
- Understands Bitcoin Core and Ord concepts
- Comfortable with command line and basic networking
- Prefers working solutions over theoretical perfection
- Values clear documentation and step-by-step approach

---

## Current Working Version Details

**Version**: v30
**Status**: ✅ Functional UTXO Explorer
**Last Updated**: November 2024

**What's Implemented**:
- Full address validation
- Bitcoin Core RPC integration via CORS proxy
- UTXO discovery (slow but working)
- Clean scrollable table display
- Direct links to Ord server for each UTXO
- Error handling and status messages
- Multi-address support

**What's Missing**:
- Automated rare sat detection
- Performance optimization
- Loading indicators
- Result caching

---

*This document is meant to be shared with Claude Code Web to continue the project seamlessly.*