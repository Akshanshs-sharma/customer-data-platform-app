## 🛠️ Flow 2: Retrieve Complete Customer Profile

### 🎯 Step A: Textual Flowchart — Profile Retrieval Engine

```text
[ INPUT PATH: Target Unique Database System Key: PartyId ]
                           │
                           ▼
               📝 [ 1. INITIALIZE MEMORY CONTAINER ]
               Instantiate Empty Multi-Domain Map/Dictionary Object: CustomerProfile
                           │
                           ▼
               🔎 [ 2. VERIFY BASE PLATFORM PROFILE RECORD ]
               Query CDP_PARTY WHERE PARTY_ID == Input.PartyId AND PARTY_TYPE_ID == 'PERSON'
                                  / \
                                 /   \
                         NOT    /     \    FOUND
                        FOUND  /       \
                              ▼         ▼
     [ THROW: ProfileNotFoundException ]   ┌────────────────────────────────────────────────┐
     [ Terminate Execution Lifecycle   ]   │ Extract: CREATED_STAMP, LAST_UPDATED_STAMP     │
                                           │ Assign: CustomerProfile.metaAttributes         │
                                           └──────────────────────┬─────────────────────────┘
                                                                  │
                                                                  ▼
                                           🔎 [ 3. EXTRACT PROFILE BIOMETRIC DETAILS ]
                                           Query CDP_PERSON WHERE PARTY_ID == Input.PartyId
                                           Extract: FIRST_NAME, LAST_NAME, BIRTH_DATE, GENDER
                                           Assign: CustomerProfile.personalDetails
                                                                  │
                                                                  ▼
                                           🔎 [ 4. COLLECT PLATFORM CROSS-REFERENCE IDENTIFIERS ]
                                           Query CDP_PARTY_IDENTIFICATION WHERE PARTY_ID == Input.PartyId
                                           Loop Rows -> Map ID_VALUE by PARTY_IDENT_TYPE_ID
                                           Assign: CustomerProfile.externalIdentifiers
                                                                  │
                                                                  ▼
                                           🔎 [ 5. ASSEMBLE ACTIVE COMMUNICATION MECHANISMS ]
                                           Execute Dynamic SQL Join Query:
                                           CDP_PARTY_CONTACT_MECH ──► CDP_CONTACT_MECHANISM 
                                                                  ├──► CDP_EMAIL_ADDRESS
                                                                  ├──► CDP_TELECOM_NUMBER
                                                                  └──► CDP_POSTAL_ADDRESS
                                           WHERE PARTY_ID == Input.PartyId AND THRU_DATE IS NULL
                                                                  │
                                                        🔁 Loop Join Results
                                             Distribute elements based on CONTACT_MECH_TYPE_ID
                                             ├── 'EMAIL_ADDRESS' ──► Append to CustomerProfile.emails[]
                                             ├── 'TELECOM_NUMBER' ─► Append to CustomerProfile.phones[]
                                             └── 'POSTAL_ADDRESS' ─► Append to CustomerProfile.addresses[]
                                                                  │
                                                                  ▼
                                           🔎 [ 6. HARVEST TRAITS, PREFERENCES & METRICS ]
                                           Query CDP_PARTY_TAG WHERE PARTY_ID == Input.PartyId ──► Array -> .tags[]
                                           Query CDP_PARTY_MARKETING_METRIC WHERE PARTY_ID == Input.PartyId ──► .metrics
                                           Query CDP_PARTY_PREFERENCE WHERE PARTY_ID == Input.PartyId ──► .preferences
                                           Query CDP_PARTY_ATTRIBUTE WHERE PARTY_ID == Input.PartyId ──► .customAttributes
                                                                  │
                                                                  ▼
                                           📜 [ 7. VALIDATE PROFILE DATA COMPLETENESS ]
                                           Does profile contain at least one primary email or tracking identity?
                                                      ├── NO  ──► [ THROW: CorruptedDataException ]
                                                      └── YES ──► RETURN CustomerProfile Map Structure
                                                                  [ SUCCESS STOP ]

```

---

### 📝 Step B: Language-Agnostic Pseudo-code — Profile Retrieval

```text
SERVICE METHOD: party.retrieveCustomerProfile
    INPUT PARAMETERS:
        string targetPartyId
    
    OUTPUT PARAMETERS:
        map customerProfileStructure

    // Step 1: Initialize Data Aggregation Structure
    map profileMap = new map()
    
    // Step 2: Query Core Party Record
    RECORD partyRecord = SELECT FROM CDP_PARTY
                         WHERE PARTY_ID == targetPartyId AND PARTY_TYPE_ID == "PERSON"
                         
    IF partyRecord is null THEN
        RAISE EXCEPTION "Lookup Failure: No person-type profile matching identifier " + targetPartyId + " exists."
    END IF
    
    profileMap.put("partyId", partyRecord.PARTY_ID)
    profileMap.put("createdAt", partyRecord.CREATED_STAMP)
    profileMap.put("lastUpdatedAt", partyRecord.LAST_UPDATED_STAMP)

    // Step 3: Populate Biographical Data
    RECORD personRecord = SELECT FROM CDP_PERSON WHERE PARTY_ID == targetPartyId
    IF personRecord is NOT null THEN
        map personalDetails = new map()
        personalDetails.put("firstName", personRecord.FIRST_NAME)
        personalDetails.put("lastName", personRecord.LAST_NAME)
        personalDetails.put("birthDate", personRecord.BIRTH_DATE)
        personalDetails.put("gender", personRecord.GENDER)
        profileMap.put("personalDetails", personalDetails)
    END IF

    // Step 4: Harvest Identity Integration Cross-References
    LIST identRecords = SELECT FROM CDP_PARTY_IDENTIFICATION WHERE PARTY_ID == targetPartyId
    map externalIds = new map()
    FOR EACH row IN identRecords
        externalIds.put(row.PARTY_IDENT_TYPE_ID, row.ID_VALUE)
    END FOR
    profileMap.put("externalIdentifiers", externalIds)

    // Step 5: Gather Omni-Channel Contact Mechanisms (Active Records Only)
    LIST contactMechanisms = SELECT PCM.CONTACT_MECH_PURPOSE_TYPE_ID, CM.CONTACT_MECH_TYPE_ID, 
                                    EM.EMAIL_ADDRESS, EM.IS_VERIFIED,
                                    TEL.CONTACT_NUMBER,
                                    PA.ADDRESS1, PA.ADDRESS2, PA.CITY, PA.STATE_PROVINCE_GEO_ID, PA.COUNTRY_GEO_ID, PA.POSTAL_CODE, PA.TO_NAME
                             FROM CDP_PARTY_CONTACT_MECH PCM
                             JOIN CDP_CONTACT_MECHANISM CM ON PCM.CONTACT_MECH_ID == CM.CONTACT_MECH_ID
                             LEFT JOIN CDP_EMAIL_ADDRESS EM ON CM.CONTACT_MECH_ID == EM.CONTACT_MECH_ID
                             LEFT JOIN CDP_TELECOM_NUMBER TEL ON CM.CONTACT_MECH_ID == TEL.CONTACT_MECH_ID
                             LEFT JOIN CDP_POSTAL_ADDRESS PA ON CM.CONTACT_MECH_ID == PA.CONTACT_MECH_ID
                             WHERE PCM.PARTY_ID == targetPartyId AND (PCM.THRU_DATE is null OR PCM.THRU_DATE > SYSTEM.getCurrentTimestamp())

    list emailList = new list()
    list phoneList = new list()
    list addressList = new list()

    FOR EACH contact IN contactMechanisms
        map item = new map()
        item.put("purpose", contact.CONTACT_MECH_PURPOSE_TYPE_ID)
        
        IF contact.CONTACT_MECH_TYPE_ID == "EMAIL_ADDRESS" THEN
            item.put("email", contact.EMAIL_ADDRESS)
            item.put("verified", contact.IS_VERIFIED)
            emailList.add(item)
        ELSE IF contact.CONTACT_MECH_TYPE_ID == "TELECOM_NUMBER" THEN
            item.put("number", contact.CONTACT_NUMBER)
            phoneList.add(item)
        ELSE IF contact.CONTACT_MECH_TYPE_ID == "POSTAL_ADDRESS" THEN
            item.put("addressLine1", contact.ADDRESS1)
            item.put("addressLine2", contact.ADDRESS2)
            item.put("city", contact.CITY)
            item.put("state", contact.STATE_PROVINCE_GEO_ID)
            item.put("country", contact.COUNTRY_GEO_ID)
            item.put("postalCode", contact.POSTAL_CODE)
            item.put("recipientName", contact.TO_NAME)
            addressList.add(item)
        END IF
    END FOR

    profileMap.put("emails", emailList)
    profileMap.put("phones", phoneList)
    profileMap.put("addresses", addressList)

    // Step 6: Harvest Behavioral Metrics, Tags, Preferences, and Custom Attributes
    LIST tagsRecords = SELECT FROM CDP_PARTY_TAG WHERE PARTY_ID == targetPartyId
    list tagValues = new list()
    FOR EACH tag IN tagsRecords
        tagValues.add(tag.TAG_VALUE)
    END FOR
    profileMap.put("tags", tagValues)

    RECORD metricRecord = SELECT FROM CDP_PARTY_MARKETING_METRIC WHERE PARTY_ID == targetPartyId
    IF metricRecord is NOT null THEN
        map metrics = new map()
        metrics.put("ordersCount", metricRecord.ORDERS_COUNT)
        metrics.put("totalSpent", metricRecord.TOTAL_SPENT)
        metrics.put("lastOrderId", metricRecord.LAST_ORDER_ID)
        metrics.put("lastOrderName", metricRecord.LAST_ORDER_NAME)
        profileMap.put("metrics", metrics)
    END IF

    LIST prefRecords = SELECT FROM CDP_PARTY_PREFERENCE WHERE PARTY_ID == targetPartyId
    map preferences = new map()
    FOR EACH pref IN prefRecords
        preferences.put(pref.PARTY_PREF_TYPE_ID, pref.PREFERENCE_VALUE)
    END FOR
    profileMap.put("preferences", preferences)

    LIST attrRecords = SELECT FROM CDP_PARTY_ATTRIBUTE WHERE PARTY_ID == targetPartyId
    map attributes = new map()
    FOR EACH attr IN attrRecords
        attributes.put(attr.ATTR_NAME, attr.ATTR_VALUE)
    END FOR
    profileMap.put("customAttributes", attributes)

    // Step 7: Return Aggregated Structural Context Object Map
    SET customerProfileStructure = profileMap
    RETURN customerProfileStructure
END SERVICE METHOD

```

