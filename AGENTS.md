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

### The Reality of Parallel Agents in Cursor

Cursor doesn't have a built-in "orchestrator" that automatically spins up agent pipelines. Here's what actually happens:

**Each "agent" = one Cursor Composer chat.** You open a new Composer tab, paste the agent's prompt (with its role and the files it should read), and let it run. Multiple Composer tabs can run simultaneously — that's your parallelism.

### Step-by-Step: Running a Phase

#### 1. You kick off the Architect (1 Composer tab)

Open a new Composer. Paste the Architect prompt for the current phase. It reads PLAN.md and AGENTS.md, then writes task specs to `tasks/phase-N/`. Wait for it to finish.

#### 2. You kick off Coders in parallel (multiple Composer tabs)

The Architect's output tells you which tasks can run in parallel (Group A, Group B, etc.). For Group A tasks:

- Open **Composer Tab 1**: "You are the Coder Agent. Read and implement `tasks/phase-1/T1.1-install-nginx.md`"
- Open **Composer Tab 2**: "You are the Coder Agent. Read and implement `tasks/phase-1/T1.6-env-file.md`"
- Open **Composer Tab 3**: "You are the Coder Agent. Read and implement `tasks/phase-1/T1.7-gitignore.md`"

All three run at the same time, writing to different files. No conflicts because the Architect specifically assigned non-overlapping files to each.

Once Group A finishes, kick off Group B the same way.

#### 3. You kick off QA (1-2 Composer tabs)

After coders finish, open a Composer for the QA Agent. It tests what was built. If tasks are independent, you can run multiple QA agents in parallel.

#### 4. You kick off reviewers in parallel (3 Composer tabs)

- **Tab 1**: Gap Analyst Agent
- **Tab 2**: User Persona Agent
- **Tab 3**: Security Reviewer Agent

All three read the code and QA reports, write to separate files in `reviews/`. No conflicts.

#### 5. You read the feedback and decide

Open the feedback file. If there are critical issues, you paste the feedback into a new Architect Composer to scope fixes. If it's clean, you move to the next phase.

### How File Conflicts Are Prevented

The entire system is designed so parallel agents never touch the same file:

| Agent | Writes To | Never Touches |
|-------|-----------|---------------|
| Architect | `tasks/phase-N/*.md` | Source code, status, qa, reviews |
| Coder A | Specific source files + `status/T-A.md` | Files assigned to Coder B |
| Coder B | Different source files + `status/T-B.md` | Files assigned to Coder A |
| QA | `qa/*.md` | Source code, tasks |
| Gap Analyst | `reviews/gap-*.md` | Source code, other review files |
| Persona Agent | `reviews/persona-*.md` | Source code, other review files |
| Security Reviewer | `reviews/security-*.md` | Source code, other review files |

**If two agents need to modify the same file** (e.g., two tasks both touch `mbo_streaming_server.py`), they CANNOT run in parallel. The Architect must put them in sequential groups.

### What You (Mason) Actually Do

Your role in the loop:

1. **Start each step** by opening Composer tabs with the right prompts
2. **Glance at outputs** as they complete — you don't need to read every line, but skim the QA and review summaries
3. **Make go/no-go decisions** — "phase looks good, move on" or "fix these issues first"
4. **Approve or reject ideas** — Product Ideation Agent writes suggestions, you decide (see approval workflow below)
5. **Provide blocked info** — when a task says "BLOCKED until Mason provides X" (like Stripe keys), you unblock it

You are NOT doing the coding, testing, or reviewing. You're the project manager reading dashboards and saying "go" or "fix."

### Quick-Start Prompt: Running Any Agent

Copy-paste this into a new Composer, replacing the bracketed values:

```
Read chadbacktesting/AGENTS.md and chadbacktesting/PLAN.md.

You are the [AGENT ROLE] for the Chad Backtesting project.
Current phase: Phase [N].
[If applicable: Current task: tasks/phase-N/[task-id].md]
[If applicable: Prior feedback to address: feedback/phase-N-round-[M].md]

Follow your role instructions exactly as defined in AGENTS.md.
Write your output to the correct directory as specified in your role.
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

**Role**: Implementation. Takes a task spec from Architect and writes the code. Has access to all files in both `arkon/` and `chadbacktesting/` repos.

**When to Use**: After Architect has produced a task spec.

**Input**: A specific `tasks/<phase>/<task-id>.md` spec + PLAN.md for broader context.

**Output**: Code changes committed to the appropriate repo. Updates the task status in `status/<task-id>.md`.

**Prompt Template**:
```
You are the Coder Agent for the Chad Backtesting project.

Read the task spec at tasks/[phase]/[task-id].md.
Read PLAN.md for system architecture context.
Read AGENTS.md section "Target User Persona" to understand who you're building for.

Your job:
1. Implement exactly what the spec describes
2. Do NOT modify files not listed in the spec unless absolutely necessary
3. Do NOT touch the database on the Hetzner server or overwrite strategies.db
4. All Arkon chart code stays in arkon/ — never copy to chadbacktesting/
5. Write clean code with no unnecessary comments
6. After implementation, write a brief status update to status/[task-id].md describing what you did

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

This is the step-by-step playbook for running agents through any phase.

#### Step 0: Pre-flight Check
```
Read PLAN.md → confirm the phase's task list
Read feedback/ → check if there's unresolved feedback from a prior round
Verify prerequisites: previous phase is marked complete
```

#### Step 1: Spin Up Architect Agent
```
Input: PLAN.md (phase task list) + any feedback from prior rounds
Output: Task specs in tasks/[phase]/
Parallelizable: NO (must complete before Coders start)
```

#### Step 2: Spin Up Coder Agent(s) — PARALLEL
```
For each task spec where dependencies are met:
    Spin up a Coder Agent with that task spec
    Coder implements and writes to status/

Parallel groups (example for Phase 1):
    Group A (no deps): T1.1, T1.6, T1.7 (can run simultaneously)
    Group B (needs Group A): T1.2, T1.3 (need nginx installed first)
    Group C (needs Group B): T1.4, T1.5, T1.8-T1.15 (need SSL + nginx configured)
```

#### Step 3: Spin Up QA Agent — After Each Coder Completes
```
For each completed task:
    QA Agent tests against acceptance criteria
    Writes report to qa/

Can run in parallel for independent tasks.
```

#### Step 4: Spin Up Review Agents — PARALLEL (after all QA completes)
```
All three run simultaneously:
    Gap Analyst Agent → reviews/gap-[phase].md
    User Persona Agent → reviews/persona-[phase].md
    Security Reviewer Agent → reviews/security-[phase].md
```

#### Step 5: Feedback Aggregation
```
Read all QA reports + all review documents
Write consolidated feedback to feedback/[phase]-round-[N].md
Decision: PASS (proceed) or LOOP (fix issues)
```

#### Step 6: Loop or Proceed
```
IF LOOP:
    Architect reads feedback → produces fix task specs → Coder fixes → QA retests
    (Only the failed/flagged items, not the whole phase)

IF PASS:
    Mark phase complete
    Begin next phase
```

---

## Parallel Execution Strategy

### Which Agents Can Run in Parallel

| Agent | Parallel With | Notes |
|-------|---------------|-------|
| Architect | Nothing | Must complete before coders |
| Coder A | Coder B, C, ... | As long as tasks have no dependencies on each other |
| QA (Task A) | QA (Task B) | Independent tasks can be tested in parallel |
| Gap Analyst | Persona Agent, Security Reviewer | All three review agents run in parallel |
| Persona Agent | Gap Analyst, Security Reviewer | All three review agents run in parallel |
| Security Reviewer | Gap Analyst, Persona Agent | All three review agents run in parallel |
| Feedback Aggregator | Nothing | Needs all reviews complete first |

### Phase-Level Parallelism

Some phases can overlap:
- **Phase 1 (nginx/SSL) and Phase 2 (private route)**: Phase 2 code changes can be written locally while Phase 1 sets up the server. Deploy Phase 2 after Phase 1 is verified.
- **Phase 3 (landing page) and Phase 4 (auth)**: Landing page design/build can happen while auth backend is being implemented. Integrate the "Sign in with Google" button when both are ready.
- **Phase 5 (Stripe) and Phase 6 (multi-tenant)**: Stripe integration and data isolation can be built in parallel since they touch different parts of the codebase.

### Max Parallelism Per Step

- **Step 2 (Coding)**: Up to 4 Coder agents simultaneously (limited by task independence)
- **Step 3 (QA)**: Up to 4 QA agents simultaneously
- **Step 4 (Reviews)**: Always 3 agents (Gap, Persona, Security) simultaneously
- **Total**: Never more than 4 agents running at once in any step

---

## Kicking Off the First Phase

To begin Phase 1 (Server Infrastructure), spin up the Architect Agent with:

```
You are the Architect Agent. Read PLAN.md and AGENTS.md.

The current phase is Phase 1: Server Infrastructure (nginx, SSL, Domain, Security Hardening).
Tasks T1.1 through T1.15.

Break each task into an implementable spec. Write specs to tasks/phase-1/.
Identify which tasks can run in parallel and which have dependencies.
Mark the parallel groups clearly.

Key context:
- Server: 95.216.5.147, SSH as root with ~/.ssh/hetzner_server_key
- Server dir: /opt/mbo_server
- Current state: uvicorn running on 0.0.0.0:8000, no nginx, no SSL, no domain config
- Target state: nginx reverse proxy, Let's Encrypt SSL, HTTPS-only, www→non-www redirect, WebSocket proxy
- CRITICAL: Do not break the existing backtesting tool at http://95.216.5.147:8000/
```

After Architect completes, spin up Coder agents for the first parallel group, and the loop begins.
