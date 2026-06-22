```mermaid
erDiagram
    %% Infrastructure Domain
    SYSTEM_MESSAGE_TYPE ||--o{ SYSTEM_MESSAGE : "classifies"
    SYSTEM_MESSAGE_REMOTE ||--o{ SYSTEM_MESSAGE : "originates"
    SYSTEM_MESSAGE ||--o{ SYSTEM_MESSAGE_ERROR : "logs"
    DATA_MANAGER_CONFIG ||--o{ DATA_MANAGER_LOG : "drives"
    DATA_MANAGER_LOG ||--o{ DATA_MANAGER_CONTENT : "references"

    %% Party Domain Core
    PARTY_TYPE ||--o{ PARTY : "defines structural class"
    PARTY ||--o| PERSON : "shares identity (IS-A)"
    PARTY ||--o| ORGANIZATION : "shares identity (IS-A)"
    PARTY ||--o{ PARTY_ROLE : "acts in"
    ROLE_TYPE ||--o{ PARTY_ROLE : "defines capacity"

    %% Identity Cross-Referencing
    PARTY_IDENTIFICATION_TYPE ||--o{ PARTY_IDENTIFICATION : "categorizes lookup identifier"
    PARTY ||--o{ PARTY_IDENTIFICATION : "cross-references"

    %% Contact Mechanism Domain
    CONTACT_MECH_TYPE ||--o{ CONTACT_MECHANISM : "classifies method type"
    CONTACT_MECHANISM ||--o| EMAIL_ADDRESS : "specializes as"
    CONTACT_MECHANISM ||--o| TELECOM_NUMBER : "specializes as"
    CONTACT_MECHANISM ||--o| POSTAL_ADDRESS : "specializes as"
    PARTY ||--o{ PARTY_CONTACT_MECH : "reachable through"
    CONTACT_MECHANISM ||--o{ PARTY_CONTACT_MECH : "assigned to"

    %% Analytics & Segmentation Domain
    BEHAVIORAL_EVENT_TYPE ||--o{ BEHAVIORAL_EVENT : "classifies action context"
    PARTY ||--o{ BEHAVIORAL_EVENT : "triggers timeline event"
    SEGMENT ||--o{ SEGMENT_MEMBERSHIP : "tracks cohort metrics"
    PARTY ||--o{ SEGMENT_MEMBERSHIP : "belongs to"

    %% Primary Entity Attributes Shape Mapping
    PARTY_TYPE {
        VARCHAR partyTypeId PK
        VARCHAR description
    }
    PARTY {
        VARCHAR partyId PK
        VARCHAR partyTypeId FK
        DATETIME lastUpdatedStamp
    }
    PERSON {
        VARCHAR partyId PK, FK
        VARCHAR firstName
        VARCHAR lastName
        DATE birthDate
        VARCHAR gender
    }
    ORGANIZATION {
        VARCHAR partyId PK, FK
        VARCHAR organizationName
        VARCHAR registrationNumber
    }
    PARTY_IDENTIFICATION_TYPE {
        VARCHAR partyIdentificationTypeId PK
        VARCHAR description
    }
    PARTY_IDENTIFICATION {
        VARCHAR partyId PK, FK
        VARCHAR partyIdentificationTypeId PK, FK
        VARCHAR idValue
    }
    CONTACT_MECH_TYPE {
        VARCHAR contactMechTypeId PK
        VARCHAR description
    }
    CONTACT_MECHANISM {
        VARCHAR contactMechId PK
        VARCHAR contactMechTypeId FK
    }
    BEHAVIORAL_EVENT_TYPE {
        VARCHAR behavioralEventTypeId PK
        VARCHAR description
    }
    BEHAVIORAL_EVENT {
        VARCHAR eventId PK
        VARCHAR partyId FK
        VARCHAR behavioralEventTypeId FK
        DATETIME eventDate
        VARCHAR targetProductId
        DECIMAL eventValueNumeric
        LONGTEXT eventPayload
    }
    SEGMENT {
        VARCHAR segmentId PK
        VARCHAR segmentName
        LONGTEXT segmentRuleCriteria
    }
    SEGMENT_MEMBERSHIP {
        VARCHAR segmentId PK, FK
        VARCHAR partyId PK, FK
        DATETIME fromDate PK
        DATETIME thruDate
    }
