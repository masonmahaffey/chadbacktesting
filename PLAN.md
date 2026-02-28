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

**Critical constraint:** The existing Hetzner server, database (`strategies.db`), cached market data, and all deployed code must NOT be disrupted. The private backtesting system must continue working throughout development.

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

A public-facing SaaS product at `http://95.216.5.147:8000/` (and eventually a custom domain) that lets traders sign up, pay, and use the Arkon backtesting tool. Branding is **GigaChad** meme-inspired - serious product, hilarious personality.

### Architecture Decision Points

These need to be decided before implementation:

1. **Hosting approach**: 
   - Option A: Everything on the same Hetzner server (port 8000) - simpler, add routes to existing FastAPI
   - Option B: Separate frontend deployment (Vercel/Netlify) + Hetzner stays as API-only - more scalable
   - Option C: New service on same server at a different port, reverse proxy with nginx

2. **Frontend framework**: 
   - Option A: Next.js / React (standard SaaS choice)
   - Option B: Plain HTML/CSS/JS (consistent with arkon.html approach)
   - Option C: Astro (good for marketing + app hybrid)

3. **Auth system**:
   - Option A: Clerk / Auth0 (managed, fast to implement)
   - Option B: Custom auth with JWT + SQLite
   - Option C: Supabase Auth

4. **Payments**:
   - Stripe (standard choice)
   - Pricing tiers TBD

5. **Multi-tenancy**:
   - Currently the `owner` field in the DB separates users
   - Need to add proper authentication to protect user data
   - Each user should only see their own strategies/trades

### Proposed SaaS Components

#### Landing Page
- GigaChad themed hero section
- Feature highlights (the backtesting tool screenshots)
- Pricing section
- Sign up / Login CTAs
- Testimonials section (can be tongue-in-cheek GigaChad themed)

#### Auth System
- Sign up / Login pages
- Email verification
- Password reset
- Session management

#### Dashboard
- User's strategies list
- Performance metrics / stats
- Quick-launch backtesting

#### Backtesting Tool
- This IS `arkon.html` - embedded or served for authenticated users
- Needs to be gated behind auth
- User's `owner` field tied to their account

#### Account / Settings
- Profile management
- Subscription management
- Billing portal (Stripe)

#### Admin Panel (internal)
- User management
- Subscription stats
- Usage metrics

### Branding Notes

- **Name**: Chad Backtesting (or "ChadTest", "GigaBacktest", etc.)
- **Personality**: Confident, funny, meme-forward. "Stop being a virgin trader. Backtest like a Chad."
- **Visual style**: Clean modern SaaS design with GigaChad meme imagery as accents
- **Color palette**: Dark mode primary (consistent with Arkon's dark charts), with bold accent colors
- **Tone**: Serious about the product, hilarious in copy. "Your P&L is crying. Fix it."

---

## Task Breakdown

### Phase 0: Foundation (Do First)

- [x] **T0.1** - Create chadbacktesting repo and push to GitHub
- [ ] **T0.2** - Decide on architecture (hosting, frontend framework, auth, payments)
- [ ] **T0.3** - Set up project scaffold in `chadbacktesting/` based on architecture decisions

### Phase 1: Private Route Migration (Initiative 1)

- [ ] **T1.1** - Add `/pvt` route to `mbo_streaming_server.py` that serves `arkon.html`
- [ ] **T1.2** - Verify `arkon.html` works correctly when served from `/pvt` (API calls, WebSocket, screenshots, all features)
- [ ] **T1.3** - Deploy updated `mbo_streaming_server.py` to Hetzner server
- [ ] **T1.4** - Verify `/pvt` works on production server
- [ ] **T1.5** - Update deployment scripts to reference `/pvt` as the private URL
- [ ] **T1.6** - Keep `/` also serving arkon.html during transition (don't break anything)

### Phase 2: SaaS Landing Page (Initiative 2 - Start)

- [ ] **T2.1** - Design landing page wireframe/mockup
- [ ] **T2.2** - Source GigaChad assets and create brand kit (logo, colors, fonts)
- [ ] **T2.3** - Build landing page with hero, features, pricing, CTA sections
- [ ] **T2.4** - Set up the landing page to be served at `/` on the Hetzner server
- [ ] **T2.5** - Add email collection / waitlist functionality

### Phase 3: Authentication

- [ ] **T3.1** - Choose and set up auth provider (Clerk / Auth0 / custom)
- [ ] **T3.2** - Build sign up / login pages
- [ ] **T3.3** - Add auth middleware to protect `/api/*` endpoints
- [ ] **T3.4** - Map authenticated user to `owner` field in database
- [ ] **T3.5** - Gate access to the backtesting tool behind auth

### Phase 4: Payments

- [ ] **T4.1** - Set up Stripe account and API keys
- [ ] **T4.2** - Define pricing tiers (free trial, monthly, annual)
- [ ] **T4.3** - Implement Stripe Checkout / subscription management
- [ ] **T4.4** - Build billing portal / account management page
- [ ] **T4.5** - Add subscription checks to backtesting tool access

### Phase 5: Multi-tenant Backtesting

- [ ] **T5.1** - Ensure each user's data is properly isolated (strategies, trades, settings)
- [ ] **T5.2** - Add per-user data storage limits
- [ ] **T5.3** - Implement user-specific screenshot storage
- [ ] **T5.4** - Add admin dashboard for user/subscription management

### Phase 6: Polish & Launch

- [ ] **T6.1** - Custom domain setup (DNS, SSL)
- [ ] **T6.2** - SEO optimization
- [ ] **T6.3** - Performance testing / load testing
- [ ] **T6.4** - Error monitoring (Sentry or similar)
- [ ] **T6.5** - Analytics (Mixpanel / PostHog / Google Analytics)
- [ ] **T6.6** - Public launch

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

### URLs
| URL | What |
|-----|------|
| `http://95.216.5.147:8000/` | Current: Arkon backtesting. Future: Chad Backtesting landing page |
| `http://95.216.5.147:8000/pvt` | Future: Private Arkon backtesting (after Initiative 1) |
| `http://95.216.5.147:8000/docs` | FastAPI auto-generated API documentation |
| `http://95.216.5.147:8000/health` | Health check endpoint |

---

## Files That Agents Will Need to Modify

### Initiative 1 (Private Route)
1. `arkon/mbo_streaming_server.py` - Add `/pvt` route (~line 1334)
2. `arkon/update_frontend.sh` - Update URL references
3. `arkon/update_frontend.bat` - Update URL references (if exists)
4. `arkon/deploy_mbo_server.bat` - Update URL references
5. `arkon/deploy_mbo_server.sh` - Update URL references

### Initiative 2 (SaaS Product)
1. `chadbacktesting/` - Entire new SaaS codebase
2. `arkon/mbo_streaming_server.py` - Add SaaS routes, auth middleware, landing page serving
3. `arkon/arkon.html` - Potentially add auth token handling for API calls
4. Server config - Potentially nginx for reverse proxy, SSL certs, etc.
