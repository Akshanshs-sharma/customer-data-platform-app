### Flow 5.4: Transactional Database Upsert Committer.

---


```text
            [ INPUT PATH: List normalizedCustomersBuffer passed from Flow 5.3 ]
                                              │
                                              ▼
                        ┌───────────────────────────────────────────┐
                        │ 🔄 LOOP: For Each customer               │◄──────────────────────┐
                        │          In normalizedCustomersBuffer     │                       │
                        └─────────────────────┬─────────────────────┘                       │
                                              │                                             │
                                              ▼                                             │
                                   [ Start DB Transaction ]                                 │
                                   - Begin Isolated Connection Session                      │
                                   - Set Strategy = customer.pipeline_strategy              │
                                              │                                             │
                                              ▼                                             │
                                   [ Seed Layer Insulation ]                                │
                                   - INSERT IGNORE/ODKU into CDP_ROLE_TYPE                  │
                                     (Guarantees 'CUSTOMER' role type exists)               │
                                              │                                             │
                                            / \                                             │
                                           /   \                                            │
                                    Strategy == 'CREATE'?                                   │
                                           \   /                                            │
                                            \ /                                             │
                                             │                                              │
                                    YES ─────┴───── NO (UPDATE)                             │
                                     │                │                                     │
                                     ▼                ▼                                     │
                        [ Insert Root Identity ]   [ Update Core Identity ]                 │
                        - Insert into CDP_PARTY    - UPDATE CDP_PARTY timestamp             │
                        - Insert into CDP_PERSON   - UPDATE CDP_PERSON biographical         │
                        - Insert CDP_PARTY_ROLE    - Check & reactivate soft-deleted role   │
                          (CUSTOMER role link)       (If expired, insert new active role)   │
                                     │                │                                     │
                                     └────────┬───────┘                                     │
                                              │                                             │
                                              ▼                                             │
                                 [ High-Perf Upsert Identifiers ]                           │
                                 - INSERT ... ON DUPLICATE KEY UPDATE                       │
                                 - Maps 'SHOPIFY_CUST_ID' & 'SHOPIFY_GRAPHQL_GID'           │
                                              │                                             │
                                              ▼                                             │
                                 [ Reconcile Primary Email ]                                │
                                 - If Strategy == UPDATE and string changed:                │
                                   - Soft-terminate old active association (THRU_DATE=NOW)  │
                                 - If new record required:                                  │
                                   - 1st: Insert parent CDP_CONTACT_MECHANISM ◄── [FIXED]   │
                                   - 2nd: Insert domain child CDP_EMAIL_ADDRESS             │
                                   - 3rd: Link active CDP_PARTY_CONTACT_MECH                │
                                              │                                             │
                                              ▼                                             │
                                 [ Reconcile Primary Phone ]                                │
                                 - If Strategy == UPDATE and string changed:                │
                                   - Soft-terminate old active association (THRU_DATE=NOW)  │
                                 - If new record required:                                  │
                                   - 1st: Insert parent CDP_CONTACT_MECHANISM ◄── [FIXED]   │
                                   - 2nd: Insert domain child CDP_TELECOM_NUMBER            │
                                   - 3rd: Link active CDP_PARTY_CONTACT_MECH                │
                                              │                                             │
                                              ▼                                             │
                                 [ Reconcile Geographic Addresses ]                         │
                                 - Loop through flattened_addresses array:                  │
                                   - Soft-terminate old active record for specific purpose  │
                                   - 1st: Insert parent CDP_CONTACT_MECHANISM ◄── [FIXED]   │
                                   - 2nd: Insert child CDP_POSTAL_ADDRESS                   │
                                   - 3rd: Link active address CDP_PARTY_CONTACT_MECH        │
                                   - Inline Phone? Repeat parent/child telecom flow above   │
                                              │                                             │
                                              ▼                                             │
                                 [ Refresh Profiles, Traits & Metrics ]                     │
                                 - Clear and reload CDP_PARTY_TAG rows                      │
                                 - ODKU Upsert CDP_PARTY_PREFERENCE (Handles microsecond)   │
                                 - ODKU Upsert CDP_PARTY_ATTRIBUTE (Custom metadata notes)  │
                                 - ODKU Upsert CDP_PARTY_MARKETING_METRIC (Cache targets)   │
                                              │                                             │
                                              ▼                                             │
                                   / \────────┴────────/ \                                  │
                                  /   \               /   \                                 │
                                Any DB Errors?      Success Commit                          │
                                  \   /               \   /                                 │
                                   \ /                 \ /                                  │
                                    │ YES               │ NO                                │
                                    ▼                   ▼                                   │
                          [ Rollback Transaction ]   [ Commit Transaction ]                 │
                          - Undo alterations for     - Save modifications permanently       │
                            this customer profile    - Log clean record iteration           │
                                    │                   │                                   │
                                    └─────────┬─────────┘                                   │
                                              │                                             │
                                              ▼                                             │
                                     / \──────┴──────/ \                                    │
                                    /   \           /   \                                   │
                                  More Customers? ── YES ───────────────────────────────────┘
                                    \   /           /   \                                   
                                     \ /             \ /                                    
                                      │ NO                                                  
                                      ▼                                                     
                               [ SUCCESS STOP ]                                             

```

---


### 📝 Step B: Corrected Language-Agnostic Pseudo-code — Transactional Committer

```text
SERVICE METHOD: pipeline.commitTransactionalDatabaseUpsert
    INPUT PARAMETERS:
        LIST normalizedCustomersBuffer
    
    OUTPUT PARAMETERS:
        boolean commitBatchSuccess

    DATABASE_CONNECTION db = DATABASE.getConnection()
    commitBatchSuccess = true

    FOR EACH MAP customer IN normalizedCustomersBuffer DO
        string partyId = customer.get("assigned_party_id")
        string strategy = customer.get("pipeline_strategy")
        timestamp now = SYSTEM.getCurrentTimestamp()

        // Open an isolated atomic transaction per customer profile block
        BEGIN TRANSACTION ON db
            TRY
                // --- SEED LAYER INSULATION ---
                // Guarantee role runtime safety bounds prior to profile inheritance attachment
                INSERT INTO CDP_ROLE_TYPE (ROLE_TYPE_ID, DESCRIPTION) 
                VALUES ("CUSTOMER", "Omni-channel Core Consumer Base Profile Account")
                ON DUPLICATE KEY UPDATE DESCRIPTION = VALUES(DESCRIPTION)

                IF strategy == "CREATE" THEN
                    // 1. Persist Root Identity Row
                    INSERT INTO CDP_PARTY (PARTY_ID, PARTY_TYPE_ID, CREATED_STAMP, LAST_UPDATED_STAMP)
                    VALUES (partyId, "PERSON", customer.get("created_stamp"), customer.get("last_updated_stamp"))

                    // 2. Persist Biographical Characteristics
                    INSERT INTO CDP_PERSON (PARTY_ID, FIRST_NAME, LAST_NAME, GENDER, BIRTH_DATE)
                    VALUES (partyId, customer.get("first_name"), customer.get("last_name"), null, null)

                    // 3. Register Customer Role Binding
                    INSERT INTO CDP_PARTY_ROLE (PARTY_ID, ROLE_TYPE_ID, FROM_DATE, THRU_DATE)
                    VALUES (partyId, "CUSTOMER", now, null)
                ELSE
                    // 1. Refresh System Operational Update Timestamps
                    UPDATE CDP_PARTY 
                    SET LAST_UPDATED_STAMP = now 
                    WHERE PARTY_ID = partyId

                    // 2. Refresh Biographical Identity Vectors
                    UPDATE CDP_PERSON 
                    SET FIRST_NAME = customer.get("first_name"), LAST_NAME = customer.get("last_name") 
                    WHERE PARTY_ID = partyId

                    // 3. Reactivate Expired Customer Association Role Records if soft-deleted
                    RECORD activeRole = SELECT FROM CDP_PARTY_ROLE 
                                        WHERE PARTY_ID = partyId AND ROLE_TYPE_ID = "CUSTOMER" 
                                          AND (THRU_DATE IS NULL OR THRU_DATE > now)
                    
                    IF activeRole IS NULL THEN
                        INSERT INTO CDP_PARTY_ROLE (PARTY_ID, ROLE_TYPE_ID, FROM_DATE, THRU_DATE)
                        VALUES (partyId, "CUSTOMER", now, null)
                    END IF
                END IF

                // 4. Upsert External Cross-Reference Mapping Identifiers
                CALL db.upsertPartyIdentification(partyId, "SHOPIFY_GRAPHQL_GID", customer.get("graphql_gid"))
                CALL db.upsertPartyIdentification(partyId, "SHOPIFY_CUST_ID", customer.get("shopify_cust_id"))
                
                IF customer.get("shopify_multipass_id") IS NOT NULL THEN
                    CALL db.upsertPartyIdentification(partyId, "SHOPIFY_MULTIPASS_ID", customer.get("shopify_multipass_id"))
                END IF

                // 5. Reconcile Primary Email Vectors (Temporal Immutable Pattern)
                IF customer.get("email_address") IS NOT NULL THEN
                    RECORD activeEmail = SELECT pc.CONTACT_MECH_ID, e.EMAIL_ADDRESS 
                                         FROM CDP_PARTY_CONTACT_MECH pc
                                         JOIN CDP_EMAIL_ADDRESS e ON pc.CONTACT_MECH_ID = e.CONTACT_MECH_ID
                                         WHERE pc.PARTY_ID = partyId AND pc.CONTACT_MECH_PURPOSE_TYPE_ID = "PRIMARY_EMAIL"
                                           AND (pc.THRU_DATE IS NULL OR pc.THRU_DATE > now)
                    
                    IF activeEmail IS NULL OR activeEmail.get("EMAIL_ADDRESS") != customer.get("email_address") THEN
                        IF activeEmail IS NOT NULL THEN
                            UPDATE CDP_PARTY_CONTACT_MECH 
                            SET THRU_DATE = now 
                            WHERE PARTY_ID = partyId AND CONTACT_MECH_ID = activeEmail.get("CONTACT_MECH_ID")
                        END IF

                        string emailMechId = SYSTEM.generateUUID()
                        INSERT INTO CDP_CONTACT_MECHANISM (CONTACT_MECH_ID, CONTACT_MECH_TYPE_ID) VALUES (emailMechId, "EMAIL_ADDRESS")
                        INSERT INTO CDP_EMAIL_ADDRESS (CONTACT_MECH_ID, EMAIL_ADDRESS, IS_VERIFIED) VALUES (emailMechId, customer.get("email_address"), customer.get("email_verified"))
                        INSERT INTO CDP_PARTY_CONTACT_MECH (PARTY_ID, CONTACT_MECH_ID, CONTACT_MECH_PURPOSE_TYPE_ID, FROM_DATE, THRU_DATE)
                        VALUES (partyId, emailMechId, "PRIMARY_EMAIL", now, null)
                    END IF
                END IF

                // 6. Reconcile Primary Telecom Channels (Temporal Immutable Pattern)
                IF customer.get("raw_phone") IS NOT NULL THEN
                    string cleanPhone = STRING.replaceAll(customer.get("raw_phone"), "[^0-9+]", "")
                    RECORD activePhone = SELECT pc.CONTACT_MECH_ID, t.CONTACT_NUMBER 
                                         FROM CDP_PARTY_CONTACT_MECH pc
                                         JOIN CDP_TELECOM_NUMBER t ON pc.CONTACT_MECH_ID = t.CONTACT_MECH_ID
                                         WHERE pc.PARTY_ID = partyId AND pc.CONTACT_MECH_PURPOSE_TYPE_ID = "PRIMARY_PHONE"
                                           AND (pc.THRU_DATE IS NULL OR pc.THRU_DATE > now)

                    IF activePhone IS NULL OR activePhone.get("CONTACT_NUMBER") != cleanPhone THEN
                        IF activePhone IS NOT NULL THEN
                            UPDATE CDP_PARTY_CONTACT_MECH 
                            SET THRU_DATE = now 
                            WHERE PARTY_ID = partyId AND CONTACT_MECH_ID = activePhone.get("CONTACT_MECH_ID")
                        END IF

                        string phoneMechId = SYSTEM.generateUUID()
                        INSERT INTO CDP_CONTACT_MECHANISM (CONTACT_MECH_ID, CONTACT_MECH_TYPE_ID) VALUES (phoneMechId, "TELECOM_NUMBER")
                        INSERT INTO CDP_TELECOM_NUMBER (CONTACT_MECH_ID, COUNTRY_CODE, AREA_CODE, CONTACT_NUMBER) VALUES (phoneMechId, null, null, cleanPhone)
                        INSERT INTO CDP_PARTY_CONTACT_MECH (PARTY_ID, CONTACT_MECH_ID, CONTACT_MECH_PURPOSE_TYPE_ID, FROM_DATE, THRU_DATE)
                        VALUES (partyId, phoneMechId, "PRIMARY_PHONE", now, null)
                    END IF
                END IF

                // 7. Reconcile Flattened Geographic Address Structures
                LIST inboundAddresses = customer.get("flattened_addresses")
                FOR EACH MAP addr IN inboundAddresses DO
                    string purpose = addr.get("purpose_type_id")

                    RECORD activeAddressAssoc = SELECT CONTACT_MECH_ID 
                                                 FROM CDP_PARTY_CONTACT_MECH
                                                 WHERE PARTY_ID = partyId AND CONTACT_MECH_PURPOSE_TYPE_ID = purpose
                                                   AND (THRU_DATE IS NULL OR THRU_DATE > now)
                    
                    IF activeAddressAssoc IS NOT NULL THEN
                        UPDATE CDP_PARTY_CONTACT_MECH 
                        SET THRU_DATE = now 
                        WHERE PARTY_ID = partyId AND CONTACT_MECH_ID = activeAddressAssoc.get("CONTACT_MECH_ID")
                    END IF

                    string postalMechId = SYSTEM.generateUUID()
                    INSERT INTO CDP_CONTACT_MECHANISM (CONTACT_MECH_ID, CONTACT_MECH_TYPE_ID) VALUES (postalMechId, "POSTAL_ADDRESS")
                    INSERT INTO CDP_POSTAL_ADDRESS (CONTACT_MECH_ID, ADDRESS1, ADDRESS2, CITY, STATE_PROVINCE_GEO_ID, COUNTRY_GEO_ID, POSTAL_CODE, TO_NAME)
                    VALUES (postalMechId, addr.get("address1"), addr.get("address2"), addr.get("city"), addr.get("state_province_geo_id"), addr.get("country_geo_id"), addr.get("postal_code"), addr.get("to_name"))
                    
                    INSERT INTO CDP_PARTY_CONTACT_MECH (PARTY_ID, CONTACT_MECH_ID, CONTACT_MECH_PURPOSE_TYPE_ID, FROM_DATE, THRU_DATE)
                    VALUES (partyId, postalMechId, purpose, now, null)

                    // Secondary Address-Inline Phone Synchronization
                    IF addr.get("address_inline_phone") IS NOT NULL THEN
                        string cleanShipPhone = STRING.replaceAll(addr.get("address_inline_phone"), "[^0-9+]", "")
                        
                        RECORD activeShipPhoneAssoc = SELECT CONTACT_MECH_ID 
                                                       FROM CDP_PARTY_CONTACT_MECH
                                                       WHERE PARTY_ID = partyId AND CONTACT_MECH_PURPOSE_TYPE_ID = "SHIPPING_PHONE"
                                                         AND (THRU_DATE IS NULL OR THRU_DATE > now)
                        IF activeShipPhoneAssoc IS NOT NULL THEN
                            UPDATE CDP_PARTY_CONTACT_MECH 
                            SET THRU_DATE = now 
                            WHERE PARTY_ID = partyId AND CONTACT_MECH_ID = activeShipPhoneAssoc.get("CONTACT_MECH_ID")
                        END IF

                        string shipPhoneMechId = SYSTEM.generateUUID()
                        INSERT INTO CDP_CONTACT_MECHANISM (CONTACT_MECH_ID, CONTACT_MECH_TYPE_ID) VALUES (shipPhoneMechId, "TELECOM_NUMBER")
                        INSERT INTO CDP_TELECOM_NUMBER (CONTACT_MECH_ID, COUNTRY_CODE, AREA_CODE, CONTACT_NUMBER) VALUES (shipPhoneMechId, null, null, cleanShipPhone)
                        INSERT INTO CDP_PARTY_CONTACT_MECH (PARTY_ID, CONTACT_MECH_ID, CONTACT_MECH_PURPOSE_TYPE_ID, FROM_DATE, THRU_DATE)
                        VALUES (partyId, shipPhoneMechId, "SHIPPING_PHONE", now, null)
                    END IF
                END FOR

                // 8. Reconcile Dynamic Custom Metadata Arrays (Delete-and-Reload Pattern)
                DELETE FROM CDP_PARTY_TAG WHERE PARTY_ID = partyId
                LIST tags = customer.get("tags_list")
                FOR EACH string tag IN tags DO
                    INSERT INTO CDP_PARTY_TAG (PARTY_ID, TAG_VALUE) VALUES (partyId, STRING.trim(tag))
                END FOR

                // 9. Reconcile Consent States & Communication Channels
                CALL db.upsertPartyPreference(partyId, "GLOBAL_MARKETING", customer.get("accepts_marketing_global") ? "subscribed" : "unsubscribed", now)
                
                IF customer.get("email_consent_state") IS NOT NULL THEN
                    CALL db.upsertPartyPreference(partyId, "EMAIL_MARKETING", customer.get("email_consent_state"), customer.get("email_consent_updated_at"))
                    CALL db.upsertPartyAttribute(partyId, "email_marketing_opt_in_level", customer.get("email_opt_in_level"))
                END IF

                IF customer.get("sms_consent_state") IS NOT NULL THEN
                    CALL db.upsertPartyPreference(partyId, "SMS_MARKETING", customer.get("sms_consent_state"), customer.get("sms_consent_updated_at"))
                    CALL db.upsertPartyAttribute(partyId, "sms_consent_source", customer.get("sms_consent_source"))
                    CALL db.upsertPartyAttribute(partyId, "sms_marketing_opt_in_level", customer.get("sms_opt_in_level"))
                END IF

                // 10. Reconcile Semi-Structured Attributes
                CALL db.upsertPartyAttribute(partyId, "shopify_customer_note", customer.get("shopify_customer_note"))
                CALL db.upsertPartyAttribute(partyId, "shopify_account_state", customer.get("shopify_account_state"))
                CALL db.upsertPartyAttribute(partyId, "shopify_tax_exempt", customer.get("shopify_tax_exempt") ? "true" : "false")
                CALL db.upsertPartyAttribute(partyId, "shopify_preferred_currency", customer.get("preferred_currency"))
                CALL db.upsertPartyAttribute(partyId, "shopify_marketing_opt_in_level", customer.get("marketing_opt_in_level"))

                // 11. Reconcile Cache Aggregation Targets
                CALL db.upsertMarketingMetrics(partyId, customer.get("orders_count"), customer.get("total_spent"), customer.get("last_order_id"), customer.get("last_order_name"))

                // Securely close customer isolated boundary context
                COMMIT TRANSACTION ON db
                LOG.info("Inbound synchronization successful for internal party tracker ID: " + partyId)

            CATCH EXCEPTION AS dbError
                // Roll back local isolation block without breaking neighboring records
                ROLLBACK TRANSACTION ON db
                LOG.error("Fatal transaction execution error on profile: " + partyId + ". Context dropped. Reason: " + dbError.message)
                commitBatchSuccess = false
            END TRY
    END FOR

    RETURN commitBatchSuccess
END SERVICE METHOD


// --- RE-ENGINEERED HIGH-PERFORMANCE SQL HELPER WRAPPERS ---

HELPER METHOD db.upsertPartyIdentification(string partyId, string idType, string val)
    IF val IS NOT NULL THEN
        INSERT INTO CDP_PARTY_IDENTIFICATION (PARTY_ID, PARTY_IDENT_TYPE_ID, ID_VALUE)
        VALUES (partyId, idType, val)
        ON DUPLICATE KEY UPDATE ID_VALUE = VALUES(ID_VALUE)
    END IF
END HELPER METHOD

HELPER METHOD db.upsertPartyPreference(string partyId, string prefType, string val, timestamp updateTime)
    IF val IS NOT NULL THEN
        // Explicitly handle identical sub-second processing timestamp overlap conflicts safely
        UPDATE CDP_PARTY_PREFERENCE 
        SET THRU_DATE = updateTime 
        WHERE PARTY_ID = partyId AND PARTY_PREF_TYPE_ID = prefType AND THRU_DATE IS NULL 
          AND FROM_DATE < updateTime
        
        INSERT INTO CDP_PARTY_PREFERENCE (PARTY_ID, PARTY_PREF_TYPE_ID, PREFERENCE_VALUE, FROM_DATE, THRU_DATE)
        VALUES (partyId, prefType, val, updateTime, null)
        ON DUPLICATE KEY UPDATE PREFERENCE_VALUE = VALUES(PREFERENCE_VALUE), THRU_DATE = null
    END IF
END HELPER METHOD

HELPER METHOD db.upsertPartyAttribute(string partyId, string attrName, string val)
    IF val IS NOT NULL THEN
        INSERT INTO CDP_PARTY_ATTRIBUTE (PARTY_ID, ATTR_NAME, ATTR_VALUE, DESCRIPTION)
        VALUES (partyId, attrName, val, null)
        ON DUPLICATE KEY UPDATE ATTR_VALUE = VALUES(ATTR_VALUE)
    END IF
END HELPER METHOD

HELPER METHOD db.upsertMarketingMetrics(string partyId, integer count, decimal spent, string orderId, string orderName)
    INSERT INTO CDP_PARTY_MARKETING_METRIC (PARTY_ID, ORDERS_COUNT, TOTAL_SPENT, LAST_ORDER_ID, LAST_ORDER_NAME, LAST_CALCULATED_STAMP)
    VALUES (partyId, count, spent, orderId, orderName, SYSTEM.getCurrentTimestamp())
    ON DUPLICATE KEY UPDATE 
        ORDERS_COUNT = VALUES(ORDERS_COUNT),
        TOTAL_SPENT = VALUES(TOTAL_SPENT),
        LAST_ORDER_ID = VALUES(LAST_ORDER_ID),
        LAST_ORDER_NAME = VALUES(LAST_ORDER_NAME),
        LAST_CALCULATED_STAMP = VALUES(LAST_CALCULATED_STAMP)
END HELPER METHOD

```
