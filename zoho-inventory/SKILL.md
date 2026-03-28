---
name: zoho-inventory-integration
description: Integrates Zoho Inventory API into a Shopify accounting platform. Covers OAuth 2.0 authentication for India and global DCs, Items/Sales Orders/Invoices/Purchase Orders CRUD, India GST fields (HSN codes, place of supply, gst_treatment), multi-warehouse transfers, Zoho Books bidirectional sync, pagination, and rate limiting. Use when building or modifying Zoho Inventory integration, syncing Shopify products or orders to Zoho, or handling inventory operations.
---

# Zoho Inventory Integration Skill
## For: Shopify Accounting Platform (India + Singapore)
## Version: 1.0 | March 2026

---

## Table of Contents
1. [Overview & Architecture](#1-overview--architecture)
2. [Authentication Implementation](#2-authentication-implementation)
3. [Core API Integration Patterns](#3-core-api-integration-patterns)
4. [India GST-Specific Fields](#4-india-gst-specific-fields)
5. [Zoho Inventory ↔ Zoho Books Sync](#5-zoho-inventory--zoho-books-sync)
6. [Webhooks & Real-time Sync](#6-webhooks--real-time-sync)
7. [Pagination & Bulk Operations](#7-pagination--bulk-operations)
8. [Rate Limiting & Error Handling](#8-rate-limiting--error-handling)
9. [Data Model Reference](#9-data-model-reference)
10. [Implementation Checklist](#10-implementation-checklist)

---

## 1. Overview & Architecture

### 1.1 What This Integration Does

This integration connects a Shopify store to Zoho Inventory (with automatic Zoho Books sync) to provide complete inventory management and GST-compliant accounting for India and Singapore merchants. It handles:

- **Product sync**: Shopify products → Zoho Inventory items (with HSN codes, tax preferences)
- **Order fulfillment**: Shopify orders → Zoho Sales Orders → Packages → Shipments
- **Invoicing**: Auto-generate Zoho Invoices from fulfilled Shopify orders
- **Purchasing**: Create Zoho Purchase Orders for restocking triggered by low stock
- **Stock management**: Inventory adjustments, multi-warehouse transfer orders
- **Accounting**: All transactions auto-sync to Zoho Books for GST filing

### 1.2 Data Flow

```
Shopify Store
     │
     │ (Shopify webhooks: orders/create, products/update, fulfillments/create)
     ▼
Your Platform (Node.js/TypeScript)
     │
     ├─── Products ──────────► Zoho Inventory Items (POST /items)
     │                              │
     ├─── Customer + Order ─────► Zoho Contact + Sales Order (POST /salesorders)
     │                              │
     ├─── Fulfillment ──────────► Package + Shipment Order (POST /packages, /shipmentorders)
     │                              │
     ├─── Low Stock Alert ──────► Purchase Order (POST /purchaseorders)
     │                              │
     └─── Stock Correction ──────► Inventory Adjustment (POST /inventoryadjustments)
                                    │
                                    │ (automatic bidirectional sync)
                                    ▼
                               Zoho Books
                               (Invoices, Bills, COGS entries, GST reports)
```

### 1.3 When to Use Zoho Inventory API vs Zoho Books API

| Use Case | API to Use | Endpoint Base |
|---|---|---|
| Item/product management | Zoho Inventory | `https://www.zohoapis.in/inventory/v1/` |
| Sales orders | Zoho Inventory | `https://www.zohoapis.in/inventory/v1/` |
| Packages & shipments | Zoho Inventory | `https://www.zohoapis.in/inventory/v1/` |
| Transfer orders | Zoho Inventory | `https://www.zohoapis.in/inventory/v1/` |
| Inventory adjustments | Zoho Inventory | `https://www.zohoapis.in/inventory/v1/` |
| Invoices (read/reference) | Either (shared data) | Both APIs see same invoices |
| Invoices (create) | Zoho Inventory | `https://www.zohoapis.in/inventory/v1/` |
| GST returns, financial reports | Zoho Books | `https://www.zohoapis.in/books/v3/` |
| Chart of accounts, journal entries | Zoho Books | `https://www.zohoapis.in/books/v3/` |
| Customer payments, reconciliation | Zoho Books | `https://www.zohoapis.in/books/v3/` |

**Rule of thumb**: Use Zoho Inventory API for all inventory-touching operations. Use Zoho Books API for pure accounting operations. Both share the same `organization_id` and data syncs automatically between them.

### 1.4 Data Center Configuration

This platform targets two data centers:

| Market | DC | Auth URL | API Base URL |
|---|---|---|---|
| India | `.in` | `https://accounts.zoho.in` | `https://www.zohoapis.in/inventory/v1` |
| Singapore / Global | `.com` | `https://accounts.zoho.com` | `https://www.zohoapis.com/inventory/v1` |

Every token, API call, and organization ID is DC-specific. Store and use the correct DC per merchant.

---

## 2. Authentication Implementation

### 2.1 Overview

Zoho uses OAuth 2.0 with the authorization code flow. Access tokens expire after **1 hour**. Refresh tokens are permanent (up to 20 per user per client app).

**Critical DC rule**: The auth domain used to generate a token must match the API domain used for API calls. India merchants must use `accounts.zoho.in` for auth and `www.zohoapis.in` for API calls. Mixing causes `401` errors.

### 2.2 Required Scopes

Use `ZohoInventory.FullAccess.all` for maximum access, or enumerate granular scopes below:

```typescript
// Recommended: use FullAccess for simplicity during development
export const ZOHO_SCOPES_FULL = 'ZohoInventory.FullAccess.all';

// Granular scopes for production (principle of least privilege)
export const ZOHO_SCOPES_GRANULAR = [
  'ZohoInventory.items.CREATE',
  'ZohoInventory.items.UPDATE',
  'ZohoInventory.items.READ',
  'ZohoInventory.items.DELETE',
  'ZohoInventory.compositeitems.CREATE',
  'ZohoInventory.compositeitems.UPDATE',
  'ZohoInventory.compositeitems.READ',
  'ZohoInventory.compositeitems.DELETE',
  'ZohoInventory.inventoryadjustments.CREATE',
  'ZohoInventory.inventoryadjustments.READ',
  'ZohoInventory.inventoryadjustments.UPDATE',
  'ZohoInventory.inventoryadjustments.DELETE',
  'ZohoInventory.transferorders.CREATE',
  'ZohoInventory.transferorders.READ',
  'ZohoInventory.transferorders.UPDATE',
  'ZohoInventory.transferorders.DELETE',
  'ZohoInventory.settings.CREATE',
  'ZohoInventory.settings.UPDATE',
  'ZohoInventory.settings.READ',
  'ZohoInventory.salesorders.CREATE',
  'ZohoInventory.salesorders.UPDATE',
  'ZohoInventory.salesorders.READ',
  'ZohoInventory.salesorders.DELETE',
  'ZohoInventory.packages.CREATE',
  'ZohoInventory.packages.UPDATE',
  'ZohoInventory.packages.READ',
  'ZohoInventory.packages.DELETE',
  'ZohoInventory.shipmentorders.CREATE',
  'ZohoInventory.shipmentorders.UPDATE',
  'ZohoInventory.shipmentorders.READ',
  'ZohoInventory.invoices.CREATE',
  'ZohoInventory.invoices.UPDATE',
  'ZohoInventory.invoices.READ',
  'ZohoInventory.invoices.DELETE',
  'ZohoInventory.purchaseorders.CREATE',
  'ZohoInventory.purchaseorders.UPDATE',
  'ZohoInventory.purchaseorders.READ',
  'ZohoInventory.purchaseorders.DELETE',
  'ZohoInventory.contacts.CREATE',
  'ZohoInventory.contacts.UPDATE',
  'ZohoInventory.contacts.READ',
  'ZohoInventory.contacts.DELETE',
].join(',');
```

### 2.3 TypeScript Types

```typescript
// src/zoho/types/auth.ts

export type ZohoDataCenter = 'in' | 'com' | 'eu' | 'com.au' | 'ca' | 'jp';

export interface ZohoAuthConfig {
  clientId: string;
  clientSecret: string;
  redirectUri: string;
  dataCenter: ZohoDataCenter;
  scopes: string;
}

export interface ZohoTokenSet {
  accessToken: string;
  refreshToken: string;
  expiresAt: number; // Unix timestamp in ms
  organizationId: string;
  dataCenter: ZohoDataCenter;
}

export interface ZohoTokenResponse {
  access_token: string;
  refresh_token?: string;
  token_type: string;
  expires_in: number; // seconds
}

export interface ZohoOrganization {
  organization_id: string;
  name: string;
  country_code: string;
  time_zone: string;
  currency_code: string;
}
```

### 2.4 DC Helper

```typescript
// src/zoho/auth/dc-config.ts

import { ZohoDataCenter } from '../types/auth';

export function getAuthBaseUrl(dc: ZohoDataCenter): string {
  const map: Record<ZohoDataCenter, string> = {
    'in':     'https://accounts.zoho.in',
    'com':    'https://accounts.zoho.com',
    'eu':     'https://accounts.zoho.eu',
    'com.au': 'https://accounts.zoho.com.au',
    'ca':     'https://accounts.zohocloud.ca',
    'jp':     'https://accounts.zoho.jp',
  };
  return map[dc];
}

export function getApiBaseUrl(dc: ZohoDataCenter): string {
  const map: Record<ZohoDataCenter, string> = {
    'in':     'https://www.zohoapis.in/inventory/v1',
    'com':    'https://www.zohoapis.com/inventory/v1',
    'eu':     'https://www.zohoapis.eu/inventory/v1',
    'com.au': 'https://www.zohoapis.com.au/inventory/v1',
    'ca':     'https://www.zohoapis.ca/inventory/v1',
    'jp':     'https://www.zohoapis.jp/inventory/v1',
  };
  return map[dc];
}
```

### 2.5 Auth URL Generation

```typescript
// src/zoho/auth/oauth.ts

import crypto from 'crypto';
import { ZohoAuthConfig } from '../types/auth';
import { getAuthBaseUrl } from './dc-config';

export function generateAuthUrl(
  config: ZohoAuthConfig,
  merchantId: string // your internal merchant ID, used as CSRF state
): { url: string; state: string } {
  const state = `${merchantId}:${crypto.randomBytes(16).toString('hex')}`;

  const params = new URLSearchParams({
    scope: config.scopes,
    client_id: config.clientId,
    response_type: 'code',
    redirect_uri: config.redirectUri,
    access_type: 'offline',       // request refresh token
    prompt: 'Consent',             // always show consent screen (required for refresh token)
    state,
  });

  const url = `${getAuthBaseUrl(config.dataCenter)}/oauth/v2/auth?${params.toString()}`;

  return { url, state };
}
```

### 2.6 Token Exchange (Authorization Code → Access + Refresh Tokens)

```typescript
// src/zoho/auth/token-exchange.ts

import axios from 'axios';
import { ZohoAuthConfig, ZohoTokenResponse, ZohoTokenSet } from '../types/auth';
import { getAuthBaseUrl } from './dc-config';
import { fetchOrganizationId } from './organization';

export async function exchangeCodeForTokens(
  config: ZohoAuthConfig,
  code: string,
): Promise<ZohoTokenSet> {
  const authBase = getAuthBaseUrl(config.dataCenter);

  const params = new URLSearchParams({
    code,
    client_id: config.clientId,
    client_secret: config.clientSecret,
    redirect_uri: config.redirectUri,
    grant_type: 'authorization_code',
  });

  const response = await axios.post<ZohoTokenResponse>(
    `${authBase}/oauth/v2/token`,
    params.toString(),
    {
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    }
  );

  const { access_token, refresh_token, expires_in } = response.data;

  if (!refresh_token) {
    throw new Error(
      'No refresh token returned. Ensure access_type=offline and prompt=Consent were set in the auth URL.'
    );
  }

  // Retrieve organization ID immediately after token exchange
  const organizationId = await fetchOrganizationId(
    access_token,
    config.dataCenter
  );

  return {
    accessToken: access_token,
    refreshToken: refresh_token,
    expiresAt: Date.now() + expires_in * 1000,
    organizationId,
    dataCenter: config.dataCenter,
  };
}
```

### 2.7 Fetch Organization ID

```typescript
// src/zoho/auth/organization.ts

import axios from 'axios';
import { ZohoDataCenter, ZohoOrganization } from '../types/auth';
import { getApiBaseUrl } from './dc-config';

export async function fetchOrganizationId(
  accessToken: string,
  dc: ZohoDataCenter
): Promise<string> {
  const apiBase = getApiBaseUrl(dc);

  const response = await axios.get<{
    code: number;
    organizations: ZohoOrganization[];
  }>(`${apiBase}/organizations`, {
    headers: { Authorization: `Zoho-oauthtoken ${accessToken}` },
  });

  if (response.data.code !== 0 || !response.data.organizations?.length) {
    throw new Error('Failed to fetch Zoho organization ID');
  }

  // Use the first organization (most installations have one)
  return response.data.organizations[0].organization_id;
}
```

### 2.8 Token Refresh

```typescript
// src/zoho/auth/token-refresh.ts

import axios from 'axios';
import { ZohoAuthConfig, ZohoTokenSet, ZohoTokenResponse } from '../types/auth';
import { getAuthBaseUrl } from './dc-config';

// Refresh if within 5 minutes of expiry (proactive refresh at 55 min mark)
const TOKEN_REFRESH_BUFFER_MS = 5 * 60 * 1000;

export function isTokenExpired(tokenSet: ZohoTokenSet): boolean {
  return Date.now() >= tokenSet.expiresAt - TOKEN_REFRESH_BUFFER_MS;
}

export async function refreshAccessToken(
  config: ZohoAuthConfig,
  tokenSet: ZohoTokenSet,
): Promise<ZohoTokenSet> {
  const authBase = getAuthBaseUrl(tokenSet.dataCenter);

  const params = new URLSearchParams({
    refresh_token: tokenSet.refreshToken,
    client_id: config.clientId,
    client_secret: config.clientSecret,
    grant_type: 'refresh_token',
  });

  const response = await axios.post<ZohoTokenResponse>(
    `${authBase}/oauth/v2/token`,
    params.toString(),
    {
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    }
  );

  const { access_token, expires_in } = response.data;

  return {
    ...tokenSet,
    accessToken: access_token,
    expiresAt: Date.now() + expires_in * 1000,
  };
}
```

### 2.9 Token Storage (Multi-Tenant)

```typescript
// src/zoho/auth/token-store.ts
// Store tokens per merchant. Use database (Postgres/Redis) in production.

import { ZohoTokenSet } from '../types/auth';

// Interface — implement with your DB/cache layer
export interface TokenStore {
  save(merchantId: string, tokenSet: ZohoTokenSet): Promise<void>;
  load(merchantId: string): Promise<ZohoTokenSet | null>;
  delete(merchantId: string): Promise<void>;
}

// Example in-memory implementation (replace with database in production)
export class InMemoryTokenStore implements TokenStore {
  private store = new Map<string, ZohoTokenSet>();

  async save(merchantId: string, tokenSet: ZohoTokenSet): Promise<void> {
    this.store.set(merchantId, tokenSet);
  }

  async load(merchantId: string): Promise<ZohoTokenSet | null> {
    return this.store.get(merchantId) ?? null;
  }

  async delete(merchantId: string): Promise<void> {
    this.store.delete(merchantId);
  }
}

// Production: PostgreSQL-backed token store
// CREATE TABLE zoho_tokens (
//   merchant_id    TEXT PRIMARY KEY,
//   access_token   TEXT NOT NULL,
//   refresh_token  TEXT NOT NULL,
//   expires_at     BIGINT NOT NULL,
//   organization_id TEXT NOT NULL,
//   data_center    TEXT NOT NULL,
//   updated_at     TIMESTAMPTZ DEFAULT NOW()
// );
```

### 2.10 Auth Middleware (Express)

```typescript
// src/zoho/auth/middleware.ts
// Automatically refreshes tokens before API calls

import { ZohoAuthConfig, ZohoTokenSet } from '../types/auth';
import { TokenStore } from './token-store';
import { isTokenExpired, refreshAccessToken } from './token-refresh';

export class ZohoAuthMiddleware {
  constructor(
    private config: ZohoAuthConfig,
    private tokenStore: TokenStore
  ) {}

  /**
   * Returns a valid access token for the merchant.
   * Refreshes automatically if expired.
   */
  async getValidTokenSet(merchantId: string): Promise<ZohoTokenSet> {
    const tokenSet = await this.tokenStore.load(merchantId);

    if (!tokenSet) {
      throw new Error(
        `No Zoho token found for merchant ${merchantId}. User must complete OAuth flow.`
      );
    }

    if (isTokenExpired(tokenSet)) {
      const refreshed = await refreshAccessToken(this.config, tokenSet);
      await this.tokenStore.save(merchantId, refreshed);
      return refreshed;
    }

    return tokenSet;
  }

  /**
   * Returns Authorization header value for the merchant.
   */
  async getAuthHeader(merchantId: string): Promise<string> {
    const tokenSet = await this.getValidTokenSet(merchantId);
    return `Zoho-oauthtoken ${tokenSet.accessToken}`;
  }
}
```

### 2.11 OAuth Callback Handler (Express Route)

```typescript
// src/routes/zoho-callback.ts

import { Router, Request, Response } from 'express';
import { ZohoAuthConfig } from '../zoho/types/auth';
import { exchangeCodeForTokens } from '../zoho/auth/token-exchange';
import { TokenStore } from '../zoho/auth/token-store';

export function createZohoCallbackRouter(
  config: ZohoAuthConfig,
  tokenStore: TokenStore
): Router {
  const router = Router();

  router.get('/zoho/callback', async (req: Request, res: Response) => {
    const { code, state, error } = req.query as Record<string, string>;

    if (error) {
      return res.status(400).json({ error: `OAuth error: ${error}` });
    }

    if (!code || !state) {
      return res.status(400).json({ error: 'Missing code or state' });
    }

    // Validate state: format is "merchantId:randomHex"
    const [merchantId] = state.split(':');
    if (!merchantId) {
      return res.status(400).json({ error: 'Invalid state parameter' });
    }

    try {
      const tokenSet = await exchangeCodeForTokens(config, code);
      await tokenStore.save(merchantId, tokenSet);

      res.json({
        success: true,
        message: 'Zoho Inventory connected successfully',
        organizationId: tokenSet.organizationId,
      });
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Unknown error';
      res.status(500).json({ error: `Token exchange failed: ${message}` });
    }
  });

  return router;
}
```

---

## 3. Core API Integration Patterns

### 3.0 Base HTTP Client

All module clients extend this base client. It handles auth, org ID injection, and error normalization.

```typescript
// src/zoho/client/base-client.ts

import axios, { AxiosInstance, AxiosRequestConfig, AxiosError } from 'axios';
import { ZohoTokenSet } from '../types/auth';
import { getApiBaseUrl } from '../auth/dc-config';
import { ZohoAuthMiddleware } from '../auth/middleware';

export interface ZohoApiError {
  code: number;
  message: string;
  httpStatus: number;
}

export class ZohoBaseClient {
  private axiosInstance: AxiosInstance;

  constructor(
    private merchantId: string,
    private authMiddleware: ZohoAuthMiddleware,
    private tokenSet: ZohoTokenSet
  ) {
    this.axiosInstance = axios.create({
      baseURL: getApiBaseUrl(tokenSet.dataCenter),
      timeout: 30000,
    });
  }

  /**
   * Builds query params that ALL Zoho Inventory requests require.
   */
  private baseParams(): Record<string, string> {
    return { organization_id: this.tokenSet.organizationId };
  }

  async get<T>(path: string, params: Record<string, unknown> = {}): Promise<T> {
    return this.request<T>({ method: 'GET', url: path, params: { ...this.baseParams(), ...params } });
  }

  async post<T>(path: string, data: unknown, params: Record<string, unknown> = {}): Promise<T> {
    return this.request<T>({ method: 'POST', url: path, data, params: { ...this.baseParams(), ...params } });
  }

  async put<T>(path: string, data: unknown, params: Record<string, unknown> = {}): Promise<T> {
    return this.request<T>({ method: 'PUT', url: path, data, params: { ...this.baseParams(), ...params } });
  }

  async delete<T>(path: string, params: Record<string, unknown> = {}): Promise<T> {
    return this.request<T>({ method: 'DELETE', url: path, params: { ...this.baseParams(), ...params } });
  }

  private async request<T>(config: AxiosRequestConfig, retryCount = 0): Promise<T> {
    const authHeader = await this.authMiddleware.getAuthHeader(this.merchantId);

    try {
      const response = await this.axiosInstance.request<{ code: number; message: string } & T>({
        ...config,
        headers: {
          ...config.headers,
          Authorization: authHeader,
          'Content-Type': 'application/json',
        },
      });

      // Zoho wraps all responses: code 0 = success, non-zero = error
      if (response.data.code !== 0) {
        const err: ZohoApiError = {
          code: response.data.code,
          message: response.data.message,
          httpStatus: response.status,
        };
        throw err;
      }

      return response.data;
    } catch (error) {
      if (axios.isAxiosError(error)) {
        return this.handleAxiosError<T>(error, config, retryCount);
      }
      throw error;
    }
  }

  private async handleAxiosError<T>(
    error: AxiosError,
    config: AxiosRequestConfig,
    retryCount: number
  ): Promise<T> {
    const status = error.response?.status;

    if (status === 401 && retryCount === 0) {
      // Token might have just expired between check and call — retry once
      return this.request<T>(config, 1);
    }

    if (status === 429 && retryCount < 5) {
      // Rate limited — exponential backoff
      const delay = Math.pow(2, retryCount) * 1000;
      await sleep(delay);
      return this.request<T>(config, retryCount + 1);
    }

    if (status === 500 && retryCount < 3) {
      await sleep(5000);
      return this.request<T>(config, retryCount + 1);
    }

    const zohoError: ZohoApiError = {
      code: (error.response?.data as any)?.code ?? status ?? 0,
      message: (error.response?.data as any)?.message ?? error.message,
      httpStatus: status ?? 0,
    };
    throw zohoError;
  }
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

---

### 3.1 Items Sync (Shopify Products ↔ Zoho Items)

**Endpoints:**
- `POST /items` — Create item
- `GET /items` — List all items
- `GET /items/{item_id}` — Retrieve item
- `PUT /items/{item_id}` — Update item
- `DELETE /items/{item_id}` — Delete item
- `POST /items/{item_id}/active` — Mark active
- `POST /items/{item_id}/inactive` — Mark inactive

```typescript
// src/zoho/modules/items.ts

import { ZohoBaseClient } from '../client/base-client';

export interface ZohoItemTaxPreference {
  tax_id: number;
  tax_specification: 'intra' | 'inter';
}

export interface ZohoItemLocation {
  location_id: string;
  location_name?: string;
  is_primary?: boolean;
  initial_stock?: number;
  initial_stock_rate?: number;
  location_stock_on_hand?: string;
  location_available_stock?: string;
  location_actual_available_stock?: string;
}

export interface ZohoItemCreateRequest {
  name: string;                                    // REQUIRED
  unit?: string;                                   // e.g. "qty", "kg"
  item_type?: 'inventory' | 'sales' | 'purchases' | 'sales_and_purchases';
  product_type?: 'goods' | 'service';
  is_taxable?: boolean;
  tax_id?: number;
  description?: string;
  purchase_description?: string;
  rate?: number;                                   // Sales price
  purchase_rate?: number;                          // Purchase/cost price
  reorder_level?: number;
  vendor_id?: number;
  purchase_account_id?: number;
  inventory_account_id?: number;
  sku?: string;
  upc?: number;
  ean?: number;
  isbn?: string;
  part_number?: string;
  group_id?: string;
  group_name?: string;
  attribute_name1?: string;
  attribute_option_name1?: string;
  locations?: ZohoItemLocation[];
  // India GST fields
  hsn_or_sac?: string;                            // HSN for goods, SAC for services
  item_tax_preferences?: ZohoItemTaxPreference[];
  custom_fields?: Array<{ customfield_id: string; value: string }>;
}

export interface ZohoItem extends ZohoItemCreateRequest {
  item_id: number;
  status: 'active' | 'inactive';
  tax_name?: string;
  tax_percentage?: number;
  vendor_name?: string;
  account_name?: string;
  created_time: string;
  last_modified_time: string;
}

export interface ZohoItemListResponse {
  code: number;
  message: string;
  items: ZohoItem[];
  page_context: {
    page: number;
    per_page: number;
    has_more_page: boolean;
  };
}

export interface ZohoItemResponse {
  code: number;
  message: string;
  item: ZohoItem;
}

export class ZohoItemsClient {
  constructor(private client: ZohoBaseClient) {}

  async create(data: ZohoItemCreateRequest): Promise<ZohoItem> {
    const response = await this.client.post<ZohoItemResponse>('/items', data);
    return response.item;
  }

  async list(params: {
    page?: number;
    per_page?: number;
    last_modified_time?: string; // ISO datetime for incremental sync
    filter_by?: string;
  } = {}): Promise<ZohoItemListResponse> {
    return this.client.get<ZohoItemListResponse>('/items', params);
  }

  async retrieve(itemId: string): Promise<ZohoItem> {
    const response = await this.client.get<ZohoItemResponse>(`/items/${itemId}`);
    return response.item;
  }

  async update(itemId: string, data: Partial<ZohoItemCreateRequest>): Promise<ZohoItem> {
    const response = await this.client.put<ZohoItemResponse>(`/items/${itemId}`, data);
    return response.item;
  }

  async delete(itemId: string): Promise<void> {
    await this.client.delete(`/items/${itemId}`);
  }

  async markActive(itemId: string): Promise<void> {
    await this.client.post(`/items/${itemId}/active`, {});
  }

  async markInactive(itemId: string): Promise<void> {
    await this.client.post(`/items/${itemId}/inactive`, {});
  }
}

// ---- Shopify → Zoho Item Mapper ----

export interface ShopifyProduct {
  id: number;
  title: string;
  body_html: string;
  vendor: string;
  product_type: string;
  variants: Array<{
    id: number;
    sku: string;
    price: string;
    compare_at_price: string | null;
    inventory_quantity: number;
    weight: number;
    weight_unit: string;
  }>;
  tags: string; // comma-separated, e.g. "hsn:85423100,gst:18"
}

/**
 * Maps a Shopify product variant to a Zoho Inventory item create request.
 * Extracts HSN code from Shopify tags using format "hsn:XXXXXXXX"
 */
export function mapShopifyProductToZohoItem(
  product: ShopifyProduct,
  variant: ShopifyProduct['variants'][0],
  opts: {
    taxId?: number;
    purchaseAccountId?: number;
    inventoryAccountId?: number;
    locationId?: string;
  } = {}
): ZohoItemCreateRequest {
  // Extract HSN code from Shopify tags
  const tags = product.tags.split(',').map(t => t.trim());
  const hsnTag = tags.find(t => t.startsWith('hsn:'));
  const hsnCode = hsnTag ? hsnTag.replace('hsn:', '') : undefined;

  const name = product.variants.length > 1
    ? `${product.title} - ${variant.sku}`
    : product.title;

  const item: ZohoItemCreateRequest = {
    name,
    unit: 'qty',
    item_type: 'inventory',
    product_type: 'goods',
    is_taxable: true,
    rate: parseFloat(variant.price),
    purchase_rate: parseFloat(variant.compare_at_price ?? variant.price) * 0.6, // estimate if no cost
    sku: variant.sku || `SHOPIFY-${variant.id}`,
    description: product.body_html?.replace(/<[^>]*>/g, '').slice(0, 500),
    reorder_level: 5, // default — configure per merchant
  };

  if (opts.taxId) item.tax_id = opts.taxId;
  if (opts.purchaseAccountId) item.purchase_account_id = opts.purchaseAccountId;
  if (opts.inventoryAccountId) item.inventory_account_id = opts.inventoryAccountId;

  // India GST fields
  if (hsnCode) {
    item.hsn_or_sac = hsnCode;
    item.item_tax_preferences = [
      { tax_id: opts.taxId!, tax_specification: 'intra' }, // intra for same-state sales
      { tax_id: opts.taxId!, tax_specification: 'inter' }, // inter for cross-state
    ];
  }

  // Set initial stock per location if provided
  if (opts.locationId && variant.inventory_quantity > 0) {
    item.locations = [{
      location_id: opts.locationId,
      initial_stock: variant.inventory_quantity,
      initial_stock_rate: item.purchase_rate ?? 0,
    }];
  }

  return item;
}
```

---

### 3.2 Sales Orders (Shopify Orders → Zoho Sales Orders)

**Endpoints:**
- `POST /salesorders` — Create
- `GET /salesorders` — List
- `GET /salesorders/{salesorder_id}` — Retrieve
- `PUT /salesorders/{salesorder_id}` — Update
- `POST /salesorders/{salesorder_id}/status/confirmed` — Confirm
- `POST /salesorders/{salesorder_id}/status/void` — Void

```typescript
// src/zoho/modules/sales-orders.ts

import { ZohoBaseClient } from '../client/base-client';

export interface ZohoAddress {
  address: string;
  city: string;
  state: string;
  zip: string;
  country: string;
  phone?: string;
}

export interface ZohoSalesOrderLineItem {
  item_id?: number;
  name: string;
  rate: number;
  quantity: number;
  unit?: string;
  tax_id?: number;
  discount?: number;
  location_id?: string;
  // India GST
  hsn_or_sac?: string;
  // Read-only fields returned by API
  line_item_id?: number;
  item_total?: number;
  quantity_invoiced?: number;
  quantity_packed?: number;
  quantity_shipped?: number;
}

export interface ZohoSalesOrderCreateRequest {
  customer_id: number;                  // REQUIRED
  salesorder_number: string;            // REQUIRED (or omit + use auto-numbering)
  date?: string;                        // yyyy-mm-dd
  shipment_date?: string;
  reference_number?: string;            // External order ID (Shopify order number)
  line_items: ZohoSalesOrderLineItem[]; // REQUIRED
  billing_address?: ZohoAddress;
  shipping_address?: ZohoAddress;
  delivery_method?: string;
  discount?: number;
  is_discount_before_tax?: boolean;
  shipping_charge?: number;
  adjustment?: number;
  adjustment_description?: string;
  notes?: string;
  location_id?: string;
  // India GST fields
  gst_no?: string;                      // Customer GSTIN (15 chars)
  gst_treatment?: 'business_gst' | 'business_none' | 'overseas' | 'consumer';
  place_of_supply?: string;             // 2-char state code, e.g. "TN"
  is_pre_gst?: boolean;
}

export interface ZohoSalesOrder extends ZohoSalesOrderCreateRequest {
  salesorder_id: number;
  status: 'draft' | 'confirmed' | 'fulfilled' | 'void';
  sub_total: number;
  tax_total: number;
  total: number;
  created_time: string;
  last_modified_time: string;
}

export interface ZohoSalesOrderResponse {
  code: number;
  message: string;
  salesorder: ZohoSalesOrder;
}

export interface ZohoSalesOrderListResponse {
  code: number;
  message: string;
  salesorders: ZohoSalesOrder[];
  page_context: { page: number; per_page: number; has_more_page: boolean };
}

export class ZohoSalesOrdersClient {
  constructor(private client: ZohoBaseClient) {}

  async create(
    data: ZohoSalesOrderCreateRequest,
    ignoreAutoNumber = false
  ): Promise<ZohoSalesOrder> {
    const response = await this.client.post<ZohoSalesOrderResponse>(
      '/salesorders',
      data,
      ignoreAutoNumber ? { ignore_auto_number_generation: true } : {}
    );
    return response.salesorder;
  }

  async list(params: {
    page?: number;
    per_page?: number;
    status?: string;
    last_modified_time?: string;
    reference_number?: string;
  } = {}): Promise<ZohoSalesOrderListResponse> {
    return this.client.get<ZohoSalesOrderListResponse>('/salesorders', params);
  }

  async retrieve(salesOrderId: string): Promise<ZohoSalesOrder> {
    const response = await this.client.get<ZohoSalesOrderResponse>(`/salesorders/${salesOrderId}`);
    return response.salesorder;
  }

  async update(salesOrderId: string, data: Partial<ZohoSalesOrderCreateRequest>): Promise<ZohoSalesOrder> {
    const response = await this.client.put<ZohoSalesOrderResponse>(`/salesorders/${salesOrderId}`, data);
    return response.salesorder;
  }

  async confirm(salesOrderId: string): Promise<void> {
    await this.client.post(`/salesorders/${salesOrderId}/status/confirmed`, {});
  }

  async void(salesOrderId: string): Promise<void> {
    await this.client.post(`/salesorders/${salesOrderId}/status/void`, {});
  }
}

// ---- Shopify Order → Zoho Sales Order Mapper ----

export interface ShopifyOrder {
  id: number;
  order_number: number;
  name: string; // "#1001"
  created_at: string;
  processed_at: string;
  customer: {
    id: number;
    email: string;
    first_name: string;
    last_name: string;
    default_address?: {
      address1: string;
      city: string;
      province_code: string;
      zip: string;
      country_code: string;
      phone: string;
    };
  };
  line_items: Array<{
    id: number;
    variant_id: number;
    sku: string;
    title: string;
    quantity: number;
    price: string;
    tax_lines: Array<{ title: string; rate: number; price: string }>;
  }>;
  billing_address: ShopifyOrder['customer']['default_address'];
  shipping_address: ShopifyOrder['customer']['default_address'];
  shipping_lines: Array<{ price: string }>;
  total_price: string;
  subtotal_price: string;
  total_tax: string;
  note?: string;
  note_attributes: Array<{ name: string; value: string }>;
}

/**
 * Maps a Shopify order to a Zoho Sales Order.
 * Requires the Zoho customer_id (looked up from contacts by email beforehand).
 * Requires zohoItemIdMap: { [shopifyVariantId]: zohoItemId }
 */
export function mapShopifyOrderToZohoSalesOrder(
  order: ShopifyOrder,
  zohoCustomerId: number,
  zohoItemIdMap: Record<number, number>,
  opts: {
    taxId?: number;
    placeOfSupply?: string;       // e.g. "TN" for Tamil Nadu
    gstTreatment?: ZohoSalesOrderCreateRequest['gst_treatment'];
    gstNo?: string;               // Customer GSTIN
    locationId?: string;
    hsnMap?: Record<string, string>; // { [sku]: hsnCode }
  } = {}
): ZohoSalesOrderCreateRequest {
  const shippingCharge = order.shipping_lines.reduce(
    (sum, line) => sum + parseFloat(line.price), 0
  );

  const lineItems: ZohoSalesOrderLineItem[] = order.line_items.map(item => ({
    item_id: zohoItemIdMap[item.variant_id],
    name: item.title,
    rate: parseFloat(item.price),
    quantity: item.quantity,
    unit: 'qty',
    tax_id: opts.taxId,
    hsn_or_sac: opts.hsnMap?.[item.sku],
    location_id: opts.locationId,
  }));

  const billingAddr = order.billing_address;
  const shippingAddr = order.shipping_address;

  const so: ZohoSalesOrderCreateRequest = {
    customer_id: zohoCustomerId,
    salesorder_number: `SHOP-${order.order_number}`,
    date: order.processed_at.split('T')[0],
    reference_number: order.name, // "#1001"
    line_items: lineItems,
    shipping_charge: shippingCharge > 0 ? shippingCharge : undefined,
    notes: order.note,
  };

  if (billingAddr) {
    so.billing_address = {
      address: billingAddr.address1,
      city: billingAddr.city,
      state: billingAddr.province_code,
      zip: billingAddr.zip,
      country: billingAddr.country_code,
      phone: billingAddr.phone,
    };
  }

  if (shippingAddr) {
    so.shipping_address = {
      address: shippingAddr.address1,
      city: shippingAddr.city,
      state: shippingAddr.province_code,
      zip: shippingAddr.zip,
      country: shippingAddr.country_code,
      phone: shippingAddr.phone,
    };
  }

  // India GST fields
  if (opts.placeOfSupply) so.place_of_supply = opts.placeOfSupply;
  if (opts.gstTreatment) so.gst_treatment = opts.gstTreatment;
  if (opts.gstNo) so.gst_no = opts.gstNo;

  return so;
}
```

---

### 3.3 Invoices (From Fulfilled Shopify Orders)

**Endpoints:**
- `POST /invoices` — Create
- `GET /invoices/{invoice_id}` — Retrieve
- `PUT /invoices/{invoice_id}` — Update
- `POST /invoices/{invoice_id}/status/sent` — Mark sent
- `POST /invoices/{invoice_id}/status/void` — Void

```typescript
// src/zoho/modules/invoices.ts

import { ZohoBaseClient } from '../client/base-client';

export interface ZohoInvoiceLineItem {
  item_id?: string;
  name: string;
  rate: number;
  quantity: number;
  unit?: string;
  tax_id?: string;
  discount?: number;
  location_id?: string;
  hsn_or_sac?: string;            // India only
}

export interface ZohoInvoiceCreateRequest {
  customer_id: string;            // REQUIRED
  line_items: ZohoInvoiceLineItem[]; // REQUIRED
  invoice_number?: string;
  date?: string;                  // yyyy-mm-dd
  due_date?: string;
  payment_terms?: number;         // Days, e.g. 15, 30
  salesorder_id?: string;         // Link to source sales order
  reference_number?: string;
  shipping_charge?: number;
  adjustment?: number;
  notes?: string;
  // India GST fields
  gst_no?: string;
  gst_treatment?: 'business_gst' | 'business_none' | 'overseas' | 'consumer';
  place_of_supply?: string;
  is_pre_gst?: boolean;
}

export interface ZohoInvoice extends ZohoInvoiceCreateRequest {
  invoice_id: string;
  invoice_number: string;
  status: 'sent' | 'draft' | 'overdue' | 'paid' | 'void' | 'unpaid' | 'partially_paid' | 'viewed';
  sub_total: number;
  tax_total: number;
  total: string;
  balance: string;
  payment_made: number;
  invoice_url: string;
  created_time: string;
  last_modified_time: string;
}

export interface ZohoInvoiceResponse {
  code: number;
  message: string;
  invoice: ZohoInvoice;
}

export class ZohoInvoicesClient {
  constructor(private client: ZohoBaseClient) {}

  async create(data: ZohoInvoiceCreateRequest): Promise<ZohoInvoice> {
    const response = await this.client.post<ZohoInvoiceResponse>('/invoices', data);
    return response.invoice;
  }

  async retrieve(invoiceId: string): Promise<ZohoInvoice> {
    const response = await this.client.get<ZohoInvoiceResponse>(`/invoices/${invoiceId}`);
    return response.invoice;
  }

  async update(invoiceId: string, data: Partial<ZohoInvoiceCreateRequest>): Promise<ZohoInvoice> {
    const response = await this.client.put<ZohoInvoiceResponse>(`/invoices/${invoiceId}`, data);
    return response.invoice;
  }

  async markSent(invoiceId: string): Promise<void> {
    await this.client.post(`/invoices/${invoiceId}/status/sent`, {});
  }

  async void(invoiceId: string): Promise<void> {
    await this.client.post(`/invoices/${invoiceId}/status/void`, {});
  }

  async emailInvoice(invoiceId: string, toEmail: string): Promise<void> {
    await this.client.post(`/invoices/${invoiceId}/email`, {
      to_mail_ids: [toEmail],
      send_from_org_email_id: true,
    });
  }
}

/**
 * Creates a Zoho Invoice from a fulfilled Shopify order.
 * Call this when Shopify fires the fulfillments/create webhook.
 */
export function mapShopifyOrderToZohoInvoice(
  order: {
    id: number;
    order_number: number;
    line_items: Array<{
      variant_id: number;
      title: string;
      quantity: number;
      price: string;
      sku: string;
    }>;
    shipping_lines: Array<{ price: string }>;
    note?: string;
  },
  zohoCustomerId: string,
  salesOrderId: string,
  opts: {
    taxId?: string;
    placeOfSupply?: string;
    gstTreatment?: ZohoInvoiceCreateRequest['gst_treatment'];
    gstNo?: string;
    hsnMap?: Record<string, string>;
    zohoItemIdMap?: Record<number, string>;
    locationId?: string;
    paymentTermsDays?: number;
  } = {}
): ZohoInvoiceCreateRequest {
  const shippingCharge = order.shipping_lines.reduce(
    (sum, l) => sum + parseFloat(l.price), 0
  );

  return {
    customer_id: zohoCustomerId,
    salesorder_id: salesOrderId,
    invoice_number: `INV-SHOP-${order.order_number}`,
    date: new Date().toISOString().split('T')[0],
    payment_terms: opts.paymentTermsDays ?? 0, // 0 = due on receipt
    reference_number: `#${order.order_number}`,
    shipping_charge: shippingCharge > 0 ? shippingCharge : undefined,
    notes: order.note,
    line_items: order.line_items.map(item => ({
      item_id: opts.zohoItemIdMap?.[item.variant_id],
      name: item.title,
      rate: parseFloat(item.price),
      quantity: item.quantity,
      unit: 'qty',
      tax_id: opts.taxId,
      hsn_or_sac: opts.hsnMap?.[item.sku],
      location_id: opts.locationId,
    })),
    gst_no: opts.gstNo,
    gst_treatment: opts.gstTreatment,
    place_of_supply: opts.placeOfSupply,
  };
}
```

---

### 3.4 Purchase Orders (Restocking)

**Endpoints:**
- `POST /purchaseorders` — Create
- `GET /purchaseorders/{purchaseorder_id}` — Retrieve
- `PUT /purchaseorders/{purchaseorder_id}` — Update
- `POST /purchaseorders/{purchaseorder_id}/status/issued` — Issue (send to vendor)
- `POST /purchaseorders/{purchaseorder_id}/status/cancelled` — Cancel

```typescript
// src/zoho/modules/purchase-orders.ts

import { ZohoBaseClient } from '../client/base-client';

export interface ZohoPurchaseOrderLineItem {
  item_id: number;
  name?: string;
  rate?: number;                     // Purchase rate
  quantity: number;
  unit?: string;
  location_id?: string;              // Destination warehouse
  // India GST
  hsn_or_sac?: string;
  tax_id?: number;
  tax_exemption_code?: string;
  // Reverse charge (India)
  reverse_charge_tax_id?: string;
  reverse_charge_tax_name?: string;
  reverse_charge_tax_percentage?: number;
  reverse_charge_tax_amount?: number;
}

export interface ZohoPurchaseOrderCreateRequest {
  vendor_id: number;                 // REQUIRED
  purchaseorder_number?: string;     // auto-generated if omitted
  date?: string;
  expected_delivery_date?: string;
  reference_number?: string;
  line_items: ZohoPurchaseOrderLineItem[]; // REQUIRED
  delivery_address?: {
    address: string;
    city: string;
    state: string;
    zip: string;
    country: string;
  };
  is_drop_shipment?: boolean;
  salesorder_id?: number;            // For drop-ship orders
  notes?: string;
  // India GST fields
  gst_no?: string;                   // Vendor GSTIN
  gst_treatment?: 'business_gst' | 'business_none' | 'overseas' | 'consumer';
  source_of_supply?: string;         // Origin state code
  destination_of_supply?: string;    // Destination state code
  is_pre_gst?: boolean;
  is_reverse_charge_applied?: boolean;
}

export interface ZohoPurchaseOrder extends ZohoPurchaseOrderCreateRequest {
  purchaseorder_id: number;
  status: 'draft' | 'issued' | 'Partially_Received' | 'received' | 'cancelled';
  sub_total: number;
  tax_total: number;
  total: number;
  purchasereceives: Array<{ receive_id: string; date: string; status: string }>;
  bills: Array<{ bill_id: string; bill_number: string; status: string }>;
  created_time: string;
  last_modified_time: string;
}

export interface ZohoPurchaseOrderResponse {
  code: number;
  message: string;
  purchaseorder: ZohoPurchaseOrder;
}

export class ZohoPurchaseOrdersClient {
  constructor(private client: ZohoBaseClient) {}

  async create(data: ZohoPurchaseOrderCreateRequest): Promise<ZohoPurchaseOrder> {
    const response = await this.client.post<ZohoPurchaseOrderResponse>('/purchaseorders', data);
    return response.purchaseorder;
  }

  async retrieve(poId: string): Promise<ZohoPurchaseOrder> {
    const response = await this.client.get<ZohoPurchaseOrderResponse>(`/purchaseorders/${poId}`);
    return response.purchaseorder;
  }

  async update(poId: string, data: Partial<ZohoPurchaseOrderCreateRequest>): Promise<ZohoPurchaseOrder> {
    const response = await this.client.put<ZohoPurchaseOrderResponse>(`/purchaseorders/${poId}`, data);
    return response.purchaseorder;
  }

  async issue(poId: string): Promise<void> {
    await this.client.post(`/purchaseorders/${poId}/status/issued`, {});
  }

  async cancel(poId: string): Promise<void> {
    await this.client.post(`/purchaseorders/${poId}/status/cancelled`, {});
  }
}
```

---

### 3.5 Inventory Adjustments (Stock Corrections)

**Endpoints:**
- `POST /inventoryadjustments` — Create
- `GET /inventoryadjustments` — List
- `GET /inventoryadjustments/{adjustment_id}` — Retrieve
- `PUT /inventoryadjustments/{adjustment_id}` — Update
- `DELETE /inventoryadjustments/{adjustment_id}` — Delete

> **Note**: The `/inventoryadjustments` endpoint was not live during API research (returned 404). The endpoint pattern is confirmed by scope documentation. Verify against the live API and check if a plan upgrade (Standard+) is required.

```typescript
// src/zoho/modules/inventory-adjustments.ts

import { ZohoBaseClient } from '../client/base-client';

export interface ZohoAdjustmentLineItem {
  item_id: number;
  name?: string;
  quantity_adjusted: number;         // Positive = increase, negative = decrease
  purchase_rate?: number;
  location_id?: string;
}

export interface ZohoInventoryAdjustmentCreateRequest {
  date: string;                      // yyyy-mm-dd
  reason?: string;                   // e.g. "Damaged goods write-off", "Cycle count correction"
  reference_number?: string;
  location_id?: string;              // Warehouse being adjusted
  line_items: ZohoAdjustmentLineItem[];
  account_id?: number;               // GL account for the adjustment
  description?: string;
}

export interface ZohoInventoryAdjustment extends ZohoInventoryAdjustmentCreateRequest {
  adjustment_id: string;
  adjustment_type: 'quantity' | 'value';
  status: string;
  created_time: string;
  last_modified_time: string;
}

export interface ZohoInventoryAdjustmentResponse {
  code: number;
  message: string;
  inventoryadjustment: ZohoInventoryAdjustment;
}

export class ZohoInventoryAdjustmentsClient {
  constructor(private client: ZohoBaseClient) {}

  async create(data: ZohoInventoryAdjustmentCreateRequest): Promise<ZohoInventoryAdjustment> {
    const response = await this.client.post<ZohoInventoryAdjustmentResponse>(
      '/inventoryadjustments',
      data
    );
    return response.inventoryadjustment;
  }

  async retrieve(adjustmentId: string): Promise<ZohoInventoryAdjustment> {
    const response = await this.client.get<ZohoInventoryAdjustmentResponse>(
      `/inventoryadjustments/${adjustmentId}`
    );
    return response.inventoryadjustment;
  }

  async list(params: { page?: number; per_page?: number } = {}): Promise<{
    inventoryadjustments: ZohoInventoryAdjustment[];
    page_context: { page: number; per_page: number; has_more_page: boolean };
  }> {
    return this.client.get('/inventoryadjustments', params);
  }

  /**
   * Convenience: adjust a single item's stock at a warehouse.
   * Positive delta = add stock, negative delta = remove stock.
   */
  async adjustSingleItem(opts: {
    itemId: number;
    quantityDelta: number;
    locationId?: string;
    reason: string;
    referenceNumber?: string;
    purchaseRate?: number;
  }): Promise<ZohoInventoryAdjustment> {
    return this.create({
      date: new Date().toISOString().split('T')[0],
      reason: opts.reason,
      reference_number: opts.referenceNumber,
      location_id: opts.locationId,
      line_items: [{
        item_id: opts.itemId,
        quantity_adjusted: opts.quantityDelta,
        location_id: opts.locationId,
        purchase_rate: opts.purchaseRate,
      }],
    });
  }
}
```

---

### 3.6 Transfer Orders (Multi-Warehouse Stock Transfers)

**Endpoints:**
- `POST /transferorders` — Create
- `GET /transferorders` — List
- `GET /transferorders/{transfer_order_id}` — Retrieve
- `PUT /transferorders/{transfer_order_id}` — Update
- `DELETE /transferorders/{transfer_order_id}` — Delete
- `POST /transferorders/{transfer_order_id}/markastransferred` — Mark as transferred/received

```typescript
// src/zoho/modules/transfer-orders.ts

import { ZohoBaseClient } from '../client/base-client';

export interface ZohoTransferOrderLineItem {
  item_id: number;
  name: string;
  quantity_transfer: number;
  unit?: string;
  description?: string;
}

export interface ZohoTransferOrderCreateRequest {
  from_location_id: string;          // Source warehouse location ID
  to_location_id: string;            // Destination warehouse location ID
  date: string;                      // yyyy-mm-dd
  transfer_order_number?: string;    // Auto-generated if omitted
  line_items: ZohoTransferOrderLineItem[];
  is_intransit_order?: boolean;      // true = two-step (in-transit), false = immediate
  notes?: string;
  reference_number?: string;
}

export interface ZohoTransferOrder extends ZohoTransferOrderCreateRequest {
  transfer_order_id: string;
  status: 'draft' | 'in_transit' | 'transferred';
  created_time: string;
  last_modified_time: string;
}

export interface ZohoTransferOrderResponse {
  code: number;
  message: string;
  transfer_order: ZohoTransferOrder;
}

export class ZohoTransferOrdersClient {
  constructor(private client: ZohoBaseClient) {}

  async create(data: ZohoTransferOrderCreateRequest): Promise<ZohoTransferOrder> {
    const response = await this.client.post<ZohoTransferOrderResponse>('/transferorders', data);
    return response.transfer_order;
  }

  async retrieve(transferOrderId: string): Promise<ZohoTransferOrder> {
    const response = await this.client.get<ZohoTransferOrderResponse>(`/transferorders/${transferOrderId}`);
    return response.transfer_order;
  }

  async update(transferOrderId: string, data: Partial<ZohoTransferOrderCreateRequest>): Promise<ZohoTransferOrder> {
    const response = await this.client.put<ZohoTransferOrderResponse>(`/transferorders/${transferOrderId}`, data);
    return response.transfer_order;
  }

  async markAsTransferred(transferOrderId: string): Promise<void> {
    await this.client.post(`/transferorders/${transferOrderId}/markastransferred`, {});
  }

  async delete(transferOrderId: string): Promise<void> {
    await this.client.delete(`/transferorders/${transferOrderId}`);
  }

  async list(params: { page?: number; per_page?: number } = {}): Promise<{
    transfer_orders: ZohoTransferOrder[];
    page_context: { page: number; per_page: number; has_more_page: boolean };
  }> {
    return this.client.get('/transferorders', params);
  }
}
```

---

### 3.7 Packages & Shipments (Fulfillment Tracking)

**Package Endpoints:**
- `POST /packages?salesorder_id={id}` — Create package (salesorder_id required as query param)
- `GET /packages/{package_id}` — Retrieve
- `PUT /packages/{package_id}` — Update

**Shipment Endpoints:**
- `POST /shipmentorders?salesorder_id={id}` — Create shipment
- `GET /shipmentorders/{shipmentorder_id}` — Retrieve
- `POST /shipmentorders/{shipmentorder_id}/status/delivered` — Mark delivered

```typescript
// src/zoho/modules/packages-shipments.ts

import { ZohoBaseClient } from '../client/base-client';

// --- Packages ---

export interface ZohoPackageLineItem {
  so_line_item_id: number;           // Line item ID from the Sales Order
  quantity: number;
}

export interface ZohoPackageCreateRequest {
  date: string;                      // yyyy-mm-dd
  line_items: ZohoPackageLineItem[];
  package_number?: string;
  notes?: string;
}

export interface ZohoPackage extends ZohoPackageCreateRequest {
  package_id: number;
  salesorder_id: number;
  status: string;
  created_time: string;
}

export interface ZohoPackageResponse {
  code: number;
  message: string;
  package: ZohoPackage;
}

// --- Shipment Orders ---

export interface ZohoShipmentOrderCreateRequest {
  shipment_number: string;           // REQUIRED
  date: string;                      // REQUIRED
  delivery_method: string;           // REQUIRED — carrier name, e.g. "FedEx", "Delhivery"
  tracking_number: string;           // REQUIRED
  notes?: string;
}

export interface ZohoShipmentOrder extends ZohoShipmentOrderCreateRequest {
  shipmentorder_id: number;
  salesorder_id: number;
  package_ids: number[];
  carrier: string;
  status: 'shipped' | 'delivered';
  detailed_status?: string;
  created_time: string;
}

export interface ZohoShipmentOrderResponse {
  code: number;
  message: string;
  shipmentorder: ZohoShipmentOrder;
}

export class ZohoPackagesClient {
  constructor(private client: ZohoBaseClient) {}

  async create(salesOrderId: string, data: ZohoPackageCreateRequest): Promise<ZohoPackage> {
    const response = await this.client.post<ZohoPackageResponse>(
      '/packages',
      data,
      { salesorder_id: salesOrderId }     // Must be in query params, not body
    );
    return response.package;
  }

  async retrieve(packageId: string): Promise<ZohoPackage> {
    const response = await this.client.get<ZohoPackageResponse>(`/packages/${packageId}`);
    return response.package;
  }

  async update(packageId: string, data: Partial<ZohoPackageCreateRequest>): Promise<ZohoPackage> {
    const response = await this.client.put<ZohoPackageResponse>(`/packages/${packageId}`, data);
    return response.package;
  }
}

export class ZohoShipmentsClient {
  constructor(private client: ZohoBaseClient) {}

  async create(
    salesOrderId: string,
    packageIds: string[],
    data: ZohoShipmentOrderCreateRequest
  ): Promise<ZohoShipmentOrder> {
    const response = await this.client.post<ZohoShipmentOrderResponse>(
      '/shipmentorders',
      data,
      {
        salesorder_id: salesOrderId,
        package_ids: packageIds.join(','),
      }
    );
    return response.shipmentorder;
  }

  async retrieve(shipmentId: string): Promise<ZohoShipmentOrder> {
    const response = await this.client.get<ZohoShipmentOrderResponse>(`/shipmentorders/${shipmentId}`);
    return response.shipmentorder;
  }

  async markDelivered(shipmentId: string): Promise<void> {
    await this.client.post(`/shipmentorders/${shipmentId}/status/delivered`, {});
  }
}

// ---- Shopify Fulfillment → Zoho Package + Shipment ----

export interface ShopifyFulfillment {
  id: number;
  order_id: number;
  status: string;
  tracking_company: string;
  tracking_number: string;
  tracking_url: string;
  line_items: Array<{
    id: number;
    variant_id: number;
    quantity: number;
  }>;
  created_at: string;
}

/**
 * Full flow: Shopify fulfillment → Zoho Package → Zoho Shipment
 * soLineItemIdMap: { [shopifyLineItemId]: zohoSoLineItemId }
 */
export async function createPackageAndShipment(
  packagesClient: ZohoPackagesClient,
  shipmentsClient: ZohoShipmentsClient,
  fulfillment: ShopifyFulfillment,
  zohoSalesOrderId: string,
  soLineItemIdMap: Record<number, number>
): Promise<{ package: ZohoPackage; shipment: ZohoShipmentOrder }> {
  // Step 1: Create the package
  const packageData: ZohoPackageCreateRequest = {
    date: fulfillment.created_at.split('T')[0],
    package_number: `PKG-${fulfillment.id}`,
    line_items: fulfillment.line_items.map(li => ({
      so_line_item_id: soLineItemIdMap[li.id],
      quantity: li.quantity,
    })),
  };

  const pkg = await packagesClient.create(zohoSalesOrderId, packageData);

  // Step 2: Create the shipment order linked to this package
  const shipmentData: ZohoShipmentOrderCreateRequest = {
    shipment_number: `SHIP-${fulfillment.id}`,
    date: fulfillment.created_at.split('T')[0],
    delivery_method: fulfillment.tracking_company || 'Unknown',
    tracking_number: fulfillment.tracking_number,
  };

  const shipment = await shipmentsClient.create(
    zohoSalesOrderId,
    [String(pkg.package_id)],
    shipmentData
  );

  return { package: pkg, shipment };
}
```

---

## 4. India GST-Specific Fields

### 4.1 GST Treatment Determination Logic

```typescript
// src/zoho/gst/gst-treatment.ts

export type GstTreatment = 'business_gst' | 'business_none' | 'overseas' | 'consumer';

export interface CustomerGstProfile {
  gstin?: string;        // 15-char GSTIN, e.g. "22AAAAA0000A1Z5"
  isExport: boolean;     // true if shipping outside India
  isB2B: boolean;        // true if customer is a business (has GSTIN or GST-registered)
}

/**
 * Determines gst_treatment from customer profile.
 */
export function determineGstTreatment(profile: CustomerGstProfile): GstTreatment {
  if (profile.isExport) return 'overseas';
  if (profile.gstin && profile.isB2B) return 'business_gst';
  if (!profile.gstin && profile.isB2B) return 'business_none';
  return 'consumer'; // Default: end consumer, B2C
}

/**
 * Validates a 15-character GSTIN.
 * Format: 2-digit state code + 10-char PAN + 1 entity code + 1 char + 1 checksum
 */
export function isValidGstin(gstin: string): boolean {
  const gstinRegex = /^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}Z[0-9A-Z]{1}$/;
  return gstinRegex.test(gstin);
}
```

### 4.2 Place of Supply Determination

```typescript
// src/zoho/gst/place-of-supply.ts

// Indian state codes as per GST registration
export const INDIA_STATE_CODES: Record<string, string> = {
  'Andhra Pradesh': 'AP',
  'Arunachal Pradesh': 'AR',
  'Assam': 'AS',
  'Bihar': 'BR',
  'Chhattisgarh': 'CG',
  'Goa': 'GA',
  'Gujarat': 'GJ',
  'Haryana': 'HR',
  'Himachal Pradesh': 'HP',
  'Jharkhand': 'JH',
  'Karnataka': 'KA',
  'Kerala': 'KL',
  'Madhya Pradesh': 'MP',
  'Maharashtra': 'MH',
  'Manipur': 'MN',
  'Meghalaya': 'ML',
  'Mizoram': 'MZ',
  'Nagaland': 'NL',
  'Odisha': 'OR',
  'Punjab': 'PB',
  'Rajasthan': 'RJ',
  'Sikkim': 'SK',
  'Tamil Nadu': 'TN',
  'Telangana': 'TS',
  'Tripura': 'TR',
  'Uttar Pradesh': 'UP',
  'Uttarakhand': 'UK',
  'West Bengal': 'WB',
  'Delhi': 'DL',
  'Jammu and Kashmir': 'JK',
  'Ladakh': 'LA',
};

// Shopify province_code → Zoho state code mapping (Shopify uses ISO codes)
const SHOPIFY_TO_ZOHO_STATE: Record<string, string> = {
  'AP': 'AP', 'AR': 'AR', 'AS': 'AS', 'BR': 'BR', 'CG': 'CG',
  'GA': 'GA', 'GJ': 'GJ', 'HR': 'HR', 'HP': 'HP', 'JH': 'JH',
  'KA': 'KA', 'KL': 'KL', 'MP': 'MP', 'MH': 'MH', 'MN': 'MN',
  'ML': 'ML', 'MZ': 'MZ', 'NL': 'NL', 'OR': 'OR', 'PB': 'PB',
  'RJ': 'RJ', 'SK': 'SK', 'TN': 'TN', 'TS': 'TS', 'TR': 'TR',
  'UP': 'UP', 'UK': 'UK', 'WB': 'WB', 'DL': 'DL',
  // Shopify may use full names — normalize as needed
};

export function getPlaceOfSupply(shopifyProvinceCode: string): string | undefined {
  return SHOPIFY_TO_ZOHO_STATE[shopifyProvinceCode.toUpperCase()];
}
```

### 4.3 CGST/SGST vs IGST Determination

```typescript
// src/zoho/gst/tax-selection.ts

/**
 * Determines whether a transaction attracts intra-state or inter-state GST.
 * Intra = CGST + SGST (same state)
 * Inter = IGST (different states, or export)
 *
 * @param sellerStateCode  - 2-char state code of the seller (your org's state)
 * @param buyerStateCode   - 2-char state code from place_of_supply
 * @param isExport         - true if buyer is overseas
 */
export function determineTaxSpecification(
  sellerStateCode: string,
  buyerStateCode: string,
  isExport: boolean
): 'intra' | 'inter' {
  if (isExport) return 'inter'; // Export = IGST (or zero-rated)
  if (sellerStateCode.toUpperCase() === buyerStateCode.toUpperCase()) {
    return 'intra'; // Same state = CGST + SGST
  }
  return 'inter'; // Different state = IGST
}

/**
 * Sets item_tax_preferences for a Zoho item based on seller state.
 * Creates both intra and inter preferences so Zoho picks the right one per transaction.
 */
export function buildItemTaxPreferences(
  intraTaxId: number,
  interTaxId: number
): Array<{ tax_id: number; tax_specification: 'intra' | 'inter' }> {
  return [
    { tax_id: intraTaxId, tax_specification: 'intra' },
    { tax_id: interTaxId, tax_specification: 'inter' },
  ];
}
```

### 4.4 HSN Code Mapping

```typescript
// src/zoho/gst/hsn-mapping.ts

/**
 * HSN code lookup table — extend with your product catalog's HSN codes.
 * Shopify stores these as product tags in format "hsn:XXXXXXXX"
 */
export const DEFAULT_HSN_MAP: Record<string, string> = {
  'electronics':         '8542',    // Electronic integrated circuits
  'mobile-phones':       '8517',    // Telephone sets
  'computers':           '8471',    // Automatic data processing machines
  'clothing':            '6109',    // T-shirts, singlets
  'footwear':            '6403',    // Footwear
  'food-beverages':      '2202',    // Waters, flavoured
  'cosmetics':           '3304',    // Beauty preparations
  'books':               '4901',    // Printed books
  'furniture':           '9403',    // Other furniture
  'pharmaceuticals':     '3004',    // Medicaments
};

/**
 * Extract HSN code from Shopify product tags.
 * Tags should include "hsn:85423100" format.
 */
export function extractHsnFromTags(tags: string): string | undefined {
  const tagArray = tags.split(',').map(t => t.trim());
  const hsnTag = tagArray.find(t => t.toLowerCase().startsWith('hsn:'));
  return hsnTag ? hsnTag.split(':')[1]?.trim() : undefined;
}
```

### 4.5 Reverse Charge Handling (Purchase Orders)

```typescript
// src/zoho/gst/reverse-charge.ts

/**
 * Applies reverse charge fields to a PO line item.
 * Used when buying from unregistered vendors (RCM applies).
 *
 * Under RCM, the buyer self-assesses and pays GST directly to the government.
 */
export function applyReverseCharge(
  lineItem: {
    purchase_rate: number;
    quantity: number;
  },
  reverseChargeTaxId: string,
  reverseChargeTaxPercentage: number // e.g. 18 for 18% GST
): {
  reverse_charge_tax_id: string;
  reverse_charge_tax_name: string;
  reverse_charge_tax_percentage: number;
  reverse_charge_tax_amount: number;
} {
  const taxableAmount = lineItem.purchase_rate * lineItem.quantity;
  const taxAmount = (taxableAmount * reverseChargeTaxPercentage) / 100;

  return {
    reverse_charge_tax_id: reverseChargeTaxId,
    reverse_charge_tax_name: 'RCM GST',
    reverse_charge_tax_percentage: reverseChargeTaxPercentage,
    reverse_charge_tax_amount: Math.round(taxAmount * 100) / 100, // Round to 2 decimals
  };
}
```

---

## 5. Zoho Inventory ↔ Zoho Books Sync

### 5.1 What Syncs Automatically (Bidirectional, Real-Time)

When Zoho Books and Zoho Inventory are under the same organization:

| Entity | Sync Direction | Notes |
|---|---|---|
| Items | ↔ Both | Full two-way sync |
| Contacts | ↔ Both | Full two-way sync |
| Sales Orders | ↔ Both | Full two-way sync |
| Purchase Orders | ↔ Both | Full two-way sync |
| Invoices | ↔ Both | Full two-way sync |
| Bills | ↔ Both | Full two-way sync |
| Settings (taxes, currencies) | ↔ Both | Full two-way sync |
| COGS journal entry | Inventory → Books | Auto-created when shipment marked delivered |
| Retainer Invoices | ↔ Both | Full two-way sync |

**Key implication for the platform**: Create entities via Zoho Inventory API. They automatically appear in Zoho Books. You do NOT need to call the Zoho Books API separately for Items, Contacts, Sales Orders, Invoices, or Purchase Orders.

### 5.2 What Does NOT Auto-Sync

| Entity | Why | What to Do |
|---|---|---|
| Packages | Inventory-only concept | Manage exclusively via Inventory API |
| Shipment Orders | Inventory-only concept | Manage exclusively via Inventory API |
| Transfer Orders | Inventory-only concept | Manage exclusively via Inventory API |
| Inventory Adjustments | Inventory-only concept | Manage exclusively via Inventory API |
| Workflow rules/automations | Not shared between apps | Configure separately in each app |

### 5.3 API Endpoint Disambiguation

Both APIs access the same underlying data. Use:
- **Inventory API** (`/inventory/v1/`) for all inventory operations
- **Books API** (`/books/v3/`) for accounting-specific operations (chart of accounts, journal entries, bank reconciliation, GST returns)

```typescript
// The same organization_id works for both APIs
const INVENTORY_BASE = 'https://www.zohoapis.in/inventory/v1';
const BOOKS_BASE = 'https://www.zohoapis.in/books/v3';

// Same access token works for both (if scopes include both)
// Add Books scopes if needed:
// ZohoBooks.fullaccess.all OR ZohoBooks.accountants.READ, etc.
```

### 5.4 Preventing Duplicate Records

Before creating any entity via API, check if it already exists. This prevents duplicates when the native Shopify-Zoho integration is also active.

```typescript
// src/zoho/utils/dedup.ts

import { ZohoItemsClient } from '../modules/items';
import { ZohoBaseClient } from '../client/base-client';

/**
 * Find existing Zoho item by SKU.
 * Returns null if not found.
 */
export async function findItemBySku(
  client: ZohoBaseClient,
  sku: string
): Promise<{ item_id: number; name: string } | null> {
  // Zoho Inventory list endpoint supports filtering by SKU via search
  const response = await client.get<{
    code: number;
    items: Array<{ item_id: number; name: string; sku: string }>;
  }>('/items', { search_text: sku });

  if (response.code !== 0 || !response.items?.length) return null;

  // Exact match on SKU
  const match = response.items.find(item => item.sku === sku);
  return match ?? null;
}

/**
 * Find existing Zoho contact by email.
 */
export async function findContactByEmail(
  client: ZohoBaseClient,
  email: string
): Promise<{ contact_id: number; contact_name: string } | null> {
  const response = await client.get<{
    code: number;
    contacts: Array<{ contact_id: number; contact_name: string; email: string }>;
  }>('/contacts', { email });

  if (response.code !== 0 || !response.contacts?.length) return null;

  return response.contacts.find(c => c.email === email) ?? null;
}

/**
 * Find existing Sales Order by reference_number (Shopify order name).
 */
export async function findSalesOrderByReference(
  client: ZohoBaseClient,
  referenceNumber: string
): Promise<{ salesorder_id: number } | null> {
  const response = await client.get<{
    code: number;
    salesorders: Array<{ salesorder_id: number; reference_number: string }>;
  }>('/salesorders', { reference_number: referenceNumber });

  if (response.code !== 0 || !response.salesorders?.length) return null;

  return response.salesorders.find(so => so.reference_number === referenceNumber) ?? null;
}
```

### 5.5 Conflict Resolution Strategy

```typescript
// src/zoho/utils/upsert.ts

import { ZohoBaseClient } from '../client/base-client';
import { ZohoItemCreateRequest, ZohoItem } from '../modules/items';
import { findItemBySku } from './dedup';

/**
 * Upsert an item: update if exists by SKU, create if not.
 * This prevents duplicates when both native Shopify integration and your platform are active.
 */
export async function upsertItem(
  client: ZohoBaseClient,
  data: ZohoItemCreateRequest
): Promise<{ item: ZohoItem; action: 'created' | 'updated' }> {
  if (data.sku) {
    const existing = await findItemBySku(client, data.sku);
    if (existing) {
      const updated = await client.put<{ code: number; item: ZohoItem }>(
        `/items/${existing.item_id}`,
        data
      );
      return { item: updated.item, action: 'updated' };
    }
  }

  const created = await client.post<{ code: number; item: ZohoItem }>('/items', data);
  return { item: created.item, action: 'created' };
}
```

---

## 6. Webhooks & Real-time Sync

### 6.1 Outgoing Webhooks (Zoho → Your Platform)

Zoho Inventory has **no programmatic webhook subscription API**. All outgoing webhooks are configured via UI:

**Setup path:** Settings → Organization Settings → Workflow Rules → Create Rule

**Configuration per module:**
1. Select module: Sales Orders, Purchase Orders, Invoices, Bills
2. Trigger type: Event Based (Created, Edited, Created or Edited, Deleted)
3. Actions: Add Webhook
4. Webhook URL: `https://your-platform.com/webhooks/zoho/{event_type}`
5. Method: POST
6. Entity Parameters: Append All

**Configure webhooks for these events:**

| Module | Event | Your Endpoint | Purpose |
|---|---|---|---|
| Sales Orders | Status changed to `fulfilled` | `/webhooks/zoho/so-fulfilled` | Trigger invoice creation |
| Invoices | Status changed to `paid` | `/webhooks/zoho/invoice-paid` | Update payment records |
| Purchase Orders | Status = `received` | `/webhooks/zoho/po-received` | Update stock levels |
| Items | Created or Edited | `/webhooks/zoho/item-updated` | Sync back to Shopify |

### 6.2 Incoming Webhook Receiver (Express)

```typescript
// src/routes/zoho-webhook.ts

import { Router, Request, Response } from 'express';
import crypto from 'crypto';

// Zoho doesn't sign webhook payloads — use a shared secret in custom parameters
const WEBHOOK_SECRET = process.env.ZOHO_WEBHOOK_SECRET!;

export function createZohoWebhookRouter(): Router {
  const router = Router();

  // Raw body needed for signature verification (if you add HMAC later)
  router.use('/webhooks/zoho', express.raw({ type: 'application/json' }));

  router.post('/webhooks/zoho/:eventType', async (req: Request, res: Response) => {
    const { eventType } = req.params;

    // Validate shared secret from custom parameters (set in Zoho workflow config)
    const providedSecret = req.headers['x-zoho-secret'] as string;
    if (providedSecret !== WEBHOOK_SECRET) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    let payload: Record<string, unknown>;
    try {
      payload = JSON.parse(req.body.toString());
    } catch {
      return res.status(400).json({ error: 'Invalid JSON payload' });
    }

    // Respond immediately — process asynchronously to avoid Zoho timeout
    res.status(200).json({ received: true });

    // Process in background
    setImmediate(() => handleZohoWebhookEvent(eventType, payload));
  });

  return router;
}

async function handleZohoWebhookEvent(
  eventType: string,
  payload: Record<string, unknown>
): Promise<void> {
  try {
    switch (eventType) {
      case 'so-fulfilled':
        await handleSalesOrderFulfilled(payload);
        break;
      case 'invoice-paid':
        await handleInvoicePaid(payload);
        break;
      case 'po-received':
        await handlePurchaseOrderReceived(payload);
        break;
      case 'item-updated':
        await handleItemUpdated(payload);
        break;
      default:
        console.warn(`Unknown Zoho webhook event type: ${eventType}`);
    }
  } catch (error) {
    console.error(`Error processing Zoho webhook ${eventType}:`, error);
    // Queue for retry or send to dead-letter queue
  }
}

async function handleSalesOrderFulfilled(payload: Record<string, unknown>): Promise<void> {
  const salesorderId = payload.salesorder_id as string;
  const customerId = payload.customer_id as string;
  // Create invoice for fulfilled order
  // ... call ZohoInvoicesClient.create(...)
}

async function handleInvoicePaid(payload: Record<string, unknown>): Promise<void> {
  const invoiceId = payload.invoice_id as string;
  // Mark corresponding Shopify order as paid if needed
  // ... update your DB payment status
}

async function handlePurchaseOrderReceived(payload: Record<string, unknown>): Promise<void> {
  const poId = payload.purchaseorder_id as string;
  // Sync updated stock levels back to Shopify
  // ... update Shopify inventory levels
}

async function handleItemUpdated(payload: Record<string, unknown>): Promise<void> {
  const itemId = payload.item_id as string;
  // Sync price or stock changes back to Shopify product
}
```

### 6.3 Polling Strategy (Fallback When Webhooks Aren't Configured)

Use `last_modified_time` for incremental polling. Zoho supports filtering by this field on list endpoints.

```typescript
// src/zoho/sync/polling.ts

import { ZohoBaseClient } from '../client/base-client';

/**
 * Polls Zoho for records modified since lastSyncTime.
 * Store lastSyncTime in your DB and update after each successful poll.
 */
export async function pollModifiedRecords(
  client: ZohoBaseClient,
  module: 'salesorders' | 'invoices' | 'items' | 'purchaseorders',
  lastSyncTime: Date
): Promise<unknown[]> {
  const isoTime = lastSyncTime.toISOString().replace('T', ' ').split('.')[0]; // "2026-03-28 10:00:00"

  let page = 1;
  const allRecords: unknown[] = [];

  while (true) {
    const response = await client.get<{
      code: number;
      [key: string]: unknown;
      page_context: { has_more_page: boolean };
    }>(`/${module}`, {
      last_modified_time: isoTime,
      page,
      per_page: 200,
    });

    // Module key varies: "salesorders", "invoices", "items", "purchaseorders"
    const records = response[module] as unknown[];
    if (records?.length) allRecords.push(...records);

    if (!response.page_context?.has_more_page) break;
    page++;
  }

  return allRecords;
}

/**
 * Polling job — run every 5 minutes as a cron job.
 */
export async function runIncrementalSync(
  client: ZohoBaseClient,
  merchantId: string,
  getLastSyncTime: (merchantId: string, module: string) => Promise<Date>,
  saveLastSyncTime: (merchantId: string, module: string, time: Date) => Promise<void>,
  processRecords: (module: string, records: unknown[]) => Promise<void>
): Promise<void> {
  const modules = ['salesorders', 'invoices', 'items', 'purchaseorders'] as const;

  for (const module of modules) {
    const lastSync = await getLastSyncTime(merchantId, module);
    const records = await pollModifiedRecords(client, module, lastSync);

    if (records.length > 0) {
      await processRecords(module, records);
    }

    await saveLastSyncTime(merchantId, module, new Date());
  }
}
```

---

## 7. Pagination & Bulk Operations

### 7.1 Pagination Implementation

All list endpoints use page-based pagination. Max page size is 200. Use `page_context.has_more_page` to detect end of results.

```typescript
// src/zoho/utils/pagination.ts

import { ZohoBaseClient } from '../client/base-client';

export interface PageContext {
  page: number;
  per_page: number;
  has_more_page: boolean;
}

/**
 * Generic paginator — fetches all pages from any Zoho list endpoint.
 * Returns a flat array of all records.
 *
 * @param client - ZohoBaseClient instance
 * @param path - Endpoint path, e.g. "/items"
 * @param recordKey - Key in response that holds the array, e.g. "items"
 * @param params - Additional query params (filters, date ranges, etc.)
 * @param onPage - Optional callback called after each page (for progress tracking)
 */
export async function fetchAllPages<T>(
  client: ZohoBaseClient,
  path: string,
  recordKey: string,
  params: Record<string, unknown> = {},
  onPage?: (page: number, records: T[]) => void
): Promise<T[]> {
  const allRecords: T[] = [];
  let page = 1;

  while (true) {
    const response = await client.get<{
      code: number;
      [key: string]: unknown;
      page_context: PageContext;
    }>(path, { ...params, page, per_page: 200 });

    const records = (response[recordKey] as T[]) ?? [];
    allRecords.push(...records);

    if (onPage) onPage(page, records);

    if (!response.page_context?.has_more_page) break;

    page++;

    // Safety: pause 600ms between pages to stay within 100 req/min rate limit
    // (max ~80 req/min with 600ms gaps, plus headroom for other calls)
    await new Promise(r => setTimeout(r, 650));
  }

  return allRecords;
}

// Usage example:
// const allItems = await fetchAllPages<ZohoItem>(client, '/items', 'items');
// const allSalesOrders = await fetchAllPages<ZohoSalesOrder>(client, '/salesorders', 'salesorders', { status: 'confirmed' });
```

### 7.2 Full Inventory Sync (Initial Setup)

```typescript
// src/zoho/sync/initial-sync.ts

import { ZohoBaseClient } from '../client/base-client';
import { fetchAllPages } from '../utils/pagination';
import { ZohoItem } from '../modules/items';
import { ZohoSalesOrder } from '../modules/sales-orders';

export interface SyncStats {
  itemsSynced: number;
  salesOrdersSynced: number;
  errors: string[];
}

/**
 * Full initial sync from Zoho Inventory → your platform DB.
 * Run once on merchant onboarding, then use incremental sync.
 */
export async function runFullInitialSync(
  client: ZohoBaseClient,
  handlers: {
    onItems: (items: ZohoItem[]) => Promise<void>;
    onSalesOrders: (orders: ZohoSalesOrder[]) => Promise<void>;
    onProgress: (msg: string) => void;
  }
): Promise<SyncStats> {
  const stats: SyncStats = { itemsSynced: 0, salesOrdersSynced: 0, errors: [] };

  // Sync Items
  handlers.onProgress('Syncing items...');
  try {
    const items = await fetchAllPages<ZohoItem>(
      client, '/items', 'items', {},
      (page, records) => handlers.onProgress(`  Items page ${page}: ${records.length} records`)
    );
    await handlers.onItems(items);
    stats.itemsSynced = items.length;
  } catch (err) {
    stats.errors.push(`Items sync failed: ${(err as Error).message}`);
  }

  // Sync Sales Orders (last 90 days)
  handlers.onProgress('Syncing sales orders...');
  const ninetyDaysAgo = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000)
    .toISOString().split('T')[0];
  try {
    const salesOrders = await fetchAllPages<ZohoSalesOrder>(
      client, '/salesorders', 'salesorders',
      { date_after: ninetyDaysAgo },
      (page, records) => handlers.onProgress(`  Sales Orders page ${page}: ${records.length} records`)
    );
    await handlers.onSalesOrders(salesOrders);
    stats.salesOrdersSynced = salesOrders.length;
  } catch (err) {
    stats.errors.push(`Sales Orders sync failed: ${(err as Error).message}`);
  }

  return stats;
}
```

### 7.3 Incremental Sync (Ongoing)

```typescript
// src/zoho/sync/incremental-sync.ts

import { ZohoBaseClient } from '../client/base-client';
import { fetchAllPages } from '../utils/pagination';
import { ZohoItem } from '../modules/items';
import { ZohoSalesOrder } from '../modules/sales-orders';

/**
 * Fetches records modified since lastModifiedTime.
 * Use last_modified_time query param for all major endpoints.
 * Format: "yyyy-MM-dd HH:mm:ss" (Zoho uses this format)
 */
export function toZohoDateTime(date: Date): string {
  return date.toISOString()
    .replace('T', ' ')
    .split('.')[0]; // "2026-03-28 10:00:00"
}

export async function fetchModifiedItems(
  client: ZohoBaseClient,
  since: Date
): Promise<ZohoItem[]> {
  return fetchAllPages<ZohoItem>(
    client, '/items', 'items',
    { last_modified_time: toZohoDateTime(since) }
  );
}

export async function fetchModifiedSalesOrders(
  client: ZohoBaseClient,
  since: Date
): Promise<ZohoSalesOrder[]> {
  return fetchAllPages<ZohoSalesOrder>(
    client, '/salesorders', 'salesorders',
    { last_modified_time: toZohoDateTime(since) }
  );
}
```

---

## 8. Rate Limiting & Error Handling

### 8.1 Rate Limits

| Limit Type | Value | Scope |
|---|---|---|
| Per-minute | 100 requests/min | Per organization |
| Daily (Free) | 1,000 requests/day | Per organization |
| Daily (Standard) | 2,000 requests/day | Per organization |
| Daily (Professional) | 5,000 requests/day | Per organization |
| Daily (Premium/Enterprise) | 10,000 requests/day | Per organization |
| Concurrent (Free) | 5 concurrent calls | Per organization |
| Concurrent (Paid) | 10 concurrent calls | Per organization |

**Safe operation target:** Stay at ≤80 requests/min to allow headroom. For bulk operations, add a 750ms delay between requests.

### 8.2 Retry Logic with Exponential Backoff

```typescript
// src/zoho/utils/retry.ts

export interface RetryConfig {
  maxRetries: number;
  baseDelayMs: number;
  maxDelayMs: number;
  retryableStatuses: number[];
}

export const DEFAULT_RETRY_CONFIG: RetryConfig = {
  maxRetries: 5,
  baseDelayMs: 1000,
  maxDelayMs: 32000,
  retryableStatuses: [429, 500, 502, 503, 504],
};

/**
 * Executes a function with exponential backoff retry.
 * Retries on rate limit (429) and server errors (5xx).
 */
export async function withRetry<T>(
  fn: () => Promise<T>,
  config: RetryConfig = DEFAULT_RETRY_CONFIG,
  context = 'API call'
): Promise<T> {
  let lastError: unknown;

  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      const httpStatus = (error as any)?.httpStatus ?? (error as any)?.response?.status;
      const isRetryable = config.retryableStatuses.includes(httpStatus);

      if (!isRetryable || attempt === config.maxRetries) {
        throw error;
      }

      // Exponential backoff: 1s, 2s, 4s, 8s, 16s (capped at maxDelayMs)
      const delay = Math.min(
        config.baseDelayMs * Math.pow(2, attempt),
        config.maxDelayMs
      );

      console.warn(
        `${context} failed with ${httpStatus} (attempt ${attempt + 1}/${config.maxRetries}). Retrying in ${delay}ms...`
      );

      await new Promise(r => setTimeout(r, delay));
    }
  }

  throw lastError;
}
```

### 8.3 Circuit Breaker Pattern

```typescript
// src/zoho/utils/circuit-breaker.ts

export enum CircuitState {
  CLOSED = 'CLOSED',     // Normal operation
  OPEN = 'OPEN',         // Failing — reject all calls
  HALF_OPEN = 'HALF_OPEN', // Testing recovery
}

export class CircuitBreaker {
  private state = CircuitState.CLOSED;
  private failureCount = 0;
  private lastFailureTime = 0;
  private successCount = 0;

  constructor(
    private readonly failureThreshold = 5,
    private readonly recoveryTimeMs = 60000, // 1 minute
    private readonly halfOpenSuccessThreshold = 2
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() - this.lastFailureTime > this.recoveryTimeMs) {
        this.state = CircuitState.HALF_OPEN;
        this.successCount = 0;
      } else {
        throw new Error('Circuit breaker OPEN — Zoho API is unavailable. Try again later.');
      }
    }

    try {
      const result = await fn();

      if (this.state === CircuitState.HALF_OPEN) {
        this.successCount++;
        if (this.successCount >= this.halfOpenSuccessThreshold) {
          this.reset();
        }
      } else {
        this.failureCount = 0; // Reset on success
      }

      return result;
    } catch (error) {
      this.recordFailure();
      throw error;
    }
  }

  private recordFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.failureCount >= this.failureThreshold) {
      this.state = CircuitState.OPEN;
      console.error(`Circuit breaker opened after ${this.failureCount} failures.`);
    }
  }

  private reset(): void {
    this.state = CircuitState.CLOSED;
    this.failureCount = 0;
    this.successCount = 0;
    console.info('Circuit breaker CLOSED — Zoho API recovered.');
  }

  getState(): CircuitState {
    return this.state;
  }
}
```

### 8.4 Error Code Handling Reference

```typescript
// src/zoho/utils/error-handler.ts

export interface ZohoError {
  code: number;
  message: string;
  httpStatus: number;
}

export function classifyZohoError(error: ZohoError): {
  type: 'auth' | 'validation' | 'notFound' | 'rateLimit' | 'server' | 'unknown';
  retryable: boolean;
  userMessage: string;
} {
  switch (error.httpStatus) {
    case 400:
      return {
        type: 'validation',
        retryable: false,
        userMessage: `Invalid request: ${error.message}`,
      };

    case 401:
      return {
        type: 'auth',
        retryable: true, // After token refresh
        userMessage: 'Authentication failed. Please reconnect your Zoho account.',
      };

    case 404:
      return {
        type: 'notFound',
        retryable: false,
        userMessage: `Resource not found: ${error.message}`,
      };

    case 429:
      return {
        type: 'rateLimit',
        retryable: true,
        userMessage: 'Zoho rate limit reached. Operations will resume automatically.',
      };

    case 500:
    case 502:
    case 503:
    case 504:
      return {
        type: 'server',
        retryable: true,
        userMessage: 'Zoho server error. Retrying automatically.',
      };

    default:
      // Application-level errors (code != 0 in response body)
      if (error.code === 1002) {
        return {
          type: 'notFound',
          retryable: false,
          userMessage: `Record does not exist: ${error.message}`,
        };
      }
      return {
        type: 'unknown',
        retryable: false,
        userMessage: `Unexpected error: ${error.message}`,
      };
  }
}

/**
 * Common validation errors and how to fix them:
 *
 * "customer_id is required"     → Ensure Zoho contact exists before creating SO/Invoice
 * "Invalid date format"         → Use yyyy-mm-dd format
 * "Duplicate salesorder_number" → Check existing SOs before creating; use unique numbering
 * "Invalid organization_id"     → Re-fetch org ID from /organizations endpoint
 * "Item is inactive"            → Call POST /items/{id}/active before using in transactions
 * "Rate limit exceeded"         → Back off 60 seconds, then retry
 * "Invalid AuthToken"           → Refresh access token, then retry once
 */
```

---

## 9. Data Model Reference

### 9.1 Complete TypeScript Interfaces

```typescript
// src/zoho/types/models.ts

// ---- ITEMS ----

export interface ZohoItemTaxPreference {
  tax_id: number;
  tax_specification: 'intra' | 'inter';
}

export interface ZohoItemLocation {
  location_id: string;
  location_name?: string;
  is_primary?: boolean;
  status?: 'active' | 'inactive';
  initial_stock?: number;
  initial_stock_rate?: number;
  location_stock_on_hand?: string;
  location_available_stock?: string;
  location_actual_available_stock?: string;
}

export interface ZohoItem {
  item_id: number;
  name: string;
  group_id?: string;
  group_name?: string;
  unit?: string;
  item_type: 'inventory' | 'sales' | 'purchases' | 'sales_and_purchases';
  product_type: 'goods' | 'service';
  is_taxable: boolean;
  tax_id?: number;
  tax_name?: string;
  tax_percentage?: number;
  description?: string;
  purchase_description?: string;
  rate?: number;
  purchase_rate?: number;
  pricebook_rate?: number;
  reorder_level?: number;
  vendor_id?: number;
  vendor_name?: string;
  purchase_account_id?: number;
  purchase_account_name?: string;
  account_name?: string;
  inventory_account_id?: number;
  sku?: string;
  upc?: number;
  ean?: number;
  isbn?: string;
  part_number?: string;
  status: 'active' | 'inactive';
  is_combo_product?: boolean;
  attribute_name1?: string;
  attribute_option_name1?: string;
  locations?: ZohoItemLocation[];
  // India GST
  hsn_or_sac?: string;
  item_tax_preferences?: ZohoItemTaxPreference[];
  custom_fields?: Array<{ customfield_id: string; label: string; value: string }>;
  created_time: string;
  last_modified_time: string;
}

// ---- CONTACTS ----

export interface ZohoContactAddress {
  attention?: string;
  address: string;
  street2?: string;
  city?: string;
  state?: string;
  zip?: string;
  country?: string;
  phone?: string;
  fax?: string;
}

export interface ZohoContact {
  contact_id: number;
  contact_name: string;
  company_name?: string;
  contact_type: 'customer' | 'vendor';
  email?: string;
  phone?: string;
  mobile?: string;
  billing_address?: ZohoContactAddress;
  shipping_address?: ZohoContactAddress;
  currency_id?: number;
  currency_code?: string;
  // India GST
  gst_no?: string;                   // 15-digit GSTIN
  gst_treatment?: 'business_gst' | 'business_none' | 'overseas' | 'consumer';
  place_of_contact?: string;         // State code
  status: 'active' | 'inactive';
  created_time: string;
  last_modified_time: string;
}

// ---- SALES ORDERS ----

export interface ZohoSalesOrderLineItem {
  line_item_id?: number;
  item_id?: number;
  name: string;
  description?: string;
  rate: number;
  quantity: number;
  unit?: string;
  tax_id?: number;
  tax_percentage?: number;
  item_total?: number;
  discount?: number;
  location_id?: string;
  is_invoiced?: boolean;
  quantity_invoiced?: number;
  quantity_packed?: number;
  quantity_shipped?: number;
  // India
  hsn_or_sac?: string;
}

export interface ZohoSalesOrder {
  salesorder_id: number;
  salesorder_number: string;
  date: string;
  status: 'draft' | 'confirmed' | 'fulfilled' | 'void';
  shipment_date?: string;
  reference_number?: string;
  customer_id: number;
  customer_name?: string;
  currency_code?: string;
  exchange_rate?: number;
  discount?: number;
  discount_amount?: number;
  delivery_method?: string;
  location_id?: string;
  line_items: ZohoSalesOrderLineItem[];
  billing_address?: ZohoContactAddress;
  shipping_address?: ZohoContactAddress;
  shipping_charge?: number;
  adjustment?: number;
  sub_total?: number;
  tax_total?: number;
  total?: number;
  notes?: string;
  terms?: string;
  // India
  gst_no?: string;
  gst_treatment?: 'business_gst' | 'business_none' | 'overseas' | 'consumer';
  place_of_supply?: string;
  is_pre_gst?: boolean;
  created_time: string;
  last_modified_time: string;
}

// ---- PURCHASE ORDERS ----

export interface ZohoPurchaseOrderLineItem {
  line_item_id?: number;
  item_id: number;
  name?: string;
  rate?: number;
  purchase_rate?: number;
  quantity: number;
  quantity_received?: number;
  unit?: string;
  location_id?: string;
  // India
  hsn_or_sac?: string;
  tax_id?: number;
  tax_exemption_code?: string;
  reverse_charge_tax_id?: string;
  reverse_charge_tax_name?: string;
  reverse_charge_tax_percentage?: number;
  reverse_charge_tax_amount?: number;
}

export interface ZohoPurchaseOrder {
  purchaseorder_id: number;
  purchaseorder_number: string;
  date: string;
  expected_delivery_date?: string;
  status: 'draft' | 'issued' | 'Partially_Received' | 'received' | 'cancelled';
  vendor_id: number;
  vendor_name?: string;
  is_drop_shipment?: boolean;
  salesorder_id?: number;
  is_backorder?: boolean;
  currency_code?: string;
  reference_number?: string;
  line_items: ZohoPurchaseOrderLineItem[];
  purchasereceives?: Array<{ receive_id: string; date: string; status: string }>;
  bills?: Array<{ bill_id: string; bill_number: string; status: string }>;
  sub_total?: number;
  tax_total?: number;
  total?: number;
  // India
  gst_no?: string;
  gst_treatment?: 'business_gst' | 'business_none' | 'overseas' | 'consumer';
  source_of_supply?: string;
  destination_of_supply?: string;
  is_pre_gst?: boolean;
  is_reverse_charge_applied?: boolean;
  created_time: string;
  last_modified_time: string;
}

// ---- INVOICES ----

export interface ZohoInvoiceLineItem {
  line_item_id?: string;
  item_id?: string;
  name: string;
  description?: string;
  rate: number;
  quantity: number;
  unit?: string;
  tax_id?: string;
  tax_percentage?: number;
  discount?: number;
  item_total?: number;
  location_id?: string;
  // India
  hsn_or_sac?: string;
}

export interface ZohoInvoice {
  invoice_id: string;
  invoice_number: string;
  date: string;
  due_date?: string;
  status: 'sent' | 'draft' | 'overdue' | 'paid' | 'void' | 'unpaid' | 'partially_paid' | 'viewed';
  payment_terms?: number;
  payment_terms_label?: string;
  customer_id: string;
  customer_name?: string;
  currency_code?: string;
  exchange_rate?: number;
  salesorder_id?: string;
  reference_number?: string;
  discount?: number;
  is_discount_before_tax?: boolean;
  is_inclusive_tax?: boolean;
  line_items: ZohoInvoiceLineItem[];
  shipping_charge?: number;
  adjustment?: number;
  sub_total?: number;
  tax_total?: number;
  total?: string;
  balance?: string;
  payment_made?: number;
  credits_applied?: number;
  write_off_amount?: number;
  billing_address?: ZohoContactAddress;
  shipping_address?: ZohoContactAddress;
  invoice_url?: string;
  is_emailed?: boolean;
  allow_partial_payments?: boolean;
  // India
  gst_no?: string;
  gst_treatment?: 'business_gst' | 'business_none' | 'overseas' | 'consumer';
  place_of_supply?: string;
  is_pre_gst?: boolean;
  created_time: string;
  last_modified_time: string;
}

// ---- TRANSFER ORDERS ----

export interface ZohoTransferOrderLineItem {
  item_id: number;
  name: string;
  quantity_transfer: number;
  unit?: string;
  description?: string;
}

export interface ZohoTransferOrder {
  transfer_order_id: string;
  transfer_order_number?: string;
  date: string;
  from_location_id: string;
  to_location_id: string;
  line_items: ZohoTransferOrderLineItem[];
  is_intransit_order: boolean;
  status: 'draft' | 'in_transit' | 'transferred';
  created_time: string;
  last_modified_time: string;
}

// ---- PACKAGES ----

export interface ZohoPackage {
  package_id: number;
  package_number?: string;
  salesorder_id: number;
  date: string;
  status: string;
  line_items: Array<{
    so_line_item_id: number;
    quantity: number;
  }>;
  notes?: string;
  created_time: string;
}

// ---- SHIPMENT ORDERS ----

export interface ZohoShipmentOrder {
  shipmentorder_id: number;
  shipment_number: string;
  salesorder_id: number;
  package_ids: number[];
  date: string;
  delivery_method: string;
  tracking_number: string;
  carrier?: string;
  status: 'shipped' | 'delivered';
  detailed_status?: string;
  notes?: string;
  created_time: string;
}
```

### 9.2 Field Mappings: Shopify → Zoho Inventory

| Shopify Field | Zoho Field | Module | Notes |
|---|---|---|---|
| `product.title` | `item.name` | Items | Use `title + variant` for multi-variant |
| `variant.sku` | `item.sku` | Items | Required for dedup |
| `variant.price` | `item.rate` | Items | Sales price |
| `variant.compare_at_price` | `item.purchase_rate` | Items | Estimate if no cost price |
| `variant.inventory_quantity` | `item.locations[].initial_stock` | Items | Per warehouse |
| `product.tags` (hsn:XXXX) | `item.hsn_or_sac` | Items | Parse from tags |
| `order.order_number` | `salesorder.salesorder_number` | SO | Prefix: "SHOP-{number}" |
| `order.name` | `salesorder.reference_number` | SO | "#1001" |
| `order.customer.id` | (lookup by email) | SO | Find/create Zoho contact first |
| `order.line_items[].price` | `salesorder.line_items[].rate` | SO | Per unit price |
| `order.line_items[].quantity` | `salesorder.line_items[].quantity` | SO | |
| `order.shipping_address.province_code` | `salesorder.place_of_supply` | SO | India: 2-char state code |
| `fulfillment.tracking_number` | `shipmentorder.tracking_number` | Shipment | |
| `fulfillment.tracking_company` | `shipmentorder.delivery_method` | Shipment | Carrier name |
| `customer.email` | `contact.email` | Contacts | Primary lookup key |
| `customer.first_name + last_name` | `contact.contact_name` | Contacts | |
| `customer.default_address` | `contact.billing_address` | Contacts | |
| `note_attributes[gstin]` | `salesorder.gst_no` | SO | Store GSTIN as note attribute |
| `note_attributes[gst_treatment]` | `salesorder.gst_treatment` | SO | Custom field in checkout |

---

## 10. Implementation Checklist

### Phase 1: Infrastructure Setup
- [ ] Register Zoho app at [api-console.zoho.com](https://api-console.zoho.com/) — create Server-based app
- [ ] Configure redirect URI: `https://your-platform.com/zoho/callback`
- [ ] Store `ZOHO_CLIENT_ID`, `ZOHO_CLIENT_SECRET` in environment variables
- [ ] Set `ZOHO_WEBHOOK_SECRET` for webhook validation
- [ ] Create `zoho_tokens` table in your database (see schema in section 2.9)
- [ ] Implement `TokenStore` with database persistence (replace InMemoryTokenStore)
- [ ] Deploy the OAuth callback route (`/zoho/callback`)

### Phase 2: Authentication
- [ ] Implement `generateAuthUrl()` — returns DC-specific OAuth URL
- [ ] Implement OAuth callback handler — exchanges code for tokens
- [ ] Implement `fetchOrganizationId()` — called immediately after token exchange
- [ ] Implement `refreshAccessToken()` — proactive refresh at 55-minute mark
- [ ] Implement `ZohoAuthMiddleware.getValidTokenSet()` — auto-refreshes on every API call
- [ ] Test: connect a Zoho India org, verify token storage and refresh
- [ ] Test: connect a Zoho Global (Singapore) org, verify DC-specific URLs

### Phase 3: Core Clients
- [ ] Implement `ZohoBaseClient` with all CRUD methods + error handling
- [ ] Verify circuit breaker + retry logic works against live API (test with 429 simulation)
- [ ] Implement `ZohoItemsClient` — create, update, list, deactivate
- [ ] Implement `ZohoContactsClient` — create, find by email
- [ ] Implement `ZohoSalesOrdersClient` — create, confirm, void
- [ ] Implement `ZohoInvoicesClient` — create, mark sent, email
- [ ] Implement `ZohoPurchaseOrdersClient` — create, issue
- [ ] Implement `ZohoPackagesClient` — create with salesorder_id as query param
- [ ] Implement `ZohoShipmentsClient` — create, mark delivered
- [ ] Implement `ZohoInventoryAdjustmentsClient` — create adjustment
- [ ] Implement `ZohoTransferOrdersClient` — create, mark as transferred

### Phase 4: GST Configuration
- [ ] Create tax records in Zoho for India:
  - CGST 9% + SGST 9% (intra-state, combined 18%)
  - IGST 18% (inter-state)
  - CGST 6% + SGST 6% (intra-state, combined 12%)
  - IGST 12% (inter-state)
  - CGST 2.5% + SGST 2.5% (intra-state, combined 5%)
  - IGST 5% (inter-state)
  - Zero-rated (exports)
- [ ] Store tax IDs per merchant org (query `GET /settings/taxes` to retrieve)
- [ ] Implement `determineGstTreatment()` — business vs consumer vs overseas
- [ ] Implement `getPlaceOfSupply()` — map Shopify province_code → Zoho state code
- [ ] Implement HSN extraction from Shopify product tags
- [ ] Configure seller state code per merchant org (for CGST/SGST vs IGST determination)

### Phase 5: Shopify Integration
- [ ] Set up Shopify webhook subscriptions:
  - `products/create` and `products/update` → sync to Zoho items
  - `orders/create` → create Zoho Sales Order + Contact (if new)
  - `fulfillments/create` → create Zoho Package + Shipment Order + Invoice
  - `orders/cancelled` → void Zoho Sales Order
  - `inventory_levels/update` → trigger Zoho Inventory Adjustment (if manual override)
- [ ] Implement `findItemBySku()` before creating any item (prevent duplicates)
- [ ] Implement `findContactByEmail()` before creating any contact
- [ ] Implement `findSalesOrderByReference()` before creating any SO
- [ ] Build `zohoItemIdMap`: maintain `{ shopifyVariantId → zohoItemId }` mapping in DB
- [ ] Build `zohoContactIdMap`: maintain `{ shopifyCustomerId → zohoContactId }` mapping in DB
- [ ] Build `zohoSoIdMap`: maintain `{ shopifyOrderId → zohoSalesOrderId }` mapping in DB
- [ ] Build `soLineItemIdMap`: maintain `{ shopifyLineItemId → zohoSoLineItemId }` mapping in DB (needed for packages)

### Phase 6: Webhooks & Sync
- [ ] Configure Zoho Inventory outgoing webhooks via Zoho UI:
  - Sales Orders → fulfilled → `POST /webhooks/zoho/so-fulfilled`
  - Invoices → paid → `POST /webhooks/zoho/invoice-paid`
  - Purchase Orders → received → `POST /webhooks/zoho/po-received`
- [ ] Implement webhook receiver with shared secret validation
- [ ] Implement polling fallback (`runIncrementalSync`) — schedule every 5 minutes
- [ ] Implement `runFullInitialSync` — run once on merchant onboarding
- [ ] Store `last_sync_time` per merchant per module in DB

### Phase 7: Warehouses (Multi-Location)
- [ ] Fetch warehouse/location list on org connect: `GET /settings/warehouses`
- [ ] Store `location_id` per merchant's warehouses in DB
- [ ] Map Shopify locations to Zoho locations — store mapping in DB
- [ ] Set `location_id` on all Sales Order, Purchase Order, and Transfer Order line items
- [ ] Test Transfer Order creation between two warehouse locations

### Phase 8: Testing & Validation
- [ ] Test India org: Verify `gst_no`, `place_of_supply`, `hsn_or_sac` fields appear on all transactions
- [ ] Test token refresh: Manually expire token, verify auto-refresh fires
- [ ] Test rate limiting: Send 110 requests in a minute, verify backoff triggers at 429
- [ ] Test duplicate prevention: Create same item twice, verify upsert runs update (not create)
- [ ] Test full Shopify order flow: `products/create` → `orders/create` → `fulfillments/create` → verify SO + Package + Shipment + Invoice created in Zoho
- [ ] Test Zoho → Books sync: Verify items and sales orders created via Inventory API appear in Books
- [ ] Test Singapore org: Verify `.com` DC URLs used (not `.in`)
- [ ] Verify circuit breaker opens after 5 consecutive 500 errors
- [ ] Load test: Simulate bulk product import of 1,000 products with rate limiting

### Phase 9: Production Readiness
- [ ] Move all secrets to environment variables / secrets manager
- [ ] Add structured logging for all Zoho API calls (include `merchantId`, `endpoint`, `httpStatus`, `duration`)
- [ ] Set up alerts for: circuit breaker open, daily API quota >80% used, token refresh failures
- [ ] Implement dead-letter queue for failed webhook events (retry up to 3x then alert)
- [ ] Document merchant onboarding flow: which DC to select, how to authorize, what to verify
- [ ] Set API request budget tracking per merchant org per day (alert at 80% of daily limit)
- [ ] Review and pin Zoho API version (`v1`) — monitor Zoho changelog for deprecations

---

## Appendix A: Quick Reference — All Endpoints

**Base URL (India):** `https://www.zohoapis.in/inventory/v1`
**Base URL (Global/Singapore):** `https://www.zohoapis.com/inventory/v1`

All requests require `?organization_id={id}` query parameter.

| Module | Create | List | Retrieve | Update | Delete | Special |
|---|---|---|---|---|---|---|
| Items | `POST /items` | `GET /items` | `GET /items/{id}` | `PUT /items/{id}` | `DELETE /items/{id}` | `POST /items/{id}/active` |
| Composite Items | `POST /compositeitems` | `GET /compositeitems` | `GET /compositeitems/{id}` | `PUT /compositeitems/{id}` | `DELETE /compositeitems/{id}` | — |
| Item Groups | `POST /itemgroups` | `GET /itemgroups` | `GET /itemgroups/{id}` | `PUT /itemgroups/{id}` | `DELETE /itemgroups/{id}` | — |
| Sales Orders | `POST /salesorders` | `GET /salesorders` | `GET /salesorders/{id}` | `PUT /salesorders/{id}` | `DELETE /salesorders/{id}` | `POST /salesorders/{id}/status/confirmed` |
| Purchase Orders | `POST /purchaseorders` | `GET /purchaseorders` | `GET /purchaseorders/{id}` | `PUT /purchaseorders/{id}` | `DELETE /purchaseorders/{id}` | `POST /purchaseorders/{id}/status/issued` |
| Invoices | `POST /invoices` | `GET /invoices` | `GET /invoices/{id}` | `PUT /invoices/{id}` | `DELETE /invoices/{id}` | `POST /invoices/{id}/status/sent` |
| Packages | `POST /packages?salesorder_id={id}` | `GET /packages` | `GET /packages/{id}` | `PUT /packages/{id}` | `DELETE /packages/{id}` | — |
| Shipment Orders | `POST /shipmentorders?salesorder_id={id}` | — | `GET /shipmentorders/{id}` | `PUT /shipmentorders/{id}` | `DELETE /shipmentorders/{id}` | `POST /shipmentorders/{id}/status/delivered` |
| Transfer Orders | `POST /transferorders` | `GET /transferorders` | `GET /transferorders/{id}` | `PUT /transferorders/{id}` | `DELETE /transferorders/{id}` | `POST /transferorders/{id}/markastransferred` |
| Inv. Adjustments | `POST /inventoryadjustments` | `GET /inventoryadjustments` | `GET /inventoryadjustments/{id}` | `PUT /inventoryadjustments/{id}` | `DELETE /inventoryadjustments/{id}` | — |
| Contacts | `POST /contacts` | `GET /contacts` | `GET /contacts/{id}` | `PUT /contacts/{id}` | `DELETE /contacts/{id}` | — |
| Organizations | — | `GET /organizations` | — | — | — | Required for org ID lookup |
| Warehouses | — | `GET /settings/warehouses` | — | — | — | Requires `settings.READ` scope |

---

## Appendix B: Critical Gotchas

1. **DC mismatch causes 401**: India orgs MUST use `accounts.zoho.in` for auth and `www.zohoapis.in` for API. Using `.com` endpoints returns 401.

2. **Packages require `salesorder_id` as a QUERY param, not in body**: `POST /packages?organization_id=X&salesorder_id=Y` — this is different from most endpoints.

3. **Shipments require both `salesorder_id` and `package_ids` as query params**: Pass comma-separated package IDs.

4. **Authorization code expires in 60 seconds**: Exchange the code immediately in the callback handler.

5. **Refresh token limit is 20 per user per app**: If exceeded, the oldest is auto-deleted. Always reuse the existing refresh token — don't re-authorize unless the merchant explicitly disconnects.

6. **`item_type` must be `inventory` if the item belongs to a group**: The API returns a validation error otherwise.

7. **`salesorder_number` must be unique**: Attempting to create a second SO with the same number returns a 400. Check with `findSalesOrderByReference()` before creating.

8. **`gst_no` must be exactly 15 characters**: Validate with the regex before sending to Zoho.

9. **`place_of_supply` accepts 2-char codes only**: "TN", "MH", "KA" — not full state names.

10. **Inventory Adjustments endpoint may return 404**: This endpoint was not live during research. Test against your Zoho plan's API — it may require Standard plan or higher. Fallback: create a Bill or manual correction in Zoho Books if the endpoint is unavailable.

11. **Pricebooks endpoint returns 404 on lower plans**: `/pricebooks` requires Premium plan or higher. Reference `pricebook_id` fields on existing entities, but don't try to create/list pricebooks on Free/Standard plans.

12. **Zoho Inventory's native Shopify integration vs your custom integration**: If the merchant already has the native Zoho-Shopify integration active, ALL operations must use upsert (not create) to prevent duplicates. Disable the native integration or coordinate carefully.

13. **Books API version is v3, Inventory API version is v1**: Don't mix these. `zohoapis.in/inventory/v1/` and `zohoapis.in/books/v3/` are separate APIs with different schemas even though they share data.

14. **`last_modified_time` filter format**: Use `"2026-03-28 10:00:00"` (space-separated, no T, no timezone) — not ISO 8601 format.

---

*Sources: [Zoho Inventory API v1](https://www.zoho.com/inventory/api/v1/) | [Zoho OAuth Protocol](https://www.zoho.com/accounts/protocol/oauth.html) | [Zoho Books-Inventory Integration](https://www.zoho.com/books/integrations/zoho-inventory/) | [Zoho Data Sync Documentation](https://www.zoho.com/us/books/kb/integrations/data-sync.html) | [Zoho Inventory Webhooks](https://www.zoho.com/us/inventory/help/settings/incoming-webhooks.html)*
