# HOROOMART — Data Dictionary

Generated from `information_schema` of a database built by running all migrations from empty. It describes the schema exactly as the application creates it.

| Tables | Columns | Enforced foreign keys | Indexes |
|---:|---:|---:|---:|
| 65 | 864 | 157 | 301 |

Diagrams: see [ERD.md](ERD.md). DDL: see [schema.sql](schema.sql).

## Conventions

- **Tenancy.** Business tables carry `organization_id`; most operational tables also carry `branch_id` and/or `store_id`.
- **Money** is stored as `DECIMAL`, never floating point.
- **Soft deletes** (`deleted_at`) on master data and financial documents, so history stays reconstructible.
- **Idempotency keys** on sales, payments, transfers and stock adjustments, so a retried request cannot double-post.
- **Document numbers** (sale, PO, transfer numbers) are unique *per organization*, not globally.
- **Referential integrity** is enforced by the database (InnoDB foreign keys), not only by application code. `ON DELETE` behaviour is chosen per relationship: `CASCADE` for owned children, `RESTRICT` where deletion would destroy financial history, `SET NULL` for optional links.

## Contents

- [Organization & Tenancy](#organization--tenancy) — `organizations`, `branches`, `stores`, `settings`
- [Identity & Access Control](#identity--access-control) — `users`, `roles`, `permissions`, `model_has_roles`, `model_has_permissions`, `role_has_permissions`, `trusted_devices`, `user_sale_reads`, `user_notification_reads`
- [Catalog & Inventory](#catalog--inventory) — `items`, `categories`, `manufacturers`, `item_stocks`, `store_stocks`, `stock_adjustments`, `stock_requests`, `stock_request_items`, `goods_receipts`, `goods_receipt_items`, `stock_unit_transfers`, `stock_unit_transfer_items`, `branch_transfers`, `branch_transfer_items`
- [Procurement](#procurement) — `suppliers`, `purchase_orders`, `purchase_order_items`
- [Sales & Customers](#sales--customers) — `sales`, `sale_items`, `customers`, `credit_sales`, `credit_sale_items`, `credit_sale_payments`
- [Finance & Treasury](#finance--treasury) — `banks`, `bank_transactions`, `payment_methods`, `payments`, `payment_transfers`, `expenses`, `expense_categories`, `fixed_expenses`, `loans`, `loan_installments`, `loan_authorized_users`, `personal_categories`, `personal_transactions`, `personal_bank_transactions`
- [Human Resources](#human-resources) — `employees`, `payroll_runs`, `payroll_items`
- [Fixed Assets](#fixed-assets) — `assets`, `asset_categories`
- [Platform & Operations](#platform--operations) — `audit_logs`, `alert_logs`, `export_jobs`, `database_restores`
- [Framework](#framework) — `cache`, `cache_locks`, `sessions`, `failed_jobs`, `migrations`, `password_reset_tokens`

## Organization & Tenancy

Every business record is scoped to an organization. Branches are points of sale; stores are org-level warehouses used by the optional Store → POS workflow.

### `organizations`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `name` | `varchar(191)` |  |  |  |  |
| `legal_name` | `varchar(191)` | ✓ |  |  |  |
| `tax_id` | `varchar(191)` | ✓ |  | UQ |  |
| `registration_number` | `varchar(191)` | ✓ |  |  |  |
| `email` | `varchar(191)` | ✓ |  |  |  |
| `phone` | `varchar(191)` | ✓ |  |  |  |
| `address` | `text` | ✓ |  |  |  |
| `city` | `varchar(191)` | ✓ |  |  |  |
| `state` | `varchar(191)` | ✓ |  |  |  |
| `postal_code` | `varchar(191)` | ✓ |  |  |  |
| `country` | `varchar(191)` |  | `US` |  |  |
| `logo` | `varchar(191)` | ✓ |  |  |  |
| `website` | `varchar(191)` | ✓ |  |  |  |
| `currency` | `varchar(3)` |  | `ETB` |  |  |
| `default_tax_rate` | `decimal(5,2)` |  | `15.00` |  |  |
| `timezone` | `varchar(191)` |  | `UTC` |  |  |
| `fiscal_year_start` | `date` | ✓ |  |  |  |
| `settings` | `json` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (1)</summary>

- **unique** `organizations_tax_id_unique` (`tax_id`)

</details>

### `branches`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `code` | `varchar(191)` |  |  | UQ |  |
| `name` | `varchar(191)` |  |  |  |  |
| `email` | `varchar(191)` | ✓ |  |  |  |
| `phone` | `varchar(191)` | ✓ |  |  |  |
| `address` | `text` | ✓ |  |  |  |
| `city` | `varchar(191)` | ✓ |  |  |  |
| `state` | `varchar(191)` | ✓ |  |  |  |
| `postal_code` | `varchar(191)` | ✓ |  |  |  |
| `country` | `varchar(191)` |  | `US` |  |  |
| `manager_id` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `opening_date` | `date` | ✓ |  |  |  |
| `settings` | `json` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- **unique** `branches_code_unique` (`code`)
- index `branches_manager_id_foreign` (`manager_id`)
- index `branches_organization_id_is_active_index` (`organization_id`, `is_active`)

</details>

### `stores`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `manager_id` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `code` | `varchar(191)` |  |  | UQ |  |
| `name` | `varchar(191)` |  |  |  |  |
| `email` | `varchar(191)` | ✓ |  |  |  |
| `phone` | `varchar(191)` | ✓ |  |  |  |
| `address` | `text` | ✓ |  |  |  |
| `city` | `varchar(191)` | ✓ |  |  |  |
| `state` | `varchar(191)` | ✓ |  |  |  |
| `postal_code` | `varchar(191)` | ✓ |  |  |  |
| `country` | `varchar(191)` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `settings` | `json` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- **unique** `stores_code_unique` (`code`)
- index `stores_manager_id_foreign` (`manager_id`)
- index `stores_organization_id_is_active_index` (`organization_id`, `is_active`)

</details>

### `settings`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` | ✓ |  | IX | → `organizations.id` (on delete cascade) |
| `key` | `varchar(191)` |  |  |  |  |
| `value` | `text` | ✓ |  |  |  |
| `type` | `varchar(191)` |  | `string` |  |  |
| `description` | `text` | ✓ |  |  |  |
| `group` | `varchar(191)` |  | `general` | IX |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- index `settings_group_index` (`group`)
- index `settings_organization_id_key_index` (`organization_id`, `key`)
- **unique** `settings_organization_id_key_unique` (`organization_id`, `key`)

</details>

## Identity & Access Control

Users, role-based permissions (Spatie model: roles are organization-scoped), 2FA trusted devices, and per-user read receipts.

### `users`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` | ✓ |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete set null) |
| `store_id` | `bigint unsigned` | ✓ |  | IX | → `stores.id` (on delete set null) |
| `employee_id` | `varchar(191)` | ✓ |  | UQ |  |
| `first_name` | `varchar(191)` |  |  |  |  |
| `last_name` | `varchar(191)` |  |  |  |  |
| `email` | `varchar(191)` |  |  | UQ |  |
| `email_verified_at` | `timestamp` | ✓ |  |  |  |
| `password` | `varchar(191)` |  |  |  |  |
| `must_change_password` | `tinyint(1)` |  | `0` |  |  |
| `phone` | `varchar(191)` | ✓ |  |  |  |
| `address` | `text` | ✓ |  |  |  |
| `city` | `varchar(191)` | ✓ |  |  |  |
| `state` | `varchar(191)` | ✓ |  |  |  |
| `postal_code` | `varchar(191)` | ✓ |  |  |  |
| `country` | `varchar(191)` |  | `US` |  |  |
| `date_of_birth` | `date` | ✓ |  |  |  |
| `hire_date` | `date` | ✓ |  |  |  |
| `position` | `varchar(191)` | ✓ |  |  |  |
| `department` | `varchar(191)` | ✓ |  |  |  |
| `salary` | `text` | ✓ |  |  |  |
| `avatar` | `varchar(191)` | ✓ |  |  |  |
| `settings` | `json` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` | IX |  |
| `last_login_at` | `timestamp` | ✓ |  |  |  |
| `remember_token` | `varchar(100)` | ✓ |  |  |  |
| `two_factor_secret` | `text` | ✓ |  |  |  |
| `two_factor_enabled` | `tinyint(1)` |  | `0` |  |  |
| `two_factor_confirmed_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (6)</summary>

- index `users_branch_id_foreign` (`branch_id`)
- **unique** `users_email_unique` (`email`)
- **unique** `users_employee_id_unique` (`employee_id`)
- index `users_is_active_index` (`is_active`)
- index `users_organization_id_branch_id_index` (`organization_id`, `branch_id`)
- index `users_store_id_foreign` (`store_id`)

</details>

### `roles`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `name` | `varchar(125)` |  |  | IX |  |
| `guard_name` | `varchar(125)` |  |  |  |  |
| `organization_id` | `bigint unsigned` | ✓ |  | IX | → `organizations.id` (on delete cascade) |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- **unique** `roles_name_guard_name_unique` (`name`, `guard_name`)
- index `roles_org_guard_idx` (`organization_id`, `guard_name`)

</details>

### `permissions`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `name` | `varchar(125)` |  |  | IX |  |
| `guard_name` | `varchar(125)` |  |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (1)</summary>

- **unique** `permissions_name_guard_name_unique` (`name`, `guard_name`)

</details>

### `model_has_roles`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `role_id` | `bigint unsigned` |  |  | PK |  |
| `model_type` | `varchar(191)` |  |  | PK |  |
| `model_id` | `bigint unsigned` |  |  | PK |  |

<details><summary>Indexes (1)</summary>

- index `model_has_roles_model_id_model_type_index` (`model_id`, `model_type`)

</details>

### `model_has_permissions`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `permission_id` | `bigint unsigned` |  |  | PK |  |
| `model_type` | `varchar(191)` |  |  | PK |  |
| `model_id` | `bigint unsigned` |  |  | PK |  |

<details><summary>Indexes (1)</summary>

- index `model_has_permissions_model_id_model_type_index` (`model_id`, `model_type`)

</details>

### `role_has_permissions`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `permission_id` | `bigint unsigned` |  |  | PK |  |
| `role_id` | `bigint unsigned` |  |  | PK |  |

<details><summary>Indexes (1)</summary>

- index `role_has_permissions_role_id_foreign` (`role_id`)

</details>

### `trusted_devices`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `user_id` | `bigint unsigned` |  |  | IX | → `users.id` (on delete cascade) |
| `token_hash` | `varchar(64)` |  |  | UQ |  |
| `device_name` | `varchar(191)` | ✓ |  |  |  |
| `user_agent` | `varchar(500)` | ✓ |  |  |  |
| `ip_address` | `varchar(45)` | ✓ |  |  |  |
| `last_used_at` | `timestamp` | ✓ |  |  |  |
| `expires_at` | `timestamp` |  |  |  |  |
| `blocked_at` | `timestamp` | ✓ |  |  |  |
| `blocked_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `block_reason` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- index `trusted_devices_blocked_by_foreign` (`blocked_by`)
- **unique** `trusted_devices_token_hash_unique` (`token_hash`)
- index `trusted_devices_user_id_expires_at_index` (`user_id`, `expires_at`)

</details>

### `user_sale_reads`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `user_id` | `bigint unsigned` |  |  | IX | → `users.id` (on delete cascade) |
| `sale_id` | `bigint unsigned` |  |  | IX | → `sales.id` (on delete cascade) |
| `read_at` | `timestamp` |  | `CURRENT_TIMESTAMP` |  |  |

<details><summary>Indexes (3)</summary>

- index `user_sale_reads_sale_id_foreign` (`sale_id`)
- index `user_sale_reads_user_id_read_at_index` (`user_id`, `read_at`)
- **unique** `user_sale_reads_user_id_sale_id_unique` (`user_id`, `sale_id`)

</details>

### `user_notification_reads`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `user_id` | `bigint unsigned` |  |  | IX | → `users.id` (on delete cascade) |
| `notification_type` | `varchar(50)` |  |  |  |  |
| `reference_id` | `bigint unsigned` |  |  |  |  |
| `read_at` | `timestamp` |  | `CURRENT_TIMESTAMP` |  |  |

<details><summary>Indexes (2)</summary>

- **unique** `user_notification_reads_unique` (`user_id`, `notification_type`, `reference_id`)
- index `user_notification_reads_user_id_notification_type_index` (`user_id`, `notification_type`)

</details>

## Catalog & Inventory

Items and their per-branch / per-store stock, with every movement (adjustment, request, receipt, transfer) recorded as its own document with line items.

### `items`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `category_id` | `bigint unsigned` | ✓ |  | IX | → `categories.id` (on delete set null) |
| `manufacturer_id` | `bigint unsigned` | ✓ |  | IX | → `manufacturers.id` (on delete set null) |
| `sku` | `varchar(191)` |  |  | UQ |  |
| `barcode` | `varchar(191)` | ✓ |  | UQ |  |
| `name` | `varchar(191)` |  |  |  |  |
| `description` | `text` | ✓ |  |  |  |
| `type` | `enum('product','service','bundle')` |  | `product` |  |  |
| `unit_of_measure` | `varchar(191)` |  | `pcs` |  |  |
| `cost_price` | `decimal(10,2)` | ✓ |  |  |  |
| `selling_price` | `decimal(10,2)` | ✓ |  |  |  |
| `min_stock_level` | `int` |  | `0` |  |  |
| `max_stock_level` | `int` |  | `0` |  |  |
| `reorder_point` | `int` |  | `0` |  |  |
| `weight` | `decimal(8,2)` | ✓ |  |  |  |
| `dimensions` | `json` | ✓ |  |  |  |
| `image` | `varchar(191)` | ✓ |  |  |  |
| `tax_rate` | `decimal(5,2)` |  | `0.00` |  |  |
| `is_taxable` | `tinyint(1)` |  | `1` |  |  |
| `is_active` | `tinyint(1)` |  | `1` | IX |  |
| `settings` | `json` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (6)</summary>

- **unique** `items_barcode_unique` (`barcode`)
- index `items_category_id_foreign` (`category_id`)
- index `items_is_active_index` (`is_active`)
- index `items_manufacturer_id_foreign` (`manufacturer_id`)
- index `items_organization_id_category_id_index` (`organization_id`, `category_id`)
- **unique** `items_sku_unique` (`sku`)

</details>

### `categories`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `parent_id` | `bigint unsigned` | ✓ |  | IX | → `categories.id` (on delete set null) |
| `code` | `varchar(191)` |  |  | UQ |  |
| `name` | `varchar(191)` |  |  |  |  |
| `description` | `text` | ✓ |  |  |  |
| `image` | `varchar(191)` | ✓ |  |  |  |
| `sort_order` | `int` |  | `0` |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- **unique** `categories_code_unique` (`code`)
- index `categories_organization_id_is_active_index` (`organization_id`, `is_active`)
- index `categories_parent_id_foreign` (`parent_id`)

</details>

### `manufacturers`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `code` | `varchar(191)` |  |  | UQ |  |
| `name` | `varchar(191)` |  |  |  |  |
| `country` | `varchar(191)` | ✓ |  |  |  |
| `website` | `varchar(191)` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- **unique** `manufacturers_code_unique` (`code`)
- index `manufacturers_organization_id_is_active_index` (`organization_id`, `is_active`)

</details>

### `item_stocks`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `item_id` | `bigint unsigned` |  |  | IX | → `items.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` |  |  | IX | → `branches.id` (on delete cascade) |
| `quantity` | `int` |  | `0` |  |  |
| `store_quantity` | `int` |  | `0` |  |  |
| `reserved_quantity` | `int` |  | `0` |  |  |
| `available_quantity` | `int` |  | `0` |  |  |
| `last_updated_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `item_stocks_branch_id_quantity_index` (`branch_id`, `quantity`)
- **unique** `item_stocks_item_id_branch_id_unique` (`item_id`, `branch_id`)

</details>

### `store_stocks`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `store_id` | `bigint unsigned` |  |  | IX | → `stores.id` (on delete cascade) |
| `item_id` | `bigint unsigned` |  |  | IX | → `items.id` (on delete restrict) |
| `quantity` | `int` |  | `0` |  |  |
| `reserved_quantity` | `int` |  | `0` |  |  |
| `available_quantity` | `int` |  | `0` |  |  |
| `last_updated_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- index `store_stocks_item_id_foreign` (`item_id`)
- **unique** `store_stocks_store_id_item_id_unique` (`store_id`, `item_id`)
- index `store_stocks_store_id_quantity_index` (`store_id`, `quantity`)

</details>

### `stock_adjustments`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `idempotency_key` | `varchar(64)` | ✓ |  |  |  |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete cascade) |
| `store_id` | `bigint unsigned` | ✓ |  | IX | → `stores.id` (on delete set null) |
| `item_id` | `bigint unsigned` |  |  | IX | → `items.id` (on delete restrict) |
| `adjusted_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `adjustment_number` | `varchar(191)` |  |  | IX |  |
| `adjustment_date` | `date` |  |  |  |  |
| `adjustment_type` | `enum('increase','decrease','set')` |  | `increase` |  |  |
| `quantity_before` | `int` |  |  |  |  |
| `adjustment_quantity` | `int` |  |  |  |  |
| `cost_price` | `decimal(10,2)` | ✓ |  |  |  |
| `selling_price` | `decimal(10,2)` | ✓ |  |  |  |
| `quantity_after` | `int` |  |  |  |  |
| `reason` | `enum('damaged','expired','returned','found','theft','correction','transfer_in','transfer_out','cycle_count','vendor_return','add_stock','price_update','sale_cancelled','other')` | ✓ | `correction` |  |  |
| `reason_notes` | `text` | ✓ |  |  |  |
| `reference_number` | `varchar(191)` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (8)</summary>

- index `stock_adj_item_idx` (`item_id`)
- index `stock_adj_number_idx` (`adjustment_number`)
- index `stock_adj_org_branch_date_idx` (`organization_id`, `branch_id`, `adjustment_date`)
- index `stock_adjustments_adjusted_by_foreign` (`adjusted_by`)
- index `stock_adjustments_branch_id_foreign` (`branch_id`)
- **unique** `stock_adjustments_org_adjustment_number_unique` (`organization_id`, `adjustment_number`)
- **unique** `stock_adjustments_org_idem_unique` (`organization_id`, `idempotency_key`)
- index `stock_adjustments_store_id_foreign` (`store_id`)

</details>

### `stock_requests`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` |  |  | IX | → `branches.id` (on delete cascade) |
| `store_id` | `bigint unsigned` | ✓ |  | IX | → `stores.id` (on delete set null) |
| `requested_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `reviewed_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `request_number` | `varchar(191)` |  |  |  |  |
| `status` | `enum('pending','reviewed','converted','rejected')` |  | `pending` |  |  |
| `reason` | `text` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `reviewed_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (6)</summary>

- index `stock_requests_branch_id_foreign` (`branch_id`)
- **unique** `stock_requests_org_request_number_unique` (`organization_id`, `request_number`)
- index `stock_requests_organization_id_branch_id_status_index` (`organization_id`, `branch_id`, `status`)
- index `stock_requests_requested_by_foreign` (`requested_by`)
- index `stock_requests_reviewed_by_foreign` (`reviewed_by`)
- index `stock_requests_store_id_foreign` (`store_id`)

</details>

### `stock_request_items`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `stock_request_id` | `bigint unsigned` |  |  | IX | → `stock_requests.id` (on delete cascade) |
| `item_id` | `bigint unsigned` |  |  | IX | → `items.id` (on delete restrict) |
| `requested_quantity` | `int` |  |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `stock_request_items_item_id_foreign` (`item_id`)
- index `stock_request_items_stock_request_id_index` (`stock_request_id`)

</details>

### `goods_receipts`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete cascade) |
| `store_id` | `bigint unsigned` | ✓ |  | IX | → `stores.id` (on delete set null) |
| `purchase_order_id` | `bigint unsigned` |  |  | IX | → `purchase_orders.id` (on delete cascade) |
| `received_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `approved_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `receipt_number` | `varchar(191)` |  |  |  |  |
| `received_date` | `date` |  |  |  |  |
| `status` | `enum('submitted','approved','rejected')` |  | `submitted` |  |  |
| `reason` | `text` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `approved_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (7)</summary>

- index `goods_receipts_approved_by_foreign` (`approved_by`)
- index `goods_receipts_branch_id_foreign` (`branch_id`)
- **unique** `goods_receipts_org_receipt_number_unique` (`organization_id`, `receipt_number`)
- index `goods_receipts_organization_id_branch_id_status_index` (`organization_id`, `branch_id`, `status`)
- index `goods_receipts_purchase_order_id_index` (`purchase_order_id`)
- index `goods_receipts_received_by_foreign` (`received_by`)
- index `goods_receipts_store_id_foreign` (`store_id`)

</details>

### `goods_receipt_items`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `goods_receipt_id` | `bigint unsigned` |  |  | IX | → `goods_receipts.id` (on delete cascade) |
| `purchase_order_item_id` | `bigint unsigned` |  |  | IX | → `purchase_order_items.id` (on delete cascade) |
| `item_id` | `bigint unsigned` |  |  | IX | → `items.id` (on delete restrict) |
| `received_quantity` | `int` |  |  |  |  |
| `unit_cost` | `decimal(10,2)` | ✓ |  |  |  |
| `selling_price` | `decimal(10,2)` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- index `goods_receipt_items_goods_receipt_id_index` (`goods_receipt_id`)
- index `goods_receipt_items_item_id_foreign` (`item_id`)
- index `goods_receipt_items_purchase_order_item_id_foreign` (`purchase_order_item_id`)

</details>

### `stock_unit_transfers`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` |  |  | IX | → `branches.id` (on delete cascade) |
| `sent_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `accepted_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `transfer_number` | `varchar(191)` |  |  |  |  |
| `status` | `enum('pending','accepted','rejected')` |  | `pending` |  |  |
| `reason` | `text` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `accepted_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (5)</summary>

- index `stock_unit_transfers_accepted_by_foreign` (`accepted_by`)
- index `stock_unit_transfers_branch_id_foreign` (`branch_id`)
- **unique** `stock_unit_transfers_org_transfer_number_unique` (`organization_id`, `transfer_number`)
- index `stock_unit_transfers_organization_id_branch_id_status_index` (`organization_id`, `branch_id`, `status`)
- index `stock_unit_transfers_sent_by_foreign` (`sent_by`)

</details>

### `stock_unit_transfer_items`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `stock_unit_transfer_id` | `bigint unsigned` |  |  | IX | → `stock_unit_transfers.id` (on delete cascade) |
| `item_id` | `bigint unsigned` |  |  | IX | → `items.id` (on delete restrict) |
| `quantity` | `int` |  |  |  |  |
| `unit_cost` | `decimal(10,2)` | ✓ |  |  |  |
| `selling_price` | `decimal(10,2)` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `stock_unit_transfer_items_item_id_foreign` (`item_id`)
- index `stock_unit_transfer_items_stock_unit_transfer_id_index` (`stock_unit_transfer_id`)

</details>

### `branch_transfers`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `from_branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete restrict) |
| `from_store_id` | `bigint unsigned` | ✓ |  | IX | → `stores.id` (on delete set null) |
| `to_branch_id` | `bigint unsigned` |  |  | IX | → `branches.id` (on delete restrict) |
| `sent_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `received_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `transfer_number` | `varchar(191)` |  |  |  |  |
| `status` | `enum('pending','received','rejected')` |  | `pending` |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `reason` | `text` | ✓ |  |  |  |
| `received_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (8)</summary>

- index `branch_transfers_from_branch_id_foreign` (`from_branch_id`)
- index `branch_transfers_from_store_id_foreign` (`from_store_id`)
- **unique** `branch_transfers_org_transfer_number_unique` (`organization_id`, `transfer_number`)
- index `branch_transfers_organization_id_from_branch_id_status_index` (`organization_id`, `from_branch_id`, `status`)
- index `branch_transfers_organization_id_to_branch_id_status_index` (`organization_id`, `to_branch_id`, `status`)
- index `branch_transfers_received_by_foreign` (`received_by`)
- index `branch_transfers_sent_by_foreign` (`sent_by`)
- index `branch_transfers_to_branch_id_foreign` (`to_branch_id`)

</details>

### `branch_transfer_items`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `branch_transfer_id` | `bigint unsigned` |  |  | IX | → `branch_transfers.id` (on delete cascade) |
| `item_id` | `bigint unsigned` |  |  | IX | → `items.id` (on delete restrict) |
| `quantity` | `int` |  |  |  |  |
| `unit_cost` | `decimal(10,2)` | ✓ |  |  |  |
| `selling_price` | `decimal(10,2)` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `branch_transfer_items_branch_transfer_id_index` (`branch_transfer_id`)
- index `branch_transfer_items_item_id_foreign` (`item_id`)

</details>

## Procurement

Suppliers and purchase orders with an approval workflow and payment tracking.

### `suppliers`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `tin` | `varchar(191)` | ✓ |  |  |  |
| `name` | `varchar(191)` |  |  |  |  |
| `legal_name` | `varchar(191)` | ✓ |  |  |  |
| `tax_id` | `varchar(191)` | ✓ |  |  |  |
| `email` | `varchar(191)` | ✓ |  |  |  |
| `phone` | `varchar(191)` | ✓ |  |  |  |
| `contact_person` | `varchar(191)` | ✓ |  |  |  |
| `address` | `text` | ✓ |  |  |  |
| `city` | `varchar(191)` | ✓ |  |  |  |
| `state` | `varchar(191)` | ✓ |  |  |  |
| `postal_code` | `varchar(191)` | ✓ |  |  |  |
| `country` | `varchar(191)` |  | `US` |  |  |
| `website` | `varchar(191)` | ✓ |  |  |  |
| `payment_terms` | `enum('net_15','net_30','net_45','net_60','cod','prepaid')` |  | `net_30` |  |  |
| `credit_limit` | `int` |  | `0` |  |  |
| `rating` | `decimal(3,2)` | ✓ |  |  | Supplier rating 0-5 |
| `notes` | `text` | ✓ |  |  |  |
| `settings` | `json` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `suppliers_organization_id_is_active_index` (`organization_id`, `is_active`)
- **unique** `suppliers_organization_id_tin_unique` (`organization_id`, `tin`)

</details>

### `purchase_orders`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete cascade) |
| `store_id` | `bigint unsigned` | ✓ |  | IX | → `stores.id` (on delete set null) |
| `supplier_id` | `bigint unsigned` |  |  | IX | → `suppliers.id` (on delete restrict) |
| `stock_request_id` | `bigint unsigned` | ✓ |  | IX | → `stock_requests.id` (on delete set null) |
| `created_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `po_number` | `varchar(191)` |  |  |  |  |
| `order_date` | `date` |  |  |  |  |
| `expected_delivery_date` | `date` | ✓ |  |  |  |
| `status` | `enum('draft','pending','approved','ordered','partial','received','cancelled')` |  | `draft` |  |  |
| `reason` | `text` | ✓ |  |  |  |
| `approved_at` | `timestamp` | ✓ |  |  |  |
| `approved_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `payment_terms` | `enum('net_15','net_30','net_45','net_60','cod','prepaid')` |  | `net_30` |  |  |
| `subtotal` | `decimal(12,2)` |  | `0.00` |  |  |
| `tax_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `discount_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `shipping_cost` | `decimal(12,2)` |  | `0.00` |  |  |
| `total_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `paid_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `payment_status` | `enum('unpaid','partial','paid')` |  | `unpaid` |  |  |
| `payment_due_date` | `date` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (9)</summary>

- index `purchase_orders_approved_by_foreign` (`approved_by`)
- index `purchase_orders_branch_id_foreign` (`branch_id`)
- index `purchase_orders_created_by_foreign` (`created_by`)
- index `purchase_orders_org_paystatus_duedate_index` (`organization_id`, `payment_status`, `payment_due_date`)
- **unique** `purchase_orders_org_po_number_unique` (`organization_id`, `po_number`)
- index `purchase_orders_organization_id_branch_id_status_index` (`organization_id`, `branch_id`, `status`)
- index `purchase_orders_stock_request_id_foreign` (`stock_request_id`)
- index `purchase_orders_store_id_foreign` (`store_id`)
- index `purchase_orders_supplier_id_index` (`supplier_id`)

</details>

### `purchase_order_items`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `purchase_order_id` | `bigint unsigned` |  |  | IX | → `purchase_orders.id` (on delete cascade) |
| `item_id` | `bigint unsigned` |  |  | IX | → `items.id` (on delete restrict) |
| `quantity` | `int` |  |  |  |  |
| `received_quantity` | `int` |  | `0` |  |  |
| `unit_cost` | `decimal(10,2)` |  |  |  |  |
| `selling_price` | `decimal(10,2)` | ✓ |  |  |  |
| `margin_percentage` | `decimal(10,2)` | ✓ |  |  |  |
| `discount_percentage` | `decimal(5,2)` |  | `0.00` |  |  |
| `tax_rate` | `decimal(5,2)` |  | `0.00` |  |  |
| `line_total` | `decimal(12,2)` |  |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `purchase_order_items_item_id_foreign` (`item_id`)
- index `purchase_order_items_purchase_order_id_index` (`purchase_order_id`)

</details>

## Sales & Customers

Point-of-sale transactions and credit (on-account) sales with instalment collection.

### `sales`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` |  |  | IX | → `branches.id` (on delete cascade) |
| `user_id` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `customer_id` | `bigint unsigned` | ✓ |  | IX | → `customers.id` (on delete set null) |
| `sale_number` | `varchar(191)` |  |  |  |  |
| `idempotency_key` | `varchar(64)` | ✓ |  |  |  |
| `sale_date` | `datetime` |  |  |  |  |
| `status` | `enum('draft','pending','completed','cancelled','refunded')` | ✓ | `pending` | IX |  |
| `reason` | `text` | ✓ |  |  |  |
| `payment_status` | `enum('pending','partial','paid','refunded')` |  | `pending` |  |  |
| `subtotal` | `decimal(12,2)` |  | `0.00` |  |  |
| `tax_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `discount_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `total_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `amount_paid` | `decimal(12,2)` |  | `0.00` |  |  |
| `change_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (8)</summary>

- index `sales_branch_id_foreign` (`branch_id`)
- index `sales_customer_id_index` (`customer_id`)
- **unique** `sales_org_idempotency_key_unique` (`organization_id`, `idempotency_key`)
- **unique** `sales_org_sale_number_unique` (`organization_id`, `sale_number`)
- index `sales_org_status_date_index` (`organization_id`, `status`, `sale_date`)
- index `sales_organization_id_branch_id_sale_date_index` (`organization_id`, `branch_id`, `sale_date`)
- index `sales_status_payment_status_index` (`status`, `payment_status`)
- index `sales_user_id_foreign` (`user_id`)

</details>

### `sale_items`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `sale_id` | `bigint unsigned` |  |  | IX | → `sales.id` (on delete cascade) |
| `item_id` | `bigint unsigned` |  |  | IX | → `items.id` (on delete restrict) |
| `quantity` | `int` |  |  |  |  |
| `unit_price` | `decimal(10,2)` |  |  |  |  |
| `additional_price` | `decimal(10,2)` |  | `0.00` |  |  |
| `discount_percentage` | `decimal(5,2)` |  | `0.00` |  |  |
| `tax_rate` | `decimal(5,2)` |  | `0.00` |  |  |
| `line_total` | `decimal(12,2)` |  |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `sale_items_item_id_index` (`item_id`)
- index `sale_items_sale_id_index` (`sale_id`)

</details>

### `customers`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `code` | `varchar(191)` |  |  | UQ |  |
| `first_name` | `varchar(191)` |  |  |  |  |
| `last_name` | `varchar(191)` |  |  |  |  |
| `email` | `varchar(191)` | ✓ |  |  |  |
| `phone` | `text` | ✓ |  |  |  |
| `address` | `text` | ✓ |  |  |  |
| `city` | `varchar(191)` | ✓ |  |  |  |
| `state` | `varchar(191)` | ✓ |  |  |  |
| `postal_code` | `varchar(191)` | ✓ |  |  |  |
| `country` | `varchar(191)` |  | `ET` |  |  |
| `date_of_birth` | `date` | ✓ |  |  |  |
| `gender` | `enum('male','female','other')` | ✓ |  |  |  |
| `credit_limit` | `decimal(10,2)` |  | `0.00` |  |  |
| `balance` | `decimal(10,2)` |  | `0.00` |  |  |
| `settings` | `json` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- index `customers_code_index` (`code`)
- **unique** `customers_code_unique` (`code`)
- index `customers_organization_id_is_active_index` (`organization_id`, `is_active`)

</details>

### `credit_sales`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` |  |  | IX | → `branches.id` (on delete cascade) |
| `customer_id` | `bigint unsigned` |  |  | IX | → `customers.id` (on delete restrict) |
| `created_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `approved_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `credit_sale_number` | `varchar(191)` |  |  |  |  |
| `status` | `enum('pending','confirmed','partial','paid','overdue','cancelled')` |  | `pending` |  |  |
| `reason` | `text` | ✓ |  |  |  |
| `subtotal` | `decimal(12,2)` |  | `0.00` |  |  |
| `tax_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `discount_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `total_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `paid_amount` | `decimal(12,2)` |  | `0.00` |  |  |
| `tax_rate` | `decimal(5,2)` |  | `0.00` |  |  |
| `issued_date` | `date` |  |  |  |  |
| `due_date` | `date` | ✓ |  | IX |  |
| `confirmed_at` | `timestamp` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (8)</summary>

- index `credit_sales_approved_by_foreign` (`approved_by`)
- index `credit_sales_branch_id_foreign` (`branch_id`)
- index `credit_sales_created_by_foreign` (`created_by`)
- index `credit_sales_customer_id_status_index` (`customer_id`, `status`)
- index `credit_sales_due_date_index` (`due_date`)
- **unique** `credit_sales_org_credit_sale_number_unique` (`organization_id`, `credit_sale_number`)
- index `credit_sales_org_status_duedate_index` (`organization_id`, `status`, `due_date`)
- index `credit_sales_organization_id_branch_id_status_index` (`organization_id`, `branch_id`, `status`)

</details>

### `credit_sale_items`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `credit_sale_id` | `bigint unsigned` |  |  | IX | → `credit_sales.id` (on delete cascade) |
| `item_id` | `bigint unsigned` | ✓ |  | IX | → `items.id` (on delete set null) |
| `item_name` | `varchar(191)` |  |  |  |  |
| `item_sku` | `varchar(191)` | ✓ |  |  |  |
| `quantity` | `decimal(10,2)` |  |  |  |  |
| `unit_price` | `decimal(12,2)` |  |  |  |  |
| `discount_percentage` | `decimal(5,2)` |  | `0.00` |  |  |
| `tax_rate` | `decimal(5,2)` |  | `0.00` |  |  |
| `line_total` | `decimal(12,2)` |  |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `credit_sale_items_credit_sale_id_index` (`credit_sale_id`)
- index `credit_sale_items_item_id_foreign` (`item_id`)

</details>

### `credit_sale_payments`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `idempotency_key` | `varchar(64)` | ✓ |  |  |  |
| `credit_sale_id` | `bigint unsigned` |  |  | IX | → `credit_sales.id` (on delete cascade) |
| `created_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `amount` | `decimal(12,2)` |  |  |  |  |
| `payment_method` | `enum('cash','telebirr','bank_transfer','cheque')` |  | `cash` |  |  |
| `bank_id` | `bigint unsigned` | ✓ |  | IX | → `banks.id` (on delete set null) |
| `bank_name` | `varchar(255)` | ✓ |  |  |  |
| `payment_date` | `date` |  |  |  |  |
| `reference` | `varchar(191)` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (4)</summary>

- index `credit_sale_payments_bank_id_foreign` (`bank_id`)
- index `credit_sale_payments_created_by_foreign` (`created_by`)
- index `credit_sale_payments_credit_sale_id_index` (`credit_sale_id`)
- **unique** `credit_sale_payments_idem_unique` (`credit_sale_id`, `idempotency_key`)

</details>

## Finance & Treasury

Bank and mobile-wallet accounts with a transaction ledger, payment methods mapped to settlement accounts, inter-account transfers, expenses (one-off and recurring) and loans.

### `banks`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `account_type` | `enum('bank','cash','wallet')` |  | `bank` |  |  |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete set null) |
| `name` | `varchar(191)` |  |  |  |  |
| `code` | `varchar(191)` | ✓ |  |  |  |
| `account_number` | `text` | ✓ |  |  |  |
| `account_name` | `varchar(191)` | ✓ |  |  |  |
| `branch_name` | `varchar(191)` | ✓ |  |  |  |
| `swift_code` | `text` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `opening_balance` | `decimal(14,2)` |  | `0.00` |  |  |
| `current_balance` | `decimal(14,2)` |  | `0.00` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- **unique** `banks_branch_cash_unique` (`branch_id`, `account_type`)
- index `banks_org_account_type_index` (`organization_id`, `account_type`)
- index `banks_organization_id_is_active_index` (`organization_id`, `is_active`)

</details>

### `bank_transactions`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `idempotency_key` | `varchar(64)` | ✓ |  |  |  |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `bank_id` | `bigint unsigned` |  |  | IX | → `banks.id` (on delete cascade) |
| `transaction_type` | `enum('credit','debit')` |  |  |  |  |
| `source_type` | `varchar(191)` |  |  |  |  |
| `source_id` | `bigint unsigned` | ✓ |  |  |  |
| `payment_id` | `bigint unsigned` | ✓ |  | IX | → `payments.id` (on delete set null) |
| `amount` | `decimal(12,2)` |  |  |  |  |
| `balance_before` | `decimal(14,2)` |  |  |  |  |
| `balance_after` | `decimal(14,2)` |  |  |  |  |
| `description` | `varchar(191)` | ✓ |  |  |  |
| `reference_number` | `varchar(100)` | ✓ |  |  |  |
| `recorded_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `transaction_date` | `date` |  |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (6)</summary>

- index `bank_transactions_bank_id_payment_id_index` (`bank_id`, `payment_id`)
- index `bank_transactions_bank_id_transaction_date_index` (`bank_id`, `transaction_date`)
- **unique** `bank_transactions_org_idem_unique` (`organization_id`, `idempotency_key`)
- index `bank_transactions_organization_id_transaction_date_index` (`organization_id`, `transaction_date`)
- index `bank_transactions_payment_id_foreign` (`payment_id`)
- index `bank_transactions_recorded_by_foreign` (`recorded_by`)

</details>

### `payment_methods`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `code` | `varchar(191)` |  |  |  |  |
| `name` | `varchar(191)` |  |  |  |  |
| `type` | `varchar(191)` |  |  |  | cash, card, bank_transfer, digital_wallet, etc. |
| `default_bank_id` | `bigint unsigned` | ✓ |  | IX | → `banks.id` (on delete set null) |
| `description` | `text` | ✓ |  |  |  |
| `settings` | `json` | ✓ |  |  |  |
| `requires_authorization` | `tinyint(1)` |  | `0` |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- index `payment_methods_default_bank_id_foreign` (`default_bank_id`)
- **unique** `payment_methods_org_code_unique` (`organization_id`, `code`)
- index `payment_methods_organization_id_is_active_index` (`organization_id`, `is_active`)

</details>

### `payments`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `idempotency_key` | `varchar(64)` | ✓ |  |  |  |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` |  |  | IX | → `branches.id` (on delete cascade) |
| `payable_type` | `varchar(191)` |  |  | IX |  |
| `payable_id` | `bigint unsigned` |  |  |  |  |
| `payment_method_id` | `bigint unsigned` |  |  | IX | → `payment_methods.id` (on delete restrict) |
| `bank_id` | `bigint unsigned` | ✓ |  | IX | → `banks.id` (on delete set null) |
| `user_id` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `reference_number` | `varchar(191)` | ✓ |  |  |  |
| `amount` | `decimal(12,2)` |  |  |  |  |
| `payment_date` | `date` |  |  |  |  |
| `status` | `enum('pending','completed','failed','refunded')` |  | `pending` |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `metadata` | `json` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (8)</summary>

- index `payments_bank_id_foreign` (`bank_id`)
- index `payments_branch_id_foreign` (`branch_id`)
- **unique** `payments_org_idem_unique` (`organization_id`, `idempotency_key`)
- index `payments_organization_id_branch_id_payment_date_index` (`organization_id`, `branch_id`, `payment_date`)
- index `payments_payable_status_date_index` (`payable_type`, `status`, `payment_date`)
- index `payments_payable_type_payable_id_index` (`payable_type`, `payable_id`)
- index `payments_payment_method_id_foreign` (`payment_method_id`)
- index `payments_user_id_foreign` (`user_id`)

</details>

### `payment_transfers`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `idempotency_key` | `varchar(64)` | ✓ |  |  |  |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `from_bank_id` | `bigint unsigned` |  |  | IX | → `banks.id` (on delete restrict) |
| `to_bank_id` | `bigint unsigned` |  |  | IX | → `banks.id` (on delete restrict) |
| `amount` | `decimal(14,2)` |  |  |  |  |
| `reference_number` | `varchar(100)` |  |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `transferred_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `status` | `enum('pending','approved','rejected')` |  | `pending` |  |  |
| `approved_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `approved_at` | `timestamp` | ✓ |  |  |  |
| `rejection_reason` | `text` | ✓ |  |  |  |
| `transaction_date` | `date` |  |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (7)</summary>

- index `payment_transfers_approved_by_foreign` (`approved_by`)
- index `payment_transfers_from_bank_id_foreign` (`from_bank_id`)
- **unique** `payment_transfers_org_idem_unique` (`organization_id`, `idempotency_key`)
- **unique** `payment_transfers_org_reference_number_unique` (`organization_id`, `reference_number`)
- index `payment_transfers_organization_id_transaction_date_index` (`organization_id`, `transaction_date`)
- index `payment_transfers_to_bank_id_foreign` (`to_bank_id`)
- index `payment_transfers_transferred_by_foreign` (`transferred_by`)

</details>

### `expenses`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` |  |  | IX | → `branches.id` (on delete cascade) |
| `expense_category_id` | `bigint unsigned` |  |  | IX | → `expense_categories.id` (on delete restrict) |
| `created_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `approved_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `expense_number` | `varchar(191)` |  |  |  |  |
| `title` | `varchar(191)` |  |  |  |  |
| `description` | `text` | ✓ |  |  |  |
| `amount` | `decimal(12,2)` |  |  |  |  |
| `expense_date` | `date` |  |  | IX |  |
| `status` | `enum('pending','approved','rejected')` |  | `pending` |  |  |
| `reason` | `text` | ✓ |  |  |  |
| `payment_method` | `enum('cash','telebirr','bank_transfer','cheque')` |  | `cash` |  |  |
| `bank_id` | `bigint unsigned` | ✓ |  | IX | → `banks.id` (on delete set null) |
| `bank_name` | `varchar(191)` | ✓ |  |  |  |
| `reference` | `varchar(191)` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `approved_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (9)</summary>

- index `expenses_approved_by_foreign` (`approved_by`)
- index `expenses_bank_id_foreign` (`bank_id`)
- index `expenses_branch_id_foreign` (`branch_id`)
- index `expenses_created_by_foreign` (`created_by`)
- index `expenses_expense_category_id_foreign` (`expense_category_id`)
- index `expenses_expense_date_index` (`expense_date`)
- index `expenses_org_branch_status_date_index` (`organization_id`, `branch_id`, `status`, `expense_date`)
- **unique** `expenses_org_expense_number_unique` (`organization_id`, `expense_number`)
- index `expenses_organization_id_branch_id_status_index` (`organization_id`, `branch_id`, `status`)

</details>

### `expense_categories`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `name` | `varchar(191)` |  |  |  |  |
| `description` | `varchar(191)` | ✓ |  |  |  |
| `color` | `varchar(191)` |  | `#6366f1` |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (1)</summary>

- index `expense_categories_organization_id_is_active_index` (`organization_id`, `is_active`)

</details>

### `fixed_expenses`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete cascade) |
| `expense_category_id` | `bigint unsigned` |  |  | IX | → `expense_categories.id` (on delete restrict) |
| `title` | `varchar(191)` |  |  |  |  |
| `description` | `text` | ✓ |  |  |  |
| `amount` | `decimal(14,2)` |  |  |  |  |
| `frequency` | `enum('one_time','weekly','monthly','quarterly','yearly')` |  |  |  |  |
| `due_date` | `date` |  |  | IX |  |
| `alert_days_before` | `int unsigned` |  | `3` |  |  |
| `payment_method` | `enum('cash','telebirr','bank_transfer','cheque')` |  | `cash` |  |  |
| `bank_id` | `bigint unsigned` | ✓ |  | IX | → `banks.id` (on delete set null) |
| `auto_generate_expense` | `tinyint(1)` |  | `1` |  |  |
| `status` | `enum('active','paused','cancelled')` |  | `active` |  |  |
| `last_generated_expense_id` | `bigint unsigned` | ✓ |  | IX | → `expenses.id` (on delete set null) |
| `created_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (7)</summary>

- index `fixed_expenses_bank_id_foreign` (`bank_id`)
- index `fixed_expenses_branch_id_foreign` (`branch_id`)
- index `fixed_expenses_created_by_foreign` (`created_by`)
- index `fixed_expenses_due_date_status_index` (`due_date`, `status`)
- index `fixed_expenses_expense_category_id_foreign` (`expense_category_id`)
- index `fixed_expenses_last_generated_expense_id_foreign` (`last_generated_expense_id`)
- index `fixed_expenses_organization_id_status_index` (`organization_id`, `status`)

</details>

### `loans`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete cascade) |
| `direction` | `enum('payable','receivable')` |  |  |  |  |
| `counterparty_type` | `enum('bank','person','company')` |  |  |  |  |
| `counterparty_name` | `varchar(191)` |  |  |  |  |
| `counterparty_contact` | `varchar(191)` | ✓ |  |  |  |
| `counterparty_address` | `varchar(191)` | ✓ |  |  |  |
| `loan_number` | `varchar(191)` |  |  |  |  |
| `principal_amount` | `decimal(14,2)` |  |  |  |  |
| `interest_rate` | `decimal(5,2)` | ✓ |  |  |  |
| `start_date` | `date` |  |  |  |  |
| `due_date` | `date` |  |  |  |  |
| `repayment_frequency` | `enum('one_time','weekly','monthly','quarterly','custom')` |  | `monthly` |  |  |
| `outstanding_balance` | `decimal(14,2)` |  | `0.00` |  |  |
| `status` | `enum('pending','active','paid','defaulted','cancelled','rejected')` |  | `pending` |  |  |
| `approved_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `approved_at` | `timestamp` | ✓ |  |  |  |
| `rejection_reason` | `text` | ✓ |  |  |  |
| `agreement_terms` | `text` | ✓ |  |  |  |
| `agreement_document` | `varchar(191)` | ✓ |  |  |  |
| `bank_id` | `bigint unsigned` | ✓ |  | IX | → `banks.id` (on delete set null) |
| `alert_days_before` | `int unsigned` |  | `3` |  |  |
| `created_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (7)</summary>

- index `loans_approved_by_foreign` (`approved_by`)
- index `loans_bank_id_foreign` (`bank_id`)
- index `loans_branch_id_foreign` (`branch_id`)
- index `loans_created_by_foreign` (`created_by`)
- **unique** `loans_org_loan_number_unique` (`organization_id`, `loan_number`)
- index `loans_organization_id_branch_id_index` (`organization_id`, `branch_id`)
- index `loans_organization_id_direction_status_index` (`organization_id`, `direction`, `status`)

</details>

### `loan_installments`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `loan_id` | `bigint unsigned` |  |  | IX | → `loans.id` (on delete cascade) |
| `installment_number` | `int unsigned` |  |  |  |  |
| `due_date` | `date` |  |  | IX |  |
| `amount_due` | `decimal(14,2)` |  |  |  |  |
| `amount_paid` | `decimal(14,2)` |  | `0.00` |  |  |
| `status` | `enum('pending','partial','paid','overdue')` |  | `pending` |  |  |
| `paid_at` | `timestamp` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `loan_installments_due_date_status_index` (`due_date`, `status`)
- index `loan_installments_loan_id_status_index` (`loan_id`, `status`)

</details>

### `loan_authorized_users`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `loan_id` | `bigint unsigned` |  |  | IX | → `loans.id` (on delete cascade) |
| `user_id` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `granted_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `granted_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- index `loan_authorized_users_granted_by_foreign` (`granted_by`)
- **unique** `loan_authorized_users_loan_id_user_id_unique` (`loan_id`, `user_id`)
- index `loan_authorized_users_user_id_foreign` (`user_id`)

</details>

### `personal_categories`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `name` | `varchar(191)` |  |  |  |  |
| `type` | `enum('income','expense')` |  |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (1)</summary>

- index `personal_categories_organization_id_type_is_active_index` (`organization_id`, `type`, `is_active`)

</details>

### `personal_transactions`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `bank_id` | `bigint unsigned` |  |  | IX | → `banks.id` (on delete cascade) |
| `personal_category_id` | `bigint unsigned` |  |  | IX | → `personal_categories.id` (on delete restrict) |
| `created_by` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `approved_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `type` | `enum('income','expense')` |  |  |  |  |
| `transaction_number` | `varchar(191)` |  |  |  |  |
| `title` | `varchar(191)` |  |  |  |  |
| `description` | `text` | ✓ |  |  |  |
| `amount` | `decimal(12,2)` |  |  |  |  |
| `transaction_date` | `date` |  |  | IX |  |
| `status` | `enum('pending','approved','rejected')` |  | `pending` |  |  |
| `reference` | `varchar(191)` | ✓ |  |  |  |
| `reason` | `text` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `approved_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (8)</summary>

- index `personal_transactions_approved_by_foreign` (`approved_by`)
- index `personal_transactions_bank_id_foreign` (`bank_id`)
- index `personal_transactions_created_by_foreign` (`created_by`)
- **unique** `personal_transactions_org_transaction_number_unique` (`organization_id`, `transaction_number`)
- index `personal_transactions_organization_id_status_index` (`organization_id`, `status`)
- index `personal_transactions_organization_id_type_index` (`organization_id`, `type`)
- index `personal_transactions_personal_category_id_foreign` (`personal_category_id`)
- index `personal_transactions_transaction_date_index` (`transaction_date`)

</details>

### `personal_bank_transactions`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `bank_id` | `bigint unsigned` |  |  | IX | → `banks.id` (on delete cascade) |
| `transaction_type` | `enum('credit','debit')` |  |  |  |  |
| `source_type` | `varchar(191)` |  |  |  |  |
| `source_id` | `bigint unsigned` | ✓ |  |  |  |
| `amount` | `decimal(12,2)` |  |  |  |  |
| `balance_before` | `decimal(14,2)` |  |  |  |  |
| `balance_after` | `decimal(14,2)` |  |  |  |  |
| `description` | `varchar(191)` | ✓ |  |  |  |
| `reference_number` | `varchar(100)` | ✓ |  |  |  |
| `recorded_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `transaction_date` | `date` |  |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- index `personal_bank_transactions_recorded_by_foreign` (`recorded_by`)
- index `personal_bank_tx_bank_date_idx` (`bank_id`, `transaction_date`)
- index `personal_bank_tx_org_date_idx` (`organization_id`, `transaction_date`)

</details>

## Human Resources

Employees and payroll runs with per-employee payroll lines.

### `employees`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete cascade) |
| `user_id` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `employee_number` | `varchar(191)` |  |  |  |  |
| `first_name` | `varchar(191)` |  |  |  |  |
| `last_name` | `varchar(191)` |  |  |  |  |
| `email` | `varchar(191)` | ✓ |  |  |  |
| `phone` | `varchar(191)` | ✓ |  |  |  |
| `address` | `varchar(191)` | ✓ |  |  |  |
| `date_of_birth` | `date` | ✓ |  |  |  |
| `gender` | `varchar(191)` | ✓ |  |  |  |
| `position` | `varchar(191)` | ✓ |  |  |  |
| `department` | `varchar(191)` | ✓ |  |  |  |
| `employee_type` | `enum('full_time','part_time','contract','daily')` |  | `full_time` |  |  |
| `hire_date` | `date` | ✓ |  |  |  |
| `termination_date` | `date` | ✓ |  |  |  |
| `status` | `enum('active','inactive','terminated')` |  | `active` |  |  |
| `basic_salary` | `text` | ✓ |  |  |  |
| `payment_method` | `enum('cash','telebirr','bank_transfer','cheque')` |  | `bank_transfer` |  |  |
| `bank_id` | `bigint unsigned` | ✓ |  | IX | → `banks.id` (on delete set null) |
| `created_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (7)</summary>

- index `employees_bank_id_foreign` (`bank_id`)
- index `employees_branch_id_foreign` (`branch_id`)
- index `employees_created_by_foreign` (`created_by`)
- **unique** `employees_org_employee_number_unique` (`organization_id`, `employee_number`)
- index `employees_organization_id_branch_id_index` (`organization_id`, `branch_id`)
- index `employees_organization_id_status_index` (`organization_id`, `status`)
- index `employees_user_id_foreign` (`user_id`)

</details>

### `payroll_runs`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete cascade) |
| `run_number` | `varchar(191)` |  |  |  |  |
| `period_start` | `date` |  |  |  |  |
| `period_end` | `date` |  |  |  |  |
| `pay_date` | `date` |  |  |  |  |
| `status` | `enum('draft','approved','paid','cancelled')` |  | `draft` |  |  |
| `total_gross` | `decimal(14,2)` |  | `0.00` |  |  |
| `total_deductions` | `decimal(14,2)` |  | `0.00` |  |  |
| `total_net` | `decimal(14,2)` |  | `0.00` |  |  |
| `created_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `approved_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `approved_at` | `timestamp` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (5)</summary>

- index `payroll_runs_approved_by_foreign` (`approved_by`)
- index `payroll_runs_branch_id_foreign` (`branch_id`)
- index `payroll_runs_created_by_foreign` (`created_by`)
- **unique** `payroll_runs_org_run_number_unique` (`organization_id`, `run_number`)
- index `payroll_runs_organization_id_status_index` (`organization_id`, `status`)

</details>

### `payroll_items`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `payroll_run_id` | `bigint unsigned` |  |  | IX | → `payroll_runs.id` (on delete cascade) |
| `employee_id` | `bigint unsigned` |  |  | IX | → `employees.id` (on delete restrict) |
| `basic_salary` | `decimal(14,2)` |  | `0.00` |  |  |
| `allowances` | `decimal(14,2)` |  | `0.00` |  |  |
| `bonus` | `decimal(14,2)` |  | `0.00` |  |  |
| `deductions` | `decimal(14,2)` |  | `0.00` |  |  |
| `gross_pay` | `decimal(14,2)` |  | `0.00` |  |  |
| `net_pay` | `decimal(14,2)` |  | `0.00` |  |  |
| `payment_method` | `enum('cash','telebirr','bank_transfer','cheque')` |  | `bank_transfer` |  |  |
| `bank_id` | `bigint unsigned` | ✓ |  | IX | → `banks.id` (on delete set null) |
| `status` | `enum('pending','paid')` |  | `pending` |  |  |
| `paid_at` | `timestamp` | ✓ |  |  |  |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (3)</summary>

- index `payroll_items_bank_id_foreign` (`bank_id`)
- index `payroll_items_employee_id_index` (`employee_id`)
- index `payroll_items_payroll_run_id_status_index` (`payroll_run_id`, `status`)

</details>

## Fixed Assets

Asset register with categories and unique serial numbers.

### `assets`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `branch_id` | `bigint unsigned` | ✓ |  | IX | → `branches.id` (on delete cascade) |
| `asset_category_id` | `bigint unsigned` | ✓ |  | IX | → `asset_categories.id` (on delete set null) |
| `asset_number` | `varchar(191)` |  |  |  |  |
| `name` | `varchar(191)` |  |  |  |  |
| `description` | `text` | ✓ |  |  |  |
| `serial_number` | `varchar(191)` | ✓ |  |  |  |
| `purchase_date` | `date` | ✓ |  |  |  |
| `purchase_cost` | `decimal(14,2)` |  | `0.00` |  |  |
| `current_value` | `decimal(14,2)` |  | `0.00` |  |  |
| `condition` | `enum('new','good','fair','poor','damaged')` |  | `good` |  |  |
| `status` | `enum('in_use','in_storage','under_repair','disposed')` |  | `in_use` |  |  |
| `location` | `varchar(191)` | ✓ |  |  |  |
| `assigned_to` | `bigint unsigned` | ✓ |  |  |  |
| `created_by` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `notes` | `text` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |
| `deleted_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (8)</summary>

- index `assets_asset_category_id_foreign` (`asset_category_id`)
- index `assets_branch_id_foreign` (`branch_id`)
- index `assets_created_by_foreign` (`created_by`)
- **unique** `assets_org_asset_number_unique` (`organization_id`, `asset_number`)
- **unique** `assets_org_serial_unique` (`organization_id`, `serial_number`)
- index `assets_organization_id_asset_category_id_index` (`organization_id`, `asset_category_id`)
- index `assets_organization_id_branch_id_index` (`organization_id`, `branch_id`)
- index `assets_organization_id_status_index` (`organization_id`, `status`)

</details>

### `asset_categories`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `name` | `varchar(191)` |  |  |  |  |
| `description` | `varchar(191)` | ✓ |  |  |  |
| `color` | `varchar(191)` | ✓ |  |  |  |
| `is_active` | `tinyint(1)` |  | `1` |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (1)</summary>

- index `asset_categories_organization_id_is_active_index` (`organization_id`, `is_active`)

</details>

## Platform & Operations

Audit trail, operational alerts, asynchronous report exports and controlled database restores.

### `audit_logs`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` | ✓ |  | IX | → `organizations.id` (on delete cascade) |
| `user_id` | `bigint unsigned` | ✓ |  | IX | → `users.id` (on delete set null) |
| `action` | `varchar(191)` |  |  | IX |  |
| `model_type` | `varchar(191)` | ✓ |  | IX |  |
| `model_id` | `bigint unsigned` | ✓ |  |  |  |
| `old_values` | `json` | ✓ |  |  |  |
| `new_values` | `json` | ✓ |  |  |  |
| `ip_address` | `varchar(191)` | ✓ |  |  |  |
| `user_agent` | `varchar(191)` | ✓ |  |  |  |
| `created_at` | `timestamp` |  |  |  |  |

<details><summary>Indexes (5)</summary>

- index `audit_logs_action_index` (`action`)
- index `audit_logs_model_type_model_id_index` (`model_type`, `model_id`)
- index `audit_logs_organization_id_created_at_index` (`organization_id`, `created_at`)
- index `audit_logs_user_created_index` (`user_id`, `created_at`)
- index `audit_logs_user_id_index` (`user_id`)

</details>

### `alert_logs`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX | → `organizations.id` (on delete cascade) |
| `alertable_type` | `varchar(191)` |  |  | IX |  |
| `alertable_id` | `bigint unsigned` |  |  |  |  |
| `channel` | `enum('email','whatsapp','in_app')` |  |  |  |  |
| `recipient_user_id` | `bigint unsigned` |  |  | IX | → `users.id` (on delete cascade) |
| `stage` | `enum('upcoming','due','overdue')` |  |  |  |  |
| `sent_at` | `timestamp` | ✓ |  |  |  |
| `status` | `enum('sent','failed','skipped')` |  | `sent` |  |  |
| `error_message` | `varchar(191)` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (4)</summary>

- index `alert_logs_alertable_type_alertable_id_index` (`alertable_type`, `alertable_id`)
- index `alert_logs_dedup_idx` (`alertable_type`, `alertable_id`, `stage`, `recipient_user_id`, `created_at`)
- index `alert_logs_organization_id_stage_index` (`organization_id`, `stage`)
- index `alert_logs_recipient_user_id_foreign` (`recipient_user_id`)

</details>

### `export_jobs`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `organization_id` | `bigint unsigned` |  |  | IX |  |
| `user_id` | `bigint unsigned` |  |  | IX |  |
| `label` | `varchar(191)` |  |  |  |  |
| `report_type` | `varchar(191)` |  |  |  |  |
| `format` | `varchar(10)` |  |  |  |  |
| `params` | `json` | ✓ |  |  |  |
| `status` | `varchar(20)` |  | `pending` |  |  |
| `file_path` | `varchar(191)` | ✓ |  |  |  |
| `error_message` | `text` | ✓ |  |  |  |
| `completed_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (2)</summary>

- index `export_jobs_organization_id_user_id_created_at_index` (`organization_id`, `user_id`, `created_at`)
- index `export_jobs_user_id_foreign` (`user_id`)

</details>

### `database_restores`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `user_id` | `bigint unsigned` |  |  | IX | → `users.id` (on delete restrict) |
| `backup_filename` | `varchar(191)` |  |  |  |  |
| `target_database` | `varchar(191)` |  |  |  |  |
| `status` | `enum('pending','running','success','failed')` |  | `pending` |  |  |
| `progress` | `tinyint unsigned` |  | `0` |  |  |
| `current_step` | `varchar(191)` | ✓ |  |  |  |
| `error_message` | `text` | ✓ |  |  |  |
| `started_at` | `timestamp` | ✓ |  |  |  |
| `finished_at` | `timestamp` | ✓ |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |
| `updated_at` | `timestamp` | ✓ |  |  |  |

<details><summary>Indexes (1)</summary>

- index `database_restores_user_id_index` (`user_id`)

</details>

## Framework

Laravel infrastructure tables (cache, sessions, queue failures, migrations, password resets).

### `cache`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `key` | `varchar(191)` |  |  | PK |  |
| `value` | `mediumtext` |  |  |  |  |
| `expiration` | `int` |  |  |  |  |

### `cache_locks`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `key` | `varchar(191)` |  |  | PK |  |
| `owner` | `varchar(191)` |  |  |  |  |
| `expiration` | `int` |  |  |  |  |

### `sessions`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `varchar(191)` |  |  | PK |  |
| `user_id` | `bigint unsigned` | ✓ |  | IX |  |
| `ip_address` | `varchar(45)` | ✓ |  |  |  |
| `user_agent` | `text` | ✓ |  |  |  |
| `payload` | `longtext` |  |  |  |  |
| `last_activity` | `int` |  |  | IX |  |

<details><summary>Indexes (2)</summary>

- index `sessions_last_activity_index` (`last_activity`)
- index `sessions_user_id_index` (`user_id`)

</details>

### `failed_jobs`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `bigint unsigned` |  |  | PK | auto_increment |
| `uuid` | `varchar(191)` |  |  | UQ |  |
| `connection` | `text` |  |  |  |  |
| `queue` | `text` |  |  |  |  |
| `payload` | `longtext` |  |  |  |  |
| `exception` | `longtext` |  |  |  |  |
| `failed_at` | `timestamp` |  | `CURRENT_TIMESTAMP` |  |  |

<details><summary>Indexes (1)</summary>

- **unique** `failed_jobs_uuid_unique` (`uuid`)

</details>

### `migrations`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `id` | `int unsigned` |  |  | PK | auto_increment |
| `migration` | `varchar(191)` |  |  |  |  |
| `batch` | `int` |  |  |  |  |

### `password_reset_tokens`

| Column | Type | Null | Default | Key | Notes |
|---|---|:-:|---|:-:|---|
| `email` | `varchar(191)` |  |  | PK |  |
| `token` | `varchar(191)` |  |  |  |  |
| `created_at` | `timestamp` | ✓ |  |  |  |

