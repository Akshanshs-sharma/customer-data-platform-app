## 🛠️ Moving Forward: Flow 4 — Soft-Delete Data Profile Lifecycle

### 🎯 Step A: Textual Flowchart — Soft-Delete Engine

```text
[ INPUT PATH: Target Unique Database System Key: PartyId ]
                           │
                           ▼
               📝 [ 1. START PRIVACY TRANSACTION ]
               Open Atomic Connection Block with Exclusive Row-Locking
                           │
                           ▼
               🔎 [ 2. VALIDATE TARGET IDENTITY ]
               Query CDP_PARTY WHERE PARTY_ID == Input.PartyId
                                  / \
                                 /   \
                         NOT    /     \    FOUND
                        FOUND  /       \
                              ▼         ▼
     [ THROW: ProfileNotFoundException ]   ┌────────────────────────────────────────────────┐
     [ Rollback Active Transaction Unit]   │ Proceed to Decommissioning Pipeline            │
     [ Terminate Execution Lifecycle   ]   └──────────────────────┬─────────────────────────┘
                                                                  │
                                                                  ▼
                                           💾 [ 3. CLOSE CONTACT MECHANISM TIME WINDOWS ]
                                           UPDATE CDP_PARTY_CONTACT_MECH 
                                           SET THRU_DATE = SYSTEM.getCurrentTimestamp()
                                           WHERE PARTY_ID == Input.PartyId AND THRU_DATE IS NULL
                                                                  │
                                                                  ▼
                                           💾 [ 4. DEACTIVATE BUSINESS ROLE CAPACITY ]
                                           UPDATE CDP_PARTY_ROLE 
                                           SET THRU_DATE = SYSTEM.getCurrentTimestamp()
                                           WHERE PARTY_ID == Input.PartyId AND THRU_DATE IS NULL
                                                                  │
                                                                  ▼
                                           💾 [ 5. MERGE PRIVACY COMPLIANCE PREFERENCES ]
                                           UPSERT INTO CDP_PARTY_PREFERENCE 
                                           (PARTY_ID, PARTY_PREF_TYPE_ID, PREFERENCE_VALUE, FROM_DATE)
                                           VALUES (Input.PartyId, 'GLOBAL_MARKETING', 'unsubscribed', NOW)
                                                                  │
                                                                  ▼
                                           💾 [ 6. COMPACT IDENTIFICATION CLUSTER ]
                                           UPDATE CDP_PARTY 
                                           SET LAST_UPDATED_STAMP = SYSTEM.getCurrentTimestamp()
                                           WHERE PARTY_ID == Input.PartyId
                                                                  │
                                                                  ▼
                                           / \────────────────────┘
                                          /   \
                                         /     \
                                  Any DB Errors? ── YES ──► [ HALT ENGINE PIPELINE ]
                                         \     /             [ Execute Transaction Rollback ]
                                          \   /
                                           \ /
                                            │ NO
                                            ▼
                                 📜 [ 7. COMMIT AUDIT STATE ]
                                 Flush Invalidation Changes into MySQL Ledger
                                 Return ExecutionSuccess = TRUE
                                 [ SUCCESS STOP ]

```

---

### 📝 Step B: Language-Agnostic Pseudo-code — Soft-Delete

```text
SERVICE METHOD: party.softDeleteCustomerProfile
    INPUT PARAMETERS:
        string targetPartyId
    
    OUTPUT PARAMETERS:
        boolean executionSuccess

    // Step 1: Open Secure Database Isolation Session
    BEGIN TRANSACTION
        TRY
            // Step 2: Verify Profile Existence and Guard Against Orphaning Records
            RECORD partyRecord = SELECT FOR UPDATE FROM CDP_PARTY
                                 WHERE PARTY_ID == targetPartyId
                                 
            IF partyRecord is null THEN
                RAISE EXCEPTION "Deletion Error: Target customer profile matching key " + targetPartyId + " does not exist."
            END IF

            // Step 3: Close Active Association Windows for Every Contact Mechanism Vector
            // This immediately marks all emails, phones, and addresses as inactive for this customer.
            UPDATE CDP_PARTY_CONTACT_MECH
            SET THRU_DATE = SYSTEM.getCurrentTimestamp()
            WHERE PARTY_ID == targetPartyId
              AND (THRU_DATE is null OR THRU_DATE > SYSTEM.getCurrentTimestamp())

            // Step 4: Revoke System Role Context
            // The person record remains, but they can no longer be retrieved by customer query filters.
            UPDATE CDP_PARTY_ROLE
            SET THRU_DATE = SYSTEM.getCurrentTimestamp()
            WHERE PARTY_ID == targetPartyId
              AND ROLE_TYPE_ID = "CUSTOMER"
              AND (THRU_DATE is null OR THRU_DATE > SYSTEM.getCurrentTimestamp())

            // Step 5: Enforce Global Marketing Opt-Out Rules
            // This ensures they are immediately removed from all outbound marketing communication campaigns.
            UPDATE CDP_PARTY_PREFERENCE
            SET PREFERENCE_VALUE = "unsubscribed",
                FROM_DATE = SYSTEM.getCurrentTimestamp()
            WHERE PARTY_ID == targetPartyId
              AND PARTY_PREF_TYPE_ID = "GLOBAL_MARKETING"

            // Step 6: Log Master Profile Disabling Date Stamp
            UPDATE CDP_PARTY
            SET LAST_UPDATED_STAMP = SYSTEM.getCurrentTimestamp()
            WHERE PARTY_ID == targetPartyId

            // Step 7: Commit Safe Changes to the Database
            COMMIT TRANSACTION
            SET executionSuccess = true
            RETURN executionSuccess

        CATCH DATABASE_EXCEPTION OR SYSTEM_EXCEPTION AS error
            // Step 8: Abort Pipeline and Restore State on System Error
            ROLLBACK TRANSACTION
            LOG.error("Failed to execute soft-delete lifecycle routine for ID " + targetPartyId + ". Trace: " + error.message)
            SET executionSuccess = false
            RETURN executionSuccess
        END TRY
END SERVICE METHOD

```

