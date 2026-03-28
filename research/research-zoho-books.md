# Zoho Books API — Comprehensive Integration Reference
### For Shopify Accounting Platform (India & Singapore Focus)
**Research Date:** March 28, 2026  
**Source:** [Zoho Books API Documentation v3](https://www.zoho.com/books/api/v3/)

---

## Table of Contents

1. [Authentication & OAuth 2.0](#1-authentication--oauth-20)
2. [Data Centers & Base URLs](#2-data-centers--base-urls)
3. [Organizations & Multi-Org](#3-organizations--multi-org)
4. [Rate Limits](#4-rate-limits)
5. [Pagination, Filtering & Sorting](#5-pagination-filtering--sorting)
6. [Error Codes & HTTP Status Codes](#6-error-codes--http-status-codes)
7. [Core API Endpoints](#7-core-api-endpoints)
   - 7.1 [Contacts](#71-contacts)
   - 7.2 [Invoices](#72-invoices)
   - 7.3 [Bills & Expenses](#73-bills--expenses)
   - 7.4 [Chart of Accounts](#74-chart-of-accounts)
   - 7.5 [Journal Entries](#75-journal-entries)
   - 7.6 [Bank Accounts](#76-bank-accounts)
   - 7.7 [Bank Transactions & Reconciliation](#77-bank-transactions--reconciliation)
   - 7.8 [Customer Payments](#78-customer-payments)
   - 7.9 [Vendor Payments](#79-vendor-payments)
   - 7.10 [Credit Notes](#710-credit-notes)
   - 7.11 [Vendor Credits (Debit Notes)](#711-vendor-credits-debit-notes)
   - 7.12 [Recurring Invoices](#712-recurring-invoices)
8. [Taxes & GST Settings API](#8-taxes--gst-settings-api)
9. [India GST Features](#9-india-gst-features)
10. [Singapore GST Features](#10-singapore-gst-features)
11. [Webhooks](#11-webhooks)
12. [Multi-Currency](#12-multi-currency)
13. [Reports API](#13-reports-api)

---

## 1. Authentication & OAuth 2.0

**Source:** [Zoho Books OAuth Documentation](https://www.zoho.com/books/api/v3/oauth/)

Zoho Books uses OAuth 2.0 exclusively. Every API call requires a valid access token.

### 1.1 App Registration

Register your application at the **Zoho API Developer Console**:
- **India DC:** https://api-console.zoho.in
- **Global DC:** https://api-console.zoho.com

**Two registration types:**

| Type | Use Case | Redirect URI Required |
|------|----------|-----------------------|
| **Server-based Application** | Production integrations, web apps | Yes — full authorization code flow |
| **Self Client** | Testing, scripts, internal tools | No — generate grant token directly in Developer Console |

### 1.2 Authorization Code Flow (Server-Based)

**Step 1 — Get Authorization Code:**
```
GET {Accounts_URL}/oauth/v2/auth
  ?response_type=code
  &client_id={client_id}
  &scope={scopes}
  &redirect_uri={redirect_uri}
  &access_type=offline
  &state={random_state}
```

**Step 2 — Exchange Code for Tokens:**
```
POST {Accounts_URL}/oauth/v2/token
  ?code={authorization_code}
  &client_id={client_id}
  &client_secret={client_secret}
  &redirect_uri={redirect_uri}
  &grant_type=authorization_code
```

**Step 3 — Refresh Access Token:**
```
POST {Accounts_URL}/oauth/v2/token
  ?refresh_token={refresh_token}
  &client_id={client_id}
  &client_secret={client_secret}
  &grant_type=refresh_token
```

**Response:**
```json
{
  "access_token": "1000.xxxxx",
  "refresh_token": "1000.xxxxx",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### 1.3 Token Endpoints by Data Center

| Data Center | Accounts URL | Token Endpoint |
|-------------|-------------|----------------|
| **India** | `https://accounts.zoho.in` | `https://accounts.zoho.in/oauth/v2/token` |
| **US/Singapore** | `https://accounts.zoho.com` | `https://accounts.zoho.com/oauth/v2/token` |
| Europe | `https://accounts.zoho.eu` | `https://accounts.zoho.eu/oauth/v2/token` |
| Australia | `https://accounts.zoho.com.au` | `https://accounts.zoho.com.au/oauth/v2/token` |

> **Critical:** India organizations MUST use `accounts.zoho.in`. Tokens from `zoho.com` will NOT work for India org API calls.

### 1.4 Using the Access Token

Pass the token in every API request header:
```
Authorization: Zoho-oauthtoken {access_token}
```

### 1.5 Token Validity

| Token | Validity |
|-------|----------|
| Access Token | 1 hour (3600 seconds) |
| Refresh Token | Permanent until revoked; max 20 per user |
| Grant Token (web app) | 2 minutes |
| Grant Token (Self Client) | Configurable (1 hour default) |

### 1.6 OAuth Scopes — Complete List

| Module | Scope Pattern |
|--------|--------------|
| Contacts | `ZohoBooks.contacts.CREATE/UPDATE/READ/DELETE/ALL` |
| Settings (Taxes, Currencies, Webhooks, Orgs) | `ZohoBooks.settings.CREATE/UPDATE/READ/DELETE/ALL` |
| Estimates | `ZohoBooks.estimates.CREATE/UPDATE/READ/DELETE/ALL` |
| Invoices | `ZohoBooks.invoices.CREATE/UPDATE/READ/DELETE/ALL` |
| Customer Payments | `ZohoBooks.customerpayments.CREATE/UPDATE/READ/DELETE/ALL` |
| Credit Notes | `ZohoBooks.creditnotes.CREATE/UPDATE/READ/DELETE/ALL` |
| Projects | `ZohoBooks.projects.CREATE/UPDATE/READ/DELETE/ALL` |
| Expenses | `ZohoBooks.expenses.CREATE/UPDATE/READ/DELETE/ALL` |
| Sales Orders | `ZohoBooks.salesorders.CREATE/UPDATE/READ/DELETE/ALL` |
| Purchase Orders | `ZohoBooks.purchaseorders.CREATE/UPDATE/READ/DELETE/ALL` |
| Bills | `ZohoBooks.bills.CREATE/UPDATE/READ/DELETE/ALL` |
| Vendor Credits (Debit Notes) | `ZohoBooks.debitnotes.CREATE/UPDATE/READ/DELETE/ALL` |
| Vendor Payments | `ZohoBooks.vendorpayments.CREATE/UPDATE/READ/DELETE/ALL` |
| Banking | `ZohoBooks.banking.CREATE/UPDATE/READ/DELETE/ALL` |
| Journals/Accountants | `ZohoBooks.accountants.CREATE/UPDATE/READ/DELETE/ALL` |

**Multiple scopes** are separated by comma or space:
```
scope=ZohoBooks.invoices.ALL,ZohoBooks.contacts.ALL,ZohoBooks.settings.READ
```

---

## 2. Data Centers & Base URLs

**Source:** [Zoho Books API Introduction](https://www.zoho.com/books/api/v3/introduction/)

| Data Center | Domain | Base API URI | Accounts URL |
|-------------|--------|-------------|--------------|
| **India** | `.in` | `https://www.zohoapis.in/books/` | `https://accounts.zoho.in` |
| US / **Singapore** | `.com` | `https://www.zohoapis.com/books/` | `https://accounts.zoho.com` |
| Europe | `.eu` | `https://www.zohoapis.eu/books/` | `https://accounts.zoho.eu` |
| Australia | `.com.au` | `https://www.zohoapis.com.au/books/` | `https://accounts.zoho.com.au` |
| Japan | `.jp` | `https://www.zohoapis.jp/books/` | `https://accounts.zoho.jp` |
| Canada | `.ca` | `https://www.zohoapis.ca/books/` | `https://accounts.zoho.ca` |
| China | `.com.cn` | `https://www.zohoapis.com.cn/books/` | `https://accounts.zoho.com.cn` |
| Saudi Arabia | `.sa` | `https://www.zohoapis.sa/books/` | `https://accounts.zoho.sa` |

### Critical Notes for India & Singapore

- **India** organizations are registered on `zoho.in` and use `https://www.zohoapis.in/books/v3/`
- **Singapore** has NO dedicated Zoho data center. Singapore organizations use the `.com` (US) domain: `https://www.zohoapis.com/books/v3/`
- To determine the correct domain, check the URL of the organization's Zoho Books web app (e.g., `books.zoho.in` → use `.in` endpoints)

### Full Endpoint Format

```
{Base_API_URI}v3/{endpoint}?organization_id={org_id}
```

**India example:**
```
https://www.zohoapis.in/books/v3/invoices?organization_id=10234695
```

**Singapore example:**
```
https://www.zohoapis.com/books/v3/invoices?organization_id=10234695
```

---

## 3. Organizations & Multi-Org

**Source:** [Zoho Books Organizations API](https://www.zoho.com/books/api/v3/organizations/)

### 3.1 API Endpoints

| Method | Path | Description | Scope |
|--------|------|-------------|-------|
| `POST` | `/organizations` | Create an organization | `ZohoBooks.settings.CREATE` |
| `GET` | `/organizations` | List all organizations | `ZohoBooks.settings.READ` |
| `PUT` | `/organizations/{organization_id}` | Update an organization | `ZohoBooks.settings.UPDATE` |
| `GET` | `/organizations/{organization_id}` | Get a specific organization | `ZohoBooks.settings.READ` |

> **Note:** `organization_id` is NOT required as a query parameter for these calls (it IS the resource being fetched). It IS required for all other API calls.

### 3.2 Key Organization Fields

| Field | Type | Description |
|-------|------|-------------|
| `organization_id` | string | Unique numeric ID — required in every other API call |
| `name` | string | Organization name |
| `is_default_org` | boolean | Whether this is the default org |
| `plan_name` | string | `FREE`, `STANDARD`, `PROFESSIONAL`, `PREMIUM`, `ELITE`, `ULTIMATE` |
| `plan_type` | number | Numeric plan code |
| `currency_code` | string | Base currency (e.g., `INR` for India, `SGD` for Singapore) |
| `currency_id` | string | Base currency ID |
| `time_zone` | string | Org timezone |
| `fiscal_year_start_month` | string/number | Start of financial year |
| `language_code` | string | UI language |
| `date_format` | string | Date format |
| `tax_group_enabled` | boolean | Whether tax groups are enabled |
| `is_org_active` | boolean | Whether the org is active |

### 3.3 List Organizations Response

```json
{
  "code": 0,
  "message": "success",
  "organizations": [
    {
      "organization_id": "10229182",
      "name": "MyMediset India Pvt Ltd",
      "is_default_org": false,
      "plan_name": "PROFESSIONAL",
      "currency_code": "INR",
      "currency_symbol": "₹",
      "time_zone": "Asia/Calcutta",
      "fiscal_year_start_month": 3,
      "is_org_active": true
    },
    {
      "organization_id": "10229183",
      "name": "MyMediset Singapore Pte Ltd",
      "plan_name": "PROFESSIONAL",
      "currency_code": "SGD",
      "time_zone": "Asia/Singapore",
      "is_org_active": true
    }
  ]
}
```

### 3.4 Multi-Org Pattern for Shopify Integration

The `organization_id` query parameter switches context between organizations in all API calls:

```
# India org
GET https://www.zohoapis.in/books/v3/invoices?organization_id=10229182

# Singapore org  
GET https://www.zohoapis.com/books/v3/invoices?organization_id=10229183
```

**Implementation pattern for multi-tenant Shopify:**
1. On first connect, call `GET /organizations` to retrieve all org IDs
2. Store `organization_id` + DC domain per org/tenant
3. Include the correct `organization_id` in every subsequent API call
4. The same OAuth access token can access multiple organizations if the authenticated user has access to all of them

---

## 4. Rate Limits

**Source:** [Zoho Books API Introduction](https://www.zoho.com/books/api/v3/introduction/)

### 4.1 Per-Minute Limits

| Limit Type | Limit | HTTP Code on Exceed | Error Code |
|-----------|-------|--------------------|-----------:|
| Per organization | 100 req/min | 429 | 44 |
| Per account | 100 req/min | 429 | 44 |
| Concurrent (Free plan) | 5 in-flight | 429 | 1070 |
| Concurrent (Paid plans) | 10 in-flight (soft) | 429 | 1070 |

### 4.2 Daily Limits by Plan

| Plan | Daily API Calls |
|------|----------------|
| Free | 1,000 |
| Standard | 2,000 |
| Professional | 5,000 |
| Premium | 10,000 |
| Elite | 10,000 |
| Ultimate | 10,000 |

### 4.3 Rate Limit Error Responses

```json
// Daily limit exceeded
{
  "code": 45,
  "message": "The API call for this organization has exceeded the maximum call rate limit of 1000."
}

// Per-minute limit exceeded
{
  "code": 44,
  "message": "For security reasons your organization has been blocked as it have exceeded the maximum number of requests per minute that can originate from an organization."
}

// Concurrent limit exceeded
{
  "code": 1070,
  "message": "You have reached the maximum number of in process requests allowed. Kindly try again after some time."
}
```

### 4.4 Rate Limit Handling Strategy

- Implement exponential backoff on HTTP 429 responses
- Use `Retry-After` header if present
- Consider request queuing with a token bucket algorithm (100 req/min = ~1.67 req/sec)
- For bulk Shopify sync operations, batch API calls and add delays between batches

---

## 5. Pagination, Filtering & Sorting

**Source:** [Zoho Books Pagination](https://www.zoho.com/books/api/v3/pagination/)

### 5.1 Pagination Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | 1 | Page number to retrieve |
| `per_page` | integer | 200 | Records per page (max: 200) |

### 5.2 page_context Response Object

Every list endpoint returns a `page_context` node:

```json
{
  "code": 0,
  "message": "success",
  "invoices": [...],
  "page_context": {
    "page": 2,
    "per_page": 25,
    "has_more_page": true,
    "report_name": "Invoices",
    "applied_filter": "Status.All",
    "sort_column": "created_time",
    "sort_order": "D"
  }
}
```

| Field | Description |
|-------|-------------|
| `page` | Current page |
| `per_page` | Records returned |
| `has_more_page` | `true` if additional pages exist — use to implement pagination loop |
| `report_name` | Resource name |
| `applied_filter` | Active filter |
| `sort_column` | Current sort column |
| `sort_order` | `A` = ascending, `D` = descending |

### 5.3 Common Filtering Parameters

Most list endpoints support these filter/search parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `filter_by` | string | Predefined filter (e.g., `Status.All`, `Status.Active`, `PaymentMode.Cash`) |
| `search_text` | string | Full-text search across key fields |
| `last_modified_time` | string | Filter by last modification timestamp (ISO 8601) |
| `date` | string | Filter by exact date (yyyy-mm-dd) |
| `date_start` | string | Date range start |
| `date_end` | string | Date range end |
| `sort_column` | string | Column to sort by |
| `sort_order` | string | `A` (ascending) or `D` (descending) |

### 5.4 String Filter Variants

For string fields, three filter variants are supported:
- `{field}` — exact match
- `{field}_startswith` — prefix match
- `{field}_contains` — substring match

**Example:** `customer_name_contains=Shopify`

### 5.5 Numeric Filter Variants

For numeric fields (amounts, quantities):
- `{field}_less_than`
- `{field}_less_equals`
- `{field}_greater_than`
- `{field}_greater_equals`

**Example:** `amount_greater_than=1000`

### 5.6 Pagination Loop Pattern

```python
page = 1
while True:
    response = get_invoices(page=page, per_page=200)
    process(response["invoices"])
    if not response["page_context"]["has_more_page"]:
        break
    page += 1
```

---

## 6. Error Codes & HTTP Status Codes

**Source:** [Zoho Books API Errors](https://www.zoho.com/books/api/v3/errors/)

### 6.1 HTTP Status Codes

| HTTP Code | Meaning | Action Required |
|-----------|---------|-----------------|
| `200` | OK — Success | Process response |
| `201` | Created | Record created successfully |
| `400` | Bad Request — Invalid input | Fix request body/parameters |
| `401` | Unauthorized — Invalid/expired token | Refresh access token |
| `404` | URL Not Found | Check endpoint path; verify org has this resource |
| `405` | Method Not Allowed | Check HTTP method (GET vs POST etc.) |
| `429` | Rate Limit Exceeded | Implement backoff; see Rate Limits section |
| `500` | Internal Server Error | Retry after delay; contact Zoho if persistent |

### 6.2 Zoho Application Error Codes (in response body)

| Code | Description |
|------|-------------|
| `0` | Success |
| `44` | Rate limit exceeded (per minute) |
| `45` | Daily API limit exceeded |
| `1000` | Internal error |
| `1002` | Resource does not exist (e.g., "Invoice does not exist") |
| `1070` | Concurrent rate limit exceeded |

### 6.3 Error Response Format

All errors follow this structure:
```json
{
  "code": 1002,
  "message": "Invoice does not exist."
}
```

---

## 7. Core API Endpoints

**Base path for all endpoints below:**
- India: `https://www.zohoapis.in/books/v3/`
- Singapore: `https://www.zohoapis.com/books/v3/`

All endpoints require `?organization_id={org_id}` query parameter.

---

### 7.1 Contacts

**Source:** [Zoho Books Contacts API](https://www.zoho.com/books/api/v3/contacts/)  
**OAuth Scope:** `ZohoBooks.contacts.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/contacts` | Create a contact |
| `GET` | `/contacts` | List contacts |
| `PUT` | `/contacts/{contact_id}` | Update a contact |
| `GET` | `/contacts/{contact_id}` | Get a contact |
| `DELETE` | `/contacts/{contact_id}` | Delete a contact |
| `POST` | `/contacts/{contact_id}/active` | Mark contact as active |
| `POST` | `/contacts/{contact_id}/inactive` | Mark contact as inactive |
| `GET` | `/contacts/{contact_id}/comments` | List comments |
| `POST` | `/contacts/{contact_id}/email` | Email a contact |

#### Key Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contact_name` | string | Yes | Full name |
| `contact_type` | string | Yes | `customer` or `vendor` |
| `company_name` | string | No | Company name |
| `email` | string | No | Primary email |
| `currency_id` | string | No | Preferred currency |
| `payment_terms` | integer | No | Net days (e.g., `30`) |
| `billing_address` | object | No | Billing address |
| `shipping_address` | object | No | Shipping address |

#### India-Specific Fields

| Field | Type | Description |
|-------|------|-------------|
| `gst_no` | string | 15-digit GSTIN (e.g., `22AAAAA0000A1Z5`) |
| `gst_treatment` | string | `business_gst` / `business_none` / `overseas` / `consumer` |
| `place_of_contact` | string | State code for place of contact (e.g., `TN` for Tamil Nadu) |
| `pan_no` | string | PAN number for TDS purposes |

#### gst_treatment Values for India

| Value | Meaning |
|-------|---------|
| `business_gst` | GST-registered business in India |
| `business_none` | Business not registered for GST |
| `overseas` | Customer/vendor outside India |
| `consumer` | End consumer (B2C) |

#### Create Contact Request (India GST-compliant)

```json
{
  "contact_name": "ABC Distributors",
  "contact_type": "customer",
  "company_name": "ABC Distributors Pvt Ltd",
  "email": "accounts@abcdist.com",
  "gst_no": "22AAAAA0000A1Z5",
  "gst_treatment": "business_gst",
  "place_of_contact": "MH",
  "currency_id": "982000000000190",
  "billing_address": {
    "address": "123 Business Park",
    "city": "Mumbai",
    "state": "Maharashtra",
    "zip": "400001",
    "country": "India"
  }
}
```

#### List Contacts — Query Filters

| Parameter | Description |
|-----------|-------------|
| `contact_name` / `contact_name_contains` | Filter by name |
| `contact_type` | `customer` or `vendor` |
| `filter_by` | `ContactType.Customer`, `ContactType.Vendor`, `Status.Active`, `Status.Inactive` |
| `last_modified_time` | ISO 8601 timestamp |
| `search_text` | Full-text search |

---

### 7.2 Invoices

**Source:** [Zoho Books Invoices API](https://www.zoho.com/books/api/v3/invoices/)  
**OAuth Scope:** `ZohoBooks.invoices.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/invoices` | Create an invoice |
| `GET` | `/invoices` | List invoices |
| `PUT` | `/invoices/{invoice_id}` | Update an invoice |
| `GET` | `/invoices/{invoice_id}` | Get an invoice |
| `DELETE` | `/invoices/{invoice_id}` | Delete an invoice |
| `POST` | `/invoices/{invoice_id}/status/sent` | Mark as sent |
| `POST` | `/invoices/{invoice_id}/status/void` | Void an invoice |
| `POST` | `/invoices/{invoice_id}/status/draft` | Convert to draft |
| `POST` | `/invoices/{invoice_id}/email` | Email the invoice |
| `GET` | `/invoices/{invoice_id}/payments` | List payments for invoice |
| `GET` | `/invoices/{invoice_id}/creditnotes` | List credit notes applied |

#### Invoice Status Values

`draft` → `sent` → `viewed` → `partially_paid` → `paid`  
`overdue` (auto, when past due date), `void`

#### Core Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `customer_id` | string | Yes | Contact ID |
| `invoice_number` | string | No | Auto-generated if blank |
| `date` | string | Yes | Invoice date (yyyy-mm-dd) |
| `due_date` | string | No | Payment due date |
| `currency_id` | string | No | Invoice currency |
| `exchange_rate` | double | No | Rate vs base currency |
| `line_items` | array | Yes | Line items |
| `discount` | string | No | e.g., `10%` or `50` (flat) |
| `is_discount_before_tax` | boolean | No | Apply discount before tax |
| `is_inclusive_tax` | boolean | No | Tax-inclusive pricing |
| `payment_terms` | integer | No | Net days |
| `template_id` | string | No | PDF template |
| `notes` | string | No | Customer-facing notes |
| `terms` | string | No | Terms and conditions |

#### Line Item Fields

| Field | Type | Description |
|-------|------|-------------|
| `item_id` | string | Item catalog ID (optional) |
| `name` | string | Item name |
| `description` | string | Line description |
| `quantity` | double | Quantity |
| `rate` | double | Unit price |
| `unit` | string | Unit of measure |
| `tax_id` | string | Tax ID (from taxes settings) |
| `account_id` | string | Revenue account |
| `item_total` | double | Calculated line total |
| `discount` | string | Line-level discount |

#### India-Specific Invoice Fields

| Field | Type | Description |
|-------|------|-------------|
| `place_of_supply` | string | 2-letter state code (e.g., `TN`, `MH`, `KA`) — determines CGST+SGST vs IGST |
| `gst_no` | string | Customer's GSTIN |
| `gst_treatment` | string | `business_gst` / `business_none` / `overseas` / `consumer` |
| `is_pre_gst` | boolean | Pre-GST transaction flag |
| `is_reverse_charge_applied` | boolean | Reverse charge applicable |
| `tcs_tax_id` | string | TCS tax ID (if applicable) |

#### India-Specific Line Item Fields

| Field | Type | Description |
|-------|------|-------------|
| `hsn_or_sac` | string | HSN code (goods) or SAC code (services) |
| `product_type` | string | `goods` or `services` |
| `tax_id` | string | Tax ID (IGST, CGST+SGST group, etc.) |
| `reverse_charge_tax_id` | string | Reverse charge tax ID |
| `tds_tax_id` | string | TDS tax applied to this line |

#### Create Invoice Request (India GST)

```json
{
  "customer_id": "460000000000111",
  "invoice_number": "INV-00001",
  "date": "2026-03-28",
  "due_date": "2026-04-27",
  "place_of_supply": "MH",
  "gst_treatment": "business_gst",
  "gst_no": "27AAAAA0000A1Z5",
  "is_inclusive_tax": false,
  "line_items": [
    {
      "item_id": "460000000053015",
      "name": "Surgical Gloves - Medium",
      "hsn_or_sac": "40151910",
      "product_type": "goods",
      "quantity": 100,
      "rate": 50.00,
      "unit": "Box",
      "tax_id": "460000000057089"
    }
  ],
  "notes": "Payment due within 30 days.",
  "terms": "Goods once sold are not returnable."
}
```

#### List Invoices — Key Filters

| Parameter | Description |
|-----------|-------------|
| `status` | `sent`, `draft`, `overdue`, `paid`, `void`, `unpaid`, `partially_paid` |
| `customer_id` | Filter by customer |
| `date_start` / `date_end` | Date range |
| `filter_by` | `Status.All`, `Status.Sent`, `Status.Draft`, `Status.OverDue`, `Status.Paid`, `Status.Void` |
| `search_text` | Search invoice number, customer name |
| `last_modified_time` | Sync filter — ISO 8601 |
| `sort_column` | `date`, `created_time`, `invoice_number`, `customer_name`, `total`, `balance` |

---

### 7.3 Bills & Expenses

**Source:** [Zoho Books Bills API](https://www.zoho.com/books/api/v3/bills/)  
**OAuth Scope:** `ZohoBooks.bills.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/bills` | Create a bill |
| `GET` | `/bills` | List bills |
| `PUT` | `/bills/{bill_id}` | Update a bill |
| `GET` | `/bills/{bill_id}` | Get a bill |
| `DELETE` | `/bills/{bill_id}` | Delete a bill |
| `POST` | `/bills/{bill_id}/status/open` | Mark as open |
| `POST` | `/bills/{bill_id}/status/void` | Void a bill |
| `GET` | `/bills/{bill_id}/payments` | List payments |
| `POST` | `/bills/{bill_id}/payments` | Pay a bill |

#### Core Bill Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `vendor_id` | string | Yes | Vendor contact ID |
| `bill_number` | string | No | Auto-generated if blank |
| `date` | string | Yes | Bill date |
| `due_date` | string | No | Payment due date |
| `account_id` | string | No | Expense account |
| `line_items` | array | Yes | Line items |
| `payment_terms` | integer | No | Net days |
| `currency_id` | string | No | Bill currency |
| `exchange_rate` | double | No | Rate vs base currency |

#### India-Specific Bill Fields

| Field | Type | Description |
|-------|------|-------------|
| `source_of_supply` | string | State where goods/services originate (vendor's state) |
| `destination_of_supply` | string | State where goods/services are delivered |
| `gst_no` | string | Vendor's GSTIN |
| `gst_treatment` | string | Vendor's GST treatment |
| `is_reverse_charge_applied` | boolean | Reverse charge mechanism applicable |

#### India-Specific Bill Line Item Fields

| Field | Type | Description |
|-------|------|-------------|
| `hsn_or_sac` | string | HSN/SAC code |
| `product_type` | string | `goods` or `services` |
| `reverse_charge_tax_id` | string | Reverse charge tax for this line |
| `tax_id` | string | GST tax ID |
| `tds_tax_id` | string | TDS tax for this line |

---

### 7.4 Chart of Accounts

**Source:** [Zoho Books Chart of Accounts API](https://www.zoho.com/books/api/v3/chartofaccounts/)  
**OAuth Scope:** `ZohoBooks.accountants.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/chartofaccounts` | Create an account |
| `GET` | `/chartofaccounts` | List accounts |
| `PUT` | `/chartofaccounts/{account_id}` | Update an account |
| `GET` | `/chartofaccounts/{account_id}` | Get an account |
| `DELETE` | `/chartofaccounts/{account_id}` | Delete an account |
| `GET` | `/chartofaccounts/transactions` | Get ledger transactions for an account |
| `POST` | `/chartofaccounts/{account_id}/active` | Activate |
| `POST` | `/chartofaccounts/{account_id}/inactive` | Deactivate |

#### Account Types

| Type | Category |
|------|----------|
| `other_asset` | Asset |
| `other_current_asset` | Asset |
| `cash` | Asset |
| `bank` | Asset |
| `fixed_asset` | Asset |
| `accounts_receivable` | Asset |
| `other_liability` | Liability |
| `other_current_liability` | Liability |
| `accounts_payable` | Liability |
| `credit_card` | Liability |
| `long_term_liability` | Liability |
| `equity` | Equity |
| `income` | Income |
| `other_income` | Income |
| `cost_of_goods_sold` | Expense |
| `expense` | Expense |
| `other_expense` | Expense |

#### Create Account Request

```json
{
  "account_name": "GST Input - IGST",
  "account_type": "other_current_asset",
  "currency_id": "982000000000190",
  "description": "Input IGST credit account"
}
```

#### Get Ledger Transactions

```
GET /chartofaccounts/transactions?organization_id={org_id}&account_id={account_id}&date_start=2026-01-01&date_end=2026-03-31
```

---

### 7.5 Journal Entries

**Source:** [Zoho Books Journals API](https://www.zoho.com/books/api/v3/journals/)  
**OAuth Scope:** `ZohoBooks.accountants.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/journals` | Create a journal |
| `GET` | `/journals` | List journals |
| `PUT` | `/journals/{journal_id}` | Update a journal |
| `GET` | `/journals/{journal_id}` | Get a journal |
| `DELETE` | `/journals/{journal_id}` | Delete a journal |
| `POST` | `/journals/{journal_id}/status/publish` | Publish a draft journal |

#### Journal Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `journal_date` | string | Yes | Date (yyyy-mm-dd) |
| `journal_number` | string | No | Auto-generated |
| `reference_number` | string | No | External reference |
| `notes` | string | No | Journal description |
| `currency_id` | string | No | Currency |
| `exchange_rate` | double | No | Exchange rate |
| `line_items` | array | Yes | Debit/credit entries |
| `status` | string | No | `draft` or `published` |

#### Line Item Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `account_id` | string | Yes | GL account ID |
| `debit_or_credit` | string | Yes | `debit` or `credit` |
| `amount` | double | Yes | Amount |
| `description` | string | No | Line description |
| `tax_id` | string | No | Tax ID |
| `project_id` | string | No | Project association |

#### Create Journal Request

```json
{
  "journal_date": "2026-03-31",
  "reference_number": "JE-2026-001",
  "notes": "GST liability offset - March 2026",
  "line_items": [
    {
      "account_id": "460000000000364",
      "debit_or_credit": "debit",
      "amount": 18000.00,
      "description": "Output GST - IGST"
    },
    {
      "account_id": "460000000000365",
      "debit_or_credit": "credit",
      "amount": 18000.00,
      "description": "GST Payable"
    }
  ]
}
```

#### Journal Status Flow

- Create with `status: "draft"` → publish via `POST /journals/{id}/status/publish`
- Published journals create accounting entries and cannot be edited (only reversed via new journal)

---

### 7.6 Bank Accounts

**Source:** [Zoho Books Bank Accounts API](https://www.zoho.com/books/api/v3/bankaccounts/)  
**OAuth Scope:** `ZohoBooks.banking.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/bankaccounts` | Create a bank account |
| `GET` | `/bankaccounts` | List bank accounts |
| `PUT` | `/bankaccounts/{account_id}` | Update a bank account |
| `GET` | `/bankaccounts/{account_id}` | Get a bank account |
| `DELETE` | `/bankaccounts/{account_id}` | Delete a bank account |
| `POST` | `/bankaccounts/{account_id}/statements` | Import bank statement |
| `GET` | `/bankaccounts/{account_id}/statement/lastimported` | Last imported statement |
| `DELETE` | `/bankaccounts/{account_id}/statement` | Delete imported statement |

#### Import Bank Statement (JSON Feed)

```
POST /bankaccounts/{account_id}/statements
```

```json
{
  "date_format": "dd MMM yyyy",
  "transactions": [
    {
      "date": "28 Mar 2026",
      "debit_or_credit": "credit",
      "amount": 50000.00,
      "payee": "ABC Distributors",
      "description": "Payment received INV-00001",
      "reference_number": "REF-123456"
    }
  ]
}
```

---

### 7.7 Bank Transactions & Reconciliation

**Source:** [Zoho Books Bank Transactions API](https://www.zoho.com/books/api/v3/bank-transactions/)  
**OAuth Scope:** `ZohoBooks.banking.{ACTION}`

#### Core Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/banktransactions` | Create a manual bank transaction |
| `GET` | `/banktransactions` | List transactions |
| `PUT` | `/banktransactions/{bank_transaction_id}` | Update a transaction |
| `GET` | `/banktransactions/{bank_transaction_id}` | Get a transaction |
| `DELETE` | `/banktransactions/{bank_transaction_id}` | Delete a transaction |

#### Reconciliation/Matching Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/banktransactions/uncategorized/{transaction_id}/match` | Match imported transaction to existing record |
| `GET` | `/banktransactions/uncategorized/{transaction_id}/match` | Get matching transaction suggestions |
| `POST` | `/banktransactions/{transaction_id}/unmatch` | Unmatch a matched transaction |
| `POST` | `/banktransactions/uncategorized/{transaction_id}/exclude` | Exclude from reconciliation |
| `POST` | `/banktransactions/uncategorized/{transaction_id}/restore` | Restore excluded transaction |
| `POST` | `/banktransactions/uncategorized/{transaction_id}/categorize` | Categorize an uncategorized transaction |
| `POST` | `/banktransactions/uncategorized/{transaction_id}/categorize/expenses` | Categorize as expense |
| `POST` | `/banktransactions/uncategorized/{transaction_id}/categorize/vendorpayments` | Categorize as vendor payment |
| `POST` | `/banktransactions/uncategorized/{transaction_id}/categorize/customerpayments` | Categorize as customer payment |
| `POST` | `/banktransactions/uncategorized/{transaction_id}/categorize/creditnoterefunds` | Categorize as credit note refund |
| `POST` | `/banktransactions/{transaction_id}/uncategorize` | Remove categorization |

#### Transaction Types (Createable)

`deposit` | `transfer_fund` | `card_payment` | `sales_without_invoices` | `expense_refund` | `owner_contribution` | `interest_income` | `other_income` | `owner_drawings` | `sales_return`

> **Note:** Cannot create via API: Expense, Vendor Payment, Bill Payment, Customer Payment, Credit Note Refund (these are created via their respective endpoints and auto-appear in banking)

#### Key Transaction Fields

| Field | Type | Description |
|-------|------|-------------|
| `from_account_id` | string | Source account |
| `to_account_id` | string | Destination account |
| `transaction_type` | string | Type (see above) |
| `amount` | double | Transaction amount |
| `date` | string | Transaction date |
| `payment_mode` | string | `cash`, `cheque`, `bank_transfer`, etc. |
| `reference_number` | string | Reference number |
| `description` | string | Description |
| `customer_id` | string | Associated customer |
| `vendor_id` | string | Associated vendor |
| `currency_id` | string | Currency |
| `exchange_rate` | integer | Exchange rate |

#### Match Transaction Request

```json
{
  "transactions_to_be_matched": [
    {
      "transaction_id": "460000000048017",
      "transaction_type": "customerpayment"
    }
  ]
}
```

#### List Transactions — Query Filters

| Parameter | Description |
|-----------|-------------|
| `account_id` | Filter by bank account |
| `status` | `uncategorized`, `categorized`, `manually_added`, `matched`, `excluded` |
| `filter_by` | `Status.All`, `Status.Uncategorized`, `Status.Categorized`, etc. |
| `date_start` / `date_end` | Date range |
| `transaction_type` | Specific type |

---

### 7.8 Customer Payments

**Source:** [Zoho Books Customer Payments API](https://www.zoho.com/books/api/v3/customerpayments/)  
**OAuth Scope:** `ZohoBooks.customerpayments.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/customerpayments` | Create a payment |
| `GET` | `/customerpayments` | List payments |
| `PUT` | `/customerpayments/{payment_id}` | Update a payment |
| `GET` | `/customerpayments/{payment_id}` | Get a payment |
| `DELETE` | `/customerpayments/{payment_id}` | Delete a payment |
| `POST` | `/customerpayments/{payment_id}/refunds` | Refund a payment |
| `GET` | `/customerpayments/{payment_id}/refunds` | List refunds |

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `customer_id` | string | Customer contact ID |
| `amount` | double | Payment amount |
| `date` | string | Payment date |
| `payment_mode` | string | See modes below |
| `invoices` | array | Invoices being paid |

#### Payment Modes

`check` | `cash` | `creditcard` | `banktransfer` | `bankremittance` | `autotransaction` | `others`

#### Invoice Application Array

```json
"invoices": [
  {
    "invoice_id": "460000000070001",
    "amount_applied": 5000.00
  }
]
```

#### India-Specific Fields

| Field | Type | Description |
|-------|------|-------------|
| `tax_amount_withheld` | double | TDS amount withheld on this payment |

#### Create Payment Request

```json
{
  "customer_id": "460000000000111",
  "payment_mode": "banktransfer",
  "amount": 59000.00,
  "date": "2026-03-28",
  "paid_through_account_id": "460000000000358",
  "reference_number": "UTR-20260328-001",
  "tax_amount_withheld": 1000.00,
  "invoices": [
    {
      "invoice_id": "460000000070001",
      "amount_applied": 59000.00
    }
  ]
}
```

---

### 7.9 Vendor Payments

**Source:** [Zoho Books Vendor Payments API](https://www.zoho.com/books/api/v3/vendor-payments/)  
**OAuth Scope:** `ZohoBooks.vendorpayments.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/vendorpayments` | Create a vendor payment |
| `GET` | `/vendorpayments` | List vendor payments |
| `PUT` | `/vendorpayments/{payment_id}` | Update a vendor payment |
| `GET` | `/vendorpayments/{payment_id}` | Get a vendor payment |
| `DELETE` | `/vendorpayments/{payment_id}` | Delete a vendor payment |
| `POST` | `/vendorpayments/{payment_id}/refunds` | Refund an excess payment |
| `GET` | `/vendorpayments/{payment_id}/refunds` | List refunds |
| `POST` | `/vendorpayments/{payment_id}/email` | Email a payment receipt |

#### Required Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `vendor_id` | string | Yes | Vendor contact ID |
| `amount` | double | Yes | Total payment amount |
| `date` | string | No | Payment date |
| `payment_mode` | string | No | Payment mode |
| `paid_through_account_id` | string | No | Bank/cash account |
| `bills` | array | No | Bills being paid |
| `tax_amount_withheld` | double | No | **India only:** TDS withheld |

#### Bills Application Array

```json
"bills": [
  {
    "bill_id": "460000000053199",
    "amount_applied": 150.00,
    "tax_amount_withheld": 0.10
  }
]
```

#### Create Vendor Payment Request

```json
{
  "vendor_id": "460000000026049",
  "date": "2026-03-28",
  "amount": 50000.00,
  "payment_mode": "banktransfer",
  "paid_through_account_id": "460000000000358",
  "reference_number": "REF-20260328",
  "tax_amount_withheld": 500.00,
  "bills": [
    {
      "bill_id": "460000000053199",
      "amount_applied": 50000.00,
      "tax_amount_withheld": 500.00
    }
  ]
}
```

---

### 7.10 Credit Notes

**Source:** [Zoho Books Credit Notes API](https://www.zoho.com/books/api/v3/creditnotes/)  
**OAuth Scope:** `ZohoBooks.creditnotes.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/creditnotes` | Create a credit note |
| `GET` | `/creditnotes` | List credit notes |
| `PUT` | `/creditnotes/{creditnote_id}` | Update a credit note |
| `GET` | `/creditnotes/{creditnote_id}` | Get a credit note |
| `DELETE` | `/creditnotes/{creditnote_id}` | Delete a credit note |
| `POST` | `/creditnotes/{creditnote_id}/status/open` | Mark as open |
| `POST` | `/creditnotes/{creditnote_id}/status/void` | Void a credit note |
| `POST` | `/creditnotes/{creditnote_id}/invoices` | Apply credit to invoice |
| `GET` | `/creditnotes/{creditnote_id}/invoices` | List invoices credited |
| `POST` | `/creditnotes/{creditnote_id}/refunds` | Refund a credit note |
| `GET` | `/creditnotes/{creditnote_id}/refunds` | List refunds |

#### India-Specific Credit Note Fields

| Field | Type | Description |
|-------|------|-------------|
| `place_of_supply` | string | State code — determines CGST+SGST vs IGST on credit note |
| `gst_no` | string | Customer's GSTIN |
| `gst_treatment` | string | Customer's GST treatment |
| `is_reverse_charge_applied` | boolean | Reverse charge flag |
| `hsn_or_sac` | string (line item) | HSN/SAC code |
| `reverse_charge_tax_id` | string (line item) | Reverse charge tax |

#### Apply Credit Note to Invoice

```json
POST /creditnotes/{creditnote_id}/invoices
{
  "invoices": [
    {
      "invoice_id": "460000000070001",
      "amount_applied": 5000.00
    }
  ]
}
```

---

### 7.11 Vendor Credits (Debit Notes)

**Source:** [Zoho Books Vendor Credits API](https://www.zoho.com/books/api/v3/vendor-credits/)  
**OAuth Scope:** `ZohoBooks.debitnotes.{ACTION}`

> **Naming Note:** Zoho Books calls these "Vendor Credits" in the API but they represent Debit Notes — i.e., notes issued by the buyer to the vendor for returns or adjustments.

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/vendorcredits` | Create a vendor credit (debit note) |
| `GET` | `/vendorcredits` | List vendor credits |
| `PUT` | `/vendorcredits/{vendor_credit_id}` | Update |
| `GET` | `/vendorcredits/{vendor_credit_id}` | Get |
| `DELETE` | `/vendorcredits/{vendor_credit_id}` | Delete |
| `POST` | `/vendorcredits/{vendor_credit_id}/status/open` | Convert to open |
| `POST` | `/vendorcredits/{vendor_credit_id}/status/void` | Void |
| `POST` | `/vendorcredits/{vendor_credit_id}/bills` | Apply credits to a bill |
| `GET` | `/vendorcredits/{vendor_credit_id}/bills` | List bills credited |
| `POST` | `/vendorcredits/{vendor_credit_id}/refunds` | Refund vendor credit |
| `GET` | `/vendorcredits/{vendor_credit_id}/refunds` | List refunds |

#### India-Specific Vendor Credit Fields

| Field | Type | Description |
|-------|------|-------------|
| `gst_no` | string | Vendor's GSTIN |
| `gst_treatment` | string | Vendor's GST treatment |
| `source_of_supply` | string | State where goods/services originate |
| `destination_of_supply` | string | State where goods/services delivered |
| `is_reverse_charge_applied` | boolean | Reverse charge flag |
| `hsn_or_sac` | string (line item) | HSN/SAC code |
| `reverse_charge_tax_id` | string (line item) | Reverse charge tax |

#### Create Vendor Credit (India GST)

```json
{
  "vendor_id": "460000000020029",
  "vendor_credit_number": "DN-00002",
  "date": "2026-03-28",
  "gst_treatment": "business_gst",
  "gst_no": "22AAAAA0000A1Z5",
  "source_of_supply": "TN",
  "destination_of_supply": "MH",
  "line_items": [
    {
      "item_id": "460000000020071",
      "name": "Damaged Goods Return",
      "hsn_or_sac": "40151910",
      "quantity": 10,
      "rate": 500.00,
      "tax_id": "460000000057089"
    }
  ]
}
```

---

### 7.12 Recurring Invoices

**Source:** [Zoho Books Recurring Invoices API](https://www.zoho.com/books/api/v3/recurring-invoices/)  
**OAuth Scope:** `ZohoBooks.invoices.{ACTION}`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/recurringinvoices` | Create a recurring invoice |
| `GET` | `/recurringinvoices` | List all recurring invoices |
| `PUT` | `/recurringinvoices/{recurring_invoice_id}` | Update |
| `GET` | `/recurringinvoices/{recurring_invoice_id}` | Get |
| `DELETE` | `/recurringinvoices/{recurring_invoice_id}` | Delete |
| `POST` | `/recurringinvoices/{recurring_invoice_id}/status/stop` | Stop |
| `POST` | `/recurringinvoices/{recurring_invoice_id}/status/resume` | Resume |
| `GET` | `/recurringinvoices/{recurring_invoice_id}/comments` | History |

#### Recurrence-Specific Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `recurrence_name` | string | Yes | Unique profile name (max 100 chars) |
| `recurrence_frequency` | string | Yes | `days`, `weeks`, `months`, `years` |
| `repeat_every` | integer | No | Interval (e.g., `2` with `weeks` = every 2 weeks) |
| `start_date` | string | No | First invoice date |
| `end_date` | string | No | Last invoice date |
| `last_sent_date` | string | Response only | Last invoice generated |
| `next_invoice_date` | string | Response only | Next scheduled date |

#### Create Recurring Invoice Request (India GST)

```json
{
  "recurrence_name": "MonthlySubscription-ABC",
  "customer_id": "903000000000099",
  "recurrence_frequency": "months",
  "repeat_every": 1,
  "start_date": "2026-04-01",
  "end_date": "2027-03-31",
  "gst_treatment": "business_gst",
  "gst_no": "22AAAAA0000A1Z5",
  "place_of_supply": "TN",
  "line_items": [
    {
      "name": "Platform Subscription",
      "hsn_or_sac": "998314",
      "product_type": "services",
      "quantity": 1,
      "rate": 10000.00,
      "tax_id": "460000000057089"
    }
  ]
}
```

---

## 8. Taxes & GST Settings API

**Source:** [Zoho Books Taxes API](https://www.zoho.com/books/api/v3/taxes/)  
**OAuth Scope:** `ZohoBooks.settings.{ACTION}`

### 8.1 Tax Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/settings/taxes` | Create a tax |
| `GET` | `/settings/taxes` | List taxes |
| `PUT` | `/settings/taxes/{tax_id}` | Update a tax |
| `GET` | `/settings/taxes/{tax_id}` | Get a tax |
| `DELETE` | `/settings/taxes/{tax_id}` | Delete a tax |
| `POST` | `/settings/taxgroups` | Create a tax group |
| `GET` | `/settings/taxgroups/{tax_group_id}` | Get a tax group |
| `PUT` | `/settings/taxgroups/{tax_group_id}` | Update a tax group |
| `DELETE` | `/settings/taxgroups/{tax_group_id}` | Delete a tax group |

### 8.2 Tax Fields

| Field | Type | Description |
|-------|------|-------------|
| `tax_id` | string | Unique tax ID (used in all transaction line items) |
| `tax_name` | string | Descriptive name (e.g., `IGST 18%`, `CGST 9%`) |
| `tax_percentage` | double | Percentage rate (e.g., `18.0`) |
| `tax_type` | string | `tax` (simple) or `compound_tax` |
| `tax_specific_type` | string | **India only** — see below |
| `is_value_added` | boolean | VAT-style tax flag |
| `is_editable` | boolean | Whether tax can be modified |

### 8.3 India GST Tax Types (`tax_specific_type`)

| Value | Description |
|-------|-------------|
| `igst` | Integrated GST — inter-state supply |
| `cgst` | Central GST — intra-state supply (50% of rate) |
| `sgst` | State/UT GST — intra-state supply (50% of rate) |
| `nil` | Nil-rated supply (0%) |
| `cess` | GST Cess — luxury/demerit goods |

### 8.4 Standard India GST Tax Setup Pattern

For each GST rate slab (5%, 12%, 18%, 28%), create:
1. **IGST tax** — full rate (e.g., 18%)
2. **CGST tax** — half rate (9%), `tax_specific_type: "cgst"`
3. **SGST tax** — half rate (9%), `tax_specific_type: "sgst"`
4. **Tax Group** combining CGST + SGST (appears as combined 18% on invoices)

```json
// Create IGST 18%
{
  "tax_name": "IGST 18%",
  "tax_percentage": 18,
  "tax_type": "tax",
  "tax_specific_type": "igst",
  "is_value_added": true
}

// Create CGST 9%
{
  "tax_name": "CGST 9%",
  "tax_percentage": 9,
  "tax_type": "tax",
  "tax_specific_type": "cgst",
  "is_value_added": true
}

// Create SGST 9%
{
  "tax_name": "SGST 9%",
  "tax_percentage": 9,
  "tax_type": "tax",
  "tax_specific_type": "sgst",
  "is_value_added": true
}

// Create Tax Group: CGST+SGST = 18%
POST /settings/taxgroups
{
  "tax_group_name": "GST 18% (CGST+SGST)",
  "taxes": "cgst_tax_id,sgst_tax_id"
}
```

Use the IGST tax ID for inter-state invoices and the CGST+SGST group for intra-state invoices — determined by `place_of_supply`.

### 8.5 Singapore GST Tax Setup

**Source:** [Zoho Books Singapore Tax Help](https://www.zoho.com/en-sg/books/help/settings/taxes.html)

Default taxes in Singapore edition (auto-created by Zoho Books):

| Tax Name | Rate | Description |
|----------|------|-------------|
| `GST9 [SR]` | 9% | Standard-rated supplies (sales) |
| `GST0 [ZR]` | 0% | Zero-rated supplies (exports) |
| `GST9 [TX]` | 9% | Standard-rated purchases (input tax) |
| `GST0 [ZP]` | 0% | Zero-rated purchases |
| `GST9 [BL]` | 9% | Disallowed expenses |
| `GST9 [IM]` | 9% | Import of goods |
| `GST9 [ME]` | 9% | Import under Major Exporter Scheme |
| `GST9 [IGDS]` | 9% | Import under IGDS |
| `GST9 [SRCA-S]` | 9% | Customer Accounting supply |
| `GST9 [TXRC-TS]` | 9% | Imported services/LVG (reverse charge) |
| `GST9 [SRRC]` | 9% | Imported services/LVG (customer RC) |

---

## 9. India GST Features

**Sources:** [Zoho Books India GST Help](https://www.zoho.com/in/books/help/gst/), [GSTR-3B Filing](https://www.zoho.com/in/books/help/gst/gst-filing.html), [E-Invoicing Setup](https://www.zoho.com/in/books/help/e-invoicing/set-up.html), [E-Invoicing Functions](https://www.zoho.com/in/books/help/e-invoicing/functions-einvoice.html), [TDS Help](https://www.zoho.com/in/books/help/settings/income-tds.html), [TCS Help](https://www.zoho.com/in/books/kb/taxes/add-tcs.html)

### 9.1 CGST / SGST / IGST Handling

The tax type is determined automatically based on **place of supply** vs. **org's home state**:

| Supply Type | Condition | Taxes Applied |
|-------------|-----------|---------------|
| **Intra-state** | `place_of_supply` == org's state | CGST + SGST (at equal halves) |
| **Inter-state** | `place_of_supply` != org's state | IGST (full rate) |
| **Export** | Overseas customer / `gst_treatment: "overseas"` | Zero-rated (IGST 0% or exempt) |
| **B2C** | `gst_treatment: "consumer"` | CGST + SGST (intra) or IGST (inter) |

**State Codes (Place of Supply)** — Standard 2-letter codes per state:

| Code | State | Code | State |
|------|-------|------|-------|
| `AP` | Andhra Pradesh | `MH` | Maharashtra |
| `AR` | Arunachal Pradesh | `MP` | Madhya Pradesh |
| `AS` | Assam | `MN` | Manipur |
| `BR` | Bihar | `ML` | Meghalaya |
| `CT` | Chhattisgarh | `MZ` | Mizoram |
| `GA` | Goa | `NL` | Nagaland |
| `GJ` | Gujarat | `OR` | Odisha |
| `HR` | Haryana | `PB` | Punjab |
| `HP` | Himachal Pradesh | `RJ` | Rajasthan |
| `JH` | Jharkhand | `SK` | Sikkim |
| `KA` | Karnataka | `TN` | Tamil Nadu |
| `KL` | Kerala | `TS` | Telangana |
| `LD` | Lakshadweep | `TR` | Tripura |
| `DL` | Delhi | `UP` | Uttar Pradesh |
| `DN` | Dadra & Nagar Haveli | `UT` | Uttarakhand |
| `DD` | Daman & Diu | `WB` | West Bengal |

### 9.2 Place of Supply Logic in API

Set `place_of_supply` on every invoice/bill/credit note:

```json
{
  "customer_id": "...",
  "place_of_supply": "MH",   // Customer in Maharashtra
  // Org state is Karnataka (KA)
  // → MH != KA → IGST applies
  "line_items": [
    {
      "tax_id": "igst_18_tax_id"  // Use IGST tax
    }
  ]
}
```

### 9.3 HSN / SAC Codes

- **HSN (Harmonized System of Nomenclature)** — for goods (4, 6, or 8 digits)
- **SAC (Services Accounting Code)** — for services (typically 6 digits starting with `99`)

Set on each line item via `hsn_or_sac` field. Required for:
- B2B invoices (all turnover levels)
- B2C invoices (turnover above ₹5 crore)

```json
{
  "name": "Surgical Mask",
  "hsn_or_sac": "63079090",   // HSN for woven fabrics (masks)
  "product_type": "goods"
}
```

Common HSN/SAC codes for medical/pharma:
- `30049099` — Medicaments
- `40151910` — Medical/surgical gloves
- `63079090` — Other made-up textile articles (masks)
- `998314` — IT implementation/consulting services (SAC)

### 9.4 GSTR-1 and GSTR-3B via API

Zoho Books integrates with the GSTN (GST Network) via the **GST Portal API** — it does NOT expose GSTR filing as direct Zoho Books REST endpoints. The flow is:

#### GSTR-3B Filing Flow

1. **Enable API Access on GST Portal:**
   - Login to GST Portal → My Profile → Manage API Access → Enable
   
2. **Configure Online Filing in Zoho Books:**
   - Settings → Taxes & Compliance → Taxes → Online Filing Settings
   - Enter GSTN Username and Reporting Period

3. **Push Transactions to GSTN via Zoho Books UI (or programmatic trigger):**
   - Navigate to GST Filing → GSTR-3B → View Summary
   - Click "Push to GSTN" (OTP-based authentication via GSTN registered mobile/email)
   - Data pushed from Zoho Books to GST Portal

4. **File on GST Portal:** Complete filing at gstin.gov.in

5. **Record in Zoho Books:** Record GST payment, create journal entries for ITC

#### GSTR-1 Filing Flow

Similar flow: GST Filing → GSTR-1 → Push to GSTN → File on portal

**Important for Shopify integration:** The push to GSTN is UI-driven and OTP-gated. There is no direct REST API endpoint in Zoho Books to trigger `Push to GSTN` programmatically. The integration keeps transaction data GST-compliant via API fields; filing remains a human-in-the-loop step via Zoho Books interface.

**Supported GST Returns in Zoho Books India:**
- GSTR-1 (monthly/quarterly outward supply)
- GSTR-2 (reconciliation)
- GSTR-2B (reconciliation)
- GSTR-3B (monthly summary return)
- GSTR-9 (annual return)

### 9.5 E-Invoicing (IRN Generation)

**Source:** [Zoho Books E-Invoicing](https://www.zoho.com/in/books/help/e-invoicing/), [E-Invoicing Functions](https://www.zoho.com/in/books/help/e-invoicing/functions-einvoice.html), [E-Invoicing Setup](https://www.zoho.com/in/books/help/e-invoicing/set-up.html)

#### Overview

Zoho Books is registered as a **GST Suvidha Provider (GSP)** and integrates directly with the **Invoice Registration Portal (IRP)** for e-invoicing.

#### Eligibility

Mandatory for businesses with turnover above ₹5 crore (as of current regulations). Applicable to:
- B2B Invoices
- Credit Notes
- Debit Notes
- Export Invoices

#### E-Invoice Integration Flow

1. **Register Zoho Corporation as GSP on IRP:**
   - Login to IRP → API Registration → Create API User → Through GSP → Select "Zoho Corporation"
   - Set username/password for IRP API access

2. **Connect Zoho Books to IRP:**
   - Settings → e-Invoicing → Connect Now
   - Enter IRP credentials

3. **Push Invoice to IRP (generates IRN):**
   - From Invoice detail page → Push to IRP button
   - IRP validates and returns **IRN** (Invoice Reference Number) + **QR Code**

4. **IRN Fields in Invoice Response:**
   - `irn` — Unique 64-character hash (Invoice Reference Number)
   - `ack_no` — Acknowledgement number
   - `ack_date` — Acknowledgement date
   - `qr_code` — QR code string (embed in invoice PDF)
   - `e_invoice_status` — Status of the e-invoice

#### Bulk JSON Export for IRN Generation

Zoho Books allows bulk export of invoices as JSON in e-invoice schema format:
- Invoices → Select multiple → More → Export as JSON for E-Invoicing
- Upload to IRP portal directly for bulk IRN generation

#### Cancel E-Invoice

Cancellation must occur within **24 hours** of IRN generation. After 24 hours, the invoice can only be amended on the GST portal.

#### E-Way Bill Integration

When creating a push to IRP, you can simultaneously push e-Way Bill details — generating both IRN and EWB number in one step.

### 9.6 TDS (Tax Deducted at Source)

**Source:** [Zoho Books TDS Help](https://www.zoho.com/in/books/help/settings/income-tds.html)

TDS can be applied at:
- **Transaction level** (entire invoice/bill)
- **Line item level** (per line)

**Applicable transactions:** Invoices, Bills, Customer Payments, Vendor Payments, Credit Notes, Vendor Credits, Sales Orders, Purchase Orders, Recurring Invoices/Bills

**API fields:**
- `tds_tax_id` on line items — applies TDS to that line
- `tax_amount_withheld` on Customer Payments / Vendor Payments — amount TDS withheld at payment
- TDS taxes must be configured in Settings → Direct Taxes → TDS first

### 9.7 TCS (Tax Collected at Source)

**Source:** [Zoho Books TCS Help](https://www.zoho.com/in/books/kb/taxes/add-tcs.html)

TCS auto-triggers on invoices when cumulative sales to a PAN exceed ₹50 lakhs in a financial year.

**TCS rates:**
- With PAN: **0.1%**
- Without PAN: **1.0%**

**API field:** `tcs_tax_id` on invoice

### 9.8 Reverse Charge Mechanism

| Field | Endpoint | Description |
|-------|----------|-------------|
| `is_reverse_charge_applied` | Invoice, Bill, Credit Note, Vendor Credit | Boolean flag |
| `reverse_charge_tax_id` | Line items | Tax to apply under reverse charge |

When `is_reverse_charge_applied: true`, the buyer is liable to pay GST to the government instead of the supplier.

---

## 10. Singapore GST Features

**Sources:** [Zoho Books Singapore GST Help](https://www.zoho.com/en-sg/books/help/settings/taxes.html), [GST F5 Returns](https://www.zoho.com/en-sg/books/help/gst/f5-return.html), [Singapore GST Basics](https://www.zoho.com/en-sg/books/academy/taxes-and-compliance/gst-singapore-basics-compliance-guide.html)

### 10.1 Singapore GST Overview

| Item | Detail |
|------|--------|
| Standard GST Rate | **9%** (as of 2024/2025) |
| Previous Rate | 8% (transitional period tax codes `GST8 [SR8]`, `GST8 [TX-8]` still supported) |
| Zero-rated | 0% for exports and international services |
| Exempt | Financial services, sale/lease of residential properties, import/supply of Investment Precious Metals (IPM) |
| Filing Frequency | Quarterly |
| Reporting Form | **GST F5** |
| Tax Authority | **IRAS** (Inland Revenue Authority of Singapore) |
| Registration Number | 10-digit GST Registration Number |
| Data Center | `.com` (US/Singapore DC) |

### 10.2 Automatic Tax Rate Application

Zoho Books Singapore edition automatically applies the correct GST rate based on:
1. **Tax Category** of the item (Default Product, Electronics, Professional Services, etc.)
2. **Customer's country/region** in billing address

| Customer Type | Tax Category | Rate Applied |
|---------------|-------------|-------------|
| Local (Singapore) | Standard goods/services | 9% |
| Overseas | Any | 0% (zero-rated) |
| Local B2B — Prescribed Goods >SGD 10,000 | Electronics (smartphones, tablets) | 9% (Customer Accounting) |
| Exempt category | Financial services, IPM | Exempt |

### 10.3 GST F5 Return

**Manual filing workflow** (no direct API trigger for filing to IRAS):

1. Generate F5 Return in Zoho Books: Filing & Compliance → F5 Returns → Generate
2. Review amounts (Box 1–14 of F5 form)
3. Export as PDF
4. Submit manually to IRAS via myTax Portal
5. Mark as Filed in Zoho Books: specify Date of Filing

**F5 Return Sections Generated:**
- Box 1: Total value of standard-rated supplies
- Box 2: Total value of zero-rated supplies  
- Box 3: Total value of exempt supplies
- Box 4: Total value of out-of-scope supplies
- Box 5: Total value of taxable purchases
- Box 6: Output tax due
- Box 7: Input tax and refunds claimed
- Box 8: Net GST payable/claimable

### 10.4 API Behavior for Singapore

- No India-specific fields (no `gst_no`, `place_of_supply`, `hsn_or_sac`, `gst_treatment`)
- Tax applied via `tax_id` referencing Singapore GST tax codes
- `is_inclusive_tax` — whether price includes GST
- `tax_treatment` used for some scenarios (imported services reverse charge: `GST9 [TXRC-TS]`)

#### Invoice for Singapore Customer

```json
{
  "customer_id": "460000000000111",
  "date": "2026-03-28",
  "currency_id": "sgd_currency_id",
  "line_items": [
    {
      "name": "Platform Subscription",
      "quantity": 1,
      "rate": 500.00,
      "tax_id": "gst9_sr_tax_id"    // GST9 [SR] = 9%
    }
  ]
}
```

### 10.5 Singapore-Specific GST Scenarios

**Exported goods/services (zero-rated):**
```json
{
  "tax_id": "gst0_zr_tax_id"   // GST0 [ZR]
}
```

**Customer Accounting (Prescribed Goods >SGD 10k):**
```json
{
  "tax_id": "gst9_srca_s_tax_id"   // GST9 [SRCA-S]
}
```

**Imported services (reverse charge):**
```json
{
  "tax_id": "gst9_txrc_ts_tax_id"   // GST9 [TXRC-TS]
}
```

---

## 11. Webhooks

**Source:** [Zoho Books Webhooks API](https://www.zoho.com/books/api/v3/webhooks/)  
**OAuth Scope:** `ZohoBooks.settings.{ACTION}`

### 11.1 Webhook Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/settings/webhooks` | Create a webhook |
| `GET` | `/settings/webhooks` | List webhooks |
| `PUT` | `/settings/webhooks/{webhook_id}` | Update a webhook |
| `GET` | `/settings/webhooks/{webhook_id}` | Get a webhook |
| `DELETE` | `/settings/webhooks/{webhook_id}` | Delete a webhook |
| `GET` | `/settings/webhooks/histories` | List webhook execution histories |
| `GET` | `/settings/webhooks/histories/{history_id}` | Get history detail |
| `POST` | `/settings/webhooks/histories/{history_id}/resend` | Resend a failed webhook |
| `POST` | `/settings/webhooks/histories/bulkresend` | Bulk resend (max 10) |
| `GET` | `/settings/webhooks/editpage` | Get webhook edit page data |

### 11.2 Webhook Configuration Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `webhook_name` | string | Yes | Descriptive name |
| `entity` | string | Yes | Entity type: `invoice`, `contact`, `bill`, etc. |
| `url` | string | Yes | Target HTTPS URL |
| `method` | string | Yes | HTTP method: `POST`, `GET`, `PUT`, `DELETE` |
| `events` | array | Yes | List of trigger events |
| `secret` | string | No | HMAC secret for signature (12–50 alphanumeric chars) |
| `headers` | array | No | Custom HTTP headers `[{param_name, param_value}]` |
| `entity_parameters` | array | No | Entity field values to include in payload |
| `additional_parameters` | array | No | Custom key-value pairs in payload |
| `body_type` | string | No | `raw` or `form-data` |
| `raw_data` | string | No | Raw JSON payload template (max 100,000 chars) |
| `form_data` | array | No | Form data `[{param_name, param_value}]` |
| `query_parameters` | array | No | URL query params |
| `description` | string | No | Description |
| `is_new_response_format` | boolean | No | Use new payload format |

### 11.3 Available Webhook Entities / Events

Entities correspond to Zoho Books modules:

| Entity | Typical Events |
|--------|---------------|
| `invoice` | Created, Updated, Deleted, Status changed (Sent/Paid/Voided) |
| `contact` | Created, Updated, Deleted |
| `bill` | Created, Updated, Deleted, Paid |
| `creditnote` | Created, Voided |
| `vendorcredit` | Created, Voided |
| `payment` | Created, Deleted |
| `expense` | Created, Updated |
| `salesorder` | Created, Confirmed |
| `purchaseorder` | Created, Approved |
| `journal` | Created, Published |

### 11.4 Create Webhook Request

```json
{
  "webhook_name": "Invoice Created - Shopify Sync",
  "entity": "invoice",
  "url": "https://your-shopify-app.com/webhooks/zohobooks",
  "method": "POST",
  "events": ["invoice_created", "invoice_updated", "invoice_status_changed"],
  "secret": "shopify123zoho456",
  "headers": [
    {
      "param_name": "X-App-Source",
      "param_value": "zohobooks"
    }
  ],
  "is_new_response_format": true
}
```

### 11.5 Webhook Security (HMAC)

When `secret` is configured, Zoho Books signs the webhook payload with HMAC-SHA256. Verify on your server:

```python
import hmac, hashlib

def verify_webhook(payload_body: bytes, signature: str, secret: str) -> bool:
    computed = hmac.new(
        secret.encode('utf-8'),
        payload_body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(computed, signature)
```

### 11.6 Retry Logic

| Field | Description |
|-------|-------------|
| `retry_count` | Number of retry attempts made |
| `last_execution_time` | Timestamp of last attempt |
| `next_retry_time` | Scheduled next retry |

Failed webhooks can be manually resent via:
- `POST /settings/webhooks/histories/{history_id}/resend` (single)
- `POST /settings/webhooks/histories/bulkresend?history_ids=id1,id2` (up to 10)

### 11.7 Webhook History Filtering

```
GET /settings/webhooks/histories
  ?organization_id={org_id}
  &webhook_id={webhook_id}
  &entity_id={entity_id}
  &from_date=2026-03-01
  &to_date=2026-03-31
  &filter_by=Status.Failed
```

---

## 12. Multi-Currency

**Sources:** [Zoho Books Currency API](https://www.zoho.com/books/api/v3/currency/), [Base Currency Adjustment API](https://www.zoho.com/books/api/v3/base-currency-adjustment/)

### 12.1 Currency API Endpoints

| Method | Path | Description | Scope |
|--------|------|-------------|-------|
| `POST` | `/settings/currencies` | Create a currency | `ZohoBooks.settings.CREATE` |
| `GET` | `/settings/currencies` | List currencies | `ZohoBooks.settings.READ` |
| `PUT` | `/settings/currencies/{currency_id}` | Update | `ZohoBooks.settings.UPDATE` |
| `GET` | `/settings/currencies/{currency_id}` | Get | `ZohoBooks.settings.READ` |
| `DELETE` | `/settings/currencies/{currency_id}` | Delete | `ZohoBooks.settings.DELETE` |
| `POST` | `/settings/currencies/{currency_id}/exchangerates` | Create exchange rate | `ZohoBooks.settings.CREATE` |
| `GET` | `/settings/currencies/{currency_id}/exchangerates` | List exchange rates | `ZohoBooks.settings.READ` |
| `PUT` | `/settings/currencies/{currency_id}/exchangerates/{exchange_rate_id}` | Update rate | `ZohoBooks.settings.UPDATE` |
| `GET` | `/settings/currencies/{currency_id}/exchangerates/{exchange_rate_id}` | Get rate | `ZohoBooks.settings.READ` |
| `DELETE` | `/settings/currencies/{currency_id}/exchangerates/{exchange_rate_id}` | Delete rate | `ZohoBooks.settings.DELETE` |
| `POST` | `/basecurrencyadjustment` | Create base currency adjustment | `ZohoBooks.settings.CREATE` |

### 12.2 How Multi-Currency Works

1. **Base Currency** is set when the org is created (INR for India, SGD for Singapore) — cannot be changed after transactions exist
2. **Foreign currencies** are added via `POST /settings/currencies`
3. **Exchange rates** are managed via `/exchangerates` endpoints
4. **Automatic exchange rate feeds** via Open Exchange Rates (enabled by default — disable to manage manually)
5. All **reports** are generated in base currency; individual transaction screens show FCY (foreign currency) amounts

### 12.3 Exchange Rate Management

**Create exchange rate** for a specific date:
```json
POST /settings/currencies/{currency_id}/exchangerates
{
  "effective_date": "2026-03-28",
  "rate": 83.15    // 1 USD = 83.15 INR (for India org)
}
```

**List exchange rates** with date filter:
```
GET /settings/currencies/{currency_id}/exchangerates
  ?from_date=2026-01-01
  &sort_column=effective_date
```

### 12.4 Multi-Currency on Transactions

Every transaction supports:

| Field | Type | Description |
|-------|------|-------------|
| `currency_id` | string | Transaction currency |
| `currency_code` | string | ISO code (e.g., `USD`, `SGD`, `INR`) |
| `exchange_rate` | double | Rate at time of transaction |

Zoho Books automatically looks up the rate for the transaction date. Pass `exchange_rate` explicitly to override.

### 12.5 Base Currency Adjustment

Use when exchange rates fluctuate and you need to adjust the book value of open FCY balances:

```json
POST /basecurrencyadjustment
{
  "currency_id": "460000000000109",     // e.g., USD currency ID
  "adjustment_date": "2026-03-31",
  "exchange_rate": 83.50,
  "notes": "March month-end rate adjustment",
  "account_ids": ["460000000000364"]   // Accounts to adjust (AR, AP, etc.)
}
```

### 12.6 Currency Response Format

```json
{
  "code": 0,
  "message": "success",
  "currencies": [
    {
      "currency_id": "982000000004012",
      "currency_code": "USD",
      "currency_name": "USD - US Dollar",
      "currency_symbol": "$",
      "price_precision": 2,
      "currency_format": "1,234,567.89",
      "is_base_currency": false,
      "exchange_rate": 83.15,
      "effective_date": "2026-03-28"
    }
  ]
}
```

---

## 13. Reports API

**Source:** [Zoho Books Help](https://www.zoho.com/in/books/help/gst/), [Zoho Books API Intro](https://www.zoho.com/books/api/v3/introduction/)

### 13.1 Overview

The Zoho Books v3 API does not expose a dedicated `/reports/` endpoint for programmatic report generation. Reports are primarily UI-driven features. However, underlying financial data can be assembled via:

1. **Transactions APIs** (invoices, bills, payments) with date filters for custom reporting
2. **Chart of Accounts transactions** (`GET /chartofaccounts/transactions`) for ledger data
3. **GST Filing integration** for GSTR data (via GSTN push mechanism)

### 13.2 Available Report Data via API

| Report Type | API Approach |
|-------------|-------------|
| Sales Report | `GET /invoices?date_start=&date_end=&status=paid` |
| Purchase Report | `GET /bills?date_start=&date_end=&status=paid` |
| Payment Summary | `GET /customerpayments?date_start=&date_end=` + `GET /vendorpayments?date_start=&date_end=` |
| Outstanding Receivables | `GET /invoices?filter_by=Status.Unpaid` + `Status.PartiallyPaid` |
| Outstanding Payables | `GET /bills?filter_by=Status.Unpaid` |
| GL Ledger | `GET /chartofaccounts/transactions?account_id=&date_start=&date_end=` |
| Journal Entries | `GET /journals?date_start=&date_end=` |
| Bank Reconciliation | `GET /banktransactions?status=uncategorized` |

### 13.3 GSTR-1 / GSTR-3B Data

GSTR data is assembled by Zoho Books from transaction fields (`hsn_or_sac`, `place_of_supply`, `gst_treatment`, `gst_no`). The actual GSTR JSON can be:
- **Generated and downloaded** from the Zoho Books GST Filing UI
- **Pushed to GSTN** via the OTP-authenticated push mechanism

There is no API endpoint to trigger `GET /gstr1` or `POST /gstr3b/file` — these are portal-integrated operations, not REST endpoints.

### 13.4 Singapore F5 Return

**Source:** [GST F5 Returns Help](https://www.zoho.com/en-sg/books/help/gst/f5-return.html)

Similarly, the GST F5 Return is UI-generated in Zoho Books and exported as PDF for manual submission to IRAS via myTax Portal. No direct API endpoint to trigger F5 generation or IRAS submission programmatically.

**Workflow:** Filing & Compliance → F5 Returns → Generate → Export PDF → Submit to IRAS myTax → Mark as Filed

---

## Quick Reference: Shopify Integration Checklist

### India Organization Setup

1. Register app on **api-console.zoho.in**
2. Get token from `https://accounts.zoho.in/oauth/v2/token`
3. API base: `https://www.zohoapis.in/books/v3/`
4. Scopes needed: `ZohoBooks.contacts.ALL,ZohoBooks.invoices.ALL,ZohoBooks.bills.ALL,ZohoBooks.customerpayments.ALL,ZohoBooks.vendorpayments.ALL,ZohoBooks.creditnotes.ALL,ZohoBooks.debitnotes.ALL,ZohoBooks.settings.ALL,ZohoBooks.accountants.ALL,ZohoBooks.banking.ALL`
5. Set up GST taxes (IGST, CGST, SGST, Cess for each slab)
6. Configure e-invoicing via Settings → e-Invoicing
7. Enable GSTN Online Filing in Settings → Taxes

### Singapore Organization Setup

1. Register app on **api-console.zoho.com**
2. Get token from `https://accounts.zoho.com/oauth/v2/token`
3. API base: `https://www.zohoapis.com/books/v3/`
4. Same scope list as above
5. Default GST tax codes (`GST9 [SR]`, `GST0 [ZR]`, etc.) auto-created
6. Configure GST Registration Number in Settings → Taxes

### Per-Transaction GST Fields Summary

| Field | India | Singapore |
|-------|-------|-----------|
| Customer tax status | `gst_treatment` | (none — use billing address country) |
| Customer GSTIN | `gst_no` | (use GST registration number) |
| Supply location | `place_of_supply` | (not applicable) |
| Item tax code | `hsn_or_sac` | (not applicable) |
| Item type | `product_type` (goods/services) | (not used) |
| Tax field | `tax_id` → IGST/CGST+SGST | `tax_id` → GST9/GST0 code |
| Reverse charge | `is_reverse_charge_applied` | `GST9 [SRRC]` / `GST9 [TXRC-TS]` |
| TDS | `tds_tax_id` on line items | (not applicable) |
| TCS | `tcs_tax_id` on invoice | (not applicable) |
| E-invoice | `irn`, `ack_no`, `qr_code` | (not applicable) |

---

*All data sourced from official Zoho Books API documentation at https://www.zoho.com/books/api/v3/ and Zoho Books Help Center. Research conducted March 28, 2026.*
