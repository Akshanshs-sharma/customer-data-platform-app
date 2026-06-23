---

## 3. Sequence Diagrams

### 3.1 Real-Time Ingestion & Identity Resolution Pipeline (UC-04 & UC-06)
This sequence diagram illustrates the step-by-step temporal interaction between Shopify, the API Ingestion Controller, the Identity Resolution Service, and the persistent datastore when an incoming webhook is received.

```mermaid
sequenceDiagram
    autonumber
    actor Shopify as Shopify Platform
    box Internal CDP System Boundary
        participant Ctrl as IngestionController
        participant Audit as EventAuditLog
        participant Identity as IdentityResolutionEngine
        participant DB as ProfileDatabase
    end

    %% UC-04: Webhook Ingestion Flow
    Shopify ->> Ctrl: HTTP POST /api/v1/ingest/webhook (Payload + Signature)
    activate Ctrl
    Ctrl ->> Ctrl: Verify Webhook Checksum (HMAC Secret)

    alt Invalid Signature
        Ctrl -->> Shopify: HTTP 401 Unauthorized
    else Valid Signature
        Ctrl ->> Audit: Write Raw Event Payload (Staging)
        activate Audit
        Audit -->> Ctrl: Persisted Status Confirmation
        deactivate Audit
        
        Ctrl -->> Shopify: HTTP 202 Accepted (Async Processing Enqueued)
        deactivate Ctrl
    end

    %% UC-06: Asynchronous Identity Resolution Flow
    activate Identity
    Identity ->> Audit: Fetch Unprocessed Staging Records
    Audit -->> Identity: Return Raw Payloads

    Identity ->> DB: Scan for Existing Identifiers (Email, Phone, Shopify ID)
    activate DB
    DB -->> Identity: Return Matching Rows (Found Duplicates)

    alt Matching Identifiers Found
        Identity ->> DB: Merge Source Traits under Master Unified Record ID
        Identity ->> DB: Update Verification & Parent Profile References
    else No Matching Records Found
        Identity ->> DB: Instantiate Brand New Unified Profile Record
    end

    Identity ->> DB: Append Operational Traceability Records to Audit Table
    DB -->> Identity: Commit Transaction Success
    deactivate DB
    deactivate Identity

```
### 3.2 Segment Evaluation Workflow (UC-07)
This sequence diagram illustrates how saved rules are loaded, compiled into analytical database queries, and executed to refresh segment membership mapping records.

```mermaid
sequenceDiagram
    autonumber
    actor Caller as Data Analyst / Automation Cron
    box Internal CDP System Boundary
        participant Ctrl as SegmentController
        participant Engine as SegmentationEngine
        participant DB as ProfileDatabase
    end

    Caller ->> Ctrl: POST /api/v1/segments/{segmentId}/refresh
    activate Ctrl
    
    Ctrl ->> Engine: Trigger Evaluation Request (SegmentID)
    activate Engine
    
    Engine ->> DB: Fetch Saved Segment Rules & Metadata
    activate DB
    DB -->> Engine: Return Rule AST/Conditions (e.g., total_spent > 500)
    
    Engine ->> Engine: Compile Rules into Optimized Database Query String
    
    Engine ->> DB: Execute Dynamic Segment Query Against Active Profiles
    DB -->> Engine: Return List of Matching Unified Customer IDs
    
    Engine ->> DB: Flush Old Roster & Batch Insert New Customer ID Mappings
    DB -->> Engine: Confirm Transaction Commit (Success)
    deactivate DB
    
    Engine -->> Ctrl: Return Total Membership Count & Execution Metrics
    deactivate Engine
    
    Ctrl -->> Caller: Return HTTP 200 OK (JSON Summary Map)
    deactivate Ctrl
