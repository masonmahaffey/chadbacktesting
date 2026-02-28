# Chad Backtesting - Master Plan & System Context

**Companion documents:**
- **AGENTS.md** — Agent execution system, roles, flows, recursive build loop, user persona, prompt templates
- **PLAN.md** (this file) — Product plan, architecture, database, API, security, task breakdown

## Table of Contents
1. [Project Vision](#project-vision)
2. [Current System Architecture (Arkon)](#current-system-architecture-arkon)
3. [Server Infrastructure](#server-infrastructure)
4. [Database Schema](#database-schema)
5. [API Endpoints Reference](#api-endpoints-reference)
6. [Frontend (arkon.html)](#frontend-arkonhtml)
7. [Deployment Pipeline](#deployment-pipeline)
8. [Security, SSL & Auth Architecture](#security-ssl--auth-architecture)
9. [Initiative 1: Private Backtesting Route (/pvt)](#initiative-1-private-backtesting-route-pvt)
10. [Initiative 2: Chad Backtesting SaaS Product](#initiative-2-chad-backtesting-saas-product)
11. [Task Breakdown](#task-breakdown)

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

#### Target nginx config (production-hardened)

```nginx
# Redirect ALL HTTP → HTTPS (both www and non-www)
server {
    listen 80;
    server_name chadbacktesting.com www.chadbacktesting.com;

    # Let's Encrypt ACME challenge must remain on HTTP
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://chadbacktesting.com$request_uri;
    }
}

# Redirect www HTTPS → non-www HTTPS (301)
server {
    listen 443 ssl http2;
    server_name www.chadbacktesting.com;

    ssl_certificate /etc/letsencrypt/live/chadbacktesting.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chadbacktesting.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    return 301 https://chadbacktesting.com$request_uri;
}

# Main server block - chadbacktesting.com (HTTPS only)
server {
    listen 443 ssl http2;
    server_name chadbacktesting.com;

    # --- SSL / TLS ---
    ssl_certificate /etc/letsencrypt/live/chadbacktesting.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chadbacktesting.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;

    # --- Security Headers ---
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # --- Upload size limit (for screenshots) ---
    client_max_body_size 10M;

    # --- Rate limiting zone (defined in http block, see note below) ---
    # limit_req zone=api burst=20 nodelay;

    # --- Proxy to uvicorn ---
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_redirect off;
    }

    # --- WebSocket support ---
    location /ws/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    # --- Block direct access to sensitive paths ---
    location ~ /\.env { deny all; return 404; }
    location ~ /\.git { deny all; return 404; }
}
```

**Note**: Rate limiting (`limit_req_zone`) must be defined in the `http {}` block of `/etc/nginx/nginx.conf`:
```nginx
# In /etc/nginx/nginx.conf, inside http { }
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
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

## Security, SSL & Auth Architecture

### SSL / TLS (Let's Encrypt)

All traffic MUST go through HTTPS. No exceptions.

| Layer | Implementation |
|-------|---------------|
| **Certificate provider** | Let's Encrypt (free, auto-renewable) |
| **Certificate manager** | certbot with nginx plugin |
| **SSL termination** | nginx (uvicorn never sees raw TLS) |
| **Protocols** | TLSv1.2 and TLSv1.3 only (no TLS 1.0/1.1) |
| **HSTS** | `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` |
| **Auto-renewal** | certbot timer (verify: `systemctl status certbot.timer`) |
| **Cert scope** | Covers both `chadbacktesting.com` and `www.chadbacktesting.com` |

### Authentication Flow (Google OAuth 2.0)

```
User clicks "Sign in with Google"
    → Browser redirects to Google OAuth consent screen
    → User authorizes
    → Google redirects to https://chadbacktesting.com/auth/callback?code=...
    → Server exchanges code for Google tokens (server-side, never exposed to browser)
    → Server extracts user profile (google_id, email, name, avatar)
    → Server creates or finds user in `users` table
    → Server issues a session token (signed JWT or session ID)
    → Token stored in secure httpOnly cookie
    → Browser redirected to /dashboard or /app
```

### Session / Cookie Security

| Property | Value | Reason |
|----------|-------|--------|
| `httpOnly` | `true` | Prevents JavaScript access (XSS mitigation) |
| `Secure` | `true` | Cookie only sent over HTTPS |
| `SameSite` | `Lax` | CSRF mitigation while allowing OAuth redirects |
| `Path` | `/` | Available across the whole site |
| `Max-Age` | `604800` (7 days) | Reasonable session lifetime |
| **Signing** | HMAC-SHA256 with server secret | Prevents tampering |

### API Endpoint Protection Matrix

After auth is implemented, endpoints will be categorized:

| Category | Endpoints | Auth Required | Notes |
|----------|-----------|---------------|-------|
| **Public** | `/`, `/health`, `/auth/*` | No | Landing page, health check, OAuth flow |
| **Public API** | `/api/news`, `/api/cache/available_dates`, `/api/fields` | No | Read-only public data |
| **Authenticated** | `/app`, `/dashboard`, `/billing`, `/account` | Google Sign-In | Any logged-in user (Free, GigaChad, MegaChad) |
| **Authenticated API** | `/api/strategies`, `/api/trades`, `/api/strategy_settings`, `/api/images/*` | Google Sign-In + owner check | User can only access their own data |
| **Authenticated API** | `/api/mbo/*`, `/ws/mbo/*`, `/ws/hybrid/*` | Google Sign-In | Market data - any authenticated user |
| **Paid tier only** | `/learn/*` (future) | Google Sign-In + GigaChad/MegaChad | Learning Area |
| **Private** | `/pvt` | No SaaS auth; protected separately (see below) | Mason's private backtesting |
| **Internal** | `/admin` | Google Sign-In + admin flag | Mason only |
| **Webhook** | `/api/stripe/webhook` | Stripe signature verification | No user auth; verified by Stripe signing secret |
| **Dangerous** | `/api/cache/clear`, `/api/trades/purge` | Authenticated + admin only | Destructive operations |

### Private Route (`/pvt`) Protection

The `/pvt` route is Mason's personal backtesting tool and must NOT require Google Sign-In (it predates the SaaS). Protection options:

- **Option A (recommended)**: Secret query parameter — `/pvt?key=<secret>` — server checks against env var `PVT_ACCESS_KEY`
- **Option B**: nginx basic auth — password-protect `/pvt` at the nginx level
- **Option C**: IP whitelist in nginx — only allow Mason's IP(s)
- **Option D**: No protection — rely on the URL being unlinked/unlisted (security through obscurity, not recommended)

Choose during implementation. The key requirement: `/pvt` works without Google Sign-In but is not trivially discoverable by random users.

### CORS Policy

**Current state (INSECURE)**: `allow_origins=["*"]` in `mbo_streaming_server.py` line ~1160.

**Target state**: Lock down to the production domain:
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://chadbacktesting.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

During development, can temporarily include `http://localhost:8000` as well.

### WebSocket Authentication

Currently WebSocket connections (`/ws/mbo/*`, `/ws/hybrid/*`) have no authentication. After auth is implemented:

- The WebSocket handshake should carry the session cookie (browsers automatically send cookies on same-origin WS connections)
- The server should validate the session on the initial WS handshake and reject unauthenticated connections
- The `/pvt` WebSocket usage (from arkon.html at `/pvt`) needs to work with the `/pvt` auth mechanism, not Google Sign-In

### Stripe Webhook Security

Stripe webhooks MUST be verified using the webhook signing secret:
```python
import stripe
stripe.Webhook.construct_event(payload, sig_header, webhook_secret)
```
Never trust webhook data without signature verification. The `webhook_secret` comes from the Stripe dashboard and is stored as an environment variable.

### Image Upload Security

Current `POST /api/images/upload` needs hardening:
- **File type validation**: Only allow `.png`, `.jpg`, `.jpeg`, `.webp`
- **File size limit**: 10MB max (enforced by nginx `client_max_body_size` AND server-side)
- **Filename sanitization**: Strip path traversal characters, generate UUID filenames
- **Storage isolation**: Each user's screenshots stored under `/uploads/<user_id>/` to prevent cross-user access

### Secrets / Environment Variables

**NEVER commit secrets to git.** All secrets stored as environment variables on the server via a `.env` file at `/opt/mbo_server/.env`:

| Variable | Purpose | When Needed |
|----------|---------|-------------|
| `GOOGLE_CLIENT_ID` | Google OAuth 2.0 Client ID | Phase 4 (Auth) |
| `GOOGLE_CLIENT_SECRET` | Google OAuth 2.0 Client Secret | Phase 4 (Auth) |
| `SESSION_SECRET` | JWT/cookie signing key (random 64+ char string) | Phase 4 (Auth) |
| `PVT_ACCESS_KEY` | Secret key for `/pvt` route access | Phase 2 (Private Route) |
| `STRIPE_SECRET_KEY` | Stripe API secret key | Phase 5 (Payments) |
| `STRIPE_PUBLISHABLE_KEY` | Stripe public key (safe for frontend) | Phase 5 (Payments) |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret | Phase 5 (Payments) |
| `ANTHROPIC_API_KEY` | Claude Haiku API key | Phase 8 (Future - Learning Area) |

The server loads these via `python-dotenv` (already a dependency). The `.env` file must be in `.gitignore` and never deployed via git.

### nginx Security Headers

Already included in the hardened nginx config above:
- `Strict-Transport-Security` (HSTS) — force HTTPS for 2 years
- `X-Frame-Options: SAMEORIGIN` — prevent clickjacking
- `X-Content-Type-Options: nosniff` — prevent MIME sniffing
- `X-XSS-Protection: 1; mode=block` — legacy XSS filter
- `Referrer-Policy: strict-origin-when-cross-origin` — control referrer leakage

### Rate Limiting

nginx rate limiting to prevent abuse (especially important for free tier):
- **Global**: 10 requests/second per IP (burst 20)
- **Auth endpoints**: 5 requests/second per IP (prevent brute force on OAuth)
- **API write endpoints**: 5 requests/second per IP (prevent spam)
- **WebSocket**: Connection limit per IP (e.g., max 10 concurrent WS connections)

---

## Database Migration Strategy

### New `users` Table (Added in Phase 4)

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    google_id TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE NOT NULL,
    name TEXT,
    avatar_url TEXT,
    subscription_tier TEXT NOT NULL DEFAULT 'free',  -- 'free', 'gigachad', 'megachad'
    stripe_customer_id TEXT,
    is_admin INTEGER DEFAULT 0,
    created_at TEXT NOT NULL,
    last_login_at TEXT NOT NULL
);
```

### Migrating Mason's Existing Data

Mason's existing strategies, trades, and settings use `owner: "Mason"` (or similar). When the `users` table is added:

1. Create Mason's user record with his Google account
2. **DO NOT modify existing data.** Instead, add a mapping layer:
   - Mason's Google ID maps to owner `"Mason"` (or whatever his current owner string is)
   - New SaaS users get their Google ID or email as their owner string
3. This preserves all existing data while integrating with the new auth system

### SQLite Scalability Note

SQLite works fine for the initial launch. If the product grows to hundreds of concurrent users, consider migrating to PostgreSQL. The current codebase uses raw SQL, so migration would involve:
- Changing the connection library (sqlite3 → asyncpg/psycopg2)
- Minor SQL syntax adjustments
- Setting up PostgreSQL on the server

This is a future concern, not a launch blocker.

---

## Monitoring & Operations

### Health Monitoring

- **`/health` endpoint**: Already exists, returns cache stats and feature flags
- **Uptime monitoring**: Set up a free external monitor (UptimeRobot, Better Uptime) to ping `https://chadbacktesting.com/health` every 5 minutes
- **SSL expiry monitoring**: certbot auto-renews, but monitor cert expiry as a safety net

### Logging

- **Server logs**: `/opt/mbo_server/server.log` (uvicorn output)
- **nginx access logs**: `/var/log/nginx/access.log`
- **nginx error logs**: `/var/log/nginx/error.log`
- **Add structured logging**: After SaaS launch, add JSON logging with user ID, request path, response time for analytics

### Error Tracking

- Set up Sentry (or similar) in Phase 7 for automatic error capture
- Both server-side (Python) and client-side (JavaScript in arkon.html and SaaS pages)

### Backups

- **Database**: Automated daily backup of `strategies.db` (existing `backup_database.bat` + schedule)
- **After SaaS launch**: Increase backup frequency, add off-server backup destination
- **Market data cache**: Large but regenerable — lower backup priority
- **User uploads (screenshots)**: Should be included in backup plan

---

## arkon.html Auth Integration (for SaaS users)

When `arkon.html` is served to authenticated SaaS users at `/app`, it needs to send auth credentials with API calls. Currently it does not.

### How It Will Work

1. The session cookie (httpOnly, Secure) is set on the `chadbacktesting.com` domain
2. Since arkon.html is served from the same domain (`/app`), the browser automatically includes the cookie in all fetch() and WebSocket requests to the same origin
3. **No changes to arkon.html's `fetch()` calls are needed** if using httpOnly cookies — the browser handles it
4. The server-side auth middleware reads the cookie from the request, validates it, and extracts the user ID
5. The middleware injects the user's `owner` value before the route handler processes the request

### What DOES Need to Change in arkon.html (Eventually)

- The "Who is using Arkon?" modal (user selection) should be **removed or auto-populated** for SaaS users — they're identified by their Google account, not by manually selecting a name
- The `owner` value currently comes from a cookie/modal selection — for SaaS users it should come from the authenticated session
- The `computeBaseUrls()` function already handles HTTPS correctly (the `api.startsWith('https')` → `wss://` path works)

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

A **fully free backtesting platform** at `https://chadbacktesting.com` with premium AI-powered and educational features behind paid tiers. Branding is **GigaChad** meme-inspired - serious product, hilarious personality.

### Niche / Market Position

The core niche: **a completely free, no-strings-attached backtesting platform.** Most backtesting tools either cost money, limit features on free tiers, or gate data access. Chad Backtesting gives every user the full backtesting tool for free - load any available data, trade, journal, screenshot, tag, review - no limits.

The paid tiers (GigaChad, MegaChad) exist for advanced AI features and a **Learning Area** with interactive trading courses. The free tool is the funnel. The education and AI coaching are the upsell.

### Architecture (Confirmed Decisions)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Hosting** | Everything on Hetzner server, nginx reverse proxy | Single server simplicity, domain already pointed here |
| **Domain** | `chadbacktesting.com` | DNS already configured (A records → 95.216.5.147) |
| **SSL** | Let's Encrypt via certbot + nginx | Free, automatic renewal |
| **Auth** | Google Sign-In (OAuth 2.0) | Single sign-on, no password management needed |
| **Payments** | Stripe (keys/config to be provided by Mason later) | Industry standard, confirmed |
| **AI Provider** | Anthropic Claude Haiku (for scenario coaching in Learning Area) | Cost-efficient at scale for many users/scenarios |
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
| **Free** | $0/mo | Full backtesting tool access. Load any available data, backtest freely, save trades, strategies, trade journaling, screenshots, tags - everything the tool does today. No restrictions on core features. |
| **GigaChad** | $250/mo | Everything in Free + AI Indicator Generator + Learning Area (interactive courses with AI scenario coaching) |
| **MegaChad** | $100/mo | Everything in GigaChad + AI Chad Coach with voice-narrated retroactive analysis |

**Important: The Free tier is generous by design.** Every user gets the full backtesting experience for free with zero restrictions on the core tool. Paid tiers are for advanced AI-powered features and the educational Learning Area.

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

#### Learning Area (`/learn`) - GigaChad & MegaChad tiers
- Interactive trading courses accessible to paid subscribers
- **Any user can create their own course** and publish it for other paid users
- Courses can include text, images, videos, and most importantly: **interactive chart scenarios**
- **Scenario system**: A course author sets up a scenario by picking a specific symbol, a specific point in time, and chart configuration. The student is dropped into that exact moment in the backtesting chart and must make trading decisions.
- **AI Coaching per scenario**: After the student plays through a scenario (makes trades or observes), an AI output powered by **Claude Haiku** analyzes what they did, what the chart showed, and gives feedback/coaching. The AI sees the chart data context (timestamps, price action, indicators visible, trades placed) and provides targeted feedback.
- Course structure: modules → lessons → scenarios + content
- Leaderboards or completion tracking per course (optional)

#### Admin Panel (`/admin` - internal only)
- User management
- Subscription stats
- Usage metrics
- Course moderation

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

An AI-powered tool that generates custom trading indicators. Details TBD by Mason. This is one of the core value propositions of the GigaChad paid tier. Until Mason provides specifications and says to build it, this feature does not exist in the codebase.

#### FUTURE: Learning Area with Interactive Courses (GigaChad & MegaChad Tiers)

**Status: DO NOT BUILD. Documentation only.**

A full interactive education system where:

1. **Course Creation**: Any user (on a paid tier) can create and publish their own trading course for other paid users to take. Courses consist of modules → lessons → a mix of content and interactive chart scenarios.

2. **Interactive Chart Scenarios**: The heart of the Learning Area. A course author picks:
   - A specific trading symbol (e.g., NQ)
   - A specific point in time (a date and timestamp in the cached market data)
   - A chart configuration (timeframes, indicators, etc.)
   
   The student is dropped into that exact moment on the backtesting chart and must make trading decisions (or just observe). This leverages the existing Arkon chart and its data - the scenario just pre-loads a specific state.

3. **AI Scenario Coaching (Claude Haiku)**: After the student plays through a scenario:
   - The system captures what the chart showed (data timestamps, price action, indicator values, volume, depth, etc.)
   - The system captures what the student did (trades placed, entries, exits, P&L, timing)
   - This context is sent to **Claude Haiku** (Anthropic API) which analyzes the student's decisions against the chart data and provides targeted coaching feedback
   - Feedback is presented inline after the scenario completes
   - Claude Haiku is chosen for cost-efficiency at scale (many users, many scenarios)

4. **Course Discovery**: Browse/search published courses, filter by author, difficulty, instrument, etc.

5. **Completion Tracking**: Track which courses/scenarios a student has completed, their scores/performance.

This feature requires Mason to flesh out the full UX, the course authoring interface, scenario setup flow, and the specific Claude Haiku prompt engineering before any implementation begins.

#### FUTURE: AI Chad Coach - Voice Analysis (MegaChad Tier - $100/mo)

**Status: DO NOT BUILD. Documentation only. Mason will flesh out this feature personally.**

The most premium and differentiated feature. A coaching system that:
1. **Voice Recording During Backtesting**: While the user backtests, they narrate into their microphone everything they're factoring into their decisions (e.g., "I see a CVD divergence here, the bid depth is thinning, volume is picking up at this level...")
2. **Chart Data Correlation**: The voice narration is tied to the exact chart state - loaded data timestamps, visible indicators, price levels, volume data - everything shown on screen at the moment the user speaks
3. **Retroactive Factor Analysis**: After backtesting sessions, AI analyzes every factor the user mentioned in their narration and correlates it with their actual trade performance (entries, exits, P&L)
4. **Dynamic Graphs & Statistics**: Generates flexible visualizations and statistics showing which factors in the user's system actually correlate with winning vs losing trades, which factors they mention but don't act on, which setups they describe that lead to the best risk-adjusted returns, etc.

This is the most advanced feature. It requires significant design work from Mason before any implementation begins.

### Branding Notes

- **Name**: Chad Backtesting
- **Domain**: `chadbacktesting.com`
- **Personality**: Confident, funny, meme-forward. "Stop being a virgin trader. Backtest like a Chad."
- **Visual style**: Clean modern SaaS design with GigaChad meme imagery as accents
- **Color palette**: Dark mode primary (consistent with the dark charts), with bold accent colors
- **Tone**: Serious about the product, hilarious in copy. "Your P&L is crying. Fix it."

### Target User & UX Requirements

> Full user persona ("Jake") is documented in **AGENTS.md**.

Key UX requirements derived from persona analysis:

- **Zero-friction onboarding**: Google Sign-In → chart in under 30 seconds. No tutorial walls, no mandatory forms.
- **First-run guidance**: Subtle, skippable overlay showing "load data → take a trade → review" flow for first-time users.
- **Strategy builder/organizer**: A structured way for users to define their trading conditions, rules, and strategy plan (lives on `/dashboard` or `/strategy-builder`).
- **Statistical significance indicator**: Dashboard shows confidence level based on trade sample size so users know when they've backtested enough.
- **Simple value communication**: Landing page copy must be dead simple. No jargon. Speak directly to the user's self-interest.
- **Mobile-responsive**: All SaaS pages (landing, dashboard, billing) must work on mobile. The chart tool itself is desktop-focused.

### Product Requirements from Persona (Future - DO NOT BUILD)

> These are documented in AGENTS.md under "Product Requirements Derived from Persona".
> They are categorized as "Build Later" and must NOT be built until Mason says so.

- Motivation/accountability system (goals, reminders, schedule management)
- Text/email reminders to backtest
- AI-powered strategy formulation assistant
- Strategy condition validator ("did I follow my rules?")

See AGENTS.md for full details.

---

## Task Breakdown

### Phase 0: Foundation (Do First)

- [x] **T0.1** - Create chadbacktesting repo and push to GitHub
- [x] **T0.2** - Document full system architecture and context for agents (this file)
- [ ] **T0.3** - Decide on frontend framework
- [ ] **T0.4** - Set up project scaffold in `chadbacktesting/` based on architecture decisions

### Phase 1: Server Infrastructure (nginx, SSL, Domain, Security Hardening)

- [ ] **T1.1** - Install nginx on Hetzner server
- [ ] **T1.2** - Install certbot and obtain Let's Encrypt SSL cert for `chadbacktesting.com` + `www.chadbacktesting.com`
- [ ] **T1.3** - Deploy the production-hardened nginx config (see Domain & DNS section above) with:
  - HTTPS-only with TLSv1.2/1.3
  - HSTS, X-Frame-Options, X-Content-Type-Options, X-XSS-Protection, Referrer-Policy headers
  - HTTP → HTTPS redirect on port 80
  - 301 redirect: `www.chadbacktesting.com` → `chadbacktesting.com`
  - Reverse proxy to uvicorn on `127.0.0.1:8000`
  - WebSocket proxy for `/ws/` paths with upgrade headers
  - `client_max_body_size 10M` for screenshot uploads
  - ACME challenge passthrough for cert renewal
  - Block access to `.env` and `.git` paths
- [ ] **T1.4** - Add rate limiting zone in `/etc/nginx/nginx.conf` http block
- [ ] **T1.5** - Change `start_server_background.sh` to bind uvicorn to `127.0.0.1:8000` (not `0.0.0.0`)
- [ ] **T1.6** - Create `.env` file at `/opt/mbo_server/.env` for secrets (initially just `PVT_ACCESS_KEY`)
- [ ] **T1.7** - Add `.env` to `.gitignore` in both `arkon/` and `chadbacktesting/`
- [ ] **T1.8** - Test: `https://chadbacktesting.com/` serves the app correctly over HTTPS
- [ ] **T1.9** - Test: `https://www.chadbacktesting.com/` 301 redirects to non-www
- [ ] **T1.10** - Test: `http://chadbacktesting.com/` redirects to HTTPS
- [ ] **T1.11** - Test: WebSocket connections work through nginx (`wss://chadbacktesting.com/ws/...`)
- [ ] **T1.12** - Test: SSL Labs test (ssllabs.com) gets A or A+ rating
- [ ] **T1.13** - Update `arkon.html` `computeBaseUrls()` to handle HTTPS origin correctly
- [ ] **T1.14** - Verify certbot auto-renewal timer is active (`systemctl status certbot.timer`)
- [ ] **T1.15** - Set up UptimeRobot or similar free monitor on `https://chadbacktesting.com/health`

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

### Phase 4: Google Sign-In Authentication & Security

- [ ] **T4.1** - Set up Google Cloud project and OAuth 2.0 credentials (Client ID, Client Secret)
- [ ] **T4.2** - Store `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `SESSION_SECRET` in `/opt/mbo_server/.env`
- [ ] **T4.3** - Add `users` table to the database (see Database Migration Strategy section for schema)
- [ ] **T4.4** - Implement Google OAuth flow: `/auth/google` (redirect) → `/auth/callback` (handle code exchange) → create/find user → issue session
- [ ] **T4.5** - Implement session management with secure httpOnly cookies (Secure, SameSite=Lax, signed with SESSION_SECRET)
- [ ] **T4.6** - Build "Sign in with Google" button on landing page and any auth-required pages
- [ ] **T4.7** - Add auth middleware to protect `/app`, `/dashboard`, `/billing`, `/account` pages
- [ ] **T4.8** - Add auth middleware to protect user-specific API endpoints (`/api/strategies`, `/api/trades`, `/api/strategy_settings`, `/api/images/*`) — enforce `owner` matches authenticated user
- [ ] **T4.9** - Map authenticated user's Google ID/email to the `owner` field in strategies/trades
- [ ] **T4.10** - Create Mason's user record and map to his existing `owner` value to preserve all current data
- [ ] **T4.11** - Ensure `/pvt` route uses separate auth (secret key, not Google Sign-In)
- [ ] **T4.12** - Add WebSocket auth: validate session cookie on WS handshake, reject unauthenticated connections
- [ ] **T4.13** - Lock down CORS: change `allow_origins` from `["*"]` to `["https://chadbacktesting.com"]`
- [ ] **T4.14** - Remove or auto-populate the "Who is using Arkon?" user selection modal for SaaS users in arkon.html
- [ ] **T4.15** - All Free tier users get full backtesting access after signing in (no paywall for core features)
- [ ] **T4.16** - Add `/auth/logout` endpoint that clears the session cookie

### Phase 5: Stripe Payments (for GigaChad / MegaChad tiers only)

- [ ] **T5.1** - Mason provides Stripe API keys (test + live) - **BLOCKED until Mason provides**
- [ ] **T5.2** - Store `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET` in `.env`
- [ ] **T5.3** - Create Stripe Products and Prices: GigaChad ($250/mo), MegaChad ($100/mo)
- [ ] **T5.4** - Implement Stripe Checkout flow (Free user → upgrade to paid tier)
- [ ] **T5.5** - Implement `/api/stripe/webhook` endpoint with **signature verification** (reject unverified webhooks)
- [ ] **T5.6** - Set up Stripe Customer Portal for self-service subscription management
- [ ] **T5.7** - Store `subscription_tier` and `stripe_customer_id` on user record, update via webhook events
- [ ] **T5.8** - Build `/billing` page with current tier, upgrade options, manage subscription link
- [ ] **T5.9** - Test full flow: sign in (Free) → upgrade to GigaChad → manage → cancel → revert to Free
- [ ] **T5.10** - Paid tier features (AI Indicator Generator, Learning Area, AI Chad Coach) are NOT built yet. Gated features show "Coming Soon" until Mason says to build them.
- [ ] **T5.11** - Handle edge cases: payment failure → grace period → downgrade to Free; duplicate webhook delivery; subscription upgrade/downgrade between GigaChad ↔ MegaChad

### Phase 6: Multi-tenant Backtesting & Dashboard

- [ ] **T6.1** - Enforce data isolation: every API query filters by authenticated user's `owner` value
- [ ] **T6.2** - Implement user-specific screenshot storage paths (`/uploads/<user_id>/`)
- [ ] **T6.3** - Build `/dashboard` with user's strategies, trade count, win rate, P&L summary
- [ ] **T6.4** - Build `/account` page with Google profile info, subscription tier, sign-out
- [ ] **T6.5** - Build `/admin` panel (Mason-only): user list, subscription breakdown, total trades, storage usage
- [ ] **T6.6** - Harden destructive endpoints: `/api/cache/clear` and `/api/trades/purge` require admin auth only
- [ ] **T6.7** - Add per-user screenshot storage limits (e.g., 500MB per user)

### Phase 7: Polish & Launch

- [ ] **T7.1** - SEO: meta tags (Open Graph, Twitter Card), sitemap.xml, robots.txt
- [ ] **T7.2** - Custom 404 and error pages (GigaChad themed)
- [ ] **T7.3** - Error monitoring: Sentry for both Python server and JavaScript client
- [ ] **T7.4** - Analytics: Google Analytics or PostHog on landing page and dashboard
- [ ] **T7.5** - Legal pages: Terms of Service, Privacy Policy (required for Google OAuth and Stripe)
- [ ] **T7.6** - Performance testing: load test with multiple concurrent WebSocket connections
- [ ] **T7.7** - Security audit: run SSL Labs test, verify all headers, check CORS, review auth flows
- [ ] **T7.8** - Automated database backup (daily cron job on server, backup to off-server location)
- [ ] **T7.9** - Log rotation for `server.log` and nginx logs (logrotate config)
- [ ] **T7.10** - Favicon and social media preview images (GigaChad branded)
- [ ] **T7.11** - Public launch

### Phase 8: Future Premium Features (DO NOT START - MASON WILL INITIATE)

> **These phases are placeholders only. Do not begin work on any of them.**
> **Mason will personally flesh out specs and explicitly request implementation.**

#### 8A: Learning Area & Interactive Courses (GigaChad + MegaChad tiers)
- [ ] **T8A.1** - Course data model (courses, modules, lessons, scenarios) and database schema
- [ ] **T8A.2** - Course authoring interface (create/edit courses, add scenarios)
- [ ] **T8A.3** - Scenario setup: pick symbol, timestamp, chart config → save as scenario
- [ ] **T8A.4** - Scenario playback: student loads scenario → dropped into chart at that moment
- [ ] **T8A.5** - Claude Haiku integration: capture chart data + student actions → send to Haiku → display coaching feedback
- [ ] **T8A.6** - Course discovery, browsing, search, filtering
- [ ] **T8A.7** - Course completion tracking and student progress
- [ ] **T8A.8** - User-generated course publishing flow and moderation

#### 8B: AI Indicator Generator (GigaChad tier)
- [ ] **T8B.1** - Mason will provide specs - DO NOT BUILD

#### 8C: AI Chad Coach - Voice Analysis (MegaChad tier)
- [ ] **T8C.1** - Voice recording during backtesting sessions - Mason will provide specs
- [ ] **T8C.2** - Chart data + voice narration timestamp correlation - Mason will provide specs
- [ ] **T8C.3** - Retroactive factor analysis engine - Mason will provide specs
- [ ] **T8C.4** - Dynamic graph/statistics generation from factor analysis - Mason will provide specs

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

### Infrastructure (nginx, SSL, Domain, Security)
1. Server: Install nginx, certbot via SSH
2. Server: `/etc/nginx/sites-available/chadbacktesting.com` — production-hardened config (see Domain & DNS section)
3. Server: `/etc/nginx/nginx.conf` — add rate limiting zone in `http {}` block
4. Server: Symlink sites-available → sites-enabled
5. `arkon/start_server_background.sh` — change `0.0.0.0` → `127.0.0.1` after nginx is set up
6. Server: `/opt/mbo_server/.env` — create with all secrets (see Security section for full list)
7. Server: Set up logrotate for server.log and nginx logs
8. Server: Set up automated daily database backup cron job

### Initiative 1 (Private Route)
1. `arkon/mbo_streaming_server.py` — Add `/pvt` route (~line 1334), add `.env` loading for `PVT_ACCESS_KEY`
2. `arkon/update_frontend.sh` — Update URL references
3. `arkon/update_frontend.bat` — Update URL references
4. `arkon/deploy_mbo_server.bat` — Update URL references
5. `arkon/deploy_mbo_server.sh` — Update URL references

### Initiative 2 (SaaS Product)
1. `chadbacktesting/` — SaaS-only code (landing page, dashboard, billing pages, deployment scripts, nginx configs, brand assets)
2. `arkon/mbo_streaming_server.py` — Major additions:
   - Google OAuth routes (`/auth/google`, `/auth/callback`, `/auth/logout`)
   - `users` table and user management
   - Auth middleware (session validation on protected routes)
   - Owner enforcement on user-specific API endpoints
   - WebSocket auth on handshake
   - Stripe webhook endpoint with signature verification
   - CORS lockdown from `*` to `chadbacktesting.com`
   - Landing page and SaaS page serving routes
   - Admin routes
3. `arkon/arkon.html` — Remove/auto-populate "Who is using Arkon?" modal for SaaS users; owner auto-set from session
4. Server: nginx config, SSL certs, rate limiting, security headers

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
