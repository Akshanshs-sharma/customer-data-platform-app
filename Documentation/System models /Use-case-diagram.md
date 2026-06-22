## 2. Use Case Diagram

The Use Case Diagram establishes the relationship between external/internal actors and the core behavioral workflows implemented inside the NotNaked CDP boundary.

```mermaid
flowchart LR
    %% Actors
    Analyst[Data Analyst]
    Shopify[Shopify External]
    Automation[CDP Automation]

    %% System Boundary
    subgraph CDP_System [Customer Data Platform]
        UC01((UC-01: Define Customer Segment))
        UC02((UC-02: Query Event History))
        UC03((UC-03: Send Customer Data))
        UC04((UC-04: Ingest Shopify Webhook))
        UC05((UC-05: Run Scheduled Sync))
        UC06((UC-06: Execute Identity Resolution))
        UC07((UC-07: Evaluate Segments))
    end

    %% Analyst Relationships
    Analyst --> UC01
    Analyst --> UC02
    Analyst --> UC07

    %% Shopify Relationships
    Shopify --> UC03
    Shopify --> UC04

    %% Automation/Scheduler Relationships
    Automation --> UC05
    Automation --> UC06
    Automation --> UC07
