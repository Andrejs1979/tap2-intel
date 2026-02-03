# Epics

User stories and acceptance criteria for Tap2 Intel features.

---

## Epic 1: Real-Time Dashboards

### Overview
Live payment performance metrics visualization.

### User Stories

#### US-1.1: GMV Tracking Dashboard
**As a** finance manager
**I want to** see real-time gross merchandise volume
**So that** I can monitor revenue trends throughout the day

**Acceptance Criteria:**
- [ ] Display current day GMV with real-time updates
- [ ] Show GMV trend over time (hourly/daily/weekly views)
- [ ] Compare current period to previous period
- [ ] Break down by payment method, processor, and geography
- [ ] Data refreshes automatically every 30 seconds

#### US-1.2: Approval Rates Monitor
**As a** payments operations manager
**I want to** see payment approval rates across dimensions
**So that** I can identify and resolve processing issues quickly

**Acceptance Criteria:**
- [ ] Display overall approval rate percentage
- [ ] Break down by processor, card brand, and geography
- [ ] Show trend over time (hourly/daily/weekly)
- [ ] Alert when approval rate drops below threshold
- [ ] Drill down into decline reasons

#### US-1.3: Cost Analysis View
**As a** finance analyst
**I want to** see processing cost breakdown
**So that** I can optimize payment routing and reduce fees

**Acceptance Criteria:**
- [ ] Display total processing costs for selected period
- [ ] Break down by processor and fee type
- [ ] Calculate effective rate (cost / GMV)
- [ ] Compare costs across processors
- [ ] Export cost data as CSV

---

## Epic 2: Customer Analytics

### Overview
Understand customer payment behavior and value.

### User Stories

#### US-2.1: Cohort Analysis
**As a** product manager
**I want to** track customer cohorts over time
**So that** I can understand retention and payment behavior patterns

**Acceptance Criteria:**
- [ ] Define cohorts by signup date, first payment, or custom attribute
- [ ] Show retention curves for each cohort
- [ ] Display average order value per cohort over time
- [ ] Compare cohorts side by side
- [ ] Export cohort data

#### US-2.2: Lifetime Value Calculator
**As a** finance executive
**I want to** see predicted customer lifetime value
**So that** I can make informed acquisition and retention decisions

**Acceptance Criteria:**
- [ ] Calculate LTV using historical payment data
- [ ] Segment LTV by customer attributes
- [ ] Show LTV trends over time
- [ ] Compare predicted vs actual LTV
- [ ] Export LTV reports

#### US-2.3: Churn Prediction
**As a** customer success manager
**I want to** see churn risk signals
**So that** I can intervene before customers leave

**Acceptance Criteria:**
- [ ] Identify customers with declining payment frequency
- [ ] Flag customers with failed payment attempts
- [ ] Calculate churn probability score
- [ ] Show churn risk by segment
- [ ] Export at-risk customer lists

---

## Epic 3: Custom Reporting

### Overview
Build and schedule custom reports.

### User Stories

#### US-3.1: Report Builder
**As a** business analyst
**I want to** create custom reports with drag-and-drop interface
**So that** I can get exactly the data I need without engineering help

**Acceptance Criteria:**
- [ ] Select metrics, dimensions, and filters
- [ ] Choose visualization type (table, chart, pivot)
- [ ] Preview report before saving
- [ ] Name and save reports for reuse
- [ ] Clone and edit existing reports

#### US-3.2: Scheduled Delivery
**As a** finance manager
**I want to** receive reports via email on a schedule
**So that** I don't have to manually check dashboards

**Acceptance Criteria:**
- [ ] Set delivery frequency (daily, weekly, monthly)
- [ ] Choose delivery time and format (PDF, CSV)
- [ ] Add multiple recipients
- [ ] Manage and cancel scheduled reports
- [ ] Test email delivery

#### US-3.3: Data Export
**As a** data analyst
**I want to** export raw data via API
**So that** I can integrate with other systems

**Acceptance Criteria:**
- [ ] GraphQL API for metrics query
- [ ] REST API for bulk export
- [ ] Pagination for large datasets
- [ ] Rate limiting and API keys
- [ ] API documentation and examples

---

## Epic 4: Platform Foundation

### Overview
Core infrastructure and integrations.

### User Stories

#### US-4.1: Processor Integration
**As a** system
**I want to** connect to payment processors
**So that** I can access transaction data

**Acceptance Criteria:**
- [ ] Stripe integration with webhook support
- [ ] Tap2 processor integration
- [ ] Extensible adapter pattern for new processors
- [ ] Error handling and retry logic
- [ ] Data normalization across processors

#### US-4.2: Data Pipeline
**As a** system
**I want to** process and store transaction data efficiently
**So that** queries are fast and data is accurate

**Acceptance Criteria:**
- [ ] ClickHouse database schema
- [ ] Stream processing for real-time data
- [ ] Batch processing for historical data
- [ ] Data validation and error logging
- [ ] Automatic backups and retention

#### US-4.3: Authentication & Access Control
**As a** platform admin
**I want to** manage user access
**So that** data remains secure

**Acceptance Criteria:**
- [ ] OAuth integration with Tap2
- [ ] Role-based access control (Admin, Analyst, Viewer)
- [ ] Data scope restrictions by merchant
- [ ] Audit logging for data access
- [ ] SSO support for Enterprise tier

---

## Priority Matrix

| Epic | Priority | Estimate | Dependencies |
|------|----------|----------|--------------|
| Platform Foundation | P0 | 4 weeks | None |
| Real-Time Dashboards | P1 | 3 weeks | Platform Foundation |
| Customer Analytics | P1 | 4 weeks | Platform Foundation |
| Custom Reporting | P2 | 3 weeks | Platform Foundation, Dashboards |

---

**Last Updated:** February 2026
