---

## 4. UDM-Compliant Class / Domain Model

The Domain Model maps out the structural data contracts, data types, and lifecycle boundaries within the NotNaked CDP. It strictly implements the enterprise Universal Data Model (UDM) Party Module standard (NFR-01) by decoupling structural attributes from direct identities through explicit junction and lookup entities.

```mermaid
classDiagram
    class Party {
        +UUID partyId
        +String partyType
        +Timestamp createdAt
        +Timestamp updatedAt
    }

    class Person {
        +String firstName
        +String lastName
        +Date dateOfBirth
        +Map customTraits
    }

    class Organization {
        +String organizationName
        +String taxIdentifier
        +String industryType
    }

    class PartyRole {
        +UUID partyRoleId
        +UUID partyId
        +String roleTypeId
        +Timestamp fromDate
        +Timestamp thruDate
    }

    class RoleType {
        +String roleTypeId
        +String description
    }

    class ContactMechanism {
        +UUID contactMechanismId
        +String contactMechanismTypeId
    }

    class PostalAddress {
        +String street1
        +String street2
        +String city
        +String province
        +String zipCode
        +String country
    }

    class TelecomNumber {
        +String countryCode
        +String areaCode
        +String contactNumber
    }

    class EmailAddress {
        +String emailAddress String
    }

    class PartyContactMech {
        +UUID partyId
        +UUID contactMechanismId
        +Timestamp fromDate
        +Timestamp thruDate
        +Boolean isVerified
    }

    class ContactPreference {
        +UUID contactPreferenceId
        +UUID partyId
        +String contactMechanismTypeId
        +Integer preferenceSequence
        +Boolean marketingOptIn
    }

    class BehavioralEvent {
        +UUID eventId
        +UUID partyId
        +String eventType
        +Timestamp timestamp
        +String rawPayloadJson
    }

    class Segment {
        +UUID segmentId
        +String name
        +String ruleConditionSql
        +Timestamp createdAt
    }

    class SegmentMembership {
        +UUID segmentId
        +UUID partyId
        +Timestamp evaluatedAt
    }

    %% Inheritance Hierarchy (IS-A)
    Party <|-- Person : Specializes as Individual
    Party <|-- Organization : Specializes as Business Entity
    
    ContactMechanism <|-- PostalAddress : Specializes Type
    ContactMechanism <|-- TelecomNumber : Specializes Type
    ContactMechanism <|-- EmailAddress : Specializes Type

    %% Role and Lookup Mappings
    Party "1" *-- "0..*" PartyRole : Manifests Into
    RoleType "1" -- "0..*" PartyRole : Defines

    %% UDM Junction Mapping for Addresses and Channels
    Party "1" *-- "0..*" PartyContactMech : Structural Link
    ContactMechanism "1" -- "0..*" PartyContactMech : Resolves Destination

    %% Preference and Activity Logs
    Party "1" *-- "0..*" ContactPreference : Dictates
    Party "1" o-- "0..*" BehavioralEvent : Records Stream

    %% Segmentation Junction Entity
    Segment "1" *-- "0..*" SegmentMembership : Roster Group
    Party "1" -- "0..*" SegmentMembership : Captures State
