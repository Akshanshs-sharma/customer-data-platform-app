### Flow 5.3: Idempotency Evaluator & Profile Matcher 

---

### 📊 Step A: Technical Flowchart — Multi-Key Identity Matcher

```text
       [ INPUT PATH: List normalizedCustomersBuffer passed from Flow 5.2 ]
                                         │
                                         ▼
                     ┌───────────────────────────────────────┐
                     │ 🔄 LOOP: For Each customer            │◄──────────────────────┐
                     │          In normalizedCustomersBuffer │                       │
                     └───────────────────┬───────────────────┘                       │
                                         │                                           │
                                         ▼                                           │
                           [ Extract Matching Keys ]                                 │
                           - targetNumericId = customer.shopify_cust_id              │
                           - targetGid       = customer.graphql_gid                  │
                           - targetEmail     = customer.email_address                │
                                         │                                           │
                                         ▼                                           │
                      🔍 [ LOOKUP 1: Query by SHOPIFY_CUST_ID ]                      │
                      - SELECT PARTY_ID FROM CDP_PARTY_IDENTIFICATION                │
                        WHERE ID_VALUE = targetNumericId                            │
                        AND PARTY_IDENT_TYPE_ID = 'SHOPIFY_CUST_ID'                  │
                                         │                                           │
                                       / \─────────────────────────┐                 │
                                      /   \                        │                 │
                                Match Found?                       │                 │
                                      \   /                        │                 │
                                       \ /                         │                 │
                                        │ NO                       │ YES             │
                                        ▼                          ▼                 │
                      🔍 [ LOOKUP 2: Query by SHOPIFY_GRAPHQL_GID ]│                 │
                      - SELECT PARTY_ID FROM CDP_PARTY_IDENTIFICATION                │
                        WHERE ID_VALUE = targetGid                 │                 │
                        AND PARTY_IDENT_TYPE_ID = 'SHOPIFY_GRAPHQL_GID'              │
                                         │                         │                 │
                                       / \───────────────┐         │                 │
                                      /   \              │         │                 │
                                Match Found?             │         │                 │
                                      \   /              │         │                 │
                                       \ /               │         │                 │
                                        │ NO             │ YES     │                 │
                                        ▼                ▼         ▼                 │
                      🔍 [ LOOKUP 3: Query by Active EMAIL ]       │                 │
                      - SELECT pc.PARTY_ID FROM CDP_EMAIL_ADDRESS e                  │
                        JOIN CDP_PARTY_CONTACT_MECH pc                               │
                          ON e.CONTACT_MECH_ID = pc.CONTACT_MECH_ID                  │
                        WHERE e.EMAIL_ADDRESS = targetEmail                          │
                        AND pc.CONTACT_MECH_PURPOSE_TYPE_ID = 'PRIMARY_EMAIL'        │
                        AND (pc.THRU_DATE IS NULL OR pc.THRU_DATE > NOW())           │
                                         │               │         │                 │
                                       / \──────┐        │         │                 │
                                      /   \     │        │         │                 │
                                Match Found?    │        │         │                 │
                                      \   /     │        │         │                 │
                                       \ /      │        │         │                 │
                                        │ NO    │ YES    │         │                 │
                                        ▼       ▼        ▼         ▼                 │
                            ┌───────────────────────┐┌───────────────────────┐       │
                            │  📌 STRATEGY = CREATE ││  📌 STRATEGY = UPDATE │       │
                            │  - matchedPartyId     ││  - matchedPartyId     │       │
                            │    = GENERATE_UUID()  ││    = Found PARTY_ID   │       │
                            └───────────┬───────────┘└───────────┬───────────┘       │
                                        │                        │                   │
                                        └───────────┬────────────┘                   │
                                                    │                                │
                                                    ▼                                │
                                    [ Enforce Strategy Parameters ]                  │
                                    - customer.put("pipeline_strategy", strategy)    │
                                    - customer.put("assigned_party_id", matchedPartyId)
                                                    │                                │
                                                    ▼                                │
                                           / \──────┴──────/ \                       │
                                          /   \           /   \                      │
                                        More Profiles? ─── YES ──────────────────────┘
                                          \   /           /   \                      
                                           \ /             \ /                       
                                            │ NO                                     
                                            ▼                                        
                         [ PUSH ANNOTATED BUFFER TO STAGE 5.4 ]                      
                         - Pass normalizedCustomersBuffer to Committer               
                         [ SUCCESS STOP ]                                            

```

---

### 📝 Step B: Language-Agnostic Pseudo-code — Identity Matcher Engine

```text
SERVICE METHOD: pipeline.evaluateIdempotencyAndMatchProfiles
    INPUT PARAMETERS:
        LIST normalizedCustomersBuffer
    
    OUTPUT PARAMETERS:
        boolean matchingExecutionSuccess

    TRY
        // Acquire an active read-committed connection context from the database pool
        DATABASE_CONNECTION db = DATABASE.getConnection()

        FOR EACH MAP customer IN normalizedCustomersBuffer DO
            string targetNumericId = customer.get("shopify_cust_id")
            string targetGid = customer.get("graphql_gid")
            string targetEmail = customer.get("email_address")
            
            string resolvedPartyId = null
            string executionStrategy = "CREATE" // Default pipeline route configuration

            // --- IDENTITY RESOLUTION STEP 1: RESOLVE BY NUMERIC NATURAL KEY ---
            IF targetNumericId IS NOT NULL AND resolvedPartyId IS NULL THEN
                SQL_QUERY query1 = db.prepare("
                    SELECT PARTY_ID 
                    FROM CDP_PARTY_IDENTIFICATION 
                    WHERE ID_VALUE = ? AND PARTY_IDENT_TYPE_ID = 'SHOPIFY_CUST_ID' 
                    LIMIT 1
                ")
                SQL_RESULT result1 = query1.execute(targetNumericId)
                IF result1.hasRows() THEN
                    resolvedPartyId = result1.row(0).get("PARTY_ID")
                    executionStrategy = "UPDATE"
                END IF
            END IF

            // --- IDENTITY RESOLUTION STEP 2: RESOLVE BY GLOBAL GRAPHQL GID ---
            IF targetGid IS NOT NULL AND resolvedPartyId IS NULL THEN
                SQL_QUERY query2 = db.prepare("
                    SELECT PARTY_ID 
                    FROM CDP_PARTY_IDENTIFICATION 
                    WHERE ID_VALUE = ? AND PARTY_IDENT_TYPE_ID = 'SHOPIFY_GRAPHQL_GID' 
                    LIMIT 1
                ")
                SQL_RESULT result2 = query2.execute(targetGid)
                IF result2.hasRows() THEN
                    resolvedPartyId = result2.row(0).get("PARTY_ID")
                    executionStrategy = "UPDATE"
                END IF
            END IF

            // --- IDENTITY RESOLUTION STEP 3: RESOLVE BY PRIMARY ACTIVE LOWERCASE EMAIL ---
            // If soft-deleted (pc.THRU_DATE <= NOW()), this evaluation returns false, 
            // initiating a compliant new consumer lifecycle via a CREATE strategy.
            IF targetEmail IS NOT NULL AND resolvedPartyId IS NULL THEN
                SQL_QUERY query3 = db.prepare("
                    SELECT pc.PARTY_ID 
                    FROM CDP_EMAIL_ADDRESS e
                    INNER JOIN CDP_PARTY_CONTACT_MECH pc 
                       ON e.CONTACT_MECH_ID = pc.CONTACT_MECH_ID
                    WHERE e.EMAIL_ADDRESS = ? 
                      AND pc.CONTACT_MECH_PURPOSE_TYPE_ID = 'PRIMARY_EMAIL'
                      AND (pc.THRU_DATE IS NULL OR pc.THRU_DATE > NOW())
                    LIMIT 1
                ")
                SQL_RESULT result3 = query3.execute(targetEmail)
                IF result3.hasRows() THEN
                    resolvedPartyId = result3.row(0).get("PARTY_ID")
                    executionStrategy = "UPDATE"
                END IF
            END IF

            // --- STRATEGY PARAMETER ENFORCEMENT STAGE ---
            IF executionStrategy == "CREATE" THEN
                // Core identity does not exist or old lifecycle was terminated. Allocate a new universal tracking code.
                resolvedPartyId = SECURITY.generateUuidString()
                LOG.debug("Identity Resolution: Routing to CREATE profile strategy. Assigned Party ID: " + resolvedPartyId)
            ELSE
                LOG.debug("Identity Resolution: Active match hit discovered. Routing to UPDATE profile strategy for Party ID: " + resolvedPartyId)
            END IF

            // Annotate the memory map data object directly with runtime pipeline directives
            customer.put("pipeline_strategy", executionStrategy)
            customer.put("assigned_party_id", resolvedPartyId)
        END FOR

        // Advance to Phase 5.4: Transactional Database Upsert Committer
        CALL pipeline.commitTransactionalDatabaseUpsert(normalizedCustomersBuffer)
        
        SET matchingExecutionSuccess = true
        RETURN matchingExecutionSuccess

    CATCH EXCEPTION AS lookupFailure
        LOG.error("Fatal Operational Failure: Identity matching execution pipeline broken. Trace: " + lookupFailure.message)
        SET matchingExecutionSuccess = false
        RETURN matchingExecutionSuccess
    END TRY
END SERVICE METHOD

```
