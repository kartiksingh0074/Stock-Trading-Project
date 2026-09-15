# Stock Trading Platform

A full-stack paper-trading web app built with Next.js 15. Trade real stocks with virtual cash, track your portfolio, and compete on a leaderboard — all backed by live market data.

## Features

- **Buy/sell trading** — Serializable-isolation database transactions guarantee correct cash and holding balances even under concurrent trades (no double-spends, no lost updates). See `scripts/test-concurrent-trading.ts` for a load test that verifies this.
- **Portfolio tracking** — average-cost-basis holdings, transaction history, and per-trade portfolio snapshots that power a net-worth history chart.
- **Live market data** — real-time quotes and TradingView charting widgets via the Finnhub API, cached and rate-limited through Redis so bursts of traffic can't exceed Finnhub's API limits.
- **Watchlist** — track symbols you're interested in without holding a position.
- **Price alerts** — set upper/lower price thresholds per symbol and get notified when they're crossed.
- **Leaderboard** — global ranking by net worth.
- **Paid upgrades** — Stripe Checkout flow that credits virtual trading cash and upgrades your account tier, processed through signature-verified, idempotent webhooks.
- **Authentication** — session-based auth via Better Auth, with route protection enforced in middleware.

## Tech Stack

- **Framework:** Next.js 15 (App Router), React 19, TypeScript
- **Database:** MySQL with Prisma ORM
- **Auth:** Better Auth
- **Payments:** Stripe
- **Caching / Rate Limiting:** Upstash Redis
- **Market Data:** Finnhub API, TradingView widgets
- **Styling:** Tailwind CSS

## Architecture — Data Flow Diagram

**Context Diagram (Level 0)**

```mermaid
flowchart TD
    Trader([Trader / User])
    Finnhub[[Finnhub API]]
    Stripe[[Stripe]]

    System(("Stock Trading<br/>Platform"))

    Trader -- "sign up / trade / set alerts" --> System
    System -- "portfolio, quotes, charts" --> Trader

    System -- "quote requests" --> Finnhub
    Finnhub -- "price data" --> System

    System -- "checkout session" --> Stripe
    Stripe -- "webhook: payment completed" --> System
```

**Level 1 — Major Processes**

```mermaid
flowchart TD
    Trader([Trader])
    Finnhub[[Finnhub API]]
    Stripe[[Stripe]]

    P1("1.0 Authenticate User")
    P2("2.0 Manage Watchlist")
    P3("3.0 Execute Buy/Sell Trade")
    P4("4.0 Track Portfolio & Net Worth")
    P5("5.0 Manage Price Alerts")
    P6("6.0 Process Upgrade Payment")
    P7("7.0 Fetch & Cache Market Data")
    P8("8.0 Compute Leaderboard")

    D1[("User / Session / Account")]
    D2[("WatchlistItem")]
    D3[("Transaction")]
    D4[("PortfolioHolding")]
    D5[("PortfolioSnapshot")]
    D6[("PriceAlert")]
    D7[("Payment")]
    D8[("Redis Cache<br/>(quotes + rate limit)")]

    Trader -- credentials --> P1
    P1 -- session cookie --> Trader
    P1 <--> D1

    Trader -- add/remove symbol --> P2
    P2 <--> D2
    P2 -- symbol --> P7

    Trader -- buy/sell order --> P3
    P3 -- symbol --> P7
    P7 -- authoritative price --> P3
    P3 --> D3
    P3 --> D4
    P3 --> D5
    P3 -- confirmation --> Trader

    D4 -- holdings --> P4
    D5 -- history --> P4
    P4 -- net worth chart --> Trader

    D1 <--> P8
    P8 -- ranking --> Trader

    Trader -- create/update alert --> P5
    P5 <--> D6
    P5 -- symbol --> P7
    P7 -- current price --> P5

    Trader -- select plan --> P6
    P6 <--> D7
    P6 -- checkout session --> Stripe
    Stripe -- webhook event --> P6
    P6 -- credit cash / upgrade tier --> D1

    P7 <--> D8
    P7 <--> Finnhub
```

## Database Schema — ER Diagram

```mermaid
erDiagram
    USER ||--o{ SESSION : has
    USER ||--o{ ACCOUNT : has
    USER ||--o{ WATCHLIST_ITEM : watches
    USER ||--o{ TRANSACTION : places
    USER ||--o{ PAYMENT : makes
    USER ||--o{ PORTFOLIO_HOLDING : holds
    USER ||--o{ PORTFOLIO_SNAPSHOT : snapshots
    USER ||--o{ PRICE_ALERT : sets

    USER {
        string id PK
        string name
        string email UK
        boolean emailVerified
        string country
        decimal cashBalance
        string tier
        decimal lastNetWorth
        datetime createdAt
    }

    SESSION {
        string id PK
        string userId FK
        string token UK
        datetime expiresAt
        string ipAddress
        string userAgent
    }

    ACCOUNT {
        string id PK
        string userId FK
        string accountId
        string providerId
        string accessToken
        string refreshToken
        string password
    }

    WATCHLIST_ITEM {
        string id PK
        string userId FK
        string symbol
        string company
        datetime addedAt
    }

    TRANSACTION {
        string id PK
        string userId FK
        string symbol
        string company
        string type
        int quantity
        decimal price
        decimal totalAmount
        datetime executedAt
    }

    PAYMENT {
        string id PK
        string userId FK
        string stripeSessionId UK
        string stripePaymentIntentId
        int amountCents
        string currency
        string planId
        decimal creditsGranted
        string status
        datetime createdAt
    }

    PORTFOLIO_HOLDING {
        string id PK
        string userId FK
        string symbol
        string company
        int quantity
        decimal averageBuyPrice
        decimal totalCost
        datetime updatedAt
    }

    PORTFOLIO_SNAPSHOT {
        string id PK
        string userId FK
        decimal cashBalance
        decimal investedValue
        decimal netWorth
        datetime capturedAt
    }

    PRICE_ALERT {
        string id PK
        string userId FK
        string symbol
        string company
        string alertName
        string alertType
        decimal threshold
        string status
        datetime triggeredAt
    }
```

Notes:
- `WATCHLIST_ITEM` and `PORTFOLIO_HOLDING` each enforce a unique `(userId, symbol)` pair — one row per symbol per user.
- `PAYMENT.stripeSessionId` is unique, letting the Stripe webhook handler safely ignore duplicate delivery of the same event.
- `PORTFOLIO_SNAPSHOT` rows are append-only, written inside the same transaction as a buy/sell, and drive both the net-worth chart and (via `USER.lastNetWorth`) the leaderboard.

## Getting Started

### Prerequisites

- Node.js 20+
- A MySQL database
- API keys for [Finnhub](https://finnhub.io/), [Upstash Redis](https://upstash.com), and [Stripe](https://dashboard.stripe.com/apikeys) (Stripe is only required for the upgrade/payment flow)

### Setup

1. Install dependencies:
   ```bash
   npm install
   ```

2. Copy the environment template and fill in your values:
   ```bash
   cp .env.example .env
   ```

3. Run database migrations:
   ```bash
   npm run db:migrate
   ```

4. Start the dev server:
   ```bash
   npm run dev
   ```

The app will be available at `http://localhost:3000`.

### Scripts

| Command | Description |
| --- | --- |
| `npm run dev` | Start the dev server (Turbopack) |
| `npm run build` | Production build |
| `npm run start` | Start the production server |
| `npm run lint` | Lint the codebase |
| `npm run db:migrate` | Apply Prisma migrations |
| `npm run db:studio` | Open Prisma Studio |
| `npm run db:push` | Push schema changes without a migration |
| `npm run db:test` | Verify the MySQL connection |
| `npm run db:test:concurrency` | Run the concurrent buy/sell regression test |

## Project Structure

```
app/(root)/       Authenticated pages: portfolio, watchlist, markets, orders, wallet, leaderboard, upgrade
app/api/          Route handlers (e.g. Stripe webhooks)
components/       UI components
hooks/            Client-side hooks (e.g. live polling)
lib/actions/      Server actions (trading, portfolio, alerts, leaderboard, payments, watchlist)
lib/auth/         Session helpers
lib/better-auth/  Better Auth configuration
lib/redis/        Redis client + rate limiter
lib/stripe/       Stripe client + plan definitions
prisma/           Schema and migrations
scripts/          Standalone DB and regression-test scripts
```
