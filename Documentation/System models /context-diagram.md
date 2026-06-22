# System Modeling Specification (Phase 2)
*Project: NotNaked Customer Data Platform (CDP)*
*Reference Standard: Ian Sommerville, Software Engineering (9th Ed.), Chapter 5*

---

## 1. System Context Diagram

The context diagram defines the operational boundary of the NotNaked CDP, highlighting the external data producers, consumer actors, and the automated scheduling engines interacting with the platform.

```mermaid
graph TD
    %% System Boundary
    subgraph CDP_Boundary [NotNaked CDP System Boundary]
        CDP_Core[CDP Ingestion Engine & Profile Datastore]
    end

    %% External System Actors
    Shopify[Shopify Core Platform<br/>External Actor]
    
    %% Human Actors
    Analyst[Data Analyst / CDP Manager<br/>Human Actor]
    
    %% Automated/Internal System Actors
    Scheduler[CDP Engine Scheduler<br/>Automated Cron Actor]

    %% Data Flows
    Shopify -->|1. Real-time Webhooks & Customer Payloads| CDP_Core
    CDP_Core -.->|2. Batch Request Pulls| Shopify
    
    Analyst -->|3. Segment Definitions & Query Parameters| CDP_Core
    CDP_Core -->|4. Profile Views & Historical Logs| Analyst

    Scheduler -->|5. Scheduled Cron Trigger Evaluation| CDP_Core
