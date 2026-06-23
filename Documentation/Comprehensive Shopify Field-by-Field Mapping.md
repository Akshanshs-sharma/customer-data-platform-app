This matrix explicitly includes **every single field** present in the payload you provided. Fields that are redundant or intentionally dropped (such as the duplicate `default_address` node or overlapping names and geolocation labels) are clearly marked with an engineering resolution explanation.

---

## 📋 Comprehensive Shopify Field-by-Field Mapping Matrix

| Shopify JSON Source Path | Target MySQL Table | Target MySQL Column | Engineering Resolution & Mapping Status |
| --- | --- | --- | --- |
| `customer.id` | `CDP_PARTY_IDENTIFICATION` | `ID_VALUE` | **Mapped:** Saved as string text where `PARTY_IDENT_TYPE_ID = 'SHOPIFY_CUST_ID'`. Used to look up existing profiles during an upsert operation. |
| `customer.email` | `CDP_EMAIL_ADDRESS` | `EMAIL_ADDRESS` | **Mapped:** Captured as lowercase, clean unique key string. Relates back via `CDP_PARTY_CONTACT_MECH` using purpose `'PRIMARY_EMAIL'`. |
| `customer.created_at` | `CDP_PARTY` | `CREATED_STAMP` | **Mapped:** Extracted and parsed into standard MySQL `DATETIME` stamp format. |
| `customer.updated_at` | `CDP_PARTY` | `LAST_UPDATED_STAMP` | **Mapped:** Extracted and parsed into standard MySQL `DATETIME` stamp format. |
| `customer.first_name` | `CDP_PERSON` | `FIRST_NAME` | **Mapped:** Direct column copy string value. Trims trailing or leading spaces. |
| `customer.last_name` | `CDP_PERSON` | `LAST_NAME` | **Mapped:** Direct column copy string value. Trims trailing or leading spaces. |
| `customer.orders_count` | `CDP_PARTY_MARKETING_METRIC` | `ORDERS_COUNT` | **Mapped:** Direct integer value assignment to cache dynamic calculations for fast segmentation checks. |
| `customer.state` | `CDP_PARTY_ATTRIBUTE` | `ATTR_VALUE` | **Mapped:** Saved with `ATTR_NAME = 'shopify_account_state'`. Captures portal state (`disabled`, `invited`, `enabled`). |
| `customer.total_spent` | `CDP_PARTY_MARKETING_METRIC` | `TOTAL_SPENT` | **Mapped:** Casts incoming string `"0.00"` directly to optimized MySQL `DECIMAL(24,4)` numeric format. |
| `customer.last_order_id` | `CDP_PARTY_MARKETING_METRIC` | `LAST_ORDER_ID` | **Mapped:** Direct variable capture. Safely logs empty/null options. |
| `customer.note` | `CDP_PARTY_ATTRIBUTE` | `ATTR_VALUE` | **Mapped:** Captured securely as historical context with `ATTR_NAME = 'shopify_customer_note'`. |
| `customer.verified_email` | `CDP_EMAIL_ADDRESS` | `IS_VERIFIED` | **Mapped:** Casts boolean flag (`true` / `false`) directly into `TINYINT` / `BOOLEAN` (`1` / `0`). |
| `customer.multipass_identifier` | `CDP_PARTY_IDENTIFICATION` | `ID_VALUE` | **Mapped:** If present, tracked under `PARTY_IDENT_TYPE_ID = 'SHOPIFY_MULTIPASS_ID'`. Left empty if null. |
| `customer.tax_exempt` | `CDP_PARTY_ATTRIBUTE` | `ATTR_VALUE` | **Mapped:** Casts boolean value to string flag under custom identifier `ATTR_NAME = 'shopify_tax_exempt'`. |
| `customer.tags` | `CDP_PARTY_TAG` | `TAG_VALUE` | **Mapped:** Dynamic array parsing split by commas. Creates distinct normalized records for `'Tag1'` and `'Tag2'`. |
| `customer.last_order_name` | `CDP_PARTY_MARKETING_METRIC` | `LAST_ORDER_NAME` | **Mapped:** Direct column string match. Logs null or empty states cleanly. |
| `customer.currency` | `CDP_PARTY_ATTRIBUTE` | `ATTR_VALUE` | **Mapped:** Captured as storefront configuration reference text: `ATTR_NAME = 'shopify_preferred_currency'`. |
| `customer.phone` | `CDP_TELECOM_NUMBER` | `CONTACT_NUMBER` | **Mapped:** Primary phone contact. Cleans characters out and links via `CDP_PARTY_CONTACT_MECH` as `'PRIMARY_PHONE'`. |
| `customer.accepts_marketing` | `CDP_PARTY_PREFERENCE` | `PREFERENCE_VALUE` | **Mapped:** Global flag. Translates to string entry (`'subscribed'` / `'unsubscribed'`) for `PARTY_PREF_TYPE_ID = 'GLOBAL_MARKETING'`. |
| `customer.accepts_marketing_updated_at` | `CDP_PARTY_PREFERENCE` | `FROM_DATE` | **Mapped:** Tracks timeline window activation point for global marketing permissions. |
| `customer.marketing_opt_in_level` | `CDP_PARTY_ATTRIBUTE` | `ATTR_VALUE` | **Mapped:** Tracked under `ATTR_NAME = 'shopify_marketing_opt_in_level'` for auditing platform logs. |
| `customer.tax_exemptions` | — | — | **Not Needed / Dropped:** Raw array initialization empty row block `[]`. Extensible via attributes if values appear later. |
| `customer.email_marketing_consent.state` | `CDP_PARTY_PREFERENCE` | `PREFERENCE_VALUE` | **Mapped:** Granular channel state tracked explicitly under specialized code `PARTY_PREF_TYPE_ID = 'EMAIL_MARKETING'`. |
| `customer.email_marketing_consent.opt_in_level` | `CDP_PARTY_ATTRIBUTE` | `ATTR_VALUE` | **Mapped:** Saved under key `ATTR_NAME = 'email_marketing_opt_in_level'`. |
| `customer.email_marketing_consent.consent_updated_at` | `CDP_PARTY_PREFERENCE` | `FROM_DATE` | **Mapped:** Controls temporal window tracking for targeted email marketing campaigns. |
| `customer.sms_marketing_consent.state` | `CDP_PARTY_PREFERENCE` | `PREFERENCE_VALUE` | **Mapped:** Granular channel state tracked explicitly under specialized code `PARTY_PREF_TYPE_ID = 'SMS_MARKETING'`. |
| `customer.sms_marketing_consent.opt_in_level` | `CDP_PARTY_ATTRIBUTE` | `ATTR_VALUE` | **Mapped:** Saved under key `ATTR_NAME = 'sms_marketing_opt_in_level'`. |
| `customer.sms_marketing_consent.consent_updated_at` | `CDP_PARTY_PREFERENCE` | `FROM_DATE` | **Mapped:** Controls temporal window tracking for direct cell phone text updates. |
| `customer.sms_marketing_consent.consent_collected_from` | `CDP_PARTY_ATTRIBUTE` | `ATTR_VALUE` | **Mapped:** Tracks data origin. Captured via key parameter `ATTR_NAME = 'sms_consent_source'`. |
| `customer.admin_graphql_api_id` | `CDP_PARTY_IDENTIFICATION` | `ID_VALUE` | **Mapped:** Tracked under system cross-reference index `PARTY_IDENT_TYPE_ID = 'SHOPIFY_GRAPHQL_GID'`. |
| **`customer.default_address`** | — | — | **Not Needed / Dropped entirely:** Structural duplication of the address array data. Skipped to prevent duplicate generation. |

---

### 📦 Nested Array Processing: `customer.addresses[]`

Every single object located inside the active `addresses[]` collection maps out to distinct records using these field transformations:

| Shopify Address Sub-Field | Target MySQL Table | Target MySQL Column | Engineering Resolution & Mapping Status |
| --- | --- | --- | --- |
| `address.id` | — | — | **Not Needed:** Real internal keys are auto-assigned within our local platform database engine. |
| `address.customer_id` | — | — | **Not Needed / Handled:** Linked internally by tracing the current execution block context to `CDP_PARTY.PARTY_ID`. |
| `address.first_name` | `CDP_POSTAL_ADDRESS` | `TO_NAME` | **Resolved via concatenation:** Combined into a clean human routing display string with `last_name`. |
| `address.last_name` | `CDP_POSTAL_ADDRESS` | `TO_NAME` | **Resolved via concatenation:** Combined into a clean human routing display string with `first_name`. |
| `address.company` | `CDP_POSTAL_ADDRESS` | `TO_NAME` | **Resolved / Appended:** Merged directly into attention line context formatting rules if present. |
| `address.address1` | `CDP_POSTAL_ADDRESS` | `ADDRESS1` | **Mapped:** Direct column copy text parameter. |
| `address.address2` | `CDP_POSTAL_ADDRESS` | `ADDRESS2` | **Mapped:** Direct column copy text parameter. Safe with empty elements. |
| `address.city` | `CDP_POSTAL_ADDRESS` | `CITY` | **Mapped:** Direct column copy string value. |
| `address.province` | — | — | **Not Needed / Resolved:** Dropped in favor of using standard 2-letter geolocation identifier flags (`province_code`). |
| `address.country` | — | — | **Not Needed / Resolved:** Dropped in favor of using standard alpha-2 country code strings (`country_code`). |
| `address.zip` | `CDP_POSTAL_ADDRESS` | `POSTAL_CODE` | **Mapped:** Direct string capture mapping. |
| `address.phone` | `CDP_TELECOM_NUMBER` | `CONTACT_NUMBER` | **Secondary Telecom Invariant Mapped:** Spawns an isolated numeric contact mechanics entry tied under purpose type `'SHIPPING_PHONE'`. |
| `address.name` | — | — | **Not Needed:** Redundant display parameter. Resolved cleanly using our custom profile names or attention lines. |
| `address.province_code` | `CDP_POSTAL_ADDRESS` | `STATE_PROVINCE_GEO_ID` | **Mapped:** Standard ISO short-code state abbreviation string (e.g., `'NY'`). |
| `address.country_code` | `CDP_POSTAL_ADDRESS` | `COUNTRY_GEO_ID` | **Mapped:** Standard alpha-2 global standard geolocation tracking country code (e.g., `'US'`). |
| `address.country_name` | — | — | **Not Needed:** Redundant descriptor code dropped in favor of using clear `country_code` indexes. |
| `address.default` | `CDP_PARTY_CONTACT_MECH` | `CONTACT_MECH_PURPOSE_TYPE_ID` | **Conditional Invariant Mapped:** If boolean matches `true`, sets routing assignment to `'DEFAULT_LOCATION'`. If `false`, registers row as `'SHIPPING_LOCATION'`. |

---
