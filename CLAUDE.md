# SAT-HUNTER - Bitcoin UTXO Explorer

## Quick Reference

**Repository**: sat-hunter
**Main File**: `index.html` (single-page application)
**License**: Included in repository
**Status**: ✅ Working base implementation
**Primary Goal**: Scan Bitcoin addresses for UTXOs and analyze rare satoshis

### Quick Start
```bash
# 1. Clone repository
git clone <repo-url> && cd sat-hunter

# 2. Start CORS proxy on Bitcoin node (192.168.0.25)
node bitcoin-cors-proxy.js

# 3. Serve the application
python3 -m http.server 8080

# 4. Open browser
# http://localhost:8080/
```

### Key Configuration
- **Bitcoin Node**: 192.168.0.25:8332 (via CORS proxy at :3001)
- **Ord Server**: http://192.168.0.25
- **Configuration**: Edit CONFIG object in `index.html` (lines 168-173)

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Current Status & Features](#current-status--working-base-implementation)
3. [Infrastructure Setup](#infrastructure-setup)
4. [Architecture & Code Organization](#architecture--code-organization)
5. [Known Limitations & Issues](#known-limitations--issues)
6. [Next Steps for Completion](#next-steps-for-completion)
7. [Development Workflow](#development-workflow)
8. [File Structure](#file-structure)
9. [Resources](#resources)
10. [Guidelines for AI Assistants](#guidelines-for-ai-assistants)

---

## Project Overview

A web application that searches Bitcoin addresses for UTXOs (Unspent Transaction Outputs) and displays them with links to a local Ord server for rare satoshi analysis. This is a single-page web application designed to help Bitcoin users explore their addresses and discover potentially rare or valuable satoshis.

**Key Features**:
- Multi-address UTXO scanning
- Address validation (Legacy, SegWit, Taproot)
- Integration with Bitcoin Core via RPC
- Direct links to Ord server for manual rare sat analysis
- Clean, responsive web interface

**Technology Stack**:
- Vanilla JavaScript (ES6+)
- HTML5 & CSS3
- Bitcoin Core RPC API
- Ordinals (Ord) Server API

---

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
The main application file is `index.html` which serves the web interface. To run locally:
```bash
# From the repository directory:
python3 -m http.server 8080
# Then access: http://localhost:8080/index.html (or just http://localhost:8080/)
```

**Important**: Must be served via HTTP (not file://) for browser security reasons due to CORS requirements when accessing the Bitcoin Core RPC proxy.

---

## Architecture & Code Organization

### Application Structure
The application is a **single-page HTML file** with embedded CSS and JavaScript. This design choice prioritizes:
- **Simplicity**: Easy to deploy, no build process required
- **Portability**: Single file can be served from any web server
- **Transparency**: All code is visible and auditable in one place

### Code Sections in index.html
1. **HTML Structure (lines 1-165)**:
   - Semantic HTML5 markup
   - Configuration display panel
   - Input form for addresses
   - Status display area (dynamically updated)

2. **CSS Styles (lines 7-134)**:
   - Embedded in `<style>` tag
   - Bitcoin-themed colors (orange #f7931a)
   - Responsive design principles
   - Status message color coding (success/error/info)

3. **JavaScript Application (lines 166-587)**:
   - Configuration constants
   - Address validation logic
   - Bitcoin Core RPC client
   - UTXO scanning and display logic
   - Proxy connection testing

### Key Design Patterns

**Configuration Management** (lines 168-173):
```javascript
const CONFIG = {
    BITCOIN_RPC_URL: 'http://192.168.0.25:3001/rpc',  // CORS proxy
    ORD_API_URL: 'http://192.168.0.25',               // Ord server
    HEALTH_URL: 'http://192.168.0.25:3001/health',    // Health check
};
```
- Centralized configuration for easy updates
- All network endpoints defined in one place

**Async/Await Pattern**:
- All RPC calls use async/await for clean asynchronous code
- Error handling with try/catch blocks
- Sequential processing of multiple addresses with progress updates

**Dual-Method UTXO Discovery** (lines 381-408):
```javascript
async function getUTXOsForAddress(address) {
    try {
        // Try fast method first (listunspent)
        // Falls back to slow method (scantxoutset) if fast method fails
    }
}
```
- Optimistic approach: try fast method first
- Graceful degradation: fall back to slow but reliable method
- Console logging for debugging

### Code Conventions

**Naming Conventions**:
- Functions: camelCase (e.g., `validateAddresses`, `callBitcoinRPC`)
- Constants: UPPER_SNAKE_CASE (e.g., `CONFIG`, `SAMPLE_ADDRESSES`)
- DOM IDs: lowercase (e.g., `addresses`, `status`, `nodeStatus`)

**Error Handling Philosophy**:
- Never fail silently - always inform the user
- Provide actionable troubleshooting guidance
- Log detailed errors to console for debugging
- Use color-coded status messages (green/red/yellow)

**User Feedback**:
- Immediate validation feedback
- Progress updates during long operations
- Clear success/error states
- Links to external resources (Ord server)

### Security Considerations

**No Sensitive Data in Frontend**:
- RPC credentials stored only in CORS proxy (server-side)
- Frontend only knows proxy endpoint, not Bitcoin Core credentials
- No private keys ever handled or displayed

**CORS Proxy Requirement**:
- Browsers block direct RPC calls to Bitcoin Core (CORS policy)
- Proxy server adds CORS headers to enable browser access
- Proxy validates requests before forwarding to Bitcoin Core

**Network Architecture**:
```
Browser (index.html)
    ↓ HTTP (CORS allowed)
CORS Proxy (192.168.0.25:3001)
    ↓ RPC with auth
Bitcoin Core (192.168.0.25:8332)
```

### Detailed Code Map

**Configuration & Constants** (lines 166-180):
- `CONFIG`: Network endpoints (Bitcoin RPC, Ord API, Health check)
- `SAMPLE_ADDRESSES`: Pre-populated test addresses

**UI Functions** (lines 182-213):
- `showStatus(message, type)`: Display status messages with color coding
- `clearAll()`: Reset form and clear results
- `validateAddresses()`: Entry point for address validation flow
- `displayValidationResults(results)`: Render validation results with action buttons

**Address Validation** (lines 215-233):
- `validateBitcoinAddress(address)`: Regex-based validation for Legacy/SegWit/Taproot
- Returns: `{ valid: boolean, type: string }`

**UTXO Scanning** (lines 271-351):
- `searchRareSats()`: Entry point triggered by "Search for UTXOs" button
- `searchRareSatsForAddresses(addresses)`: Main orchestrator
  - Tests Bitcoin Core connection
  - Iterates through addresses
  - Updates progress in real-time
  - Collects all UTXOs
  - Displays final results

**Bitcoin Core RPC** (lines 353-408):
- `callBitcoinRPC(method, params)`: Generic RPC wrapper
  - Handles JSON-RPC formatting
  - Error handling and reporting
  - Returns: `Promise<result>`

- `getUTXOsForAddress(address)`: Dual-method UTXO discovery
  - **Fast path**: `listunspent` (works if address is imported/watched)
  - **Slow path**: `scantxoutset` (always works, 30-60 sec)
  - Returns: Array of UTXO objects `[{ txid, vout, amount, height }]`

**Results Display** (lines 410-503):
- `displayUTXOResults(addressUTXOs, nodeInfo)`: Renders UTXO table
  - Generates HTML for each address section
  - Creates scrollable table with columns: #, UTXO, Sats, BTC, Ord Link
  - Shows summary statistics
  - Handles error states per address

**Connection Testing** (lines 505-579):
- `checkProxyConnection()`: Tests CORS proxy health
  - Tries multiple fetch configurations
  - Detects and reports CORS issues
  - Warns about file:// protocol
  - Updates status indicator in UI

**Initialization** (lines 581-587):
- `window.onload`: Runs on page load
  - Tests proxy connection automatically
  - Pre-fills sample addresses
  - Shows initial instructions

### Critical Functions for Modification

**To add Ord API integration**:
- Modify `displayUTXOResults()` to add rare sat detection
- Add new function `getRareSatsForUTXO(txid, vout)` to query Ord API
- Update UTXO table to show rarity information

**To improve performance**:
- Modify `getUTXOsForAddress()` to import addresses before scanning
- Add caching layer (localStorage) in `searchRareSatsForAddresses()`
- Implement parallel scanning instead of sequential

**To add export functionality**:
- Add export button in `displayUTXOResults()`
- Create new function `exportResults(format)` to generate CSV/JSON

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

### Git Repository Structure
This is a git-based project. The repository contains:
- `index.html` - Main application (single-page app with embedded CSS/JS)
- `CLAUDE.md` - This context file for AI assistants
- `LICENSE` - Project license
- `.git/` - Git version control

### Setting Up Development Environment
1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd sat-hunter
   ```

2. **Set up Bitcoin Core with CORS proxy** on your Bitcoin node machine (e.g., Ubuntu at 192.168.0.25):
   - Ensure Bitcoin Core is running with RPC enabled
   - Deploy and run `bitcoin-cors-proxy.js` (see Required Files section above)
   - Ensure Ord server is running if you want rare sat analysis

3. **Update configuration** in `index.html` if needed:
   - Edit the `CONFIG` object (lines 168-173) to point to your Bitcoin node IP
   - Default configuration assumes node at `192.168.0.25`

### Testing Locally
1. **Start CORS proxy** on your Bitcoin node (e.g., Ubuntu): `node bitcoin-cors-proxy.js`
2. **Serve HTML** from repository directory: `python3 -m http.server 8080`
3. **Access app**: http://localhost:8080/ (or http://localhost:8080/index.html)
4. **Check console** (F12 in browser) for debugging info

### Git Workflow
- **Main development**: Work on feature branches
- **Testing**: Test changes locally before committing
- **Commits**: Use descriptive commit messages
- **Branches**: Follow the branch naming convention (e.g., `claude/feature-name-sessionid`)

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

### Repository Files
```
sat-hunter/
├── .git/                        # Git version control
├── .DS_Store                    # macOS metadata (should be .gitignored)
├── index.html                   # Main application - single-page web app
├── CLAUDE.md                    # This context file for AI assistants
└── LICENSE                      # Project license (674 lines)
```

### External Dependencies
```
Ubuntu Server (192.168.0.25):
├── /home/mclean/bitcoin-cors-proxy.js    # CORS proxy for Bitcoin Core RPC
├── Bitcoin Core (localhost:8332)          # Bitcoin node with RPC enabled
└── Ord Server (port 80)                   # Ordinals indexer and explorer
```

**Note**: The CORS proxy and Bitcoin infrastructure are deployed separately on an Ubuntu server. Only the web application (`index.html`) is in this repository.

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

## Current Implementation Status

**Status**: ✅ Functional UTXO Explorer (Base Implementation Complete)
**File**: index.html
**Last Major Update**: November 2024

### Implemented Features ✅
- **Address Validation**: Client-side validation for Legacy, SegWit, and Taproot addresses
- **Bitcoin Core RPC Integration**: Via CORS proxy for secure browser-to-node communication
- **UTXO Discovery**: Dual-method approach (fast `listunspent` fallback to slow `scantxoutset`)
- **Multi-Address Support**: Scan multiple addresses in a single session
- **Clean UI**: Responsive design with scrollable tables and color-coded results
- **Direct Ord Links**: Each UTXO links to Ord server web interface for manual rare sat inspection
- **Connection Testing**: Built-in proxy health check functionality
- **Error Handling**: Comprehensive error messages and troubleshooting guidance
- **Sample Data**: Pre-populated with test addresses for quick demos

### Not Yet Implemented ❌
- **Automated Rare Sat Detection**: API integration with Ord server for automatic rarity analysis
- **Performance Optimization**: Address importing or caching to speed up repeated scans
- **Loading Indicators**: Progress bars or spinners during long scans
- **Result Persistence**: localStorage caching of scan results
- **Export Functionality**: CSV/JSON export of results
- **Batch Operations**: Optimized handling of large address sets

---

## Guidelines for AI Assistants

When working on this project, follow these guidelines:

### Development Approach
1. **Incremental Changes**: Make small, testable changes rather than large refactors
2. **Test First**: Always verify APIs and endpoints work before implementing features
3. **Preserve Working Code**: Never break existing functionality when adding features
4. **Document Assumptions**: Clearly state any assumptions about APIs or behavior

### Code Modification Rules
1. **Single-File Architecture**: Keep everything in `index.html` unless absolutely necessary to split
2. **Maintain Inline Styles**: Keep CSS in the `<style>` tag for portability
3. **Preserve Comments**: Keep existing comments and add new ones for complex logic
4. **Configuration Clarity**: Always use the CONFIG object, never hardcode endpoints

### When Adding Features
1. **Read Before Writing**: Always read the full `index.html` file before making changes
2. **Understand Context**: Review this CLAUDE.md file to understand project history
3. **Test Incrementally**: Test each change in isolation before moving to the next
4. **Update Documentation**: Update this CLAUDE.md file if adding major features

### Common Tasks

**Adding a New RPC Call**:
```javascript
// Use the existing callBitcoinRPC function:
const result = await callBitcoinRPC('method_name', [param1, param2]);
```

**Adding Ord API Integration**:
```javascript
// Direct fetch to Ord server (CORS is enabled):
const response = await fetch(`${CONFIG.ORD_API_URL}/endpoint`, {
    headers: { 'Accept': 'application/json' }
});
```

**Displaying Results**:
```javascript
// Use the showStatus function with type: 'success', 'error', or 'info':
showStatus('<h3>Title</h3><p>Message</p>', 'success');
```

### Testing Checklist
Before committing changes, verify:
- [ ] File protocol detection still works (shows helpful error)
- [ ] Proxy connection test button functions
- [ ] Address validation works for all types (Legacy/SegWit/Taproot)
- [ ] UTXO scanning completes without errors
- [ ] Results display correctly in the table
- [ ] Ord links are properly formatted
- [ ] Error messages are clear and actionable
- [ ] Console has no unexpected errors (check browser DevTools)

### Git Practices
- **Branch Naming**: Use descriptive names (e.g., `feature/add-rare-sat-detection`)
- **Commit Messages**: Clear, descriptive messages explaining "what" and "why"
- **Small Commits**: Commit logical units of work, not massive changes
- **Test Before Push**: Always test locally before pushing

### What NOT to Do
- ❌ Don't add dependencies or frameworks (keep it vanilla JavaScript)
- ❌ Don't split into multiple files unless absolutely necessary
- ❌ Don't remove the CORS proxy (browsers require it for RPC)
- ❌ Don't hardcode IP addresses (use CONFIG object)
- ❌ Don't implement features without testing APIs first
- ❌ Don't commit credentials or sensitive data
- ❌ Don't break backward compatibility with existing setup

### Communication with User
When working on this project:
- Ask clarifying questions before implementing ambiguous features
- Explain trade-offs when multiple approaches are possible
- Show examples of API calls before full implementation
- Report blockers immediately (e.g., API not working as expected)
- Use Canadian English spelling

---

## Recommended Improvements

### Project Hygiene
1. **Add .gitignore file**:
   ```gitignore
   # macOS
   .DS_Store

   # Editor files
   .vscode/
   .idea/
   *.swp
   *.swo
   *~

   # Logs
   *.log
   npm-debug.log*

   # Environment files (if added later)
   .env
   .env.local
   ```

2. **Add README.md**: User-facing documentation with:
   - Installation instructions
   - Usage guide with screenshots
   - Troubleshooting section
   - Contribution guidelines

3. **Version Tagging**: Use git tags for releases (e.g., `v1.0.0`, `v1.1.0`)

### Code Quality
1. **Add JSDoc comments**: Document function signatures and return types
2. **Error boundary**: Add global error handler for unexpected exceptions
3. **Input sanitization**: Additional validation beyond regex matching
4. **Rate limiting**: Prevent API abuse with request throttling

---

## Changelog

### Current Version (November 2024)
**Added**:
- Initial working implementation of UTXO explorer
- Address validation for Legacy, SegWit, and Taproot
- Bitcoin Core RPC integration via CORS proxy
- Dual-method UTXO discovery (fast/slow fallback)
- Clean responsive UI with scrollable results
- Direct Ord server links for each UTXO
- Connection health checking
- Sample addresses pre-populated
- Comprehensive error handling

**Known Issues**:
- Slow UTXO discovery using `scantxoutset` (30-60 seconds per address)
- No automated rare sat detection (manual Ord link clicking required)
- No result caching or persistence
- No loading indicators for long operations

**Pending**:
- Ord API integration for automatic rare sat detection
- Performance optimization (address importing or caching)
- Export functionality (CSV/JSON)
- Loading spinners and progress bars

### Previous Versions
**v30 and earlier**: Development on macOS, file named `rare-sats-finder.html`
- Basic functionality established
- Multiple iterations on Ord API integration (unsuccessful)
- CORS proxy implementation
- Moved to git repository as `index.html`

---

## Document History

**Latest Update**: November 2025 (Updated for Claude Code Web session)
- Renamed from rare-sats-finder.html to index.html
- Added git repository context
- Added comprehensive Architecture & Code Organization section
- Added Guidelines for AI Assistants section
- Added Quick Reference and Table of Contents
- Updated file structure to reflect actual repository state
- Added detailed code map with line numbers
- Enhanced development workflow documentation

**Previous Updates**: November 2024
- Initial CLAUDE.md creation during macOS development
- Documentation of working v30 implementation
- Known limitations and next steps outlined

---

*This document is meant to be shared with Claude Code and other AI assistants to continue the project seamlessly.*