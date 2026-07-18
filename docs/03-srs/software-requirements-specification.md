# VArrow AI Finance — Software Requirements Specification (SRS)

## Document Control

| Version | Date | Author | Change Description |
|---|---|---|---|
| v0.1 | 2026-07-18 | Chief Software Architect | Initial draft for detailed design phase |
| v1.0 | 2026-07-18 | Chief Software Architect | Completed expanding all 88 BRD functional requirements |

---

## 1. Introduction

### 1.1 Purpose
This Software Requirements Specification (SRS) serves as the definitive, implementation-ready guide for developers, database designers, quality assurance engineers, and system administrators. It defines the functional and non-functional requirements for the VArrow AI Finance platform, ensuring complete traceability to the Business Requirements Document (BRD), the Business Module Catalog, the System Architecture, the approved Architectural Decision Records (ADRs), the Database Schema, and the API Specification.

### 1.2 Scope
VArrow AI Finance is an enterprise financial management modular monolith application designed to automate, streamline, and audit financial workflows. The system integrates identity management, shared master catalogs (products, services, contacts), sales operations (quotations, invoicing, receipts), procurement (purchase orders), financial processing (purchase invoices, payments, expenses, petty cash), workflow approvals, document management (with OCR capabilities), reporting, notifications, and human-in-the-loop AI-assisted operations. It supports English and Arabic (RTL) out-of-the-box and runs on Web and Flutter Mobile platforms.

### 1.3 Definitions, Acronyms, Abbreviations
- **AST**: Arabia Standard Time (UTC+3)
- **ADR**: Architectural Decision Record
- **BRD**: Business Requirements Document
- **DDD**: Domain-Driven Design
- **RTL**: Right-to-Left (layout orientation for Arabic script)
- **OCR**: Optical Character Recognition
- **outbox**: Transactional Outbox Pattern for reliable event-driven communication
- **JWT**: JSON Web Token
- **RBAC**: Role-Based Access Control
- **SoD**: Separation of Duties
- **PII**: Personally Identifiable Information
- **S3**: Amazon Simple Storage Service (or S3-compatible storage)

### 1.4 References
- [Vision Document](../01-vision/VISION.md)
- [Business Requirements Document (BRD)](../02-brd/BRD.md)
- [Business Module Catalog](../02-brd/BUSINESS_MODULE_CATALOG.md)
- [ADR-001: Technology Stack](../04-adr/ADR-001-technology-stack.md)
- [ADR-002: Authentication Strategy](../04-adr/ADR-002-authentication-strategy.md)
- [ADR-003: Event Bus Design](../04-adr/ADR-003-event-bus-design.md)
- [ADR-004: Shared Catalog Ownership](../04-adr/ADR-004-shared-catalog-ownership.md)
- [System Architecture](../05-architecture/SYSTEM_ARCHITECTURE.md)
- [Database Schema Specification](../06-database/database-schema.md)
- [Enterprise Entity Relationship Diagram (ERD)](../06-database/ERD.md)
- [REST API Specification](../07-api/api-specification.md)

---

## 2. Overall Description

### 2.1 Product Perspective
VArrow AI Finance is deployed as a single-company, private/on-premise or dedicated cloud deployment (not multi-tenant SaaS). It is structured as a Modular Monolith inside a single codebase (NestJS), sharing a single PostgreSQL database partitioned logically into bounded context schemas. Inter-module communication relies on synchronous context-module calls for queries and asynchronous, event-driven Transactional Outbox patterns for write state-changes.

### 2.2 User Classes and Characteristics
- **Administrators**: System setup, security configuration, user management, global settings, audit inspections.
- **Finance Staff (Accountants/Payable/Receivable)**: Recording transactions, invoice matching, payment processing, petty cash reconciliation, report analysis.
- **Sales Representatives**: Customer communications, quotation generation, receipt allocations, catalog reviews.
- **Purchasing Officers**: Vendor relations, PO generation, catalog updates.
- **Department Managers / Approvers**: Approving financial documents, setting budgets, verifying expenses.
- **General Employees**: Submitting reimbursement claims, viewing personal request histories.
- **Auditors**: Read-only access to transaction history, settings history, and immutable audit logs.

### 2.3 Operating Environment
- **Server Environment**: Linux containers (Docker) deployed to AWS ECS/EKS, running Node.js (NestJS) with NestJS Dependency Injection.
- **Database**: PostgreSQL 16+ managed service (AWS RDS).
- **Web Frontend**: Modern SPA (React) supported on Chrome, Safari, Firefox, Edge.
- **Mobile Frontend**: Cross-platform application (Flutter) supported on iOS 15+ and Android 10+.

### 2.4 Design and Implementation Constraints
- **C-01**: English and Arabic bilingual support with RTL from launch.
- **C-02**: Human-in-the-loop AI philosophy (AI cannot execute autonomous financial decisions).
- **C-03**: Single-company deployment scope.
- **C-04**: No hard deletion of financial documents.
- **C-05**: Strict separation of duties in approval chains.
- **C-06**: Loose schema boundaries between modules — no foreign keys crossing schemas (ADR-004).

---

## 3. Functional Requirements

This section contains implementation-ready specifications. All 88 functional requirements from the BRD are mapped to these modules.

### 3.1 Authentication & Identity (FR-001 to FR-008)

#### **SRS-ID-001: User Authentication & Session Management (FR-001, FR-002)**
- **Description**: Authenticate registered users via credentials and manage secure session tokens with rotation.
- **Actors**: Any registered user.
- **Preconditions**: User has an active record in `identity.users` where `status = 'ACTIVE'`.
- **Trigger**: User sends authentication request to `/api/v1/auth/login`.
- **Main Flow**:
  1. User submits `email` and plain `password`.
  2. System verifies input format.
  3. System queries `identity.users` by `email` (excluding soft-deleted).
  4. System verifies password using bcrypt.
  5. System generates short-lived JWT Access Token (15m TTL) and long-lived Refresh Token (7d TTL).
  6. System saves Refresh Token hash in `identity.refresh_tokens` and sets it in an HTTP-only secure cookie.
  7. Return `200 OK` with user details and access token.
- **Alternative Flows**:
  - **AF-1 (Token Refresh)**: Client calls `/api/v1/auth/refresh` with refresh cookie. System verifies token validity, deletes old refresh token, generates a new token pair, writes to DB, and returns `200 OK`.
- **Exception Flows**:
  - **EF-1 (Invalid Credentials)**: Return `401 Unauthorized` with generic message to prevent enumeration.
  - **EF-2 (Account Suspended/Inactive)**: If `status` is not `ACTIVE`, block access, return `403 Forbidden`.
  - **EF-3 (Token Reuse)**: If a refresh token is submitted twice, block family, invalidate all associated tokens (ADR-002), return `401 Unauthorized`.
- **Postconditions**: User is logged in. Session logged.
- **Validation Rules**: Email must be valid format, password must be non-empty.
- **Business Rules**: No session can bypass JWT validation.
- **Permissions**: Public (Login), User (Refresh).
- **Input Constraints**: Email (max 255), Password (max 128).
- **Output Requirements**: JSON object with access token and user metadata.
- **Error Handling**: RFC 7807 standard error structure.
- **Audit Requirements**: Audit record written for successful logins and all failures.
- **Events Published**: `Identity.UserAuthenticated`, `Identity.AuthenticationFailed`.
- **Events Consumed**: None.
- **Performance Requirements**: Authentication latency < 400ms (bcrypt hash verification).
- **Security Requirements**: HTTPS only. HTTP-only, Secure, SameSite=Strict cookies. Bcrypt work factor = 12.
- **Acceptance Criteria**:
  - Valid login returns JWT.
  - Wrong password returns generic 401.
  - Token reuse suspends token family.

#### **SRS-ID-002: Authorization & Access Control (FR-003, FR-004, FR-005, FR-006)**
- **Description**: Enforce role-based access control (RBAC) and allow administrators to manage users and roles.
- **Actors**: Administrator, System.
- **Preconditions**: Administrator is authenticated with appropriate roles.
- **Trigger**: Administrator accesses `/api/v1/identity/users` or `/api/v1/identity/roles`.
- **Main Flow**:
  1. Admin creates role or updates user assignments.
  2. System validates role structure, saves to `identity.roles`, and assigns permissions.
  3. Deny-by-default access check occurs on every API request.
- **Alternative Flows**:
  - **AF-1 (Deactivate User)**: Admin updates user status to `INACTIVE`. All current refresh tokens for this user are deleted.
- **Exception Flows**:
  - **EF-1 (Separation of Duties Violation)**: Attempting to assign a user to both request and approve the same document type returns `400 Bad Request`.
- **Postconditions**: User/role mapping updated in database. Access privileges modified.
- **Validation Rules**: Role name must be unique.
- **Business Rules**: System roles (e.g. SuperAdmin) cannot be deleted.
- **Permissions**: `identity.admin`.
- **Input Constraints**: Role mapping payload.
- **Output Requirements**: Updated entity representation.
- **Error Handling**: 403 Forbidden for unauthorized actions.
- **Audit Requirements**: Change in roles/permissions must be logged.
- **Events Published**: `Identity.UserCreated`, `Identity.UserUpdated`, `Identity.RoleModified`.
- **Events Consumed**: None.
- **Performance Requirements**: Permission check overhead < 10ms.
- **Security Requirements**: Optimistic locking on user edits.
- **Acceptance Criteria**:
  - Inactive users cannot make authenticated calls.
  - Role modifications take effect within 15 minutes (or immediately upon refresh).

#### **SRS-ID-003: Security Auditing & Alerts (FR-007, FR-008)**
- **Description**: Log all auth actions and trigger notifications for suspicious login behavior.
- **Actors**: System.
- **Preconditions**: Auth endpoint accessed.
- **Trigger**: Login failure event or access changes occur.
- **Main Flow**:
  1. System captures auth event.
  2. System records details in `administration.audit_logs`.
  3. If 5 failed logins occur for the same username within 15 minutes, notify admin.
- **Exception Flows**:
  - **EF-1 (DB Write Failure)**: Fail-safe to system error logs, do not crash application.
- **Postconditions**: Suspicious events logged and notifications dispatched.
- **Business Rules**: Log entries are append-only.
- **Permissions**: System process.
- **Audit Requirements**: Log IP, source country, browser fingerprint, and username.
- **Events Published**: `Identity.SuspiciousActivityDetected`.
- **Events Consumed**: `Identity.AuthenticationFailed`.
- **Security Requirements**: Block IP for 1 hour after 10 failed login attempts.
- **Acceptance Criteria**:
  - Failed logins visible in audit logs immediately.
  - Admin receives alert on threshold breach.

---

### 3.2 Master Data Management (FR-009 to FR-015)

#### **SRS-MDM-001: Trading Partners (Customers & Vendors) (FR-009, FR-010, FR-011)**
- **Description**: Maintain customer and vendor records along with their associated contacts.
- **Actors**: Sales Rep, Purchasing Officer, Finance Staff.
- **Preconditions**: Authenticated user with catalog management permission.
- **Trigger**: User calls `/api/v1/catalog/customers` or `/api/v1/catalog/vendors`.
- **Main Flow**:
  1. User inputs partner data (name, unique tax registration number, default currency).
  2. System validates tax number format and uniqueness.
  3. Records created in `catalog.customers` or `catalog.vendors`.
- **Alternative Flows**:
  - **AF-1 (Add Contact)**: Link a new contact record from `catalog.contacts` to a partner.
- **Postconditions**: Partner record active and queryable.
- **Validation Rules**: Tax registration number must be unique.
- **Business Rules**: Partners cannot be hard-deleted if they have transactional history.
- **Permissions**: `catalog.write`.
- **Input Constraints**: Partner fields.
- **Output Requirements**: Partner object with unique UUID.
- **Audit Requirements**: Log creation and modifications.
- **Events Published**: `Catalog.CustomerCreated`, `Catalog.VendorCreated`, `Catalog.CustomerUpdated`.
- **Events Consumed**: None.
- **Acceptance Criteria**:
  - Duplicate tax numbers are rejected with `409 Conflict`.
  - Contacts are successfully linked.

#### **SRS-MDM-002: Product & Service Catalog (FR-012, FR-013, FR-014, FR-015)**
- **Description**: Manage items (products and services) and ensure discontinued items cannot be added to new documents.
- **Actors**: Sales Rep, Purchasing Officer, Admin.
- **Preconditions**: Authenticated.
- **Trigger**: User updates items via `/api/v1/catalog/products` or `/api/v1/catalog/services`.
- **Main Flow**:
  1. User inputs product/service name, SKU/code, unit price, and default tax rate.
  2. System checks uniqueness of SKU/code.
  3. Writes to database.
- **Alternative Flows**:
  - **AF-1 (Discontinue)**: Toggle `is_active = false`.
- **Exception Flows**:
  - **EF-1 (Referencing Discontinued)**: Attempting to reference an inactive item in a draft quotation or purchase order returns `422 Unprocessable Entity`.
- **Postconditions**: Status updated.
- **Validation Rules**: Price must be positive `NUMERIC(19,4)`.
- **Business Rules**: Historical documents referencing discontinued items must remain unchanged.
- **Permissions**: `catalog.write`.
- **Input Constraints**: Valid SKU format, positive pricing.
- **Events Published**: `Catalog.ProductCreated`, `Catalog.ProductUpdated`, `Catalog.ProductDiscontinued`.
- **Events Consumed**: None.
- **Acceptance Criteria**:
  - Active flag toggle behaves correctly.
  - Inactive products do not appear in sales catalog selection APIs.

---

### 3.3 Sales Operations (FR-016 to FR-025)

#### **SRS-SAL-001: Quotation Lifecycle (FR-016, FR-017, FR-018, FR-019)**
- **Description**: Manage quotation creation, revision versioning, status transitions, and automatic expiration.
- **Actors**: Sales Representative, Customer.
- **Preconditions**: Customer is active.
- **Trigger**: User calls `/api/v1/sales/quotations`.
- **Main Flow**:
  1. Representative drafts quotation.
  2. System snapshots product prices and customer details at creation to preserve historical rates.
  3. Quotation enters `DRAFT` state.
  4. Representative submits for approval -> status becomes `PENDING_APPROVAL`.
  5. Upon approval, status changes to `APPROVED`.
  6. Representative prints/sends -> status changes to `SENT`.
- **Alternative Flows**:
  - **AF-1 (Revision)**: Rep modifies a draft/rejected quotation. System increments the version count (e.g., version 2) and preserves the original quotation details in the revision history.
  - **AF-2 (Expiry Job)**: A nightly cron job queries for `SENT` quotations where `valid_until < CURRENT_DATE` and sets their status to `EXPIRED`.
- **Postconditions**: Quotation updated.
- **Validation Rules**: `valid_until` date must be in the future.
- **Business Rules**: Only draft quotations can be edited.
- **Permissions**: `sales.quotations.write`.
- **Audit Requirements**: Log quotation state transitions.
- **Events Published**: `Sales.QuotationCreated`, `Sales.QuotationApproved`, `Sales.QuotationExpired`.
- **Events Consumed**: None.
- **Acceptance Criteria**:
  - Valid until constraint enforced.
  - Overdue quotations automatically flag as expired.
  - Revision history matches previous version states.

#### **SRS-SAL-002: Invoicing & Receipt Allocation (FR-020, FR-021, FR-022, FR-023, FR-024, FR-025)**
- **Description**: Issue sales invoices, identify overdue payments, record receipts, and allocate them while preventing duplicate transactions.
- **Actors**: Accountant, System.
- **Preconditions**: Customer exists.
- **Trigger**: API calls to `/api/v1/sales/sales-invoices` or `/api/v1/sales/receipts`.
- **Main Flow**:
  1. Accountant generates an invoice (optionally from an accepted quotation).
  2. System computes tax, generates a unique sequence number, and registers it.
  3. Once issued, the invoice is set to `ISSUED` and becomes **immutable** (no edits allowed).
  4. A nightly cron job transitions unpaid invoices with past due dates to `OVERDUE`.
  5. When payment is received, the accountant creates a receipt referencing the invoice.
  6. System updates invoice balance and customer credit ledger.
- **Exception Flows**:
  - **EF-1 (Duplicate Receipt)**: If an incoming receipt has the same bank transaction reference number for that partner, reject with `409 Conflict`.
- **Validation Rules**: Subtotal, Tax, and Grand Total calculations must match.
- **Business Rules**: Grand totals must use `NUMERIC(19,4)`.
- **Permissions**: `sales.invoices.write`, `sales.receipts.write`.
- **Audit Requirements**: Ledger updates require audit logs.
- **Events Published**: `Sales.SalesInvoiceIssued`, `Sales.SalesInvoiceOverdue`, `Sales.SalesInvoicePaid`, `Sales.ReceiptRecorded`.
- **Events Consumed**: None.
- **Acceptance Criteria**:
  - Issued invoices return `403` on attempts to edit.
  - Overdue state updates correctly.
  - Receipts apply exactly to outstanding balances.

---

### 3.4 Procurement Operations (FR-026 to FR-030)

#### **SRS-PRO-001: Purchase Orders & Integration (FR-026, FR-027, FR-028, FR-029, FR-030)**
- **Description**: Create purchase orders, require approval before issuance, and trace POs to invoices and payments.
- **Actors**: Purchasing Officer, Approver, Vendor.
- **Preconditions**: Vendor and items are active in the catalog.
- **Trigger**: Call to `/api/v1/procurement/purchase-orders`.
- **Main Flow**:
  1. Purchasing Officer drafts a PO.
  2. PO goes to approval flow.
  3. System blocks issuance (status transitions to `ISSUED`) until the workflow changes state to `APPROVED`.
  4. Vendor delivers goods/services and submits invoice.
  5. Accountant registers vendor invoice, linking it to the PO.
  6. Finance processes payment, completing the PO cycle.
- **Exception Flows**:
  - **EF-1 (Unapproved Issuance)**: Attempting to mark a `PENDING_APPROVAL` PO as `ISSUED` returns `403 Forbidden`.
- **Postconditions**: Document traceability paths established.
- **Business Rules**: A PO cannot be closed unless fully matched to invoices or manually cancelled.
- **Permissions**: `procurement.write`.
- **Audit Requirements**: Trace all transitions from PO creation to payment.
- **Events Published**: `Procurement.PurchaseOrderCreated`, `Procurement.PurchaseOrderApproved`, `Procurement.PurchaseOrderIssued`.
- **Events Consumed**: `Finance.PurchaseInvoiceRecorded`, `Finance.PaymentCompleted`.
- **Acceptance Criteria**:
  - System blocks issuing draft POs.
  - Procurement logs verify full trace matching.

---

### 3.5 Financial Processing (FR-031 to FR-045)

#### **SRS-FIN-001: Payables (Purchase Invoices & Payments) (FR-031, FR-032, FR-039, FR-040, FR-041)**
- **Description**: Record purchase invoices, match them to POs, verify duplicates, and record outgoing payments.
- **Actors**: Accountant, System.
- **Preconditions**: Approved vendor and optional matching PO.
- **Trigger**: Call to `/api/v1/finance/purchase-invoices` or `/api/v1/finance/payments`.
- **Main Flow**:
  1. Accountant uploads/inputs vendor invoice details.
  2. System checks for duplicate invoice numbers for the same vendor; if found, reject.
  3. System matches PO items to verify quantity/amount consistency.
  4. Invoice is set to `RECEIVED`. Once approved, status changes to `APPROVED`.
  5. Payment processed via bank transfer. Accountant registers payment.
  6. System reduces vendor payable balance.
- **Exception Flows**:
  - **EF-1 (Duplicate Invoice Check)**: Vendor invoice number + vendor ID matching existing record rejects with `409 Conflict`.
  - **EF-2 (Unauthorized Payment)**: Paying an invoice that is not in `APPROVED` status rejects with `422 Unprocessable Entity`.
- **Validation Rules**: Payment amount must be <= remaining invoice balance.
- **Business Rules**: All transactions must log matching currency values.
- **Permissions**: `finance.payables.write`.
- **Events Published**: `Finance.PurchaseInvoiceRecorded`, `Finance.PaymentCompleted`.
- **Acceptance Criteria**:
  - Prevent duplicate vendor invoices.
  - Lock payments to approved invoices only.

#### **SRS-FIN-002: Expense Management (FR-033, FR-034, FR-035, FR-036)**
- **Description**: Record general expenses, support employee claims with multiple itemized receipts under one header claim, and track reimbursement status.
- **Actors**: Employee, Finance Staff, Department Manager.
- **Preconditions**: User account is active.
- **Trigger**: User calls `/api/v1/finance/expenses` or `/api/v1/finance/employee-expenses`.
- **Main Flow**:
  1. Employee creates an expense claim header, specifying currency.
  2. Employee inserts one or more itemized receipts as `employee_expense_items` (each item contains its category, description, amount, and receipt attachment reference).
  3. System validates that each item has a valid, uploaded receipt attachment (mandatory).
  4. System updates the claim header's `total_amount` by summing the items' amounts.
  5. Claim transitions to `SUBMITTED` and enters approval routing.
  6. Upon approval, Finance staff records reimbursement.
  7. Claim status changes to `REIMBURSED`.
- **Alternative Flows**:
  - **AF-1 (Mobile Photo Submission)**: Employee uses mobile app to take photo of receipt. Mobile triggers upload and starts background OCR processing.
  - **AF-2 (Soft Delete Claim)**: Employee soft deletes a claim before submission. System marks the parent claim as deleted (`deleted_at = NOW()`) and uses **Application-Layer Cascading** to mark all linked `employee_expense_items` as soft-deleted in the same transaction.
- **Postconditions**: Expense logged, itemized receipts cataloged, status tracked.
- **Validation Rules**: 
  - Claim date must not be in the future. 
  - Item amount must be greater than or equal to zero (`CHECK (amount >= 0)`). 
  - File type must be PDF/Image, max 10MB.
- **Business Rules**: Expenses above configurable thresholds must route through manager approval.
- **Permissions**: `finance.expenses.write` (Staff), `employee.expenses.submit` (Employee).
- **Events Published**: `Finance.ExpenseRecorded`, `Finance.EmployeeExpenseSubmitted`, `Finance.EmployeeExpenseReimbursed`.
- **Events Consumed**: None.
- **Acceptance Criteria**:
  - Enforce attachment presence for every item before submission.
  - Expense status and total amount track accurately.
  - Soft-deleted parent claims correctly cascade soft-deletes to all child items.

#### **SRS-FIN-003: Petty Cash Administration (FR-037, FR-038)**
- **Description**: Manage petty cash funds, enforce limits, and record reconciliations using an append-only transaction ledger.
- **Actors**: Custodian, Finance Staff.
- **Preconditions**: Fund account created and active.
- **Trigger**: Call to `/api/v1/finance/petty-cash`.
- **Main Flow**:
  1. Custodian registers a disbursement transaction.
  2. System verifies transaction is within individual disbursement limit.
  3. System inserts transaction entry to `finance.petty_cash_ledger` (specifying type as `WITHDRAWAL`, reference number, amount, custodian ID, and description).
  4. System updates active fund balance in `finance.petty_cash` (balance must not drop below zero).
  5. Custodian performs periodic count and submits reconciliation (inserting type `RECONCILIATION_DISCREPANCY` if counts differ).
- **Exception Flows**:
  - **EF-1 (Disbursement Limit Exceeded)**: Request exceeding configured transaction limit rejects with `400 Bad Request`.
  - **EF-2 (Insufficent Fund Balance)**: Attempting to disburse more than the current fund balance rejects with `422 Unprocessable Entity` (`CHECK (balance >= 0)`).
  - **EF-3 (Ledger Alteration)**: Any SQL update/delete query on `petty_cash_ledger` fails due to database level triggers.
- **Validation Rules**: Disbursement/deposit amount must be greater than or equal to zero (`CHECK (amount >= 0)`).
- **Business Rules**: 
  - Petty cash ledger must be strictly append-only. 
  - All balance changes must be matched by a ledger record.
- **Permissions**: `finance.pettycash.custodian`, `finance.pettycash.write` (Staff).
- **Events Published**: `Finance.PettyCashDisbursed`, `Finance.PettyCashReconciled`.
- **Acceptance Criteria**:
  - Transactions reject if above limit or if balance would drop below zero.
  - Ledgers match physical count updates.
  - Verification that updates/deletions on ledger return SQL errors.

#### **SRS-FIN-004: Core Finance Utilities (FR-042, FR-043, FR-044, FR-045)**
- **Description**: Enforce multi-currency tracking, apply tax policies, compile cash flows, and trace project costs.
- **Actors**: Accountant, Finance Manager.
- **Preconditions**: Valid exchange rates and tax rules configured.
- **Trigger**: Transaction processes or queries to dashboard.
- **Main Flow**:
  1. System checks tax rules and computes VAT.
  2. Conversion endpoint translates non-reporting currencies.
  3. Materialized views compile inflows/outflows for cash flow analysis.
  4. Cost/revenue elements link to project entities.
- **Validation Rules**: All financial amounts use `NUMERIC(19,4)`.
- **Business Rules**: Currencies must be ISO 4217 standard codes.
- **Permissions**: `finance.read`.
- **Events Consumed**: All transactional payment and receipt events.
- **Acceptance Criteria**:
  - VAT calculations match defined system rates.
  - Cash flow dashboards update after payment/receipt events.

---

### 3.6 Approval Workflows (FR-046 to FR-052)

#### **SRS-WRK-001: Workflow Engine (FR-046, FR-047, FR-048, FR-049, FR-050, FR-051, FR-052)**
- **Description**: Configure routing workflows, direct documents to approvers, prevent self-approvals, and handle escalations.
- **Actors**: Approver, Submitter, System.
- **Preconditions**: Document is submitted.
- **Trigger**: System matches document to workflow configuration.
- **Main Flow**:
  1. System matches document type and amount to active workflow schema.
  2. System assigns first step to designated approver user/role.
  3. Approver reviews and selects `APPROVE`, `REJECT`, or `RETURN`.
  4. Decisions are saved in database with timestamp, identity, and comments.
  5. Upon final approval, document status updates.
- **Alternative Flows**:
  - **AF-1 (Escalation)**: Approver fails to act within the limit (e.g. 48 hours). System automatically routes the request to their supervisor.
- **Exception Flows**:
  - **EF-1 (Self-Approval)**: If an approver tries to approve a document they submitted, block the action and return `403 Forbidden` (SoD enforcement).
- **Postconditions**: Document state updated. Logs saved.
- **Business Rules**: Submitter can never approve their own request (BR-04).
- **Permissions**: `workflow.decide`.
- **Events Published**: `Workflow.ApprovalRequested`, `Workflow.ApprovalDecisionMade`, `Workflow.ApprovalEscalated`, `Workflow.WorkflowCompleted`.
- **Acceptance Criteria**:
  - Workflow routes based on value rules.
  - Self-approvals are blocked at API level.
  - Logs are saved.

---

### 3.7 Document Management & OCR (FR-053 to FR-060)

#### **SRS-DOC-001: Document Management (FR-053, FR-054, FR-055, FR-056)**
- **Description**: Upload files, link to records, enforce retention policies, and require attachments.
- **Actors**: All users.
- **Preconditions**: Valid target record exists.
- **Trigger**: API calls to `/api/v1/documents/attachments/upload`.
- **Main Flow**:
  1. User uploads a file.
  2. System validates size (< 10MB) and format (PDF/Image).
  3. System uploads file to S3 and registers details in the database.
  4. System links document to target entity.
- **Validation Rules**: Document files must have valid extensions.
- **Business Rules**: Retention policy blocks physical deletions before configured timelines (e.g. 7 years).
- **Permissions**: `documents.write`.
- **Events Published**: `Document.DocumentUploaded`.
- **Acceptance Criteria**:
  - Files upload and link correctly.
  - Attempting to delete protected documents returns `403`.

#### **SRS-DOC-002: OCR & Review (FR-057, FR-058, FR-059, FR-060)**
- **Description**: Process uploads through OCR, flag low-confidence values, and present for human review in English/Arabic.
- **Actors**: Accountant, System.
- **Preconditions**: Uploaded file exists.
- **Trigger**: Upload completes for an eligible invoice/expense document.
- **Main Flow**:
  1. System starts OCR extraction.
  2. System identifies fields: vendor name, invoice date, total amount, currency.
  3. System saves values to `document.ocr_results` with confidence scores.
  4. If any confidence score is below threshold (0.85), flag for review.
  5. Accountant reviews, updates errors, and clicks verify.
  6. Data is finalized for transaction creation.
- **Business Rules**: OCR data cannot be used in financial records without user verification (BR-19).
- **Permissions**: `documents.ocr.read`.
- **Events Published**: `Document.ExtractionReviewRequired`, `Document.DocumentProcessed`.
- **Acceptance Criteria**:
  - System flags low confidence correctly.
  - Supports English and Arabic characters.

---

### 3.8 Reporting & Dashboards (FR-061 to FR-066)

#### **SRS-RPT-001: Analytics, Reports & Dashboards (FR-061, FR-062, FR-063, FR-064, FR-065, FR-066)**
- **Description**: Render permission-aware dashboards, highlight alerts, generate parameterized reports, and support exports.
- **Actors**: Finance Leadership, Department Managers, Accountants, Sales Reps.
- **Preconditions**: Authenticated user.
- **Trigger**: Access dashboard or trigger report generation.
- **Main Flow**:
  1. System checks user permissions.
  2. Dashboard filters and renders widgets based on user scope.
  3. User requests a report (e.g., Accounts Payable Aging) with filters.
  4. System runs query and presents data.
  5. User exports report to Excel/PDF.
- **Validation Rules**: Reports must match source transactional balances.
- **Permissions**: `reports.read`.
- **Acceptance Criteria**:
  - Dashboards hide unauthorized widgets.
  - Reports export successfully.

---

### 3.9 Notifications (FR-067 to FR-072)

#### **SRS-NTF-001: Notification Dispatch & Preferences (FR-067, FR-068, FR-069, FR-070, FR-071, FR-072)**
- **Description**: Deliver notifications via in-app alerts, email, and mobile push channels, and support configuration preferences.
- **Actors**: All users.
- **Preconditions**: Target user has active notification channels.
- **Trigger**: Domain event is published.
- **Main Flow**:
  1. Notification processor consumes event.
  2. Resolves target users.
  3. Applies user channel preferences.
  4. Sends notification.
- **Business Rules**: Critical notifications (approvals, security locks) cannot be disabled (BR-23).
- **Permissions**: `notifications.read`.
- **Events Consumed**: All transactional and workflow events.
- **Acceptance Criteria**:
  - In-app notification appears within 2 seconds of event.
  - System blocks disabling critical notifications.

---

### 3.10 Audit & Administration (FR-073 to FR-077)

#### **SRS-ADM-001: Immutable Audit Ledger & Settings (FR-073, FR-074, FR-075, FR-076, FR-077)**
- **Description**: Log all actions in an immutable table, restrict queries to admins/auditors, manage platform settings, and back up data.
- **Actors**: Administrator, Auditor.
- **Preconditions**: Admin permissions.
- **Trigger**: User calls `/api/v1/admin/audit-logs` or updates settings.
- **Main Flow**:
  1. System monitors all write actions.
  2. System writes details to `administration.audit_logs`.
  3. Admin views and queries logs.
  4. Admin updates settings. System logs old/new values in setting history.
- **Exception Flows**:
  - **EF-1 (Audit Manipulation)**: Any UPDATE or DELETE query on `audit_logs` table fails due to database level triggers.
- **Business Rules**: Audit log is completely read-only for users (no updates/deletions).
- **Permissions**: `admin.audit.read`, `admin.settings.write`.
- **Events Published**: `Administration.SettingsUpdated`.
- **Acceptance Criteria**:
  - Attempts to delete logs return SQL errors.
  - Settings history updates accurately.

---

### 3.11 Localization (FR-078 to FR-081)

#### **SRS-LOC-001: Bilingual Operations & RTL (FR-078, FR-079, FR-080, FR-081)**
- **Description**: Provide English and Arabic translations, support RTL layouts, manage mixed strings, and maintain UI consistency across platforms.
- **Actors**: All users.
- **Preconditions**: Language selection updated in settings.
- **Trigger**: Client renders UI or makes API requests.
- **Main Flow**:
  1. App determines active locale.
  2. Renders strings from translation catalog.
  3. Renders Arabic in right-to-left layout.
  4. Retain mixed-language support (English SKUs or numbers within Arabic sentences) using auto direction features.
- **Acceptance Criteria**:
  - RTL behaves correctly for Arabic pages.
  - Mixed language texts display correctly without layout errors.

---

### 3.12 AI Capabilities (FR-082 to FR-088)

#### **SRS-AIC-001: AI Center Integrations (FR-082, FR-083, FR-084, FR-085, FR-086, FR-087, FR-088)**
- **Description**: Generate drafts, suggest expense categories, extract document fields, restrict AI decisions, and log statistics.
- **Actors**: Any user (requester), Accountant, System.
- **Preconditions**: AI configurations active.
- **Trigger**: User requests AI assistance.
- **Main Flow**:
  1. User calls AI helper endpoint.
  2. System resolves active provider and fallback adapter.
  3. System constructs context-aware prompt, enforcing user data permissions.
  4. Prompt is formatted specifically for the target LLM provider (using system directives and standard schemas).
  5. Sends prompt to LLM service (Gemini or Bedrock).
  6. Normalizes the unstructured JSON response (extracting JSON blocks out of markdown fences, parsing standard keys).
  7. Validates the normalized response using Zod schema models before presenting to the client.
  8. Returns validated recommendation to UI.
  9. User accepts or rejects the suggestion.
  10. Interaction metrics (token count, latency, decision) are saved in `ai.logs`.
- **Exception Flows**:
  - **EF-1 (AI Outage)**: If the primary LLM provider (Google Gemini) is unavailable, the Model Abstraction Layer automatically switches to the fallback provider adapter (AWS Bedrock). The adapter formats the prompt to match Bedrock's model structure, calls the Bedrock API, and normalizes the response through the normalization layer before returning. If both fail, return a user-friendly error. Core workflows must continue to operate.
- **Business Rules**: AI cannot make autonomous financial decisions (BR-20, C-02).
- **Permissions**: `ai.assist`.
- **Events Published**: `AI.AIAssistanceRequested`, `AI.AISuggestionReady`, `AI.AIAssistanceAccepted`.
- **Acceptance Criteria**:
  - System presents recommendations as drafts only.
  - Token counts and performance metrics save correctly.
  - Permission checks restrict AI context.
  - Response normalization parses raw outputs successfully without causing Zod validation errors.

---

## 4. Non-Functional Requirements

### 4.1 Performance (NFR-01 to NFR-04)
- **NFRS-PER-001**: **API Response Latency**: Standard transactional endpoints must respond in < 500ms (p95) under normal load (NFR-01).
- **NFRS-PER-002**: **Report Compilation**: High-volume financial reports must compile and display in < 10 seconds (NFR-02).
- **NFRS-PER-003**: **OCR Performance**: OCR processing must run asynchronously, taking < 30 seconds for standard invoices without blocking user workflows (NFR-03).
- **NFRS-PER-004**: **Concurrent Load**: The system must support at least 50 concurrent active users for a single company deployment without degradation (NFR-04).

### 4.2 Security (NFR-05 to NFR-10)
- **NFRS-SEC-001**: **Authentication Security**: Authentication must use secure JWTs. Passwords must be hashed using bcrypt (cost factor 12) (NFR-05).
- **NFRS-SEC-002**: **Session Security**: Enforce refresh token rotation and secure HTTP-only cookies (NFR-06).
- **NFRS-SEC-003**: **Access Controls**: Deny-by-default RBAC enforced at the API gateway layer (NFR-07).
- **NFRS-SEC-004**: **Separation of Duties (SoD)**: Submitter and approver must be different users for all financial transactions (NFR-08).
- **NFRS-SEC-005**: **Data Encryption**: All data in transit must use HTTPS/TLS 1.3. Data at rest must use AES-256 (AWS RDS KMS) (NFR-09).
- **NFRS-SEC-006**: **Immutable Logging**: Database-level triggers must prevent any modifications or deletions in the audit logs table (NFR-10).

### 4.3 Scalability & Data Archiving (NFR-11, NFR-12)
- **NFRS-SCA-001**: **5-Year Growth Plan**: Database tables (specifically audit logs and outbox events) must support native partitioning by `created_at` (monthly range partitions) to sustain 5+ years of operational data (NFR-11).
- **NFRS-SCA-002**: **Modular Monolith Integrity**: Modules must maintain strict boundary isolation, communicating only via the event bus or defined public APIs, allowing future extraction into microservices with zero schema rewrites (NFR-12).

### 4.4 Reliability & Availability (NFR-13, NFR-14)
- **NFRS-REL-001**: **Uptime Targets**: The platform must guarantee 99.5% uptime during standard business hours (Sun-Thu, 08:00 to 18:00 AST) (NFR-13).
- **NFRS-REL-002**: **Priority Workflows**: Core workflows (approvals, payments, invoicing) must remain functional during peak resource usage, degrading non-critical features (like AI chat) first (NFR-14).

### 4.5 Localization (NFR-15 to NFR-19)
- **NFRS-LOC-001**: **Languages**: Full translation dictionaries for English and Arabic (NFR-15).
- **NFRS-LOC-002**: **Bi-directional Layouts**: Auto RTL orientation rendering based on selected user language (NFR-16).
- **NFRS-LOC-003**: **Mixed Text Rendering**: Proper alignment for strings mixing English alphanumeric codes (SKUs, UUIDs) and Arabic text (NFR-17).
- **NFRS-LOC-004**: **Cross-Platform Parity**: Identical localized copy and layouts between React web and Flutter mobile (NFR-18).
- **NFRS-LOC-005**: **Regional Formatting**: Date format `DD/MM/YYYY` (EN) and appropriate Arabic formats. Numeric values must format correctly based on locale (NFR-19).

### 4.6 Accessibility (NFR-20, NFR-21)
- **NFRS-ACC-001**: **Web Standards**: Meet WCAG 2.1 Level AA requirements for web UI elements (NFR-20).
- **NFRS-ACC-002**: **Mobile Accessibility**: Comply with Apple Human Interface Guidelines and Android Accessibility features (NFR-21).

### 4.7 Compliance & Infrastructure (NFR-22 to NFR-29)
- **NFRS-COM-001**: **Audit Coverage**: Ensure 100% of financial modifications log an immutable audit record (NFR-22, NFR-23).
- **NFRS-COM-002**: **Document Retention**: Enforce a retention policy of 7 years for financial records before archiving (NFR-24).
- **NFRS-COM-003**: **Transaction Integrity**: Enforce ACID guarantees. Partial writes to multi-entity aggregates (like invoice headers + items) are strictly prohibited (NFR-25).
- **NFRS-COM-004**: **Backups**: Configure automated point-in-time recovery (PITR) with continuous backups (NFR-26).
- **NFRS-COM-005**: **Reliable Messaging**: Enforce at-least-once event delivery via outbox processing, verifying idempotency on the consumer side via the inbox events ledger (NFR-027, NFR-28).
- **NFRS-COM-006**: **Data Ownership**: Adhere to ADR-004 boundaries (NFR-29).

---

## 5. Coding Assumptions

- **CA-01**: NestJS is used for the backend framework. Each bounded context is isolated inside its own module.
- **CA-02**: All monetary amounts use `NUMERIC(19,4)`. Floating-point types (`float`, `double`) are prohibited for financial values.
- **CA-03**: No database-level foreign keys are allowed across schemas representing different bounded contexts. Data consistency is managed via event communication and application checks (ADR-004).
- **CA-04**: Timestamps are stored in UTC (`TIMESTAMPTZ`). Local translation is handled in user interface presentations.
- **CA-05**: Primary keys are `UUIDv4`, generated via `gen_random_uuid()` at DB level.
- **CA-06**: Write operations implement optimistic locking using an integer `version` field.
- **CA-07**: No hard deletes are allowed for transactional tables. Instead, soft deletes toggle `deleted_at`.
- **CA-08**: Event dispatching uses the Transactional Outbox pattern, running within the database transaction of the state change.
- **CA-09**: JWT Access tokens expire after 15 minutes. Refresh tokens rotate on reuse (ADR-002).

---

## 6. Testing Traceability Matrix

| BRD ID | Requirement Summary | SRS ID | Test Category |
|---|---|---|---|
| FR-001 | Authenticate users, verify status | SRS-ID-001 | Integration, E2E |
| FR-002 | Session management & refresh | SRS-ID-001 | Integration, Security |
| FR-003 | Role-based Access Control | SRS-ID-002 | Integration, security |
| FR-004 | Separation of Duties | SRS-ID-002, SRS-WRK-001 | E2E, Unit |
| FR-005 | Admin user management | SRS-ID-002 | Integration |
| FR-006 | Define roles/permissions | SRS-ID-002 | Integration |
| FR-007 | Record auth events | SRS-ID-003 | Integration |
| FR-008 | Suspicious activity notifications | SRS-ID-003 | Integration |
| FR-009 | Customer Lifecycle | SRS-MDM-001 | Unit, Integration |
| FR-010 | Vendor Lifecycle | SRS-MDM-001 | Unit, Integration |
| FR-011 | Contact associations | SRS-MDM-001 | Unit, Integration |
| FR-012 | Product Catalog | SRS-MDM-002 | Unit, Integration |
| FR-013 | Service Catalog | SRS-MDM-002 | Unit, Integration |
| FR-014 | Document data validation | SRS-MDM-002 | Integration |
| FR-015 | Exclude discontinued items | SRS-MDM-002 | Integration |
| FR-016 | Quotation Creation | SRS-SAL-001 | Integration, E2E |
| FR-017 | Quotation revisions | SRS-SAL-001 | Unit, Integration |
| FR-018 | Quotation auto-expiration | SRS-SAL-001 | Integration, Cron |
| FR-019 | Convert Quotation to Invoice | SRS-SAL-001 | Integration |
| FR-020 | Create invoices (tax/currency) | SRS-SAL-002 | Integration |
| FR-021 | Identify overdue invoices | SRS-SAL-002 | Integration, Cron |
| FR-022 | Lock issued invoices | SRS-SAL-002 | Unit, Integration |
| FR-023 | Record receipts & allocate | SRS-SAL-002 | Integration, E2E |
| FR-024 | Balance updates on receipt | SRS-SAL-002 | Unit, Integration |
| FR-025 | Deduplicate receipts | SRS-SAL-002 | Unit, Integration |
| FR-026 | Create Purchase Orders | SRS-PRO-001 | Integration, E2E |
| FR-027 | Require PO approval | SRS-PRO-001 | Integration, E2E |
| FR-028 | PO Status workflow | SRS-PRO-001 | Integration |
| FR-029 | PO -> invoice -> payment trace | SRS-PRO-001 | E2E |
| FR-030 | Procurement visibility | SRS-PRO-001 | Integration |
| FR-031 | Match invoice to PO | SRS-FIN-001 | Integration |
| FR-032 | Deduplicate purchase invoices | SRS-FIN-001 | Unit, Integration |
| FR-033 | Company expenses | SRS-FIN-002 | Integration |
| FR-034 | Reimbursement claims | SRS-FIN-002 | Integration, E2E |
| FR-035 | Mobile photo capture | SRS-FIN-002 | Mobile E2E |
| FR-036 | Claim status tracking | SRS-FIN-002 | Integration |
| FR-037 | Petty cash allocations | SRS-FIN-003 | Integration |
| FR-038 | Petty cash payment limits | SRS-FIN-003 | Unit |
| FR-039 | Outgoing payments | SRS-FIN-001 | Integration |
| FR-040 | Deduplicate payments | SRS-FIN-001 | Unit |
| FR-041 | Update payables ledger | SRS-FIN-001 | Unit, Integration |
| FR-042 | Currency exchange conversion | SRS-FIN-004 | Unit |
| FR-043 | VAT computation | SRS-FIN-004 | Unit, Integration |
| FR-044 | Cash flow report | SRS-FIN-004 | Integration |
| FR-045 | Project cost linking | SRS-FIN-004 | Integration |
| FR-046 | Configurable approval paths | SRS-WRK-001 | Unit, Integration |
| FR-047 | Route approvals by cost rules | SRS-WRK-001 | Integration |
| FR-048 | Submitter cannot approve | SRS-WRK-001 | E2E, Security |
| FR-049 | Capture decision metadata | SRS-WRK-001 | Integration |
| FR-050 | State actions (approve/reject/return) | SRS-WRK-001 | Unit, Integration |
| FR-051 | Approval event notifications | SRS-WRK-001 | Integration |
| FR-052 | Auto-escalation timeouts | SRS-WRK-001 | Integration, Cron |
| FR-053 | Document organization | SRS-DOC-001 | Integration |
| FR-054 | File attachments upload | SRS-DOC-001 | Integration, S3 |
| FR-055 | Retention policy enforcement | SRS-DOC-001 | Integration |
| FR-056 | Require attachments for invoices | SRS-DOC-001 | Integration |
| FR-057 | OCR pipeline trigger | SRS-DOC-002 | Integration |
| FR-058 | User reviews OCR results | SRS-DOC-002 | E2E |
| FR-059 | Flag low-confidence fields | SRS-DOC-002 | Unit, Integration |
| FR-060 | Support multi-language text | SRS-DOC-002 | Integration |
| FR-061 | Dashboard permissions | SRS-RPT-001 | E2E |
| FR-062 | Highlight action items | SRS-RPT-001 | E2E |
| FR-063 | Financial reports list | SRS-RPT-001 | Integration |
| FR-064 | Report variables | SRS-RPT-001 | Integration |
| FR-065 | Report downloads (Excel/PDF) | SRS-RPT-001 | Integration |
| FR-066 | Visual graphs & margins | SRS-RPT-001 | Integration |
| FR-067 | Event notification publisher | SRS-NTF-001 | Integration |
| FR-068 | UI app push | SRS-NTF-001 | E2E |
| FR-069 | Mail notification alerts | SRS-NTF-001 | Integration |
| FR-070 | Mobile OS alert push | SRS-NTF-001 | Mobile E2E |
| FR-071 | Security notification delivery | SRS-NTF-001 | Integration |
| FR-072 | User preference config | SRS-NTF-001 | Integration |
| FR-073 | Database triggers for audit logs | SRS-ADM-001 | SQL, Integration |
| FR-074 | Limit audit view permissions | SRS-ADM-001 | Security |
| FR-075 | Log queries (user/date/action) | SRS-ADM-001 | Integration |
| FR-076 | Edit parameters log history | SRS-ADM-001 | Integration |
| FR-077 | Verify cloud database backups | SRS-ADM-001 | Backup test |
| FR-078 | Language switcher | SRS-LOC-001 | E2E |
| FR-079 | RTL Arabic alignments | SRS-LOC-001 | E2E |
| FR-080 | Mixed text direction | SRS-LOC-001 | E2E |
| FR-081 | Localize mobile & web views | SRS-LOC-001 | E2E |
| FR-082 | AI text drafts suggestions | SRS-AIC-001 | Integration |
| FR-083 | AI category lookup suggestions | SRS-AIC-001 | Integration |
| FR-084 | AI receipt details extractor | SRS-AIC-001 | Integration |
| FR-085 | User edit/confirm interface | SRS-AIC-001 | E2E |
| FR-086 | Verify AI context permissions | SRS-AIC-001 | Integration |
| FR-087 | Force final user confirmation | SRS-AIC-001 | E2E |
| FR-088 | Logs accepted ratios metric | SRS-AIC-001 | Integration |

---

## 7. Appendix

### 7.1 Glossary
- **Transactional Outbox**: An outbox table within the local DB schema where events are written within the transaction block. A publisher process polls this outbox and sends events to a broker.
- **Idempotency Key**: A unique client-supplied header to ensure that a request (e.g. creating a payment) is processed exactly once even on retries.
- **Separation of Duties (SoD)**: A safety control preventing a single user from controlling multiple stages of a sensitive financial workflow (e.g. requesting and approving a payment).
- **Master Data**: Critical business objects (customers, products) referenced across transactional events, managed separately to preserve document history.
