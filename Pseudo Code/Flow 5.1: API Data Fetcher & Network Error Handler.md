 **Flow 5.1: API Data Fetcher & Network Error Handler**. 
---

### 📊 Step A: Technical Flowchart — Cost & Error-Aware GraphQL Fetcher

```text
                  [ START: Trigger Ingestion Pipeline Execution Loop ]
                                            │
                                            ▼
                             [ Initialize API Client Config ]
                             - Access Token & Endpoint Target Base URL
                             - Set Cursor Variable = NULL
                                            │
                                            ▼
                    ┌────────────────────────────────────────────────┐
                    │ 🔄 TOP OF PAGINATION LOOP                      │◄──────────────────────┐
                    └───────────────────────┬────────────────────────┘                       │
                                            │                                                │
                                            ▼                                                │
                             [ 1. ASSEMBLE GRAPHQL REQUEST Payload ]                         │
                             - Load Finalized Complete Master GraphQL Query String           │
                             - Attach Variables: { first: 50, cursor: CurrentCursor }        │
                                            │                                                │
                                            ▼                                                │
                       ⚡ [ 2. EXECUTE NETWORK POST OPERATION ]                              │
                       - Header: Content-Type = application/json                              │
                                            │                                                │
                     / \────────────────────┴────────────────────/ \                         │
                    /   \                                       /   \                        │
             HTTP Status == 200?                          Any Raw Network Drop?              │
                    \   /                                       \   /                        │
                     \ /                                         \ /                         │
                      │                                           │                          │
            ⚡ YES ───┴─── NO ──┐                         NO ─────┴───── YES ──┐             │
              │                 ▼                                              ▼             │
              │        / \──────┴──────/ \                           ┌──────────────────┐    │
              │       /   \           /   \                          │ 📈 INCREMENT     │    │
              │   HTTP Code 429?  Other 5xx/4xx                      │ Retry Counter    │    │
              │       \   /           \   /                          └────────┬─────────┘    │
              │        \ /             \ /                                    │              │
              │         │ YES           │ Other                               ▼              │
              │         │               ▼                            / \──────┴──────/ \     │
              │         │       [ HALT PIPELINE RUN ]               /   \           /   \    │
              │         │       [ Throw System Error]          Retry <= Max?     Exceeded    │
              │         ▼                                           \   /           \   /    │
              │   [ Backoff Sleep: ]                                 \ /             \ /     │
              │   [ Fixed 5 Seconds]                                  │ Max OK        │      │
              │         │                                             ▼               ▼      │
              │         └───────────────┼────────────────────► [ Sleep: 2^Retry ] [ FATAL ]  │
              │                         │                             └───────┬───────┘      │
              │                         ▼                                     └──────────────┘
              │               ┌──────────────────┐
              │               │ Clear Retry Count│
              │               └──────────────────┘
              ▼
    [ 3. DECODE JSON RESPONSE BODY ]
              │
              ▼
     / \──────┴────────────────────────────/ \
    /   \                                 /   \
Payload Contains "errors" Node?           Clean Body
    \   /                                 \   /
     \ /                                   \ /
      │ YES                                 │ NO
      ▼                                     ▼
[ HALT PIPELINE RUN ]             [ 4. EXTRACT METADATA & DATA NODES ]
[ Throw GraphQL Syntax/Auth Error] - data.customers, extensions.cost
                                            │
                                            ▼
                                  [ 5. EVALUATE RUNTIME QUERY COST ]
                                  - RequestedCost     = cost.requestedQueryCost
                                  - AvailablePoints   = cost.throttleStatus.currentlyAvailable
                                  - RestoreRatePerSec = cost.throttleStatus.restoreRate
                                            │
                                            ▼
                                  [ 6. PASS PAYLOAD TO PIPELINE STAGE 5.2 ]
                                  - Push raw edges down to flattener queue
                                            │
                                            ▼
                                   / \──────┴────────────────────────────/ \
                                  /   \                                 /   \
                              Has Next Page? (pageInfo.hasNextPage)  Last Page Reached
                                  \   /                                 \   /
                                   \ /                                   \ /
                                    │ YES                                 │ NO
                                    │                                     ▼
                                    ├────────────────────────────► [ SUCCESS STOP ]
                                    │                              [ Pipeline Idle Run Finished ]
                                    ▼
              [ 7. COMPUTE SAFETY BACKOFF PROFILE ]
              - Shortfall = RequestedCost - AvailablePoints
              - If Shortfall > 0:
                  - WaitSeconds = Ceil(Shortfall / RestoreRatePerSec)
                  - EXECUTE SYSTEM SLEEP(WaitSeconds)
              - Update Cursor Variable = data.customers.pageInfo.endCursor
                    │
                    └────────────────────────────────────────────────────────────────────────┘

```

---

### 📝 Step B: Language-Agnostic Pseudo-code — Network Error Boundary

```text
SERVICE METHOD: shopify.fetchCustomerGraphQLBatch
    INPUT PARAMETERS:
        string shopifyBaseUrl
        string adminAccessToken
        integer batchSize = 50
    
    OUTPUT PARAMETERS:
        boolean pipelineExecutionStatus

    // Initialize state metrics
    string currentCursor = null
    boolean hasNextPage = true
    integer maxNetworkRetries = 5
    
    // Hardcoded production-locked finalized customer query string
    string graphqlQueryString = "query GetShopifyCustomersPaginationComplete($first: Int!, $cursor: String) {
      customers(first: $first, after: $cursor) {
        pageInfo { hasNextPage endCursor }
        edges {
          node {
            id legacyResourceId createdAt updatedAt firstName lastName email verifiedEmail phone note state taxExempt multipassIdentifier tags
            defaultCurrency { isoCode }
            numberOfOrders amountSpent { amount }
            lastOrder { id name }
            acceptsMarketing acceptsMarketingUpdatedAt marketingOptInLevel
            emailMarketingConsent { marketingState optInLevel consentUpdatedAt }
            smsMarketingConsent { marketingState optInLevel consentUpdatedAt consentCollectedFrom }
            addresses { address1 address2 city provinceCode countryCodeV2 zip phone firstName lastName company default }
          }
        }
      }
    }"

    WHILE hasNextPage == true DO
        integer retryAttempt = 0
        boolean requestSuccessful = false
        MAP responsePayload = null

        // Inner Retry Loop for handling flaky connections and structural network drops
        WHILE requestSuccessful == false DO
            TRY
                // Assemble the standard GraphQL JSON context headers
                MAP requestHeaders = {
                    "X-Shopify-Access-Token": adminAccessToken,
                    "Content-Type": "application/json" // Standardized JSON payload boundary mapping
                }
                MAP requestVariables = {
                    "first": batchSize,
                    "cursor": currentCursor
                }
                MAP requestBody = {
                    "query": graphqlQueryString,
                    "variables": requestVariables
                }

                // Execute the explicit network HTTP POST request invocation
                HTTP_RESPONSE response = NETWORK.post(shopifyBaseUrl + "/admin/api/2026-04/graphql.json", requestHeaders, requestBody)

                IF response.statusCode == 200 THEN
                    responsePayload = JSON.decode(response.body)
                    
                    // CRITICAL PRODUCTION FIX: Intercept GraphQL errors sent inside an HTTP 200 frame
                    IF responsePayload.contains("errors") THEN
                        LOG.error("Fatal Error: GraphQL execution syntax/auth failure within HTTP 200: " + responsePayload.get("errors"))
                        RETURN false
                    END IF

                    requestSuccessful = true
                    retryAttempt = 0 // Reset error counters upon complete success
                
                ELSE IF response.statusCode == 429 THEN
                    // Handle burst limits outside typical cost calculations
                    retryAttempt = retryAttempt + 1
                    IF retryAttempt > maxNetworkRetries THEN
                        LOG.error("Fatal Error: Burst limit HTTP 429 exceeded maximum safe retry attempts.")
                        RETURN false
                    END IF
                    
                    LOG.warn("Rate limit HTTP 429 hit. Initiating baseline network sleep backoff.")
                    SYSTEM.sleep(5.0) // Halt execution for a flat 5 seconds before retrying
                
                ELSE
                    // Catch unexpected status responses (e.g., 500 Server Error, 401 Unauthorized)
                    LOG.error("Fatal Connection Error: Shopify endpoint returned unhandled status code: " + response.statusCode)
                    RETURN false
                END IF

            CATCH NETWORK_EXCEPTION AS connectionError
                // Handle socket timeouts, packet losses, or connection drops
                retryAttempt = retryAttempt + 1
                IF retryAttempt > maxNetworkRetries THEN
                    LOG.error("Fatal Network Error: Connection unreachable. Out of retry attempts. Trace: " + connectionError.message)
                    RETURN false
                END IF

                // Compute explicit exponential backoff pause duration
                decimal sleepDuration = MATH.power(2, retryAttempt)
                LOG.warn("Network connection dropped. Attempting automated restart. Retry code: " + retryAttempt + ". Sleeping for " + sleepDuration + " seconds.")
                SYSTEM.sleep(sleepDuration)
            END TRY
        END WHILE

        // --- GRAPHQL COST ANALYSIS & THROTTLE MITIGATION STAGE ---
        
        // Extract inner payload elements safely now that errors are ruled out
        MAP dataNode = responsePayload.get("data")
        MAP customersCollection = dataNode.get("customers")
        MAP costExtension = responsePayload.get("extensions").get("cost")

        // Parse runtime constraint numbers from the response metadata
        integer requestedQueryCost = costExtension.get("requestedQueryCost")
        MAP throttleStatus = costExtension.get("throttleStatus")
        integer currentlyAvailable = throttleStatus.get("currentlyAvailable")
        integer restoreRatePerSecond = throttleStatus.get("restoreRate")

        // Route raw customer collection payload directly to the Stage 5.2 transformation processing queue
        LIST customerEdges = customersCollection.get("edges")
        CALL pipeline.normalizeAndFlattenPayloadBatch(customerEdges)

        // Read page pagination state indicators
        MAP pageInfo = customersCollection.get("pageInfo")
        hasNextPage = pageInfo.get("hasNextPage")

        IF hasNextPage == true THEN
            currentCursor = pageInfo.get("endCursor")

            // Proactively evaluate rate-limit capacities for the subsequent batch request
            integer safetyShortfall = requestedQueryCost - currentlyAvailable

            IF safetyShortfall > 0 THEN
                // Calculate the exact time needed for the point bucket to recover
                decimal requiredWaitSeconds = MATH.ceiling(safetyShortfall / restoreRatePerSecond)
                
                LOG.info("GraphQL Throttle Optimization Triggered. Shortfall: " + safetyShortfall + " points. System sleep enforced: " + requiredWaitSeconds + " seconds.")
                SYSTEM.sleep(requiredWaitSeconds)
            END IF
        END IF
    END WHILE

    LOG.info("Shopify Ingestion Pipeline Inbound Feed completed successfully.")
    RETURN true
END SERVICE METHOD

```

---

With Flow 5.1 fully corrected and solidified, we are ready to advance to **Flow 5.3: Idempotency Evaluator & Profile Matcher**, where we'll implement our multi-key validation check (`SHOPIFY_CUST_ID` ➔ `SHOPIFY_GRAPHQL_GID` ➔ `email`). Shall we construct the technical flowchart and pseudo-code component for it?
