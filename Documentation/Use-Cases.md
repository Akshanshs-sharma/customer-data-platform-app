### Use Case Specifications

#### Use Case ID: UC-01

* **Actor:** Data Analyst
* **Goal:** Establish behavioral target rules and definitions for customer segmentation (FR-08).
* **Precondition:** * The Data Analyst is authenticated and logged into the CDP management interface.
* The system metadata contains valid searchable attributes, behavioral event types, and trait references.


* **Steps:**
1. The Data Analyst provides a unique segment name and defines specific rule conditions (e.g., total spent greater than 3 times).
2. The system validates the rule query syntax to ensure logical and operational consistency.
3. The system saves the finalized segment rule definition with a unique Segment ID into the repository.
4. The system confirms successful segment creation to the Analyst.


* **Postcondition:** A new segment definition is successfully registered and made available for downstream membership evaluation without modifying any profile records.

---

#### Use Case ID: UC-04

* **Actor:** Shopify (External System)
* **Description:** Stream transactional storefront modifications into the platform in real time (FR-05, FR-11).
* **Precondition:** * Active webhook topics are registered in Shopify pointing to the CDP's ingestion URL.
* The CDP ingestion service is online and accessible over the network.


* **Steps:**
1. An event occurs on the Shopify storefront (e.g., order creation or customer profile modification).
2. Shopify dispatches an HTTP POST payload containing the operational customer data to the ingestion endpoint.
3. The system captures the request and verifies the webhook checksum metadata header using the shared secret signature.
4. The system parses structural attributes to verify payload formatting against basic ingestion schemas.
5. The system writes the raw event entry safely to the Event History transaction logs.


* **Postcondition:** The incoming payload is safely logged and staged inside the ingestion pool, maintaining absolute data integrity (NFR-03).

---

#### Use Case ID: UC-06

* **Actor:** CDP Automation / Scheduler
* **Description:** Reconcile independent data streams into unified customer master records (FR-06).
* **Precondition:** * Raw customer profiles or event payloads have been ingested and logged into the staging tables.
* Identity resolution heuristics (matching strings for email, phone, or anonymous ID) are configured.


* **Steps:**
1. The background automation processor scans un-reconciled rows within the staging tables.
2. The system identifies matching explicit identification string markers across separate profile database rows.
3. The system merges matching records under a single, unified Master Record ID, preserving distinct entity children like multiple addresses.
4. The system updates the operational audit trail logs to record the merge operation details.


* **Postcondition:** Isolated tracking fragments are consolidated into single, unified profiles with clear traceability (NFR-04).

---

#### Use Case ID: UC-07


* **Actor:** CDP Automation / Data Analyst
* **Goal:** Run the specific logical rules of a saved segment against the active customer database to calculate and update current membership rosters (FR-09).
* **Precondition:** * The segment has been formally created and its logical syntax is saved in the repository (FR-08).
* The system contains processed, up-to-date customer profiles, traits, and associated event tables to query against.


* **Steps:**
1. The actor triggers the evaluation (either an automated clock schedule kicks off, or the Analyst manually hits "Refresh Segment" in the interface).
2. The system retrieves the saved rules and conditions (e.g., `total_spent > 500` AND `country == 'IN'`) for that specific segment.
3. The system translates these criteria into a optimized database query and executes it against the customer profile datastore.
4. The system compiles the matching list of explicit Customer IDs and writes/flushes the results to the segment membership mapping table.
5. The system saves execution performance logs and returns the final total membership count to the caller.


* **Postcondition:** The segment roster is explicitly updated, ensuring downstream tools or activation workflows access the true, active list of matching profiles.

---

#### Use Case ID: UC-05

* **Actor:** CDP Automation / Scheduler
* **Goal:** Automatically execute a scheduled batch pull from Shopify's Customer API to sync updates and maintain data consistency (FR-11).
* **Precondition:** * The CDP's internal scheduling service (e.g., cron engine) is operational.
* API connection credentials and cursor tokens (tracking the `last_sync_timestamp`) for NotNaked’s Shopify store are securely stored and accessible.


* **Steps:**
1. The internal CDP Scheduler triggers the sync job at a pre-configured time interval (e.g., nightly at 00:00).
2. The system retrieves the timestamp of the last successful synchronization.
3. The system executes a paginated HTTP GET request to the Shopify Customer API, filtering for records modified after that specific timestamp.
4. The system streams the batch payload back into the CDP boundary, writing the raw ingestion entries directly to the audit tables (NFR-03, NFR-04).
5. The system updates the `last_sync_timestamp` configuration marker upon receiving a successful HTTP `200 OK` network response from Shopify.


* **Postcondition:** The system safely enqueues the newly fetched batch records for identity processing, updating the scheduler tracking state to guarantee no data overlaps or omissions occur on the next run.

---

#### Use Case ID: UC-03

* **Actor:** Shopify (External System)
* **Goal:** Send real-time customer mutations (creation, updates) from the ecommerce storefront to the CDP to keep customer records unified and current.
* **Precondition:** * The CDP's REST API endpoint for customer ingestion is live and listening (FR-11).
* Valid authentication credentials or webhook secret signatures are established between Shopify and the CDP.


* **Steps:**
1. A customer changes state in Shopify (e.g., account created, address edited, marketing opt-in modified).
2. Shopify constructs a standardized JSON payload containing the customer’s identity attributes, addresses, and communication preferences.
3. Shopify transmits an HTTP POST request containing the payload and verification headers to the CDP's ingestion API.
4. The system validates the request headers to ensure authenticity and checks the structural integrity of the payload.
5. The system saves the raw payload into the event repository queue for downstream async processing and returns an HTTP `202 Accepted` or `200 OK` status to Shopify.


* **Postcondition:** The incoming data is safely persisted in the system's tracking logs without blocking Shopify's queue, waiting to be picked up by the identity resolution and profile management layers.

---

#### Use Case ID: UC-02

* **Actor:** Data Analyst
* **Goal:** Query and review a comprehensive, chronological timeline of behavioral events associated with a specific customer profile.
* **Precondition:** * The Data Analyst is authenticated and logged into the CDP management interface.
* The system contains existing records of unified customer profiles (FR-01) and ingested behavioral events (FR-05) mapped to those profiles.


* **Steps:**
1. The Data Analyst requests to view a customer profile by providing a unique identifier (e.g., Customer ID or primary email).
2. The system retrieves the customer profile and queries the event log repository for all historical entries matching that profile's identifiers.
3. The system sorts the retrieved events chronologically (newest to oldest).
4. The system displays the customer profile metadata along with the filterable, step-by-step event timeline to the Analyst.


* **Postcondition:** The system successfully displays the event log timeline without altering the state of the customer profile or historical log data (read-only action).

