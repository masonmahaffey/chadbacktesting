# Chad Backtesting - Master Plan & System Context

## Table of Contents
1. [Project Vision](#project-vision)
2. [Current System Architecture (Arkon)](#current-system-architecture-arkon)
3. [Server Infrastructure](#server-infrastructure)
4. [Database Schema](#database-schema)
5. [API Endpoints Reference](#api-endpoints-reference)
6. [Frontend (arkon.html)](#frontend-arkonhtml)
7. [Deployment Pipeline](#deployment-pipeline)
8. [Initiative 1: Private Backtesting Route (/pvt)](#initiative-1-private-backtesting-route-pvt)
9. [Initiative 2: Chad Backtesting SaaS Product](#initiative-2-chad-backtesting-saas-product)
10. [Task Breakdown](#task-breakdown)

---

## Project Vision

**Chad Backtesting** is a backtesting SaaS product built around the Arkon charting engine. It uses the **GigaChad meme** for branding to make the product hilarious, fun, and memorable while being a serious, high-quality trading backtesting tool.

The existing Arkon backtesting system (currently at `http://95.216.5.147:8000/`) is a private, personal tool. It will continue to function exactly as-is but will be moved to a private route (`/pvt`). The root URL will become the public-facing Chad Backtesting SaaS product.

**Critical constraints:**
- The existing Hetzner server, database (`strategies.db`), cached market data, and all deployed code must NOT be disrupted. The private backtesting system must continue working throughout development.
- **All Arkon chart code stays in `arkon/` only.** The `chadbacktesting/` repo contains ONLY the SaaS product code (landing page, auth, billing, dashboard, etc.). The backtesting chart is never copied or duplicated - it is served directly from `arkon/` assets on the server.
- **Domain**: `chadbacktesting.com` (DNS already configured, A records pointing to Hetzner server)
- **Payments**: Stripe (confirmed)

---

## Domain & DNS

### Current DNS Configuration

| Record Type | Host | Value | Notes |
|-------------|------|-------|-------|
| A | `@` | `95.216.5.147` | Root domain → Hetzner server |
| A | `www` | `95.216.5.147` | www subdomain → Hetzner server |

### Required Server Configuration

An **nginx reverse proxy** is needed on the Hetzner server to handle:

1. **SSL termination** via Let's Encrypt (certbot) for `chadbacktesting.com`
2. **301 redirect**: `www.chadbacktesting.com` → `chadbacktesting.com` (canonical non-www)
3. **Proxy pass** to uvicorn on `127.0.0.1:8000`
4. **HTTP → HTTPS redirect** on port 80

#### Target nginx config (conceptual)

```nginx
# Redirect www → non-www (301)
server {
    listen 80;
    listen 443 ssl;
    server_name www.chadbacktesting.com;

    ssl_certificate /etc/letsencrypt/live/chadbacktesting.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chadbacktesting.com/privkey.pem;

    return 301 https://chadbacktesting.com$request_uri;
}

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name chadbacktesting.com;
    return 301 https://chadbacktesting.com$request_uri;
}

# Main server block
server {
    listen 443 ssl;
    server_name chadbacktesting.com;

    ssl_certificate /etc/letsencrypt/live/chadbacktesting.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chadbacktesting.com/privkey.pem;

    # Proxy everything to uvicorn
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket support
    location /ws/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400;
    }
}
```

#### Setup commands (on server)
```bash
# Install nginx
apt update && apt install -y nginx

# Install certbot for Let's Encrypt
apt install -y certbot python3-certbot-nginx

# Get SSL certificate (covers both www and non-www)
certbot --nginx -d chadbacktesting.com -d www.chadbacktesting.com

# Auto-renewal is set up automatically by certbot
# Verify: systemctl status certbot.timer
```

#### Important: Existing direct IP access

After nginx is set up, the uvicorn server should bind to `127.0.0.1:8000` (localhost only) instead of `0.0.0.0:8000`. This means:
- `https://chadbacktesting.com/` → nginx → uvicorn (public access)
- `https://chadbacktesting.com/pvt` → nginx → uvicorn (private backtesting)
- `http://95.216.5.147:8000/` → would stop working (intentional, everything goes through nginx now)

If direct IP access is still needed during transition, keep `0.0.0.0:8000` temporarily and update `start_server_background.sh` later.

---

## Current System Architecture (Arkon)

### Overview

Arkon is a market-by-order (MBO) data visualization and backtesting platform for NQ futures. It consists of:

- **Backend**: FastAPI (Python) server running on uvicorn
- **Frontend**: Single monolithic HTML file (`arkon.html`, ~55,000+ lines) with embedded JavaScript
- **Charting**: NightVision charting library (custom build, served from `/static/nightvision-local/`)
- **Database**: SQLite (`strategies.db`) for strategies, trades, trade settings, and tags
- **Data**: Pre-processed MBO data stored as Parquet files in `mbo_processed_cache/`
- **News**: Economic calendar data from `news.csv`

### Local Project Location

```
c:\Users\mason\Documents\arkon\
```

Key files:
| File | Purpose |
|------|---------|
| `mbo_streaming_server.py` | The entire backend server (~3,300+ lines) |
| `arkon.html` | The entire frontend (~55,000+ lines) |
| `data_downloader.py` | Downloads MBO data from Databento |
| `data_loader.py` | Loads and processes MBO data |
| `mbo_processor.py` | Processes raw MBO data into cached bars |
| `news.csv` | Economic news calendar data |
| `requirements.txt` | Python dependencies |
| `update_frontend.sh` | Script to deploy just arkon.html to server |
| `deploy_mbo_server.bat` | Full Windows deployment script |
| `deploy_mbo_server.sh` | Full Linux/Mac deployment script |
| `start_server_background.sh` | Server startup script (runs ON the server) |

---

## Server Infrastructure

### Hetzner Dedicated Server

| Property | Value |
|----------|-------|
| **IP** | `95.216.5.147` |
| **Port** | `8000` |
| **OS** | Ubuntu (Linux) |
| **SSH User** | `root` |
| **SSH Key** | `~/.ssh/hetzner_server_key` (local path: `%USERPROFILE%\.ssh\hetzner_server_key`) |
| **Server Directory** | `/opt/mbo_server` |
| **Process** | `uvicorn mbo_streaming_server:app --host 0.0.0.0 --port 8000` |
| **URL** | `http://95.216.5.147:8000/` |

### Server File Structure

```
/opt/mbo_server/
├── mbo_streaming_server.py          # FastAPI backend server
├── data_loader.py                   # Data loading utilities
├── requirements.txt                 # Python dependencies
├── strategies.db                    # ⚠️ LIVE DATABASE - DO NOT OVERWRITE
├── server.log                       # Server output log
├── server.pid                       # PID file for running server
├── news.csv                         # Economic news calendar
├── start_mbo_server.sh              # Server start script
├── start_server_background.sh       # Background start with PID tracking
├── static/
│   └── arkon.html                   # Frontend HTML file served at /
├── uploads/
│   └── trading-screenshots/         # Trade screenshot storage
├── mbo_processed_cache/
│   ├── cache_index.json             # Index of all cached data files
│   └── *.parquet                    # Processed MBO data (one per day per symbol per interval)
└── *.db.backup.*                    # Database backups with timestamps
```

### SSH Access Patterns

```bash
# Test connection
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "echo OK"

# Check server status
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "ps aux | grep uvicorn"

# View logs
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "cd /opt/mbo_server && tail -f server.log"

# Restart server
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "cd /opt/mbo_server && ./start_server_background.sh"

# Stop server
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "pkill -f uvicorn"

# Upload file
scp -i ~/.ssh/hetzner_server_key localfile root@95.216.5.147:/opt/mbo_server/
```

### Windows (.bat) SSH Access Patterns

The Windows deployment scripts use additional SSH options:
```batch
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i "%USERPROFILE%\.ssh\hetzner_server_key" root@95.216.5.147 "command"
```

---

## Database Schema

**File**: `strategies.db` (SQLite)
**Location on server**: `/opt/mbo_server/strategies.db`

### Tables

#### `strategies`
```sql
CREATE TABLE strategies (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    owner TEXT NOT NULL,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    UNIQUE(owner, name)
);
```

#### `trades`
```sql
CREATE TABLE trades (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    owner TEXT NOT NULL,
    strategy_id INTEGER,
    strategy_name TEXT,
    r_label TEXT,                    -- Risk:Reward label
    trade_count INTEGER,            -- Global trade number for strategy
    days_trade_number INTEGER,      -- Trade number within the day
    trade_number INTEGER,           -- Legacy alias for days_trade_number
    trading_day TEXT,               -- Backtesting date
    entry_time TEXT NOT NULL,
    exit_time TEXT,
    side TEXT NOT NULL,             -- 'buy' or 'sell'
    entry_price REAL NOT NULL,
    exit_price REAL,
    quantity INTEGER NOT NULL,
    pnl_ticks REAL,
    entry_screenshot_url TEXT,      -- Screenshot at trade entry
    exit_screenshot_url TEXT,       -- Screenshot at trade exit
    review TEXT,                    -- Written trade review/journal
    outcome TEXT,                   -- 'win', 'loss', etc.
    real_world_date TEXT,           -- Real date (not backtesting date)
    checklist_states_json TEXT,     -- JSON of tag checklist states
    created_at TEXT NOT NULL,
    FOREIGN KEY(strategy_id) REFERENCES strategies(id)
);
```

#### `strategy_settings`
```sql
CREATE TABLE strategy_settings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    owner TEXT NOT NULL,
    strategy_id INTEGER,
    strategy_name TEXT,
    stop_ticks INTEGER NOT NULL,           -- Stop loss distance in ticks
    rr_json TEXT NOT NULL,                 -- JSON array of R:R ratios [1, 1.5, 2, 3]
    system_rules_json TEXT,                -- JSON array of tags/rules objects
    show_checklist_after_trades INTEGER DEFAULT 0,  -- Show checklist after trade
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    UNIQUE(owner, strategy_id)
);
-- Also: UNIQUE INDEX idx_settings_owner_strategy_name ON strategy_settings(owner, strategy_name)
```

### Database Protection Rules

- The server's `strategies.db` contains live trade data and MUST NEVER be overwritten during deployment
- `deploy_mbo_server.bat` already checks if the DB exists on server and skips copying if so
- Backups are created automatically with timestamps: `strategies.db.backup.YYYYMMDD_HHMMSS`
- Backup/restore scripts: `backup_database.bat`, `restore_database.bat`, `purge_database.bat`

---

## API Endpoints Reference

All endpoints are defined in `mbo_streaming_server.py`. The server runs on port 8000.

### HTML / Static

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | Serves `static/arkon.html` as the main client |
| STATIC | `/static/*` | Mounted StaticFiles directory |
| STATIC | `/uploads/*` | Mounted uploads directory (screenshots) |

### Strategy CRUD

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/strategies` | List strategies (optional `?owner=`) |
| POST | `/api/strategies` | Create strategy `{name, owner}` |
| PUT | `/api/strategies/{id}` | Update strategy |
| DELETE | `/api/strategies/{id}` | Delete strategy |

### Trades CRUD

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/trades` | List trades (optional `?owner=`, `?strategy_id=`, `?strategy_name=`) |
| POST | `/api/trades` | Create trade (full TradeCreate payload) |
| PUT | `/api/trades/{id}` | Update trade |
| DELETE | `/api/trades/{id}` | Delete single trade |
| DELETE | `/api/trades/purge` | Purge all trades |
| DELETE | `/api/trades/strategy` | Delete all trades for a strategy |

### Strategy Settings

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/strategy_settings` | Get settings `?owner=&strategy_id=` |
| POST | `/api/strategy_settings` | Upsert settings (stop_ticks, rr_list, tags, checklist toggle) |

### Market Data

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/mbo/{symbol}` | Fetch cached MBO data for a symbol |
| GET | `/api/cache/info` | Cache statistics |
| GET | `/api/cache/available_dates` | Available trading dates in cache |
| DELETE | `/api/cache/clear` | Clear cache |
| GET | `/api/fields` | Data field documentation |
| GET | `/api/example/{symbol}` | Example data |

### Images

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/images/upload` | Upload screenshot (multipart form) |
| GET | `/api/images/{name}` | Retrieve uploaded image |

### News

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/news?date=YYYY-MM-DD` | Get economic news events for a date |

### WebSocket

| Path | Description |
|------|-------------|
| `ws://host:8000/ws/mbo/{symbol}` | Stream cached MBO data bars |
| `ws://host:8000/ws/hybrid/{symbol}` | Hybrid streaming endpoint |

### Health

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Health check with cache stats |

---

## Frontend (arkon.html)

### How It Connects to the Server

The frontend determines its API base URL via the `computeBaseUrls()` function (line ~24706):

```javascript
function computeBaseUrls() {
    const apiOverride = getQueryParam('api');      // ?api=https://...
    const remoteParam = getQueryParam('remote');    // ?remote (flag)
    const apiFromStorage = localStorage.getItem('API_BASE_URL');

    if (remoteParam !== null) {
        const remoteApi = 'http://95.216.5.147:8000';
        return { api: remoteApi, ws: remoteApi.replace(/^http/, 'ws') };
    }

    let origin = window.location.origin;
    if (!origin || origin === 'null' || origin.startsWith('file://')) {
        origin = 'http://localhost:8000';
    }

    const api = apiOverride || apiFromStorage || origin;
    const ws = api.startsWith('https') ? api.replace(/^https/, 'wss') : api.replace(/^http/, 'ws');
    return { api, ws };
}
```

**Key behavior**: When served from the server at `http://95.216.5.147:8000/`, it uses `window.location.origin` which resolves to `http://95.216.5.147:8000`. All API calls are then made relative to this origin (e.g., `${API_BASE_URL}/api/trades`). This means moving the HTML to `/pvt` does NOT break API calls - the origin stays the same.

### API Call Pattern

All fetch calls use:
```javascript
fetch(`${API_BASE_URL}/api/endpoint...`)
```

WebSocket connections use:
```javascript
new WebSocket(`${WS_BASE_URL}/ws/mbo/${symbol}`)
```

### Features
- Multi-chart layout (Chart 1, 2, 3) with configurable timeframes (5s, 10s, 15s, 30s, 1m, 2m, 5m, 15m)
- NightVision charting library with heatmap overlays (CVD, Depth Delta)
- Volume Profile, Delta Profile, Dynamic VP
- Fibonacci, measurement, reference line tools
- Trading simulation (buy/sell/stop/limit orders, flatten, cancel)
- Trade journaling with screenshots, reviews, tags
- Strategy management with configurable stop distances and R:R ratios
- News event warnings
- Random day mode for unbiased backtesting
- ZigZag/swing detection with configurable thresholds
- Streaming mode (250ms candle playback)
- Step-through mode (candle by candle)

---

## Deployment Pipeline

### Frontend-Only Update (arkon.html)

**Script**: `arkon/update_frontend.sh` or `arkon/update_frontend.bat`

Steps:
1. SSH test connection
2. Backup current `static/arkon.html` on server with timestamp
3. SCP new `arkon.html` → `/opt/mbo_server/static/arkon.html`
4. Check if server needs restart (only if `mbo_streaming_server.py` was modified)
5. Verify frontend is accessible via curl

**No server restart needed** for HTML-only changes (served statically via FileResponse).

### Full Server Deployment

**Script**: `arkon/deploy_mbo_server.bat` (Windows) or `arkon/deploy_mbo_server.sh` (Linux/Mac)

Steps:
1. SSH test connection
2. Create `/opt/mbo_server` directory
3. Copy: `mbo_streaming_server.py`, `requirements.txt`, `data_loader.py`, startup scripts
4. Create `static/` dir, copy `arkon.html`
5. Create `uploads/trading-screenshots/` dir
6. **Database protection**: Check if `strategies.db` exists on server; SKIP copy if it does
7. Install Python dependencies via pip3
8. Start server via `start_server_background.sh`
9. Verify server responds on port 8000

### Server Startup (on-server)

**Script**: `/opt/mbo_server/start_server_background.sh`

```bash
cd /opt/mbo_server
pkill -f "uvicorn mbo_streaming_server:app" 2>/dev/null || true
nohup python3 -m uvicorn mbo_streaming_server:app --host 0.0.0.0 --port 8000 > server.log 2>&1 &
```

---

## Initiative 1: Private Backtesting Route (/pvt)

### Goal

Move the personal backtesting tool from `/` to `/pvt` so the root URL can serve the Chad Backtesting SaaS landing page.

### What Changes

#### Server Changes (mbo_streaming_server.py)

1. **Add `/pvt` route** that serves `static/arkon.html`:
   ```python
   @app.get("/pvt")
   async def private_backtesting():
       html_file = Path("static/arkon.html")
       if html_file.exists():
           return FileResponse(html_file, media_type="text/html")
       return {"error": "arkon.html not found"}
   ```

2. **Keep root `/` route** but modify it to serve the SaaS landing page (once built). For now, it can serve arkon.html at BOTH routes during transition.

3. **All API endpoints stay unchanged** at `/api/*` and `/ws/*`. The arkon.html frontend already uses `window.location.origin` for API base URL, so it works from any path on the same origin.

#### Frontend Changes (arkon.html)

- **Likely NO changes needed** for the `/pvt` route itself. The `computeBaseUrls()` function uses `window.location.origin` (e.g., `http://95.216.5.147:8000`) which is path-independent.
- Verify that no hardcoded root-relative paths (`/static/...`, etc.) in the HTML would break.

#### Deployment Changes

- Update `update_frontend.sh` to mention `/pvt` as the new access URL
- Update documentation references

### What DOES NOT Change

- Database (`strategies.db`) - untouched
- All API endpoints (`/api/*`, `/ws/*`) - unchanged
- Data cache (`mbo_processed_cache/`) - unchanged
- Screenshots (`/uploads/`) - unchanged
- Server process / port / startup - unchanged

### Risk Assessment

**Low risk.** Adding a new route is additive. The existing system keeps working. If anything goes wrong, reverting is just removing the new route.

---

## Initiative 2: Chad Backtesting SaaS Product

### Vision

A public-facing SaaS product at `https://chadbacktesting.com` that lets traders sign up, pay, and use the backtesting tool. Branding is **GigaChad** meme-inspired - serious product, hilarious personality.

### Architecture (Confirmed Decisions)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Hosting** | Everything on Hetzner server, nginx reverse proxy | Single server simplicity, domain already pointed here |
| **Domain** | `chadbacktesting.com` | DNS already configured (A records → 95.216.5.147) |
| **SSL** | Let's Encrypt via certbot + nginx | Free, automatic renewal |
| **Auth** | Google Sign-In (OAuth 2.0) | Single sign-on, no password management needed |
| **Payments** | Stripe (keys/config to be provided by Mason later) | Industry standard, confirmed |
| **www redirect** | 301 `www.chadbacktesting.com` → `chadbacktesting.com` | Canonical non-www |

### Architecture Decision Points (Still Open)

1. **Frontend framework**: 
   - Option A: Next.js / React (standard SaaS choice, needs Node.js on server or separate build)
   - Option B: Plain HTML/CSS/JS (consistent with arkon.html, simplest to deploy)
   - Option C: Astro (good for marketing + app hybrid, static output)

2. **Multi-tenancy**:
   - Currently the `owner` field in the DB separates users
   - With Google Sign-In, the user's Google account ID or email becomes their `owner` identity
   - Need auth middleware to protect user data so each user only sees their own strategies/trades

### Code Separation Rule

**The `chadbacktesting/` repo does NOT contain any Arkon chart code.** It only contains:
- SaaS marketing / landing pages
- Auth integration (Google Sign-In OAuth flow)
- Dashboard pages
- Billing / account management pages
- Deployment scripts for the SaaS layer
- nginx configuration templates
- Any SaaS-specific API code (user accounts, billing webhooks, etc.)

The backtesting chart (`arkon.html`) stays in `arkon/` and is served directly by the existing FastAPI server. The SaaS layer gates access to it via auth but never duplicates it.

### Pricing Tiers

| Tier | Price | What They Get |
|------|-------|---------------|
| **Free** | $0/mo | Full backtesting tool access. Load any available data, backtest freely, save trades, strategies, trade journaling, screenshots, tags - everything the tool does today. No restrictions on features. |
| **GigaChad** | $250/mo | Everything in Free + AI Indicator Generator |
| **MegaChad** | TBD | Everything in GigaChad + AI Chad Coach with voice-narrated analysis |

**Important: The Free tier is generous by design.** Every user gets the full backtesting experience for free. Paid tiers are for advanced AI-powered features only.

### Proposed SaaS Components

#### Landing Page (`/`)
- GigaChad themed hero section
- Feature highlights (backtesting tool screenshots)
- Pricing section showing Free / GigaChad / MegaChad tiers
- "Sign in with Google" CTA
- Testimonials section (tongue-in-cheek GigaChad themed)

#### Auth System
- **Google Sign-In only** (OAuth 2.0) - no email/password
- User clicks "Sign in with Google" → Google OAuth flow → redirect back with token
- Server creates/finds user account based on Google profile (email, name, avatar)
- Session management via JWT or secure httpOnly cookies
- User's Google email or Google ID becomes their `owner` identity in the database

#### Dashboard (`/dashboard`)
- User's strategies list
- Performance metrics / stats
- Quick-launch backtesting button
- Current subscription tier badge (Free / GigaChad / MegaChad)

#### Backtesting Tool (`/app` or `/backtest`)
- This IS `arkon.html` - served for authenticated users
- Gated behind Google Sign-In auth (any tier, including Free)
- User's `owner` field tied to their Google account
- Served by the existing FastAPI server at a protected route

#### Account / Settings (`/account`, `/billing`)
- Profile info (from Google account)
- Current subscription tier
- Upgrade to GigaChad / MegaChad via Stripe Checkout
- Manage subscription via Stripe Customer Portal
- Billing history

#### Admin Panel (`/admin` - internal only)
- User management
- Subscription stats
- Usage metrics

### Stripe Integration Details

- **Stripe Checkout**: For upgrading from Free → GigaChad or MegaChad
- **Stripe Customer Portal**: For subscription management (cancel, upgrade, payment method)
- **Stripe Webhooks**: `/api/stripe/webhook` endpoint to handle:
  - `checkout.session.completed` - activate paid subscription
  - `customer.subscription.updated` - plan changes
  - `customer.subscription.deleted` - revert to Free tier
  - `invoice.payment_failed` - handle failed payments
- **Stripe API keys**: To be provided by Mason when ready for implementation
- **Note**: Free users do NOT go through Stripe at all. Only paid tier upgrades touch Stripe.

---

### Future Features (DO NOT BUILD UNTIL MASON SAYS SO)

> **AGENTS: READ THIS CAREFULLY.** The following features are part of the long-term product vision
> but are explicitly NOT to be built now. Do not implement, scaffold, stub out, or create
> placeholder code for these features. They are documented here for context only. Mason will
> personally flesh out the specs and explicitly request implementation when the time comes.

#### FUTURE: AI Indicator Generator (GigaChad Tier - $250/mo)

**Status: DO NOT BUILD. Documentation only.**

An AI-powered tool that generates custom trading indicators. Details TBD by Mason. This is the core value proposition of the GigaChad paid tier. Until Mason provides specifications and says to build it, this feature does not exist in the codebase.

#### FUTURE: AI Chad Coach (MegaChad Tier - Price TBD)

**Status: DO NOT BUILD. Documentation only. Mason will flesh out this feature personally.**

A premium AI coaching feature that:
1. **Voice Recording During Backtesting**: While the user backtests, they narrate into their microphone everything they're factoring into their decisions (e.g., "I see a CVD divergence here, the bid depth is thinning, volume is picking up at this level...")
2. **Chart Data Correlation**: The voice narration is tied to the exact chart state - loaded data timestamps, visible indicators, price levels, volume data - everything shown on screen at the moment the user speaks
3. **Retroactive Factor Analysis**: After backtesting sessions, AI analyzes every factor the user mentioned in their narration and correlates it with their actual trade performance (entries, exits, P&L)
4. **Dynamic Graphs & Statistics**: Generates flexible visualizations and statistics showing which factors in the user's system actually correlate with winning vs losing trades, which factors they mention but don't act on, which setups they describe that lead to the best risk-adjusted returns, etc.

This is the most advanced and differentiated feature of the product. It requires significant design work from Mason before any implementation begins.

### Branding Notes

- **Name**: Chad Backtesting
- **Domain**: `chadbacktesting.com`
- **Personality**: Confident, funny, meme-forward. "Stop being a virgin trader. Backtest like a Chad."
- **Visual style**: Clean modern SaaS design with GigaChad meme imagery as accents
- **Color palette**: Dark mode primary (consistent with the dark charts), with bold accent colors
- **Tone**: Serious about the product, hilarious in copy. "Your P&L is crying. Fix it."

---

## Task Breakdown

### Phase 0: Foundation (Do First)

- [x] **T0.1** - Create chadbacktesting repo and push to GitHub
- [x] **T0.2** - Document full system architecture and context for agents (this file)
- [ ] **T0.3** - Decide on frontend framework
- [ ] **T0.4** - Set up project scaffold in `chadbacktesting/` based on architecture decisions

### Phase 1: Server Infrastructure (nginx, SSL, Domain)

- [ ] **T1.1** - Install nginx on Hetzner server
- [ ] **T1.2** - Configure nginx as reverse proxy to uvicorn on port 8000
- [ ] **T1.3** - Install certbot and obtain Let's Encrypt SSL cert for `chadbacktesting.com` + `www.chadbacktesting.com`
- [ ] **T1.4** - Configure nginx 301 redirect: `www.chadbacktesting.com` → `chadbacktesting.com`
- [ ] **T1.5** - Configure nginx HTTP → HTTPS redirect on port 80
- [ ] **T1.6** - Configure nginx WebSocket proxy for `/ws/` paths
- [ ] **T1.7** - Test: `https://chadbacktesting.com/` serves the app correctly (temporarily still arkon.html)
- [ ] **T1.8** - Test: `https://www.chadbacktesting.com/` 301 redirects to `https://chadbacktesting.com/`
- [ ] **T1.9** - Test: WebSocket connections work through nginx (`wss://chadbacktesting.com/ws/...`)
- [ ] **T1.10** - Update `arkon.html` `computeBaseUrls()` to handle HTTPS origin / `chadbacktesting.com` domain correctly
- [ ] **T1.11** - Decide: lock down direct IP:8000 access or keep as fallback during dev

### Phase 2: Private Route Migration (Initiative 1)

- [ ] **T2.1** - Add `/pvt` route to `mbo_streaming_server.py` that serves `arkon.html`
- [ ] **T2.2** - Verify `arkon.html` works correctly when served from `/pvt` (API calls, WebSocket, screenshots, all features)
- [ ] **T2.3** - Deploy updated `mbo_streaming_server.py` to Hetzner server
- [ ] **T2.4** - Verify `https://chadbacktesting.com/pvt` works on production
- [ ] **T2.5** - Update deployment scripts to reference `/pvt` as the private URL
- [ ] **T2.6** - Keep `/` also serving arkon.html during transition (don't break anything)

### Phase 3: SaaS Landing Page (Initiative 2 - Start)

- [ ] **T3.1** - Source GigaChad assets and create brand kit (logo, colors, fonts)
- [ ] **T3.2** - Build landing page with hero, features, pricing, CTA sections
- [ ] **T3.3** - Deploy landing page to be served at `/` on the Hetzner server (replaces arkon.html at root)
- [ ] **T3.4** - Add email collection / waitlist functionality
- [ ] **T3.5** - Verify `/pvt` still works after root changes

### Phase 4: Google Sign-In Authentication

- [ ] **T4.1** - Set up Google Cloud project and OAuth 2.0 credentials (Client ID, Client Secret)
- [ ] **T4.2** - Add `users` table to the database (google_id, email, name, avatar_url, subscription_tier, stripe_customer_id, created_at)
- [ ] **T4.3** - Implement Google OAuth flow on the server (redirect to Google, handle callback, create/find user, issue session)
- [ ] **T4.4** - Build "Sign in with Google" button on landing page and any auth-required pages
- [ ] **T4.5** - Implement session management (JWT or httpOnly cookies)
- [ ] **T4.6** - Add auth middleware to protect `/app` (backtesting tool) and `/api/*` endpoints
- [ ] **T4.7** - Map authenticated user's Google ID/email to the `owner` field in strategies/trades
- [ ] **T4.8** - Ensure `/pvt` route bypasses SaaS auth (Mason's private access, no Google login needed)
- [ ] **T4.9** - All Free tier users get full backtesting access after signing in (no paywall for core features)

### Phase 5: Stripe Payments (for GigaChad / MegaChad tiers only)

- [ ] **T5.1** - Mason provides Stripe API keys (test + live) - **BLOCKED until Mason provides**
- [ ] **T5.2** - Create Stripe Products and Prices: GigaChad ($250/mo), MegaChad (price TBD)
- [ ] **T5.3** - Implement Stripe Checkout flow (Free user → upgrade to paid tier)
- [ ] **T5.4** - Implement `/api/stripe/webhook` endpoint for subscription lifecycle events
- [ ] **T5.5** - Set up Stripe Customer Portal for self-service subscription management
- [ ] **T5.6** - Store subscription tier on user record, update via webhooks
- [ ] **T5.7** - Build `/billing` page with current tier, upgrade options, manage subscription
- [ ] **T5.8** - Test full flow: sign in (Free) → upgrade to GigaChad → manage → cancel → revert to Free
- [ ] **T5.9** - Note: Paid tier features (AI Indicator Generator, AI Chad Coach) are NOT built yet. Subscription upgrade should work but gated features show "Coming Soon" until Mason says to build them.

### Phase 6: Multi-tenant Backtesting

- [ ] **T6.1** - Ensure each user's data is properly isolated (strategies, trades, settings)
- [ ] **T6.2** - Add per-user data storage limits
- [ ] **T6.3** - Implement user-specific screenshot storage paths
- [ ] **T6.4** - Build `/dashboard` with user's strategies and performance stats
- [ ] **T6.5** - Build `/admin` panel for user/subscription management (internal)

### Phase 7: Polish & Launch

- [ ] **T7.1** - SEO optimization (meta tags, sitemap, robots.txt)
- [ ] **T7.2** - Performance testing / load testing
- [ ] **T7.3** - Error monitoring (Sentry or similar)
- [ ] **T7.4** - Analytics (Mixpanel / PostHog / Google Analytics)
- [ ] **T7.5** - Legal pages (Terms of Service, Privacy Policy)
- [ ] **T7.6** - Public launch

### Phase 8: Future Premium Features (DO NOT START - MASON WILL INITIATE)

> **These phases are placeholders only. Do not begin work on any of them.**

- [ ] **T8.1** - AI Indicator Generator (GigaChad tier feature) - Mason will provide specs
- [ ] **T8.2** - AI Chad Coach voice recording system (MegaChad tier feature) - Mason will provide specs
- [ ] **T8.3** - Chart data + voice correlation engine - Mason will provide specs
- [ ] **T8.4** - Retroactive factor analysis and dynamic graph generation - Mason will provide specs

---

## Quick Reference Commands

### Server Access
```bash
# SSH into server
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147

# Check what's running
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "ps aux | grep uvicorn"

# View server logs
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "cd /opt/mbo_server && tail -100 server.log"

# Restart server
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "cd /opt/mbo_server && ./start_server_background.sh"

# Check database size
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "ls -lh /opt/mbo_server/strategies.db"

# List all files on server
ssh -i ~/.ssh/hetzner_server_key root@95.216.5.147 "ls -la /opt/mbo_server/"
```

### Local Development
```bash
# Run server locally
cd c:\Users\mason\Documents\arkon
python -m uvicorn mbo_streaming_server:app --host 0.0.0.0 --port 8000

# Deploy frontend only
cd c:\Users\mason\Documents\arkon
.\update_frontend.bat

# Full deployment
cd c:\Users\mason\Documents\arkon
.\deploy_mbo_server.bat
```

### URLs (After Full Setup)

| URL | What |
|-----|------|
| `https://chadbacktesting.com/` | Chad Backtesting SaaS landing page |
| `https://chadbacktesting.com/pvt` | Mason's private Arkon backtesting (no auth) |
| `https://chadbacktesting.com/app` | Authenticated user backtesting tool (arkon.html gated by auth+subscription) |
| `https://chadbacktesting.com/auth/google` | Google OAuth redirect |
| `https://chadbacktesting.com/auth/callback` | Google OAuth callback |
| `https://chadbacktesting.com/dashboard` | User dashboard |
| `https://chadbacktesting.com/billing` | Subscription management |
| `https://chadbacktesting.com/docs` | FastAPI auto-generated API docs |
| `https://chadbacktesting.com/health` | Health check endpoint |
| `https://www.chadbacktesting.com/*` | 301 redirects to `https://chadbacktesting.com/*` |
| `http://95.216.5.147:8000/` | Direct IP access (may be locked down after nginx setup) |

---

## Files That Agents Will Need to Modify

### Infrastructure (nginx, SSL, Domain)
1. Server: Install nginx, certbot - commands run via SSH
2. Server: Create `/etc/nginx/sites-available/chadbacktesting.com` config
3. Server: Symlink to `/etc/nginx/sites-enabled/`
4. `arkon/start_server_background.sh` - Potentially change `0.0.0.0` → `127.0.0.1` after nginx is set up

### Initiative 1 (Private Route)
1. `arkon/mbo_streaming_server.py` - Add `/pvt` route (~line 1334)
2. `arkon/update_frontend.sh` - Update URL references
3. `arkon/update_frontend.bat` - Update URL references (if exists)
4. `arkon/deploy_mbo_server.bat` - Update URL references
5. `arkon/deploy_mbo_server.sh` - Update URL references

### Initiative 2 (SaaS Product)
1. `chadbacktesting/` - SaaS-only code (landing page, auth pages, dashboard, billing pages, deployment scripts)
2. `arkon/mbo_streaming_server.py` - Add SaaS routes (landing page serving, auth middleware, Stripe webhook endpoint, user management API)
3. `arkon/arkon.html` - Potentially add auth token handling for API calls when served to SaaS users
4. Server: nginx config for domain routing, SSL, www redirect

### What `chadbacktesting/` Repo Contains (and Does NOT Contain)

**Contains:**
- Landing page HTML/CSS/JS (or framework-based pages)
- Auth integration code (Google Sign-In OAuth flow)
- Dashboard page templates
- Billing/account page templates
- GigaChad brand assets (images, logo, fonts)
- nginx config templates
- SaaS deployment scripts
- Stripe integration code (if separate from main server)
- This planning document (PLAN.md)

**Does NOT contain:**
- `arkon.html` (stays in `arkon/`)
- `mbo_streaming_server.py` (stays in `arkon/`)
- NightVision charting library (stays in `arkon/static/nightvision-local/`)
- Market data, parquet files, cache (stays in `arkon/`)
- `strategies.db` or any database files
- Any data processing scripts
