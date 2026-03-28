# Shopify API Research: Building an Accounting App
**Compiled:** March 2026  
**Purpose:** Comprehensive technical reference for building a Shopify accounting SaaS platform  
**Primary Sources:** [Shopify Dev Docs](https://shopify.dev/docs), [Shopify API Reference](https://shopify.dev/docs/api)

---

## Table of Contents
1. [Shopify App Types](#1-shopify-app-types)
2. [Authentication & OAuth 2.0](#2-authentication--oauth-20)
3. [Admin REST API — Key Accounting Endpoints](#3-admin-rest-api--key-accounting-endpoints)
4. [Admin GraphQL API](#4-admin-graphql-api)
5. [Shopify Payments / Balance API](#5-shopify-payments--balance-api)
6. [Webhooks](#6-webhooks)
7. [Billing API](#7-billing-api)
8. [Rate Limits](#8-rate-limits)
9. [Shopify Tax API](#9-shopify-tax-api)
10. [Shopify Markets (Multi-Currency)](#10-shopify-markets-multi-currency)
11. [App Bridge & Embedded Apps](#11-app-bridge--embedded-apps)
12. [Data Privacy & GDPR Webhooks](#12-data-privacy--gdpr-webhooks)

---

## 1. Shopify App Types

### Overview

As of January 2023, Shopify supports two active app distribution types — [Public Apps](https://shopify.dev/docs/apps/launch/distribution) and Custom Apps. The former "Private Apps" category was deprecated in January 2022 and all existing private apps were auto-migrated to Custom Apps by January 2023.

| Feature | Public App | Custom App |
|---|---|---|
| **Distribution** | Shopify App Store (any merchant can install) | Single store or partner-managed install |
| **Review Required** | Yes — formal Shopify review process | No |
| **OAuth Flow** | Standard OAuth 2.0 (authorization code grant) | Client credentials or direct install |
| **Billing** | Must use Shopify Billing API | Flexible |
| **Ideal For** | SaaS accounting platforms serving many merchants | Single-merchant integrations, ERPs |
| **App Store Listing** | Yes | No |

### Recommendation for a SaaS Accounting Platform

**Public App** is the correct choice for a multi-tenant SaaS accounting platform like myMediset. It allows any Shopify merchant to discover and install the app through the Shopify App Store. All billing must go through the [Shopify Billing API](https://shopify.dev/docs/apps/billing).

### App Store Submission Requirements (Key Requirements)

Source: [Shopify App Store Requirements](https://shopify.dev/docs/apps/launch/shopify-app-store/app-store-requirements)

**Policy & Platform:**
- Must use Shopify APIs — apps that don't use Shopify APIs are not permitted
- Must use session tokens for authentication (requirement 1.1.1)
- Must use [Shopify Billing API](https://shopify.dev/docs/apps/billing) or Managed Pricing for all charges (no off-platform billing)
- Cannot circumvent core Shopify workflows
- Must build web-based apps (no desktop-app-only requirement)

**Functionality:**
- Must be free from UI bugs and error pages (404s, 500s) during review
- Must implement App Bridge (latest version, all apps since March 13, 2024)
- Must provide consistent embedded experience within Shopify Admin
- Admin extensions must be feature-complete

**Installation:**
- App must be installed only from Shopify-owned surfaces
- No manual myshopify.com URL entry during installation

**Branding:**
- App name must be consistent between Developer Dashboard and App Store listing
- Screenshots must be 1600×900px (16:9), 3–6 screenshots required

**Privacy & Compliance:**
- Must implement all 3 mandatory GDPR compliance webhooks
- Must verify HMAC signatures on webhooks

**Geographic Requirements:**
- Must specify geographic requirements (e.g., India-only, specific currencies) in the app listing

---

## 2. Authentication & OAuth 2.0

### Three OAuth Grant Types

Source: [Shopify Token Acquisition Docs](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens), [EzApps OAuth Guide 2026](https://ezapps.io/blogs/shopify-oauth-access-tokens-guide)

| Grant Type | Use Case | Token Expiry | Requires User |
|---|---|---|---|
| **Authorization Code** | Public apps — merchant-authorized access | Never expires (offline) | Yes |
| **Client Credentials** | Your own stores, machine-to-machine | 24 hours (86400s) | No |
| **Token Exchange** | Embedded apps with App Bridge | Short-lived (online) | No (uses session token) |

### Authorization Code Flow (for SaaS/Public Apps)

This is the standard flow for public apps. A merchant installs the app, Shopify presents the OAuth consent screen, and upon approval, your app receives a permanent offline access token.

**Step-by-step:**
1. Build auth URL: `https://{shop}.myshopify.com/admin/oauth/authorize?client_id={api_key}&scope={scopes}&redirect_uri={redirect_url}&state={nonce}`
2. Merchant consents on Shopify's grant screen
3. Shopify redirects to `redirect_uri` with `?code={code}&hmac={hmac}&shop={shop}&state={state}`
4. Verify `state` (CSRF protection) and `hmac` signature
5. Exchange code for token: `POST https://{shop}.myshopify.com/admin/oauth/access_token`
   ```json
   {
     "client_id": "api_key",
     "client_secret": "api_secret",
     "code": "auth_code"
   }
   ```
6. Response: `{ "access_token": "shpua_xxxx", "scope": "read_orders,read_products" }`

**HMAC Verification on Callback:**
```javascript
const hmac = crypto
  .createHmac('sha256', clientSecret)
  .update(queryParamsWithoutHmac)
  .digest('hex');
// Compare with timing-safe equal
```

### Token Exchange Flow (for Embedded Apps)

For embedded apps using App Bridge, use the token exchange grant to exchange the short-lived session token (JWT from App Bridge) for an API access token:

```
POST https://{shop}.myshopify.com/admin/oauth/access_token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
client_id={client_id}
client_secret={client_secret}
subject_token={session_token_from_app_bridge}
subject_token_type=urn:ietf:params:oauth:token-type:id_token
requested_token_type=urn:shopify:params:oauth:token-type:offline-access-token
```

### Offline vs Online Access Tokens

- **Offline tokens** (`offline-access-token`): Never expire. Used for background jobs, cron tasks, syncing. Best for accounting apps that need to run scheduled data pulls.
- **Online tokens**: Expire when the merchant logs out. Tied to a specific user session. Used for user-present interactions in embedded apps.

For an accounting app, **request an offline token** so you can sync orders, payouts, and financial data without the merchant being actively logged in.

### Access Scopes for an Accounting App

Source: [Shopify API Access Scopes](https://shopify.dev/docs/api/usage/access-scopes)

| Scope | What It Accesses |
|---|---|
| `read_orders` | Orders, transactions, fulfillments, abandoned checkouts |
| `read_all_orders` | All orders beyond the default 60-day window (**requires Shopify approval**) |
| `read_products` | Products, variants, collections |
| `read_customers` | Customer data |
| `read_inventory` | Inventory levels and items |
| `read_shipping` | Shipping rates, carrier services |
| `read_locations` | Store locations |
| `read_shopify_payments_payouts` | Shopify Payments payouts, balance, and balance transactions |
| `read_shopify_payments_disputes` | Dispute/chargeback data |
| `read_shopify_payments_dispute_evidences` | Dispute evidence documents |
| `read_returns` | Returns and return line items |
| `read_draft_orders` | Draft orders |
| `read_reports` | Analytics/reporting via ShopifyQL |
| `read_markets` | Markets configuration (for multi-currency) |

> **Critical:** `read_all_orders` is a restricted scope. You must request approval from Shopify through the Partner Dashboard before you can use it. Without it, you can only access orders from the last 60 days. Accounting apps that need full historical data **must** request this scope.

**To request `read_all_orders` approval:**
1. Partner Dashboard → Apps → [Your App] → API access
2. Access requests section → Read all orders scope → Request access
3. Describe your app's accounting use case

### Checking Granted Scopes

```
GET https://{shop}.myshopify.com/admin/oauth/access_scopes.json
X-Shopify-Access-Token: {access_token}
```

---

## 3. Admin REST API — Key Accounting Endpoints

> **Important Deprecation Notice:** The REST Admin API was declared a legacy API as of October 1, 2024. Starting April 1, 2025, all **new public apps** must be built exclusively with the GraphQL Admin API. Existing apps should migrate to GraphQL. REST documentation is retained below as it remains functional for existing apps and useful for reference.

All REST endpoints follow this pattern:
```
https://{store_name}.myshopify.com/admin/api/2026-01/{resource}.json
X-Shopify-Access-Token: {access_token}
```

Source: [REST Admin API Reference](https://shopify.dev/docs/api/admin-rest)

### 3.1 Orders

**Endpoint:** `GET /admin/api/2026-01/orders.json`

**Key Query Parameters:**
| Parameter | Description |
|---|---|
| `status` | `open`, `closed`, `cancelled`, `any` |
| `financial_status` | `authorized`, `pending`, `paid`, `partially_paid`, `refunded`, `voided`, `partially_refunded`, `any` |
| `fulfillment_status` | `shipped`, `partial`, `unshipped`, `any`, `unfulfilled` |
| `created_at_min` / `created_at_max` | Date range filter |
| `updated_at_min` / `updated_at_max` | Updated date range |
| `limit` | Max 250 per page |
| `page_info` | Cursor for pagination |
| `fields` | Comma-separated field list to reduce response size |

**Key Financial Fields in Order Object:**
```json
{
  "id": 820982911946154508,
  "name": "#1001",
  "created_at": "2024-01-15T10:00:00-05:00",
  "currency": "USD",
  "presentment_currency": "CAD",
  "financial_status": "paid",
  "fulfillment_status": "fulfilled",
  
  "subtotal_price": "100.00",
  "subtotal_price_set": {
    "shop_money": {"amount": "100.00", "currency_code": "USD"},
    "presentment_money": {"amount": "135.00", "currency_code": "CAD"}
  },
  "total_tax": "13.50",
  "total_tax_set": { "shop_money": {...}, "presentment_money": {...} },
  "total_price": "113.50",
  "total_price_set": {...},
  "total_discounts": "10.00",
  "total_discounts_set": {...},
  "total_shipping_price_set": {...},
  "current_total_price": "113.50",
  "current_subtotal_price": "100.00",
  "current_total_tax": "13.50",
  "taxes_included": false,
  "tax_lines": [
    {
      "title": "State tax",
      "price": "13.50",
      "price_set": {"shop_money": {...}, "presentment_money": {...}},
      "rate": 0.06,
      "channel_liable": false
    }
  ],
  "line_items": [
    {
      "id": 866550311766439020,
      "title": "Product Name",
      "quantity": 2,
      "price": "50.00",
      "price_set": {...},
      "sku": "ABC-123",
      "variant_id": 123456,
      "product_id": 789012,
      "taxable": true,
      "tax_lines": [...],
      "discount_allocations": [...],
      "total_discount": "10.00",
      "fulfillment_status": "fulfilled",
      "vendor": "My Vendor"
    }
  ],
  "shipping_lines": [
    {
      "title": "Standard Shipping",
      "price": "5.00",
      "price_set": {...},
      "discounted_price": "5.00",
      "tax_lines": [...],
      "code": "Standard"
    }
  ],
  "discount_codes": [{"code": "SAVE10", "amount": "10.00", "type": "percentage"}],
  "discount_applications": [...],
  "refunds": [...],
  "payment_gateway_names": ["shopify_payments"],
  "gateway": "shopify_payments",
  "processing_method": "direct",
  "customer": {"id": 123, "email": "customer@example.com", ...},
  "billing_address": {...},
  "shipping_address": {...},
  "note": "Customer note",
  "tags": "wholesale,VIP",
  "source_name": "web",
  "buyer_accepts_marketing": true
}
```

**`financial_status` Values:**
- `authorized` — payment authorized but not captured
- `pending` — payment pending
- `paid` — payment captured
- `partially_paid` — partial payment received
- `refunded` — fully refunded
- `voided` — payment voided
- `partially_refunded` — partial refund issued

### 3.2 Order Transactions

**Endpoint:** `GET /admin/api/2026-01/orders/{order_id}/transactions.json`

Returns all payment gateway transactions for a specific order.

```json
{
  "transactions": [
    {
      "id": 389404469,
      "order_id": 450789469,
      "kind": "sale",
      "status": "success",
      "amount": "41.94",
      "currency": "USD",
      "gateway": "shopify_payments",
      "created_at": "2024-01-15T10:00:00-05:00",
      "processed_at": "2024-01-15T10:00:00-05:00",
      "parent_id": null,
      "receipt": {...},
      "payment_id": "ch_xxx",
      "message": null,
      "error_code": null,
      "payment_details": {
        "credit_card_bin": "123456",
        "avs_result_code": "Y",
        "cvv_result_code": "M",
        "credit_card_number": "•••• •••• •••• 1234",
        "credit_card_company": "Visa"
      }
    }
  ]
}
```

**Transaction `kind` values:**
- `authorization` — funds authorized, not captured
- `capture` — funds captured from a prior authorization
- `sale` — single-step charge (authorization + capture)
- `void` — cancels an authorization
- `refund` — returns funds to customer

**Transaction `status` values:** `pending`, `failure`, `success`, `error`

### 3.3 Shopify Payments Payouts (REST)

**Endpoint:** `GET /admin/api/2026-01/shopify_payments/payouts.json`

**Requires:** `read_shopify_payments_payouts` scope (or legacy `shopify_payments` scope)

Source: [Shopify Payouts REST Reference](https://shopify.dev/docs/api/admin-rest/latest/resources/payouts)

```json
{
  "payouts": [
    {
      "id": 623721858,
      "status": "paid",
      "date": "2024-01-15",
      "currency": "USD",
      "amount": "1234.56"
    }
  ]
}
```

**Payout `status` values:**
- `scheduled` — created with transactions assigned, not yet submitted to bank
- `in_transit` — submitted to bank for processing
- `paid` — deposited into bank
- `failed` — declined by bank
- `canceled` — canceled by Shopify

### 3.4 Shopify Payments Balance Transactions (REST)

**Endpoint:** `GET /admin/api/2026-01/shopify_payments/balance/transactions.json`

Source: [Shopify Transactions REST Reference](https://shopify.dev/docs/api/admin-rest/latest/resources/transactions)

**Key Fields:**
| Field | Description |
|---|---|
| `id` | Unique transaction ID |
| `type` | `charge`, `refund`, `dispute`, `reserve`, `adjustment`, `credit`, `debit`, `payout`, `payout_failure`, `payout_cancellation` |
| `amount` | Gross transaction amount |
| `fee` | Total fees deducted |
| `net` | Net amount (amount - fee) |
| `currency` | ISO 4217 currency code |
| `payout_id` | ID of the payout this transaction was included in |
| `payout_status` | Status of the associated payout, or `pending` |
| `source_id` | ID of the originating resource |
| `source_type` | `charge`, `refund`, `dispute`, `reserve`, `adjustment`, `payout` |
| `source_order_transaction_id` | Links back to the order transaction that caused this balance transaction |
| `processed_at` | When the transaction was processed |

**Filtering by payout:**
```
GET /admin/api/2026-01/shopify_payments/balance/transactions.json?payout_id=623721858
```

**Reconciliation Key:** Use `source_order_transaction_id` to link a balance transaction back to a specific order transaction. Use `payout_id` to see all transactions included in a specific bank deposit.

### 3.5 Refunds

**Endpoint:** `GET /admin/api/2026-01/orders/{order_id}/refunds.json`

```json
{
  "refunds": [
    {
      "id": 929361461,
      "order_id": 450789469,
      "created_at": "2024-01-20T10:00:00-05:00",
      "note": "Customer returned item",
      "transactions": [
        {
          "kind": "refund",
          "status": "success",
          "amount": "41.94",
          "gateway": "shopify_payments"
        }
      ],
      "refund_line_items": [
        {
          "line_item_id": 866550311766439020,
          "quantity": 1,
          "restock_type": "return",
          "subtotal": "41.94",
          "total_tax": "5.03",
          "subtotal_set": {...},
          "total_tax_set": {...}
        }
      ],
      "order_adjustments": [
        {
          "id": 1234,
          "order_id": 450789469,
          "refund_id": 929361461,
          "reason": "Shipping adjustment",
          "amount": "-5.00",
          "tax_amount": "-0.00",
          "amount_set": {...},
          "tax_amount_set": {...}
        }
      ]
    }
  ]
}
```

### 3.6 Products and Variants

**Endpoint:** `GET /admin/api/2026-01/products.json`

Key accounting fields:
```json
{
  "id": 632910392,
  "title": "IPod Nano - 8GB",
  "vendor": "Apple",
  "product_type": "Cult Products",
  "status": "active",
  "variants": [
    {
      "id": 808950810,
      "sku": "IPOD2008PINK",
      "price": "199.00",
      "compare_at_price": "229.00",
      "cost": "150.00",  // Cost of goods (Shopify Plus)
      "taxable": true,
      "inventory_quantity": 10,
      "inventory_management": "shopify",
      "weight": 0.2,
      "weight_unit": "kg"
    }
  ]
}
```

### 3.7 Customers

**Endpoint:** `GET /admin/api/2026-01/customers.json`

Key accounting fields:
```json
{
  "id": 207119551,
  "email": "bob.norman@example.com",
  "orders_count": 1,
  "total_spent": "199.65",
  "currency": "USD",
  "tax_exempt": false,
  "tax_exemptions": [],
  "note": null,
  "tags": "VIP",
  "default_address": {...}
}
```

### 3.8 Shop Info

**Endpoint:** `GET /admin/api/2026-01/shop.json`

Critical for accounting: provides the store's base currency, timezone, and country.

```json
{
  "shop": {
    "id": 548380009,
    "name": "John Smith Test Store",
    "email": "j.smith@example.com",
    "domain": "shop.example.com",
    "myshopify_domain": "jsmith.myshopify.com",
    "currency": "USD",
    "timezone": "Eastern Time (US & Canada)",
    "iana_timezone": "America/New_York",
    "country_code": "US",
    "country_name": "United States",
    "county_taxes": true,
    "taxes_included": false,
    "tax_shipping": false,
    "money_format": "${{amount}}",
    "money_with_currency_format": "${{amount}} USD",
    "has_shopify_payments": true,
    "plan_name": "shopify",
    "checkout_api_supported": true
  }
}
```

### 3.9 Inventory Levels

**Endpoint:** `GET /admin/api/2026-01/inventory_levels.json?inventory_item_ids={ids}&location_ids={ids}`

### 3.10 Fulfillments / Shipping

**Endpoint:** `GET /admin/api/2026-01/orders/{order_id}/fulfillments.json`

Key accounting fields: tracking numbers, carrier, line items fulfilled, fulfillment service, created/updated timestamps.

### 3.11 Disputes (Chargebacks)

**Endpoint:** `GET /admin/api/2026-01/shopify_payments/disputes.json`

**Requires:** `read_shopify_payments_disputes` scope

```json
{
  "disputes": [
    {
      "id": 598735659,
      "order_id": 625362839,
      "type": "chargeback",
      "amount": "100.00",
      "currency": "USD",
      "reason": "fraudulent",
      "status": "won",
      "evidence_due_by": "2024-02-01",
      "evidence_sent_on": "2024-01-25",
      "final_reply_by": "2024-02-01",
      "initiated_at": "2024-01-10T10:00:00Z"
    }
  ]
}
```

---

## 4. Admin GraphQL API

### Overview

The GraphQL Admin API is Shopify's **current recommended API** (REST is legacy as of Oct 2024). All new public apps must use GraphQL.

**Base Endpoint:**
```
POST https://{shop}.myshopify.com/admin/api/2026-01/graphql.json
X-Shopify-Access-Token: {access_token}
Content-Type: application/json
```

Source: [GraphQL Admin API](https://shopify.dev/docs/api/admin-graphql), [Bulk Operations Guide](https://shopify.dev/docs/api/usage/bulk-operations/queries)

### Advantages Over REST

| Dimension | GraphQL | REST |
|---|---|---|
| Data fetching | Request exactly the fields you need | Fixed response shape |
| Network efficiency | Single request for nested data | Multiple round trips |
| Pagination | Cursor-based (stable, no skipping pages) | Offset-based (deprecated) or cursor |
| Bulk data | Native bulk operations for large exports | Manual pagination only |
| Real-time costs | Query cost estimation before execution | No cost estimation |
| Status | **Active** (all new apps must use this) | Legacy (deprecated Oct 2024) |

### Cursor-Based Pagination

```graphql
query GetOrders($cursor: String) {
  orders(first: 250, after: $cursor, query: "created_at:>=2024-01-01") {
    pageInfo {
      hasNextPage
      endCursor
    }
    edges {
      cursor
      node {
        id
        name
        createdAt
        displayFinancialStatus
        displayFulfillmentStatus
        totalPriceSet {
          shopMoney { amount currencyCode }
          presentmentMoney { amount currencyCode }
        }
        subtotalPriceSet {
          shopMoney { amount currencyCode }
        }
        totalTaxSet {
          shopMoney { amount currencyCode }
        }
        lineItems(first: 50) {
          edges {
            node {
              id
              name
              quantity
              sku
              originalUnitPriceSet {
                shopMoney { amount currencyCode }
              }
              taxLines {
                title
                rate
                priceSet {
                  shopMoney { amount currencyCode }
                }
              }
            }
          }
        }
        transactions(first: 10) {
          id
          kind
          status
          amountSet {
            shopMoney { amount currencyCode }
          }
          gateway
          processedAt
        }
      }
    }
  }
}
```

Pass `endCursor` as the `cursor` variable in each subsequent request.

### Bulk Operations for Large Data Exports

Source: [Shopify Bulk Operations Guide](https://shopify.dev/docs/api/usage/bulk-operations/queries)

Bulk operations are **the correct approach** for accounting data syncs — they are asynchronous, not subject to standard rate limits per query, and return JSONL files.

**As of API version 2026-01:** Up to 5 bulk query operations can run concurrently per shop.

**Step 1: Start the bulk operation**
```graphql
mutation {
  bulkOperationRunQuery(
    query: """
    {
      orders(query: "created_at:>=2024-01-01") {
        edges {
          node {
            id
            name
            createdAt
            totalPriceSet {
              shopMoney { amount currencyCode }
            }
            lineItems {
              edges {
                node {
                  id
                  name
                  quantity
                  originalUnitPriceSet {
                    shopMoney { amount currencyCode }
                  }
                  taxLines {
                    title rate
                    priceSet { shopMoney { amount currencyCode } }
                  }
                }
              }
            }
          }
        }
      }
    }
    """
  ) {
    bulkOperation {
      id
      status
    }
    userErrors {
      field
      message
    }
  }
}
```

**Step 2: Poll for completion (or use webhook)**

Poll with:
```graphql
query {
  bulkOperation(id: "gid://shopify/BulkOperation/720918") {
    id
    status        # CREATED | RUNNING | COMPLETED | FAILED | CANCELED
    errorCode
    objectCount
    fileSize
    url           # Download URL (available when COMPLETED)
    partialDataUrl
  }
}
```

Or subscribe to the `bulk_operations/finish` webhook topic.

**Step 3: Download JSONL file**

The `url` field points to a Google Cloud Storage URL. Download and parse line by line:
```jsonl
{"id":"gid://shopify/Order/1","name":"#1001","createdAt":"2024-01-15T10:00:00Z","totalPriceSet":{"shopMoney":{"amount":"100.00","currencyCode":"USD"}}}
{"id":"gid://shopify/LineItem/1","name":"Widget","quantity":1,"__parentId":"gid://shopify/Order/1"}
```

Note: Nested connection items include a `__parentId` field linking them to the parent object.

**Bulk Operation Constraints:**
- Maximum 5 total connections in a single query
- Connections must implement the `Node` interface
- Maximum 2 levels deep for nested connections
- Cannot use top-level `node` or `nodes` fields
- Operation must complete within 10 days
- The `first` argument is ignored in bulk operations

**Key GraphQL Queries for Accounting:**

```graphql
# Shopify Payments payouts (GraphQL)
query {
  shopifyPaymentsAccount {
    payouts(first: 50) {
      edges {
        node {
          id
          status
          issuedAt
          net { amount currencyCode }
          gross { amount currencyCode }
          fee { amount currencyCode }
          summary {
            chargesGross { amount currencyCode }
            refundsFeeGross { amount currencyCode }
            adjustmentsFeeAmount { amount currencyCode }
            chargesFee { amount currencyCode }
          }
        }
      }
    }
    balanceTransactions(first: 50) {
      edges {
        node {
          id
          type
          net { amount currencyCode }
          fee { amount currencyCode }
          amount { amount currencyCode }
          associatedPayout { id status issuedAt }
          sourceOrderTransaction { id order { id name } }
        }
      }
    }
  }
}
```

---

## 5. Shopify Payments / Balance API

Source: [Shopify Payouts REST Docs](https://shopify.dev/docs/api/admin-rest/latest/resources/payouts), [Shopify Dev Community](https://community.shopify.dev/t/shopify-payments-balancetransaction-vs-payouts/28897)

### Architecture

Shopify Payments has three main concepts:

| Concept | Description | API Resource |
|---|---|---|
| **Payout** | A bank deposit from Shopify to the merchant's account | `ShopifyPaymentsPayout` |
| **Balance Transaction** | A single financial event affecting the Shopify Payments balance | `ShopifyPaymentsBalanceTransaction` |
| **Dispute** | A chargeback filed by a customer | `ShopifyPaymentsDispute` |

### Payouts

- Each payout has a `status`: `scheduled`, `in_transit`, `paid`, `failed`, `canceled`
- Payout amount = sum of all balance transactions included in that payout
- Multiple orders from multiple days can be included in a single payout
- Payout period and timing depends on the merchant's payout schedule (daily by default for established merchants)

**Payout components (what's netted into the deposit):**
- **Charges (gross):** Sales + sales tax + gratuities + gift cards issued
- **Refunds:** Shown at gross (includes tax component)
- **Fees:** Shopify payment processing fees (plan-dependent: e.g., 2.9% + $0.30 on Basic)
- **Adjustments (positive):** Channel promotion credits, Shop Cash payments, shipping label adjustments
- **Adjustments (negative):** Chargebacks, Shop Cash campaign promotions, Shopify loan repayments, currency conversion fees
- **Net deposit** = Charges - Refunds - Fees + Adjustments

### Balance Transactions

Each balance transaction represents one event (charge, refund, dispute, fee, etc.) and links to:
- `source_order_transaction_id` → the order transaction that caused it
- `payout_id` → which bank deposit it was included in

**Balance Transaction Types:**
| Type | Description |
|---|---|
| `charge` | Customer payment captured |
| `refund` | Refund issued to customer |
| `dispute` | Chargeback deducted |
| `reserve` | Funds held by Shopify |
| `adjustment` | Manual adjustments (tax, channel credits, etc.) |
| `credit` | Credit from Shopify |
| `debit` | Debit to balance |
| `payout` | Actual payout transfer |
| `payout_failure` | Failed payout attempt |
| `payout_cancellation` | Canceled payout |

### Payout Reconciliation Logic

**The canonical reconciliation approach:**

1. Fetch all payouts: `GET /shopify_payments/payouts.json`
2. For each payout, fetch associated balance transactions: `GET /shopify_payments/balance/transactions.json?payout_id={id}`
3. Each balance transaction with `source_type=charge` or `source_type=refund` has a `source_order_transaction_id`
4. Use that ID to look up the corresponding order and order transaction
5. Reconcile: `sum(balance_transactions.net) = payout.amount`

**Important caveats:**
- One payout covers many orders across multiple business days
- Shopify records the sale at order time; the payout arrives 2–7 days later (timing difference)
- Fees are deducted at the balance transaction level, not the order level
- `shop_money` in order = base currency; `presentment_money` = customer-facing currency
- Shop Pay Installments are separate from standard Shopify Payments transactions
- The `adjustment_order_transactions` array on adjustment-type balance transactions links back to affected orders

### Disputes (Chargebacks)

**Endpoint:** `GET /admin/api/2026-01/shopify_payments/disputes.json`

**Dispute lifecycle:** `needs_response` → `under_review` → `won` / `lost` / `accepted`

**Dispute types:** `chargeback`, `inquiry`

**Dispute reasons:** `fraudulent`, `unrecognized`, `duplicate`, `subscription_cancelled`, `product_unacceptable`, `product_not_received`, `goods_services_returned_or_refused`, `credit_not_processed`, `general`

Webhooks: `disputes/create`, `disputes/update`

---

## 6. Webhooks

### Overview

Webhooks allow your app to receive real-time notifications when events occur in a merchant's store. They eliminate the need for polling.

Source: [Shopify Webhook HTTPS Docs](https://shopify.dev/docs/apps/build/webhooks/subscribe/https)

### Webhook Topics for Accounting Apps

**Orders & Transactions (critical for accounting):**
| Topic | Trigger |
|---|---|
| `orders/create` | New order placed |
| `orders/paid` | Order payment captured |
| `orders/updated` | Any order field updated |
| `orders/cancelled` | Order cancelled |
| `orders/fulfilled` | Order fulfilled |
| `orders/edited` | Order line items edited |
| `order_transactions/create` | New payment transaction on order |
| `refunds/create` | Refund created |

**Payments & Financial:**
| Topic | Trigger |
|---|---|
| `tender_transactions/create` | Tender transaction created |
| `disputes/create` | Chargeback/dispute opened |
| `disputes/update` | Dispute status changed |
| `payment_terms/create` | Payment terms created |
| `payment_terms/update` | Payment terms updated |
| `payment_schedules/due` | Payment schedule payment due |

**App & Compliance:**
| Topic | Trigger |
|---|---|
| `app/uninstalled` | App uninstalled from store |
| `app_subscriptions/update` | App subscription status changed |
| `bulk_operations/finish` | Bulk GraphQL operation completed |

**Mandatory GDPR Webhooks (required for App Store):**
| Topic | Trigger |
|---|---|
| `customers/data_request` | Customer requests their data |
| `customers/redact` | Customer requests data deletion |
| `shop/redact` | Shop requests data deletion (48h after uninstall) |

### Webhook Registration

**Via REST API:**
```http
POST /admin/api/2026-01/webhooks.json
{
  "webhook": {
    "topic": "orders/create",
    "address": "https://yourapp.com/webhooks/orders-create",
    "format": "json"
  }
}
```

**Via app config TOML (recommended for mandatory webhooks):**
```toml
[webhooks]
api_version = "2026-01"

[[webhooks.subscriptions]]
topics = ["orders/create", "orders/paid", "refunds/create", "disputes/create"]
uri = "https://yourapp.com/webhooks"

[[webhooks.subscriptions]]
compliance_topics = ["customers/data_request", "customers/redact", "shop/redact"]
uri = "https://yourapp.com/webhooks/compliance"
```

**Via GraphQL:**
```graphql
mutation {
  webhookSubscriptionCreate(
    topic: ORDERS_CREATE
    webhookSubscription: {
      format: JSON
      callbackUrl: "https://yourapp.com/webhooks/orders-create"
    }
  ) {
    webhookSubscription { id }
    userErrors { field message }
  }
}
```

### HMAC Verification

Every webhook request from Shopify includes an `X-Shopify-Hmac-Sha256` header containing a base64-encoded HMAC-SHA256 digest of the raw request body, signed with your app's client secret.

**Node.js implementation:**
```javascript
const crypto = require('crypto');

function verifyShopifyWebhook(rawBody, hmacHeader, clientSecret) {
  const computed = crypto
    .createHmac('sha256', clientSecret)
    .update(rawBody, 'utf8')
    .digest('base64');
  
  // CRITICAL: Use timing-safe comparison to prevent timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(hmacHeader),
    Buffer.from(computed)
  );
}

// Express example
app.post('/webhooks', express.raw({ type: 'application/json' }), (req, res) => {
  const hmac = req.get('X-Shopify-Hmac-Sha256');
  const isValid = verifyShopifyWebhook(req.body, hmac, process.env.SHOPIFY_CLIENT_SECRET);
  
  if (!isValid) return res.status(401).send('Unauthorized');
  
  // Acknowledge immediately, process async
  res.status(200).send('OK');
  processWebhookAsync(req.body, req.get('X-Shopify-Topic'));
});
```

**Key pitfalls:**
- Must use the **raw** request body (before JSON parsing)
- Body parser middleware (`express.json()`) will parse the body and break HMAC verification — use `express.raw()` on webhook routes
- After rotating the client secret, HMAC may take up to 1 hour to use the new secret

### Mandatory Webhooks for App Compliance

Apps submitted to the Shopify App Store **must** implement all 3 GDPR compliance webhooks:

1. `customers/data_request` — Merchant receives a customer data request, your app must supply stored customer data
2. `customers/redact` — Customer requests deletion; delete their data (delayed 10 days for recent-order customers)
3. `shop/redact` — Fired 48 hours after app uninstall; delete all shop data

**Configuration requirement:** Register these via the TOML `compliance_topics` field. They cannot be registered solely via the Admin API.

**Testing compliance webhooks:**
- `customers/data_request`: Trigger from Shopify Admin → Customers → [Customer] → "Request customer data"
- `customers/redact`: Admin → Customers → [Customer] → "Erase personal data" (wait 10+ days)
- `shop/redact`: Uninstall app from dev store, wait 48 hours

### Webhook Delivery & Retry Policy

Source: [Shopify Webhook HTTPS Docs](https://shopify.dev/docs/apps/build/webhooks/subscribe/https)

- Shopify expects a `2xx` response within **5 seconds**
- If no response or error returned: **retries 8 times over approximately 4 hours**
- After 8 consecutive failures: webhook subscription is automatically **deleted** if created via Admin API
- Shopify sends a notification to the Partner account emergency email before deletion
- Third-party sources suggest up to 19 failures trigger deletion and 48-hour retry window — implementations differ

**Best practices:**
1. Immediately return `200 OK`, process asynchronously via queue (BullMQ, SQS, RabbitMQ)
2. Use `X-Shopify-Webhook-Id` header as idempotency key to prevent duplicate processing
3. Track all webhook attempts in your database
4. Monitor endpoint error rates and alert at early failure thresholds

**Important webhook headers:**
| Header | Description |
|---|---|
| `X-Shopify-Topic` | The webhook topic (e.g., `orders/create`) |
| `X-Shopify-Hmac-Sha256` | Base64-encoded HMAC signature |
| `X-Shopify-Shop-Domain` | Shop's myshopify.com domain |
| `X-Shopify-Webhook-Id` | Unique ID for this delivery (use as idempotency key) |
| `X-Shopify-Event-Id` | Unique ID for the event itself |
| `X-Shopify-Api-Version` | API version used |

---

## 7. Billing API

Source: [Shopify Billing API Docs](https://shopify.dev/docs/apps/billing), [RecurringApplicationCharge REST](https://shopify.dev/docs/api/admin-rest/latest/resources/recurringapplicationcharge)

### Requirement

**Apps listed on the Shopify App Store must use the Shopify Billing API** for all charges. Off-platform billing (e.g., Stripe, PayPal charged outside Shopify) is not permitted. (App Store requirement 1.2.1)

### Billing Models

| Type | Description | GraphQL Mutation | REST Endpoint |
|---|---|---|---|
| **Recurring subscription (30-day)** | Fixed monthly charge | `appSubscriptionCreate` | `POST /recurring_application_charges.json` |
| **Recurring subscription (annual)** | Fixed yearly charge | `appSubscriptionCreate` | — |
| **Usage-based (capped)** | Variable charge up to a capped amount, billed monthly | `appSubscriptionCreate` + `appUsageRecordCreate` | `POST /recurring_application_charges.json` with `capped_amount` |
| **One-time purchase** | Single charge | `appPurchaseOneTimeCreate` | `POST /application_charges.json` |

### Subscription Flow

1. Merchant triggers plan selection in your app
2. Your app calls `appSubscriptionCreate` mutation
3. Shopify returns a `confirmationUrl`
4. Redirect merchant to `confirmationUrl` (Shopify-hosted consent page)
5. Merchant approves → redirected to your `returnUrl`
6. Merchant declines → redirected to Shopify admin with decline notification

**GraphQL: Create a recurring subscription**
```graphql
mutation appSubscriptionCreate($name: String!, $returnUrl: URL!, $test: Boolean) {
  appSubscriptionCreate(
    name: $name
    returnUrl: $returnUrl
    test: $test
    lineItems: [
      {
        plan: {
          appRecurringPricingDetails: {
            price: { amount: 29.99, currencyCode: USD }
            interval: EVERY_30_DAYS
          }
        }
      }
    ]
  ) {
    appSubscription {
      id
      status
    }
    confirmationUrl
    userErrors {
      field
      message
    }
  }
}
```

**REST: Create a recurring charge**
```
POST /admin/api/2026-01/recurring_application_charges.json
{
  "recurring_application_charge": {
    "name": "Pro Plan",
    "price": 29.99,
    "return_url": "https://yourapp.com/billing/return",
    "trial_days": 14,
    "test": false
  }
}
```

Response includes `confirmation_url` — redirect merchant to this URL.

**Maximum subscription price:** $10,000/month

### Usage-Based Billing (Capped)

Allows charging based on usage (e.g., per order synced), up to a monthly cap. The merchant must approve a maximum cap, and you create usage records as events occur.

```graphql
# Create subscription with usage component
mutation {
  appSubscriptionCreate(
    name: "Pay Per Sync Plan"
    returnUrl: "https://yourapp.com/billing"
    lineItems: [
      {
        plan: {
          appRecurringPricingDetails: {
            price: { amount: 10.00, currencyCode: USD }  # Base fee
            interval: EVERY_30_DAYS
          }
        }
      }
      {
        plan: {
          appUsagePricingDetails: {
            cappedAmount: { amount: 100.00, currencyCode: USD }
            terms: "$0.01 per order synced"
          }
        }
      }
    ]
  ) { ... }
}

# Record a usage event
mutation {
  appUsageRecordCreate(
    subscriptionLineItemId: "gid://shopify/AppSubscriptionLineItem/xxx"
    price: { amount: 0.01, currencyCode: USD }
    description: "1 order synced"
  ) {
    appUsageRecord { id }
    userErrors { field message }
  }
}
```

### Pricing Adjustments

| Feature | Description |
|---|---|
| **Trial days** | `trial_days` parameter — merchant not charged during trial |
| **Subscription discounts** | Percentage or fixed-price discount for N billing cycles |
| **App credits** | Grant credits to merchants toward future charges |
| **Refunds** | Full or partial refund of a specific charge |

### Billing Webhooks

- `APP_SUBSCRIPTIONS_UPDATE` — subscription status changed (active, declined, cancelled, frozen)
- `APP_PURCHASES_ONE_TIME_UPDATE` — one-time charge status changed
- `APP_SUBSCRIPTIONS_APPROACHING_CAPPED_AMOUNT` — usage crosses 90% of cap

---

## 8. Rate Limits

Source: [Shopify API Limits](https://shopify.dev/docs/api/usage/limits), [REST Rate Limits](https://shopify.dev/docs/api/admin-rest/usage/rate-limits)

### REST Admin API

**Method:** Leaky bucket algorithm (request-based)

| Plan | Bucket Size | Restore Rate |
|---|---|---|
| Standard | 40 requests | 2 requests/second |
| Advanced Shopify | 80 requests | 4 requests/second |
| Shopify Plus | 400 requests | 20 requests/second |
| Commerce Components | 800 requests | 40 requests/second |

**Rate limit response:**
- HTTP `429 Too Many Requests`
- `X-Shopify-Shop-Api-Call-Limit: 40/40` header shows current usage
- `Retry-After` header specifies seconds to wait

**Monitoring header:** `X-Shopify-Shop-Api-Call-Limit: {used}/{total}` — monitor this to throttle proactively.

### GraphQL Admin API

**Method:** Calculated query cost (point-based)

| Plan | Points/Second Limit |
|---|---|
| Standard | 100 points/second |
| Advanced | 200 points/second |
| Shopify Plus | 1,000 points/second |
| Commerce Components | 2,000 points/second |

**Single query hard limit:** 1,000 points (regardless of plan)

**Cost calculation:**
| Field type | Cost |
|---|---|
| Scalar | 0 |
| Enum | 0 |
| Object | 1 |
| Connection (with `first: N`) | N × (cost of items in connection) |
| Mutation | 10 (minimum) |

**Cost in response:**
```json
{
  "extensions": {
    "cost": {
      "requestedQueryCost": 232,
      "actualQueryCost": 32,
      "throttleStatus": {
        "maximumAvailable": 1000.0,
        "currentlyAvailable": 768.0,
        "restoreRate": 50.0
      }
    }
  }
}
```

Note: The bucket is refunded the difference between `requestedQueryCost` and `actualQueryCost` after execution.

### Handling 429 Responses

**REST:**
```javascript
async function shopifyFetch(url, options, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    const response = await fetch(url, options);
    
    if (response.status === 429) {
      const retryAfter = parseInt(response.headers.get('Retry-After') || '2');
      await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
      continue;
    }
    
    // Proactive throttle: slow down when bucket is > 75% full
    const limitHeader = response.headers.get('X-Shopify-Shop-Api-Call-Limit');
    if (limitHeader) {
      const [used, total] = limitHeader.split('/').map(Number);
      if (used / total > 0.75) {
        await new Promise(resolve => setTimeout(resolve, 500)); // 500ms delay
      }
    }
    
    return response;
  }
  throw new Error('Max retries exceeded');
}
```

**GraphQL:**
```javascript
// Check throttleStatus in response extensions
if (extensions?.cost?.throttleStatus?.currentlyAvailable < 200) {
  const delay = (1000 / extensions.cost.throttleStatus.restoreRate) * 200;
  await new Promise(resolve => setTimeout(resolve, delay));
}
```

**Best practices:**
- Use bulk operations for large data exports — bulk operation queries are not subject to per-request rate limits
- Implement exponential backoff with jitter for 429 responses
- Keep a client-side token bucket to proactively throttle before hitting server limits
- Never use `read_all_orders` + date-range-based polling for real-time sync — use webhooks instead

---

## 9. Shopify Tax API

Source: [Shopify Order REST Docs](https://shopify.dev/docs/api/admin-rest/latest/resources/order), [Shopify Developer Community on Tax with Markets](https://community.shopify.dev/t/line-item-tax-calculation-with-markets/5563)

### How Shopify Handles Taxes

Shopify calculates taxes at checkout based on:
- The merchant's tax settings (Settings → Taxes & Duties)
- The customer's billing/shipping address (for destination-based tax)
- The product's taxability (`taxable: true/false`)
- The jurisdiction (country, state/province, county)

Taxes are recorded in the order object at two levels:
1. **Order-level `tax_lines`** — aggregated taxes for the entire order
2. **Line-item-level `tax_lines`** — taxes per individual product line item
3. **Shipping-line-level `tax_lines`** — taxes on shipping costs

> **Note:** Tax lines specified on an order via API can be set at either the order level OR line item level, but not both simultaneously.

### Tax Line Object Structure

```json
{
  "title": "State tax",
  "price": "13.50",
  "price_set": {
    "shop_money": { "amount": "13.50", "currency_code": "USD" },
    "presentment_money": { "amount": "18.23", "currency_code": "CAD" }
  },
  "rate": 0.06,
  "channel_liable": false
}
```

| Field | Description |
|---|---|
| `title` | Tax name (e.g., "GST", "VAT", "State tax", "TAX") |
| `price` | Amount of tax in shop currency |
| `price_set` | Tax amount in both shop and presentment currencies |
| `rate` | Tax rate as decimal (e.g., 0.18 for 18%) |
| `channel_liable` | Whether the sales channel (e.g., marketplace) is responsible for remitting this tax |

### Key Tax Fields on Order Object

| Field | Description |
|---|---|
| `taxes_included` | `true` if prices include tax (VAT-inclusive pricing) |
| `total_tax` | Sum of all tax lines in shop currency |
| `total_tax_set` | Tax in shop + presentment currency |
| `current_total_tax` | Current tax total (adjusts after refunds/edits) |
| `estimated_taxes` | `true` if tax is estimated (e.g., pending address confirmation) |
| `tax_lines` | Array of order-level tax lines |

### India-Specific Tax Behavior (GST)

Shopify supports India's GST structure with these specific behaviors:

- **GST components:** India uses CGST + SGST (intra-state) or IGST (inter-state). Shopify will generate multiple `tax_lines` entries — one per component.
- **Tax title format:** Shopify typically shows `"CGST"`, `"SGST"`, or `"IGST"` in the `title` field for Indian stores.
- **HSN codes:** Not natively stored in Shopify orders. You would store these as product metafields and pull them when generating tax reports.
- **GST registration (GSTIN):** Shopify does not store GSTIN natively. You must add it as metafields or in order notes.
- **Taxes included:** Indian B2C prices are typically GST-inclusive (`taxes_included: true`). B2B may vary.
- **`channel_liable`:** For marketplace-facilitated sales (e.g., orders placed through Meta/Instagram), `channel_liable: true` means the marketplace remits the tax. Your accounting should not record this as a liability.

**Critical caveat:** Shopify's tax engine does not fully support India's GST compliance requirements (e-invoicing under IRP, e-waybill, HSN-level reporting). You will need to supplement with a GST compliance layer, likely via a third-party GST software integration or a custom module.

### Tax Line Ordering Warning

The ordering of tax lines in the API response array is **not guaranteed to be consistent** between orders. Always identify tax lines by their `title` field, not by array index position.

---

## 10. Shopify Markets (Multi-Currency)

Source: [Shopify Multi-Currency Docs](https://shopify.dev/docs/storefronts/themes/markets/multiple-currencies-languages), [Community: Tax with Markets](https://community.shopify.dev/t/line-item-tax-calculation-with-markets/5563)

### Overview

Shopify Markets enables merchants to sell in multiple currencies and languages from a single store. This significantly affects how order data is structured.

### Dual Currency Fields in Order Data

Every monetary value in orders appears in two forms:

```json
{
  "price_set": {
    "shop_money": {
      "amount": "792.02",
      "currency_code": "DKK"   // Merchant's base currency (for accounting)
    },
    "presentment_money": {
      "amount": "88.00",
      "currency_code": "GBP"   // Currency shown to and charged from customer
    }
  }
}
```

**For accounting purposes, always use `shop_money`** — this is the canonical value in the merchant's base currency, regardless of what currency the customer paid in.

### Fields Affected by Multi-Currency

All `_set` suffix fields have both `shop_money` and `presentment_money`:
- `subtotal_price_set`
- `total_price_set`
- `total_tax_set`
- `total_shipping_price_set`
- `total_discounts_set`
- `line_items[].price_set`
- `line_items[].tax_lines[].price_set`
- `shipping_lines[].price_set`
- `shipping_lines[].tax_lines[].price_set`

### Multi-Currency Caveats for Accounting

1. **Tax calculation discrepancies with dynamic exchange rates:** When Markets uses dynamic exchange rates with rounding and price adjustments, tax amounts in `presentment_money` may not precisely match what you'd calculate by applying the rate to the item price. This is expected behavior from Shopify's conversion fee (typically 2%) and rounding settings. **Always use `shop_money` for accounting.**

2. **Manual payment methods don't preserve presentment currency:** If a customer uses a manual payment method (e.g., bank transfer, invoice), Shopify reverts the order to the shop's base currency. `presentment_currency` will equal `shop_currency`. This is a platform limitation.

3. **Multi-currency works only with Shopify Payments:** For presentment in a non-base currency to be preserved, the merchant must use Shopify Payments. Third-party payment gateways force conversion back to base currency.

4. **Currency conversion fees in payouts:** Shopify charges a 1.5% (US) or 2% (other) currency conversion fee, which appears as a deduction in balance transactions.

### How to Get Current Exchange Rate

Shopify does not expose the exchange rate used in an order directly. To derive it:
```
exchange_rate = price_set.shop_money.amount / price_set.presentment_money.amount
```

Apply this consistently across all line items in the same order to get the rate used for that order.

### Markets API Access

```graphql
query {
  markets(first: 10) {
    edges {
      node {
        id
        name
        enabled
        primary
        regions {
          edges {
            node {
              ... on MarketRegionCountry {
                code
                name
              }
            }
          }
        }
        currencySettings {
          baseCurrency { currencyCode }
        }
      }
    }
  }
}
```

Requires `read_markets` scope.

---

## 11. App Bridge & Embedded Apps

Source: [About App Bridge](https://shopify.dev/docs/api/app-bridge), [App Store Requirements](https://shopify.dev/docs/apps/launch/shopify-app-store/app-store-requirements)

### What is App Bridge?

App Bridge is Shopify's JavaScript library that allows apps to:
- Render UI within the Shopify Admin iframe (embedded experience)
- Communicate between the app and Shopify Admin
- Obtain session tokens for authentication (Token Exchange flow)
- Use Shopify's navigation, modals, toasts, and resource pickers natively

**Requirement:** As of March 13, 2024, all apps submitted to the Shopify App Store must use the **latest version of App Bridge** (`app-bridge.js` script tag before any other scripts).

```html
<!-- Must be the FIRST script tag -->
<script src="https://cdn.shopify.com/shopifycloud/app-bridge.js"></script>
```

### Polaris Design System

[Polaris](https://polaris.shopify.com/) is Shopify's official React-based component library and design system. While technically optional, it is effectively required in practice:

- Apps that use Polaris consistently pass design review
- Apps that use custom design systems face much tougher scrutiny
- The "Built for Shopify" badge (which increases App Store discoverability) essentially requires Polaris compliance
- The design review is intertwined with Polaris/App Bridge requirements

**Core Polaris components for an accounting app:**
- `DataTable` — for displaying order/transaction data
- `ResourceList` — for listing payouts, customers
- `Filters` + `DatePicker` — date range selection for reports
- `Card` — for metric summaries
- `Page` + `Layout` — page structure
- `Banner` — for alerts/errors
- `Modal` — for confirmations
- `Badge` — for status indicators (order status, payout status)

### Embedded App Setup (Remix example)

```javascript
// app/shopify.server.js
import { shopifyApp } from "@shopify/shopify-app-remix/server";

const shopify = shopifyApp({
  apiKey: process.env.SHOPIFY_API_KEY,
  apiSecretKey: process.env.SHOPIFY_API_SECRET,
  appUrl: process.env.SHOPIFY_APP_URL,
  scopes: [
    "read_orders",
    "read_all_orders",
    "read_products",
    "read_customers",
    "read_shopify_payments_payouts",
    "read_shopify_payments_disputes",
    "read_inventory",
    "read_locations",
    "read_markets",
    "read_returns",
    "read_reports"
  ],
  authPathPrefix: "/auth",
});

export default shopify;
```

### Session Tokens

Session tokens are short-lived JWTs issued by Shopify to embedded apps via App Bridge. They:
- Are available only in embedded contexts
- Contain the shop domain, user ID, and expiry
- Must be exchanged for API access tokens (Token Exchange flow) before making Admin API calls
- Should NOT be cached long-term

---

## 12. Data Privacy & GDPR Webhooks

Source: [Shopify Privacy Law Compliance](https://shopify.dev/docs/apps/build/compliance/privacy-law-compliance)

### Overview

Shopify requires all apps distributed through the Shopify App Store to implement mandatory compliance webhooks, regardless of whether the app stores customer data. This ensures merchants can fulfill customer data subject requests.

### The Three Mandatory Webhooks

| Topic | When Triggered | Your Obligation |
|---|---|---|
| `customers/data_request` | Customer requests to see their stored data | Provide resource IDs of stored data to the merchant |
| `customers/redact` | Customer requests data deletion | Delete/anonymize all stored customer data (delayed 10 days if recent orders exist) |
| `shop/redact` | 48 hours after app uninstall | Delete all store-related data from your systems |

### Implementation Requirements

1. Register endpoints via TOML `compliance_topics` (cannot use Admin API for mandatory webhooks)
2. Endpoints must verify HMAC signatures (return `401` for invalid, `200` for valid)
3. Endpoints must be accessible over HTTPS with valid SSL certificates
4. Must respond within 5 seconds

### Payload Examples

**customers/data_request:**
```json
{
  "shop_id": 954889,
  "shop_domain": "jsmith.myshopify.com",
  "customer": {
    "id": 191167,
    "email": "john@example.com",
    "phone": "+15555555555"
  },
  "orders_requested": [
    299938,
    280263,
    220458
  ]
}
```

**customers/redact:**
```json
{
  "shop_id": 954889,
  "shop_domain": "jsmith.myshopify.com",
  "customer": {
    "id": 191167,
    "email": "john@example.com",
    "phone": "+15555555555"
  },
  "orders_to_redact": [
    299938,
    280263
  ]
}
```

**shop/redact:**
```json
{
  "shop_id": 954889,
  "shop_domain": "jsmith.myshopify.com"
}
```

### TOML Configuration

```toml
[webhooks]
api_version = "2026-01"

[[webhooks.subscriptions]]
compliance_topics = ["customers/data_request", "customers/redact", "shop/redact"]
uri = "https://yourapp.com/webhooks/compliance"
```

### Compliance Data Handling for an Accounting App

As an accounting app, you likely store:
- Order data (IDs, amounts, line items)
- Customer references (IDs, emails for reconciliation)
- Payout records
- Transaction records

**For `customers/redact`:**
- Anonymize customer email/name fields in your stored order records
- Keep order amounts and transaction IDs (needed for audit trail) but remove PII
- Document your data retention policy

**For `shop/redact`:**
- Delete or anonymize all stored order, payout, customer, and configuration data for the shop
- Archive anonymized data if required for your own accounting/audit purposes

---

## Appendix: Recommended Scopes for a Shopify Accounting App

```toml
# shopify.app.toml
scopes = [
  "read_orders",
  "read_all_orders",          # Requires Shopify approval
  "read_products",
  "read_customers",
  "read_inventory",
  "read_locations",
  "read_shipping",
  "read_shopify_payments_payouts",
  "read_shopify_payments_disputes",
  "read_returns",
  "read_draft_orders",
  "read_markets",
  "read_reports"
]
```

## Appendix: API Version Strategy

| API Version | Status | Notes |
|---|---|---|
| `2026-01` | Latest stable | Supports 5 concurrent bulk operations |
| `2025-10` | Supported | — |
| `2025-07` | Supported | — |
| `2024-01` | Supported | REST still functional |

**Recommendation:** Build against `2026-01`. Shopify releases a new stable API version quarterly. All supported versions are maintained for a minimum of 12 months.

## Appendix: Key Reconciliation Concepts

```
ORDER FLOW (for accounting):
1. order/create → Record revenue in accrual
2. order/paid → Order transaction captured; balance transaction created in Shopify Payments
3. balance_transaction (type=charge) → Linked to order via source_order_transaction_id
4. balance_transaction (type=refund) → If customer refunded
5. balance_transaction (type=dispute) → If chargeback opened
6. payout → Bank deposit; contains aggregated balance transactions
   Payout amount = Σ(balance_transactions.net) where payout_id = payout.id

RECONCILIATION CHECK:
sum(balance_transactions WHERE payout_id = X AND status = 'paid') = payout.amount

MULTI-CURRENCY:
Always record in shop_money currency
Exchange rate = shop_money.amount / presentment_money.amount

TAX:
taxes_included=false: tax is additional (price + tax = total)
taxes_included=true: tax is embedded (price already includes tax)
channel_liable=true: marketplace remits tax; do NOT record as your liability
```

---

*Sources: [Shopify Dev Docs](https://shopify.dev/docs), [Shopify API Reference](https://shopify.dev/docs/api), [Shopify Admin REST API](https://shopify.dev/docs/api/admin-rest), [Shopify Access Scopes](https://shopify.dev/docs/api/usage/access-scopes), [Shopify Bulk Operations](https://shopify.dev/docs/api/usage/bulk-operations/queries), [Shopify Privacy Compliance](https://shopify.dev/docs/apps/build/compliance/privacy-law-compliance), [Shopify App Store Requirements](https://shopify.dev/docs/apps/launch/shopify-app-store/app-store-requirements), [Shopify Billing API](https://shopify.dev/docs/apps/billing), [Shopify API Rate Limits](https://shopify.dev/docs/api/usage/limits)*
