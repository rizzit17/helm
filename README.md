# Helm : AI Finance Copilot

Personal finance app where an LLM agent is the primary interface. Log spending conversationally, receive proactive insights, scan receipts, import bank statements, and simulate financial scenarios in real time.

> **Positioning note:** This space is competitive — Origin's AI Advisor, Rocket Money, and SMS-based Indian apps like axio/Walnut have existed for years. This is a deliberate implementation with specific architecture choices: LangGraph tool isolation (LLM never writes raw SQL), closed-loop agent actions requiring explicit user confirmation before executing, and zero hardcoded demo data anywhere in the UI.

## Tech Stack

| Layer | Choice |
|---|---|
| Mobile | React Native (Expo SDK 51), TypeScript, expo-router, @shopify/react-native-skia |
| Backend | FastAPI, PostgreSQL (asyncpg), SQLAlchemy 2.0 async, Alembic, LangGraph, APScheduler, Redis |
| Agent LLM | Groq (`openai/gpt-oss-120b`) — conversational agent, supervisor routing, PDF statement extraction |
| Receipt / OCR | Google Gemini 2.5 Flash — photo receipt → structured line items |
| Auth | JWT access tokens + refresh token rotation with reuse detection; biometric gate (expo-local-authentication) |

## Architecture

```
Mobile (RN/Expo) ──HTTPS──> FastAPI ──> PostgreSQL
                               │
                               ├──> Redis (rate-limit + session)
                               ├──> LangGraph multi-agent (supervisor → query agent | planning agent)
                               │     tools call service layer — no raw SQL from LLM
                               └──> APScheduler ──> Expo Push API ──> Mobile
```

### Agent Routing

The supervisor classifies every message with a fast heuristic first; the LLM supervisor only runs when the heuristic returns `"query"`:

| Route | Agents invoked | When |
|---|---|---|
| `query` | query agent | transactions, spending totals, budget status, cashflow |
| `planning` | planning agent | reallocation, savings goals, what-if scenarios |
| `both` | query agent → planning agent | compound: "what is my X AND what would happen if Y?" |

---

## Setup

### Prerequisites

- Python 3.11+
- Node.js 18+ / npm
- Docker (for Postgres + Redis)
- Android emulator or physical Android device

### Backend

```bash
cd backend

# 1. Virtualenv
python -m venv .venv && source .venv/bin/activate

# 2. Dependencies
pip install -r requirements.txt

# 3. Environment — copy and fill in required keys
cp .env.example .env
# Required: DATABASE_URL, SECRET_KEY, GROQ_API_KEY
# Required for receipt scanning: GOOGLE_API_KEY

# 4. Postgres + Redis (Docker)
docker run -d --name helm-pg \
  -e POSTGRES_DB=helm -e POSTGRES_USER=helm -e POSTGRES_PASSWORD=helm \
  -p 5432:5432 postgres:16
docker run -d --name helm-redis -p 6379:6379 redis:7

# 5. Migrations
alembic upgrade head

# 6. Start
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

Interactive API docs: `http://localhost:8000/docs`

### Mobile (native dev client)

The app uses `expo-local-authentication` and `@shopify/react-native-skia` — **Expo Go will not work.** A native dev client build is required.

```bash
cd mobile
npm install

# Create mobile/.env pointing at your backend
cp .env.example .env
# For the Android emulator: EXPO_PUBLIC_API_URL=http://10.0.2.2:8000/api/v1
# For the deployed backend: EXPO_PUBLIC_API_URL=https://helm-euvm.onrender.com/api/v1

# First run — build and install the native dev client (~10 min)
npx expo run:android

# Subsequent runs — JS bundler only
npx expo start --dev-client
```

If Metro loses connection after the emulator sleeps:
```bash
adb reverse tcp:8081 tcp:8081
```

**Physical Android device:**
```bash
# Target a specific device when emulator + phone are both connected
ANDROID_SERIAL=<device-serial> npx expo run:android

# Wireless development without USB — creates a public ngrok tunnel
npx expo start --tunnel
# Then shake the device → Dev Settings → change bundle location to the tunnel URL
```

`EXPO_PUBLIC_API_URL` is inlined at JS bundle time by Metro. Changing the backend URL requires a full Metro rebuild, not just a Gradle rebuild.

### Deploying to production

The production stack is Render (compute) + Supabase (Postgres). Docker is not used in production.

**Supabase:** Create a new project. Use the **Session mode pooler** URL from the Supabase dashboard (`aws-REGION.pooler.supabase.com:5432`) — the direct connection hostname is IPv6-only without a paid add-on and won't work on most cloud hosts.

**Render:** Deploy the `backend/` directory as a Python web service. Set all `.env.example` keys as environment variables in the Render dashboard. The start command is:
```
uvicorn main:app --host 0.0.0.0 --port $PORT
```

After deploying, rebuild the mobile dev client with `EXPO_PUBLIC_API_URL` pointing at your Render service URL.

---

## Design System

Forest-dark palette + Playfair Display / Geist typography. All tokens live in `mobile/lib/design.ts`.

```typescript
// Colors
T.bg         = "#0e1511"   // near-black foundation
T.card       = "#161d19"   // surface-container-low (cards, list rows)
T.surface    = "#1a211d"   // surface-container
T.panel      = "#242c27"   // surface-container-high
T.border     = "#2f3632"   // outline-variant (dividers)
T.borderHi   = "#3c4a42"   // outline (focus rings)
T.emerald    = "#5af0b3"   // growth, credits, positive states
T.coral      = "#ffb4ab"   // debits, alerts, over-budget
T.gold       = "#dcc66e"   // goals, premium insights, near-budget warning

// Font families
F.serif      = "PlayfairDisplay_400Regular"   // headings, large amounts
F.sans       = "Geist_300Light"               // body copy
F.sansMedium = "Geist_500Medium"              // caps labels (letterSpacing 1.65)
F.mono       = "Geist_400Regular"             // transaction data, numbers

// Utilities
fmtINR(n)    // ₹1,00,000 format — no toLocaleString (unreliable on Hermes)
```

**Critical Android rule:** Never combine a NativeWind custom-font class (`font-serif`, `font-mono`, etc.) with an inline `style` prop on the same `<Text>`. NativeWind 4.x + Hermes crashes silently — the screen goes blank with no error. Use pure inline styles via `T` and `F` constants exclusively.

---

## Feature Checklist

### Phase 1 — Foundation
- [x] JWT auth + refresh token rotation with server-side reuse detection
- [x] Biometric gate on app open (expo-local-authentication)
- [x] Transactions CRUD with category tagging and merchant field
- [x] Budgets CRUD + real-time pace projection (spent / limit × 100)
- [x] Budget bar colors: emerald (≤80%), gold (80–100%), coral (over budget)
- [x] LangGraph query agent — tools: `get_transactions`, `get_budget_status`, `get_spending_summary`, `forecast_cashflow`, `get_subscriptions`
- [x] Multi-turn conversation memory (persisted per user in PostgreSQL)
- [x] Daily insight job (APScheduler) with materiality filter — suppresses trivial alerts (>15% pace deviation AND >20% projected overspend)
- [x] Expo push notifications
- [x] Insights feed with unread indicators + mark-read
- [x] Rate-limited auth endpoints (Redis)
- [x] Audit log for every agent-initiated action
- [x] Cash-flow Skia chart on Overview (22-day rolling window)

### Phase 2 — Agent capabilities
- [x] Multi-agent supervisor routing (heuristic + LLM fallback, three routes: query / planning / both)
- [x] Closed-loop budget rebalancing — `propose_reallocation` → ProposalCard (CONFIRM / DISMISS) → `execute_reallocation` → 30s undo window
- [x] Goal-setting agent — `propose_savings_goal` → GoalProposalCard (SET GOAL / DISMISS) → creates goal on confirm
- [x] Subscription / recurring pattern detection (`recurring_pattern` model + `subscription_service`)

### Phase 3 — Multimodal
- [x] Receipt scanning — photo → Gemini 2.5 Flash → structured draft → review → bulk import
- [x] Receipt scan 25-second client-side timeout with user-friendly error message
- [x] Bank statement PDF import — PDF → `openai/gpt-oss-120b` extraction → review screen with per-row edit/include + duplicate detection → bulk import

---

## Architecture Decisions

See `docs/adr/`:
- `001` — Agent tool isolation: LLM never writes raw SQL; all tools call the service layer
- `002` — Refresh token rotation with server-side reuse detection
- `003` — Insight materiality filter: prevents notification fatigue from trivial alerts
- `004` — Multi-agent supervisor routing: heuristic-first, LLM fallback, three routes

---

## Hard Rules

1. No stub or placeholder features — everything visible in the UI is functional
2. Agent actions that modify data always require explicit user confirmation before executing
3. Zero hardcoded demo data anywhere in the UI
4. One auth flow, one route per screen — no dead routes
