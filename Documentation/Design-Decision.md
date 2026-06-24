## 🏗️ Domain 1: Database Architecture & Structural Modeling

### 1. Unified Data Model (UDM) Party Inheritance Pattern

* **The Reality:** Shopify maps customer data as a flat object (First Name, Last Name, and metadata are columns in a single row).
* **The Trade-off:** * *Option A (Flat Model):* Keep a single flat table for customers. This makes simple inserts incredibly fast and reduces relational joins. However, it breaks down if a single corporate identity contains multiple sub-identities, or if an anonymous entity transitions into a known lead.
* *Option B (Relational Separation):* Split records into an abstract structural layer (`CDP_PARTY`), a biographical sub-type (`CDP_PERSON`), and a distinct metadata context (`CDP_PARTY_ATTRIBUTE`).


* **Our Strategic Choice:** **Option B.** By using a strict table-inheritance design linked via standard structural foreign keys (`PARTY_ID`), our data core remains infinitely extensible. We can support B2B profiles, employees, or retail shoppers without ever changing our primary table structures.

### 2. High-Performance Metric Caching Layer

* **The Reality:** Marketers segment customers based on behavioral financial summaries (e.g., `"total_spent > $500"` or `"orders_count = 0"`).
* **The Trade-off:**
* *Option A (On-the-Fly Aggregation):* Calculate these numbers in real time by executing heavy SQL `SUM()` and `COUNT()` operations across massive transaction ledger tables whenever a query runs. This guarantees 100% real-time accuracy but completely bottlenecks database CPU performance at scale.
* *Option B (Materialized Cache State):* Design a dedicated, separate table (`CDP_PARTY_MARKETING_METRIC`) to maintain a computed state (`ORDERS_COUNT`, `TOTAL_SPENT`).


* **Our Strategic Choice:** **Option B.** We chose to use fixed numeric data types (`DECIMAL(24,4)` and `INT`) inside a dedicated metric caching entity. This scales up read performance for marketing campaigns significantly, accepting the trade-off that our ingestion engine must handle incrementing these numbers during updates.

---

## 🔄 Domain 2: Multi-Valued Contact Mechanics

### 3. Deep Normalization of Identity Communication Channels

* **The Reality:** A customer might have three shipping addresses, two emails, and multiple mobile numbers.
* **The Trade-off:**
* *Option A (Clustered Columns):* Add fixed columns directly to the profile row like `email_1`, `email_2`, `shipping_address_1`, `shipping_address_2`. This avoids relational joins but creates rigid architectural limitations and empty database cells.
* *Option B (Vector Extraction):* Completely isolate contact types into standardized intersection maps (`CDP_CONTACT_MECHANISM`, `CDP_POSTAL_ADDRESS`, `CDP_EMAIL_ADDRESS`, `CDP_PARTY_CONTACT_MECH`).


* **Our Strategic Choice:** **Option B.** Every communication vector is treated as an independent row linked through an intersection ledger. This allows an unlimited number of contact points per profile and makes tracking cross-channel routing direct.

---

## 🛠️ Domain 3: Data Access Logic & Mutability Strategy

### 4. Immutable History / Append-Only Postal Address Resolution

* **The Reality:** Handling address updates at runtime introduces string parsing complications. Minor differences like `"76 B, Park Ave"` vs `"76b Park Ave"` are hard for code to accurately match.
* **The Trade-off:**
* *Option A (String Comparison Engine):* Write a complex text-matching validation block to parse, sanitize, and compare incoming elements with existing records to determine if a real change occurred. This is computationally expensive and prone to false matches.
* *Option B (Immutable History Reset):* Avoid text matching entirely. When an address update payload comes in, mark the old relationship as inactive (`THRU_DATE = NOW`) and drop a brand-new row into the tables.


* **Our Strategic Choice:** **Option B.** This senior-architect shortcut eliminates validation complexity. It ensures 100% clean operations, eliminates calculation bottlenecks, and preserves an exact historical audit trail of historical delivery destinations for old invoices.

---

## 🌐 Domain 4: Shopify Integration Boundaries

### 5. Root Node De-duplication & Array-Only Flattening Invariant

* **The Reality:** The raw Shopify Customer JSON payload duplicates address data. It exposes an `addresses[]` collection while repeating a copy of the default location data under a separate `default_address` root object.
* **The Trade-off:**
* *Option A (Multi-Pass Ingestion):* Process the `default_address` node independently, and then parse the `addresses[]` list while checking for duplicates. This requires heavy synchronization filters to avoid creating redundant entries in your tables.
* *Option B (Array Isolation):* Completely ignore and drop the root `default_address` node. Parse *only* the unified `addresses[]` array, dynamically determining the location purpose by inspecting the boolean `"default": true/false` flag inside each item.


* **Our Strategic Choice:** **Option B.** This simplifies our ingestion code. By evaluating the nested boolean flag within a single looping path, we achieve cleaner execution logic and enforce data integrity out of the box.

--- 

### 6. Role-to-Role Contextual Boundaries (`CDP_PARTY_ROLE`)

* **The Reality:** In a basic database, you might directly link an address or preference to a raw profile ID.
* **The Trade-off:** Linking data to a raw identity works fine if a customer is *only* ever a customer. But if that same person later becomes an employee, a vendor, or an affiliate manager, their data gets messy and hard to secure.
* **Our Strategic Choice:** **Explicit Role Decoupling.** We chose to enforce that a profile *must* explicitly register an active entry in `CDP_PARTY_ROLE` with `ROLE_TYPE_ID = 'CUSTOMER'`. This acts as a security barrier, ensuring that operational marketing systems only touch records within a verified business context.

### 7. Explicit Tracking-Type Resolution over Universal Keys

* **The Reality:** We need to track multi-system integration IDs, like Shopify IDs, Google tracking tokens, and ERP keys.
* **The Trade-off:** Making separate columns in your primary customer table for every single external system ID creates a rigid schema that requires a migration script every time a new marketing tool is added.
* **Our Strategic Choice:** **Cross-Reference Registry (`CDP_PARTY_IDENTIFICATION`).** We chose an open, extensible catalog design. Every incoming third-party key is treated as an independent row categorized by an identification type, such as `PARTY_IDENT_TYPE_ID = 'SHOPIFY_CUST_ID'`. This allows us to connect an unlimited number of downstream systems without changing a single column structure.

---
