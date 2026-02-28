# Project State Tracker

Last updated: 2026-02-28
Current phase: 1
Current round: 0 (not started)

---

## Phase 0: Foundation
**Status**: COMPLETE

| Task | Status | Notes |
|------|--------|-------|
| T0.1 — Create repo, push to GitHub | DONE | github.com/masonmahaffey/chadbacktesting |
| T0.2 — Document system architecture | DONE | PLAN.md written |
| T0.3 — Decide on frontend framework | TODO | Open decision |
| T0.4 — Set up project scaffold | TODO | Blocked on T0.3 |

---

## Phase 1: Server Infrastructure (nginx, SSL, Domain, Security Hardening)
**Status**: NOT STARTED

| Task | Status | Assignee | Files Touched | Blocker | Notes |
|------|--------|----------|---------------|---------|-------|
| T1.1 — Install nginx on Hetzner | TODO | — | Server: apt install | — | |
| T1.2 — Obtain Let's Encrypt SSL cert | TODO | — | Server: certbot | T1.1 | Needs nginx first |
| T1.3 — Deploy hardened nginx config | TODO | — | Server: /etc/nginx/ | T1.2 | Needs cert first |
| T1.4 — Add rate limiting zone | TODO | — | Server: /etc/nginx/nginx.conf | T1.3 | |
| T1.5 — Bind uvicorn to 127.0.0.1 | TODO | — | arkon/start_server_background.sh | T1.3 | After nginx proxies |
| T1.6 — Create .env on server | TODO | — | Server: /opt/mbo_server/.env | — | Parallel with T1.1 |
| T1.7 — Add .gitignore entries | TODO | — | arkon/.gitignore, chadbacktesting/.gitignore | — | Parallel with T1.1 |
| T1.8 — Test HTTPS loads | TODO | — | — | T1.3 | Verify only |
| T1.9 — Test www→non-www redirect | TODO | — | — | T1.3 | Verify only |
| T1.10 — Test HTTP→HTTPS redirect | TODO | — | — | T1.3 | Verify only |
| T1.11 — Test WebSocket through nginx | TODO | — | — | T1.3 | Verify only |
| T1.12 — SSL Labs test A/A+ | TODO | — | — | T1.3 | Verify only |
| T1.13 — Update arkon.html computeBaseUrls() | TODO | — | arkon/arkon.html | T1.3 | HTTPS origin handling |
| T1.14 — Verify certbot auto-renewal | TODO | — | — | T1.2 | Verify only |
| T1.15 — Set up uptime monitor | TODO | — | External service | T1.8 | After HTTPS works |

### Parallel Groups
- **Group A** (no deps): T1.1, T1.6, T1.7
- **Group B** (needs T1.1): T1.2
- **Group C** (needs T1.2): T1.3, T1.14
- **Group D** (needs T1.3): T1.4, T1.5, T1.8, T1.9, T1.10, T1.11, T1.12, T1.13
- **Group E** (needs T1.8): T1.15

---

## Phase 2: Private Route Migration
**Status**: NOT STARTED

| Task | Status | Assignee | Files Touched | Blocker | Notes |
|------|--------|----------|---------------|---------|-------|
| T2.1 — Add /pvt route | TODO | — | arkon/mbo_streaming_server.py | Phase 1 done | |
| T2.2 — Verify /pvt works | TODO | — | — | T2.1 | QA |
| T2.3 — Deploy to Hetzner | TODO | — | Server | T2.2 | Needs Mason approval |
| T2.4 — Verify production /pvt | TODO | — | — | T2.3 | QA |
| T2.5 — Update deploy scripts | TODO | — | arkon/update_frontend.sh, .bat | T2.4 | |
| T2.6 — Keep / serving arkon.html | TODO | — | — | — | Verify, don't break |

---

## Phases 3-8: NOT STARTED
See PLAN.md for full task lists.

---

## Review Status

| Review | Phase | Status | File |
|--------|-------|--------|------|
| — | — | — | No reviews yet |

---

## Mason Approval Queue

| Item | Status | Notes |
|------|--------|-------|
| T0.3 — Frontend framework decision | WAITING | Need Mason's input |
| Phase 1 deployment to production | WAITING | After Phase 1 implementation + reviews pass |
| Phase 2 deployment to production | WAITING | After Phase 2 implementation + reviews pass |
| Stripe API keys | WAITING | Mason provides when ready (Phase 5) |
| Google OAuth credentials | WAITING | Mason provides when ready (Phase 4) |
