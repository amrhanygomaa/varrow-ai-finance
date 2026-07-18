# VArrow AI Finance — REST API Specification

| | |
|---|---|
| **Document** | REST API Specification |
| **Status** | Final |
| **API Version** | v1 |
| **Protocol** | HTTPS / REST |
| **Architecture** | Modular Monolith (ADR-001) |
| **Date** | 2026-07-18 |

---

## About This Document

This document is the **Single Source of Truth** for every REST API endpoint exposed by the VArrow AI Finance backend. All frontend (React Web, Flutter Mobile) and integration implementations must conform to the contracts defined here.

### Governing References

| Document | Relevance |
|---|---|
| [System Architecture](../05-architecture/SYSTEM_ARCHITECTURE.md) | Bounded contexts, event contracts, cross-context interactions |
| [Database Schema](../06-database/database-schema.md) | Entity definitions, data types, constraints |
| [ERD](../06-database/ERD.md) | Relationships, aggregate boundaries |
| [ADR-001](../04-adr/ADR-001-technology-stack.md) | Technology stack (NestJS, PostgreSQL) |
| [ADR-002](../04-adr/ADR-002-authentication-strategy.md) | JWT authentication, refresh token rotation, RBAC |
| [ADR-003](../04-adr/ADR-003-event-bus-design.md) | Transactional Outbox, event naming, idempotency |
| [ADR-004](../04-adr/ADR-004-shared-catalog-ownership.md) | Catalog ownership, cross-context read models |

---

# Part I — Global API Standards

## 1. API Versioning

All endpoints are prefixed with `/api/v1`. Version is embedded in the URL path. When a breaking change is introduced, a new version (`/api/v2`) is published alongside the old version with a documented deprecation schedule.

```
Base URL: https://{host}/api/v1
```

---

## 2. Authentication & Authorization

Per [ADR-002](../04-adr/ADR-002-authentication-strategy.md):

- **Authentication:** Every API request (except `/api/v1/auth/login` and `/api/v1/auth/refresh`) must include a valid JWT access token in the `Authorization` header.
- **Header Format:** `Authorization: Bearer <access_token>`
- **Authorization:** Role-Based Access Control (RBAC). Each endpoint declares its required permissions as `resource:action` pairs. The server evaluates the `roles` claim from the JWT against the permission matrix.
- **Separation of Duties:** Enforced at the business logic layer by the Workflow Engine, not at the API gateway level.

---

## 3. Standard Request Headers

| Header | Required | Description |
|---|---|---|
| `Authorization` | Yes (except auth endpoints) | `Bearer <JWT>` |
| `Content-Type` | Yes (for request bodies) | `application/json` |
| `Accept-Language` | No | `en` or `ar`. Defaults to `en`. Controls response message language. |
| `X-Request-Id` | No | UUID. Client-generated. If absent, the server generates one. Propagated to all logs and events for tracing. |
| `X-Correlation-Id` | No | UUID. Used to correlate related requests across a multi-step workflow. |
| `Idempotency-Key` | Conditional | UUID. **Required** on all financial write operations (POST for invoices, receipts, payments, expenses). See §8. |

---

## 4. Standard Response Headers

| Header | Always Present | Description |
|---|---|---|
| `X-Request-Id` | Yes | Echoes or generates the request trace ID. |
| `X-Correlation-Id` | Yes (when provided) | Echoes the correlation ID. |
| `X-RateLimit-Limit` | Yes (on rate-limited endpoints) | Max requests per window. |
| `X-RateLimit-Remaining` | Yes (on rate-limited endpoints) | Remaining requests in current window. |
| `X-RateLimit-Reset` | Yes (on rate-limited endpoints) | ISO 8601 UTC timestamp when the window resets. |

---

## 5. Success Response Format

All successful responses use a consistent envelope:

```json
{
  "status": "success",
  "data": { },
  "meta": {
    "requestId": "uuid",
    "timestamp": "2026-07-18T10:00:00.000Z"
  }
}
```

For paginated lists:

```json
{
  "status": "success",
  "data": [],
  "meta": {
    "requestId": "uuid",
    "timestamp": "2026-07-18T10:00:00.000Z"
  },
  "pagination": {
    "totalCount": 1250,
    "pageSize": 25,
    "currentPage": 3,
    "totalPages": 50,
    "hasNextPage": true,
    "hasPreviousPage": true,
    "cursor": "eyJpZCI6Ijk..."
  }
}
```

---

## 6. Error Response Format (RFC 7807)

All error responses conform to [RFC 7807 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc7807):

```json
{
  "type": "https://api.varrow.tech/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "The 'email' field must be a valid email address.",
  "instance": "/api/v1/users",
  "requestId": "uuid",
  "timestamp": "2026-07-18T10:00:00.000Z",
  "errors": [
    {
      "field": "email",
      "code": "INVALID_FORMAT",
      "message": "Must be a valid email address."
    }
  ]
}
```

### Standard HTTP Status Codes

| Code | Usage |
|---|---|
| `200 OK` | Successful GET, PUT, PATCH |
| `201 Created` | Successful POST that creates a resource |
| `204 No Content` | Successful DELETE |
| `400 Bad Request` | Malformed request syntax |
| `401 Unauthorized` | Missing or invalid JWT |
| `403 Forbidden` | Authenticated but insufficient permissions |
| `404 Not Found` | Resource does not exist or is soft-deleted |
| `409 Conflict` | Optimistic locking conflict (`version` mismatch) or duplicate business key |
| `422 Unprocessable Entity` | Validation failure on request body |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Unexpected server error |

---

## 7. Pagination Standard

- **Default method:** Offset-based pagination for general use (`?page=1&pageSize=25`).
- **Preferred method for large datasets:** Cursor-based (keyset) pagination for Audit Logs, Notifications, and Event tables (`?cursor=<opaque_token>&pageSize=25`).
- **Default page size:** `25`
- **Maximum page size:** `100`
- **Minimum page size:** `1`

---

## 8. Idempotency

Financial write operations (creating invoices, receipts, payments, expenses, petty cash disbursements) **must** include an `Idempotency-Key` header (UUID).

- The server stores the key and the response for a configurable TTL (default: 24 hours).
- If a duplicate key is received, the server returns the original response without re-executing the operation.
- Missing the `Idempotency-Key` on a financial POST endpoint returns `400 Bad Request`.

---

## 9. Filtering, Sorting, and Search Standards

### Filtering

Query parameters use field-based filtering with operators:

```
GET /api/v1/sales/invoices?status=ISSUED&customer_id=uuid&due_date_gte=2026-01-01&due_date_lte=2026-12-31
```

| Suffix | Operator |
|---|---|
| (none) | Equals |
| `_gte` | Greater than or equal |
| `_lte` | Less than or equal |
| `_gt` | Greater than |
| `_lt` | Less than |
| `_ne` | Not equal |
| `_in` | In list (comma-separated) |

### Sorting

```
GET /api/v1/sales/invoices?sort=created_at:desc,invoice_number:asc
```

- Format: `field:direction` (comma-separated for multi-sort).
- Default sort: `created_at:desc`.

### Search

Full-text search across searchable fields:

```
GET /api/v1/catalog/products?search=laptop
```

- The `search` parameter performs a case-insensitive search across pre-defined searchable fields (e.g., `name`, `sku`, `description`).
- Backed by PostgreSQL GIN indexes on JSONB fields for multi-language search.

---

## 10. Date, Time & Currency Formats

| Concern | Format |
|---|---|
| Dates | ISO 8601: `YYYY-MM-DD` (e.g., `2026-07-18`) |
| Timestamps | ISO 8601 UTC: `YYYY-MM-DDTHH:mm:ss.sssZ` (e.g., `2026-07-18T10:00:00.000Z`) |
| Currency codes | ISO 4217 3-letter: `SAR`, `USD`, `EGP` |
| Monetary amounts | String representation of NUMERIC(19,4): `"1234.5600"` |

> **Monetary amounts are serialized as strings** in JSON to prevent floating-point precision loss in JavaScript clients.

---

## 11. File Upload & Download Standards

- **Upload:** `multipart/form-data` to the `/api/v1/documents/attachments/upload` endpoint.
- **Maximum file size:** 10 MB (configurable via Administration Settings).
- **Allowed MIME types:** `application/pdf`, `image/jpeg`, `image/png`, `image/webp`.
- **Download:** `GET /api/v1/documents/attachments/{id}/download` returns a pre-signed S3 URL with a short TTL (5 minutes).
- **Metadata:** File metadata (name, size, MIME type, S3 key) is stored in the database. The binary file is stored in S3.

---

## 12. API Naming Conventions

| Convention | Rule | Example |
|---|---|---|
| URL paths | `kebab-case`, plural nouns | `/api/v1/sales/sales-invoices` |
| Path parameters | `camelCase` | `{invoiceId}` |
| Query parameters | `snake_case` | `?customer_id=uuid&page_size=25` |
| Request/Response body fields | `camelCase` | `{ "invoiceNumber": "INV-001" }` |

---

## 13. Deprecation Strategy

1. Deprecated endpoints receive a `Sunset` header with the removal date: `Sunset: Sat, 01 Jan 2028 00:00:00 GMT`.
2. A `Deprecation` header is added: `Deprecation: true`.
3. Deprecated endpoints are documented in the changelog and remain functional for a minimum of 6 months.
4. After the sunset date, the endpoint returns `410 Gone`.

---

## 14. Rate Limiting

| Endpoint Category | Limit |
|---|---|
| Authentication (`/auth/login`) | 10 requests / 15 minutes per IP |
| AI endpoints | 30 requests / minute per user |
| File uploads | 20 requests / minute per user |
| General API | 120 requests / minute per user |

---

# Part II — Module API Specifications

---

## Module 1: Identity — Authentication

**Base Route:** `/api/v1/auth`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/login` | POST | No | Public | Authenticate with email/password. Returns access token + sets refresh cookie. |
| `/refresh` | POST | Cookie | Public | Rotate refresh token and issue new access token. |
| `/logout` | POST | Yes | Authenticated | Invalidate refresh token family. |
| `/password/change` | POST | Yes | Authenticated | Change own password. Invalidates all refresh tokens. |
| `/password/reset-request` | POST | No | Public | Request password reset email. |
| `/password/reset` | POST | No | Public | Reset password with token. |

### POST `/login`

**Request Body:**
```json
{
  "email": "user@varrow.tech",
  "password": "SecurePass123"
}
```

**Success Response (200):**
```json
{
  "status": "success",
  "data": {
    "accessToken": "eyJhbG...",
    "expiresIn": 900,
    "user": {
      "id": "uuid",
      "email": "user@varrow.tech",
      "firstName": "Ahmed",
      "lastName": "Hassan",
      "roles": ["ADMIN", "FINANCE_MANAGER"]
    }
  }
}
```

**Error Responses:** `401` (Invalid credentials), `423` (Account locked), `422` (Validation error).

**Business Rules:**
- Inactive/locked users are rejected regardless of credential validity (ADR-002 §4).
- Failed attempts increment rate-limit counter. After 5 failures in 15 minutes, the account is temporarily locked.

**Events Published:** `Identity.UserAuthenticated`, `Identity.AuthenticationFailed`

**Audit:** All login attempts (success and failure) are recorded.

---

## Module 2: Identity — Users

**Base Route:** `/api/v1/identity`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/users` | GET | Yes | `users:read` | List all users (paginated, filterable). |
| `/users/{userId}` | GET | Yes | `users:read` | Get user by ID. |
| `/users` | POST | Yes | `users:create` | Create a new user. |
| `/users/{userId}` | PUT | Yes | `users:update` | Update user profile. |
| `/users/{userId}` | DELETE | Yes | `users:delete` | Soft-delete user. Invalidates all refresh tokens. |
| `/users/{userId}/roles` | GET | Yes | `users:read` | List roles assigned to user. |
| `/users/{userId}/roles` | PUT | Yes | `users:assign-roles` | Replace user's role assignments. |
| `/users/me` | GET | Yes | Authenticated | Get current user's profile. |

### POST `/users`

**Request Body:**
```json
{
  "email": "newuser@varrow.tech",
  "firstName": "Sara",
  "lastName": "Ali",
  "password": "InitialPass123!",
  "roleIds": ["uuid-role-1", "uuid-role-2"]
}
```

**Validation Rules:**
- `email`: Required, valid email, unique.
- `firstName`, `lastName`: Required, 1-100 characters.
- `password`: Required, must meet configurable password policy (default: min 8 chars, 1 upper, 1 lower, 1 digit).

**Success Response (201):** Returns created user (without `passwordHash`).

**Events Published:** `Identity.UserCreated`

**Audit:** User creation is recorded with the creating user's identity.

### GET `/users`

**Query Parameters:** `?status=ACTIVE&search=ahmed&sort=created_at:desc&page=1&pageSize=25`

**Filtering:** `status` (`ACTIVE`, `INACTIVE`, `LOCKED`), `search` (email, name).

---

## Module 3: Identity — Roles & Permissions

**Base Route:** `/api/v1/identity`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/roles` | GET | Yes | `roles:read` | List all roles. |
| `/roles/{roleId}` | GET | Yes | `roles:read` | Get role with permissions. |
| `/roles` | POST | Yes | `roles:create` | Create a custom role. |
| `/roles/{roleId}` | PUT | Yes | `roles:update` | Update role name/description. |
| `/roles/{roleId}` | DELETE | Yes | `roles:delete` | Delete a non-system role. |
| `/roles/{roleId}/permissions` | GET | Yes | `roles:read` | List permissions for a role. |
| `/roles/{roleId}/permissions` | PUT | Yes | `roles:update` | Replace permissions for a role. |
| `/permissions/resources` | GET | Yes | `roles:read` | List all available resources and actions. |

**Business Rules:**
- System roles (`is_system = true`) cannot be modified or deleted. Returns `403`.
- Deleting a role that is assigned to users returns `409 Conflict`.

**Events Published:** `Identity.PermissionsChanged`

---

## Module 4: Catalog — Customers

**Base Route:** `/api/v1/catalog`
**Ownership:** Catalog Module (ADR-004). Write access restricted to catalog management roles.

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/customers` | GET | Yes | `customers:read` | List customers (paginated, filterable, searchable). |
| `/customers/{customerId}` | GET | Yes | `customers:read` | Get customer by ID. |
| `/customers` | POST | Yes | `customers:create` | Create a new customer. |
| `/customers/{customerId}` | PUT | Yes | `customers:update` | Update customer details. |
| `/customers/{customerId}` | DELETE | Yes | `customers:delete` | Soft-delete customer. |
| `/customers/{customerId}/contacts` | GET | Yes | `customers:read` | List contacts linked to customer. |

### POST `/customers`

**Request Body:**
```json
{
  "name": "Acme Corporation",
  "type": "CUSTOMER",
  "taxNumber": "300012345600003",
  "billingCurrency": "SAR",
  "status": "ACTIVE"
}
```

**Validation Rules:**
- `name`: Required, 1-255 characters.
- `type`: Required, one of `CUSTOMER`, `VENDOR`, `BOTH`.
- `taxNumber`: Required, unique.
- `billingCurrency`: Required, valid ISO 4217 code.

**Events Published:** `Catalog.CustomerCreated`

---

## Module 5: Catalog — Vendors

**Base Route:** `/api/v1/catalog`
**Ownership:** Catalog Module (ADR-004).

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/vendors` | GET | Yes | `vendors:read` | List vendors (paginated, filterable, searchable). |
| `/vendors/{vendorId}` | GET | Yes | `vendors:read` | Get vendor by ID. |
| `/vendors` | POST | Yes | `vendors:create` | Create a new vendor. |
| `/vendors/{vendorId}` | PUT | Yes | `vendors:update` | Update vendor. |
| `/vendors/{vendorId}` | DELETE | Yes | `vendors:delete` | Soft-delete vendor. |
| `/vendors/{vendorId}/contacts` | GET | Yes | `vendors:read` | List contacts linked to vendor. |

**Events Published:** `Catalog.VendorCreated`, `Catalog.VendorUpdated`

---

## Module 6: Catalog — Contacts

**Base Route:** `/api/v1/catalog`
**Ownership:** Catalog Module (ADR-004).

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/contacts` | GET | Yes | `contacts:read` | List contacts. |
| `/contacts/{contactId}` | GET | Yes | `contacts:read` | Get contact by ID. |
| `/contacts` | POST | Yes | `contacts:create` | Create a contact. |
| `/contacts/{contactId}` | PUT | Yes | `contacts:update` | Update contact. |
| `/contacts/{contactId}` | DELETE | Yes | `contacts:delete` | Soft-delete contact. |

**Events Published:** `Catalog.ContactCreated`, `Catalog.ContactUpdated`, `Catalog.ContactDeactivated`

---

## Module 7: Catalog — Products

**Base Route:** `/api/v1/catalog`
**Ownership:** Catalog Module (ADR-004). Only the Catalog module writes. Sales/Procurement consume via read models.

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/products` | GET | Yes | `products:read` | List products (paginated, filterable, searchable). |
| `/products/{productId}` | GET | Yes | `products:read` | Get product by ID. |
| `/products` | POST | Yes | `products:create` | Create a new product. |
| `/products/{productId}` | PUT | Yes | `products:update` | Update product. Requires `version` for optimistic locking. |
| `/products/{productId}` | DELETE | Yes | `products:delete` | Soft-delete (discontinue) product. |

### POST `/products`

**Request Body:**
```json
{
  "sku": "LAPTOP-001",
  "name": { "en": "Business Laptop", "ar": "لابتوب أعمال" },
  "description": { "en": "High-performance laptop", "ar": "لابتوب عالي الأداء" },
  "basePrice": "4500.0000",
  "isActive": true
}
```

**Validation Rules:**
- `sku`: Required, unique, 1-50 characters.
- `name`: Required JSONB object with at least `en` key.
- `basePrice`: Required, must be >= 0, string representation of NUMERIC(19,4).

### PUT `/products/{productId}`

**Request Body includes `version`:**
```json
{
  "sku": "LAPTOP-001",
  "name": { "en": "Business Laptop Pro", "ar": "لابتوب أعمال برو" },
  "basePrice": "5200.0000",
  "isActive": true,
  "version": 3
}
```

- If `version` does not match the current database version, returns `409 Conflict`.

**Events Published:** `Catalog.ProductCreated`, `Catalog.ProductUpdated`, `Catalog.ProductDiscontinued`

---

## Module 8: Catalog — Services

**Base Route:** `/api/v1/catalog`
**Ownership:** Catalog Module (ADR-004).

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/services` | GET | Yes | `services:read` | List services. |
| `/services/{serviceId}` | GET | Yes | `services:read` | Get service by ID. |
| `/services` | POST | Yes | `services:create` | Create a new service. |
| `/services/{serviceId}` | PUT | Yes | `services:update` | Update service. |
| `/services/{serviceId}` | DELETE | Yes | `services:delete` | Soft-delete (retire) service. |

**Events Published:** `Catalog.ServiceCreated`, `Catalog.ServiceUpdated`, `Catalog.ServiceRetired`

---

## Module 9: Sales — Quotations

**Base Route:** `/api/v1/sales`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/quotations` | GET | Yes | `quotations:read` | List quotations (paginated, filterable). |
| `/quotations/{quotationId}` | GET | Yes | `quotations:read` | Get quotation with items. |
| `/quotations` | POST | Yes | `quotations:create` | Create a draft quotation with items. |
| `/quotations/{quotationId}` | PUT | Yes | `quotations:update` | Update a DRAFT quotation. |
| `/quotations/{quotationId}` | DELETE | Yes | `quotations:delete` | Soft-delete a DRAFT quotation. |
| `/quotations/{quotationId}/submit` | POST | Yes | `quotations:submit` | Submit for approval. |
| `/quotations/{quotationId}/send` | POST | Yes | `quotations:send` | Mark as sent to customer. |
| `/quotations/{quotationId}/accept` | POST | Yes | `quotations:update` | Mark as accepted by customer. |
| `/quotations/{quotationId}/reject` | POST | Yes | `quotations:update` | Mark as rejected by customer. |
| `/quotations/{quotationId}/convert-to-invoice` | POST | Yes | `sales-invoices:create` | Create a sales invoice from this quotation. |

### POST `/quotations`

**Request Body:**
```json
{
  "customerId": "uuid",
  "currencyCode": "SAR",
  "validUntil": "2026-08-18",
  "items": [
    {
      "productId": "uuid",
      "quantity": "10.0000",
      "unitPrice": "450.0000"
    },
    {
      "serviceId": "uuid",
      "quantity": "1.0000",
      "unitPrice": "2000.0000"
    }
  ]
}
```

**Business Rules:**
- Only DRAFT quotations can be edited or deleted.
- Each item must reference either `productId` or `serviceId`, not both.
- `totalAmount` is computed server-side from items.
- Product/Service data is snapshotted at creation time (immutable price reference).

**Filtering:** `?status=DRAFT&customer_id=uuid&valid_until_gte=2026-01-01`

**Events Published:** `Sales.QuotationCreated`, `Sales.QuotationApproved`, `Sales.QuotationAccepted`, `Sales.QuotationExpired`

---

## Module 10: Sales — Sales Invoices

**Base Route:** `/api/v1/sales`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/sales-invoices` | GET | Yes | `sales-invoices:read` | List sales invoices (paginated, filterable). |
| `/sales-invoices/{invoiceId}` | GET | Yes | `sales-invoices:read` | Get invoice with items. |
| `/sales-invoices` | POST | Yes | `sales-invoices:create` | Create a draft sales invoice. Requires `Idempotency-Key`. |
| `/sales-invoices/{invoiceId}` | PUT | Yes | `sales-invoices:update` | Update a DRAFT invoice. |
| `/sales-invoices/{invoiceId}/issue` | POST | Yes | `sales-invoices:issue` | Issue the invoice (transitions to ISSUED, immutable). |
| `/sales-invoices/{invoiceId}/cancel` | POST | Yes | `sales-invoices:cancel` | Cancel an issued invoice. |

### POST `/sales-invoices`

**Request Body:**
```json
{
  "customerId": "uuid",
  "quotationId": "uuid (optional)",
  "currencyCode": "SAR",
  "dueDate": "2026-08-18",
  "items": [
    {
      "productId": "uuid",
      "description": "Business Laptop",
      "quantity": "10.0000",
      "unitPrice": "450.0000",
      "taxRate": "0.1500"
    }
  ]
}
```

**Business Rules:**
- `invoiceNumber` is auto-generated (server-side sequential).
- Once ISSUED, the invoice is immutable. No edits allowed.
- `subtotal`, `taxTotal`, `grandTotal` are computed server-side.
- `taxRate` and `taxAmount` per item are derived from the Finance Tax module.

**Idempotency:** Required via `Idempotency-Key` header.

**Events Published:** `Sales.SalesInvoiceIssued`, `Sales.SalesInvoiceOverdue`, `Sales.SalesInvoicePaid`

**Events Consumed:** `Workflow.ApprovalDecisionMade`, `Finance.TaxCalculated`, `Finance.CurrencyRateResolved`

---

## Module 11: Sales — Receipts

**Base Route:** `/api/v1/sales`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/receipts` | GET | Yes | `receipts:read` | List receipts. |
| `/receipts/{receiptId}` | GET | Yes | `receipts:read` | Get receipt by ID. |
| `/receipts` | POST | Yes | `receipts:create` | Record a customer payment. Requires `Idempotency-Key`. |
| `/receipts/{receiptId}` | PUT | Yes | `receipts:update` | Update a receipt (before reconciliation). |

### POST `/receipts`

**Request Body:**
```json
{
  "customerId": "uuid",
  "invoiceId": "uuid",
  "amount": "4500.0000",
  "currencyCode": "SAR",
  "paymentMethod": "BANK_TRANSFER",
  "referenceNumber": "TXN-12345",
  "receiptDate": "2026-07-18"
}
```

**Business Rules:**
- `amount` must not exceed the remaining balance on the referenced invoice.
- When the invoice balance reaches zero, the system publishes `Sales.SalesInvoicePaid`.

**Idempotency:** Required.

**Events Published:** `Sales.ReceiptRecorded`

---

## Module 12: Procurement — Purchase Orders

**Base Route:** `/api/v1/procurement`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/purchase-orders` | GET | Yes | `purchase-orders:read` | List POs (paginated, filterable). |
| `/purchase-orders/{poId}` | GET | Yes | `purchase-orders:read` | Get PO with items. |
| `/purchase-orders` | POST | Yes | `purchase-orders:create` | Create a draft PO. |
| `/purchase-orders/{poId}` | PUT | Yes | `purchase-orders:update` | Update a DRAFT PO. |
| `/purchase-orders/{poId}` | DELETE | Yes | `purchase-orders:delete` | Soft-delete a DRAFT PO. |
| `/purchase-orders/{poId}/submit` | POST | Yes | `purchase-orders:submit` | Submit for approval. |
| `/purchase-orders/{poId}/issue` | POST | Yes | `purchase-orders:issue` | Issue PO to vendor. |
| `/purchase-orders/{poId}/cancel` | POST | Yes | `purchase-orders:cancel` | Cancel PO. |

**Business Rules:**
- Only DRAFT POs can be edited or deleted.
- `poNumber` is auto-generated.
- Issuing a PO triggers the Finance context to prepare for expected vendor invoices.

**Events Published:** `Procurement.PurchaseOrderCreated`, `Procurement.PurchaseOrderApproved`, `Procurement.PurchaseOrderIssued`

**Events Consumed:** `Workflow.ApprovalDecisionMade`, `Finance.PurchaseInvoiceRecorded`, `Finance.PaymentCompleted`

---

## Module 13: Finance — Purchase Invoices

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/purchase-invoices` | GET | Yes | `purchase-invoices:read` | List purchase invoices. |
| `/purchase-invoices/{invoiceId}` | GET | Yes | `purchase-invoices:read` | Get purchase invoice with items. |
| `/purchase-invoices` | POST | Yes | `purchase-invoices:create` | Record a vendor invoice. Requires `Idempotency-Key`. |
| `/purchase-invoices/{invoiceId}` | PUT | Yes | `purchase-invoices:update` | Update before approval. |
| `/purchase-invoices/{invoiceId}/submit` | POST | Yes | `purchase-invoices:submit` | Submit for approval. |
| `/purchase-invoices/{invoiceId}/approve` | POST | Yes | `purchase-invoices:approve` | Approve for payment. |

**Business Rules:**
- Must reference a valid `vendorId`.
- Optionally references a `poId` for three-way matching.
- Must be approved via Workflow Engine before payment can proceed.

**Idempotency:** Required.

**Events Published:** `Finance.PurchaseInvoiceRecorded`, `Finance.PurchaseInvoiceApproved`

---

## Module 14: Finance — Payments

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/payments` | GET | Yes | `payments:read` | List payments. |
| `/payments/{paymentId}` | GET | Yes | `payments:read` | Get payment by ID. |
| `/payments` | POST | Yes | `payments:create` | Record an outgoing payment. Requires `Idempotency-Key`. |

### POST `/payments`

**Request Body:**
```json
{
  "vendorId": "uuid",
  "invoiceId": "uuid",
  "amount": "15000.0000",
  "currencyCode": "SAR",
  "paymentMethod": "BANK_TRANSFER",
  "referenceNumber": "PAY-67890",
  "paymentDate": "2026-07-18"
}
```

**Business Rules:**
- The referenced purchase invoice must be in `APPROVED` status.
- `amount` must not exceed the remaining balance.

**Idempotency:** Required.

**Events Published:** `Finance.PaymentCompleted`

---

## Module 15: Finance — Expenses & Expense Categories

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/expenses` | GET | Yes | `expenses:read` | List expenses. |
| `/expenses/{expenseId}` | GET | Yes | `expenses:read` | Get expense by ID. |
| `/expenses` | POST | Yes | `expenses:create` | Record an expense. Requires `Idempotency-Key`. |
| `/expenses/{expenseId}` | PUT | Yes | `expenses:update` | Update expense. |

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/expense-categories` | GET | Yes | `expense-categories:read` | List categories. |
| `/expense-categories` | POST | Yes | `expense-categories:create` | Create category. |
| `/expense-categories/{categoryId}` | PUT | Yes | `expense-categories:update` | Update category. |
| `/expense-categories/{categoryId}` | DELETE | Yes | `expense-categories:delete` | Delete (restricted if expenses reference it). |

**Events Published:** `Finance.ExpenseRecorded`, `Finance.ExpenseApproved`

---

## Module 16: Finance — Employee Expenses

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/employee-expenses` | GET | Yes | `employee-expenses:read` | List employee expenses. |
| `/employee-expenses/{id}` | GET | Yes | `employee-expenses:read` | Get by ID. |
| `/employee-expenses` | POST | Yes | `employee-expenses:create` | Submit expense claim. Requires `Idempotency-Key`. |
| `/employee-expenses/{id}/approve` | POST | Yes | `employee-expenses:approve` | Approve claim. |
| `/employee-expenses/{id}/reimburse` | POST | Yes | `employee-expenses:reimburse` | Mark as reimbursed. |

**Events Published:** `Finance.EmployeeExpenseSubmitted`, `Finance.EmployeeExpenseReimbursed`

---

## Module 17: Finance — Petty Cash

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/petty-cash` | GET | Yes | `petty-cash:read` | List petty cash funds. |
| `/petty-cash/{fundId}` | GET | Yes | `petty-cash:read` | Get fund details with ledger. |
| `/petty-cash` | POST | Yes | `petty-cash:create` | Create a petty cash fund. |
| `/petty-cash/{fundId}/disburse` | POST | Yes | `petty-cash:disburse` | Record a disbursement. Requires `Idempotency-Key`. |
| `/petty-cash/{fundId}/replenish` | POST | Yes | `petty-cash:replenish` | Replenish fund. Requires `Idempotency-Key`. |
| `/petty-cash/{fundId}/reconcile` | POST | Yes | `petty-cash:reconcile` | Reconcile fund. |
| `/petty-cash/{fundId}/ledger` | GET | Yes | `petty-cash:read` | Get append-only ledger entries (cursor pagination). |

**Business Rules:**
- Ledger is append-only. No modifications or deletions.
- Disbursement cannot exceed current balance.

**Events Published:** `Finance.PettyCashDisbursed`, `Finance.PettyCashReconciled`

---

## Module 18: Finance — Cash Flow

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/cash-flow` | GET | Yes | `cash-flow:read` | Get cash flow summary (filterable by date range, currency). |
| `/cash-flow/daily` | GET | Yes | `cash-flow:read` | Get daily inflow/outflow breakdown. |

**Business Rules:**
- Read-only endpoint. Data is derived from the Materialized View aggregating receipts and payments.
- No write operations.

**Query Parameters:** `?from=2026-01-01&to=2026-07-18&currency_code=SAR`

---

## Module 19: Finance — Currencies & Exchange Rates

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/currencies` | GET | Yes | `currencies:read` | List supported currencies. |
| `/currencies` | POST | Yes | `currencies:create` | Add a currency. |

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/exchange-rates` | GET | Yes | `exchange-rates:read` | List exchange rates. |
| `/exchange-rates` | POST | Yes | `exchange-rates:create` | Set an exchange rate for a date. |
| `/exchange-rates/convert` | GET | Yes | `exchange-rates:read` | Convert amount between currencies. |

### GET `/exchange-rates/convert`

**Query Parameters:** `?from=USD&to=SAR&amount=1000.0000&date=2026-07-18`

**Events Published:** `Finance.CurrencyRateResolved`

---

## Module 20: Finance — Taxes

**Base Route:** `/api/v1/finance`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/taxes` | GET | Yes | `taxes:read` | List tax configurations. |
| `/taxes/{taxId}` | GET | Yes | `taxes:read` | Get tax by ID. |
| `/taxes` | POST | Yes | `taxes:create` | Create a tax rule. |
| `/taxes/{taxId}` | PUT | Yes | `taxes:update` | Update a tax rule. |
| `/taxes/{taxId}` | DELETE | Yes | `taxes:delete` | Deactivate a tax rule. |

**Events Published:** `Finance.TaxCalculated`

---

## Module 21: Projects — Projects & Profitability

**Base Route:** `/api/v1/projects`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/` | GET | Yes | `projects:read` | List projects. |
| `/{projectId}` | GET | Yes | `projects:read` | Get project details. |
| `/` | POST | Yes | `projects:create` | Create a project. |
| `/{projectId}` | PUT | Yes | `projects:update` | Update project. |
| `/{projectId}/profitability` | GET | Yes | `projects:read` | Get profitability analysis (revenue vs. costs). |

**Business Rules:**
- Profitability is computed by aggregating invoices, receipts, expenses, and POs linked to the project.
- Read-only profitability view derived from materialized data.

---

## Module 22: Workflow — Approval Workflows

**Base Route:** `/api/v1/workflow`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/approval-workflows` | GET | Yes | `workflows:read` | List approval workflow definitions. |
| `/approval-workflows/{workflowId}` | GET | Yes | `workflows:read` | Get workflow with steps. |
| `/approval-workflows` | POST | Yes | `workflows:create` | Create a workflow definition. |
| `/approval-workflows/{workflowId}` | PUT | Yes | `workflows:update` | Update workflow and steps. |
| `/approval-workflows/{workflowId}` | DELETE | Yes | `workflows:delete` | Delete workflow. |

**Base Route:** `/api/v1/workflow`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/approval-requests` | GET | Yes | `approvals:read` | List pending approval requests for current user. |
| `/approval-requests/{requestId}` | GET | Yes | `approvals:read` | Get approval request details. |
| `/approval-requests/{requestId}/approve` | POST | Yes | `approvals:decide` | Approve the request. |
| `/approval-requests/{requestId}/reject` | POST | Yes | `approvals:decide` | Reject with reason. |
| `/approval-requests/{requestId}/return` | POST | Yes | `approvals:decide` | Return for revision. |

**Business Rules:**
- Separation of duties: submitter cannot be the approver (ADR-002, BR-04).
- Approval steps are sequential by `stepOrder`.

**Events Published:** `Workflow.ApprovalRequested`, `Workflow.ApprovalDecisionMade`, `Workflow.ApprovalEscalated`, `Workflow.WorkflowCompleted`

---

## Module 23: Documents — Attachments & OCR

**Base Route:** `/api/v1/documents`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/attachments/upload` | POST | Yes | `attachments:create` | Upload a file. `multipart/form-data`. |
| `/attachments/{attachmentId}` | GET | Yes | `attachments:read` | Get attachment metadata. |
| `/attachments/{attachmentId}/download` | GET | Yes | `attachments:read` | Get pre-signed S3 download URL. |
| `/attachments` | GET | Yes | `attachments:read` | List attachments for a record. Filter by `module_reference` and `record_id`. |
| `/attachments/{attachmentId}` | DELETE | Yes | `attachments:delete` | Soft-delete attachment. |
| `/ocr/{attachmentId}` | GET | Yes | `ocr:read` | Get OCR extraction results. |
| `/ocr/{attachmentId}/verify` | POST | Yes | `ocr:verify` | Mark OCR result as verified by human. |

### POST `/attachments/upload`

**Content-Type:** `multipart/form-data`

**Form Fields:**
- `file`: The file binary.
- `moduleReference`: e.g., `PurchaseInvoice`, `Expense`.
- `recordId`: UUID of the parent business record.

**Validation Rules:**
- Max file size: 10 MB.
- Allowed MIME types: `application/pdf`, `image/jpeg`, `image/png`, `image/webp`.

**Events Published:** `Document.DocumentUploaded`, `Document.DocumentProcessed`, `Document.ExtractionReviewRequired`

---

## Module 24: Notifications

**Base Route:** `/api/v1/notifications`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/` | GET | Yes | Authenticated | List current user's notifications (cursor pagination). |
| `/unread-count` | GET | Yes | Authenticated | Get count of unread notifications. |
| `/{notificationId}/read` | POST | Yes | Authenticated | Mark as read. |
| `/read-all` | POST | Yes | Authenticated | Mark all as read. |
| `/preferences` | GET | Yes | Authenticated | Get user notification preferences. |
| `/preferences` | PUT | Yes | Authenticated | Update notification preferences. |

---

## Module 25: Administration — Audit Logs

**Base Route:** `/api/v1/admin`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/audit-logs` | GET | Yes | `audit-logs:read` | List audit log entries (cursor pagination). |
| `/audit-logs/{logId}` | GET | Yes | `audit-logs:read` | Get single audit entry. |

**Business Rules:**
- Read-only API. No POST, PUT, or DELETE endpoints exist.
- Audit logs are written internally by event consumers, never by API clients.
- Cursor-based pagination only (high-volume table).

**Query Parameters:** `?user_id=uuid&action=CREATE&resource=SalesInvoice&from=2026-01-01&to=2026-07-18&cursor=xxx&pageSize=50`

---

## Module 26: Administration — Settings

**Base Route:** `/api/v1/admin`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/settings` | GET | Yes | `settings:read` | List all settings. |
| `/settings/{key}` | GET | Yes | `settings:read` | Get setting by key. |
| `/settings/{key}` | PUT | Yes | `settings:update` | Update a setting value. |

**Business Rules:**
- Settings are key-value pairs with JSONB values.
- Updating a setting publishes `Administration.SettingsUpdated` which is consumed by all contexts.

**Events Published:** `Administration.SettingsUpdated`

---

## Module 27: AI Center

**Base Route:** `/api/v1/ai`

| Endpoint | Method | Auth | Permissions | Description |
|---|---|---|---|---|
| `/conversations` | GET | Yes | `ai:read` | List user's AI conversations. |
| `/conversations` | POST | Yes | `ai:create` | Start a new AI conversation. |
| `/conversations/{conversationId}` | GET | Yes | `ai:read` | Get conversation with messages. |
| `/conversations/{conversationId}/messages` | POST | Yes | `ai:create` | Send a message (prompt) in conversation. |
| `/conversations/{conversationId}/messages/{messageId}/accept` | POST | Yes | `ai:decide` | Accept AI suggestion. |
| `/conversations/{conversationId}/messages/{messageId}/reject` | POST | Yes | `ai:decide` | Reject AI suggestion. |
| `/prompts` | GET | Yes | `ai:read` | List available prompt templates. |
| `/prompts/{promptId}` | GET | Yes | `ai:read` | Get prompt template. |
| `/logs` | GET | Yes | `ai-logs:read` | List AI usage logs (admin). Cursor pagination. |

**Rate Limit:** 30 requests / minute per user.

**Business Rules:**
- All AI outputs are presented for human review, never autonomous (System Architecture 7).
- Token usage and latency are logged for cost monitoring.

**Events Published:** `AI.AIAssistanceRequested`, `AI.AISuggestionReady`, `AI.AIAssistanceAccepted`, `AI.AIAssistanceRejected`

---

# Part III — Event Contract Summary

The following table consolidates all domain events published and consumed across the platform, per [ADR-003](../04-adr/ADR-003-event-bus-design.md) and [System Architecture](../05-architecture/SYSTEM_ARCHITECTURE.md):

| Event | Publisher | Consumers |
|---|---|---|
| `Identity.UserAuthenticated` | Auth | Administration, Reporting |
| `Identity.AuthenticationFailed` | Auth | Administration, Notification |
| `Identity.UserCreated` | Users | Notification, Administration |
| `Identity.UserUpdated` | Users | Notification, Administration |
| `Identity.UserDeactivated` | Users | Workflow, Administration, Notification |
| `Identity.RoleAssigned` | Roles | Workflow, Administration |
| `Identity.PermissionsChanged` | Roles | Workflow, Administration |
| `Identity.SuspiciousActivityDetected` | System | Notification, Administration |
| `Catalog.ProductCreated` | Products | Sales (read-model), Procurement (read-model), Administration |
| `Catalog.ProductUpdated` | Products | Sales, Procurement, Administration |
| `Catalog.ProductDiscontinued` | Products | Sales, Procurement, Administration |
| `Catalog.ServiceCreated` | Services | Sales, Procurement, Administration |
| `Catalog.ServiceUpdated` | Services | Sales, Procurement, Administration |
| `Catalog.ServiceRetired` | Services | Sales, Procurement, Administration |
| `Catalog.CustomerCreated` | Customers | Notification, Administration |
| `Catalog.CustomerUpdated` | Customers | Sales, Procurement, Administration |
| `Catalog.VendorCreated` | Vendors | Notification, Administration |
| `Catalog.VendorUpdated` | Vendors | Sales, Procurement, Administration |
| `Catalog.ContactCreated` | Contacts | Administration |
| `Catalog.ContactUpdated` | Contacts | Administration |
| `Catalog.ContactDeactivated` | Contacts | Administration |
| `Sales.QuotationCreated` | Quotations | Workflow, Notification, Administration |
| `Sales.QuotationApproved` | Quotations | Notification, Reporting |
| `Sales.QuotationAccepted` | Quotations | Notification, Reporting |
| `Sales.QuotationExpired` | Quotations | Notification, Reporting |
| `Sales.SalesInvoiceIssued` | Invoices | Finance, Workflow, Notification, Reporting, Administration |
| `Sales.SalesInvoiceOverdue` | Invoices | Notification, Reporting, Administration |
| `Sales.SalesInvoicePaid` | Receipts | Finance, Reporting, Notification |
| `Sales.ReceiptRecorded` | Receipts | Finance, Reporting, Notification |
| `Procurement.PurchaseOrderCreated` | POs | Workflow, Notification, Administration |
| `Procurement.PurchaseOrderApproved` | POs | Notification, Finance, Reporting |
| `Procurement.PurchaseOrderIssued` | POs | Finance, Notification, Reporting, Administration |
| `Finance.PurchaseInvoiceRecorded` | Purchase Invoices | Procurement, Workflow, Notification, Reporting, Administration |
| `Finance.PurchaseInvoiceApproved` | Purchase Invoices | Payments, Notification, Reporting |
| `Finance.PaymentCompleted` | Payments | Procurement, Sales, Reporting, Notification, Administration |
| `Finance.ExpenseRecorded` | Expenses | Workflow, Notification, Administration |
| `Finance.ExpenseApproved` | Expenses | Notification, Reporting |
| `Finance.EmployeeExpenseSubmitted` | Employee Expenses | Workflow, Notification |
| `Finance.EmployeeExpenseReimbursed` | Employee Expenses | Notification, Reporting |
| `Finance.PettyCashDisbursed` | Petty Cash | Notification, Administration |
| `Finance.PettyCashReconciled` | Petty Cash | Notification, Administration |
| `Finance.CashPositionChanged` | Cash Flow | Notification, Reporting |
| `Finance.TaxCalculated` | Taxes | Sales, Reporting |
| `Finance.CurrencyRateResolved` | Exchange Rates | Sales, Reporting |
| `Workflow.ApprovalRequested` | Workflows | Notification |
| `Workflow.ApprovalDecisionMade` | Workflows | Sales, Procurement, Finance, Notification, Administration |
| `Workflow.ApprovalEscalated` | Workflows | Notification, Administration |
| `Workflow.WorkflowCompleted` | Workflows | Sales, Procurement, Finance, Notification, Reporting |
| `Document.DocumentUploaded` | Attachments | Administration |
| `Document.DocumentProcessed` | OCR | Sales, Finance, Notification |
| `Document.ExtractionReviewRequired` | OCR | Notification, Administration |
| `AI.AIAssistanceRequested` | AI Center | Administration, AI Log |
| `AI.AISuggestionReady` | AI Center | Sales, Procurement, Finance, Document Engine, Reporting |
| `AI.AIAssistanceAccepted` | AI Center | Administration, Reporting, AI Log |
| `AI.AIAssistanceRejected` | AI Center | Administration, AI Log |
| `Administration.SettingsUpdated` | Settings | Identity, Workflow, Document Engine, AI, Reporting, Notification |

---

# Part IV — OpenAPI 3.1 Structure Recommendation

The following describes the recommended organization of the future `openapi.yaml` specification file. The YAML itself is NOT generated in this document.

## Recommended File Structure

```
docs/07-api/
  api-specification.md          <-- This document (source of truth)
  openapi/
    openapi.yaml              <-- Root spec file (refs all components)
    info.yaml                 <-- API metadata, contact, license
    security.yaml             <-- Security schemes (Bearer JWT)
    paths/
      auth.yaml
      identity-users.yaml
      identity-roles.yaml
      catalog-customers.yaml
      catalog-vendors.yaml
      catalog-contacts.yaml
      catalog-products.yaml
      catalog-services.yaml
      sales-quotations.yaml
      sales-invoices.yaml
      sales-receipts.yaml
      procurement-purchase-orders.yaml
      finance-purchase-invoices.yaml
      finance-payments.yaml
      finance-expenses.yaml
      finance-employee-expenses.yaml
      finance-petty-cash.yaml
      finance-cash-flow.yaml
      finance-currencies.yaml
      finance-exchange-rates.yaml
      finance-taxes.yaml
      projects.yaml
      workflow-definitions.yaml
      workflow-approvals.yaml
      documents.yaml
      notifications.yaml
      admin-audit-logs.yaml
      admin-settings.yaml
      ai.yaml
    schemas/
      common/
        pagination.yaml
        error.yaml             <-- RFC 7807 Problem Details
        success-envelope.yaml
        base-entity.yaml       <-- id, timestamps, version
      identity/
        user.yaml
        role.yaml
        permission.yaml
      catalog/
        customer.yaml
        vendor.yaml
        contact.yaml
        product.yaml
        service.yaml
      sales/
        quotation.yaml
        quotation-item.yaml
        sales-invoice.yaml
        sales-invoice-item.yaml
        receipt.yaml
      procurement/
        purchase-order.yaml
        purchase-order-item.yaml
      finance/
        purchase-invoice.yaml
        purchase-invoice-item.yaml
        payment.yaml
        expense.yaml
        expense-category.yaml
        employee-expense.yaml
        petty-cash.yaml
        cash-flow.yaml
        currency.yaml
        exchange-rate.yaml
        tax.yaml
      workflow/
        approval-workflow.yaml
        approval-step.yaml
        approval-request.yaml
      documents/
        attachment.yaml
        ocr-result.yaml
      admin/
        audit-log.yaml
        setting.yaml
      notifications/
        notification.yaml
      projects/
        project.yaml
        project-profitability.yaml
      ai/
        conversation.yaml
        prompt.yaml
        ai-log.yaml
    parameters/
      pagination.yaml          <-- page, pageSize, cursor
      filtering.yaml           <-- Common filter params
      sorting.yaml             <-- sort param
```

### Design Principles for the OpenAPI Structure

1. **Modular by Bounded Context:** Path files and schema files are organized by their owning bounded context, mirroring the System Architecture.
2. **$ref Composition:** The root `openapi.yaml` uses `$ref` to compose all path and schema files, keeping each file small and maintainable.
3. **Shared Components:** Common schemas (Base Entity, Pagination, Error) are in a `common/` directory and referenced by all modules.
4. **One Schema Per Entity:** Each database entity from the Database Schema maps to exactly one OpenAPI schema file.
5. **Request vs. Response Schemas:** Each entity schema file defines both `CreateRequest`, `UpdateRequest`, and `Response` variants (using `allOf` composition from the base entity).
6. **Security Scheme:** Defined once in `security.yaml` as `bearerAuth` (JWT) and applied globally with per-endpoint overrides for public endpoints.

---

*This document is the Single Source of Truth for the VArrow AI Finance REST API. All implementations must conform to these contracts. Changes must be reviewed and approved before implementation.*
