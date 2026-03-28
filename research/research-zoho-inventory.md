# Zoho Inventory API — Comprehensive Technical Reference
**For integration into a Shopify accounting platform**
**Researched:** March 28, 2026 | Source: [Zoho Inventory API Docs](https://www.zoho.com/inventory/api/v1/)

---

## Table of Contents
1. [Authentication (OAuth 2.0)](#1-authentication-oauth-20)
2. [Core API Endpoints](#2-core-api-endpoints)
   - 2.1 Items (Products)
   - 2.2 Composite Items
   - 2.3 Item Groups
   - 2.4 Inventory Adjustments
   - 2.5 Sales Orders
   - 2.6 Purchase Orders
   - 2.7 Invoices
   - 2.8 Packages
   - 2.9 Shipment Orders
   - 2.10 Warehouses / Locations (Multi-Warehouse)
   - 2.11 Transfer Orders (Stock Transfers)
   - 2.12 Price Lists (Pricebooks)
3. [Webhooks / Notifications](#3-webhooks--notifications)
4. [Rate Limits](#4-rate-limits)
5. [Data Models](#5-data-models)
   - 5.1 Items — Full Field Reference
   - 5.2 Sales Orders — Full Field Reference
   - 5.3 Purchase Orders — Full Field Reference
   - 5.4 Invoices — Full Field Reference
6. [India-Specific Fields (GST / HSN)](#6-india-specific-fields-gst--hsn)
7. [Zoho Inventory ↔ Zoho Books Sync](#7-zoho-inventory--zoho-books-sync)
8. [Pagination](#8-pagination)
9. [Error Codes and Retry Strategies](#9-error-codes-and-retry-strategies)

---

## 1. Authentication (OAuth 2.0)

**Source:** [Zoho Inventory OAuth Docs](https://www.zoho.com/inventory/api/v1/oauth/) | [Zoho OAuth Protocol](https://www.zoho.com/accounts/protocol/oauth.html) | [Server-Based Apps](https://www.zoho.com/accounts/protocol/oauth/web-server-applications.html)

### 1.1 Overview

Zoho uses **OAuth 2.0**. All tokens must be passed in the `Authorization` header:

```
Authorization: Zoho-oauthtoken {access_token}
```
or
```
Authorization: Bearer {access_token}
```

Access tokens expire after **1 hour**. Refresh tokens are permanent (up to 20 per user per app).

---

### 1.2 Data Center / Domain Mapping

**Critical:** Token endpoint domain must match the user's data center.

| Data Center | App Domain | Auth Base URL | API Base URL |
|---|---|---|---|
| United States | `.com` | `https://accounts.zoho.com/` | `https://www.zohoapis.com/inventory/` |
| India | `.in` | `https://accounts.zoho.in/` | `https://www.zohoapis.in/inventory/` |
| Europe | `.eu` | `https://accounts.zoho.eu/` | `https://www.zohoapis.eu/inventory/` |
| Australia | `.com.au` | `https://accounts.zoho.com.au/` | `https://www.zohoapis.com.au/inventory/` |
| Canada | `.ca` | `https://accounts.zohocloud.ca/` | `https://www.zohoapis.ca/inventory/` |
| Japan | `.jp` | `https://accounts.zoho.jp/` | `https://www.zohoapis.jp/inventory/` |
| China | `.com.cn` | (China-specific) | `https://www.zohoapis.com.cn/inventory/` |
| Saudi Arabia | `.sa` | (SA-specific) | `https://www.zohoapis.sa/inventory/` |

> **For India:** Use `accounts.zoho.in` for all auth calls and `www.zohoapis.in` for all API calls.

---

### 1.3 App Registration

Register at: `https://accounts.zoho.com/developerconsole`

Two client types:
- **Server-based app** (web app with backend — confidential client, supports authorization code flow + refresh tokens)
- **Self-client** (for scripts/testing — generates grant token directly in console without redirect URI, used for server-to-server integration)

---

### 1.4 Authorization Code Flow (Server-Based App)

**Step 1 — Authorization URL (GET)**

```
https://accounts.zoho.com/oauth/v2/auth
  ?scope=ZohoInventory.FullAccess.all
  &client_id=1000.0SRSxxxxxxxxxxxxxxxxxxxx239V
  &state=testing
  &response_type=code
  &redirect_uri=https://yourapp.com/callback
  &access_type=offline
  &prompt=Consent
```

| Parameter | Required | Description |
|---|---|---|
| `scope` | Yes | Space or comma-separated scopes |
| `client_id` | Yes | From developer console |
| `response_type` | Yes | Always `code` |
| `redirect_uri` | Yes | Must match registered URI |
| `access_type` | No | `offline` = get refresh token; `online` = access token only |
| `state` | No | CSRF protection opaque string |
| `prompt` | No | `Consent` = always show consent screen |

The authorization code returned is valid for **60 seconds** only.

---

**Step 2 — Exchange Code for Tokens (POST)**

```
POST https://accounts.zoho.com/oauth/v2/token
Content-Type: application/x-www-form-urlencoded

code=1000.dd7eXXX...XXX9bb8
&client_id=1000.0SRS...239V
&client_secret=fb01XXX...8abf
&redirect_uri=https://yourapp.com/callback
&grant_type=authorization_code
```

**Response:**
```json
{
  "access_token": "1000.41d9XXX...XXXc2d1",
  "refresh_token": "1000.8ecdXXX...XXX5cb7",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

---

**Step 3 — Refresh the Access Token (POST)**

```
POST https://accounts.zoho.com/oauth/v2/token
Content-Type: application/x-www-form-urlencoded

refresh_token=1000.8ecdXXX...XXX5cb7
&client_id=1000.0SRS...239V
&client_secret=fb01XXX...8abf
&redirect_uri=https://yourapp.com/callback
&grant_type=refresh_token
```

**Response:**
```json
{
  "access_token": "1000.newTokenHere...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

---

**Step 4 — Revoke a Token (POST)**

```
POST https://accounts.zoho.com/oauth/v2/token/revoke
  ?token=1000.8ecdXXX...XXXebdc
```

---

### 1.5 Self-Client Flow (Server-to-Server / Script Use)

1. Go to [Zoho API Console](https://api-console.zoho.com/) → Self Client
2. Generate a grant token directly with specified scopes (valid for 3–10 minutes)
3. Exchange for access + refresh token via the same `oauth/v2/token` endpoint using `grant_type=authorization_code`
4. This refresh token is permanent — use it to mint new access tokens unattended

---

### 1.6 Complete Scope Reference

| Scope | Description |
|---|---|
| `ZohoInventory.FullAccess.all` | Full access to all modules |
| `ZohoInventory.items.CREATE` | Create items |
| `ZohoInventory.items.UPDATE` | Update items |
| `ZohoInventory.items.READ` | Read items |
| `ZohoInventory.items.DELETE` | Delete items |
| `ZohoInventory.compositeitems.CREATE` | Create composite items |
| `ZohoInventory.compositeitems.UPDATE` | Update composite items |
| `ZohoInventory.compositeitems.READ` | Read composite items |
| `ZohoInventory.compositeitems.DELETE` | Delete composite items |
| `ZohoInventory.inventoryadjustments.CREATE` | Create inventory adjustments |
| `ZohoInventory.inventoryadjustments.READ` | Read inventory adjustments |
| `ZohoInventory.inventoryadjustments.UPDATE` | Update inventory adjustments |
| `ZohoInventory.inventoryadjustments.DELETE` | Delete inventory adjustments |
| `ZohoInventory.transferorders.CREATE` | Create transfer orders |
| `ZohoInventory.transferorders.READ` | Read transfer orders |
| `ZohoInventory.transferorders.UPDATE` | Update transfer orders |
| `ZohoInventory.transferorders.DELETE` | Delete transfer orders |
| `ZohoInventory.settings.CREATE` | Create settings (taxes, currencies, users) |
| `ZohoInventory.settings.UPDATE` | Update settings |
| `ZohoInventory.settings.READ` | Read settings + get organization_id |
| `ZohoInventory.settings.DELETE` | Delete settings |
| `ZohoInventory.salesorders.CREATE` | Create sales orders |
| `ZohoInventory.salesorders.UPDATE` | Update sales orders |
| `ZohoInventory.salesorders.READ` | Read sales orders |
| `ZohoInventory.salesorders.DELETE` | Delete sales orders |
| `ZohoInventory.packages.CREATE` | Create packages |
| `ZohoInventory.packages.UPDATE` | Update packages |
| `ZohoInventory.packages.READ` | Read packages |
| `ZohoInventory.packages.DELETE` | Delete packages |
| `ZohoInventory.shipmentorders.CREATE` | Create shipment orders |
| `ZohoInventory.shipmentorders.UPDATE` | Update shipment orders |
| `ZohoInventory.shipmentorders.READ` | Read shipment orders |
| `ZohoInventory.shipmentorders.DELETE` | Delete shipment orders |
| `ZohoInventory.invoices.CREATE` | Create invoices |
| `ZohoInventory.invoices.UPDATE` | Update invoices |
| `ZohoInventory.invoices.READ` | Read invoices |
| `ZohoInventory.invoices.DELETE` | Delete invoices |
| `ZohoInventory.customerpayments.CREATE` | Create customer payments |
| `ZohoInventory.customerpayments.UPDATE` | Update customer payments |
| `ZohoInventory.customerpayments.READ` | Read customer payments |
| `ZohoInventory.customerpayments.DELETE` | Delete customer payments |
| `ZohoInventory.salesreturns.CREATE` | Create sales returns |
| `ZohoInventory.salesreturns.UPDATE` | Update sales returns |
| `ZohoInventory.salesreturns.READ` | Read sales returns |
| `ZohoInventory.salesreturns.DELETE` | Delete sales returns |
| `ZohoInventory.creditnotes.CREATE` | Create credit notes |
| `ZohoInventory.creditnotes.UPDATE` | Update credit notes |
| `ZohoInventory.creditnotes.READ` | Read credit notes |
| `ZohoInventory.creditnotes.DELETE` | Delete credit notes |
| `ZohoInventory.purchaseorders.CREATE` | Create purchase orders |
| `ZohoInventory.purchaseorders.UPDATE` | Update purchase orders |
| `ZohoInventory.purchaseorders.READ` | Read purchase orders |
| `ZohoInventory.purchaseorders.DELETE` | Delete purchase orders |
| `ZohoInventory.purchasereceives.CREATE` | Create purchase receives |
| `ZohoInventory.purchasereceives.READ` | Read purchase receives |
| `ZohoInventory.purchasereceives.UPDATE` | Update purchase receives |
| `ZohoInventory.purchasereceives.DELETE` | Delete purchase receives |
| `ZohoInventory.bills.CREATE` | Create bills |
| `ZohoInventory.bills.UPDATE` | Update bills |
| `ZohoInventory.bills.READ` | Read bills |
| `ZohoInventory.bills.DELETE` | Delete bills |
| `ZohoInventory.contacts.CREATE` | Create contacts |
| `ZohoInventory.contacts.UPDATE` | Update contacts |
| `ZohoInventory.contacts.READ` | Read contacts |
| `ZohoInventory.contacts.DELETE` | Delete contacts |

---

### 1.7 Organization ID

Every API request requires the `organization_id` query parameter.

```
GET https://www.zohoapis.in/inventory/v1/items?organization_id=10234695
```

To retrieve your organization ID:

```
GET https://www.zohoapis.in/inventory/v1/organizations
Authorization: Zoho-oauthtoken {access_token}
Scope: ZohoInventory.settings.READ
```

---

## 2. Core API Endpoints

**Base URL (India):** `https://www.zohoapis.in/inventory/v1/`
**Base URL (US):** `https://www.zohoapis.com/inventory/v1/`

All requests require `?organization_id={id}` as a query parameter.

---

### 2.1 Items (Products)

**Source:** [Items API](https://www.zoho.com/inventory/api/v1/items/)

| Operation | Method | Endpoint |
|---|---|---|
| Create item | POST | `/items` |
| List all items | GET | `/items` |
| Bulk fetch item details | GET | `/itemdetails` |
| Retrieve item | GET | `/items/{item_id}` |
| Update item | PUT | `/items/{item_id}` |
| Delete item | DELETE | `/items/{item_id}` |
| Update custom field | PUT | `/item/{item_id}/customfields` |
| Delete item image | DELETE | `/items/{item_id}/image` |
| Mark as active | POST | `/items/{item_id}/active` |
| Mark as inactive | POST | `/items/{item_id}/inactive` |

**Create Item — Required fields:** `name`

**Create Item — Full Request Body:**
```json
{
  "name": "Bags-small",
  "group_id": 4815000000044220,
  "group_name": "Bags",
  "unit": "qty",
  "item_type": "inventory",
  "product_type": "goods",
  "is_taxable": true,
  "tax_id": 4815000000044043,
  "description": "description",
  "purchase_account_id": 4815000000035003,
  "inventory_account_id": 4815000000035001,
  "attribute_name1": "Small",
  "rate": 6.0,
  "purchase_rate": 6.0,
  "reorder_level": 5,
  "locations": [
    {
      "location_id": "460000000038080",
      "initial_stock": 50,
      "initial_stock_rate": 500
    }
  ],
  "vendor_id": 4815000000044080,
  "sku": "SK123",
  "upc": 111111111111,
  "ean": 111111111112,
  "isbn": "111111111113",
  "part_number": "111111111114",
  "attribute_option_name1": "Small",
  "purchase_description": "Purchase description",
  "hsn_or_sac": 85423100,
  "item_tax_preferences": [
    {
      "tax_id": 4815000000044043,
      "tax_specification": "intra"
    }
  ],
  "custom_fields": [
    {
      "customfield_id": "46000000012845",
      "value": "Normal"
    }
  ]
}
```

**`item_type` allowed values:** `inventory`, `sales`, `purchases`, `sales_and_purchases`
- If item belongs to a group → must be `inventory`

**`product_type` allowed values:** `goods`, `service`

**List Items — Query Parameters:**
- `page` (int, default: 1)
- `per_page` (int, default: 200)
- `organization_id` (required)

**Bulk Fetch — Query Parameters:**
- `item_ids` (string, comma-separated): e.g. `item_ids=4815000000044208,4815000000044274`
- `organization_id` (required)

---

### 2.2 Composite Items

**Source:** [Composite Items API](https://www.zoho.com/inventory/api/v1/compositeitems/)

| Operation | Method | Endpoint |
|---|---|---|
| Create composite item | POST | `/compositeitems` |
| List all composite items | GET | `/compositeitems` |
| Update composite item | PUT | `/compositeitems/{composite_item_id}` |
| Retrieve composite item | GET | `/compositeitems/{composite_item_id}` |
| Delete composite item | DELETE | `/compositeitems/{composite_item_id}` |
| Mark as active | POST | `/compositeitems/{composite_item_id}/active` |
| Mark as inactive | POST | `/compositeitems/{composite_item_id}/inactive` |
| Create assembly (bundle) | POST | `/bundles` |
| List assembly history | GET | `/bundles` |
| Retrieve assembly | GET | `/bundles/{bundle_id}` |
| Delete assembly | DELETE | `/bundles/{bundle_id}` |

**Required fields for create:** `name`, `mapped_items` (with `item_id`, `quantity`), `sku`, `item_type` (always `inventory`), `rate`, `product_type`

**`mapped_items` sub-fields:**
- `item_id` (long, required) — constituent item ID
- `quantity` (double) — qty used in bundle
- `line_item_id` (long) — set when updating

**India-only fields on composite items:**
- `hsn_or_sac` (string) — HSN/SAC code
- `item_tax_preferences[].tax_specification` — `intra` or `inter`

**Attach image separately:** `POST /compositeitems/{composite_item_id}/image` with `form-data` parameter `image`

---

### 2.3 Item Groups

**Source:** [Item Groups API](https://www.zoho.com/inventory/api/v1/itemgroups/)

| Operation | Method | Endpoint |
|---|---|---|
| Create item group | POST | `/itemgroups` |
| List all item groups | GET | `/itemgroups` |
| Update item group | PUT | `/itemgroups/{itemgroup_id}` |
| Retrieve item group | GET | `/itemgroups/{itemgroup_id}` |
| Delete item group | DELETE | `/itemgroups/{itemgroup_id}` |
| Mark as active | POST | `/itemgroups/{itemgroup_id}/active` |
| Mark as inactive | POST | `/itemgroups/{itemgroup_id}/inactive` |

**Required fields:** `group_name`, `unit`

**Optional fields:** `brand`, `manufacturer`, `description`, `tax_id`, `attribute_name1`, `items[]`, `attributes[]`

**Items sub-fields:** `name`, `rate`, `purchase_rate`, `sku`, `reorder_level`, `initial_stock`, `initial_stock_rate`, `vendor_id`, `upc`, `ean`, `isbn`, `part_number`, `attribute_option_name1`

**Scope:** `ZohoInventory.items.CREATE` / `.READ` / `.UPDATE` / `.DELETE`

---

### 2.4 Inventory Adjustments

**Note:** The `/inventoryadjustments` endpoint returned 404 during research, suggesting it may use a different path or require a specific plan. Use scope `ZohoInventory.inventoryadjustments.*` for this module.

Based on scope documentation, the endpoint pattern is:
- `POST /inventoryadjustments` — Create adjustment
- `GET /inventoryadjustments` — List adjustments
- `GET /inventoryadjustments/{adjustment_id}` — Retrieve
- `PUT /inventoryadjustments/{adjustment_id}` — Update
- `DELETE /inventoryadjustments/{adjustment_id}` — Delete

Inventory adjustments allow manually increasing or decreasing stock levels for items. They record quantity changes with a reason code.

---

### 2.5 Sales Orders

**Source:** [Sales Orders API](https://www.zoho.com/inventory/api/v1/salesorders/)

| Operation | Method | Endpoint |
|---|---|---|
| Create sales order | POST | `/salesorders` |
| List all sales orders | GET | `/salesorders` |
| Update sales order | PUT | `/salesorders/{salesorder_id}` |
| Retrieve sales order | GET | `/salesorders/{salesorder_id}` |
| Delete sales order | DELETE | `/salesorders/{salesorder_id}` |
| Bulk delete | DELETE | `/salesorders` |
| Mark as Confirmed | POST | `/salesorders/{salesorder_id}/status/confirmed` |
| Mark as Void | POST | `/salesorders/{salesorder_id}/status/void` |
| Bulk confirm | POST | `/salesorders/status/confirmed` |

**Create Sales Order — Required fields:** `customer_id`, `salesorder_number`, `line_items[]`

**Query Parameters:**
- `ignore_auto_number_generation` (boolean) — if `true`, auto-numbering is skipped and `salesorder_number` becomes mandatory

**Minimal Create Request Body:**
```json
{
  "customer_id": 4815000000044080,
  "salesorder_number": "SO-00003",
  "date": "2015-05-28",
  "shipment_date": "2015-06-02",
  "line_items": [
    {
      "item_id": 4815000000044100,
      "name": "Laptop",
      "rate": 122.00,
      "quantity": 2,
      "unit": "qty",
      "tax_id": 4815000000044043,
      "hsn_or_sac": "80540"
    }
  ],
  "place_of_supply": "TN",
  "gst_treatment": "business_gst",
  "gst_no": "22AAAAA0000A1Z5"
}
```

**Status values:** `draft`, `confirmed`, `fulfilled`, `void`

**India-specific fields on Sales Orders:**
- `gst_no` — 15-digit GSTIN of the customer
- `gst_treatment` — `business_gst`, `business_none`, `overseas`, `consumer`
- `place_of_supply` — 2-char state code (e.g. `TN`, `MH`, `KA`)
- `is_pre_gst` — boolean, for pre-July 2017 transactions
- `hsn_or_sac` per line item

---

### 2.6 Purchase Orders

**Source:** [Purchase Orders API](https://www.zoho.com/inventory/api/v1/purchaseorders/)

| Operation | Method | Endpoint |
|---|---|---|
| Create purchase order | POST | `/purchaseorders` |
| List all purchase orders | GET | `/purchaseorders` |
| Update purchase order | PUT | `/purchaseorders/{purchaseorder_id}` |
| Retrieve purchase order | GET | `/purchaseorders/{purchaseorder_id}` |
| Delete purchase order | DELETE | `/purchaseorders/{purchaseorder_id}` |
| Mark as Issued | POST | `/purchaseorders/{purchaseorder_id}/status/issued` |
| Mark as Cancelled | POST | `/purchaseorders/{purchaseorder_id}/status/cancelled` |

**Required fields:** `purchaseorder_number`, `vendor_id`, `line_items[]`

**India-specific fields on Purchase Orders:**
- `gst_no` — 15-digit GSTIN of the vendor
- `gst_treatment` — `business_gst`, `business_none`, `overseas`, `consumer`
- `source_of_supply` — state code for origin of goods
- `destination_of_supply` — state code for delivery destination
- `is_pre_gst` — boolean
- `is_reverse_charge_applied` — boolean, for reverse charge transactions
- Per line item: `reverse_charge_tax_id`, `reverse_charge_tax_name`, `reverse_charge_tax_percentage`, `reverse_charge_tax_amount`, `hsn_or_sac`, `tax_exemption_code`, `tax_exemption_id`

**Additional features:**
- `is_drop_shipment` (boolean) — for drop-ship orders, link to `salesorder_id`
- `is_backorder` (boolean) — for backorders

**Status values:** `draft`, `issued`, `Partially_Received`, `received`, `cancelled`

**`purchasereceives` sub-array in response** — lists all goods receipts tied to the PO.
**`bills` sub-array in response** — lists all bills generated from the PO.

---

### 2.7 Invoices

**Source:** [Invoices API](https://www.zoho.com/inventory/api/v1/invoices/)

| Operation | Method | Endpoint |
|---|---|---|
| Create invoice | POST | `/invoices` |
| List invoices | GET | `/invoices` |
| Update invoice | PUT | `/invoices/{invoice_id}` |
| Get invoice | GET | `/invoices/{invoice_id}` |
| Delete invoice | DELETE | `/invoices/{invoice_id}` |
| Update custom field | PUT | `/invoice/{invoice_id}/customfields` |
| Mark as sent | POST | `/invoices/{invoice_id}/status/sent` |
| Void invoice | POST | `/invoices/{invoice_id}/status/void` |
| Mark as draft | POST | `/invoices/{invoice_id}/status/draft` |
| Email invoice | POST | `/invoices/{invoice_id}/email` |
| Get email content | GET | `/invoices/{invoice_id}/email` |
| Bulk email | POST | `/invoices/email` |
| Payment reminder content | GET | `/invoices/{invoice_id}/paymentreminder` |
| Disable reminder | POST | `/invoices/{invoice_id}/paymentreminder/disable` |
| Enable reminder | POST | `/invoices/{invoice_id}/paymentreminder/enable` |
| Bulk export PDF | GET | `/invoices/pdf` |
| Bulk print | GET | `/invoices/print` |
| Write off | POST | `/invoices/{invoice_id}/writeoff` |
| Cancel write off | POST | `/invoices/{invoice_id}/writeoff/cancel` |
| Update billing address | PUT | `/invoices/{invoice_id}/address/billing` |
| Update shipping address | PUT | `/invoices/{invoice_id}/address/shipping` |
| List templates | GET | `/invoices/templates` |
| Update template | PUT | `/invoices/{invoice_id}/templates/{template_id}` |
| List payments | GET | `/invoices/{invoice_id}/payments` |
| List credits applied | GET | `/invoices/{invoice_id}/creditsapplied` |
| Apply credits | POST | `/invoices/{invoice_id}/credits` |
| Delete payment | DELETE | `/invoices/{invoice_id}/payments/{invoice_payment_id}` |
| Delete credit | DELETE | `/invoices/{invoice_id}/creditsapplied/{creditnotes_invoice_id}` |
| Add attachment | POST | `/invoices/{invoice_id}/attachment` |
| Get attachment | GET | `/invoices/{invoice_id}/attachment` |
| Delete attachment | DELETE | `/invoices/{invoice_id}/attachment` |
| Add comment | POST | `/invoices/{invoice_id}/comments` |
| List comments | GET | `/invoices/{invoice_id}/comments` |
| Update comment | PUT | `/invoices/{invoice_id}/comments/{comment_id}` |
| Delete comment | DELETE | `/invoices/{invoice_id}/comments/{comment_id}` |

**Required fields:** `customer_id`, `line_items[]`

**Status values:** `sent`, `draft`, `overdue`, `paid`, `void`, `unpaid`, `partially_paid`, `viewed`

**India-specific fields on Invoices:**
- `gst_no` — 15-digit GSTIN
- `gst_treatment` — `business_gst`, `business_none`, `overseas`, `consumer`
- `place_of_supply` — 2-char state code
- `is_pre_gst` — boolean
- Per line item: `hsn_or_sac`

---

### 2.8 Packages

**Source:** [Packages API](https://www.zoho.com/inventory/api/v1/packages/)

| Operation | Method | Endpoint |
|---|---|---|
| Create package | POST | `/packages` |
| List all packages | GET | `/packages` |
| Update package | PUT | `/packages/{package_id}` |
| Retrieve package | GET | `/packages/{package_id}` |
| Delete package | DELETE | `/packages/{package_id}` |
| Bulk print | GET | `/packages/print` |

**Note:** Creating a package requires `salesorder_id` as a query parameter:
```
POST /packages?organization_id=10234695&salesorder_id=504366000000062000
```

**Required fields:** `date`, `line_items[]` (with `so_line_item_id` and `quantity`)

**Package body:**
```json
{
  "package_number": "PA-00001",
  "date": "2017-01-11",
  "line_items": [
    {
      "so_line_item_id": 504366000000062000,
      "quantity": 2
    }
  ],
  "notes": "notes"
}
```

---

### 2.9 Shipment Orders

**Source:** [Shipment Orders API](https://www.zoho.com/inventory/api/v1/shipmentorders/)

| Operation | Method | Endpoint |
|---|---|---|
| Create shipment order | POST | `/shipmentorders` |
| Update shipment order | PUT | `/shipmentorders/{shipmentorder_id}` |
| Retrieve shipment order | GET | `/shipmentorders/{shipmentorder_id}` |
| Delete shipment order | DELETE | `/shipmentorders/{shipmentorder_id}` |
| Mark as Delivered | POST | `/shipmentorders/{shipmentorder_id}/status/delivered` |

**Note:** Creating a shipment requires `salesorder_id` and `package_ids` as query parameters:
```
POST /shipmentorders?organization_id=10234695&salesorder_id=4815000000044895
```

**Required fields:** `shipment_number`, `date`, `delivery_method`, `tracking_number`

**Key tracking fields in response:**
- `carrier` — carrier name (e.g. "FedEx")
- `tracking_number` — shipment tracking number
- `status` — `shipped`, `delivered`
- `detailed_status` — verbose status from courier

---

### 2.10 Warehouses / Locations (Multi-Warehouse)

**Note:** The `/warehouses` endpoint returned 404. Warehouses are managed as **Locations** in Zoho Inventory. Each item carries a `locations[]` array.

**Location fields on items and orders:**
```json
"locations": [
  {
    "location_id": "460000000038080",
    "location_name": "Main Warehouse",
    "status": "active",
    "is_primary": true,
    "location_stock_on_hand": "50",
    "location_available_stock": "45",
    "location_actual_available_stock": "40"
  }
]
```

Multi-warehouse is enabled via the **Premium plan** or higher. When creating or updating items, specify stock per location:
```json
"locations": [
  {
    "location_id": "460000000038080",
    "initial_stock": 50,
    "initial_stock_rate": 500
  }
]
```

On Sales Orders and Purchase Orders, specify `location_id` at both order level and line-item level.

Retrieve locations via settings:
```
GET /settings/warehouses?organization_id={id}
Scope: ZohoInventory.settings.READ
```

---

### 2.11 Transfer Orders (Stock Transfers Between Locations)

**Source:** [Transfer Orders API](https://www.zoho.com/inventory/api/v1/transferorders/)

| Operation | Method | Endpoint |
|---|---|---|
| Create transfer order | POST | `/transferorders` |
| List all transfer orders | GET | `/transferorders` |
| Update transfer order | PUT | `/transferorders/{transfer_order_id}` |
| Retrieve transfer order | GET | `/transferorders/{transfer_order_id}` |
| Delete transfer order | DELETE | `/transferorders/{transfer_order_id}` |
| Mark as Received/Transferred | POST | `/transferorders/{transfer_order_id}/markastransferred` |

**Required fields:** `transfer_order_number` (or auto-generated), `date`, `from_location_id`, `to_location_id`, `line_items[]`

**Line item required fields:** `item_id`, `name`, `quantity_transfer`

**Create Transfer Order — Full Request Body:**
```json
{
  "transfer_order_number": "TO-00001",
  "date": "2018-03-23",
  "from_location_id": "460000000038080",
  "to_location_id": "460000000039090",
  "line_items": [
    {
      "item_id": 4815000000044100,
      "name": "Laptop-white/15inch/dell",
      "description": "Just a sample description.",
      "quantity_transfer": 2,
      "unit": "qty"
    }
  ],
  "is_intransit_order": false
}
```

**`is_intransit_order`**: `true` = in-transit (two-step transfer), `false` = immediate transfer

**Query parameters for list:**
- `page` (int, default: 1)
- `per_page` (int, default: 200)
- `ignore_auto_number_generation` (boolean)

---

### 2.12 Price Lists (Pricebooks)

**Note:** The `/pricebooks` endpoint returned 404 during research, suggesting it may require a specific plan (Premium+). Pricebooks are referenced in other endpoints via `pricebook_id`.

**Pricebook fields on items and orders:**
- `pricebook_id` (string) — ID of the pricelist to apply
- `pricebook_rate` (double) — the price computed after applying the pricelist

**Applying a pricebook to a Sales Order:**
```json
{
  "pricebook_id": "4815000000044054",
  ...
}
```

The pricebook_rate is computed server-side and returned in item responses. Price lists can define percentage-based or fixed markups/discounts relative to the standard `rate` field.

---

## 3. Webhooks / Notifications

**Source:** [Incoming Webhooks Help](https://www.zoho.com/us/inventory/help/settings/incoming-webhooks.html) | [Automation Help](https://www.zoho.com/us/inventory/help/settings/automation.html) | [Webhooks & Functions](https://www.zoho.com/us/inventory/customize-workflows-functions/)

### 3.1 Overview

Zoho Inventory supports two webhook directions:

| Type | Direction | Purpose |
|---|---|---|
| **Outgoing Webhooks** (Workflow-triggered) | Zoho → Your server | Push notifications when Zoho Inventory events occur |
| **Incoming Webhooks** | Your server → Zoho | Trigger Zoho Inventory actions from external events |

**There is no native webhook subscription API** (like Shopify's `POST /webhooks`). Instead, outgoing webhooks are configured via the Workflow Automation UI in settings.

---

### 3.2 Outgoing Webhooks (Workflow-Triggered)

Configure at: **Settings → Organization Settings → Workflow Actions → Webhooks**

**Supported Modules for Workflow Triggers:**
- Sales Orders
- Purchase Orders
- Invoices
- Bills
- (Contacts, Items via field updates)

**Supported Trigger Types:**

| Trigger Type | Description |
|---|---|
| **Event Based** | Fires when a record is Created, Edited, Created or Edited, or Deleted |
| **Date Based** | Fires N days before/after Order Date, Shipment Date, or Created Time |

**Event-Based sub-options:**
- When any field is updated
- When any selected field is updated (specify up to 3 fields)
- When all selected fields are updated

**Execution frequency:** Just Once or Every Time (for edit-based triggers)

**Webhook configuration fields:**
- `URL to notify` — your HTTPS endpoint
- `Method` — POST, PUT, or DELETE
- `Custom Parameters` — static key-value pairs (e.g., auth tokens)
- `Entity Parameters` — Append All or Append Selected Zoho fields in the payload

**Sample Webhook Payload (Sales Order):**
```json
{
  "date": "2016-05-23",
  "tax_total": "0.0",
  "salesorder_id": "7605000000295003",
  "discount": "10.00%",
  "shipment_date": "2016-05-23",
  "billing_address": {
    "zip": "94588",
    "country": "USA",
    "address": "4910 Hopyard Rd",
    "city": "Pleasanton",
    "state": "CA"
  },
  "line_items": [...],
  "currency_code": "INR",
  "total": "10820.0",
  "salesorder_number": "SO-00002",
  "customer_name": "Arun",
  "customer_id": "7605000000101007",
  "status": "invoiced"
}
```

---

### 3.3 Incoming Webhooks (External → Zoho Inventory)

Configure at: **Settings → Developer Data → Incoming Webhooks**

**How to create:**
1. Go to Settings → Developer Data → Incoming Webhooks
2. Click `+ New Incoming Webhooks`
3. Enter name, description
4. Write a **Deluge script** defining the action to take
5. Click Save — Zoho generates two URLs:
   - **OAuth URL** — uses Zoho auth token
   - **ZAPI Key URL** — uses a static API key

**Incoming webhook invocation attributes:**
| Attribute | Description |
|---|---|
| `Header` | HTTP headers from the API request |
| `Params` | URL query parameters |
| `Body` | String containing the request body payload |

**Use cases:**
- Create invoice when a form is submitted
- Add contact when a lead fills a form
- Trigger stock actions from external events

---

### 3.4 Custom Functions (Deluge-Based)

Zoho Inventory supports custom automation via **Deluge** code. These are triggered by workflow rules on:
- Invoices
- Sales Orders
- Purchase Orders
- Bills

Custom functions can call back into the Zoho Inventory API using `invokeUrl` with a connection reference, allowing complex conditional logic.

---

## 4. Rate Limits

**Source:** [Zoho Inventory API Introduction](https://www.zoho.com/inventory/api/v1/)

### 4.1 Request-per-Minute Limit

**100 requests per minute per organization** — applies to ALL plans.

On exceeding: HTTP `429` is returned with:
```json
{
  "code": 429,
  "message": "Rate limit exceeded"
}
```

### 4.2 Daily API Request Limits by Plan

| Plan | API Requests / Day |
|---|---|
| Free | 1,000 |
| Standard | 2,000 |
| Professional | 5,000 |
| Premium | 10,000 |
| Enterprise | 10,000 |

### 4.3 Concurrent Request Limits

| Plan | Max Concurrent Calls |
|---|---|
| Free | 5 |
| Paid (Standard, Professional, Premium, Enterprise) | 10 (soft limit) |

On exceeding concurrent limit: HTTP `429` is returned.

### 4.4 Rate Limit Headers

Zoho returns `X-RateLimit-*` headers (standard Zoho pattern). Monitor responses for `429` status and implement exponential backoff.

---

## 5. Data Models

### 5.1 Items — Full Field Reference

**Source:** [Items API](https://www.zoho.com/inventory/api/v1/items/)

| Field | Type | R/W | Description |
|---|---|---|---|
| `item_id` | long | R | Server-generated unique ID |
| `name` | string | W* | Item name (required) |
| `group_id` | string | W | Parent group ID |
| `group_name` | string | W | Parent group name |
| `unit` | string | W | Unit of measurement (e.g. "qty", "kg") |
| `item_type` | string | W | `inventory`, `sales`, `purchases`, `sales_and_purchases` |
| `product_type` | string | W | `goods` or `service` |
| `is_taxable` | boolean | W | Whether item is taxable |
| `tax_id` | long | W | Tax ID applied to item |
| `tax_name` | string | R | Tax name |
| `tax_percentage` | double | R | Tax percentage |
| `tax_type` | string | R | Tax type |
| `description` | string | W | Description |
| `purchase_description` | string | W | Purchase-facing description |
| `rate` | double | W | Sales price |
| `purchase_rate` | double | W | Purchase price |
| `pricebook_rate` | double | R | Rate after pricelist applied |
| `reorder_level` | double | W | Reorder point quantity |
| `vendor_id` | long | W | Preferred vendor ID |
| `vendor_name` | string | R | Preferred vendor name |
| `purchase_account_id` | long | W | Accounts payable account ID |
| `purchase_account_name` | string | R | AP account name |
| `account_name` | string | W | Sales account name |
| `inventory_account_id` | long | W | Inventory asset account ID |
| `sku` | string | W | Stock Keeping Unit (unique) |
| `upc` | long | W | 12-digit UPC barcode |
| `ean` | long | W | 13-digit EAN barcode |
| `isbn` | string | W | ISBN |
| `part_number` | string | W | MPN |
| `status` | string | R | `active` or `inactive` |
| `source` | string | R | Record source |
| `is_linked_with_zohocrm` | boolean | R | CRM link flag |
| `is_combo_product` | boolean | R | Is it a composite item? |
| `attribute_id1` | long | R | Attribute ID (for item groups) |
| `attribute_name1` | string | W | Attribute name |
| `attribute_option_id1` | long | R | Attribute option ID |
| `attribute_option_name1` | long | W | Attribute option name |
| `image_id` | long | R | Image ID |
| `image_name` | string | R | Image filename |
| `image_type` | string | R | Image file type |
| `locations[]` | array | W | Per-warehouse stock data |
| `locations[].location_id` | string | W | Warehouse location ID |
| `locations[].location_name` | string | R | Warehouse name |
| `locations[].is_primary` | boolean | R | Is primary warehouse |
| `locations[].location_stock_on_hand` | string | R | Stock (invoice/bill based) |
| `locations[].location_available_stock` | string | R | Stock (shipment/receive based) |
| `locations[].location_actual_available_stock` | string | R | Available minus ordered |
| `custom_fields[]` | array | W | Custom fields |
| `documents[]` | array | W | Attached documents |
| `created_time` | string | R | ISO timestamp |
| `last_modified_time` | string | R | ISO timestamp |
| **India-only fields** | | | |
| `hsn_or_sac` | string | W | HSN/SAC code |
| `item_tax_preferences[]` | array | W | India GST tax preferences |
| `item_tax_preferences[].tax_id` | long | W | Tax ID |
| `item_tax_preferences[].tax_specification` | string | W | `intra` (within state) or `inter` (cross-state) |

---

### 5.2 Sales Orders — Full Field Reference

| Field | Type | Description |
|---|---|---|
| `salesorder_id` | long | Server-generated ID |
| `salesorder_number` | string | Unique SO number |
| `date` | string | SO date (yyyy-mm-dd) |
| `status` | string | `draft`, `confirmed`, `fulfilled`, `void` |
| `shipment_date` | string | Expected shipment date |
| `reference_number` | string | External reference |
| `customer_id` | long | Customer ID (required) |
| `customer_name` | string | Customer name |
| `currency_id` | long | Currency ID |
| `currency_code` | string | Currency code |
| `exchange_rate` | double | Exchange rate vs base currency |
| `discount` | double | Discount (% or amount) |
| `discount_amount` | double | Computed discount |
| `is_discount_before_tax` | boolean | Pre-tax discount |
| `discount_type` | string | `entity_level` or `item_level` |
| `delivery_method` | string | Carrier name |
| `delivery_method_id` | long | Carrier ID |
| `pricebook_id` | string | Applied pricelist ID |
| `salesperson_id` | string | Salesperson ID |
| `salesperson_name` | string | Salesperson name |
| `estimate_id` | long | Linked estimate (from Zoho Books) |
| `line_items[]` | array | Line items |
| `line_items[].item_id` | long | Item ID |
| `line_items[].line_item_id` | long | Line item ID |
| `line_items[].name` | string | Item name |
| `line_items[].rate` | double | Unit price |
| `line_items[].quantity` | double | Qty ordered |
| `line_items[].quantity_invoiced` | double | Qty invoiced |
| `line_items[].quantity_packed` | double | Qty packed |
| `line_items[].quantity_shipped` | double | Qty shipped |
| `line_items[].tax_id` | long | Tax ID |
| `line_items[].tax_percentage` | double | Tax rate % |
| `line_items[].item_total` | double | Line total |
| `line_items[].is_invoiced` | boolean | Invoiced? |
| `line_items[].location_id` | string | Warehouse for this line |
| `line_items[].hsn_or_sac` | string | **India only** HSN/SAC |
| `location_id` | string | Order-level warehouse |
| `shipping_charge` | double | Shipping fee |
| `adjustment` | double | Miscellaneous adjustment |
| `adjustment_description` | string | Reason for adjustment |
| `sub_total` | double | Pre-tax subtotal |
| `tax_total` | double | Total tax |
| `total` | double | Grand total |
| `taxes[]` | array | Tax breakdown |
| `billing_address` | object | Customer billing address |
| `shipping_address` | object | Customer shipping address |
| `notes` | string | Internal notes |
| `terms` | string | Terms and conditions text |
| `template_id` | long | PDF template ID |
| `created_time` | string | ISO timestamp |
| `last_modified_time` | string | ISO timestamp |
| **India-only** | | |
| `gst_no` | string | Customer 15-digit GSTIN |
| `gst_treatment` | string | `business_gst`, `business_none`, `overseas`, `consumer` |
| `place_of_supply` | string | 2-char state code |
| `is_pre_gst` | boolean | Pre-July 2017 transaction |

---

### 5.3 Purchase Orders — Full Field Reference

| Field | Type | Description |
|---|---|---|
| `purchaseorder_id` | long | Server-generated ID |
| `purchaseorder_number` | string | Unique PO number |
| `date` | string | PO date |
| `expected_delivery_date` | string | Expected delivery date |
| `status` | string | `draft`, `issued`, `Partially_Received`, `received`, `cancelled` |
| `vendor_id` | long | Vendor ID (required) |
| `vendor_name` | string | Vendor name |
| `is_drop_shipment` | boolean | Drop shipment flag |
| `salesorder_id` | long | Linked SO (for drop shipment) |
| `is_backorder` | boolean | Is a backorder? |
| `is_inclusive_tax` | boolean | Tax-inclusive pricing? |
| `currency_code` | string | Currency |
| `exchange_rate` | double | Exchange rate |
| `delivery_date` | string | Actual delivery date |
| `ship_via` | string | Shipping carrier |
| `reference_number` | string | Vendor reference |
| `delivery_customer_id` | long | Drop-ship customer ID |
| `delivery_address` | array | Drop-ship delivery address |
| `delivery_org_address_id` | long | Org's own delivery address ID |
| `line_items[]` | array | Line items |
| `line_items[].item_id` | long | Item ID |
| `line_items[].purchase_rate` | double | Purchase price |
| `line_items[].quantity` | double | Ordered qty |
| `line_items[].quantity_received` | double | Received qty |
| `line_items[].location_id` | string | Destination warehouse |
| `line_items[].hsn_or_sac` | string | **India only** |
| `line_items[].reverse_charge_tax_id` | string | **India only** |
| `line_items[].reverse_charge_tax_percentage` | double | **India only** |
| `line_items[].reverse_charge_tax_amount` | double | **India only** |
| `line_items[].tax_exemption_code` | string | India/AU/CA/MX |
| `purchasereceives[]` | array | Goods receipt records |
| `bills[]` | array | Linked bills |
| `sub_total` | double | Subtotal |
| `tax_total` | double | Tax total |
| `total` | double | Grand total |
| **India-only** | | |
| `gst_no` | string | Vendor 15-digit GSTIN |
| `gst_treatment` | string | GST treatment type |
| `source_of_supply` | string | Origin state code |
| `destination_of_supply` | string | Destination state code |
| `is_pre_gst` | boolean | Pre-July 2017 |
| `is_reverse_charge_applied` | boolean | Reverse charge applicable? |

---

### 5.4 Invoices — Full Field Reference

| Field | Type | Description |
|---|---|---|
| `invoice_id` | string | Server-generated ID |
| `invoice_number` | string | Invoice number (max 100 chars) |
| `date` | string | Invoice date (yyyy-mm-dd) |
| `status` | string | `sent`, `draft`, `overdue`, `paid`, `void`, `unpaid`, `partially_paid`, `viewed` |
| `due_date` | string | Payment due date |
| `payment_terms` | integer | Net payment days (15, 30, 60) |
| `payment_terms_label` | string | e.g., "Net 15 Days" |
| `payment_expected_date` | string | Expected payment date |
| `customer_id` | string | Customer ID (required) |
| `customer_name` | string | Customer name |
| `currency_code` | string | Invoice currency |
| `exchange_rate` | float | Exchange rate |
| `discount` | float | Entity-level discount (% or amount) |
| `is_discount_before_tax` | boolean | Pre-tax discount? |
| `discount_type` | string | `entity_level` or `item_level` |
| `is_inclusive_tax` | boolean | Tax-inclusive? |
| `line_items[]` | array | Line items |
| `line_items[].item_id` | string | Item ID |
| `line_items[].name` | string | Item name (max 100) |
| `line_items[].rate` | double | Unit price |
| `line_items[].quantity` | float | Quantity |
| `line_items[].discount` | float | Line discount |
| `line_items[].tax_id` | string | Tax ID |
| `line_items[].tax_percentage` | float | Tax % |
| `line_items[].item_total` | float | Line total |
| `line_items[].location_id` | string | Warehouse |
| `line_items[].hsn_or_sac` | string | **India only** |
| `shipping_charge` | string | Shipping fee |
| `adjustment` | double | Miscellaneous adjustment |
| `sub_total` | float | Pre-tax total |
| `tax_total` | double | Tax total |
| `total` | string | Grand total |
| `balance` | string | Outstanding balance |
| `payment_made` | float | Total payments received |
| `credits_applied` | float | Credits applied |
| `write_off_amount` | float | Written-off amount |
| `allow_partial_payments` | boolean | Allow partial? |
| `payment_reminder_enabled` | boolean | Reminders active? |
| `billing_address` | object | Billing address |
| `shipping_address` | object | Shipping address |
| `payment_options` | object | Payment gateways |
| `template_id` | string | PDF template |
| `salesperson_id` | string | Salesperson ID |
| `invoice_url` | string | Secure payment link |
| `is_emailed` | boolean | Has been emailed? |
| `is_viewed_by_client` | boolean | Client has viewed? |
| `recurring_invoice_id` | string | Parent recurring invoice |
| `created_time` | string | Creation timestamp |
| `last_modified_time` | string | Last modified timestamp |
| **India-only** | | |
| `gst_no` | string | Customer 15-digit GSTIN |
| `gst_treatment` | string | `business_gst`, `business_none`, `overseas`, `consumer` |
| `place_of_supply` | string | 2-char state code (e.g. "TN") |
| `is_pre_gst` | boolean | Pre-July 2017 transaction |

---

## 6. India-Specific Fields (GST / HSN)

**Source:** [Items API](https://www.zoho.com/inventory/api/v1/items/), [Sales Orders API](https://www.zoho.com/inventory/api/v1/salesorders/), [Purchase Orders API](https://www.zoho.com/inventory/api/v1/purchaseorders/)

### 6.1 HSN/SAC Codes

Applied at **item level** and also at **line-item level** in transactions:
```json
"hsn_or_sac": "85423100"
```

HSN = Harmonized System of Nomenclature (for goods)
SAC = Services Accounting Code (for services)

The same field `hsn_or_sac` holds both. Present on:
- Items (`ZohoInventory.items.*`)
- Composite Items
- Sales Order line items
- Purchase Order line items
- Invoice line items

### 6.2 GST Treatment Values

Used on Contacts, Sales Orders, Purchase Orders, Invoices:

| Value | Meaning |
|---|---|
| `business_gst` | GST-registered business |
| `business_none` | Unregistered business |
| `overseas` | Overseas (export) |
| `consumer` | End consumer (B2C) |

### 6.3 GST Fields on Transactions

| Field | Where | Description |
|---|---|---|
| `gst_no` | SO, PO, Invoice | 15-digit GSTIN of counterparty |
| `gst_treatment` | SO, PO, Invoice | See table above |
| `place_of_supply` | SO, Invoice | 2-char state code for delivery |
| `source_of_supply` | PO | 2-char state code for origin |
| `destination_of_supply` | PO | 2-char state code for destination |
| `is_pre_gst` | SO, PO, Invoice | True for pre-July 2017 transactions |
| `is_reverse_charge_applied` | PO | True when buyer pays reverse charge |

### 6.4 State Codes (Sample)

| State | Code |
|---|---|
| Andhra Pradesh | AP |
| Karnataka | KA |
| Maharashtra | MH |
| Tamil Nadu | TN |
| Telangana | TS |
| Delhi | DL |
| Gujarat | GJ |
| Rajasthan | RJ |
| Uttar Pradesh | UP |
| West Bengal | WB |

### 6.5 Intra vs Inter State Tax

Controlled via `item_tax_preferences[].tax_specification`:
- `intra` = within same state (CGST + SGST applies)
- `inter` = cross-state transaction (IGST applies)

This allows Zoho to auto-select the correct tax component.

### 6.6 Reverse Charge Fields (Purchase Orders)

Per line item:
```json
{
  "reverse_charge_tax_id": "460000000026068",
  "reverse_charge_tax_name": "inter",
  "reverse_charge_tax_percentage": 10,
  "reverse_charge_tax_amount": 100
}
```

Header flag: `"is_reverse_charge_applied": true`

---

## 7. Zoho Inventory ↔ Zoho Books Sync

**Source:** [Data Sync Documentation](https://www.zoho.com/us/books/kb/integrations/data-sync.html) | [Books-Inventory Integration](https://www.zoho.com/books/integrations/zoho-inventory/)

### 7.1 Overview

When Zoho Books and Zoho Inventory are integrated (same Zoho organization), data syncs **bidirectionally and in real time**. Any operation in Books reflects in Inventory and vice versa.

### 7.2 What Syncs Automatically

| Zoho Books | ↔ | Zoho Inventory |
|---|---|---|
| Items | ↔ | Items |
| Contacts | ↔ | Contacts |
| Sales Orders | ↔ | Sales Orders |
| Purchase Orders | ↔ | Purchase Orders |
| Invoices | ↔ | Invoices |
| Bills | ↔ | Bills |
| Settings (taxes, currencies) | ↔ | Settings |
| Retainer Invoices | ↔ | Retainer Invoices |
| Reports (common) | ↔ | Reports |

### 7.3 What Does NOT Auto-Sync

- **Workflow rules** — Zoho has confirmed these do NOT sync between Books and Inventory
- **Packages and Shipments** — Inventory-specific, exist only in Inventory
- **Transfer Orders** — Inventory-specific
- **Inventory Adjustments** — Inventory-specific
- Custom workflows/automations created in one app are not replicated to the other

### 7.4 Key Integration Behavior

- **Single Org Context:** Both apps share the same organization ID. The `organization_id` used in API calls is identical for both Books and Inventory APIs.
- **Inventory-Side Accounting:** When a shipment is marked delivered in Inventory, the COGS (Cost of Goods Sold) journal entry is automatically created in Books.
- **Books-Side Invoicing:** Invoices created in Books appear instantly in Inventory for reference.
- **API Access:** You can use the Zoho Books API (`https://www.zohoapis.in/books/v3/`) or Zoho Inventory API (`https://www.zohoapis.in/inventory/v1/`) to access the same shared data.

### 7.5 Shopify Integration Note

Zoho Inventory has a native Shopify integration. When connected:
- Shopify orders → Zoho Inventory Sales Orders
- Zoho Inventory stock levels → synced back to Shopify
- Invoices can be auto-generated from fulfilled Shopify orders
- All of this then flows to Zoho Books automatically

---

## 8. Pagination

**Source:** [Pagination API](https://www.zoho.com/inventory/api/v1/pagination/)

### 8.1 Mechanism

All list endpoints use **page-based pagination**. Default page size is **200 records**.

**Request:**
```
GET /items?organization_id=10234695&page=2&per_page=25
```

**Parameters:**
| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | integer | 1 | Page number to fetch |
| `per_page` | integer | 200 | Records per page (max 200) |

**Response includes `page_context` object:**
```json
{
  "code": 0,
  "message": "success",
  "items": [{...}, {...}],
  "page_context": {
    "page": 2,
    "per_page": 25,
    "has_more_page": false
  }
}
```

### 8.2 Traversal Strategy

```python
page = 1
all_records = []
while True:
    response = get(f"/items?page={page}&per_page=200")
    all_records.extend(response["items"])
    if not response["page_context"]["has_more_page"]:
        break
    page += 1
```

### 8.3 Key Pagination Notes

- `has_more_page: true` means there is at least one more page
- `has_more_page: false` means you have reached the last page
- Maximum `per_page` is **200**
- No cursor-based pagination — only page number
- Applies to all list endpoints: items, salesorders, invoices, purchaseorders, transferorders, packages, etc.

---

## 9. Error Codes and Retry Strategies

**Source:** [Errors API](https://www.zoho.com/inventory/api/v1/errors/)

### 9.1 HTTP Status Codes

| HTTP Code | Meaning | Action |
|---|---|---|
| `200` | Success | Process response |
| `201` | Created | Record added successfully |
| `400` | Bad Request | Fix request (malformed or missing parameter) — do not retry |
| `401` | Unauthorized (Invalid AuthToken) | Refresh access token, then retry |
| `404` | URL Not Found | Check URL or resource existence — do not retry |
| `405` | Method Not Allowed | Check HTTP verb — do not retry |
| `429` | Too Many Requests | Back off and retry after delay |
| `500` | Server Error | Retry with exponential backoff; contact support if persistent |

### 9.2 Application-Level Error Codes

Zoho Inventory returns application-level codes in the `code` field of the JSON response (distinct from HTTP status):

| App Code | Meaning |
|---|---|
| `0` | Success |
| `1002` | Record does not exist (e.g., "Invoice does not exist.") |
| Non-zero | Error — check `message` field for details |

**Example error response:**
```json
{
  "code": 1002,
  "message": "Invoice does not exist."
}
```

### 9.3 401 Error — Token Handling

The most common error. Causes:
1. Access token expired (1-hour TTL)
2. Wrong data center domain used (e.g., using `.com` token for `.in` org)
3. Insufficient scope for the operation

**Resolution:**
1. If expired → POST to `oauth/v2/token` with `grant_type=refresh_token`
2. If domain mismatch → ensure token was generated from the correct DC (`accounts.zoho.in` for India)
3. If scope issue → re-authorize with the required scope

### 9.4 429 Error — Rate Limit Handling

**Two types:**
1. **Per-minute limit exceeded** (100 req/min) — wait 60 seconds and retry
2. **Daily limit exceeded** — wait until midnight UTC for the quota to reset; consider upgrading plan

**Recommended backoff strategy:**
```
Attempt 1: wait 1s
Attempt 2: wait 2s
Attempt 3: wait 4s
Attempt 4: wait 8s
Attempt 5: wait 16s
Max retries: 5
```

### 9.5 500 Server Error

Rare. Retry up to 3 times with 5-second intervals. If persistent, contact `support@zoho-inventory.com`.

### 9.6 Common Validation Errors (400)

These return `code: 400` with descriptive messages:
- Missing required field (e.g., `customer_id`)
- Invalid date format (use `yyyy-mm-dd`)
- Duplicate `salesorder_number` or `sku`
- Invalid `organization_id`
- Item marked inactive cannot be used in transactions

---

## Appendix: Quick Reference — Key API Endpoints Summary

| Module | Create | List | Retrieve | Update | Delete |
|---|---|---|---|---|---|
| Items | `POST /items` | `GET /items` | `GET /items/{id}` | `PUT /items/{id}` | `DELETE /items/{id}` |
| Composite Items | `POST /compositeitems` | `GET /compositeitems` | `GET /compositeitems/{id}` | `PUT /compositeitems/{id}` | `DELETE /compositeitems/{id}` |
| Item Groups | `POST /itemgroups` | `GET /itemgroups` | `GET /itemgroups/{id}` | `PUT /itemgroups/{id}` | `DELETE /itemgroups/{id}` |
| Sales Orders | `POST /salesorders` | `GET /salesorders` | `GET /salesorders/{id}` | `PUT /salesorders/{id}` | `DELETE /salesorders/{id}` |
| Purchase Orders | `POST /purchaseorders` | `GET /purchaseorders` | `GET /purchaseorders/{id}` | `PUT /purchaseorders/{id}` | `DELETE /purchaseorders/{id}` |
| Invoices | `POST /invoices` | `GET /invoices` | `GET /invoices/{id}` | `PUT /invoices/{id}` | `DELETE /invoices/{id}` |
| Packages | `POST /packages` | `GET /packages` | `GET /packages/{id}` | `PUT /packages/{id}` | `DELETE /packages/{id}` |
| Shipment Orders | `POST /shipmentorders` | — | `GET /shipmentorders/{id}` | `PUT /shipmentorders/{id}` | `DELETE /shipmentorders/{id}` |
| Transfer Orders | `POST /transferorders` | `GET /transferorders` | `GET /transferorders/{id}` | `PUT /transferorders/{id}` | `DELETE /transferorders/{id}` |

---

## Appendix: Important Notes for Shopify-Accounting Integration

1. **India Org:** Always use `accounts.zoho.in` for auth and `www.zohoapis.in` for API calls. Mixing domains causes 401 errors.

2. **Organization ID:** Retrieve once on connect via `GET /organizations` and cache it.

3. **Refresh Token Storage:** Store securely. Per-user limit is 20 tokens. Oldest is auto-deleted when exceeded.

4. **Pagination:** Max 200 records per call. For large catalogs, implement loop with `has_more_page` check.

5. **India GST Compliance:** For every transaction, always pass `gst_no`, `gst_treatment`, and `place_of_supply`. Pass `hsn_or_sac` on every line item.

6. **Dual-Stack (Books + Inventory):** If the merchant uses both Zoho Books and Inventory, creating a Sales Order or Invoice via the Inventory API automatically appears in Books. Avoid creating duplicates by checking for existing records first.

7. **Rate Limits:** For bulk syncs (e.g., initial product catalog import), implement a queue with ≤80 requests/min to maintain safe headroom below the 100/min limit.

8. **Webhook Limitations:** No programmatic webhook registration API exists. Webhooks must be configured via the UI or by using Incoming Webhooks (which receive from your server, not push to it). For outgoing notifications from Zoho to your server, use the Workflow Automation webhook UI.

9. **Access Token Refresh:** Proactively refresh the token at ~55 minutes (before the 60-minute expiry) rather than waiting for a 401.

10. **API Version:** Current version is `v1` (path: `/inventory/v1/`). Zoho Books uses `v3` (path: `/books/v3/`). These are separate versioned APIs despite sharing data.
