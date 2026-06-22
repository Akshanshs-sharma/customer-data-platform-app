### Use Case Specifications

#### Use Case ID: UC-01 — Define Customer Segment
* **Actor:** Data Analyst
* **Goal:** Establish behavioral target rules and definitions for customer segmentation (FR-08).
* **Precondition:** * The Data Analyst is authenticated and logged into the CDP management interface.
* **Steps:**
  1. The Data Analyst provides a unique segment name and defines specific rule conditions (e.g., purchased > 3 times).
  2. The system validates the rule query syntax to ensure logical and operational consistency.
  3. The system saves the finalized segment rule definition with a unique Segment ID into the segment repository.
  4. The system confirms successful segment creation to the Analyst.
* **Postcondition:** A new segment definition is successfully registered and made available for downstream membership evaluation without modifying any profile records.

---

#### Use Case ID: UC-02 — Query Event History
* **Actor:** Data Analyst
* **Goal:** Query and review a comprehensive, chronological timeline of behavioral events associated with a specific customer profile (FR-10).
* **Precondition:** * The Data Analyst is authenticated and logged into the CDP management interface.
  * The system contains existing records of unified customer profiles and ingested behavioral events mapped to those profiles.
* **Steps:**
  1. The Data Analyst requests to view a customer profile by providing a unique identifier (e.g., Customer ID or primary email).
  2. The system retrieves the customer profile and queries the event log repository for all historical entries matching that profile's identifiers.
  3. The system sorts the retrieved events chronologically (newest to oldest).
  4. The system displays the customer profile metadata along with the filterable, step-by-step event timeline to the Analyst.
* **Postcondition:** The system successfully displays the event log timeline without altering the state of the customer profile or historical log data (read-only action).

---

#### Use Case ID: UC-03 — Shopify Webhook Ingestion
* **Actor:** Shopify (External System)
* **Goal:** Stream real-time data payloads (including profile modifications and behavioral events) into the platform over HTTP (FR-05, FR-11).
* **Precondition:** * Active webhook topics are registered in Shopify pointing to the CDP's ingestion URL.
  * The CDP ingestion service is online and an authentic webhook secret signature configuration is shared.
* **Steps:**
  1. An event occurs on the Shopify storefront (e.g., customer account created, address modified, or a transactional event like `product_purchased`).
  2. Shopify dispatches an HTTP POST payload containing the raw JSON dataset and verification checksum header to the ingestion endpoint.
  3. The system captures the request and verifies the webhook checksum metadata header using the shared secret signature.
  4. The system parses structural attributes to verify payload formatting against basic ingestion schemas.
  5. The system writes the raw payload entry safely into the persistent Event History transaction logs.
  6. The system returns a synchronous HTTP `202 Accepted` status code to Shopify to unblock its external event queue.
* **Postcondition:** The incoming payload is safely logged and staged inside the ingestion pool, maintaining absolute data integrity (NFR-03).

---

#### Use Case ID: UC-04 — Run Scheduled Shopify Sync
* **Actor:** CDP Automation / Scheduler
* **Goal:** Automatically execute a scheduled batch pull from Shopify's Customer API to sync updates and maintain data consistency (FR-11).
* **Precondition:** * The CDP's internal scheduling service (the internal Scheduler engine) is operational.
  * API connection credentials and cursor tokens (tracking the `last_sync_timestamp`) for NotNaked’s Shopify store are securely stored and accessible.
* **Steps:**
  1. The internal CDP Scheduler triggers the sync job at a pre-configured time interval (e.g., nightly at 00:00).
  2. The system retrieves the timestamp of the last successful synchronization.
  3. The system executes a paginated HTTP GET request to the Shopify Customer API, filtering for records modified after that specific timestamp.
  4. The system streams the batch payload back into the CDP boundary, writing the raw ingestion entries directly to the staging logs (NFR-03, NFR-04).
  5. The system updates the `last_sync_timestamp` configuration marker upon receiving a successful HTTP `200 OK` network response from Shopify.
* **Postcondition:** The system safely enqueues the newly fetched batch records for identity processing, updating the scheduler tracking state to guarantee no data overlaps or omissions occur on the next run.

---

#### Use Case ID: UC-05 — Execute Identity Resolution Rules
* **Actor:** CDP Automation / Scheduler
* **Goal:** Reconcile independent data streams and transient logs into single customer master units (FR-06).
* **Precondition:** * Raw customer profiles or event payloads have been ingested and logged into the staging tables.
  * Identity resolution heuristics (matching strings for email, phone, or anonymous ID) are configured.
* **Steps:**
  1. The background automation processor scans un-reconciled rows within the staging tables.
  2. The system identifies matching explicit identification string markers across separate profile database rows.
  3. The system merges matching records under a single, unified Master Record ID, preserving distinct entity children like multiple addresses via UDM links.
  4. The system updates the operational audit trail logs to record the merge operation details.
* **Postcondition:** Isolated tracking fragments are consolidated into single, unified profiles with clear traceability (NFR-04).

---

#### Use Case ID: UC-06 — Evaluate Segments
* **Actor:** CDP Automation / Data Analyst
* **Goal:** Run the specific logical rules of a saved segment against the active customer database to calculate and update current membership rosters (FR-09).
* **Precondition:** * The segment has been formally created and its logical syntax is saved in the repository (FR-08).
  * The system contains processed, up-to-date customer profiles, traits, and associated event tables to query against.
* **Steps:**
  1. The actor triggers the evaluation (either an automated clock schedule kicks off, or the Analyst manually hits "Refresh Segment" in the interface).
  2. The system retrieves the saved rules and conditions (e.g., `total_spent > 500` AND `country == 'IN'`) for that specific segment.
  3. The system translates these criteria into an optimized database query and executes it against the customer profile datastore.
  4. The system compiles the matching list of explicit Customer IDs and writes/flushes the results to the segment membership mapping table.
  5. The system saves execution performance logs and returns the final total membership count to the caller.
* **Postcondition:** The segment roster is explicitly updated, ensuring downstream tools or activation workflows access the true, active list of matching profiles.
