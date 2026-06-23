# System Modeling Specification (Phase 2)
*Project: NotNaked Customer Data Platform (CDP)*
*Reference Standard: Ian Sommerville, Software Engineering (9th Ed.), Chapter 5*

---

## 1. System Context Diagram

The context diagram defines the operational boundary of the NotNaked CDP, highlighting the external data producers, human administrators, and internal automated processes interacting with the core engine.

```mermaid
graph TD
    %% System Boundary
    subgraph CDP_Boundary [NotNaked CDP System Boundary]
        CDP_Core[CDP Ingestion Engine & Profile Datastore]
        Scheduler[CDP Engine Scheduler<br/>Internal Automated Actor]
    end

    %% External System Actors
    Shopify[Shopify Core Platform<br/>External System Actor]
    Analyst[Data Analyst / CDP Manager<br/>Human Actor]

    %% Data Flows
    Shopify -->|1. Transmits Real-time Webhook Payloads| CDP_Core
    CDP_Core -.->|2. Returns Synchronous HTTP Status Acknowledgement| Shopify
    CDP_Core -->|3. Initiates Scheduled Batch API Data Pulls| Shopify
    
    Analyst -->|4. Configures Segment Rules & Query Constraints| CDP_Core
    CDP_Core -->|5. Serves Profile Views & Chronological Event History| Analyst

    Scheduler -->|6. Triggers Asynchronous Processing, Identity Resolution & Segment Evaluation| CDP_Core
