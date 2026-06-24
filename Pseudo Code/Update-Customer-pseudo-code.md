## 🛠️ Updated Flow 3: Update Existing Profile (Immutable Address Pattern)

### 🎯 Step A: Textual Flowchart — Profile Update Engine

```text
[ INPUT PATH: Target Unique System Key: PartyId, Map containing new parameters: PersonalDetails, Addresses, Preferences ]
                           │
                           ▼
               🛑 [ 1. LOCK PROFILE TRANSACTION ]
               Begin Atomic DB Transaction with ROW-LEVEL EXCLUSIVE LOCK on CDP_PARTY
                           │
                           ▼
               🔎 [ 2. VERIFY IDENTITY RECORD EXISTENCE ]
               Query CDP_PARTY WHERE PARTY_ID == Input.PartyId
                                  / \
                                 /   \
                         NOT    /     \    FOUND
                        FOUND  /       \
                              ▼         ▼
     [ THROW: ProfileNotFoundException ]   ┌────────────────────────────────────────────────┐
     [ Rollback Active Transaction Unit]   │ Update CDP_PARTY.LAST_UPDATED_STAMP            │
     [ Terminate Execution Lifecycle   ]   └──────────────────────┬─────────────────────────┘
                                                                  │
                                                                  ▼
                                           💾 [ 3. RECONCILE BIOMETRIC CHANGES ]
                                           Is FirstName or LastName changed in payload?
                                              ├── YES ──► UPDATE CDP_PERSON SET fields WHERE PARTY_ID == PartyId
                                              └── NO  ──► Skip
                                                                  │
                                                                  ▼
                                     💾 [ 4. IMMUTABLE ADRESS COLLECTION RECONCILIATION ]
                                  🔁 Loop over each item inside incoming Addresses[] array
                                     Is there an active entry in CDP_PARTY_CONTACT_MECH for this purpose?
                                              │
                                              ├── YES ──► 🛑 Soft-Terminate Current Active Association:
                                              │             UPDATE CDP_PARTY_CONTACT_MECH 
                                              │             SET THRU_DATE = SYSTEM.getCurrentTimestamp()
                                              │             WHERE PARTY_ID == PartyId AND PURPOSE == currentPurpose
                                              │
                                              └── NO  ──► Skip directly to insertion
                                                                  │
                                                                  ▼
                                                 💾 🆕 [ 5. INSERT REPLACEMENT ADDRESS ]
                                                 1. Create brand new unique ContactMechId string
                                                 2. INSERT INTO CDP_CONTACT_MECHANISM ('POSTAL_ADDRESS')
                                                 3. INSERT INTO CDP_POSTAL_ADDRESS (Address text values)
                                                 4. INSERT INTO CDP_PARTY_CONTACT_MECH (New association mapping)
                                                                  │
                                                                  ▼
                                           💾 [ 6. RE-EVALUATE PREFERENCES & TRAITS ]
                                           Update or Insert into CDP_PARTY_PREFERENCE with updated consent fields
                                           Update metrics layer (CDP_PARTY_MARKETING_METRIC) with latest tracking sums
                                                                  │
                                                                  ▼
                                           / \────────────────────┘
                                          /   \
                                         /     \
                                  Any DB Errors? ── YES ──► [ ABORT ENGINE PIPELINE ]
                                         \     /             [ Execute Rollback & Restore Previous State ]
                                          \   /
                                           \ /
                                            │ NO
                                            ▼
                                 📜 [ 7. COMMIT PROFILE DELTA STATE ]
                                 Flush Changes Permanent into MySQL Ledger
                                 Return ExecutionSuccess = TRUE
                                 [ SUCCESS STOP ]

```

---

### 📝 Step B: Language-Agnostic Pseudo-code — Profile Update

```text
SERVICE METHOD: party.updateCustomerProfile
    INPUT PARAMETERS:
        string targetPartyId
        map updatePayloadStructure
    
    OUTPUT PARAMETERS:
        boolean executionSuccess

    // Step 1: Initialize Database Transaction & Exclusive Lock Block
    BEGIN TRANSACTION
        TRY
            // Step 2: Confirm Profile Existence and Secure Mutation Intent via SELECT FOR UPDATE
            RECORD partyRecord = SELECT FOR UPDATE FROM CDP_PARTY
                                 WHERE PARTY_ID == targetPartyId
                                 
            IF partyRecord is null THEN
                RAISE EXCEPTION "Update Error: Profile with ID " + targetPartyId + " does not exist."
            END IF

            // Step 3: Mutate Biometric Data Columns
            IF updatePayloadStructure.contains("personalDetails") THEN
                map personal = updatePayloadStructure.get("personalDetails")
                
                UPDATE CDP_PERSON
                SET FIRST_NAME = COALESCE(personal.get("firstName"), FIRST_NAME),
                    LAST_NAME = COALESCE(personal.get("lastName"), LAST_NAME),
                    BIRTH_DATE = COALESCE(personal.get("birthDate"), BIRTH_DATE)
                WHERE PARTY_ID == targetPartyId
            END IF

            // Step 4: Reconcile Multi-Valued Postal Collections (Immutable History Pattern)
            IF updatePayloadStructure.contains("addresses") THEN
                list inboundAddresses = updatePayloadStructure.get("addresses")
                
                FOR EACH addr IN inboundAddresses
                    string currentPurpose = addr.get("purpose")
                    
                    // Look up any currently active record for this specific address purpose context
                    RECORD activeLocationAssociation = SELECT CONTACT_MECH_ID
                                                       FROM CDP_PARTY_CONTACT_MECH
                                                       WHERE PARTY_ID == targetPartyId
                                                         AND CONTACT_MECH_PURPOSE_TYPE_ID == currentPurpose
                                                         AND (THRU_DATE is null OR THRU_DATE > SYSTEM.getCurrentTimestamp())
                    
                    // If an active association exists, soft-decommission it by updating its thru-date
                    IF activeLocationAssociation is NOT null THEN
                        UPDATE CDP_PARTY_CONTACT_MECH
                        SET THRU_DATE = SYSTEM.getCurrentTimestamp()
                        WHERE PARTY_ID == targetPartyId
                          AND CONTACT_MECH_ID == activeLocationAssociation.CONTACT_MECH_ID
                          AND CONTACT_MECH_PURPOSE_TYPE_ID == currentPurpose
                    END IF
                    
                    // Bypassing string comparison completely: Always drop down a new address record entry
                    string newPostalMechId = SYSTEM.generateUUID()
                    
                    INSERT INTO CDP_CONTACT_MECHANISM (CONTACT_MECH_ID, CONTACT_MECH_TYPE_ID) 
                    VALUES (newPostalMechId, "POSTAL_ADDRESS")
                    
                    INSERT INTO CDP_POSTAL_ADDRESS (CONTACT_MECH_ID, ADDRESS1, ADDRESS2, CITY, STATE_PROVINCE_GEO_ID, COUNTRY_GEO_ID, POSTAL_CODE, TO_NAME)
                    VALUES (newPostalMechId, addr.get("addressLine1"), addr.get("addressLine2"), addr.get("city"), addr.get("state"), addr.get("country"), addr.get("postalCode"), addr.get("recipientName"))
                    
                    INSERT INTO CDP_PARTY_CONTACT_MECH (PARTY_ID, CONTACT_MECH_ID, CONTACT_MECH_PURPOSE_TYPE_ID, FROM_DATE, THRU_DATE)
                    VALUES (targetPartyId, newPostalMechId, currentPurpose, SYSTEM.getCurrentTimestamp(), null)
                END FOR
            END IF

            // Step 5: Update Preference Configurations
            IF updatePayloadStructure.contains("preferences") THEN
                map prefs = updatePayloadStructure.get("preferences")
                FOR EACH key IN prefs.keySet()
                    UPDATE CDP_PARTY_PREFERENCE
                    SET PREFERENCE_VALUE = prefs.get(key), FROM_DATE = SYSTEM.getCurrentTimestamp()
                    WHERE PARTY_ID == targetPartyId AND PARTY_PREF_TYPE_ID == key
                END FOR
            END IF

            // Update Master Profile Modification Timestamp
            UPDATE CDP_PARTY SET LAST_UPDATED_STAMP = SYSTEM.getCurrentTimestamp() WHERE PARTY_ID == targetPartyId

            // Step 6: Commit Final Pipeline State Changes
            COMMIT TRANSACTION
            SET executionSuccess = true
            RETURN executionSuccess

        CATCH DATABASE_EXCEPTION OR SYSTEM_EXCEPTION AS error
            // Step 7: Rollback and Fail State Interception
            ROLLBACK TRANSACTION
            LOG.error("Failed executing profile update transaction on ID " + targetPartyId + ". Trace: " + error.message)
            SET executionSuccess = false
            RETURN executionSuccess
        END TRY
END SERVICE METHOD

```
