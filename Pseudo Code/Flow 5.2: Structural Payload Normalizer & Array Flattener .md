### Flow 5.2: Structural Payload Normalizer & Array Flattener 

---

### 📊 Step A: Technical Flowchart — Payload Normalizer Engine

```text
       [ INPUT PATH: List customerEdges passed from Flow 5.1 ]
                                  │
                                  ▼
                    [ Initialize Clean Storage ]
               - Create List normalizedCustomersBuffer
                                  │
                                  ▼
               ┌──────────────────────────────────────┐
               │ 🔄 LOOP: For Each edge In customerEdges│◄──────────────────────┐
               └──────────────────┬───────────────────┘                       │
                                  │                                           │
                                  ▼                                           │
                   [ 1. EXTRACT ROOT NODE: edge.node ]                        │
                                  │                                           │
                                  ▼                                           │
               [ 2. INITIALIZE FLAT FLAT_CUSTOMER_MAP ]                       │
                                  │                                           │
                                  ▼                                           │
             [ 3. EXTRACT CORE IDENTITY & PRIMITIVE FIELDS ]                  │
             - Map id, legacyResourceId, firstName, lastName                  │
             - Convert email to Lowercase, extract phone                      │
             - Parse dates, note, state, taxExempt, multipass                 │
                                  │                                           │
                                  ▼                                           │
                [ 4. EXTRACT FLATTENED METRIC FIELDS ]                        │
                - Extract numberOfOrders (Int)                                │
                - Extract amountSpent.amount (Decimal/Float)                  │
                - Extract lastOrder.id, lastOrder.name                        │
                                  │                                           │
                                  ▼                                           │
               [ 5. EXTRACT MARKETING CONSENT STATE MAPS ]                    │
               - Map acceptsMarketing / acceptsMarketingUpdatedAt             │
               - Extract emailMarketingConsent / smsMarketingConsent profiles │
                                  │                                           │
                                  ▼                                           │
                     [ 6. EXTRACT TAGS ARRAY ]                                │
                     - Clean tags array strings directly                      │
                                  │                                           │
                                  ▼                                           │
                   [ 🛡️ SAFE ARRAY INITIALIZATION ]                           │
                   - Extract rawAddresses = node.get("addresses")             │
                   - IF rawAddresses IS NULL ──► Set rawAddresses = []        │
                                  │                                           │
                                  ▼                                           │
               ┌──────────────────────────────────────┐                       │
               │ 🔄 NESTED LOOP: For Each address     │◄──────────────┐       │
               │               In rawAddresses        │               │       │
               └──────────────────┬───────────────────┘               │       │
                                  │                                   │       │
                                  ▼                                   │       │
                 [ 7. CONSTRUCT FLAT_ADDRESS_MAP ]                    │       │
                 - Concatenate name lines -> TO_NAME                  │       │
                 - Map street lines, city, postal code                │       │
                 - Map provinceCode & countryCodeV2                   │       │
                 - 🛡️ Evaluate address.default safely:                │       │
                   TRUE  -> purpose = 'DEFAULT_LOCATION'              │       │
                   FALSE -> purpose = 'SHIPPING_LOCATION'             │       │
                 - Check inline address.phone -> SHIPPING_PHONE       │       │
                                  │                                   │       │
                                  ▼                                   │       │
                 [ Append FLAT_ADDRESS_MAP to Local List ]            │       │
                                  │                                   │       │
                                  ▼                                   │       │
                           / \────┴────/ \                            │       │
                          /   \       /   \                           │       │
                        More Addresses? ─── YES ──────────────────────┘       │
                          \   /       \   /                                   │
                           \ /         \ /                                    │
                            │ NO                                              │
                            ▼                                                 │
            [ Attach Flattened Address List to FLAT_CUSTOMER_MAP ]            │
                            │                                                 │
                            ▼                                                 │
         [ Append FLAT_CUSTOMER_MAP to normalizedCustomersBuffer ]            │
                            │                                                 │
                            ▼                                                 │
                     / \────┴────/ \                                          │
                    /   \       /   \                                         │
                  More Edges? ───── YES ──────────────────────────────────────┘
                    \   /       \   /                                         
                     \ /         \ /                                          
                      │ NO                                                    
                      ▼                                                       
   [ 8. PUSH MEMORY BUFFER TO STAGE 5.3 ]                                     
   - Pass normalizedCustomersBuffer to Idempotency Engine                     
   [ SUCCESS STOP ]                                                           

```

---

### 📝 Step B: Language-Agnostic Pseudo-code — Payload Normalizer

```text
SERVICE METHOD: pipeline.normalizeAndFlattenPayloadBatch
    INPUT PARAMETERS:
        LIST customerEdges
    
    OUTPUT PARAMETERS:
        boolean normalizationSuccess

    // Create an in-memory execution batch buffer to pass down to the Matcher
    LIST normalizedCustomersBuffer = []

    TRY
        FOR EACH MAP edge IN customerEdges DO
            MAP node = edge.get("node")
            
            // Initialize our flat data transfer object container
            MAP flatCustomer = {}

            // 1. Map Core Profile Attributes
            flatCustomer.put("graphql_gid", node.get("id"))
            flatCustomer.put("shopify_cust_id", STRING.valueOf(node.get("legacyResourceId")))
            flatCustomer.put("first_name", STRING.trim(node.get("firstName")))
            flatCustomer.put("last_name", STRING.trim(node.get("lastName")))
            flatCustomer.put("created_stamp", node.get("createdAt"))
            flatCustomer.put("last_updated_stamp", node.get("updatedAt"))
            
            // Enforce lowercase mapping invariant on primary communication lines
            IF node.get("email") IS NOT NULL THEN
                flatCustomer.put("email_address", STRING.toLowerCase(STRING.trim(node.get("email"))))
            ELSE
                flatCustomer.put("email_address", NULL)
            END IF
            flatCustomer.put("email_verified", node.get("verifiedEmail"))
            flatCustomer.put("raw_phone", node.get("phone"))

            // 2. Map Customer Base Attributes
            flatCustomer.put("shopify_customer_note", node.get("note"))
            flatCustomer.put("shopify_account_state", node.get("state"))
            flatCustomer.put("shopify_tax_exempt", node.get("taxExempt"))
            flatCustomer.put("shopify_multipass_id", node.get("multipassIdentifier"))
            
            IF node.get("defaultCurrency") IS NOT NULL THEN
                flatCustomer.put("preferred_currency", node.get("defaultCurrency").get("isoCode"))
            ELSE
                flatCustomer.put("preferred_currency", NULL)
            END IF

            // 3. Map Marketing Metrics Attributes
            flatCustomer.put("orders_count", node.get("numberOfOrders"))
            
            IF node.get("amountSpent") IS NOT NULL THEN
                flatCustomer.put("total_spent", DECIMAL.valueOf(node.get("amountSpent").get("amount")))
            ELSE
                flatCustomer.put("total_spent", 0.0000)
            END IF

            IF node.get("lastOrder") IS NOT NULL THEN
                flatCustomer.put("last_order_id", node.get("lastOrder").get("id"))
                flatCustomer.put("last_order_name", node.get("lastOrder").get("name"))
            ELSE
                flatCustomer.put("last_order_id", NULL)
                flatCustomer.put("last_order_name", NULL)
            END IF

            // 4. Map Omnichannel Preferences Consent State Tracking
            flatCustomer.put("accepts_marketing_global", node.get("acceptsMarketing"))
            flatCustomer.put("global_consent_updated_at", node.get("acceptsMarketingUpdatedAt"))
            flatCustomer.put("marketing_opt_in_level", node.get("marketingOptInLevel"))

            // Parse detailed inner consent nodes safely
            IF node.get("emailMarketingConsent") IS NOT NULL THEN
                MAP emailConsent = node.get("emailMarketingConsent")
                flatCustomer.put("email_consent_state", emailConsent.get("marketingState"))
                flatCustomer.put("email_opt_in_level", emailConsent.get("optInLevel"))
                flatCustomer.put("email_consent_updated_at", emailConsent.get("consentUpdatedAt"))
            END IF

            IF node.get("smsMarketingConsent") IS NOT NULL THEN
                MAP smsConsent = node.get("smsMarketingConsent")
                flatCustomer.put("sms_consent_state", smsConsent.get("marketingState"))
                flatCustomer.put("sms_opt_in_level", smsConsent.get("optInLevel"))
                flatCustomer.put("sms_consent_updated_at", smsConsent.get("consentUpdatedAt"))
                flatCustomer.put("sms_consent_source", smsConsent.get("consentCollectedFrom"))
            END IF

            // 5. Extract Core Profile Tags Array
            flatCustomer.put("tags_list", node.get("tags")) // Array of strings pass-through

            // 6. Execute Nested Array Processing for Location Contexts
            LIST rawAddresses = node.get("addresses")
            
            // CRITICAL PRODUCTION FIX: Guard against null addresses object
            IF rawAddresses IS NULL THEN
                rawAddresses = []
            END IF
            
            LIST flattenedAddresses = []

            FOR EACH MAP address IN rawAddresses DO
                MAP flatAddress = {}
                
                // Enforce Attention Name Concatenation Invariant cleanly
                string attentionName = ""
                IF address.get("firstName") IS NOT NULL THEN attentionName = attentionName + address.get("firstName") + " " END IF
                IF address.get("lastName") IS NOT NULL THEN attentionName = attentionName + address.get("lastName") END IF
                IF address.get("company") IS NOT NULL THEN attentionName = attentionName + " - " + address.get("company") END IF
                flatAddress.put("to_name", STRING.trim(attentionName))

                // Map physical geographic keys
                flatAddress.put("address1", address.get("address1"))
                flatAddress.put("address2", address.get("address2"))
                flatAddress.put("city", address.get("city"))
                flatAddress.put("state_province_geo_id", address.get("provinceCode"))
                flatAddress.put("country_geo_id", address.get("countryCodeV2"))
                flatAddress.put("postal_code", address.get("zip"))
                flatAddress.put("address_inline_phone", address.get("phone"))

                // CRITICAL PRODUCTION FIX: Null-safe boolean conversion check
                IF address.get("default") IS NOT NULL AND boolean.valueOf(address.get("default")) == true THEN
                    flatAddress.put("purpose_type_id", "DEFAULT_LOCATION")
                ELSE
                    flatAddress.put("purpose_type_id", "SHIPPING_LOCATION")
                END IF

                flattenedAddresses.add(flatAddress)
            END FOR

            flatCustomer.put("flattened_addresses", flattenedAddresses)
            
            // Push completely safe flat data representation into the loop execution array
            normalizedCustomersBuffer.add(flatCustomer)
        END FOR

        // Advance to Phase 5.3: Identity Resolution matching pipeline
        CALL pipeline.evaluateIdempotencyAndMatchProfiles(normalizedCustomersBuffer)
        SET normalizationSuccess = true
        RETURN normalizationSuccess

    CATCH EXCEPTION AS mappingFailure
        LOG.error("Fatal Operational Failure: Inbound normalization transformation broken. Trace: " + mappingFailure.message)
        SET normalizationSuccess = false
        RETURN normalizationSuccess
    END TRY
END SERVICE METHOD

```

---
