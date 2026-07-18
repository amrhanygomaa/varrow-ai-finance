# VArrow AI Finance — Database Architecture & Schema Specification

| | |
|---|---|
| **Document** | Database Schema Specification |
| **Status** | Final |
| **Database** | PostgreSQL |
| **Architecture** | Modular Monolith (ADR-001) |
| **Date** | 2026-07-17 |

---

## 1. Database Principles

To ensure consistency, performance, and financial accuracy, the database adheres to the following principles:

- **PostgreSQL Best Practices:** Use native types (e.g., `JSONB`, `UUID`), enforce constraints at the DB level, and rely on ACID guarantees.
- **UUID Strategy:** Primary keys use `UUIDv4` (generated via `gen_random_uuid()`) to prevent enumeration attacks, simplify distributed ID generation, and ease future microservice extraction.
- **Naming Conventions:** `snake_case` for schemas, tables, columns, and indexes. Plural table names (e.g., `users`, `sales_invoices`).
- **Timezone Strategy:** All dates and timestamps are stored in UTC (`TIMESTAMPTZ`). Application layers handle localization.
- **Soft Delete Strategy:** A `deleted_at` timestamp is used for soft deletes. Financial records (invoices, payments, outbox events) are *never* permanently deleted automatically.
  - **Child Entity Handling**: To handle soft deletes of parent-child compositions (e.g. SalesInvoice and its SalesInvoiceItems), the system uses **Application-Layer Cascading**. When a service marks a parent entity as deleted (setting `deleted_at = NOW()`), the service MUST update `deleted_at = NOW()` on all child items in the same database transaction. Database-level `ON DELETE CASCADE` is reserved for physical deletes during historical archiving.
- **Audit Fields:** Every table tracks `created_at`, `updated_at`, `created_by`, and `updated_by` for strict traceability.
- **Versioning / Optimistic Locking:** A `version` integer column is incremented on every update to prevent concurrent update anomalies (Optimistic Concurrency Control).
- **Multi-language Support:** Translation-heavy text fields (e.g., product names, setting descriptions) use `JSONB` columns `{"en": "Name", "ar": "الاسم"}` or dedicated translation tables if querying by text is critical.
- **Financial Precision:** All monetary amounts use `NUMERIC(19, 4)` to prevent floating-point rounding errors and accommodate fractional tax calculations.
- **Money Handling:** Currency is always explicitly stored alongside amounts (e.g., `amount NUMERIC`, `currency_code CHAR(3)`).
- **Indexing Strategy:** Foreign keys, frequent search terms, and `deleted_at` are indexed.
- **Schema Segregation:** Tables are grouped into PostgreSQL schemas mapping to Bounded Contexts (e.g., `identity.users`, `catalog.products`, `finance.invoices`) enforcing boundary discipline (ADR-004).

---

## 2. Base Entity

All domain entities inherit the following core columns to ensure uniform auditing and concurrency control.

| Column | Data Type | Nullable | Default | Purpose |
|---|---|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | Universally unique primary key. *Exception: Partitioned tables (`audit_logs`, `outbox_events`, `petty_cash_ledger`) use a composite primary key `(id, created_at)` to satisfy PostgreSQL partitioning requirements.* |
| `created_at` | `TIMESTAMPTZ` | No | `NOW()` | Audit: When the record was created. Used as the partition key for high-volume logs. |
| `updated_at` | `TIMESTAMPTZ` | No | `NOW()` | Audit: When the record was last modified. |
| `deleted_at` | `TIMESTAMPTZ` | Yes | `NULL` | Soft delete marker. If not null, record is considered deleted. |
| `created_by` | `UUID` | Yes | `NULL` | Audit: UUID reference to `identity.users.id` (no physical cross-schema foreign key). Nullable for system-generated records. |
| `updated_by` | `UUID` | Yes | `NULL` | Audit: UUID reference to `identity.users.id`. |
| `version` | `INTEGER` | No | `1` | Optimistic locking. Must be incremented on every update. |

---

## 3. Entity Design

### 3.1 Identity Schema (`identity`)

#### **Users**
- **Purpose:** Central authentication and identity directory.
- **Columns:** `id`, `email` (VARCHAR 255), `password_hash` (VARCHAR 255), `first_name` (VARCHAR 100), `last_name` (VARCHAR 100), `status` (VARCHAR 20).
- **Constraints/Keys:** `UNIQUE(email)`. PK `id`.
- **Business Rules:** `status` IN ('ACTIVE', 'INACTIVE', 'LOCKED').
- **Soft Delete:** Yes.
- **Row Growth:** Low (bounded by employee count).

#### **Roles**
- **Purpose:** Defines security access profiles.
- **Columns:** `id`, `name` (VARCHAR 50), `description` (TEXT), `is_system` (BOOLEAN).
- **Constraints/Keys:** `UNIQUE(name)`.
- **Business Rules:** System roles cannot be modified/deleted.

#### **Permissions**
- **Purpose:** Maps specific module capabilities to roles.
- **Columns:** `role_id` (UUID), `resource` (VARCHAR 50), `action` (VARCHAR 50).
- **Constraints/Keys:** PK `(role_id, resource, action)`. FK `role_id` -> `roles.id` (CASCADE).
- **Business Rules:** Explicit grant list (deny-by-default).

*(Note: User-Role assignment is a many-to-many join table: `user_roles(user_id, role_id)`).*

#### **Refresh Tokens**
- **Purpose:** Securely tracks user refresh tokens for session rotation and reuse detection (ADR-002).
- **Columns:** `id` (UUID PK), `user_id` (UUID FK -> `users.id` CASCADE), `token_hash` (VARCHAR 64 - SHA-256 of the token), `token_family` (UUID), `is_revoked` (BOOLEAN), `expires_at` (TIMESTAMPTZ), `created_at` (TIMESTAMPTZ).
- **Constraints/Keys:** `UNIQUE(token_hash)`.
- **Business Rules:** 
  - Token hashes must be unique.
  - Refresh tokens are stored **ONLY** as SHA-256 hashes.
  - Revoking a family sets `is_revoked = true` for all tokens sharing `token_family`.

---

### 3.2 Master Data / Catalog Schema (`catalog`) (per ADR-004)

#### **Products**
- **Purpose:** Authoritative record of items bought/sold.
- **Columns:** `id`, `sku` (VARCHAR 50), `name` (JSONB for EN/AR), `description` (JSONB), `base_price` (NUMERIC(19,4)), `is_active` (BOOLEAN).
- **Constraints/Keys:** `UNIQUE(sku)`.
- **Business Rules:** Only Catalog module writes. Sales/Procurement read via synced views.

#### **Services**
- **Purpose:** Authoritative record of services provided/procured.
- **Columns:** `id`, `code` (VARCHAR 50), `name` (JSONB), `billing_type` (VARCHAR 20 - HOURLY/FIXED).
- **Constraints/Keys:** `UNIQUE(code)`.

#### **Contacts**
- **Purpose:** Shared registry of individuals representing Customers/Vendors.
- **Columns:** `id`, `first_name`, `last_name`, `email`, `phone`, `job_title`.
- **Business Rules:** Independent lifecycle from Companies.

#### **Customers** & **Vendors** (Trading Partners)
- **Purpose:** Shared registry of organizations.
- **Columns:** `id`, `type` (CUSTOMER, VENDOR, BOTH), `name` (VARCHAR 255), `tax_number` (VARCHAR 50), `billing_currency` (CHAR(3)), `status`.
- **Constraints/Keys:** `UNIQUE(tax_number)`.

---

### 3.3 Sales Schema (`sales`)

#### **Quotations**
- **Purpose:** Price offers sent to customers.
- **Columns:** `id`, `quotation_number` (VARCHAR 50), `customer_id` (UUID), `total_amount` (NUMERIC(19,4)), `currency_code` (CHAR(3)), `status` (VARCHAR 20), `valid_until` (DATE).
- **Constraints/Keys:** `UNIQUE(quotation_number)`. FK `customer_id` -> `catalog.customers`.
- **Business Rules:** Statuses: DRAFT, SENT, ACCEPTED, REJECTED, EXPIRED.

#### **Quotation Items**
- **Purpose:** Line items on a quotation.
- **Columns:** `id`, `quotation_id` (UUID), `product_id` (UUID), `service_id` (UUID), `quantity` (NUMERIC(19,4)), `unit_price` (NUMERIC(19,4)), `total` (NUMERIC(19,4)).
- **Constraints/Keys:** FK `quotation_id` -> `quotations` (CASCADE).
- **Business Rules:** One of `product_id` or `service_id` must be set. Holds immutable price snapshot.

#### **Sales Invoices**
- **Purpose:** Billing record for revenue collection.
- **Columns:** `id`, `invoice_number` (VARCHAR 50), `customer_id` (UUID), `quotation_id` (UUID, null), `subtotal` (NUMERIC), `tax_total` (NUMERIC), `grand_total` (NUMERIC), `currency_code`, `status`, `due_date`.
- **Constraints/Keys:** `UNIQUE(invoice_number)`.
- **Business Rules:** Immutable once ISSUED.

#### **Sales Invoice Items**
- **Purpose:** Invoice line items.
- **Columns:** `id`, `invoice_id`, `product_id`, `service_id`, `description`, `quantity`, `unit_price`, `tax_rate`, `tax_amount`, `total`.
- **Constraints/Keys:** FK `invoice_id` -> `sales_invoices` (CASCADE).

#### **Receipts**
- **Purpose:** Customer payments received.
- **Columns:** `id`, `receipt_number`, `customer_id`, `invoice_id`, `amount`, `currency_code`, `payment_method`, `reference_number`, `receipt_date`.
- **Business Rules:** `amount` cannot exceed remaining invoice balance.

---

### 3.4 Procurement Schema (`procurement`)

#### **Purchase Orders** (POs)
- **Purpose:** Formal request to purchase from a vendor.
- **Columns:** `id`, `po_number`, `vendor_id`, `total_amount`, `currency_code`, `status`, `expected_delivery_date`.
- **Constraints/Keys:** `UNIQUE(po_number)`. FK `vendor_id` -> `catalog.vendors`.

#### **Purchase Order Items**
- **Purpose:** Line items on a PO.
- **Columns:** `id`, `po_id`, `product_id`, `service_id`, `quantity`, `unit_price`, `total`.
- **Constraints/Keys:** FK `po_id` -> `purchase_orders` (CASCADE).

---

### 3.5 Finance Schema (`finance`)

#### **Purchase Invoices**
- **Purpose:** Vendor bills to be paid.
- **Columns:** `id`, `vendor_invoice_number` (VARCHAR 50), `vendor_id` (UUID - cross-context), `po_id` (UUID - cross-context), `subtotal` (NUMERIC(19,4)), `tax_total` (NUMERIC(19,4)), `grand_total` (NUMERIC(19,4)), `currency_code` (CHAR(3)), `status` (VARCHAR 20), `due_date` (DATE).
- **Constraints/Keys:** `CHECK (subtotal >= 0)`, `CHECK (tax_total >= 0)`, `CHECK (grand_total >= 0)`.
- **Business Rules:** Must be approved via Workflow Engine before payment.

#### **Purchase Invoice Items**
- **Purpose:** Vendor bill line items.
- **Columns:** `id`, `invoice_id` (UUID FK -> `purchase_invoices.id` CASCADE), `description` (TEXT), `quantity` (NUMERIC(19,4)), `unit_price` (NUMERIC(19,4)), `total` (NUMERIC(19,4)).
- **Constraints/Keys:** `CHECK (quantity > 0)`, `CHECK (unit_price >= 0)`, `CHECK (total >= 0)`.

#### **Payments**
- **Purpose:** Outgoing money transfers to vendors.
- **Columns:** `id`, `payment_number` (VARCHAR 50), `vendor_id` (UUID - cross-context), `invoice_id` (UUID FK -> `purchase_invoices.id` RESTRICT), `amount` (NUMERIC(19,4)), `currency_code` (CHAR(3)), `payment_method` (VARCHAR 20), `reference_number` (VARCHAR 50), `payment_date` (TIMESTAMPTZ).
- **Constraints/Keys:** `CHECK (amount > 0)`.

#### **Expenses** & **Expense Categories**
- **Purpose:** Direct company spending not tied to POs.
- **Columns (Expenses):** `id`, `category_id` (UUID FK -> `expense_categories.id` RESTRICT), `amount` (NUMERIC(19,4)), `currency_code` (CHAR(3)), `date` (DATE), `description` (TEXT), `receipt_attachment_id` (UUID - cross-context).
- **Columns (Categories):** `id`, `name` (VARCHAR 100), `gl_account_code` (VARCHAR 50).
- **Constraints/Keys:** `CHECK (amount > 0)`.

#### **Employee Expenses**
- **Purpose:** Staff reimbursement claims.
- **Columns:** `id`, `employee_id` (UUID - loose cross-context reference to `identity.users.id`), `total_amount` (NUMERIC(19,4)), `currency_code` (CHAR(3)), `status` (VARCHAR 20), `submitted_date` (TIMESTAMPTZ).
- **Constraints/Keys:** `CHECK (total_amount >= 0)`.
- **Business Rules:** Enforces application-layer cascading soft delete to `employee_expense_items`.

#### **Employee Expense Items**
- **Purpose:** Itemized receipts inside an employee expense claim.
- **Columns:** `id`, `employee_expense_id` (UUID FK -> `employee_expenses.id` CASCADE), `category_id` (UUID FK -> `expense_categories.id` RESTRICT), `amount` (NUMERIC(19,4)), `currency_code` (CHAR(3)), `description` (TEXT), `receipt_attachment_id` (UUID - loose cross-context reference to `document.attachments.id`).
- **Constraints/Keys:** `CHECK (amount >= 0)`.

#### **Petty Cash**
- **Purpose:** Small cash on hand tracking.
- **Columns:** `id`, `custodian_id` (UUID - loose cross-context reference to `identity.users.id`), `balance` (NUMERIC(19,4)), `currency_code` (CHAR(3)), `location` (VARCHAR 100).
- **Constraints/Keys:** `CHECK (balance >= 0)`.
- **Business Rules:** Transactions tracked via append-only `petty_cash_ledger`.

#### **Petty Cash Ledger**
- **Purpose:** Append-only ledger tracking all disbursements, deposits, and reconciliations for petty cash.
- **Columns:** `id` (UUID), `petty_cash_id` (UUID FK -> `petty_cash.id` RESTRICT), `amount` (NUMERIC(19,4)), `type` (VARCHAR 20 - DEPOSIT, WITHDRAWAL, RECONCILIATION_DISCREPANCY), `description` (TEXT), `reference_number` (VARCHAR 50), `created_at` (TIMESTAMPTZ), `created_by` (UUID).
- **Constraints/Keys:** Composite primary key `(id, created_at)` for monthly range partitioning. `CHECK (amount >= 0)`.
- **Business Rules:**
  - Strictly append-only. Triggers prevent updates and deletes.
  - Partitioned monthly by `created_at`.

#### **Cash Flow**
- **Purpose:** Daily aggregation of receipts and payments.
- **Columns:** `date`, `inflow`, `outflow`, `net`, `currency_code`.
- **Business Rules:** Implemented as a Materialized View for reporting, querying across `sales.receipts` and `finance.payments` schemas (documented cross-context dependency exception).

#### **Currencies** & **Exchange Rates**
- **Purpose:** Multi-currency support and conversion.
- **Columns (Rates):** `id`, `base_currency` (CHAR(3)), `target_currency` (CHAR(3)), `rate` (NUMERIC(19,6)), `effective_date` (DATE).
- **Constraints/Keys:** `CHECK (rate > 0)`.

#### **Taxes**
- **Purpose:** Tax/VAT configuration rules.
- **Columns:** `id`, `name` (VARCHAR 50), `rate` (NUMERIC(5,4)), `is_active` (BOOLEAN).
- **Constraints/Keys:** `CHECK (rate >= 0 AND rate <= 1.0)`.

---

### 3.6 Workflow Schema (`workflow`)

#### **Approval Workflows**
- **Purpose:** Defines routing rules for document approvals.
- **Columns:** `id`, `document_type` (VARCHAR), `min_amount`, `max_amount`, `currency_code`.

#### **Approval Steps**
- **Purpose:** Sequential approvers for a workflow.
- **Columns:** `id`, `workflow_id`, `step_order` (INTEGER), `approver_role_id` (UUID), `approver_user_id` (UUID).
- **Business Rules:** Exactly one of `role_id` or `user_id` must be set.

---

### 3.7 Document Schema (`document`)

#### **Documents** & **Attachments**
- **Purpose:** File metadata storage (S3 references).
- **Columns:** `id`, `file_name`, `mime_type`, `size_bytes`, `s3_object_key`, `module_reference` (e.g., 'PurchaseInvoice'), `record_id` (UUID).

#### **OCR Results**
- **Purpose:** Extracted data from physical documents.
- **Columns:** `id`, `attachment_id`, `extracted_data` (JSONB), `confidence_score` (NUMERIC), `status` (PENDING_REVIEW, VERIFIED).

---

### 3.8 Administration Schema (`administration`)

#### **Settings**
- **Purpose:** Global platform configuration.
- **Columns:** `id`, `key` (VARCHAR), `value` (JSONB), `description`.

#### **Audit Logs**
- **Purpose:** Immutable tracking of sensitive actions.
- **Columns:** `id` (UUID), `event_time` (TIMESTAMPTZ), `user_id` (UUID), `action` (VARCHAR 100), `resource` (VARCHAR 50), `resource_id` (UUID), `old_values` (JSONB), `new_values` (JSONB), `ip_address` (VARCHAR 45), `created_at` (TIMESTAMPTZ).
- **Constraints/Keys:** Composite primary key `(id, created_at)` for monthly range partitioning.
- **Business Rules:** Append-only. Updates and Deletes are explicitly blocked by PostgreSQL database triggers.

---

### 3.9 Project Schema (`project`)

#### **Projects**
- **Purpose:** Track revenues and costs for specific engagements.
- **Columns:** `id`, `project_code`, `name`, `customer_id`, `budget`, `status`.

#### **Project Profitability**
- **Purpose:** Materialized ledger mapping invoices, expenses, and POs to a project.

---

### 3.10 AI Schema (`ai`)

#### **AI Conversations** & **Prompts** & **Logs**
- **Purpose:** Track human-in-the-loop AI interactions and audit token usage.
- **Columns:** `id`, `user_id`, `context_module`, `prompt_text`, `response_text`, `provider` (Gemini/Bedrock), `tokens_used`, `latency_ms`.

---

### 3.11 Infrastructure Schema (`infrastructure`)

#### **Outbox Events** (Per ADR-003)
- **Columns:** `id` (UUID), `aggregate_type` (VARCHAR 50), `aggregate_id` (UUID), `event_name` (VARCHAR 100), `payload` (JSONB), `version` (INTEGER), `retry_count` (INTEGER), `next_retry_at` (TIMESTAMPTZ), `dead_letter_at` (TIMESTAMPTZ), `created_at` (TIMESTAMPTZ).
- **Constraints/Keys:** Composite primary key `(id, created_at)` for monthly range partitioning.
- **Business Rules:** 
  - Written in the same database transaction as the domain entity update.
  - Never automatically deleted for financial events.
  - **Immutability and Mutability Rules**: Triggers block `DELETE` operations. Triggers block `UPDATE` operations on all columns *except* metadata fields required for delivery retry and status tracking: `published` (boolean), `retry_count`, `next_retry_at`, `last_error`, and `dead_letter_at`.

#### **Inbox Events** (Per ADR-003)
- **Columns:** `event_id` (UUID PK), `processed_at` (TIMESTAMPTZ).
- **Business Rules:** Ensures idempotent event processing.

---

## 4. Relationships

- **Aggregate Boundaries:** Relationships crossing bounded contexts (e.g., `SalesInvoice` -> `Customer`) do NOT use strict database Foreign Keys at the schema level to maintain loose coupling. Instead, they use UUID references checked via application logic or event-carried state transfer.
- **Within Aggregates:** Strict Foreign Keys are enforced (e.g., `SalesInvoiceItem` -> `SalesInvoice`) with `ON DELETE CASCADE` because items cannot exist without their parent invoice.
- **Reference Data:** Shared catalogs (Products/Services) are copied into financial transaction tables (as Value Objects in JSONB or explicit columns) so historical invoices are immune to future catalog price/name changes.

---

## 5. Database Constraints

- **CHECK Constraints:** Used to enforce data validity at the lowest level (e.g., `CHECK (total_amount >= 0)`, `CHECK (status IN ('DRAFT', 'ISSUED'))`).
- **UNIQUE Constraints:** Applied to business identifiers (SKU, Invoice Number, Tax ID) to prevent logical duplicates.
- **NOT NULL:** Enforced aggressively. Only truly optional fields are nullable.

---

## 6. Performance & Indexing

- **Primary Indexes:** All `id` columns automatically indexed.
- **Foreign Key Indexes:** All UUID reference columns (e.g., `customer_id`, `invoice_id`) are explicitly indexed to prevent table scans during joins.
- **Partial Indexes:** Used for status queues. E.g., `CREATE INDEX idx_pending_outbox ON outbox_events(created_at) WHERE dead_letter_at IS NULL AND next_retry_at <= NOW()`.
- **GIN Indexes:** Applied to `JSONB` columns (like multi-language names or OCR results) to allow fast key/value querying (`@>` operator).
- **Pagination:** Keyset pagination (cursor-based filtering on `created_at` or `id`) is preferred over `OFFSET/LIMIT` for large datasets (e.g., Audit Logs).

---

## 7. Security

- **Encryption at Rest:** Handled by AWS RDS (KMS encryption).
- **PII:** Contact personal information (emails, phones) is restricted via application-level RBAC.
- **Secrets:** Passwords hashed via `bcrypt`. API keys stored in AWS Secrets Manager, not in the database.
- **Immutable Ledgers:** `audit_logs` is strictly protected against both `UPDATE` and `DELETE` actions via PostgreSQL triggers. The `outbox_events` and `petty_cash_ledger` tables are protected against `DELETE` actions, but allow `UPDATE` operations exclusively on status/retry/reconciliation metadata columns.

---

## 8. Backup Strategy

- **Automated Backups:** AWS RDS continuous automated backups with 35-day retention.
- **Point-in-Time Recovery (PITR):** Enabled for disaster recovery.
- **Cross-Region Replication:** Configurable based on exact availability SLA (OQ-03).

---

## 9. Archiving Strategy

- **Cold Storage:** Financial documents must be retained for 7 years. `deleted_at` records and historical `audit_logs` > 3 years old are moved to a secondary PostgreSQL archive database or AWS S3 via AWS Database Migration Service (DMS) / Glue jobs to maintain active DB performance.

---

## 10. Future Scalability

- **Partitioning:** The `audit_logs`, `outbox_events`, and `petty_cash_ledger` tables are designed for native PostgreSQL range partitioning by `created_at` (monthly chunks) to handle massive growth.
- **Read Replicas:** The architecture supports directing intensive Reporting and Analytics queries to an RDS Read Replica.
- **Microservices Migration:** Because schemas are strictly segregated by Bounded Context and cross-schema Foreign Keys are avoided, schemas can be detached into separate databases (Database-per-Service) with zero schema rewrites.
