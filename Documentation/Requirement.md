# Requirements Engineering Specification (Phase 1)
*Project: NotNaked Customer Data Platform (CDP)* *Reference Standard: Ian Sommerville, Software Engineering (9th Ed.)*

---

## 1. System Actors
An actor represents any entity outside the system boundary that interacts with or triggers the CDP.

1. **Data Analyst / CDP Manager (Human Actor)**: The platform administrator who configures rules, builds segments, and reviews analytical customer timelines.
2. **Shopify (External System Actor)**: The primary transactional ecommerce source pushing active customer data, contact mutations, and storefront events via APIs/Webhooks.
3. **CDP Automation / Scheduler (Internal System Actor)**: Automated background cron processors and automation workflows executing deterministic rules (e.g., matching identities, cron evaluations).

---

## 2. Functional Requirements (FR)

| ID | Requirement Specification | Primary Initiator |
| :--- | :--- | :--- |
| **FR-01** | **Customer Profile Management**: The system shall allow creation and storage of a unified customer profile containing core identity attributes (first name, last name, date of birth). | Shopify / Data Analyst |
| **FR-02** | **Multi-Address Management**: The system shall support multiple addresses per customer profile, including billing and shipping address types, storing address components (street, city, province, zip, country, phone). | Shopify |
| **FR-03** | **Contact Mechanism Management**: The system shall support multiple contact mechanisms per customer (email, phone, social media handles), each typed and independently manageable. | Shopify |
| **FR-04** | **Customer Preferences**: The system shall store customer preference data including marketing opt-ins and preferred communication channels. | Shopify |
| **FR-05** | **Event Ingestion**: The system shall accept and store behavioral events from external sources (e.g., `page_viewed`, `product_purchased`, `form_submitted`) tied to a customer profile. | Shopify |
| **FR-06** | **Identity Resolution**: The system shall merge profiles when multiple identifiers (email, phone, anonymous ID) are found to belong to the same customer. | CDP Automation |
| **FR-07** | **Trait Management**: The system shall allow custom traits/attributes to be attached to a customer profile (e.g., `plan_type`, `signup_source`). | CDP Automation / Data Analyst |
| **FR-08** | **Segment Definition**: The system shall allow creation of customer segments based on rule conditions (e.g., customers who purchased more than 3 times). | Data Analyst |
| **FR-09** | **Segment Evaluation**: The system shall evaluate which customers match a given segment's rules and return that list. | Data Analyst / CDP Automation |
| **FR-10** | **Event History**: The system shall maintain a queryable history of all events for a customer profile. | Data Analyst |
| **FR-11** | **Shopify Customer Ingestion**: The system shall retrieve customer data from the Shopify Customer API, transform it to the internal data model, and store it — handling duplicates via upsert logic. | CDP Automation / Shopify |

---

## 3. Non-Functional Requirements (NFR)

* **NFR-01: UDM Compliance** — The database schema shall follow the Party module of the Universal Data Model (UDM) to ensure structural consistency and extensibility.
* **NFR-02: API-first** — All system capabilities shall be exposed via programmatic REST APIs.
* **NFR-03: Data Integrity** — No event or profile update shall be silently dropped or lost; processing failures must be captured and logged.
* **NFR-04: Auditability** — Core customer profile mutations shall be completely traceable (who changed what, and when).
