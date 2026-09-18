# HOROOMART — Architecture & Database Design

**HOROOMART** is a multi-branch business management (ERP) platform for trading businesses,
built by [HOROO Innovations](https://github.com/horooinnovations-art). Supermarkets,
wholesalers, distributors, pharmacies and multi-branch shops run point of sale, inventory and
warehousing, procurement, credit sales, finance, HR and payroll, fixed assets and reporting
from a single system, with every record scoped to its organization.

This repository publishes the system's **design**: its database schema, domain architecture,
financial-integrity model, security architecture and engineering practices. The application
source code is proprietary and is not included.

---

## What's here

| Document | Contents |
|---|---|
| [Architecture](docs/ARCHITECTURE.md) | Domain structure, multi-tenancy, request lifecycle, service layer, integrations |
| [Database: data dictionary](database/SCHEMA.md) | Every table and column, with types, keys, defaults, foreign keys and indexes |
| [Database: ER diagrams](database/ERD.md) | One entity-relationship diagram per domain, plus a system-wide dependency map |
| [Database: DDL](database/schema.sql) | Complete MySQL schema, structure only, no data |
| [Financial integrity](docs/FINANCIAL_INTEGRITY.md) | How money movements are posted, locked, de-duplicated and kept immutable |
| [Security architecture](docs/SECURITY_ARCHITECTURE.md) | Authentication, authorization, data protection and application hardening |
| [Engineering practices](docs/ENGINEERING_PRACTICES.md) | Test strategy, concurrency control, data integrity and the audit process |

## The system at a glance

| | |
|---|---|
| **Domains** | 15 bounded modules: Organization, User, Inventory, Procurement, Sales, Customer, Supplier, Financial, HR, Loans, Assets, Alerts, Reports, AI, Shared |
| **Database** | 65 tables · 864 columns · 157 enforced foreign keys · 301 indexes |
| **Automated tests** | 306 tests · 948 assertions, run against a database migrated from empty |
| **Stack** | Laravel 13 · PHP 8.4 · MySQL 8 (InnoDB) · Redis · Blade + Alpine.js (CSP build) · Tailwind CSS · Vite |
| **Languages** | English, Amharic (አማርኛ), Afaan Oromoo |
| **Deployment** | Progressive Web App with offline sale queue; REST API (`/api/v1`) with token authentication |

## Capabilities

- **Point of sale.** Server-side pricing, per-item tax, cash/bank/mobile-wallet tender,
  idempotent checkout, receipts, and an offline queue that syncs when connectivity returns.
- **Credit sales.** On-account sales with instalment collection and aging.
- **Inventory.** Per-branch and per-store stock, adjustments, stock requests, goods receipts,
  branch-to-branch and store-to-branch transfers, with stock reserved at dispatch. Bin cards
  give full movement history per item.
- **Two operating models, per organization.** *POS-only*, where purchases land directly at
  branches, and *Store & POS*, where purchases land in a central store and are issued to
  branches before they become sellable.
- **Procurement.** Suppliers, purchase orders with approval workflow, partial receipts and
  payment tracking.
- **Finance.** Bank and mobile-wallet accounts (e.g. Telebirr) with a running-balance ledger,
  payment-method-to-account mapping, inter-account transfers with approval, one-off and
  recurring expenses, loans with instalment schedules, and financial reports.
- **HR & payroll.** Employee records and payroll runs that post to the ledger.
- **Fixed assets.** An asset register with categories and serial-number tracking.
- **Analytics.** Sales trends, anomaly detection and per-item demand forecasting (a regularized
  regression model trained on each organization's own sales history).
- **Reporting.** 18+ report types exported to PDF, Excel and CSV, generated asynchronously
  on a job queue so large reports never time out a request.
- **Operations.** Daily database backups with off-site replication and automated verification,
  due-date alerts via email and WhatsApp, and failure alerting.

## Design principles

1. **The database enforces integrity, not just the application.** Relationships are real
   foreign keys with a deliberate `ON DELETE` rule each. Application bugs cannot create orphans.
2. **Every money movement reaches a ledger, or it fails loudly.** No payment path can record a
   transaction without posting it to an account. See [Financial integrity](docs/FINANCIAL_INTEGRITY.md).
3. **Retries are safe.** Sales, payments, transfers and stock adjustments carry idempotency keys,
   so a double-tap or network retry cannot double-post.
4. **Concurrency is designed for, not hoped away.** Stock, balances, approvals and document
   numbering are protected by row locks and verified by concurrent-request tests.
5. **Finalized financial records are immutable.** Completed sales, approved expenses and
   confirmed credit sales cannot be edited. Corrections are new, auditable records.
6. **Nobody approves their own work.** Segregation of duties is enforced in code, not policy.

---

© 2026 HOROO Innovations. All rights reserved. This repository is published for reference and
evaluation; see [LICENSE](LICENSE).
