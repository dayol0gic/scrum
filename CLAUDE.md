# Project Context

## Project Overview
A production-grade multi-agent ecosystem for high-frequency trading (HFT), built on Supabase. The platform provides:
- **IAM (Identity & Access Management)**: Role-based access control, fine-grained permissions, API key management, and audit logging for all agent and user actions
- **External Data Integration**: Real-time and historical market data ingestion from Bloomberg (via BLPAPI), with an extensible adapter pattern for additional data sources (Refinitiv, Quandl, etc.)
- **HFT Trade Execution Engine**: Low-latency order management system (OMS) with support for multiple asset classes, execution algorithms, and broker connectivity
- **Reporting & Analytics**: Real-time P&L, risk metrics, execution quality analysis (slippage, fill rates), and regulatory reporting

## Tech Stack
- **Framework**: Next.js 15 (App Router)
- **Language**: TypeScript (strict mode)
- **UI**: React 19, Tailwind CSS
- **Backend**: Supabase (Auth, Database, Realtime, Storage, Edge Functions)
- **Database**: PostgreSQL (via Supabase) with TimescaleDB extension for time-series market data
- **IAM**: Supabase Auth + custom RBAC via PostgreSQL Row Level Security (RLS) policies
- **Market Data**: Bloomberg B-PIPE / BLPAPI (REST adapter), extensible to other data sources
- **Execution**: FIX Protocol 4.4/5.0 for broker connectivity, custom OMS built on Supabase Edge Functions
- **Testing**: Jest, React Testing Library, Playwright (E2E)
- **Linting**: ESLint, Prettier

## Build & Test Commands

```bash
# Install dependencies
npm install

# Linting - must pass with no errors
npm run lint

# Run all tests - all must pass
npm run test

# Build verification - must succeed
npm run build

# Start development server
npm run dev

# Run E2E tests
npm run test:e2e

# Generate Supabase types
npm run supabase:types
```

## Project Structure
```
app/                        # Next.js pages and routes
  (auth)/                   # Auth flow pages (login, MFA, etc.)
  (dashboard)/              # Main application pages
  api/                      # API route handlers
components/                 # Reusable UI components
  iam/                      # IAM-related components (roles, permissions, keys)
  trading/                  # Trade execution UI components
  reporting/                # Charts, tables, report views
  data/                     # Market data display components
lib/
  supabase/                 # Supabase client, server, admin instances
  iam/                      # RBAC logic, permission checks, audit helpers
  bloomberg/                # Bloomberg API adapter and data normalizers
  execution/                # OMS logic, order lifecycle, FIX protocol helpers
  reporting/                # P&L calculations, risk metrics, report generators
  adapters/                 # Extensible interface for external data sources
types/                      # TypeScript type definitions
  iam.ts                    # Roles, permissions, API key types
  market-data.ts            # Tick, OHLCV, order book types
  execution.ts              # Order, fill, position, trade types
  reporting.ts              # Report, metric, P&L types
__tests__/                  # Unit and integration tests
  iam/
  execution/
  reporting/
  bloomberg/
e2e/                        # Playwright E2E tests
supabase/
  migrations/               # Database migrations
  functions/                # Edge Functions (execution engine, data ingestion)
  seed/                     # Seed data for development
public/                     # Static assets
```

## Domain Guidelines

### IAM (Identity & Access Management)
- All access control is enforced via **Supabase Row Level Security (RLS)** policies — never bypass RLS
- Roles: `super_admin`, `admin`, `trader`, `risk_manager`, `analyst`, `readonly`
- Every user action and agent action must be written to the `audit_log` table
- API keys must be hashed (SHA-256) before storage — never store plaintext keys
- MFA must be enforced for `trader`, `risk_manager`, and `admin` roles
- Session tokens expire after 8 hours; refresh tokens after 30 days
- All permission checks use the `lib/iam/permissions.ts` helper — never inline permission logic

### External Data Sources (Bloomberg & others)
- All data source integrations implement the `DataSourceAdapter` interface in `lib/adapters/`
- Bloomberg data is ingested via a Supabase Edge Function that proxies BLPAPI requests
- Raw market data is stored in TimescaleDB hypertables for efficient time-series queries
- Data normalization (Bloomberg field names → internal schema) happens in `lib/bloomberg/normalizers.ts`
- Always handle Bloomberg API rate limits and reconnection gracefully
- Cache frequently accessed reference data (securities, exchanges) in Supabase with TTL

### HFT Trade Execution Engine
- **Latency is critical**: execution logic lives in Supabase Edge Functions (not Next.js API routes) for minimal cold start
- Order lifecycle states: `pending` → `submitted` → `partially_filled` → `filled` | `cancelled` | `rejected`
- All orders must be validated against position limits and risk checks before submission
- FIX message parsing/generation lives exclusively in `lib/execution/fix/`
- Every order state change must be logged to `order_audit_log` with microsecond timestamps
- Never modify a filled or cancelled order — create a new correcting entry instead
- Position calculations must be atomic — use PostgreSQL transactions

### Reporting
- P&L calculations run server-side in PostgreSQL functions — never in the browser
- All reports are generated asynchronously via Supabase Edge Functions and stored in Supabase Storage
- Real-time metrics (live P&L, open positions) use Supabase Realtime subscriptions
- Report access is subject to IAM role checks — analysts and above can view, only admins can export
- Slippage = (fill price - arrival price) / arrival price × 10000 (in basis points)

## Development Guidelines

### Coding Standards
- **No `any` types** — use proper TypeScript generics and type narrowing
- All database queries go through typed Supabase client — no raw SQL in application code (use RPC functions or migrations for complex queries)
- Error handling must be explicit — never swallow errors silently
- All financial calculations use **integer arithmetic in minor units** (e.g. cents, basis points) — never floating point for money
- Log all errors with structured logging (include `user_id`, `action`, `resource`, `timestamp`)

### Security Rules
- Never log sensitive data (API keys, passwords, PII, order details) to console
- All external API calls (Bloomberg, brokers) go through server-side Edge Functions — never expose credentials to the browser
- Validate and sanitize all inputs, especially order quantities, prices, and symbol names
- Rate-limit all public-facing API routes

### Git Workflow

**Branch Strategy**:
- `main` - Production (stable, deployed)
- `develop` - Development integration branch
- `feature/<feature-name>` - Feature branches (from `develop`)
- `fix/<bug-name>` - Bug fix branches (from `develop`)

**Merge Strategy**:
- All PRs target `develop`
- Only humans merge `develop` → `main`

**Commit Messages** (conventional commits):
- `feat:` new features
- `fix:` bug fixes
- `refactor:` refactoring
- `test:` test changes
- `docs:` documentation
- `chore:` build/tooling
- `perf:` performance improvements

### Testing Requirements
- Every new feature MUST have unit tests
- Execution engine logic MUST have integration tests against a Supabase test instance
- IAM permission logic MUST be tested for both allowed and denied cases
- Financial calculations MUST be tested with known inputs and expected outputs
- All tests must pass before PR approval

## Multi-Agent Development Process

- **Scrum Master**: Manages Kanban board, creates tickets, assigns work, tracks status
- **Planner**: Analyzes feature requests, creates detailed implementation plans
- **Fullstack Developer**: Implements features end-to-end
- **QA Tester**: Reviews PRs, runs lint/tests/build, reports bugs or approves

### Kanban Workflow
Backlog → Planning → Developing → Testing → Human Review → Done

## Notes for AI Agents

### Critical Rules
- **ALWAYS** branch from `develop`, never from `main`
- **ALWAYS** target `develop` in PRs
- **NEVER** disable or bypass RLS policies
- **NEVER** store sensitive credentials in code — use Supabase secrets or environment variables
- **ALWAYS** run `npm run lint && npm run test && npm run build` before marking work complete
- **ALWAYS** check `supabase/migrations/` before modifying database schema — add a new migration, never edit existing ones
- Financial logic changes require a human review regardless of test status

### Example Git Commands
```bash
git checkout develop
git pull origin develop
git checkout -b feature/my-feature

gh pr create --base develop --title "feat: description" --body "..."
```

### General Guidelines
- Read this file in full before starting any task
- Follow existing patterns in the codebase
- Prioritize correctness, security, and auditability over speed
- When in doubt about a trading or financial calculation, ask for clarification before implementing
