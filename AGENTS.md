# Chad Backtesting - Agent Execution System

> This document defines the complete agent-driven development, testing, and iteration system
> for building Chad Backtesting. It is designed so that agents can be spun up in parallel,
> each with a specific role, and they feed into a recursive loop of build → test → review → fix.

---

## Table of Contents

1. [How This Works in Cursor (Practical Guide)](#how-this-works-in-cursor-practical-guide)
2. [Mason's Approval Workflow](#masons-approval-workflow)
3. [Target User Persona](#target-user-persona)
4. [Product Requirements Derived from Persona](#product-requirements-derived-from-persona)
5. [Agent Roles](#agent-roles)
6. [Agent Flow Architecture](#agent-flow-architecture)
7. [The Recursive Build Loop](#the-recursive-build-loop)
8. [Context & Document System](#context--document-system)
9. [Phase Execution Playbooks](#phase-execution-playbooks)
10. [Parallel Execution Strategy](#parallel-execution-strategy)

---

## How This Works in Cursor (Practical Guide)

### Hybrid Execution: Cloud Agents + Local Composer

This project uses **two types** of Cursor agents depending on what the task touches:

| Type | What It Is | When to Use |
|------|-----------|-------------|
| **Cloud Agent** | Runs on Cursor's cloud VMs. Clones the `chadbacktesting` GitHub repo, works on its own branch, pushes changes, creates PRs. Has its own VM to build/test/run code. Can be kicked off from phone, web, Slack, or GitHub. | Any task that only touches files in the `chadbacktesting/` repo (SaaS pages, configs, reviews, task specs) |
| **Local Composer** | Runs in your Cursor desktop IDE. Has direct access to your local filesystem and SSH keys. | Any task that touches `arkon/` files, or that needs to SSH into the Hetzner server |

#### Why the split?

- Cloud agents only see the `chadbacktesting` GitHub repo. They **cannot** access `arkon/mbo_streaming_server.py`, `arkon/arkon.html`, or your SSH keys.
- Local Composers have access to everything on your machine — both repos and SSH.
- Cloud agents can run **unlimited in parallel** with zero machine resource cost.
- Local Composers share your machine's resources (stick to 2-3 max).

#### What runs where, by phase

| Phase | Cloud Agents | Local Composer |
|-------|-------------|----------------|
| **Phase 1** (nginx/SSL) | Write nginx config templates, .gitignore | SSH into server to install/configure nginx, certbot, deploy configs |
| **Phase 2** (private route) | — | Modify `arkon/mbo_streaming_server.py`, deploy via SSH |
| **Phase 3** (landing page) | Build entire landing page, all HTML/CSS/JS, brand assets | — |
| **Phase 4** (auth) | Build auth page templates, dashboard pages | Modify `arkon/mbo_streaming_server.py` (add OAuth routes, middleware) |
| **Phase 5** (Stripe) | Build billing pages, Stripe frontend integration | Modify `arkon/mbo_streaming_server.py` (add webhook endpoint) |
| **Phase 6** (multi-tenant) | Build dashboard, admin panel pages | Modify `arkon/mbo_streaming_server.py` (data isolation middleware) |
| **Phase 7** (polish) | SEO, error pages, legal pages, analytics | Deploy scripts, server config |
| **All reviews** (QA, Gap, Persona, Security) | All run as cloud agents — read-only analysis, can run many in parallel | — |

### The State Tracker: `TRACKER.md`

Single source of truth for project state. Lives in the `chadbacktesting` GitHub repo so both cloud and local agents can read it.

**Every agent's first action**: Read TRACKER.md. Find your task. Update status.

**Format**:
```markdown
# Project State Tracker
Last updated: [ISO timestamp]
Current phase: [N]
Current round: [M]

## Phase [N]: [Name]

| Task | Status | Where | Files | Blocker | Notes |
|------|--------|-------|-------|---------|-------|
| T1.1 | DONE | LOCAL | server: nginx | — | Done round 1 |
| T1.2 | IN_PROGRESS | LOCAL | server: certbot | T1.1 | Running now |
| T1.3 | TODO | CLOUD | chadbacktesting/nginx/ | — | Config template |
| T1.4 | BLOCKED | LOCAL | — | T1.2 | Needs cert first |
```

The `Where` column tells every agent (and you) whether this task runs as a cloud agent or local Composer. No guessing.

### Self-Contained Task Specs

Each task spec the Architect writes is self-contained — includes only the context that agent needs, excerpted from PLAN.md, not the whole 1,279-line file.

A task spec also declares its execution type:

```markdown
# Task T1.3: Obtain SSL Certificate
**Phase**: 1 — Server Infrastructure
**Execution**: LOCAL (requires SSH to Hetzner server)
**Dependencies**: T1.1 (nginx installed) must be DONE
**Files to modify**: /etc/nginx/sites-available/chadbacktesting.com (on server via SSH)

## Context (excerpted from PLAN.md)
- Server: 95.216.5.147, SSH as root, key at ~/.ssh/hetzner_server_key
- Domain: chadbacktesting.com + www.chadbacktesting.com
- Certbot should cover both domains in one cert

## What to Do
1. SSH into the server
2. Run: certbot --nginx -d chadbacktesting.com -d www.chadbacktesting.com
3. Verify cert was obtained
4. Verify auto-renewal timer

## Acceptance Criteria
- [ ] SSL cert exists for both domains
- [ ] certbot.timer is active
- [ ] https://chadbacktesting.com loads

## When Done
Update TRACKER.md: set T1.3 status to DONE.
Write status/T1.3-status.md confirming what was done.
```

```markdown
# Task T3.2: Build Landing Page Hero Section
**Phase**: 3 — SaaS Landing Page
**Execution**: CLOUD (chadbacktesting repo only)
**Branch**: phase-3/T3.2-hero-section
**Dependencies**: T3.1 (brand assets) must be DONE
**Files to create/modify**: chadbacktesting/src/landing/hero.html, hero.css

## Context
- GigaChad branding, dark mode, bold accents
- See PLAN.md "Branding Notes" and AGENTS.md "Target User Persona"
- Hero must communicate: "Free backtesting. No BS."

## What to Do
1. Create hero section with GigaChad imagery
2. "Sign in with Google" CTA button
3. One-line value prop that speaks to Jake
4. Mobile-responsive

## Acceptance Criteria
- [ ] Hero section renders on desktop and mobile
- [ ] CTA button is prominent
- [ ] GigaChad branding is visible and fun
- [ ] Copy is dead simple, no jargon

## When Done
Push branch, create PR against main.
Update TRACKER.md: set T3.2 status to DONE.
```

### How Merge Conflicts Are Prevented

Cloud agents each work on their **own branch** (e.g., `phase-3/T3.2-hero-section`). The Architect ensures:

1. **No two cloud agents modify the same file** in parallel — different branches changing the same file = merge conflict when merging PRs
2. **Cloud agents never touch `arkon/` files** — those are local-only
3. **Local Composers work on `main` directly** (or a single local branch) since they're sequential anyway
4. **PRs from cloud agents are merged in dependency order** — the Architect's parallel groups define merge order

| Agent Type | Works On | Merges How |
|------------|----------|------------|
| Cloud Agent A | Branch `phase-3/T3.2-hero` | PR → merge after review |
| Cloud Agent B | Branch `phase-3/T3.3-pricing` | PR → merge after review |
| Cloud Agent C | Branch `phase-3/T3.4-features` | PR → merge after review |
| Local Composer | `main` (or local branch) | Direct commit or local PR |

### What You (Mason) Actually Do

**For phases that are mostly cloud (Phase 3, 5, 6, 7):**
1. Open `cursor.com/agents` on your phone or browser
2. Kick off the Architect agent: "Read TRACKER.md and PLAN.md. Produce task specs for Phase 3."
3. Once specs are written, kick off cloud Coder agents — one per task that can run in parallel
4. They each work on their own branch, push, and create PRs
5. Kick off cloud QA and review agents to analyze the PRs
6. Merge the PRs when reviews pass
7. Check TRACKER.md for status at any time

**For phases that are mostly local (Phase 1, 2):**
1. Open a local Composer in Cursor desktop
2. Paste the Orchestrator prompt (below)
3. It handles SSH commands, file edits, and deployment to the Hetzner server
4. You approve deployment checkpoints

**You can run cloud and local agents simultaneously.** For example: a local Composer is deploying Phase 1 nginx config to the server while cloud agents are already building Phase 3 landing page code in parallel.

### The Orchestrator Prompt (for local phases)

Use this for phases that need SSH / arkon/ access:

```
Read chadbacktesting/TRACKER.md, chadbacktesting/AGENTS.md, and chadbacktesting/PLAN.md.

You are the Orchestrator for the Chad Backtesting project (LOCAL mode).
You have access to the local filesystem and SSH keys.

Your job:
1. Read TRACKER.md to understand current state
2. Work through LOCAL tasks for the current phase
3. For tasks needing SSH: execute commands on the Hetzner server (95.216.5.147)
4. For tasks modifying arkon/ files: edit them directly
5. After each task, update TRACKER.md
6. When the phase's local tasks are done, run QA checks
7. At approval checkpoints (especially deployment), STOP and ask Mason

RULES:
- NEVER wipe or overwrite strategies.db on the server
- NEVER break the existing backtesting tool at /pvt or /
- Always update TRACKER.md after any state change
- At deployment checkpoints, STOP and ask Mason for go/no-go
```

### The Cloud Agent Prompt Template

Use this when kicking off cloud agents (from web, Slack, or GitHub):

```
Read chadbacktesting/TRACKER.md.

You are a [AGENT ROLE] for the Chad Backtesting project.
Current phase: Phase [N].
Your task: Read chadbacktesting/tasks/phase-[N]/[task-id].md and execute it.

Work on branch: phase-[N]/[task-id]
When done: push your branch, create a PR against main, and update TRACKER.md.

RULES:
- Only modify files listed in the task spec
- Only modify files in the chadbacktesting/ repo (never arkon/)
- Follow acceptance criteria exactly
```

---

## Mason's Approval Workflow

### What Needs Mason's Approval

Not everything runs autonomously. These checkpoints require Mason to explicitly sign off:

| Checkpoint | When | What Mason Does |
|------------|------|-----------------|
| **Phase completion** | After feedback aggregation shows all-pass | Read the feedback summary, say "proceed to Phase N+1" or "fix X first" |
| **Production deployment** | After any phase that changes the live server | Review what will be deployed, confirm "deploy it" |
| **Product ideas** | After Product Ideation Agent writes suggestions | Read `reviews/ideas-*.md`, approve/reject/defer each idea |
| **Architecture decisions** | When Architect flags an open question | Answer the question so the Coder can proceed |
| **Blocked tasks** | When a task needs info only Mason has (Stripe keys, Google OAuth creds, etc.) | Provide the info |
| **Scope changes** | When any agent suggests adding something not in PLAN.md | Approve or reject before it gets built |

### How Product Ideation Approval Works

The Product Ideation Agent writes ideas to `reviews/ideas-[date].md`. Each idea has this format:

```markdown
### Idea #1: [Name]
**Description**: One-line summary
**Why Jake cares**: [In Jake's voice]
**Effort**: small / medium / large
**Priority suggestion**: must-have / should-have / nice-to-have
**Status**: PENDING MASON REVIEW
```

**Your workflow:**

1. Open `reviews/ideas-[date].md` when you have a minute
2. For each idea, change the `Status` line to one of:
   - `APPROVED` — tell an Architect Agent to scope it into a task spec
   - `APPROVED - LATER` — good idea, but not now. It stays documented.
   - `REJECTED` — won't build this. Add a one-line reason if you want.
   - `NEEDS DISCUSSION` — you want to modify the idea before deciding
3. Only `APPROVED` ideas get turned into tasks. Everything else stays parked.

**Example:**
```markdown
### Idea #3: Streak tracker
**Description**: Show a "backtest streak" counter — how many days in a row you've practiced
**Why Jake cares**: "I'd want to keep my streak going, like Duolingo"
**Effort**: small
**Priority suggestion**: should-have
**Status**: APPROVED - LATER
```

Mason changed the status from `PENDING MASON REVIEW` to `APPROVED - LATER`. No agent will build this until Mason says "go build idea #3 from the ideas doc."

### How to Communicate Decisions Back to Agents

When you approve/reject things or make decisions, you have two options:

**Option A (simple)**: Just tell the next agent in your Composer prompt. Example:
```
You are the Architect Agent. Read PLAN.md and AGENTS.md.
Phase 3 is complete. Begin Phase 4.
Also: I approved ideas #2 and #5 from reviews/ideas-2026-03-15.md.
Scope them into Phase 4 tasks alongside the existing task list.
```

**Option B (tracked)**: Write your decision to a `feedback/mason-decisions-[date].md` file and tell agents to read it. Better for traceability.

---

## Target User Persona

### "Jake" — The Aspiring Day Trader

**Demographics**: Young male (20s-30s), works a blue-collar or white-collar job. Needs to make more money. Limited free time outside work and responsibilities.

**Psychology**:
- High self-confidence — believes he can make money trading
- Attracted to the independence: own your income, no boss, work remotely
- Thinks the GigaChad meme is genuinely funny
- **Lazy by self-admission** — won't do things that feel like a lot of effort
- Needs to be shown value of new features **very simply** or he'll ignore them
- Gets motivated by wins but needs external accountability to stay consistent

**Trading Background**:
- Has watched content from **Fabio Valentini** (order flow basics), **ICT / Inner Circle Trader** (zones, market structure, liquidity concepts), possibly **Timothy Sykes** and **Warrior Trading**
- Has a basic understanding of ICT concepts (order blocks, fair value gaps, liquidity sweeps, etc.)
- Does NOT have a well-defined strategy yet
- Knows he needs to backtest to develop and validate a strategy but hasn't done it systematically
- Needs help getting **organized** — formulating conditions, rules, and a plan

**What Jake Needs (in his own words)**:
1. A place to just **practice** — load data, click through charts, take trades, build intuition
2. Help **formulating a strategy** — turning vague ideas ("I like ICT zones") into specific, testable conditions
3. Once he has conditions, a way to **backtest that strategy across a ton of data** to reach statistical significance
4. **Motivation and accountability** — goals, reminders, a schedule, someone/something nudging him to actually sit down and backtest
5. **Custom indicators** — he's seen what TradingView has and wants similar flexibility
6. Everything needs to be **dead simple** — if it's complicated, he's out

**What Jake Does NOT Want**:
- Complicated setup or onboarding
- Features that aren't obviously useful
- To pay money before he sees value
- To feel like he's doing homework

**How Jake Discovers Chad Backtesting**:
- Probably through trading Twitter/X, Reddit (r/daytrading, r/futurestrading), YouTube trading community
- The GigaChad branding catches his eye and makes him laugh
- "Free backtesting? Sick. Let me try it."
- He signs in with Google (one click), immediately gets to the chart, and starts clicking around

**Jake's Journey Through the Product**:
1. **Day 1**: Signs in, loads a random day, clicks around, takes a few trades. "This is cool."
2. **Week 1**: Uses it a few more times. Starts to get a feel for it. Maybe creates a strategy.
3. **Week 2-4**: Backtests more regularly. Starts noticing patterns in his trades. Wants to get more organized.
4. **Month 2+**: Has a rough strategy. Wants to validate it with data. Sees the value in paid features (courses, AI coaching, custom indicators) because now he has context for why they'd help.

---

## Product Requirements Derived from Persona

These are features/improvements identified by analyzing Jake's needs. They are categorized by priority.

### Build Now (Core Product - Phases 1-7)

| Requirement | Why Jake Needs It | Where It Lives |
|-------------|-------------------|----------------|
| **Zero-friction onboarding** | "I'm kinda lazy" — Google Sign-In, immediately into the tool, no tutorial walls | Landing page → Google Sign-In → Chart in <30 seconds |
| **First-run guidance** | Jake doesn't know what buttons do. A subtle, skippable "here's how to load data and take your first trade" overlay | arkon.html (or a lightweight wrapper) |
| **Strategy builder / organizer** | Jake has vague ideas but no system. Needs a structured way to define conditions, rules, and a plan for his strategy | `/dashboard` or a dedicated `/strategy-builder` page |
| **Statistical significance indicator** | Jake needs to know "have I backtested enough data to trust these results?" | Dashboard trade stats — show confidence level based on sample size |
| **Simple value props on landing page** | "show me the value very simply" — landing page must instantly communicate why this tool matters, in Jake's language | Landing page copy and design |
| **Mobile-responsive design** | Jake has limited time, might check his dashboard or review trades on his phone | All SaaS pages (landing, dashboard, billing). The chart tool itself is desktop-focused. |

### Build Later (Paid Features - Phase 8+)

> **DO NOT BUILD these until Mason says so.** Documented here for product vision only.

| Requirement | Why Jake Needs It | Tier |
|-------------|-------------------|------|
| **Motivation/accountability system** | Jake is lazy and needs nudges. Goals, schedules, text/email reminders to backtest. If he misses a day, reschedule. Gamification. | GigaChad or MegaChad |
| **Custom indicators (TradingView-style)** | Jake has seen TradingView indicators and wants similar. Ties into AI Indicator Generator. | GigaChad ($250/mo) |
| **Strategy condition validator** | Once Jake defines conditions, the system helps him check: "did I follow my rules on this trade?" | GigaChad or MegaChad |
| **AI-powered strategy formulation** | Jake doesn't have a strategy. AI could interview him about what he's seen work and help him codify it into testable rules. | MegaChad ($100/mo) |
| **Text/email reminders** | "I need to be reached out to via text or email to go backtest" — scheduled nudges with motivational GigaChad energy | MegaChad ($100/mo) |
| **Schedule management** | Create a backtesting schedule. If you miss a day, it automatically reschedules and sends encouragement. | MegaChad ($100/mo) |
| **Learning Area courses** | Jake has watched ICT/Fabio content but needs structured practice with scenarios | GigaChad + MegaChad |

---

## Agent Roles

Each agent has a defined identity, responsibilities, tools, and output format.

### 1. Architect Agent

**Role**: Technical lead. Reads the plan, breaks the current phase into specific implementable tasks, assigns context to Coder Agent(s). Reviews completed work at a high level.

**When to Use**: At the start of each phase, and when Coder needs re-scoping after feedback.

**Input**: PLAN.md (current phase), any feedback from QA/Gap/Persona agents.

**Output**: A task spec written to `tasks/<phase>/<task-id>.md` with:
- Exact files to modify (with file paths)
- What to change (specific functions, routes, HTML sections)
- Acceptance criteria (what "done" looks like)
- Dependencies (what must exist before this task)
- Test criteria (how QA Agent will verify)

**Prompt Template**:
```
You are the Architect Agent for the Chad Backtesting project.

Read PLAN.md and AGENTS.md for full context. The current phase is [PHASE X].

Your job:
1. Read the task list for this phase
2. Break each task into a specific, implementable spec
3. Write the spec to tasks/[phase]/[task-id].md
4. Each spec must include: files to modify, exact changes, acceptance criteria, test criteria
5. Consider dependencies between tasks and mark which can run in parallel

Do NOT write code. Only write specs.
```

---

### 2. Coder Agent

**Role**: Implementation. Takes a task spec from Architect and writes the code.

There are two variants:

| Variant | Runs Where | Can Access |
|---------|-----------|------------|
| **Cloud Coder** | Cursor Cloud VM | `chadbacktesting/` repo only (cloned from GitHub) |
| **Local Coder** | Cursor desktop IDE | Both `arkon/` and `chadbacktesting/` + SSH to Hetzner server |

The task spec's `**Execution**` field determines which variant runs the task.

**When to Use**: After Architect has produced a task spec.

**Input**: A specific `tasks/<phase>/<task-id>.md` spec + relevant excerpts from PLAN.md (included in the spec).

**Output**:
- Cloud Coder: pushes branch, creates PR, writes `status/<task-id>.md`
- Local Coder: commits directly, writes `status/<task-id>.md`

**Prompt Template (Cloud Coder)**:
```
You are a Cloud Coder Agent for the Chad Backtesting project.

Read the task spec at tasks/[phase]/[task-id].md.
Read AGENTS.md section "Target User Persona" to understand who you're building for.

Your job:
1. Create branch: phase-[N]/[task-id]
2. Implement exactly what the spec describes
3. Only modify files listed in the spec (all within chadbacktesting/)
4. Write clean code with no unnecessary comments
5. Push your branch and create a PR against main
6. Write a brief status update to status/[task-id].md

CRITICAL RULES:
- NEVER touch arkon/ files (you don't have access anyway)
- All secrets go in .env, never in code
- Follow acceptance criteria exactly
```

**Prompt Template (Local Coder)**:
```
You are a Local Coder Agent for the Chad Backtesting project.

Read the task spec at tasks/[phase]/[task-id].md.
Read PLAN.md for system architecture context.

Your job:
1. Implement exactly what the spec describes
2. For server changes: SSH into 95.216.5.147 as root
3. For arkon/ changes: edit files directly
4. Do NOT touch the database on the Hetzner server or overwrite strategies.db
5. All Arkon chart code stays in arkon/ — never copy to chadbacktesting/
6. Write clean code with no unnecessary comments
7. Write a brief status update to status/[task-id].md

CRITICAL RULES:
- The private backtesting at /pvt must keep working at all times
- Never wipe or overwrite the production database
- All secrets go in .env, never in code
- All traffic must go through HTTPS
```

---

### 3. QA Agent

**Role**: Tester. Runs the product, clicks through flows, tests API endpoints, verifies features work. Finds bugs. Writes detailed bug reports.

**When to Use**: After Coder Agent completes a task or set of tasks.

**Input**: The task spec (acceptance criteria), the deployed/running product.

**Output**: A test report written to `qa/<task-id>-qa.md` with:
- Each acceptance criterion: PASS / FAIL
- Steps to reproduce any failures
- Screenshots or error logs if applicable
- Severity: critical / major / minor / cosmetic

**Prompt Template**:
```
You are the QA Agent for the Chad Backtesting project.

Read the task spec at tasks/[phase]/[task-id].md for acceptance criteria.
Read PLAN.md for expected behavior.

Your job:
1. Test every acceptance criterion in the spec
2. For each criterion, mark PASS or FAIL
3. For failures, provide exact steps to reproduce, error messages, and severity
4. Also test adjacent functionality that might have been broken (regression)
5. Test edge cases: empty inputs, long strings, special characters, rapid clicks, network issues
6. Write your report to qa/[task-id]-qa.md

TESTING APPROACH:
- Start the server locally or verify on production
- Test both happy path and error paths
- Check browser console for JavaScript errors
- Check server logs for Python errors
- Verify HTTPS, cookies, headers if applicable
```

---

### 4. Gap Analyst Agent

**Role**: Compares what was implemented against what the plan specified. Finds discrepancies, missing pieces, and deviations. This is the "did we actually build what we said we'd build?" check.

**When to Use**: After QA Agent passes a set of tasks, before moving to the next phase.

**Input**: PLAN.md, the task specs, the actual code that was written, QA reports.

**Output**: A gap report written to `reviews/gap-[phase].md` with:
- For each task in the phase: implemented as specified? partially? deviates?
- Missing features that the plan called for but weren't built
- Features that were built but aren't in the plan (scope creep)
- Security requirements from the Security section: met or not?
- Recommendations for what to fix before moving forward

**Prompt Template**:
```
You are the Gap Analyst Agent for the Chad Backtesting project.

Read PLAN.md (the source of truth for what should exist).
Read the task specs in tasks/[phase]/.
Read the QA reports in qa/.
Read the actual code that was implemented.

Your job:
1. For every task in this phase, verify the implementation matches the plan
2. Check the Security section of PLAN.md — are all security requirements met?
3. Check the API Endpoint Protection Matrix — are auth rules correctly enforced?
4. Check the Database Schema section — does the actual schema match?
5. List any gaps, deviations, or missing pieces
6. List any scope creep (things built that aren't in the plan)
7. Write your report to reviews/gap-[phase].md

Be ruthlessly thorough. The plan is the contract.
```

---

### 5. User Persona Agent ("Jake")

**Role**: Thinks through the product as Jake (the target user). Tries every flow from Jake's perspective. Identifies UX friction, confusing UI, missing guidance, moments where Jake would give up or get frustrated.

**When to Use**: After Gap Analyst confirms implementation matches plan. This agent checks whether what we built is actually GOOD for the user.

**Input**: The running product, AGENTS.md (Jake's persona), PLAN.md.

**Output**: A UX review written to `reviews/persona-[phase].md` with:
- Jake's journey through each feature (narrative format)
- Friction points: "Jake would get confused here because..."
- Drop-off risks: "Jake would leave the site here because..."
- Delight moments: "Jake would think this is cool because..."
- Suggestions for improvement (categorized as: fix now / fix later / nice-to-have)

**Prompt Template**:
```
You are Jake — the target user of Chad Backtesting. Read your full persona in AGENTS.md.

You are a young guy who works a regular job, wants to make money trading, thinks the GigaChad
meme is funny, is kind of lazy, has watched ICT and Fabio Valentini content, and needs a free
backtesting tool. You don't have a strategy yet. You need to practice.

Walk through the product as yourself:
1. You land on chadbacktesting.com for the first time. What do you see? Does it make you want to sign up?
2. You sign in with Google. How long does it take to get to the chart? Is it confusing?
3. You try to load data and take your first trade. Can you figure it out without a tutorial?
4. You come back the next day. Is there anything pulling you back?
5. After a week, you want to organize your strategy. Can you?
6. You see the pricing page. Does the free tier feel truly free? Are the paid tiers tempting?

For each step, write:
- What you did
- How it felt (easy? confusing? cool? annoying?)
- Would you continue or leave?
- What would make it better?

Write your review to reviews/persona-[phase].md.

Be honest. If something sucks, say it sucks. You're lazy, remember. You don't tolerate friction.
```

---

### 6. Security Reviewer Agent

**Role**: Reviews all code changes for security vulnerabilities. Checks SSL configuration, auth flows, cookie settings, CORS, input validation, SQL injection, XSS, CSRF, etc.

**When to Use**: After each phase is implemented, before deployment to production.

**Input**: All code changes in the phase, PLAN.md Security section, nginx config, server config.

**Output**: Security audit written to `reviews/security-[phase].md` with:
- Each security requirement from PLAN.md: MET / NOT MET / PARTIAL
- Vulnerabilities found (with severity and fix recommendation)
- SSL/TLS verification results
- Auth flow verification
- CORS policy verification

**Prompt Template**:
```
You are the Security Reviewer Agent for Chad Backtesting.

Read the Security, SSL & Auth Architecture section of PLAN.md.
Review all code changes made in Phase [X].

Check for:
1. SSL/TLS: Is all traffic HTTPS? Are certificates valid? HSTS enabled?
2. Auth: Are protected routes actually protected? Can you bypass auth?
3. Cookies: httpOnly? Secure? SameSite? Properly signed?
4. CORS: Is it locked down to chadbacktesting.com?
5. SQL Injection: Are all queries parameterized?
6. XSS: Is user input sanitized before rendering?
7. CSRF: Are state-changing requests protected?
8. File uploads: Type validation? Size limits? Path traversal prevention?
9. Secrets: Are any secrets hardcoded in source code? Is .env in .gitignore?
10. Rate limiting: Is it configured in nginx?
11. Stripe webhooks: Is signature verification implemented?
12. WebSocket: Are connections authenticated?

Write your audit to reviews/security-[phase].md.
Flag anything CRITICAL as a blocker — it must be fixed before deployment.
```

---

### 7. Product Ideation Agent

**Role**: After the core product is stable, this agent thinks about what's missing from the perspective of growing the user base. It generates feature ideas, growth experiments, and product improvements — but ONLY documents them. It does NOT decide to build anything.

**When to Use**: After Phase 7 launch, or periodically between phases.

**Input**: Jake's persona, user journey, existing feature set, competitor analysis.

**Output**: Ideas document written to `reviews/ideas-[date].md`. All ideas are suggestions only — Mason decides what gets built.

**Prompt Template**:
```
You are the Product Ideation Agent for Chad Backtesting.

Read Jake's persona in AGENTS.md. Read the current feature set in PLAN.md.

Think about:
1. What would make Jake tell his friends about this product?
2. What would make Jake come back every day?
3. What would make Jake willing to pay $100-250/mo?
4. What are competitors (TradingView, Bookmap, Jigsaw, NinjaTrader) doing that we're not?
5. What do none of the competitors do that would be a killer feature?

Generate 5-10 feature ideas. For each:
- Name and one-line description
- Why Jake would care (in his language)
- Effort estimate: small / medium / large
- Priority suggestion: must-have / should-have / nice-to-have

Write to reviews/ideas-[date].md.

IMPORTANT: These are suggestions only. Mason decides what gets built and when.
Do NOT create task specs or write code. Ideas only.
```

---

## Agent Flow Architecture

### The Pipeline (per phase)

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐  │
│  │ Architect │───▶│  Coder   │───▶│    QA    │───▶│ Gap Analyst  │  │
│  │  Agent    │    │  Agent   │    │  Agent   │    │    Agent     │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────┬───────┘  │
│       ▲                                                  │          │
│       │                                                  ▼          │
│       │                                          ┌──────────────┐  │
│       │                                          │ User Persona │  │
│       │                                          │   Agent      │  │
│       │                                          └──────┬───────┘  │
│       │                                                  │          │
│       │                                                  ▼          │
│       │                                          ┌──────────────┐  │
│       │                                          │  Security    │  │
│       │                                          │  Reviewer    │  │
│       │                                          └──────┬───────┘  │
│       │                                                  │          │
│       │          ┌──────────────┐                        │          │
│       └──────────│  Feedback    │◀───────────────────────┘          │
│                  │  Aggregator  │                                    │
│                  └──────────────┘                                    │
│                                                                     │
│                  ALL PASS? ──────▶ NEXT PHASE                       │
│                  FAILURES? ──────▶ LOOP BACK TO ARCHITECT           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Flow Steps

**Step 1: Architect** breaks the phase into task specs → writes to `tasks/`

**Step 2: Coder** (can be multiple in parallel) implements each task → writes status to `status/`

**Step 3: QA** tests each completed task → writes reports to `qa/`

**Step 4: Gap Analyst** compares implementation to plan → writes to `reviews/`

**Step 5: User Persona Agent** walks through as Jake → writes to `reviews/`

**Step 6: Security Reviewer** audits all changes → writes to `reviews/`

**Step 7: Feedback Aggregator** (can be Architect or a dedicated agent):
- Reads all QA reports, gap report, persona review, security audit
- Produces a single `feedback/[phase]-round-[N].md` summarizing:
  - Critical blockers (must fix)
  - Major issues (should fix)
  - Minor issues (can defer)
  - UX improvements (prioritized)
- If there are critical/major issues → loop back to Step 1 (Architect re-scopes fixes)
- If all clear → phase is DONE, proceed to next phase

---

## The Recursive Build Loop

### How It Works

```
WHILE phase is not complete:
    1. Architect reads plan + any prior feedback → produces task specs
    2. Coder(s) implement tasks (parallel where possible)
    3. QA tests implementations
    4. Gap Analyst checks against plan
    5. Persona Agent reviews as Jake
    6. Security Reviewer audits
    7. Feedback Aggregator consolidates

    IF critical/major issues exist:
        → Write issues to feedback/[phase]-round-[N].md
        → Increment round counter N
        → LOOP (Architect picks up feedback and re-scopes)

    IF no critical/major issues:
        → Mark phase complete
        → BREAK
```

### Round Tracking

Each iteration through the loop is a "round". Files are named:
- `feedback/phase-1-round-1.md` — first pass feedback
- `feedback/phase-1-round-2.md` — second pass (fixing round 1 issues)
- etc.

This creates a traceable history of every issue found and how it was resolved.

### Termination Criteria

A phase is complete when:
1. All QA acceptance criteria PASS
2. Gap Analyst finds zero critical deviations from plan
3. Security Reviewer finds zero CRITICAL or HIGH severity issues
4. User Persona Agent identifies no "Jake would leave the site" moments on core flows

Minor and cosmetic issues can be deferred to a polish pass.

---

## Context & Document System

### Directory Structure

```
chadbacktesting/
├── PLAN.md                    # Source of truth — product plan (DO NOT auto-modify)
├── AGENTS.md                  # This file — agent roles, persona, flows
├── README.md                  # Public readme
│
├── tasks/                     # Task specs written by Architect
│   ├── phase-1/
│   │   ├── T1.1-install-nginx.md
│   │   ├── T1.2-ssl-cert.md
│   │   └── ...
│   ├── phase-2/
│   └── ...
│
├── status/                    # Completion status written by Coder
│   ├── T1.1-status.md
│   ├── T1.2-status.md
│   └── ...
│
├── qa/                        # QA test reports
│   ├── T1.1-qa.md
│   ├── T1.2-qa.md
│   └── ...
│
├── reviews/                   # Review documents (Gap, Persona, Security, Ideas)
│   ├── gap-phase-1.md
│   ├── persona-phase-1.md
│   ├── security-phase-1.md
│   ├── ideas-2026-03-15.md
│   └── ...
│
├── feedback/                  # Aggregated feedback per phase per round
│   ├── phase-1-round-1.md
│   ├── phase-1-round-2.md
│   └── ...
│
└── [source code directories]  # Actual SaaS code (landing page, etc.)
```

### Document Conventions

Every document written by an agent follows this header format:
```markdown
# [Document Title]
**Agent**: [Agent Role Name]
**Phase**: [Phase Number]
**Task**: [Task ID or "Phase-level"]
**Round**: [Round number, if in feedback loop]
**Date**: [ISO date]
**Status**: [DRAFT / FINAL]

---
[Content]
```

### How Agents Read Context

Every agent MUST read these files before starting work:
1. **PLAN.md** — the full product plan and system architecture
2. **AGENTS.md** — their own role definition and the user persona
3. **The specific task spec** they're working on (from `tasks/`)
4. **Any prior feedback** for the current phase (from `feedback/`)

Coder agents additionally read:
- The actual source files they need to modify
- The status of any dependency tasks (from `status/`)

QA agents additionally read:
- The task spec's acceptance criteria
- The coder's status report

---

## Phase Execution Playbooks

### How to Execute a Phase

#### For Cloud-Heavy Phases (3, 5, 6, 7)

These phases mostly build SaaS pages/features in the `chadbacktesting/` repo.

1. **Kick off the Architect** (cloud agent or local Composer — either works):
   - "Read TRACKER.md and PLAN.md. Produce task specs for Phase N. Write them to `tasks/phase-N/`."
   - Architect writes self-contained specs, each with `**Execution**: CLOUD` or `**Execution**: LOCAL`
   - Architect identifies parallel groups

2. **Kick off Cloud Coders** (one per task that can run in parallel):
   - Go to `cursor.com/agents` (or Slack, or GitHub, or local Composer)
   - Start a cloud agent for each CLOUD task with the Cloud Agent Prompt Template
   - They each clone the repo, create their branch, implement the spec, push, and create a PR
   - You can kick off many at once — they all run on separate VMs

3. **Kick off any LOCAL tasks** in a local Composer:
   - If the phase has any local tasks (SSH, arkon/ changes), handle them in a local Composer

4. **Reviews** (all cloud agents):
   - Kick off QA, Gap Analyst, Persona, and Security Reviewer as cloud agents
   - They analyze the PRs and code, write reports to `reviews/` and `qa/`

5. **Merge PRs** when reviews pass

6. **Mason checkpoint**: Review feedback, approve/reject/fix

#### For Local-Heavy Phases (1, 2)

These phases involve SSH to the server and `arkon/` file changes.

1. **Open a local Composer** in Cursor desktop
2. **Paste the Orchestrator Prompt** (local mode, see above)
3. Say: "Begin Phase [N]"
4. The orchestrator SSHes into the server, executes commands, edits files
5. It handles the task sequence, running tasks in dependency order
6. At deployment checkpoints, it stops and asks you to confirm
7. When the phase is done, it updates TRACKER.md

#### For Mixed Phases (4)

Some tasks are cloud (HTML/CSS pages), some are local (server routes). Run them in parallel:
- Local Composer: handles OAuth route implementation in `arkon/mbo_streaming_server.py`
- Cloud agents: build auth pages, dashboard HTML, settings pages — all in `chadbacktesting/`
- Integration happens at the end when cloud PRs are merged and local server changes are deployed

### The QA → Fix Loop

```
Coder pushes branch → QA cloud agent reviews PR → FAILS (WebSocket not proxied)
    → QA writes qa/T1.3-qa.md with failure details
    → TRACKER.md: T1.3 = NEEDS_FIX
    → New coder agent picks up the fix: "Read qa/T1.3-qa.md. Fix the issue on branch phase-1/T1.3."
    → Coder fixes, pushes updated branch → QA retests → PASSES
    → TRACKER.md: T1.3 = DONE
```

This inner loop runs automatically. Mason only gets involved at phase-level checkpoints, not individual task fixes.

---

## Parallel Execution Strategy

### Cloud vs Local Parallelism

| Agent Type | Parallelism Limit | Why |
|------------|-------------------|-----|
| **Cloud agents** | Effectively unlimited (Cursor manages infra) | Each runs on its own VM, zero local resource cost |
| **Local Composers** | Max 2-3 concurrent | Share your machine's RAM/CPU |
| **Cloud + Local combined** | Cloud unlimited + Local 2-3 | They don't interfere with each other |

This means for a phase like Phase 3 (landing page), you can kick off 5-6 cloud coders simultaneously — one per page/component — while a local Composer handles Phase 1 server config in parallel.

### Which Agents Can Run in Parallel

| Agent | Parallel With | Runs As | Notes |
|-------|---------------|---------|-------|
| Architect | Nothing | Cloud or Local | Must complete before coders start |
| Cloud Coder A | Cloud Coder B, C, D... | Cloud | No overlapping files. Many can run. |
| Local Coder | Other Local Coder (max 2) | Local | Only for arkon/ or SSH tasks |
| QA (Task A) | QA (Task B, C, D...) | Cloud | Read-only analysis, many can run |
| Gap Analyst | Persona Agent, Security Reviewer | Cloud | All three run simultaneously |
| Feedback Aggregator | Nothing | Cloud or Local | Needs all reviews complete first |

### Phase-Level Parallelism

Because cloud and local agents don't share resources, phases can overlap aggressively:

| Parallel Track A (Local) | Parallel Track B (Cloud) |
|--------------------------|--------------------------|
| Phase 1: SSH into server, install nginx, configure SSL | Phase 3: Build landing page (multiple cloud coders) |
| Phase 2: Modify `arkon/mbo_streaming_server.py` for /pvt route | Phase 3 continued: Pricing page, features section, footer |
| Phase 4 (local parts): Add OAuth routes to server | Phase 4 (cloud parts): Auth pages, dashboard HTML/CSS |

Cloud agents work on branches in the GitHub repo. Local agents work on the server and `arkon/` files. They never conflict.

### Resource Considerations (Local Only)

Cloud agents have zero impact on your machine. For local Composers:
- Each Cursor Composer uses roughly 500MB-1GB RAM for context/model interaction
- 2-3 local agents + Cursor itself + your OS = ~4-6GB RAM usage
- If you have 16GB, 3 local agents is fine. 8GB, stick to 2.
- If your machine starts lagging, drop local agents to 1 and let cloud agents carry the load

### The Key Rule: File Ownership Prevents Conflicts

Regardless of how many agents run, the Architect enforces:
1. No two agents (cloud or local) modify the same file simultaneously
2. Cloud agents only touch `chadbacktesting/` repo files
3. Local agents own `arkon/` files and server configs
4. Each cloud agent works on its own Git branch

---

## Kicking Off the Project

### Quick Start (3 steps)

**Step 1: Run the Architect (cloud agent or local Composer)**

Open a new Composer (local or cloud) and paste:

```
Read chadbacktesting/PLAN.md and chadbacktesting/AGENTS.md.

You are the Architect Agent. The current phase is Phase 1: Server Infrastructure
(nginx, SSL, Domain, Security Hardening). Tasks T1.1 through T1.15.

Break each task into an implementable spec. Write specs to tasks/phase-1/.
For each spec, set **Execution** to CLOUD or LOCAL:
- LOCAL: anything requiring SSH to 95.216.5.147 or modifying arkon/ files
- CLOUD: anything that only modifies chadbacktesting/ repo files (configs, templates, docs)

Identify which tasks can run in parallel and which have dependencies.
Update TRACKER.md with the parallel groups.

Key context:
- Server: 95.216.5.147, SSH as root with ~/.ssh/hetzner_server_key
- Server dir: /opt/mbo_server
- Current state: uvicorn running on 0.0.0.0:8000, no nginx, no SSL, no domain config
- Target: nginx reverse proxy, Let's Encrypt SSL, HTTPS-only, www→non-www redirect, WebSocket proxy
- CRITICAL: Do not break the existing backtesting tool at http://95.216.5.147:8000/
```

**Step 2: Run LOCAL tasks in a local Composer**

After the Architect writes the task specs, open a local Composer and paste the **Orchestrator Prompt (local mode)** from the section above. It will pick up all LOCAL tasks and execute them sequentially via SSH.

**Step 3: Run CLOUD tasks as cloud agents**

For any CLOUD tasks the Architect produced, kick them off at `cursor.com/agents` or from another Composer using the **Cloud Agent Prompt Template** from the section above.

**In parallel**: While the local Composer handles Phase 1 server setup, you can already kick off a cloud Architect agent to produce Phase 3 (landing page) task specs — and start cloud coders on those. The two tracks don't share any files.

After all tasks complete, kick off QA and review agents (cloud), aggregate feedback, and proceed to the next phase. The full flow is documented in the Phase Execution Playbooks above.
