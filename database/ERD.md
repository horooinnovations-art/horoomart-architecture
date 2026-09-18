# HOROOMART — Entity-Relationship Diagrams

One diagram per domain, generated from the enforced foreign keys. Entities show primary keys, foreign keys and the business-significant columns; the full column list for every table is in [SCHEMA.md](SCHEMA.md).

Entities owned by another domain appear with their keys only, so each diagram shows where its domain connects to the rest of the system. Relationship labels are the foreign-key column names.

## System overview

How the domains depend on each other (an arrow means "holds a foreign key into").

```mermaid
flowchart LR
    D0["Organization & Tenancy"]
    D1["Identity & Access Control"]
    D2["Catalog & Inventory"]
    D3["Procurement"]
    D4["Sales & Customers"]
    D5["Finance & Treasury"]
    D6["Human Resources"]
    D7["Fixed Assets"]
    D8["Platform & Operations"]
    D2 -->|9| D1
    D2 -->|20| D0
    D2 -->|2| D3
    D5 -->|14| D1
    D5 -->|17| D0
    D7 -->|1| D1
    D7 -->|3| D0
    D6 -->|2| D5
    D6 -->|4| D1
    D6 -->|4| D0
    D1 -->|4| D0
    D1 -->|1| D4
    D0 -->|2| D1
    D8 -->|3| D1
    D8 -->|2| D0
    D3 -->|2| D2
    D3 -->|2| D1
    D3 -->|4| D0
    D4 -->|2| D2
    D4 -->|1| D5
    D4 -->|4| D1
    D4 -->|5| D0
```

Edge labels count the foreign keys crossing between two domains. Nearly every domain points at *Organization & Tenancy* and *Identity & Access Control*: every record is scoped to a tenant and attributed to the user who created it.

## Organization & Tenancy

Every business record is scoped to an organization. Branches are points of sale; stores are org-level warehouses used by the optional Store → POS workflow.

```mermaid
erDiagram
    organizations {
        bigint id PK
        string name
        string legal_name
        string tax_id UK
        string registration_number
        string postal_code
        timestamp updated_at
    }
    branches {
        bigint id PK
        bigint organization_id FK
        string code UK
        string name
        string postal_code
        bigint manager_id FK
        date opening_date
        timestamp updated_at
    }
    stores {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
        string code UK
        string name
        string postal_code
        timestamp updated_at
    }
    settings {
        bigint id PK
        bigint organization_id FK
        string type
        timestamp updated_at
    }
    organizations ||--o{ branches : "organization_id"
    organizations ||--o{ stores : "organization_id"
    organizations |o--o{ settings : "organization_id"
```

*Omitted for readability:* every table here also references `organizations` for tenancy, and `users` for attribution via `manager_id`.

## Identity & Access Control

Users, role-based permissions (Spatie model: roles are organization-scoped), 2FA trusted devices, and per-user read receipts.

```mermaid
erDiagram
    users {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint store_id FK
        string employee_id UK
        string first_name
        string last_name
        string email UK
        string postal_code
        date date_of_birth
        date hire_date
        timestamp updated_at
    }
    roles {
        bigint id PK
        string name
        string guard_name
        bigint organization_id FK
        timestamp updated_at
    }
    permissions {
        bigint id PK
        string name
        string guard_name
        timestamp updated_at
    }
    model_has_roles {
        bigint role_id PK
        string model_type PK
        bigint model_id PK
    }
    model_has_permissions {
        bigint permission_id PK
        string model_type PK
        bigint model_id PK
    }
    role_has_permissions {
        bigint permission_id PK
        bigint role_id PK
    }
    trusted_devices {
        bigint id PK
        bigint user_id FK
        string token_hash UK
        string device_name
        bigint blocked_by FK
        timestamp updated_at
    }
    user_sale_reads {
        bigint id PK
        bigint user_id FK
        bigint sale_id FK
    }
    user_notification_reads {
        bigint id PK
        bigint user_id FK
        string notification_type
    }
    branches {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    sales {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint user_id FK
        bigint customer_id FK
    }
    stores {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    branches |o--o{ users : "branch_id"
    stores |o--o{ users : "store_id"
    users |o--o{ trusted_devices : "blocked_by"
    users ||--o{ trusted_devices : "user_id"
    sales ||--o{ user_sale_reads : "sale_id"
    users ||--o{ user_sale_reads : "user_id"
    users ||--o{ user_notification_reads : "user_id"
```

## Catalog & Inventory

Items and their per-branch / per-store stock, with every movement (adjustment, request, receipt, transfer) recorded as its own document with line items.

```mermaid
erDiagram
    items {
        bigint id PK
        bigint organization_id FK
        bigint category_id FK
        bigint manufacturer_id FK
        string sku UK
        string barcode UK
        string name
        enum type
        timestamp updated_at
    }
    categories {
        bigint id PK
        bigint organization_id FK
        bigint parent_id FK
        string code UK
        string name
        timestamp updated_at
    }
    manufacturers {
        bigint id PK
        bigint organization_id FK
        string code UK
        string name
        timestamp updated_at
    }
    item_stocks {
        bigint id PK
        bigint item_id FK
        bigint branch_id FK
        int quantity
        int store_quantity
        int reserved_quantity
        int available_quantity
        timestamp last_updated_at
        timestamp updated_at
    }
    store_stocks {
        bigint id PK
        bigint store_id FK
        bigint item_id FK
        int quantity
        int reserved_quantity
        int available_quantity
        timestamp last_updated_at
        timestamp updated_at
    }
    stock_adjustments {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint store_id FK
        bigint item_id FK
        bigint adjusted_by FK
        string adjustment_number
        date adjustment_date
        enum adjustment_type
        int quantity_before
        int adjustment_quantity
        int quantity_after
        string reference_number
        timestamp updated_at
    }
    stock_requests {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint store_id FK
        bigint requested_by FK
        bigint reviewed_by FK
        string request_number
        enum status
        timestamp updated_at
    }
    stock_request_items {
        bigint id PK
        bigint stock_request_id FK
        bigint item_id FK
        int requested_quantity
        timestamp updated_at
    }
    goods_receipts {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint store_id FK
        bigint purchase_order_id FK
        bigint received_by FK
        bigint approved_by FK
        string receipt_number
        date received_date
        enum status
        timestamp updated_at
    }
    goods_receipt_items {
        bigint id PK
        bigint goods_receipt_id FK
        bigint purchase_order_item_id FK
        bigint item_id FK
        int received_quantity
        timestamp updated_at
    }
    stock_unit_transfers {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint sent_by FK
        bigint accepted_by FK
        string transfer_number
        enum status
        timestamp updated_at
    }
    stock_unit_transfer_items {
        bigint id PK
        bigint stock_unit_transfer_id FK
        bigint item_id FK
        int quantity
        timestamp updated_at
    }
    branch_transfers {
        bigint id PK
        bigint organization_id FK
        bigint from_branch_id FK
        bigint from_store_id FK
        bigint to_branch_id FK
        bigint sent_by FK
        bigint received_by FK
        string transfer_number
        enum status
        timestamp updated_at
    }
    branch_transfer_items {
        bigint id PK
        bigint branch_transfer_id FK
        bigint item_id FK
        int quantity
        timestamp updated_at
    }
    branches {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    purchase_order_items {
        bigint id PK
        bigint purchase_order_id FK
        bigint item_id FK
    }
    purchase_orders {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint store_id FK
        bigint supplier_id FK
        bigint stock_request_id FK
        bigint created_by FK
        bigint approved_by FK
    }
    stores {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    categories |o--o{ items : "category_id"
    manufacturers |o--o{ items : "manufacturer_id"
    categories |o--o{ categories : "parent_id"
    branches ||--o{ item_stocks : "branch_id"
    items ||--o{ item_stocks : "item_id"
    items ||--o{ store_stocks : "item_id"
    stores ||--o{ store_stocks : "store_id"
    branches |o--o{ stock_adjustments : "branch_id"
    items ||--o{ stock_adjustments : "item_id"
    stores |o--o{ stock_adjustments : "store_id"
    branches ||--o{ stock_requests : "branch_id"
    stores |o--o{ stock_requests : "store_id"
    items ||--o{ stock_request_items : "item_id"
    stock_requests ||--o{ stock_request_items : "stock_request_id"
    branches |o--o{ goods_receipts : "branch_id"
    purchase_orders ||--o{ goods_receipts : "purchase_order_id"
    stores |o--o{ goods_receipts : "store_id"
    goods_receipts ||--o{ goods_receipt_items : "goods_receipt_id"
    items ||--o{ goods_receipt_items : "item_id"
    purchase_order_items ||--o{ goods_receipt_items : "purchase_order_item_id"
    branches ||--o{ stock_unit_transfers : "branch_id"
    items ||--o{ stock_unit_transfer_items : "item_id"
    stock_unit_transfers ||--o{ stock_unit_transfer_items : "stock_unit_transfer_id"
    branches |o--o{ branch_transfers : "from_branch_id"
    stores |o--o{ branch_transfers : "from_store_id"
    branches ||--o{ branch_transfers : "to_branch_id"
    branch_transfers ||--o{ branch_transfer_items : "branch_transfer_id"
    items ||--o{ branch_transfer_items : "item_id"
```

*Omitted for readability:* every table here also references `organizations` for tenancy, and `users` for attribution via `accepted_by`, `adjusted_by`, `approved_by`, `received_by`, `requested_by`, `reviewed_by`, `sent_by`.

## Procurement

Suppliers and purchase orders with an approval workflow and payment tracking.

```mermaid
erDiagram
    suppliers {
        bigint id PK
        bigint organization_id FK
        string name
        string legal_name
        string postal_code
        timestamp updated_at
    }
    purchase_orders {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint store_id FK
        bigint supplier_id FK
        bigint stock_request_id FK
        bigint created_by FK
        string po_number
        date order_date
        date expected_delivery_date
        enum status
        bigint approved_by FK
        decimal subtotal
        decimal tax_amount
        decimal discount_amount
        decimal total_amount
        decimal paid_amount
        enum payment_status
        date payment_due_date
        timestamp updated_at
    }
    purchase_order_items {
        bigint id PK
        bigint purchase_order_id FK
        bigint item_id FK
        int quantity
        int received_quantity
        decimal line_total
        timestamp updated_at
    }
    branches {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    items {
        bigint id PK
        bigint organization_id FK
        bigint category_id FK
        bigint manufacturer_id FK
    }
    stock_requests {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint store_id FK
        bigint requested_by FK
        bigint reviewed_by FK
    }
    stores {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    branches |o--o{ purchase_orders : "branch_id"
    stock_requests |o--o{ purchase_orders : "stock_request_id"
    stores |o--o{ purchase_orders : "store_id"
    suppliers ||--o{ purchase_orders : "supplier_id"
    items ||--o{ purchase_order_items : "item_id"
    purchase_orders ||--o{ purchase_order_items : "purchase_order_id"
```

*Omitted for readability:* every table here also references `organizations` for tenancy, and `users` for attribution via `approved_by`, `created_by`.

## Sales & Customers

Point-of-sale transactions and credit (on-account) sales with instalment collection.

```mermaid
erDiagram
    sales {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint user_id FK
        bigint customer_id FK
        string sale_number
        datetime sale_date
        enum status
        enum payment_status
        decimal subtotal
        decimal tax_amount
        decimal discount_amount
        decimal total_amount
        decimal amount_paid
        decimal change_amount
        timestamp updated_at
    }
    sale_items {
        bigint id PK
        bigint sale_id FK
        bigint item_id FK
        int quantity
        decimal line_total
        timestamp updated_at
    }
    customers {
        bigint id PK
        bigint organization_id FK
        string code UK
        string first_name
        string last_name
        string postal_code
        date date_of_birth
        decimal balance
        timestamp updated_at
    }
    credit_sales {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint customer_id FK
        bigint created_by FK
        bigint approved_by FK
        string credit_sale_number
        enum status
        decimal subtotal
        decimal tax_amount
        decimal discount_amount
        decimal total_amount
        decimal paid_amount
        date issued_date
        date due_date
        timestamp updated_at
    }
    credit_sale_items {
        bigint id PK
        bigint credit_sale_id FK
        bigint item_id FK
        string item_name
        decimal quantity
        decimal line_total
        timestamp updated_at
    }
    credit_sale_payments {
        bigint id PK
        bigint credit_sale_id FK
        bigint created_by FK
        decimal amount
        bigint bank_id FK
        string bank_name
        date payment_date
        timestamp updated_at
    }
    banks {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
    }
    branches {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    items {
        bigint id PK
        bigint organization_id FK
        bigint category_id FK
        bigint manufacturer_id FK
    }
    branches ||--o{ sales : "branch_id"
    customers |o--o{ sales : "customer_id"
    items ||--o{ sale_items : "item_id"
    sales ||--o{ sale_items : "sale_id"
    branches ||--o{ credit_sales : "branch_id"
    customers ||--o{ credit_sales : "customer_id"
    credit_sales ||--o{ credit_sale_items : "credit_sale_id"
    items |o--o{ credit_sale_items : "item_id"
    banks |o--o{ credit_sale_payments : "bank_id"
    credit_sales ||--o{ credit_sale_payments : "credit_sale_id"
```

*Omitted for readability:* every table here also references `organizations` for tenancy, and `users` for attribution via `approved_by`, `created_by`, `user_id`.

## Finance & Treasury

Bank and mobile-wallet accounts with a transaction ledger, payment methods mapped to settlement accounts, inter-account transfers, expenses (one-off and recurring) and loans.

```mermaid
erDiagram
    banks {
        bigint id PK
        bigint organization_id FK
        enum account_type
        bigint branch_id FK
        string name
        string code
        text account_number
        string account_name
        string branch_name
        text swift_code
        decimal opening_balance
        decimal current_balance
        timestamp updated_at
    }
    bank_transactions {
        bigint id PK
        bigint organization_id FK
        bigint bank_id FK
        enum transaction_type
        string source_type
        bigint payment_id FK
        decimal amount
        decimal balance_before
        decimal balance_after
        string reference_number
        bigint recorded_by FK
        date transaction_date
        timestamp updated_at
    }
    payment_methods {
        bigint id PK
        bigint organization_id FK
        string code
        string name
        string type
        bigint default_bank_id FK
        timestamp updated_at
    }
    payments {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        string payable_type
        bigint payment_method_id FK
        bigint bank_id FK
        bigint user_id FK
        string reference_number
        decimal amount
        date payment_date
        enum status
        timestamp updated_at
    }
    payment_transfers {
        bigint id PK
        bigint organization_id FK
        bigint from_bank_id FK
        bigint to_bank_id FK
        decimal amount
        string reference_number
        bigint transferred_by FK
        enum status
        bigint approved_by FK
        date transaction_date
        timestamp updated_at
    }
    expenses {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint expense_category_id FK
        bigint created_by FK
        bigint approved_by FK
        string expense_number
        decimal amount
        date expense_date
        enum status
        bigint bank_id FK
        string bank_name
        timestamp updated_at
    }
    expense_categories {
        bigint id PK
        bigint organization_id FK
        string name
        timestamp updated_at
    }
    fixed_expenses {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint expense_category_id FK
        decimal amount
        date due_date
        bigint bank_id FK
        enum status
        bigint last_generated_expense_id FK
        bigint created_by FK
        timestamp updated_at
    }
    loans {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        enum counterparty_type
        string counterparty_name
        string loan_number
        decimal principal_amount
        date start_date
        date due_date
        decimal outstanding_balance
        enum status
        bigint approved_by FK
        bigint bank_id FK
        bigint created_by FK
        timestamp updated_at
    }
    loan_installments {
        bigint id PK
        bigint loan_id FK
        int installment_number
        date due_date
        decimal amount_due
        decimal amount_paid
        enum status
        timestamp updated_at
    }
    loan_authorized_users {
        bigint id PK
        bigint loan_id FK
        bigint user_id FK
        bigint granted_by FK
        timestamp updated_at
    }
    personal_categories {
        bigint id PK
        bigint organization_id FK
        string name
        enum type
        timestamp updated_at
    }
    personal_transactions {
        bigint id PK
        bigint organization_id FK
        bigint bank_id FK
        bigint personal_category_id FK
        bigint created_by FK
        bigint approved_by FK
        enum type
        string transaction_number
        decimal amount
        date transaction_date
        enum status
        timestamp updated_at
    }
    personal_bank_transactions {
        bigint id PK
        bigint organization_id FK
        bigint bank_id FK
        enum transaction_type
        string source_type
        decimal amount
        decimal balance_before
        decimal balance_after
        string reference_number
        bigint recorded_by FK
        date transaction_date
        timestamp updated_at
    }
    branches {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    branches |o--o{ banks : "branch_id"
    banks ||--o{ bank_transactions : "bank_id"
    payments |o--o{ bank_transactions : "payment_id"
    banks |o--o{ payment_methods : "default_bank_id"
    banks |o--o{ payments : "bank_id"
    branches ||--o{ payments : "branch_id"
    payment_methods ||--o{ payments : "payment_method_id"
    banks ||--o{ payment_transfers : "from_bank_id"
    banks ||--o{ payment_transfers : "to_bank_id"
    banks |o--o{ expenses : "bank_id"
    branches ||--o{ expenses : "branch_id"
    expense_categories ||--o{ expenses : "expense_category_id"
    banks |o--o{ fixed_expenses : "bank_id"
    branches |o--o{ fixed_expenses : "branch_id"
    expense_categories ||--o{ fixed_expenses : "expense_category_id"
    expenses |o--o{ fixed_expenses : "last_generated_expense_id"
    banks |o--o{ loans : "bank_id"
    branches |o--o{ loans : "branch_id"
    loans ||--o{ loan_installments : "loan_id"
    loans ||--o{ loan_authorized_users : "loan_id"
    banks ||--o{ personal_transactions : "bank_id"
    personal_categories ||--o{ personal_transactions : "personal_category_id"
    banks ||--o{ personal_bank_transactions : "bank_id"
```

*Omitted for readability:* every table here also references `organizations` for tenancy, and `users` for attribution via `approved_by`, `created_by`, `granted_by`, `recorded_by`, `transferred_by`, `user_id`.

## Human Resources

Employees and payroll runs with per-employee payroll lines.

```mermaid
erDiagram
    employees {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint user_id FK
        string employee_number
        string first_name
        string last_name
        date date_of_birth
        enum employee_type
        date hire_date
        date termination_date
        enum status
        bigint bank_id FK
        bigint created_by FK
        timestamp updated_at
    }
    payroll_runs {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        string run_number
        date pay_date
        enum status
        decimal total_gross
        decimal total_deductions
        decimal total_net
        bigint created_by FK
        bigint approved_by FK
        timestamp updated_at
    }
    payroll_items {
        bigint id PK
        bigint payroll_run_id FK
        bigint employee_id FK
        bigint bank_id FK
        enum status
        timestamp updated_at
    }
    banks {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
    }
    branches {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    banks |o--o{ employees : "bank_id"
    branches |o--o{ employees : "branch_id"
    branches |o--o{ payroll_runs : "branch_id"
    banks |o--o{ payroll_items : "bank_id"
    employees ||--o{ payroll_items : "employee_id"
    payroll_runs ||--o{ payroll_items : "payroll_run_id"
```

*Omitted for readability:* every table here also references `organizations` for tenancy, and `users` for attribution via `approved_by`, `created_by`, `user_id`.

## Fixed Assets

Asset register with categories and unique serial numbers.

```mermaid
erDiagram
    assets {
        bigint id PK
        bigint organization_id FK
        bigint branch_id FK
        bigint asset_category_id FK
        string asset_number
        string name
        string serial_number
        date purchase_date
        enum status
        bigint created_by FK
        timestamp updated_at
    }
    asset_categories {
        bigint id PK
        bigint organization_id FK
        string name
        timestamp updated_at
    }
    branches {
        bigint id PK
        bigint organization_id FK
        bigint manager_id FK
    }
    asset_categories |o--o{ assets : "asset_category_id"
    branches |o--o{ assets : "branch_id"
```

*Omitted for readability:* every table here also references `organizations` for tenancy, and `users` for attribution via `created_by`.

## Platform & Operations

Audit trail, operational alerts, asynchronous report exports and controlled database restores.

```mermaid
erDiagram
    audit_logs {
        bigint id PK
        bigint organization_id FK
        bigint user_id FK
        string model_type
    }
    alert_logs {
        bigint id PK
        bigint organization_id FK
        string alertable_type
        bigint recipient_user_id FK
        enum status
        timestamp updated_at
    }
    export_jobs {
        bigint id PK
        string report_type
        string status
        timestamp updated_at
    }
    database_restores {
        bigint id PK
        bigint user_id FK
        string backup_filename
        enum status
        timestamp updated_at
    }
```

*Omitted for readability:* every table here also references `organizations` for tenancy, and `users` for attribution via `recipient_user_id`, `user_id`.

