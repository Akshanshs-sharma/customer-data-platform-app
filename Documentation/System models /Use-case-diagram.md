---

## 2. Use Case Diagram

The Use Case Diagram establishes the dynamic boundaries between the platform's actors and the 6 core behavioral workflows of the NotNaked CDP using standard UML notation.

```mermaid
flowchart LR
    %% Actors
    Analyst(((Data Analyst)))
    Shopify(((Shopify External)))
    Automation(((CDP Automation)))

    %% System Boundary
    subgraph CDP_System [Customer Data Platform Boundary]
        UC01((UC-01: Define Customer Segment))
        UC02((UC-02: Query Event History))
        UC03((UC-03: Shopify Webhook Ingestion))
        UC04((UC-04: Run Scheduled Shopify Sync))
        UC05((UC-05: Execute Identity Resolution Rules))
        UC06((UC-06: Evaluate Segments))
    end

    %% Data Analyst Interactions
    Analyst --> UC01
    Analyst --> UC02
    Analyst --> UC06

    %% Shopify Interactions
    Shopify --> UC03

    %% Automation Service Interactions
    Automation --> UC04
    Automation --> UC05
    Automation --> UC06
