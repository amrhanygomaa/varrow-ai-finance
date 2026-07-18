# ADR-004: Shared Catalog Ownership

| | |
|---|---|
| **ADR ID** | ADR-004 |
| **Title** | Shared Catalog Ownership and Synchronization |
| **Status** | Proposed |
| **Author** | Architecture Team |
| **Date** | 2026-07-17 |
| **Project** | VArrow AI Finance |
| **Company** | VArrow.tech |

---

## Context

The VArrow AI Finance platform operates across 10 bounded contexts as defined in the [System Architecture](../05-architecture/SYSTEM_ARCHITECTURE.md). Three critical business modules are mapped as cross-context shared concerns:
1. **Contacts** (Shared between Sales and Procurement)
2. **Products** (Shared Catalog)
3. **Services** (Shared Catalog)

The System Architecture specifies that "neither context mutates the other's master data" and "each context holds a read-optimized local view of the catalog entries it needs." However, the architecture did not define which context physically owns the writes, database schemas, and APIs for these shared modules.

This ADR resolves this ambiguity by explicitly defining the domain boundaries, ownership, and synchronization strategy for Products, Services, and Contacts, ensuring consistency with DDD principles and the event bus design established in [ADR-003](ADR-003-event-bus-design.md).

---

## Decision

We will implement Products, Services, and Contacts as a **Shared Kernel / Master Data** module. While logically shared between the Sales and Procurement contexts, it will be physically implemented as an independent authoritative source within the modular monolith.

### 1. Ownership Definitions

| Module | Ownership |
|---|---|
| **Products** | **Master Data (Shared Kernel)** |
| **Services** | **Master Data (Shared Kernel)** |
| **Contacts** | **Master Data (Shared Kernel)** |

- **Write Ownership:** A dedicated `CatalogModule` (representing the Master Data Shared Kernel) exclusively owns all write operations (Create, Update, Deactivate) for these three entities.
- **Read-Only Access:** The Sales and Procurement contexts are downstream consumers. They have **read-only** access to this data and are not permitted to mutate the master records.

### 2. API Ownership

- The `CatalogModule` exposes its own REST API endpoints (e.g., `/api/catalog/products`, `/api/catalog/contacts`).
- Only users with appropriate Master Data / Catalog management roles (per Identity context) can access these write APIs.
- Sales and Procurement APIs (e.g., `/api/sales/quotations`) accept Product IDs and Contact IDs as references but do not handle the creation of the underlying catalog entities.

### 3. Database Ownership

- The master records reside in a dedicated database schema (e.g., `catalog` schema in PostgreSQL).
- **Only the `CatalogModule` is permitted to write to the `catalog` schema.**
- Sales and Procurement modules **must not** perform SQL `JOIN`s directly against the `catalog` schema tables to prevent tight database coupling.

### 4. Aggregate Boundaries

The aggregates are strictly separated:
- **Catalog Aggregates:** `Product`, `Service`, `Contact`. These are the authoritative entities containing all master data attributes (SKU, master descriptions, global status).
- **Sales Aggregates:** `Quotation`, `SalesInvoice`. These aggregates hold local Value Objects (e.g., `ProductReference`) that store the `productId` and an immutable snapshot of the product details (like name and price at the time of sale) to protect historical financial records from future catalog changes.
- **Procurement Aggregates:** `PurchaseOrder`, `Vendor`. These hold similar local references to Contacts and Products.

---

## Cross-Context Synchronization Strategy

To fulfill the architectural requirement that "each context holds a read-optimized local view," we will use **Event-Carried State Transfer** via the internal event bus ([ADR-003](ADR-003-event-bus-design.md)).

### The Synchronization Flow:
1. An administrator creates a new Product via the Catalog API.
2. The `CatalogModule` saves the `Product` to the `catalog.products` table and simultaneously writes a `Catalog.ProductCreated` event to the outbox (transactionally).
3. The Event Bus Relay publishes the event.
4. The **Sales Context** consumes `Catalog.ProductCreated` and upserts a localized, read-optimized projection into its own schema (e.g., `sales.catalog_products_view`).
5. The **Procurement Context** does the same for its own schema (e.g., `procurement.catalog_products_view`).

### Domain Events

The `CatalogModule` will publish the following Domain Events to power this synchronization:

**Products & Services:**
- `Catalog.ProductCreated` / `Catalog.ServiceCreated`
- `Catalog.ProductUpdated` / `Catalog.ServiceUpdated`
- `Catalog.ProductDiscontinued` / `Catalog.ServiceRetired` (Fulfilling BRD FR-012, FR-013)

**Contacts:**
- `Catalog.ContactCreated`
- `Catalog.ContactUpdated`
- `Catalog.ContactDeactivated`

*(Event payloads will adhere to the standard defined in ADR-003, including `version`, `requestId`, and the entity snapshot).*

---

## Future Microservices Migration Impact

This design heavily favors a seamless migration to microservices:
1. **Zero Database Coupling:** Because Sales and Procurement maintain their own read-models and do not cross-join to the catalog schema, extracting Sales or Procurement into a separate microservice requires **no database refactoring**.
2. **Network Readiness:** The interaction is already asynchronous (via events). If the monolith is split, the local EventBus is simply replaced by a distributed broker (as defined in ADR-003), and the event-carried state transfer continues exactly as before over the network.
3. **Independent Catalog Service:** The Master Data module can be extracted into a standalone `Catalog Service` trivially, as it already owns its own API, database schema, and event publishing lifecycle.

---

## Alternatives Considered

### 1. Sales Owns Selling Items, Procurement Owns Buying Items
*Description:* Split the catalog. Sales owns products sold to customers; Procurement owns items bought from vendors.
*Why Rejected:* Many businesses buy and sell the same items (trading goods). Splitting ownership would require complex reconciliation or duplication of master data, violating the single source of truth principle required by the business.

### 2. Synchronous Cross-Module Queries (API Composition)
*Description:* Instead of event-carried state transfer and local read models, Sales synchronously calls a `CatalogService.getProductById()` internal method when creating an invoice or running a report.
*Why Rejected:* While acceptable in a monolith, synchronous queries create tight runtime coupling. If the catalog goes down (in a microservices future), Sales cannot function. Furthermore, querying catalog details during heavy reporting across millions of invoice rows would cause severe performance bottlenecks compared to querying a local read-optimized table.

### 3. Direct Database Joins
*Description:* Allow Sales and Procurement to directly SQL `JOIN` to the catalog tables since it's a monolithic database.
*Why Rejected:* This is the classic monolithic anti-pattern. It makes future extraction into microservices impossible without a massive rewrite of all analytical and transactional queries.

---

## Decision Rationale

Implementing the Shared Catalog as an independent Master Data Shared Kernel with Event-Carried State Transfer is the only approach that satisfies all constraints:
- It respects the System Architecture's mandate for "read-optimized local views."
- It aligns with the Event Bus (ADR-003) for reliable asynchronous communication.
- It strictly enforces DDD aggregate boundaries by preventing database-level coupling.
- It provides a single authoritative source for master data, ensuring data integrity for financial operations.

## Consequences

### Positive
- **Strict Decoupling:** Sales and Procurement are entirely decoupled from the Catalog at runtime.
- **Microservices Ready:** The design translates 1:1 to a distributed architecture.
- **Performance:** Local read-models in Sales and Procurement mean fast, local database joins for financial reports and queries.
- **Historical Integrity:** Financial documents (Invoices, POs) copy the catalog data at the time of creation (Value Objects), preventing historical records from changing if a product's name or price changes later.

### Negative
- **Eventual Consistency:** The local views in Sales and Procurement will be eventually consistent with the Master Data. A user updating a Product must wait (typically milliseconds, but non-zero) for the event to process before seeing the update in the Sales Quotation screen.
- **Data Duplication:** Storing read-models in Sales and Procurement duplicates data across database schemas. This is a deliberate trade-off of storage space (cheap) in exchange for architectural decoupling and performance (valuable).

---

## Governing References

| Document | Relevance |
|---|---|
| [System Architecture](../05-architecture/SYSTEM_ARCHITECTURE.md) | Defines Contacts, Products, and Services as Shared Concerns requiring local read views |
| [BUSINESS_MODULE_CATALOG.md](../02-brd/BUSINESS_MODULE_CATALOG.md) | Defines the business purpose of the shared modules |
| [BRD](../02-brd/BRD.md) | FR-012, FR-013 (discontinuation flagging rules) |
| [ADR-003](ADR-003-event-bus-design.md) | Event Bus mechanism for synchronization |
| [ADR-001](ADR-001-technology-stack.md) | Modular monolith deployment target |

---

*This ADR defines the ownership and synchronization strategy for shared master data. Future changes to this strategy must be captured in subsequent ADRs.*
