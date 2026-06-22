## Requirements Reconciliation

### What the Assignment Adds That We're Missing

**1. Address Management (multi-address per customer)**
The assignment explicitly requires billing + shipping addresses, multiple addresses per customer. Our FR-01 only says "name, email, phone" — too narrow.

**2. Contact Mechanisms (multiple per customer)**
Email, phone numbers, social media handles — all as separate, multiple entries per customer. We lumped this into FR-01. It needs its own requirement.

**3. Customer Preferences**
Marketing opt-ins, preferred communication channels. We never captured this at all.

**4. Shopify Integration**
Ingesting customer data specifically from Shopify's Customer API — mapping, transformation, upsert logic. This is an entirely new requirement we didn't have.

**5. UDM Compliance**
The schema must follow the **Universal Data Model Party module** structure. This is a design constraint that affects everything — it's not a feature but it needs to be stated as a requirement.

---

## Revised & Reconciled Requirements

### Functional Requirements

**FR-01 — Customer Profile Management**
The system shall allow creation and storage of a unified customer profile containing core identity attributes (first name, last name, date of birth).

**FR-02 — Multi-Address Management**
The system shall support multiple addresses per customer profile, including billing and shipping address types, storing address components (street, city, province, zip, country, phone).

**FR-03 — Contact Mechanism Management**
The system shall support multiple contact mechanisms per customer (email, phone, social media handles), each typed and independently manageable.

**FR-04 — Customer Preferences**
The system shall store customer preference data including marketing opt-ins and preferred communication channels.

**FR-05 — Event Ingestion**
The system shall accept and store behavioral events from external sources (e.g. page_viewed, product_purchased, form_submitted) tied to a customer profile.

**FR-06 — Identity Resolution**
The system shall merge profiles when multiple identifiers (email, phone, anonymous ID) are found to belong to the same customer.

**FR-07 — Trait Management**
The system shall allow custom traits/attributes to be attached to a customer profile (e.g. plan_type, signup_source).

**FR-08 — Segment Definition**
The system shall allow creation of customer segments based on rule conditions (e.g. customers who purchased more than 3 times).

**FR-09 — Segment Evaluation**
The system shall evaluate which customers match a given segment's rules and return that list.

**FR-10 — Event History**
The system shall maintain a queryable history of all events for a customer profile.

**FR-11 — Shopify Customer Ingestion**
The system shall retrieve customer data from the Shopify Customer API, transform it to the internal data model, and store it — handling duplicates via upsert logic.

---

### Non-Functional Requirements

**NFR-01 — UDM Compliance**
The database schema shall follow the Party module of the Universal Data Model to ensure structural consistency and extensibility.

**NFR-02 — API-first**
All system capabilities shall be exposed via REST APIs.

**NFR-03 — Data Integrity**
No event or profile update shall be silently lost — failures must be logged.

**NFR-04 — Auditability**
Profile changes shall be traceable (who changed what, when).
