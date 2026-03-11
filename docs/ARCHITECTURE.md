# Architecture

Technical design and architecture decisions for Tap2 Intel.

---

## System Overview

Tap2 Intel is a cloud-native analytics platform built on Cloudflare Workers for real-time payment analytics. The system ingests transaction data from payment processors, processes it through ClickHouse for aggregation, and serves insights via GraphQL and REST APIs.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Payment       │     │   Tap2 Intel    │     │   Dashboard     │
│  Processors     │────▶│   Ingestion     │────▶│      UI         │
│  (Stripe, etc)  │     │   (Workers)     │     │   (React)       │
└─────────────────┘     └────────┬────────┘     └─────────────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │   ClickHouse    │
                        │   Analytics DB  │
                        └─────────────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │   GraphQL API   │◀────┐
                        │   (Workers)     │     │
                        └─────────────────┘     │
                                 │              │
                                 ▼              │
                        ┌─────────────────┐     │
                        │  Scheduled      │     │
                        │  Reports        │─────┘
                        │  (Cron Triggers)│
                        └─────────────────┘
```

---

## Technology Stack

### Core Infrastructure

| Component | Technology | Rationale |
|-----------|------------|-----------|
| Compute | Cloudflare Workers | Edge computing, auto-scaling, pay-per-request |
| Database | ClickHouse | Columnar storage, fast aggregations, SQL support |
| API Layer | Cloudflare Workers + GraphQL | Type-safe APIs, efficient queries |
| Frontend | React + Tremor | Component library for data visualization |
| Auth | Tap2 OAuth | Single sign-on with existing Tap2 accounts |
| Deployment | Wrangler + GitHub Actions | Zero-downtime deployments |

### Data Layer

| Technology | Use Case |
|------------|----------|
| ClickHouse Cloud | Primary analytics data warehouse |
| Cloudflare KV | Metadata cache, rate limiting |
| Cloudflare R2 | Report exports, file storage |

### Monitoring & Observability

| Tool | Purpose |
|------|---------|
| Cloudflare Analytics | Request metrics, error rates |
| Sentry | Error tracking and alerting |
| Custom Metrics | Business metrics (query count, dashboard load time) |

---

## Component Architecture

### 1. Ingestion Layer

**Responsibility:** Receive and normalize transaction data from payment processors.

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Stripe     │    │    Tap2      │    │  Processor N │
│  Webhooks    │    │   Events     │    │   Adapter    │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           ▼
                  ┌────────────────┐
                  │  Webhook       │
                  │  Handler       │
                  │  (Worker)      │
                  └────────┬───────┘
                           ▼
                  ┌────────────────┐
                  │  Normalizer    │
                  │  (Unified      │
                  │   Schema)      │
                  └────────┬───────┘
                           ▼
                  ┌────────────────┐
                  │  ClickHouse    │
                  │  Batch Writer  │
                  └────────────────┘
```

**Key Design Decisions:**
- **Async Processing:** Webhooks return 200 immediately, process in background
- **Idempotency:** Deduplication by transaction ID to handle retries
- **Batch Writes:** Accumulate records before writing to ClickHouse

### 2. Query Layer

**Responsibility:** Execute aggregation queries against ClickHouse.

```
┌────────────────┐
│  GraphQL       │
│  Request       │
└────────┬───────┘
         ▼
┌────────────────┐
│  Query         │
│  Planner       │
│  - Cache check │
│  - Authz       │
└────────┬───────┘
         ▼
┌────────────────┐
│  ClickHouse    │
│  Query         │
└────────┬───────┘
         ▼
┌────────────────┐
│  Result        │
│  Formatter     │
└────────────────┘
```

**Schema Overview:**

```sql
-- Transactions fact table
CREATE TABLE transactions (
    timestamp DateTime,
    merchant_id UInt64,
    processor String,
    transaction_id String,
    amount Decimal(18,2),
    currency String,
    status Enum8('approved'=1, 'declined'=2, 'failed'=3),
    payment_method String,
    card_brand LowCardinality(String),
    customer_id Nullable(UInt64),
    metadata String
) ENGINE = MergeTree()
ORDER BY (merchant_id, timestamp);

-- Customers dimension table
CREATE TABLE customers (
    customer_id UInt64,
    merchant_id UInt64,
    first_seen DateTime,
    attributes String
) ENGINE = ReplacingMergeTree()
ORDER BY customer_id;

-- Daily aggregates (materialized view)
CREATE MATERIALIZED VIEW transactions_daily_mv
ENGINE = SummingMergeTree()
ORDER BY (merchant_id, date)
AS SELECT
    merchant_id,
    toDate(timestamp) AS date,
    processor,
    status,
    count() AS transaction_count,
    sum(amount) AS total_amount,
    sumIf(amount, status='approved') AS gmv
FROM transactions
GROUP BY merchant_id, date, processor, status;
```

### 3. Dashboard UI

**Responsibility:** Visualize metrics and enable exploration.

```
┌─────────────────────────────────────────────────────────┐
│                      App Shell                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │  Navigation │  │  Time Range │  │  Merchant Filter│  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────┐  ┌─────────────────┐              │
│  │  GMV Chart      │  │  Approval Rate  │              │
│  │  (Tremor)       │  │  (Tremor)       │              │
│  └─────────────────┘  └─────────────────┘              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Transactions Table                             │   │
│  │  (Sortable, filterable, paginated)             │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**Frontend Structure:**
```
src/
├── components/
│   ├── charts/           # Tremor wrapper components
│   ├── filters/          # Time range, merchant selector
│   └── tables/           # Data tables with export
├── hooks/
│   ├── useMetrics.ts     # GraphQL query hook
│   └── useRealtime.ts    # WebSocket / polling hook
├── pages/
│   ├── Dashboard.tsx
│   ├── Analytics.tsx
│   └── Reports.tsx
└── graphql/
    └── schema.graphql    # Generated queries
```

### 4. Report Scheduler

**Responsibility:** Generate and deliver scheduled reports.

```
┌────────────────┐
│  Cron Trigger  │
│  (Cloudflare)  │
└────────┬───────┘
         ▼
┌────────────────┐
│  Report        │
│  Generator     │
│  - Query data  │
│  - Render PDF  │
└────────┬───────┘
         ▼
┌────────────────┐
│  Email         │
│  Delivery      │
└────────────────┘
```

---

## Data Flow

### Real-time Metric Flow

```
1. Payment occurs at processor
2. Webhook sent to Tap2 Intel (Cloudflare Worker)
3. Transaction normalized and validated
4. Batch written to ClickHouse (every 10s or 1000 records)
5. Materialized views updated automatically
6. Dashboard queries refreshed (client polling / WebSocket push)
```

### Report Generation Flow

```
1. User schedules report (frequency, filters, recipients)
2. Schedule stored in Cloudflare Durable Objects
3. Cron trigger invokes report worker at scheduled time
4. Worker queries ClickHouse for data
5. Report rendered (PDF/CSV) and stored in R2
6. Email sent with download link
7. Link expires after 7 days
```

---

## Security Model

### Authentication & Authorization

```
┌──────────────┐
│  User        │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌────────────────┐
│  Tap2        │────▶│  Session Token │
│  OAuth       │     │  (JWT)         │
└──────────────┘     └────────┬───────┘
                              │
                              ▼
                     ┌────────────────┐
                     │  Role Check    │
                     │  (Admin/       │
                     │   Analyst/     │
                     │   Viewer)      │
                     └────────┬───────┘
                              │
                              ▼
                     ┌────────────────┐
                     │  Data Scope    │
                     │  (Merchant ID  │
                     │   filtering)   │
                     └────────────────┘
```

### Access Control Matrix

| Role | Dashboards | Analytics | Reports | API |
|------|------------|-----------|---------|-----|
| Viewer | View only | View only | View | Read |
| Analyst | View only | Full | Create/Edit | Read |
| Admin | Full | Full | Full | Read/Write |

### Data Isolation

- All queries scoped by `merchant_id`
- Row-level security enforced in query layer
- No cross-merchant data access

---

## Performance Considerations

### Query Optimization

| Strategy | Implementation |
|----------|----------------|
| Materialized Views | Pre-aggregate daily metrics |
| Query Caching | Cloudflare KV for hot queries |
| Pagination | Limit result sets, use cursors |
| Async Queries | Long-running reports via job queue |

### Caching Strategy

```
┌─────────────┐
│  Request    │
└──────┬──────┘
       │
       ▼
┌─────────────┐     Cache hit?     ┌─────────────┐
│  KV Cache   │────────────────────▶│  Return     │
│  (5 min TTL)│     No             │  Cached     │
└──────┬──────┘                    └─────────────┘
       │
       ▼
┌─────────────┐
│  ClickHouse │
│  Query      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Store in   │
│  Cache      │
└─────────────┘
```

### Targets

| Metric | Target |
|--------|--------|
| Dashboard load time | <2 seconds |
| API response time | <500ms (p95) |
| Webhook processing | <100ms (async) |
| Report generation | <30 seconds |

---

## Deployment Architecture

### Environments

| Environment | Purpose | URL |
|-------------|---------|-----|
| Development | Local testing | localhost:8788 |
| Staging | Pre-production | intel-staging.tap2.com |
| Production | Live service | intel.tap2.com |

### Deployment Pipeline

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Push   │───▶│  Test   │───▶│  Deploy │───▶│  Verify │
│  to     │    │  (CI)   │    │ (Wrangler)   │          │
│  main   │    │         │    │         │    │          │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
                                   │
                                   ▼
                            ┌─────────┐
                            │ Promote │
                            │ to Prod │
                            └─────────┘
```

### Rollback Strategy

- Zero-downtime deployments via Cloudflare gradual rollout
- Previous version always available for instant rollback
- Database migrations are backward-compatible

---

## Monitoring & Observability

### Key Metrics

| Category | Metrics |
|----------|---------|
| Business | Queries per day, active users, dashboard loads |
| Technical | API latency, error rate, webhook processing time |
| Infrastructure | Worker invocations, KV hits, ClickHouse CPU |

### Alerting

| Condition | Severity | Action |
|-----------|----------|--------|
| Error rate >5% | Critical | Page on-call |
| API latency p95 >2s | Warning | Create ticket |
| Webhook processing lag >1min | Warning | Investigate |

---

## Future Considerations

### Scalability

| Component | Current | Future Scale |
|-----------|---------|--------------|
| ClickHouse | Single node | Cluster with replication |
| Workers | Free tier | Paid tier with more CPU |
| Frontend | Static assets | CDN with edge caching |

### Potential Enhancements

- Machine Learning for anomaly detection
- Real-time streaming with Kafka alternative
- Multi-region deployment for latency
- Custom SQL query builder for analysts

---

**Last Updated:** February 2026
