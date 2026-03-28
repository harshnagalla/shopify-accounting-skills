---
name: zoho-books-integration
description: Integrates Zoho Books API for full accounting operations in a Shopify platform. Covers OAuth 2.0, Contacts, GST-compliant Invoices with CGST/SGST/IGST split, Bills and Expenses, Chart of Accounts setup, Journal Entries, Bank Reconciliation for Shopify payouts, Credit Notes for refunds, India GST tax configuration, Singapore 9% GST with F5 returns, e-invoicing IRN generation, TDS/TCS handling, and multi-currency support. Use when building or modifying Zoho Books accounting integration or Shopify payout reconciliation.
---

# Zoho Books Integration Skill
## For: Shopify Accounting Platform (India + Singapore)
## Version: 1.0 | March 2026

**Source Reference:** [Zoho Books API v3 Documentation](https://www.zoho.com/books/api/v3/)

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Authentication Implementation](#2-authentication-implementation)
3. [Organization Management](#3-organization-management)
4. [Core Accounting API Integrations](#4-core-accounting-api-integrations)
   - 4.1 [Contacts](#41-contacts)
   - 4.2 [Invoices](#42-invoices)
   - 4.3 [Bills & Expenses](#43-bills--expenses)
   - 4.4 [Chart of Accounts](#44-chart-of-accounts)
   - 4.5 [Journal Entries](#45-journal-entries)
   - 4.6 [Bank Accounts](#46-bank-accounts)
   - 4.7 [Bank Transactions & Reconciliation](#47-bank-transactions--reconciliation)
   - 4.8 [Customer Payments](#48-customer-payments)
   - 4.9 [Credit Notes](#49-credit-notes)
   - 4.10 [Recurring Invoices](#410-recurring-invoices)
5. [India GST Engine](#5-india-gst-engine)
6. [Singapore GST](#6-singapore-gst)
7. [Multi-Currency Support](#7-multi-currency-support)
8. [Webhooks](#8-webhooks)
9. [Reconciliation Workflow](#9-reconciliation-workflow)
10. [Reporting](#10-reporting)
11. [Rate Limiting & Error Handling](#11-rate-limiting--error-handling)
12. [Data Model Reference](#12-data-model-reference)
13. [Implementation Checklist](#13-implementation-checklist)

---

## 1. Overview & Architecture

### What This Integration Does

This integration connects a Shopify store to Zoho Books, automating the full accounting lifecycle:

- **Shopify Orders → Zoho Books Invoices**: Every confirmed Shopify order becomes a GST-compliant invoice in Zoho Books, with correct tax treatment (CGST+SGST for intra-state, IGST for inter-state, 9% GST for Singapore).
- **Shopify Refunds → Credit Notes**: Shopify refunds trigger credit notes in Zoho Books, reversing the original tax entries.
- **Shopify Payouts → Bank Reconciliation**: Shopify Payments payouts are imported into a virtual "Shopify Payments" bank account in Zoho Books for reconciliation.
- **Shopify Fees → Bills**: Shopify subscription fees, payment gateway charges, and shipping costs are recorded as vendor bills.
- **Multi-tenant**: One deployment handles multiple merchant Zoho Books connections — each merchant has their own Zoho OAuth token + organization ID.

### Data Flow

```
Shopify Order
    │
    ▼
Tax Engine (Place of Supply + HSN/GSTIN detection)
    │
    ├──► Create/Find Contact in Zoho Books (customer record)
    │
    ├──► Create Invoice in Zoho Books (GST-compliant line items)
    │
    ├──► Record Customer Payment (when Shopify marks payment captured)
    │
    └──► On Refund: Create Credit Note + apply to invoice

Shopify Payout
    │
    ▼
Import as Bank Statement transaction into "Shopify Payments" bank account
    │
    └──► Reconcile: match payout credits to invoice payments
         ├── Fee deductions → Expense entries
         └── Refunds → Credit note refunds
```

### Role of Zoho Books

| Function | Zoho Books Module |
|----------|-------------------|
| General ledger | Chart of Accounts + Journal Entries |
| Accounts receivable | Invoices + Customer Payments |
| Accounts payable | Bills + Vendor Payments |
| GST returns data | India: GSTR-1/3B (UI-driven push); Singapore: F5 Return (UI export) |
| Bank reconciliation | Bank Accounts + Bank Transactions |
| P&L / Balance Sheet | Assembled from transaction APIs + Ledger API |
| E-invoicing (India) | IRN generation via Zoho as GSP |

### Data Centers

| Merchant Region | DC | API Base URL | Auth URL |
|-----------------|-----|-------------|----------|
| India | `.in` | `https://www.zohoapis.in/books/v3/` | `https://accounts.zoho.in` |
| Singapore | `.com` | `https://www.zohoapis.com/books/v3/` | `https://accounts.zoho.com` |

---

## 2. Authentication Implementation

### 2.1 App Registration

Register **two** Zoho applications — one per data center:

- **India DC**: [https://api-console.zoho.in](https://api-console.zoho.in) — Type: **Server-based Application**
- **Global DC (Singapore)**: [https://api-console.zoho.com](https://api-console.zoho.com) — Type: **Server-based Application**

Each app produces a `client_id` and `client_secret`. Store both pairs in environment variables:

```
ZOHO_IN_CLIENT_ID=...
ZOHO_IN_CLIENT_SECRET=...
ZOHO_IN_REDIRECT_URI=https://your-app.com/auth/zoho/in/callback

ZOHO_COM_CLIENT_ID=...
ZOHO_COM_CLIENT_SECRET=...
ZOHO_COM_REDIRECT_URI=https://your-app.com/auth/zoho/com/callback
```

### 2.2 Required OAuth Scopes

```
ZohoBooks.contacts.ALL
ZohoBooks.invoices.ALL
ZohoBooks.bills.ALL
ZohoBooks.customerpayments.ALL
ZohoBooks.vendorpayments.ALL
ZohoBooks.creditnotes.ALL
ZohoBooks.debitnotes.ALL
ZohoBooks.settings.ALL
ZohoBooks.accountants.ALL
ZohoBooks.banking.ALL
```

Joined as a single scope string:
```
ZohoBooks.contacts.ALL,ZohoBooks.invoices.ALL,ZohoBooks.bills.ALL,ZohoBooks.customerpayments.ALL,ZohoBooks.vendorpayments.ALL,ZohoBooks.creditnotes.ALL,ZohoBooks.debitnotes.ALL,ZohoBooks.settings.ALL,ZohoBooks.accountants.ALL,ZohoBooks.banking.ALL
```

### 2.3 TypeScript Implementation

#### `src/zoho/auth/config.ts`

```typescript
export type ZohoDC = 'in' | 'com';

export interface ZohoDCConfig {
  accountsUrl: string;
  apiBaseUrl: string;
  clientId: string;
  clientSecret: string;
  redirectUri: string;
}

export const DC_CONFIG: Record<ZohoDC, ZohoDCConfig> = {
  in: {
    accountsUrl: 'https://accounts.zoho.in',
    apiBaseUrl: 'https://www.zohoapis.in/books/v3',
    clientId: process.env.ZOHO_IN_CLIENT_ID!,
    clientSecret: process.env.ZOHO_IN_CLIENT_SECRET!,
    redirectUri: process.env.ZOHO_IN_REDIRECT_URI!,
  },
  com: {
    accountsUrl: 'https://accounts.zoho.com',
    apiBaseUrl: 'https://www.zohoapis.com/books/v3',
    clientId: process.env.ZOHO_COM_CLIENT_ID!,
    clientSecret: process.env.ZOHO_COM_CLIENT_SECRET!,
    redirectUri: process.env.ZOHO_COM_REDIRECT_URI!,
  },
};

export const ZOHO_SCOPES = [
  'ZohoBooks.contacts.ALL',
  'ZohoBooks.invoices.ALL',
  'ZohoBooks.bills.ALL',
  'ZohoBooks.customerpayments.ALL',
  'ZohoBooks.vendorpayments.ALL',
  'ZohoBooks.creditnotes.ALL',
  'ZohoBooks.debitnotes.ALL',
  'ZohoBooks.settings.ALL',
  'ZohoBooks.accountants.ALL',
  'ZohoBooks.banking.ALL',
].join(',');
```

#### `src/zoho/auth/oauth.ts`

```typescript
import axios from 'axios';
import { DC_CONFIG, ZohoDC, ZOHO_SCOPES } from './config';

export interface ZohoTokens {
  access_token: string;
  refresh_token: string;
  expires_at: number; // Unix timestamp ms
  dc: ZohoDC;
}

/**
 * Step 1: Build authorization URL to redirect merchant to Zoho
 */
export function buildAuthorizationUrl(dc: ZohoDC, state: string): string {
  const config = DC_CONFIG[dc];
  const params = new URLSearchParams({
    response_type: 'code',
    client_id: config.clientId,
    scope: ZOHO_SCOPES,
    redirect_uri: config.redirectUri,
    access_type: 'offline',
    state,
  });
  return `${config.accountsUrl}/oauth/v2/auth?${params.toString()}`;
}

/**
 * Step 2: Exchange authorization code for tokens
 */
export async function exchangeCodeForTokens(
  dc: ZohoDC,
  code: string
): Promise<ZohoTokens> {
  const config = DC_CONFIG[dc];
  const params = new URLSearchParams({
    code,
    client_id: config.clientId,
    client_secret: config.clientSecret,
    redirect_uri: config.redirectUri,
    grant_type: 'authorization_code',
  });

  const response = await axios.post(
    `${config.accountsUrl}/oauth/v2/token`,
    params.toString(),
    { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
  );

  if (response.data.error) {
    throw new Error(`Zoho token exchange failed: ${response.data.error}`);
  }

  return {
    access_token: response.data.access_token,
    refresh_token: response.data.refresh_token,
    expires_at: Date.now() + response.data.expires_in * 1000,
    dc,
  };
}

/**
 * Step 3: Refresh access token
 */
export async function refreshAccessToken(tokens: ZohoTokens): Promise<ZohoTokens> {
  const config = DC_CONFIG[tokens.dc];
  const params = new URLSearchParams({
    refresh_token: tokens.refresh_token,
    client_id: config.clientId,
    client_secret: config.clientSecret,
    grant_type: 'refresh_token',
  });

  const response = await axios.post(
    `${config.accountsUrl}/oauth/v2/token`,
    params.toString(),
    { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
  );

  if (response.data.error) {
    throw new Error(`Zoho token refresh failed: ${response.data.error}`);
  }

  return {
    ...tokens,
    access_token: response.data.access_token,
    expires_at: Date.now() + response.data.expires_in * 1000,
  };
}

/**
 * Returns a valid access token, refreshing if within 5 minutes of expiry
 */
export async function getValidToken(
  tokens: ZohoTokens,
  saveTokens: (t: ZohoTokens) => Promise<void>
): Promise<string> {
  const BUFFER_MS = 5 * 60 * 1000; // 5 minutes
  if (Date.now() >= tokens.expires_at - BUFFER_MS) {
    const refreshed = await refreshAccessToken(tokens);
    await saveTokens(refreshed);
    return refreshed.access_token;
  }
  return tokens.access_token;
}
```

#### `src/zoho/auth/middleware.ts`

```typescript
import { Request, Response, NextFunction } from 'express';
import { getValidToken, ZohoTokens } from './oauth';
import { getTokensForTenant, saveTokensForTenant } from '../store/tokenStore';

/**
 * Express middleware that injects a valid Zoho client onto req
 * Usage: router.use(zohoAuthMiddleware)
 */
export async function zohoAuthMiddleware(
  req: Request & { zoho?: ZohoClient },
  res: Response,
  next: NextFunction
) {
  const shopId = req.headers['x-shop-id'] as string;
  if (!shopId) {
    return res.status(400).json({ error: 'Missing x-shop-id header' });
  }

  const tokens = await getTokensForTenant(shopId);
  if (!tokens) {
    return res.status(401).json({ error: 'Zoho not connected for this shop' });
  }

  try {
    const accessToken = await getValidToken(tokens, (t) =>
      saveTokensForTenant(shopId, t)
    );
    req.zoho = new ZohoClient(accessToken, tokens.dc);
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Failed to refresh Zoho token' });
  }
}
```

### 2.4 Multi-Tenant Token Store

#### `src/zoho/store/tokenStore.ts`

```typescript
import { ZohoTokens } from '../auth/oauth';

// Replace with your actual DB layer (Postgres, Redis, etc.)
// Key: shopId (Shopify shop domain), Value: ZohoTokens + orgId

export interface TenantZohoConfig {
  tokens: ZohoTokens;
  organizationId: string;
  organizationName: string;
}

// In-memory example — replace with DB
const store = new Map<string, TenantZohoConfig>();

export async function saveTokensForTenant(
  shopId: string,
  tokens: ZohoTokens
): Promise<void> {
  const existing = store.get(shopId);
  store.set(shopId, { ...existing!, tokens });
}

export async function getTokensForTenant(
  shopId: string
): Promise<ZohoTokens | null> {
  return store.get(shopId)?.tokens ?? null;
}

export async function saveTenantConfig(
  shopId: string,
  config: TenantZohoConfig
): Promise<void> {
  store.set(shopId, config);
}

export async function getTenantConfig(
  shopId: string
): Promise<TenantZohoConfig | null> {
  return store.get(shopId) ?? null;
}
```

---

## 3. Organization Management

### 3.1 ZohoClient Base Class

#### `src/zoho/client.ts`

```typescript
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import { DC_CONFIG, ZohoDC } from './auth/config';

export class ZohoClient {
  private http: AxiosInstance;
  private baseUrl: string;
  public dc: ZohoDC;

  constructor(accessToken: string, dc: ZohoDC) {
    this.dc = dc;
    this.baseUrl = DC_CONFIG[dc].apiBaseUrl;
    this.http = axios.create({
      baseURL: this.baseUrl,
      headers: {
        Authorization: `Zoho-oauthtoken ${accessToken}`,
        'Content-Type': 'application/json',
      },
    });
  }

  async get<T>(path: string, params?: Record<string, any>): Promise<T> {
    const response = await this.http.get(path, { params });
    return response.data;
  }

  async post<T>(path: string, data: any, params?: Record<string, any>): Promise<T> {
    const response = await this.http.post(path, data, { params });
    return response.data;
  }

  async put<T>(path: string, data: any, params?: Record<string, any>): Promise<T> {
    const response = await this.http.put(path, data, { params });
    return response.data;
  }

  async delete<T>(path: string, params?: Record<string, any>): Promise<T> {
    const response = await this.http.delete(path, { params });
    return response.data;
  }
}
```

### 3.2 Organization Service

#### `src/zoho/services/organizationService.ts`

```typescript
import { ZohoClient } from '../client';

export interface ZohoOrganization {
  organization_id: string;
  name: string;
  is_default_org: boolean;
  plan_name: string;
  currency_code: string;
  currency_id: string;
  time_zone: string;
  fiscal_year_start_month: number;
  is_org_active: boolean;
}

/**
 * Lists all organizations accessible via the authenticated token.
 * Called once during OAuth setup to let the merchant select their org.
 * NOTE: No organization_id query param needed for this endpoint.
 */
export async function listOrganizations(
  client: ZohoClient
): Promise<ZohoOrganization[]> {
  const response = await client.get<{ organizations: ZohoOrganization[] }>(
    '/organizations'
  );
  return response.organizations;
}

/**
 * Gets a single organization by ID
 */
export async function getOrganization(
  client: ZohoClient,
  organizationId: string
): Promise<ZohoOrganization> {
  const response = await client.get<{ organization: ZohoOrganization }>(
    `/organizations/${organizationId}`
  );
  return response.organization;
}
```

### 3.3 Multi-Org Selection Flow

During OAuth callback, after getting tokens:

```typescript
// src/routes/auth.ts
import { exchangeCodeForTokens } from '../zoho/auth/oauth';
import { listOrganizations } from '../zoho/services/organizationService';
import { saveTenantConfig } from '../zoho/store/tokenStore';
import { ZohoClient } from '../zoho/client';

router.get('/auth/zoho/:dc/callback', async (req, res) => {
  const dc = req.params.dc as 'in' | 'com';
  const { code, state } = req.query;

  // Verify state to prevent CSRF
  const shopId = verifyOAuthState(state as string);
  if (!shopId) return res.status(400).send('Invalid state');

  // Exchange code for tokens
  const tokens = await exchangeCodeForTokens(dc, code as string);
  const client = new ZohoClient(tokens.access_token, dc);

  // Fetch orgs — let merchant pick if multiple
  const orgs = await listOrganizations(client);
  
  if (orgs.length === 1) {
    // Auto-select single org
    await saveTenantConfig(shopId, {
      tokens,
      organizationId: orgs[0].organization_id,
      organizationName: orgs[0].name,
    });
    return res.redirect('/dashboard?zoho=connected');
  }

  // Multiple orgs: redirect to org selection UI
  // Store tokens temporarily, redirect with choices
  req.session.pendingZohoTokens = tokens;
  req.session.pendingShopId = shopId;
  return res.redirect(`/auth/zoho/select-org?orgs=${encodeURIComponent(JSON.stringify(orgs))}`);
});

router.post('/auth/zoho/select-org', async (req, res) => {
  const { organization_id } = req.body;
  const tokens = req.session.pendingZohoTokens;
  const shopId = req.session.pendingShopId;
  
  const client = new ZohoClient(tokens.access_token, tokens.dc);
  const org = await getOrganization(client, organization_id);

  await saveTenantConfig(shopId, {
    tokens,
    organizationId: org.organization_id,
    organizationName: org.name,
  });
  res.redirect('/dashboard?zoho=connected');
});
```

### 3.4 Org-Scoped Request Helper

All API calls after setup inject `organization_id`:

```typescript
// src/zoho/services/base.ts
import { ZohoClient } from '../client';
import { getTenantConfig } from '../store/tokenStore';

export class OrgScopedClient {
  constructor(
    private client: ZohoClient,
    private organizationId: string
  ) {}

  async get<T>(path: string, params: Record<string, any> = {}): Promise<T> {
    return this.client.get<T>(path, { ...params, organization_id: this.organizationId });
  }

  async post<T>(path: string, data: any, params: Record<string, any> = {}): Promise<T> {
    return this.client.post<T>(path, data, { ...params, organization_id: this.organizationId });
  }

  async put<T>(path: string, data: any, params: Record<string, any> = {}): Promise<T> {
    return this.client.put<T>(path, data, { ...params, organization_id: this.organizationId });
  }

  async delete<T>(path: string, params: Record<string, any> = {}): Promise<T> {
    return this.client.delete<T>(path, { ...params, organization_id: this.organizationId });
  }
}

export async function getOrgClient(shopId: string): Promise<OrgScopedClient> {
  const config = await getTenantConfig(shopId);
  if (!config) throw new Error(`No Zoho config for shop ${shopId}`);
  // In production, also refresh token here if needed
  const client = new ZohoClient(config.tokens.access_token, config.tokens.dc);
  return new OrgScopedClient(client, config.organizationId);
}
```

---

## 4. Core Accounting API Integrations

### 4.1 Contacts

**Endpoint base:** `/contacts`  
**Scope:** `ZohoBooks.contacts.ALL`

#### TypeScript Interface

```typescript
export interface ZohoBillingAddress {
  address?: string;
  city?: string;
  state?: string;
  zip?: string;
  country?: string;
}

export type GstTreatment = 'business_gst' | 'business_none' | 'overseas' | 'consumer';

export interface ZohoContactCreate {
  contact_name: string;
  contact_type: 'customer' | 'vendor';
  company_name?: string;
  email?: string;
  currency_id?: string;
  payment_terms?: number;
  billing_address?: ZohoBillingAddress;
  shipping_address?: ZohoBillingAddress;
  // India-specific
  gst_no?: string;
  gst_treatment?: GstTreatment;
  place_of_contact?: string; // 2-letter state code
  pan_no?: string;
}

export interface ZohoContact extends ZohoContactCreate {
  contact_id: string;
  status: 'active' | 'inactive';
  created_time: string;
}
```

#### Contact Service

```typescript
// src/zoho/services/contactService.ts
import { OrgScopedClient } from './base';

export async function createContact(
  client: OrgScopedClient,
  data: ZohoContactCreate
): Promise<ZohoContact> {
  const response = await client.post<{ contact: ZohoContact }>('/contacts', data);
  if (!response.contact) throw new Error('Failed to create contact');
  return response.contact;
}

export async function findContactByEmail(
  client: OrgScopedClient,
  email: string
): Promise<ZohoContact | null> {
  const response = await client.get<{ contacts: ZohoContact[] }>('/contacts', {
    search_text: email,
    contact_type: 'customer',
  });
  return response.contacts.find((c) => c.email === email) ?? null;
}

export async function upsertContactFromShopify(
  client: OrgScopedClient,
  shopifyCustomer: ShopifyCustomer,
  dc: 'in' | 'com'
): Promise<ZohoContact> {
  // Try to find existing contact
  const existing = await findContactByEmail(client, shopifyCustomer.email);
  if (existing) return existing;

  const payload: ZohoContactCreate = {
    contact_name: `${shopifyCustomer.first_name} ${shopifyCustomer.last_name}`.trim()
      || shopifyCustomer.email,
    contact_type: 'customer',
    company_name: shopifyCustomer.default_address?.company || undefined,
    email: shopifyCustomer.email,
    billing_address: {
      address: shopifyCustomer.default_address?.address1,
      city: shopifyCustomer.default_address?.city,
      state: shopifyCustomer.default_address?.province,
      zip: shopifyCustomer.default_address?.zip,
      country: shopifyCustomer.default_address?.country,
    },
  };

  // India-specific: add GST fields if the org is India DC
  if (dc === 'in') {
    // Check for GSTIN in customer metafields (custom implementation)
    const gstin = shopifyCustomer.metafields?.find((m) => m.key === 'gstin')?.value;
    if (gstin && gstin.length === 15) {
      payload.gst_no = gstin;
      payload.gst_treatment = 'business_gst';
      // Place of contact = first 2 digits of GSTIN (state code)
      payload.place_of_contact = GSTIN_STATE_CODE_MAP[gstin.substring(0, 2)];
    } else if (shopifyCustomer.default_address?.country_code === 'IN') {
      payload.gst_treatment = 'consumer';
      payload.place_of_contact = getStateCodeFromProvince(
        shopifyCustomer.default_address.province_code
      );
    } else {
      payload.gst_treatment = 'overseas';
    }
  }

  return createContact(client, payload);
}
```

#### GSTIN State Code Lookup

```typescript
// First 2 digits of GSTIN = state code from GST portal (numeric)
// Maps to Zoho's 2-letter state codes
export const GSTIN_PREFIX_TO_ZOHO_STATE: Record<string, string> = {
  '01': 'JK', '02': 'HP', '03': 'PB', '04': 'CH', '05': 'UT',
  '06': 'HR', '07': 'DL', '08': 'RJ', '09': 'UP', '10': 'BR',
  '11': 'SK', '12': 'AR', '13': 'NL', '14': 'MN', '15': 'MZ',
  '16': 'TR', '17': 'ML', '18': 'AS', '19': 'WB', '20': 'JH',
  '21': 'OR', '22': 'CT', '23': 'MP', '24': 'GJ', '25': 'DD',
  '26': 'DN', '27': 'MH', '29': 'KA', '30': 'GA', '31': 'LD',
  '32': 'KL', '33': 'TN', '34': 'PY', '35': 'AN', '36': 'TS',
  '37': 'AP',
};
```

---

### 4.2 Invoices

**Endpoint base:** `/invoices`  
**Scope:** `ZohoBooks.invoices.ALL`

#### TypeScript Interface

```typescript
export interface ZohoLineItem {
  item_id?: string;
  name: string;
  description?: string;
  quantity: number;
  rate: number;
  unit?: string;
  tax_id?: string;
  account_id?: string;
  discount?: string;
  // India-specific
  hsn_or_sac?: string;
  product_type?: 'goods' | 'services';
  reverse_charge_tax_id?: string;
  tds_tax_id?: string;
}

export interface ZohoInvoiceCreate {
  customer_id: string;
  invoice_number?: string;
  date: string;           // yyyy-mm-dd
  due_date?: string;
  currency_id?: string;
  exchange_rate?: number;
  line_items: ZohoLineItem[];
  discount?: string;
  is_discount_before_tax?: boolean;
  is_inclusive_tax?: boolean;
  payment_terms?: number;
  notes?: string;
  terms?: string;
  // India-specific
  place_of_supply?: string;  // 2-letter state code
  gst_no?: string;
  gst_treatment?: GstTreatment;
  is_reverse_charge_applied?: boolean;
  tcs_tax_id?: string;
}

export interface ZohoInvoice extends ZohoInvoiceCreate {
  invoice_id: string;
  status: 'draft' | 'sent' | 'viewed' | 'partially_paid' | 'paid' | 'overdue' | 'void';
  total: number;
  balance: number;
  created_time: string;
  // E-invoice fields (India)
  irn?: string;
  ack_no?: string;
  ack_date?: string;
  qr_code?: string;
  e_invoice_status?: string;
}
```

#### Invoice Service — Core

```typescript
// src/zoho/services/invoiceService.ts
import { OrgScopedClient } from './base';

export async function createInvoice(
  client: OrgScopedClient,
  data: ZohoInvoiceCreate
): Promise<ZohoInvoice> {
  const response = await client.post<{ invoice: ZohoInvoice }>('/invoices', data);
  if (!response.invoice) throw new Error('Failed to create invoice');
  return response.invoice;
}

export async function getInvoice(
  client: OrgScopedClient,
  invoiceId: string
): Promise<ZohoInvoice> {
  const response = await client.get<{ invoice: ZohoInvoice }>(`/invoices/${invoiceId}`);
  return response.invoice;
}

export async function voidInvoice(
  client: OrgScopedClient,
  invoiceId: string
): Promise<void> {
  await client.post(`/invoices/${invoiceId}/status/void`, {});
}

export async function markInvoiceSent(
  client: OrgScopedClient,
  invoiceId: string
): Promise<void> {
  await client.post(`/invoices/${invoiceId}/status/sent`, {});
}

export async function listInvoices(
  client: OrgScopedClient,
  params: {
    status?: string;
    customer_id?: string;
    date_start?: string;
    date_end?: string;
    last_modified_time?: string;
    page?: number;
    per_page?: number;
  }
): Promise<{ invoices: ZohoInvoice[]; has_more_page: boolean }> {
  const response = await client.get<{
    invoices: ZohoInvoice[];
    page_context: { has_more_page: boolean };
  }>('/invoices', { per_page: 200, ...params });
  return {
    invoices: response.invoices,
    has_more_page: response.page_context.has_more_page,
  };
}
```

#### Shopify Order → Zoho Invoice Conversion

```typescript
// src/zoho/mappers/shopifyToInvoice.ts
import { ZohoInvoiceCreate, ZohoLineItem } from '../types';
import { determineGstTaxId } from '../gst/taxEngine';

/**
 * Maps a Shopify order to a Zoho Books invoice payload.
 * @param order - Shopify order object
 * @param customerId - Zoho Books contact ID (already created)
 * @param taxIds - Pre-fetched tax ID map from Zoho Books settings
 * @param orgState - 2-letter state code of the org (e.g. 'KA' for Karnataka)
 * @param dc - 'in' | 'com'
 */
export function mapShopifyOrderToInvoice(
  order: ShopifyOrder,
  customerId: string,
  taxIds: ZohoTaxIdMap,
  orgState: string,
  dc: 'in' | 'com'
): ZohoInvoiceCreate {
  const invoiceDate = order.created_at.split('T')[0]; // yyyy-mm-dd
  const dueDate = addDays(invoiceDate, 0); // Due immediately for Shopify orders

  // Determine place of supply (India only)
  const placeOfSupply = dc === 'in'
    ? getPlaceOfSupply(order.shipping_address?.province_code, order.billing_address?.province_code)
    : undefined;

  // Build line items
  const lineItems: ZohoLineItem[] = order.line_items.map((item) => {
    const isIntraState = dc === 'in' && placeOfSupply === orgState;
    const gstRate = extractGstRateFromShopifyTax(item.tax_lines);
    const taxId = dc === 'in'
      ? determineGstTaxId(gstRate, isIntraState, taxIds)
      : taxIds.sgGst9; // Default SG GST

    return {
      name: item.name,
      description: `SKU: ${item.sku || 'N/A'} | Shopify Line Item ID: ${item.id}`,
      quantity: item.quantity,
      rate: parseFloat(item.price),
      unit: 'Nos',
      tax_id: taxId,
      // India only
      ...(dc === 'in' && {
        hsn_or_sac: getHsnCode(item), // From metafields or product type
        product_type: 'goods',
      }),
    };
  });

  // Add shipping as a line item if applicable
  if (parseFloat(order.total_shipping_price_set?.shop_money?.amount || '0') > 0) {
    lineItems.push({
      name: 'Shipping',
      quantity: 1,
      rate: parseFloat(order.total_shipping_price_set.shop_money.amount),
      ...(dc === 'in' && {
        hsn_or_sac: '996812', // SAC for freight transport
        product_type: 'services',
        tax_id: determineGstTaxId(18, placeOfSupply === orgState, taxIds),
      }),
      ...(dc === 'com' && { tax_id: taxIds.sgGst9 }),
    });
  }

  const payload: ZohoInvoiceCreate = {
    customer_id: customerId,
    invoice_number: `SHO-${order.order_number}`,
    date: invoiceDate,
    due_date: dueDate,
    line_items: lineItems,
    notes: `Shopify Order #${order.name}`,
    is_inclusive_tax: false,
  };

  // India-specific fields
  if (dc === 'in') {
    const gstInfo = extractCustomerGstInfo(order);
    payload.place_of_supply = placeOfSupply;
    payload.gst_treatment = gstInfo.treatment;
    payload.gst_no = gstInfo.gstin;
  }

  return payload;
}

function getPlaceOfSupply(
  shippingProvinceCode?: string,
  billingProvinceCode?: string
): string {
  const code = shippingProvinceCode || billingProvinceCode;
  if (!code) return 'MH'; // Default fallback
  return SHOPIFY_PROVINCE_TO_ZOHO_STATE[code] || 'MH';
}

function extractGstRateFromShopifyTax(taxLines: ShopifyTaxLine[]): number {
  const total = taxLines.reduce((sum, t) => sum + parseFloat(t.rate), 0);
  return Math.round(total * 100); // Convert 0.18 → 18
}

function getHsnCode(item: ShopifyLineItem): string {
  // Check product metafields for HSN code
  const hsn = item.properties?.find((p) => p.name === 'hsn_code')?.value;
  if (hsn) return hsn;
  // Default by product type — extend this map for your catalog
  return '63079090'; // Generic goods fallback
}

// Shopify province_code → Zoho 2-letter state code
export const SHOPIFY_PROVINCE_TO_ZOHO_STATE: Record<string, string> = {
  'IN-AP': 'AP', 'IN-AR': 'AR', 'IN-AS': 'AS', 'IN-BR': 'BR',
  'IN-CT': 'CT', 'IN-GA': 'GA', 'IN-GJ': 'GJ', 'IN-HR': 'HR',
  'IN-HP': 'HP', 'IN-JH': 'JH', 'IN-KA': 'KA', 'IN-KL': 'KL',
  'IN-MP': 'MP', 'IN-MH': 'MH', 'IN-MN': 'MN', 'IN-ML': 'ML',
  'IN-MZ': 'MZ', 'IN-NL': 'NL', 'IN-OR': 'OR', 'IN-PB': 'PB',
  'IN-RJ': 'RJ', 'IN-SK': 'SK', 'IN-TN': 'TN', 'IN-TS': 'TS',
  'IN-TR': 'TR', 'IN-UP': 'UP', 'IN-UT': 'UT', 'IN-WB': 'WB',
  'IN-AN': 'AN', 'IN-CH': 'CH', 'IN-DN': 'DN', 'IN-DD': 'DD',
  'IN-DL': 'DL', 'IN-JK': 'JK', 'IN-LA': 'LA', 'IN-LD': 'LD',
  'IN-PY': 'PY',
};
```

#### CGST+SGST vs IGST Tax Selection

```typescript
// src/zoho/gst/taxEngine.ts

export interface ZohoTaxIdMap {
  // India — by slab
  igst5: string;   igst12: string;  igst18: string;  igst28: string;
  cgstSgst5: string; cgstSgst12: string; cgstSgst18: string; cgstSgst28: string;
  igst0: string;   // zero-rated / nil
  // Singapore
  sgGst9: string;  // GST9 [SR]
  sgGst0: string;  // GST0 [ZR]
}

/**
 * Returns the correct Zoho tax_id for a given GST rate + supply type.
 * intraState = true → use CGST+SGST group
 * intraState = false → use IGST
 */
export function determineGstTaxId(
  gstRatePercent: number,
  isIntraState: boolean,
  taxIds: ZohoTaxIdMap
): string {
  const rateMap: Record<number, { igst: string; cgstSgst: string }> = {
    0:  { igst: taxIds.igst0,   cgstSgst: taxIds.igst0 },
    5:  { igst: taxIds.igst5,   cgstSgst: taxIds.cgstSgst5 },
    12: { igst: taxIds.igst12,  cgstSgst: taxIds.cgstSgst12 },
    18: { igst: taxIds.igst18,  cgstSgst: taxIds.cgstSgst18 },
    28: { igst: taxIds.igst28,  cgstSgst: taxIds.cgstSgst28 },
  };

  const entry = rateMap[gstRatePercent];
  if (!entry) {
    console.warn(`Unknown GST rate: ${gstRatePercent}%, defaulting to IGST 18%`);
    return taxIds.igst18;
  }
  return isIntraState ? entry.cgstSgst : entry.igst;
}
```

---

### 4.3 Bills & Expenses

**Endpoint base:** `/bills`  
**Scope:** `ZohoBooks.bills.ALL`

Use bills to record Shopify platform fees, payment gateway charges, and shipping cost invoices.

#### TypeScript Interface

```typescript
export interface ZohoBillLineItem {
  account_id: string;
  name: string;
  description?: string;
  quantity: number;
  rate: number;
  tax_id?: string;
  // India-specific
  hsn_or_sac?: string;
  product_type?: 'goods' | 'services';
  tds_tax_id?: string;
  reverse_charge_tax_id?: string;
}

export interface ZohoBillCreate {
  vendor_id: string;
  bill_number?: string;
  date: string;
  due_date?: string;
  line_items: ZohoBillLineItem[];
  currency_id?: string;
  exchange_rate?: number;
  // India-specific
  source_of_supply?: string;
  destination_of_supply?: string;
  gst_no?: string;
  gst_treatment?: GstTreatment;
  is_reverse_charge_applied?: boolean;
}
```

#### Bill Service

```typescript
// src/zoho/services/billService.ts
import { OrgScopedClient } from './base';

export async function createBill(
  client: OrgScopedClient,
  data: ZohoBillCreate
): Promise<ZohoBill> {
  const response = await client.post<{ bill: ZohoBill }>('/bills', data);
  if (!response.bill) throw new Error('Failed to create bill');
  return response.bill;
}

/**
 * Creates a Shopify fee bill (payment gateway, subscription, etc.)
 * vendorId: pre-created Shopify Inc. vendor contact in Zoho Books
 */
export async function createShopifyFeeBill(
  client: OrgScopedClient,
  params: {
    vendorId: string;
    date: string;
    feeType: 'gateway' | 'subscription' | 'shipping' | 'other';
    amount: number;
    reference: string;
    expenseAccountId: string;
    dc: 'in' | 'com';
  }
): Promise<ZohoBill> {
  const feeLabels: Record<string, string> = {
    gateway: 'Payment Gateway Fee',
    subscription: 'Shopify Subscription Fee',
    shipping: 'Shipping Label Cost',
    other: 'Shopify Miscellaneous Fee',
  };

  return createBill(client, {
    vendor_id: params.vendorId,
    bill_number: `FEE-${params.reference}`,
    date: params.date,
    line_items: [{
      account_id: params.expenseAccountId,
      name: feeLabels[params.feeType],
      quantity: 1,
      rate: params.amount,
      ...(params.dc === 'in' && {
        hsn_or_sac: '998431', // SAC: IT enabled services
        product_type: 'services',
        // Shopify Inc. is overseas → reverse charge applies for Indian GST
      }),
    }],
    ...(params.dc === 'in' && {
      gst_treatment: 'overseas',
      is_reverse_charge_applied: true,
    }),
  });
}
```

---

### 4.4 Chart of Accounts

**Endpoint base:** `/chartofaccounts`  
**Scope:** `ZohoBooks.accountants.ALL`

#### Shopify Account Setup

Run this once per connected Zoho org during onboarding to create the required accounts:

```typescript
// src/zoho/setup/setupAccounts.ts
import { OrgScopedClient } from '../services/base';

export interface ShopifyAccountIds {
  shopifyRevenue: string;
  shopifyRefunds: string;
  shopifyGatewayFees: string;
  shopifySubscriptionFees: string;
  shopifyShippingFees: string;
  shopifyPaymentsCash: string; // "bank" account for payout reconciliation
  gstInputIgst?: string;      // India only
  gstInputCgst?: string;
  gstInputSgst?: string;
  gstOutputIgst?: string;
  gstOutputCgst?: string;
  gstOutputSgst?: string;
}

interface AccountSetupSpec {
  account_name: string;
  account_type: string;
  description: string;
}

const SHOPIFY_ACCOUNTS: AccountSetupSpec[] = [
  { account_name: 'Shopify Sales Revenue', account_type: 'income', description: 'Revenue from Shopify store sales' },
  { account_name: 'Shopify Refunds', account_type: 'income', description: 'Refunds issued on Shopify orders (contra-revenue)' },
  { account_name: 'Shopify Payment Gateway Fees', account_type: 'expense', description: 'Transaction fees charged by Shopify Payments / Stripe' },
  { account_name: 'Shopify Subscription Fees', account_type: 'expense', description: 'Monthly Shopify platform subscription cost' },
  { account_name: 'Shopify Shipping Costs', account_type: 'expense', description: 'Shipping label costs from Shopify' },
  { account_name: 'Shopify Payments Account', account_type: 'bank', description: 'Virtual bank account tracking Shopify payout balance' },
];

const INDIA_GST_ACCOUNTS: AccountSetupSpec[] = [
  { account_name: 'GST Input - IGST', account_type: 'other_current_asset', description: 'Input IGST credit' },
  { account_name: 'GST Input - CGST', account_type: 'other_current_asset', description: 'Input CGST credit' },
  { account_name: 'GST Input - SGST', account_type: 'other_current_asset', description: 'Input SGST credit' },
  { account_name: 'GST Output - IGST', account_type: 'other_current_liability', description: 'Output IGST payable' },
  { account_name: 'GST Output - CGST', account_type: 'other_current_liability', description: 'Output CGST payable' },
  { account_name: 'GST Output - SGST', account_type: 'other_current_liability', description: 'Output SGST payable' },
];

export async function setupShopifyAccounts(
  client: OrgScopedClient,
  dc: 'in' | 'com'
): Promise<ShopifyAccountIds> {
  const specs = dc === 'in'
    ? [...SHOPIFY_ACCOUNTS, ...INDIA_GST_ACCOUNTS]
    : SHOPIFY_ACCOUNTS;

  const created: Record<string, string> = {};

  for (const spec of specs) {
    // Check if account already exists
    const existing = await findAccountByName(client, spec.account_name);
    if (existing) {
      created[spec.account_name] = existing.account_id;
      continue;
    }

    const response = await client.post<{ chartofaccount: { account_id: string } }>(
      '/chartofaccounts',
      spec
    );
    created[spec.account_name] = response.chartofaccount.account_id;
    await sleep(200); // Respect rate limits
  }

  return {
    shopifyRevenue: created['Shopify Sales Revenue'],
    shopifyRefunds: created['Shopify Refunds'],
    shopifyGatewayFees: created['Shopify Payment Gateway Fees'],
    shopifySubscriptionFees: created['Shopify Subscription Fees'],
    shopifyShippingFees: created['Shopify Shipping Costs'],
    shopifyPaymentsCash: created['Shopify Payments Account'],
    ...(dc === 'in' && {
      gstInputIgst: created['GST Input - IGST'],
      gstInputCgst: created['GST Input - CGST'],
      gstInputSgst: created['GST Input - SGST'],
      gstOutputIgst: created['GST Output - IGST'],
      gstOutputCgst: created['GST Output - CGST'],
      gstOutputSgst: created['GST Output - SGST'],
    }),
  };
}

async function findAccountByName(
  client: OrgScopedClient,
  name: string
): Promise<{ account_id: string } | null> {
  const response = await client.get<{ chartofaccounts: Array<{ account_id: string; account_name: string }> }>(
    '/chartofaccounts',
    { filter_by: 'AccountType.All', search_text: name }
  );
  return response.chartofaccounts.find((a) => a.account_name === name) ?? null;
}

function sleep(ms: number) { return new Promise((r) => setTimeout(r, ms)); }
```

---

### 4.5 Journal Entries

**Endpoint base:** `/journals`  
**Scope:** `ZohoBooks.accountants.ALL`

```typescript
// src/zoho/services/journalService.ts
import { OrgScopedClient } from './base';

export interface ZohoJournalLineItem {
  account_id: string;
  debit_or_credit: 'debit' | 'credit';
  amount: number;
  description?: string;
  tax_id?: string;
}

export interface ZohoJournalCreate {
  journal_date: string;
  journal_number?: string;
  reference_number?: string;
  notes?: string;
  currency_id?: string;
  exchange_rate?: number;
  line_items: ZohoJournalLineItem[];
  status?: 'draft' | 'published';
}

export async function createJournal(
  client: OrgScopedClient,
  data: ZohoJournalCreate
): Promise<{ journal_id: string }> {
  const response = await client.post<{ journal: { journal_id: string } }>('/journals', data);
  return response.journal;
}

export async function publishJournal(
  client: OrgScopedClient,
  journalId: string
): Promise<void> {
  await client.post(`/journals/${journalId}/status/publish`, {});
}

/**
 * Creates an opening balance journal entry for a new Shopify store connection.
 * Use this to set initial AR/AP balances when first connecting.
 */
export async function createOpeningBalanceJournal(
  client: OrgScopedClient,
  params: {
    date: string;
    arAccountId: string;
    equityAccountId: string;
    openingBalance: number;
    reference: string;
  }
): Promise<void> {
  const journal = await createJournal(client, {
    journal_date: params.date,
    reference_number: params.reference,
    notes: 'Opening balance on Zoho Books connection',
    status: 'draft',
    line_items: [
      {
        account_id: params.arAccountId,
        debit_or_credit: 'debit',
        amount: params.openingBalance,
        description: 'Opening AR balance from Shopify',
      },
      {
        account_id: params.equityAccountId,
        debit_or_credit: 'credit',
        amount: params.openingBalance,
        description: 'Opening equity offset',
      },
    ],
  });

  await publishJournal(client, journal.journal_id);
}
```

---

### 4.6 Bank Accounts

**Endpoint base:** `/bankaccounts`  
**Scope:** `ZohoBooks.banking.ALL`

```typescript
// src/zoho/services/bankAccountService.ts
import { OrgScopedClient } from './base';

export interface ZohoBankAccount {
  account_id: string;
  account_name: string;
  account_type: string;
  currency_code: string;
  current_balance: number;
}

/**
 * Creates a "Shopify Payments" virtual bank account for payout tracking.
 * Call this once during setup. Store the returned account_id.
 */
export async function createShopifyPaymentsAccount(
  client: OrgScopedClient,
  currencyCode: string
): Promise<ZohoBankAccount> {
  const response = await client.post<{ bankaccount: ZohoBankAccount }>('/bankaccounts', {
    account_name: 'Shopify Payments',
    account_type: 'bank',
    currency_code: currencyCode,
    description: 'Virtual bank account tracking Shopify payout balances',
  });
  return response.bankaccount;
}

/**
 * Imports Shopify payout transactions as a bank statement feed.
 * transactions: array of payout line items from Shopify Payouts API
 */
export async function importShopifyPayoutStatement(
  client: OrgScopedClient,
  bankAccountId: string,
  transactions: ShopifyPayoutTransaction[]
): Promise<void> {
  const zohoTransactions = transactions.map((t) => ({
    date: t.processed_at.split('T')[0],
    debit_or_credit: t.type === 'payout' ? 'credit'
      : t.type === 'refund' ? 'debit'
      : t.amount >= 0 ? 'credit' : 'debit',
    amount: Math.abs(t.amount),
    payee: t.source_type || 'Shopify',
    description: `${t.type} | ${t.source_id || ''}`,
    reference_number: t.id.toString(),
  }));

  await client.post(`/bankaccounts/${bankAccountId}/statements`, {
    date_format: 'yyyy-MM-dd',
    transactions: zohoTransactions,
  });
}
```

---

### 4.7 Bank Transactions & Reconciliation

**Endpoint base:** `/banktransactions`  
**Scope:** `ZohoBooks.banking.ALL`

```typescript
// src/zoho/services/reconciliationService.ts
import { OrgScopedClient } from './base';

export interface UncategorizedTransaction {
  transaction_id: string;
  date: string;
  amount: number;
  debit_or_credit: 'debit' | 'credit';
  description: string;
  reference_number: string;
  status: 'uncategorized' | 'categorized' | 'matched' | 'excluded';
}

/**
 * Lists all uncategorized transactions in the Shopify Payments bank account
 */
export async function listUncategorizedTransactions(
  client: OrgScopedClient,
  bankAccountId: string
): Promise<UncategorizedTransaction[]> {
  const response = await client.get<{ banktransactions: UncategorizedTransaction[] }>(
    '/banktransactions',
    { account_id: bankAccountId, filter_by: 'Status.Uncategorized', per_page: 200 }
  );
  return response.banktransactions;
}

/**
 * Gets suggested matches for an uncategorized transaction.
 * Zoho Books suggests existing payments/invoices that could match.
 */
export async function getMatchSuggestions(
  client: OrgScopedClient,
  transactionId: string
): Promise<any[]> {
  const response = await client.get<{ matching_transactions: any[] }>(
    `/banktransactions/uncategorized/${transactionId}/match`
  );
  return response.matching_transactions ?? [];
}

/**
 * Matches an uncategorized bank transaction to an existing customer payment
 */
export async function matchTransactionToPayment(
  client: OrgScopedClient,
  transactionId: string,
  paymentId: string,
  transactionType: 'customerpayment' | 'creditnoterefund' | 'expense'
): Promise<void> {
  await client.post(
    `/banktransactions/uncategorized/${transactionId}/match`,
    {
      transactions_to_be_matched: [{
        transaction_id: paymentId,
        transaction_type: transactionType,
      }],
    }
  );
}

/**
 * Categorizes an uncategorized debit as an expense (for Shopify fees)
 */
export async function categorizeAsExpense(
  client: OrgScopedClient,
  transactionId: string,
  accountId: string,
  vendorId: string,
  description: string
): Promise<void> {
  await client.post(
    `/banktransactions/uncategorized/${transactionId}/categorize/expenses`,
    {
      account_id: accountId,
      vendor_id: vendorId,
      description,
    }
  );
}

/**
 * Categorizes an uncategorized credit as a customer payment
 */
export async function categorizeAsCustomerPayment(
  client: OrgScopedClient,
  transactionId: string,
  customerId: string,
  invoiceId: string,
  amount: number
): Promise<void> {
  await client.post(
    `/banktransactions/uncategorized/${transactionId}/categorize/customerpayments`,
    {
      customer_id: customerId,
      invoices: [{ invoice_id: invoiceId, amount_applied: amount }],
      payment_mode: 'autotransaction',
    }
  );
}
```

---

### 4.8 Customer Payments

**Endpoint base:** `/customerpayments`  
**Scope:** `ZohoBooks.customerpayments.ALL`

```typescript
// src/zoho/services/paymentService.ts
import { OrgScopedClient } from './base';

export type PaymentMode = 'check' | 'cash' | 'creditcard' | 'banktransfer' | 'bankremittance' | 'autotransaction' | 'others';

export interface ZohoCustomerPaymentCreate {
  customer_id: string;
  amount: number;
  date: string;
  payment_mode: PaymentMode;
  paid_through_account_id?: string;
  reference_number?: string;
  invoices?: Array<{ invoice_id: string; amount_applied: number }>;
  // India-specific
  tax_amount_withheld?: number; // TDS withheld
}

export async function createCustomerPayment(
  client: OrgScopedClient,
  data: ZohoCustomerPaymentCreate
): Promise<{ payment_id: string }> {
  const response = await client.post<{ payment: { payment_id: string } }>(
    '/customerpayments',
    data
  );
  return response.payment;
}

/**
 * Records a Shopify payment capture as a Zoho customer payment
 */
export async function recordShopifyPayment(
  client: OrgScopedClient,
  params: {
    customerId: string;
    invoiceId: string;
    amount: number;
    paymentDate: string;
    shopifyTransactionId: string;
    bankAccountId: string;
    paymentGateway: string;
  }
): Promise<{ payment_id: string }> {
  const mode: PaymentMode = params.paymentGateway.toLowerCase().includes('cod')
    ? 'cash'
    : 'autotransaction';

  return createCustomerPayment(client, {
    customer_id: params.customerId,
    amount: params.amount,
    date: params.paymentDate,
    payment_mode: mode,
    paid_through_account_id: params.bankAccountId,
    reference_number: params.shopifyTransactionId,
    invoices: [{ invoice_id: params.invoiceId, amount_applied: params.amount }],
  });
}
```

---

### 4.9 Credit Notes

**Endpoint base:** `/creditnotes`  
**Scope:** `ZohoBooks.creditnotes.ALL`

```typescript
// src/zoho/services/creditNoteService.ts
import { OrgScopedClient } from './base';

export interface ZohoCreditNoteCreate {
  customer_id: string;
  creditnote_number?: string;
  date: string;
  line_items: ZohoLineItem[];
  // India-specific
  place_of_supply?: string;
  gst_no?: string;
  gst_treatment?: GstTreatment;
  is_reverse_charge_applied?: boolean;
}

export async function createCreditNote(
  client: OrgScopedClient,
  data: ZohoCreditNoteCreate
): Promise<{ creditnote_id: string }> {
  const response = await client.post<{ creditnote: { creditnote_id: string } }>(
    '/creditnotes',
    data
  );
  return response.creditnote;
}

export async function applyCreditNoteToInvoice(
  client: OrgScopedClient,
  creditNoteId: string,
  invoiceId: string,
  amountApplied: number
): Promise<void> {
  await client.post(`/creditnotes/${creditNoteId}/invoices`, {
    invoices: [{ invoice_id: invoiceId, amount_applied: amountApplied }],
  });
}

/**
 * Handles a Shopify refund by creating a credit note and applying it to the original invoice
 */
export async function handleShopifyRefund(
  client: OrgScopedClient,
  params: {
    shopifyRefund: ShopifyRefund;
    originalInvoice: ZohoInvoice;
    taxIds: ZohoTaxIdMap;
    orgState: string;
    dc: 'in' | 'com';
  }
): Promise<void> {
  const { shopifyRefund, originalInvoice, taxIds, orgState, dc } = params;

  const lineItems: ZohoLineItem[] = shopifyRefund.refund_line_items.map((item) => {
    const isIntraState = dc === 'in' && originalInvoice.place_of_supply === orgState;
    const taxRate = 18; // Ideally match the original line item's tax rate
    return {
      name: `Refund: ${item.line_item.name}`,
      quantity: item.quantity,
      rate: parseFloat(item.line_item.price),
      tax_id: dc === 'in'
        ? determineGstTaxId(taxRate, isIntraState, taxIds)
        : taxIds.sgGst9,
      ...(dc === 'in' && {
        hsn_or_sac: item.line_item.properties?.find((p) => p.name === 'hsn_code')?.value || '63079090',
        product_type: 'goods' as const,
      }),
    };
  });

  const creditNote = await createCreditNote(client, {
    customer_id: originalInvoice.customer_id,
    creditnote_number: `CN-SHO-${shopifyRefund.id}`,
    date: shopifyRefund.created_at.split('T')[0],
    line_items: lineItems,
    ...(dc === 'in' && {
      place_of_supply: originalInvoice.place_of_supply,
      gst_treatment: originalInvoice.gst_treatment,
      gst_no: originalInvoice.gst_no,
    }),
  });

  // Apply credit note to original invoice
  const refundAmount = shopifyRefund.refund_line_items.reduce(
    (sum, item) => sum + parseFloat(item.line_item.price) * item.quantity,
    0
  );
  await applyCreditNoteToInvoice(
    client,
    creditNote.creditnote_id,
    originalInvoice.invoice_id,
    refundAmount
  );
}
```

---

### 4.10 Recurring Invoices

**Endpoint base:** `/recurringinvoices`  
**Scope:** `ZohoBooks.invoices.ALL`

```typescript
// src/zoho/services/recurringInvoiceService.ts
import { OrgScopedClient } from './base';

export interface ZohoRecurringInvoiceCreate {
  recurrence_name: string;        // Unique, max 100 chars
  customer_id: string;
  recurrence_frequency: 'days' | 'weeks' | 'months' | 'years';
  repeat_every: number;           // e.g., 1 = every month
  start_date: string;             // yyyy-mm-dd
  end_date?: string;
  line_items: ZohoLineItem[];
  // India-specific
  place_of_supply?: string;
  gst_treatment?: GstTreatment;
  gst_no?: string;
}

export async function createRecurringInvoice(
  client: OrgScopedClient,
  data: ZohoRecurringInvoiceCreate
): Promise<{ recurring_invoice_id: string }> {
  const response = await client.post<{ recurring_invoice: { recurring_invoice_id: string } }>(
    '/recurringinvoices',
    data
  );
  return response.recurring_invoice;
}

export async function stopRecurringInvoice(
  client: OrgScopedClient,
  recurringInvoiceId: string
): Promise<void> {
  await client.post(`/recurringinvoices/${recurringInvoiceId}/status/stop`, {});
}

/**
 * Creates a recurring subscription invoice for a Shopify subscription product
 */
export async function setupShopifySubscriptionInvoice(
  client: OrgScopedClient,
  params: {
    customerId: string;
    shopId: string;
    planName: string;
    monthlyAmount: number;
    startDate: string;
    taxId: string;
    dc: 'in' | 'com';
    placeOfSupply?: string;
    gstTreatment?: GstTreatment;
    gstin?: string;
  }
): Promise<{ recurring_invoice_id: string }> {
  return createRecurringInvoice(client, {
    recurrence_name: `Sub-${params.shopId}-${Date.now()}`,
    customer_id: params.customerId,
    recurrence_frequency: 'months',
    repeat_every: 1,
    start_date: params.startDate,
    line_items: [{
      name: `${params.planName} Subscription`,
      quantity: 1,
      rate: params.monthlyAmount,
      tax_id: params.taxId,
      ...(params.dc === 'in' && {
        hsn_or_sac: '998314',
        product_type: 'services',
      }),
    }],
    ...(params.dc === 'in' && {
      place_of_supply: params.placeOfSupply,
      gst_treatment: params.gstTreatment,
      gst_no: params.gstin,
    }),
  });
}
```

---

## 5. India GST Engine

### 5.1 Tax Setup — Creating All GST Rates

Run once per India org during onboarding. Creates all required tax configurations.

```typescript
// src/zoho/setup/setupIndiaGst.ts
import { OrgScopedClient } from '../services/base';
import { ZohoTaxIdMap } from '../gst/taxEngine';

interface TaxSpec {
  tax_name: string;
  tax_percentage: number;
  tax_type: 'tax';
  tax_specific_type: 'igst' | 'cgst' | 'sgst' | 'nil' | 'cess';
  is_value_added: boolean;
}

const GST_SLABS = [5, 12, 18, 28];

export async function setupIndiaGstTaxes(
  client: OrgScopedClient
): Promise<ZohoTaxIdMap> {
  const taxIds: Partial<ZohoTaxIdMap> = {};

  for (const rate of GST_SLABS) {
    // Create IGST
    const igstId = await createOrFindTax(client, {
      tax_name: `IGST ${rate}%`,
      tax_percentage: rate,
      tax_type: 'tax',
      tax_specific_type: 'igst',
      is_value_added: true,
    });
    taxIds[`igst${rate}` as keyof ZohoTaxIdMap] = igstId;

    // Create CGST (half rate)
    const cgstId = await createOrFindTax(client, {
      tax_name: `CGST ${rate / 2}%`,
      tax_percentage: rate / 2,
      tax_type: 'tax',
      tax_specific_type: 'cgst',
      is_value_added: true,
    });

    // Create SGST (half rate)
    const sgstId = await createOrFindTax(client, {
      tax_name: `SGST ${rate / 2}%`,
      tax_percentage: rate / 2,
      tax_type: 'tax',
      tax_specific_type: 'sgst',
      is_value_added: true,
    });

    // Create CGST+SGST Tax Group
    const groupId = await createOrFindTaxGroup(client, {
      tax_group_name: `GST ${rate}% (CGST+SGST)`,
      taxes: `${cgstId},${sgstId}`,
    });
    taxIds[`cgstSgst${rate}` as keyof ZohoTaxIdMap] = groupId;

    await sleep(300); // Rate limit buffer
  }

  // Nil-rated
  taxIds.igst0 = await createOrFindTax(client, {
    tax_name: 'IGST 0% (Nil)',
    tax_percentage: 0,
    tax_type: 'tax',
    tax_specific_type: 'nil',
    is_value_added: true,
  });

  return taxIds as ZohoTaxIdMap;
}

async function createOrFindTax(
  client: OrgScopedClient,
  spec: TaxSpec
): Promise<string> {
  // Check if tax already exists
  const existing = await client.get<{ taxes: Array<{ tax_id: string; tax_name: string }> }>(
    '/settings/taxes'
  );
  const found = existing.taxes.find((t) => t.tax_name === spec.tax_name);
  if (found) return found.tax_id;

  const response = await client.post<{ tax: { tax_id: string } }>('/settings/taxes', spec);
  return response.tax.tax_id;
}

async function createOrFindTaxGroup(
  client: OrgScopedClient,
  spec: { tax_group_name: string; taxes: string }
): Promise<string> {
  const response = await client.post<{ tax_group: { tax_group_id: string } }>(
    '/settings/taxgroups',
    spec
  );
  return response.tax_group.tax_group_id;
}

function sleep(ms: number) { return new Promise((r) => setTimeout(r, ms)); }
```

### 5.2 Place of Supply — All Indian State Codes

```typescript
// src/zoho/gst/stateCode.ts

/** All Indian state/UT codes as used in Zoho Books place_of_supply field */
export const INDIA_STATE_CODES: Record<string, string> = {
  'Andhra Pradesh': 'AP',
  'Arunachal Pradesh': 'AR',
  'Assam': 'AS',
  'Bihar': 'BR',
  'Chhattisgarh': 'CT',
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
  'Uttarakhand': 'UT',
  'West Bengal': 'WB',
  // Union Territories
  'Andaman and Nicobar Islands': 'AN',
  'Chandigarh': 'CH',
  'Dadra and Nagar Haveli': 'DN',
  'Daman and Diu': 'DD',
  'Delhi': 'DL',
  'Jammu and Kashmir': 'JK',
  'Ladakh': 'LA',
  'Lakshadweep': 'LD',
  'Puducherry': 'PY',
};

/** Determine IGST vs CGST+SGST based on org state vs customer state */
export function isIntraStateSupply(
  orgStateCode: string,
  customerStateCode: string
): boolean {
  return orgStateCode.toUpperCase() === customerStateCode.toUpperCase();
}
```

### 5.3 GST Treatment Rules

| Scenario | `gst_treatment` | Tax Applied |
|----------|-----------------|-------------|
| Customer has GSTIN (B2B) | `business_gst` | IGST (inter) or CGST+SGST (intra) |
| Business without GSTIN | `business_none` | IGST (inter) or CGST+SGST (intra) |
| Foreign customer | `overseas` | Zero-rated (IGST 0%) |
| End consumer (B2C, no GSTIN) | `consumer` | IGST (inter) or CGST+SGST (intra) |

```typescript
export function determineGstTreatment(
  customer: ShopifyCustomer,
  order: ShopifyOrder
): { treatment: GstTreatment; gstin?: string } {
  const countryCode = order.shipping_address?.country_code || order.billing_address?.country_code;
  
  if (countryCode !== 'IN') {
    return { treatment: 'overseas' };
  }

  const gstin = customer.metafields?.find((m) => m.key === 'gstin')?.value;
  if (gstin && isValidGstin(gstin)) {
    return { treatment: 'business_gst', gstin };
  }

  // Check if B2B via order note or tag
  const isB2B = order.tags?.includes('b2b') || order.note_attributes?.some(
    (a) => a.name === 'customer_type' && a.value === 'business'
  );

  return { treatment: isB2B ? 'business_none' : 'consumer' };
}

function isValidGstin(gstin: string): boolean {
  return /^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}Z[0-9A-Z]{1}$/.test(gstin);
}
```

### 5.4 HSN vs SAC Code Assignment

```typescript
// src/zoho/gst/hsnSac.ts

/** Common HSN codes for medical/pharma (adapt for your product catalog) */
export const HSN_CODES: Record<string, string> = {
  'medicaments': '30049099',
  'surgical_gloves': '40151910',
  'masks': '63079090',
  'syringes': '90183100',
  'bandages': '30051090',
  'thermometers': '90251110',
  'default_goods': '63079090',
};

/** SAC codes for services */
export const SAC_CODES: Record<string, string> = {
  'software': '998314',         // IT implementation/consulting
  'saas': '998314',
  'shipping': '996812',         // Freight transport
  'gateway_fee': '998431',      // IT enabled services
  'subscription': '998314',
  'default_services': '998312',
};

export function getHsnOrSacCode(
  productType: 'goods' | 'services',
  productCategory?: string
): string {
  if (productType === 'services') {
    return SAC_CODES[productCategory || 'default_services'] || SAC_CODES.default_services;
  }
  return HSN_CODES[productCategory || 'default_goods'] || HSN_CODES.default_goods;
}
```

### 5.5 GSTR-1 and GSTR-3B Data Preparation

Zoho Books assembles GSTR data from transaction fields. The API integration keeps data GST-compliant; actual filing is UI-driven.

**Fields that feed GSTR-1:**
- `gst_no` (customer GSTIN) → B2B invoice register
- `place_of_supply` → determines IGST vs CGST/SGST split
- `hsn_or_sac` → HSN summary table in GSTR-1
- `gst_treatment = 'overseas'` → export invoices table
- `gst_treatment = 'consumer'` + `total > ₹2.5 lakh` → large B2C invoices table

**Fields that feed GSTR-3B:**
- All output tax amounts (from invoice tax breakdown)
- All input tax amounts (from bill tax breakdown)
- ITC claimed = input - output

**GSTR Filing Process (Human-in-the-loop):**
1. Your integration ensures all invoices/bills have correct GST fields set
2. Accountant reviews in Zoho Books → GST Filing → GSTR-3B / GSTR-1
3. Accountant clicks "Push to GSTN" (OTP auth via GST portal mobile)
4. Accountant files on gstn.gov.in
5. No direct API endpoint exists for triggering GSTN push programmatically

### 5.6 E-Invoicing (IRN Generation)

**Eligibility:** Mandatory for turnover > ₹5 crore.

**Setup (one-time, manual):**
1. Login to IRP (einvoice1.gst.gov.in) → API Registration → Create API User → Through GSP → Select "Zoho Corporation"
2. In Zoho Books → Settings → e-Invoicing → Connect Now → enter IRP credentials

**IRN Generation Flow via API:**
```typescript
// E-invoice push is UI-triggered in Zoho Books. 
// API integration should set all required fields so the invoice is IRN-ready.
// Required fields for a valid e-invoice:

const einvoiceReadyInvoice: ZohoInvoiceCreate = {
  customer_id: '...',
  date: '2026-03-28',
  place_of_supply: 'MH',
  gst_treatment: 'business_gst',
  gst_no: '27AAAAA0000A1Z5',    // MANDATORY: Customer GSTIN
  line_items: [{
    name: 'Product Name',
    hsn_or_sac: '40151910',      // MANDATORY: HSN/SAC code
    product_type: 'goods',       // MANDATORY
    quantity: 10,
    rate: 100,
    tax_id: 'igst_18_tax_id',   // MANDATORY: Tax applied
  }],
};

// After invoice creation, IRN fields appear in the response after push:
// invoice.irn        → 64-char hash (Invoice Reference Number)
// invoice.ack_no     → Acknowledgement number
// invoice.ack_date   → Acknowledgement date
// invoice.qr_code    → QR code string (print on invoice PDF)
```

**Cancel E-Invoice (within 24 hours):**
- Only via Zoho Books UI: Invoice → Cancel E-Invoice
- After 24 hours: must amend on GST portal directly

### 5.7 TDS Handling

```typescript
// src/zoho/gst/tds.ts

/**
 * TDS is applied via tds_tax_id on line items or tax_amount_withheld on payments.
 * TDS taxes must be configured in Zoho Books: Settings → Direct Taxes → TDS
 */

// On invoice line item (vendor deducts TDS at source):
const billWithTds: ZohoBillCreate = {
  vendor_id: 'vendor_id_here',
  date: '2026-03-28',
  line_items: [{
    account_id: 'expense_account_id',
    name: 'Consulting Service',
    quantity: 1,
    rate: 100000,
    hsn_or_sac: '998314',
    product_type: 'services',
    tds_tax_id: 'tds_10_percent_id', // From Settings → Direct Taxes → TDS
  }],
};

// On customer payment (customer withholds TDS at time of payment):
const paymentWithTds = {
  customer_id: 'customer_id_here',
  amount: 90000,        // Net amount received (after TDS deduction)
  date: '2026-03-28',
  payment_mode: 'banktransfer' as PaymentMode,
  tax_amount_withheld: 10000, // TDS withheld by customer
  invoices: [{ invoice_id: 'inv_id', amount_applied: 100000 }],
};
```

### 5.8 TCS Handling

```typescript
// TCS auto-applies when cumulative sales to a PAN > ₹50 lakhs/year
// Rates: 0.1% with PAN, 1% without PAN

// Apply TCS on invoice:
const invoiceWithTcs: ZohoInvoiceCreate = {
  customer_id: 'customer_id_here',
  date: '2026-03-28',
  place_of_supply: 'MH',
  gst_treatment: 'business_gst',
  tcs_tax_id: 'tcs_0_1_percent_id', // From Settings → Direct Taxes → TCS
  line_items: [/* ... */],
};
```

### 5.9 Reverse Charge Mechanism

```typescript
// Use when buying from an unregistered vendor (RCM applies)
// OR when importing services from overseas (IGST self-assessment)

const reverseChargeBill: ZohoBillCreate = {
  vendor_id: 'overseas_vendor_id',
  date: '2026-03-28',
  is_reverse_charge_applied: true,
  gst_treatment: 'overseas',
  line_items: [{
    account_id: 'expense_account_id',
    name: 'Cloud Hosting Service',
    quantity: 1,
    rate: 50000,
    hsn_or_sac: '998315',
    product_type: 'services',
    reverse_charge_tax_id: 'igst_18_tax_id', // Tax to self-assess under RCM
  }],
};
```

---

## 6. Singapore GST

### 6.1 Tax Configuration

Singapore tax codes are auto-created by Zoho Books when the org is set up on the `.com` DC. Fetch them at startup and cache the IDs:

```typescript
// src/zoho/setup/setupSingaporeGst.ts
import { OrgScopedClient } from '../services/base';

export interface SGTaxIds {
  gst9Sr: string;        // GST9 [SR]  - 9% standard sales
  gst0Zr: string;        // GST0 [ZR]  - 0% zero-rated exports
  gst9Tx: string;        // GST9 [TX]  - 9% standard purchases
  gst0Zp: string;        // GST0 [ZP]  - 0% zero-rated purchases
  gst9Bl: string;        // GST9 [BL]  - 9% disallowed
  gst9Im: string;        // GST9 [IM]  - 9% imports
  gst9TxrcTs: string;    // GST9 [TXRC-TS] - reverse charge imported services
  gst9SrcaS: string;     // GST9 [SRCA-S] - customer accounting supply
}

export async function fetchSGTaxIds(client: OrgScopedClient): Promise<SGTaxIds> {
  const response = await client.get<{ taxes: Array<{ tax_id: string; tax_name: string }> }>(
    '/settings/taxes'
  );

  const find = (name: string) => {
    const t = response.taxes.find((t) => t.tax_name === name);
    if (!t) throw new Error(`Singapore tax '${name}' not found. Is this a Singapore org?`);
    return t.tax_id;
  };

  return {
    gst9Sr: find('GST9 [SR]'),
    gst0Zr: find('GST0 [ZR]'),
    gst9Tx: find('GST9 [TX]'),
    gst0Zp: find('GST0 [ZP]'),
    gst9Bl: find('GST9 [BL]'),
    gst9Im: find('GST9 [IM]'),
    gst9TxrcTs: find('GST9 [TXRC-TS]'),
    gst9SrcaS: find('GST9 [SRCA-S]'),
  };
}
```

### 6.2 Singapore Tax Selection Logic

```typescript
export function selectSgTaxId(
  sgTaxIds: SGTaxIds,
  params: {
    isSale: boolean;           // true = invoice/sale, false = purchase/bill
    isExport: boolean;         // customer country != 'SG'
    isCustomerAccounting: boolean; // prescribed goods >SGD 10,000
    isImportedService: boolean;    // reverse charge on overseas service
    amount?: number;
  }
): string {
  if (params.isSale) {
    if (params.isExport) return sgTaxIds.gst0Zr;
    if (params.isCustomerAccounting) return sgTaxIds.gst9SrcaS;
    return sgTaxIds.gst9Sr; // Standard 9% GST
  } else {
    // Purchase
    if (params.isImportedService) return sgTaxIds.gst9TxrcTs;
    if (params.isExport) return sgTaxIds.gst0Zp;
    return sgTaxIds.gst9Tx; // Standard 9% input tax
  }
}
```

### 6.3 GST F5 Return Process

The F5 return is UI-generated in Zoho Books. No direct API trigger exists.

**F5 Filing Workflow:**
1. Zoho Books → Filing & Compliance → F5 Returns → Generate (quarterly)
2. Review Box 1–14 amounts
3. Export PDF
4. Submit to IRAS via [myTax Portal](https://mytax.iras.gov.sg)
5. Mark as Filed in Zoho Books (specify filing date)

**F5 Box Mapping:**

| F5 Box | Content | API Data Source |
|--------|---------|-----------------|
| Box 1 | Standard-rated supplies | Sum of invoices with `GST9 [SR]` tax |
| Box 2 | Zero-rated supplies | Sum of invoices with `GST0 [ZR]` tax |
| Box 5 | Taxable purchases | Sum of bills with `GST9 [TX]` tax |
| Box 6 | Output GST | Calculated from Box 1 × 9% |
| Box 7 | Input GST | Calculated from Box 5 × 9% |
| Box 8 | Net GST | Box 6 − Box 7 |

### 6.4 Singapore Invoice Example

```typescript
export function buildSGInvoice(
  order: ShopifyOrder,
  customerId: string,
  sgTaxIds: SGTaxIds
): ZohoInvoiceCreate {
  const isExport = order.shipping_address?.country_code !== 'SG';

  return {
    customer_id: customerId,
    invoice_number: `SHO-${order.order_number}`,
    date: order.created_at.split('T')[0],
    line_items: order.line_items.map((item) => ({
      name: item.name,
      quantity: item.quantity,
      rate: parseFloat(item.price),
      tax_id: isExport ? sgTaxIds.gst0Zr : sgTaxIds.gst9Sr,
    })),
    notes: `Shopify Order #${order.name}`,
    is_inclusive_tax: false,
  };
}
```

---

## 7. Multi-Currency Support

### 7.1 Currency Setup

```typescript
// src/zoho/services/currencyService.ts
import { OrgScopedClient } from './base';

export interface ZohoCurrency {
  currency_id: string;
  currency_code: string;
  currency_symbol: string;
  is_base_currency: boolean;
  exchange_rate: number;
}

export async function listCurrencies(
  client: OrgScopedClient
): Promise<ZohoCurrency[]> {
  const response = await client.get<{ currencies: ZohoCurrency[] }>('/settings/currencies');
  return response.currencies;
}

export async function addCurrency(
  client: OrgScopedClient,
  currencyCode: string,
  symbol: string,
  format: string = '1,234,567.89'
): Promise<ZohoCurrency> {
  const response = await client.post<{ currency: ZohoCurrency }>('/settings/currencies', {
    currency_code: currencyCode,
    currency_symbol: symbol,
    currency_format: format,
    price_precision: 2,
  });
  return response.currency;
}

export async function setExchangeRate(
  client: OrgScopedClient,
  currencyId: string,
  rate: number,
  effectiveDate: string // yyyy-mm-dd
): Promise<void> {
  await client.post(`/settings/currencies/${currencyId}/exchangerates`, {
    effective_date: effectiveDate,
    rate,
  });
}

/**
 * Ensures required currencies exist for a Shopify store.
 * Shopify stores can sell in multiple currencies.
 */
export async function ensureCurrencies(
  client: OrgScopedClient,
  requiredCurrencyCodes: string[]
): Promise<Map<string, string>> { // code → currency_id
  const existing = await listCurrencies(client);
  const currencyMap = new Map(existing.map((c) => [c.currency_code, c.currency_id]));

  const CURRENCY_SYMBOLS: Record<string, { symbol: string }> = {
    USD: { symbol: '$' }, EUR: { symbol: '€' }, GBP: { symbol: '£' },
    AED: { symbol: 'AED' }, SGD: { symbol: 'S$' }, INR: { symbol: '₹' },
    MYR: { symbol: 'RM' }, AUD: { symbol: 'A$' }, CAD: { symbol: 'C$' },
  };

  for (const code of requiredCurrencyCodes) {
    if (currencyMap.has(code)) continue;
    const meta = CURRENCY_SYMBOLS[code] || { symbol: code };
    const created = await addCurrency(client, code, meta.symbol);
    currencyMap.set(code, created.currency_id);
    await sleep(300);
  }

  return currencyMap;
}

function sleep(ms: number) { return new Promise((r) => setTimeout(r, ms)); }
```

### 7.2 Multi-Currency Invoices

```typescript
// When creating an invoice in a foreign currency:
const usdInvoice: ZohoInvoiceCreate = {
  customer_id: 'customer_id_here',
  date: '2026-03-28',
  currency_id: 'usd_currency_id',   // From ensureCurrencies()
  exchange_rate: 83.15,              // 1 USD = 83.15 INR (India org base)
  line_items: [{
    name: 'Export Product',
    quantity: 10,
    rate: 500.00,  // In USD
    tax_id: 'igst_0_tax_id', // Zero-rated for exports
    ...(dc === 'in' && {
      hsn_or_sac: '40151910',
      product_type: 'goods',
    }),
  }],
  ...(dc === 'in' && {
    gst_treatment: 'overseas',
    place_of_supply: 'MH',
  }),
};
```

### 7.3 Base Currency Adjustment (Month-End)

```typescript
export async function createMonthEndFxAdjustment(
  client: OrgScopedClient,
  params: {
    currencyId: string;
    newRate: number;
    adjustmentDate: string;
    accountIds: string[];
  }
): Promise<void> {
  await client.post('/basecurrencyadjustment', {
    currency_id: params.currencyId,
    adjustment_date: params.adjustmentDate,
    exchange_rate: params.newRate,
    notes: `Month-end FX adjustment`,
    account_ids: params.accountIds,
  });
}
```

---

## 8. Webhooks

### 8.1 Webhook Setup

```typescript
// src/zoho/services/webhookService.ts
import { OrgScopedClient } from './base';

export interface ZohoWebhookCreate {
  webhook_name: string;
  entity: string;
  url: string;
  method: 'POST';
  events: string[];
  secret?: string;
  headers?: Array<{ param_name: string; param_value: string }>;
  is_new_response_format?: boolean;
}

export async function createWebhook(
  client: OrgScopedClient,
  data: ZohoWebhookCreate
): Promise<{ webhook_id: string }> {
  const response = await client.post<{ webhook: { webhook_id: string } }>(
    '/settings/webhooks',
    data
  );
  return response.webhook;
}

/**
 * Registers all Shopify-relevant webhooks for a newly connected org.
 * Call once during onboarding.
 */
export async function registerShopifyWebhooks(
  client: OrgScopedClient,
  callbackBaseUrl: string,
  hmacSecret: string
): Promise<void> {
  const webhooks: ZohoWebhookCreate[] = [
    {
      webhook_name: 'Invoice Status Changed',
      entity: 'invoice',
      url: `${callbackBaseUrl}/webhooks/zoho/invoice`,
      method: 'POST',
      events: ['invoice_created', 'invoice_updated', 'invoice_status_changed'],
      secret: hmacSecret,
      is_new_response_format: true,
    },
    {
      webhook_name: 'Payment Created',
      entity: 'payment',
      url: `${callbackBaseUrl}/webhooks/zoho/payment`,
      method: 'POST',
      events: ['payment_created', 'payment_deleted'],
      secret: hmacSecret,
      is_new_response_format: true,
    },
    {
      webhook_name: 'Credit Note Created',
      entity: 'creditnote',
      url: `${callbackBaseUrl}/webhooks/zoho/creditnote`,
      method: 'POST',
      events: ['creditnote_created', 'creditnote_voided'],
      secret: hmacSecret,
      is_new_response_format: true,
    },
    {
      webhook_name: 'Contact Updated',
      entity: 'contact',
      url: `${callbackBaseUrl}/webhooks/zoho/contact`,
      method: 'POST',
      events: ['contact_created', 'contact_updated'],
      secret: hmacSecret,
      is_new_response_format: true,
    },
  ];

  for (const webhook of webhooks) {
    await createWebhook(client, webhook);
    await sleep(200);
  }
}

function sleep(ms: number) { return new Promise((r) => setTimeout(r, ms)); }
```

### 8.2 Webhook HMAC Verification

```typescript
// src/zoho/webhooks/verify.ts
import crypto from 'crypto';
import { Request, Response, NextFunction } from 'express';

/**
 * Middleware: Verifies Zoho Books HMAC-SHA256 webhook signature.
 * Zoho sends the signature in the 'X-Zoho-Webhook-Signature' header (or similar).
 * The secret is the one you provided in the webhook creation.
 */
export function zohoWebhookVerification(secret: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const signature = req.headers['x-zoho-webhook-signature'] as string;
    if (!signature) {
      return res.status(401).json({ error: 'Missing webhook signature' });
    }

    const rawBody = (req as any).rawBody as Buffer;
    if (!rawBody) {
      return res.status(400).json({ error: 'Raw body not available' });
    }

    const computed = crypto
      .createHmac('sha256', secret)
      .update(rawBody)
      .digest('hex');

    if (!crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(signature))) {
      return res.status(401).json({ error: 'Invalid webhook signature' });
    }

    next();
  };
}

// To capture raw body, add this middleware before body-parser:
// app.use('/webhooks/zoho', (req, res, next) => {
//   let data = '';
//   req.setEncoding('utf8');
//   req.on('data', (chunk) => { data += chunk; });
//   req.on('end', () => {
//     (req as any).rawBody = Buffer.from(data);
//     next();
//   });
// });
```

### 8.3 Webhook Handler

```typescript
// src/zoho/webhooks/handler.ts
import { Router } from 'express';
import { zohoWebhookVerification } from './verify';

const router = Router();

router.post(
  '/webhooks/zoho/invoice',
  zohoWebhookVerification(process.env.ZOHO_WEBHOOK_SECRET!),
  async (req, res) => {
    const event = req.body;
    const { entity, event_type, data } = event;

    try {
      switch (event_type) {
        case 'invoice_status_changed':
          if (data.invoice?.status === 'paid') {
            // Sync payment status back to Shopify if needed
            await syncInvoicePaidToShopify(data.invoice);
          }
          break;
        case 'payment_created':
          // Log or sync payment confirmation
          await handlePaymentCreated(data.payment);
          break;
        case 'creditnote_created':
          // Sync refund status
          await handleCreditNoteCreated(data.creditnote);
          break;
      }
      res.status(200).json({ received: true });
    } catch (err) {
      console.error('Webhook handler error:', err);
      res.status(500).json({ error: 'Processing failed' });
    }
  }
);

export default router;
```

---

## 9. Reconciliation Workflow

### Step-by-Step: Reconciling Shopify Payouts with Zoho Books

#### Step 1: Create the Shopify Payments Bank Account (one-time)

```typescript
// During onboarding
const shopifyPaymentsAccount = await createShopifyPaymentsAccount(
  orgClient,
  dc === 'in' ? 'INR' : 'SGD'
);
// Save shopifyPaymentsAccount.account_id to tenant config
```

#### Step 2: Fetch Shopify Payout Transactions

```typescript
// From Shopify Admin API: GET /payouts/{payout_id}/transactions
// Each transaction includes: type (payout/refund/adjustment/fee), amount, source_id

interface ShopifyPayoutTransaction {
  id: number;
  type: 'payout' | 'refund' | 'adjustment' | 'charge' | 'chargeback' | 'chargeback_reversal';
  amount: string;           // Positive = credit to merchant
  fee: string;              // Transaction fee deducted
  net: string;              // Net after fee
  source_type: string;      // 'charge' | 'refund' | 'dispute'
  source_id: number;        // Shopify order/refund ID
  processed_at: string;
}
```

#### Step 3: Import Payout as Bank Statement

```typescript
export async function importPayoutToZoho(
  client: OrgScopedClient,
  bankAccountId: string,
  payout: ShopifyPayout,
  transactions: ShopifyPayoutTransaction[]
): Promise<void> {
  const statementTransactions = transactions.map((t) => {
    const amount = Math.abs(parseFloat(t.net));
    const isCredit = parseFloat(t.net) >= 0;

    return {
      date: t.processed_at.split('T')[0],
      debit_or_credit: isCredit ? 'credit' : 'debit',
      amount,
      payee: t.source_type === 'charge' ? 'Shopify Order' : 'Shopify Refund',
      description: `${t.type} | Source: ${t.source_type} #${t.source_id}`,
      reference_number: `SHOPIFY-${t.id}`,
    };
  });

  await client.post(`/bankaccounts/${bankAccountId}/statements`, {
    date_format: 'yyyy-MM-dd',
    transactions: statementTransactions,
  });
}
```

#### Step 4: Auto-Match Transactions to Invoices

```typescript
export async function autoReconcileShopifyPayout(
  client: OrgScopedClient,
  bankAccountId: string,
  shopifyOrderToInvoiceMap: Map<string, string> // Shopify order ID → Zoho invoice ID
): Promise<{ matched: number; unmatched: number }> {
  const uncategorized = await listUncategorizedTransactions(client, bankAccountId);
  let matched = 0;
  let unmatched = 0;

  for (const txn of uncategorized) {
    // Extract Shopify source ID from description
    const match = txn.description.match(/Source: charge #(\d+)/);
    if (!match) { unmatched++; continue; }

    const shopifyOrderId = match[1];
    const invoiceId = shopifyOrderToInvoiceMap.get(shopifyOrderId);
    if (!invoiceId) { unmatched++; continue; }

    try {
      // Get match suggestions from Zoho
      const suggestions = await getMatchSuggestions(client, txn.transaction_id);
      const invoiceMatch = suggestions.find(
        (s) => s.transaction_id === invoiceId || s.invoice_id === invoiceId
      );

      if (invoiceMatch) {
        await matchTransactionToPayment(
          client,
          txn.transaction_id,
          invoiceMatch.transaction_id,
          'customerpayment'
        );
        matched++;
      } else {
        unmatched++;
      }
    } catch (err) {
      console.error(`Failed to match transaction ${txn.transaction_id}:`, err);
      unmatched++;
    }

    await sleep(200); // Rate limit
  }

  return { matched, unmatched };
}
```

#### Step 5: Handle Fees as Expenses

```typescript
export async function reconcileShopifyFees(
  client: OrgScopedClient,
  bankAccountId: string,
  feeTransactions: ShopifyPayoutTransaction[],
  accountIds: ShopifyAccountIds,
  shopifyVendorId: string
): Promise<void> {
  const feeDebits = feeTransactions.filter((t) => parseFloat(t.fee) > 0);

  for (const txn of feeDebits) {
    const uncategorized = await listUncategorizedTransactions(client, bankAccountId);
    const feeEntry = uncategorized.find(
      (u) => u.reference_number === `SHOPIFY-${txn.id}` && u.debit_or_credit === 'debit'
    );
    if (!feeEntry) continue;

    await categorizeAsExpense(
      client,
      feeEntry.transaction_id,
      accountIds.shopifyGatewayFees,
      shopifyVendorId,
      `Shopify payment gateway fee - ${txn.source_type}`
    );
    await sleep(200);
  }
}
```

#### Step 6: Handle Chargebacks

```typescript
export async function handleChargeback(
  client: OrgScopedClient,
  params: {
    shopifyDisputeId: string;
    invoiceId: string;
    customerId: string;
    amount: number;
    date: string;
    bankAccountId: string;
  }
): Promise<void> {
  // 1. Create a credit note for the chargedback amount
  const creditNote = await createCreditNote(client, {
    customer_id: params.customerId,
    creditnote_number: `CB-${params.shopifyDisputeId}`,
    date: params.date,
    line_items: [{
      name: 'Chargeback',
      quantity: 1,
      rate: params.amount,
      account_id: 'chargeback_expense_account_id', // Pre-setup account
    }],
  });

  // 2. Apply credit note to original invoice
  await applyCreditNoteToInvoice(
    client,
    creditNote.creditnote_id,
    params.invoiceId,
    params.amount
  );
}
```

---

## 10. Reporting

### 10.1 Building P&L Data via API

Zoho Books does not expose a `/reports/pnl` REST endpoint. Build P&L data from transaction APIs:

```typescript
// src/zoho/reporting/financialReports.ts
import { OrgScopedClient } from '../services/base';

export interface PLSummary {
  period: string;
  totalRevenue: number;
  totalRefunds: number;
  netRevenue: number;
  gatewayFees: number;
  subscriptionFees: number;
  shippingCosts: number;
  totalExpenses: number;
  grossProfit: number;
}

export async function buildPLSummary(
  client: OrgScopedClient,
  dateStart: string,
  dateEnd: string,
  accountIds: ShopifyAccountIds
): Promise<PLSummary> {
  // Fetch all paid invoices in period
  const allInvoices: ZohoInvoice[] = [];
  let page = 1;
  while (true) {
    const { invoices, has_more_page } = await listInvoices(client, {
      date_start: dateStart,
      date_end: dateEnd,
      status: 'paid',
      page,
      per_page: 200,
    });
    allInvoices.push(...invoices);
    if (!has_more_page) break;
    page++;
  }

  const totalRevenue = allInvoices.reduce((sum, inv) => sum + inv.total, 0);

  // Fetch credit notes (refunds)
  const creditNotesResp = await client.get<{ creditnotes: Array<{ total: number }> }>(
    '/creditnotes',
    { date_start: dateStart, date_end: dateEnd, per_page: 200 }
  );
  const totalRefunds = creditNotesResp.creditnotes.reduce((sum, cn) => sum + cn.total, 0);

  // Fetch bills (expenses)
  const billsResp = await client.get<{ bills: Array<{ total: number; account_id: string }> }>(
    '/bills',
    { date_start: dateStart, date_end: dateEnd, per_page: 200 }
  );

  const gatewayFees = billsResp.bills
    .filter((b) => b.account_id === accountIds.shopifyGatewayFees)
    .reduce((sum, b) => sum + b.total, 0);

  const subscriptionFees = billsResp.bills
    .filter((b) => b.account_id === accountIds.shopifySubscriptionFees)
    .reduce((sum, b) => sum + b.total, 0);

  const shippingCosts = billsResp.bills
    .filter((b) => b.account_id === accountIds.shopifyShippingFees)
    .reduce((sum, b) => sum + b.total, 0);

  const totalExpenses = gatewayFees + subscriptionFees + shippingCosts;
  const netRevenue = totalRevenue - totalRefunds;
  const grossProfit = netRevenue - totalExpenses;

  return {
    period: `${dateStart} to ${dateEnd}`,
    totalRevenue,
    totalRefunds,
    netRevenue,
    gatewayFees,
    subscriptionFees,
    shippingCosts,
    totalExpenses,
    grossProfit,
  };
}
```

### 10.2 Ledger Data for an Account

```typescript
export async function getAccountLedger(
  client: OrgScopedClient,
  accountId: string,
  dateStart: string,
  dateEnd: string
): Promise<any[]> {
  const response = await client.get<{ transactions: any[] }>(
    '/chartofaccounts/transactions',
    { account_id: accountId, date_start: dateStart, date_end: dateEnd }
  );
  return response.transactions;
}
```

### 10.3 GST Report Data Aggregation (India)

```typescript
export interface GstSummary {
  period: string;
  outputIgst: number;
  outputCgst: number;
  outputSgst: number;
  inputIgst: number;
  inputCgst: number;
  inputSgst: number;
  netGstPayable: number;
}

export async function buildGstSummary(
  client: OrgScopedClient,
  accountIds: ShopifyAccountIds,
  dateStart: string,
  dateEnd: string
): Promise<GstSummary> {
  async function getLedgerTotal(accountId: string): Promise<number> {
    const txns = await getAccountLedger(client, accountId, dateStart, dateEnd);
    return txns.reduce((sum, t) => sum + (t.credit_amount || 0) - (t.debit_amount || 0), 0);
  }

  const [outputIgst, outputCgst, outputSgst, inputIgst, inputCgst, inputSgst] = await Promise.all([
    getLedgerTotal(accountIds.gstOutputIgst!),
    getLedgerTotal(accountIds.gstOutputCgst!),
    getLedgerTotal(accountIds.gstOutputSgst!),
    getLedgerTotal(accountIds.gstInputIgst!),
    getLedgerTotal(accountIds.gstInputCgst!),
    getLedgerTotal(accountIds.gstInputSgst!),
  ]);

  const totalOutput = outputIgst + outputCgst + outputSgst;
  const totalInput = inputIgst + inputCgst + inputSgst;

  return {
    period: `${dateStart} to ${dateEnd}`,
    outputIgst, outputCgst, outputSgst,
    inputIgst, inputCgst, inputSgst,
    netGstPayable: totalOutput - totalInput,
  };
}
```

---

## 11. Rate Limiting & Error Handling

### 11.1 Rate Limits Summary

| Limit | Value | Error Code |
|-------|-------|------------|
| Per org per minute | 100 requests | 44 |
| Per account per minute | 100 requests | 44 |
| Concurrent requests (paid) | 10 in-flight | 1070 |
| Daily Free plan | 1,000 | 45 |
| Daily Professional | 5,000 | 45 |
| Daily Premium/Elite/Ultimate | 10,000 | 45 |

Target: **~1 request per 650ms** to stay safely under 100/min.

### 11.2 Retry Logic with Exponential Backoff

```typescript
// src/zoho/utils/retry.ts

export interface RetryOptions {
  maxAttempts?: number;
  initialDelayMs?: number;
  maxDelayMs?: number;
  backoffFactor?: number;
}

export async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {}
): Promise<T> {
  const {
    maxAttempts = 5,
    initialDelayMs = 1000,
    maxDelayMs = 60000,
    backoffFactor = 2,
  } = options;

  let lastError: Error;
  let delay = initialDelayMs;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error: any) {
      lastError = error;

      // Check for retryable conditions
      const isRateLimited =
        error.response?.status === 429 ||
        error.response?.data?.code === 44 ||
        error.response?.data?.code === 45 ||
        error.response?.data?.code === 1070;

      const isServerError = error.response?.status >= 500;

      if (!isRateLimited && !isServerError) {
        throw error; // Don't retry client errors (400, 401, 404)
      }

      if (attempt === maxAttempts) break;

      // Use Retry-After header if present
      const retryAfter = error.response?.headers?.['retry-after'];
      const waitMs = retryAfter
        ? parseInt(retryAfter, 10) * 1000
        : Math.min(delay, maxDelayMs);

      console.warn(
        `Zoho API attempt ${attempt} failed (${error.response?.data?.code || error.response?.status}). ` +
        `Retrying in ${waitMs}ms...`
      );

      await new Promise((r) => setTimeout(r, waitMs));
      delay = Math.min(delay * backoffFactor, maxDelayMs);
    }
  }

  throw lastError!;
}
```

### 11.3 Request Queue (Token Bucket)

```typescript
// src/zoho/utils/rateLimiter.ts

export class RateLimiter {
  private queue: Array<() => void> = [];
  private running = 0;
  private intervalMs: number;
  private maxConcurrent: number;
  private timer?: NodeJS.Timeout;

  constructor(requestsPerMinute = 80, maxConcurrent = 8) {
    this.intervalMs = (60 * 1000) / requestsPerMinute;
    this.maxConcurrent = maxConcurrent;
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    await this.waitForSlot();
    this.running++;
    try {
      return await fn();
    } finally {
      this.running--;
    }
  }

  private waitForSlot(): Promise<void> {
    return new Promise((resolve) => {
      const tryRun = () => {
        if (this.running < this.maxConcurrent) {
          // Enforce minimum interval between requests
          setTimeout(resolve, this.intervalMs);
        } else {
          this.queue.push(tryRun);
        }
      };
      tryRun();
    });
  }
}

// Singleton per tenant (one limiter per org)
const limiters = new Map<string, RateLimiter>();

export function getRateLimiter(orgId: string): RateLimiter {
  if (!limiters.has(orgId)) {
    limiters.set(orgId, new RateLimiter(80, 8));
  }
  return limiters.get(orgId)!;
}
```

### 11.4 Enhanced ZohoClient with Retry + Rate Limiting

```typescript
// src/zoho/client.ts (enhanced)
import { withRetry } from './utils/retry';
import { getRateLimiter } from './utils/rateLimiter';

export class ZohoClient {
  // ... (constructor same as before)

  async get<T>(path: string, params?: Record<string, any>): Promise<T> {
    return withRetry(() =>
      getRateLimiter(this.orgId).execute(() =>
        this.http.get(path, { params }).then((r) => r.data)
      )
    );
  }

  async post<T>(path: string, data: any, params?: Record<string, any>): Promise<T> {
    return withRetry(() =>
      getRateLimiter(this.orgId).execute(() =>
        this.http.post(path, data, { params }).then((r) => r.data)
      )
    );
  }
  // ... same pattern for put, delete
}
```

### 11.5 Common Error Codes and Resolution

| Code | HTTP | Meaning | Resolution |
|------|------|---------|------------|
| `0` | 200/201 | Success | Proceed |
| `44` | 429 | Per-minute rate limit | Exponential backoff, wait 60s |
| `45` | 200 | Daily API limit | Queue remaining for next day; alert user to upgrade plan |
| `1000` | 500 | Internal server error | Retry 3x with backoff; escalate to Zoho Support if persistent |
| `1002` | 200 | Resource not found | Verify ID is correct; check org context |
| `1070` | 429 | Concurrent limit | Reduce concurrent requests; wait and retry |
| — | `401` | Token invalid/expired | Refresh access token immediately |
| — | `404` | Endpoint not found | Verify API path and org_id parameter |

```typescript
// src/zoho/utils/errorHandler.ts

export class ZohoApiError extends Error {
  constructor(
    public code: number,
    public message: string,
    public httpStatus?: number
  ) {
    super(`[${code}] ${message}`);
    this.name = 'ZohoApiError';
  }
}

export function handleZohoResponse<T>(response: any): T {
  if (response.code !== 0) {
    throw new ZohoApiError(response.code, response.message);
  }
  return response as T;
}
```

---

## 12. Data Model Reference

### 12.1 TypeScript Interfaces — Complete Set

```typescript
// src/zoho/types/index.ts

export type ZohoDC = 'in' | 'com';
export type GstTreatment = 'business_gst' | 'business_none' | 'overseas' | 'consumer';
export type ContactType = 'customer' | 'vendor';
export type PaymentMode = 'check' | 'cash' | 'creditcard' | 'banktransfer' | 'bankremittance' | 'autotransaction' | 'others';
export type ProductType = 'goods' | 'services';
export type InvoiceStatus = 'draft' | 'sent' | 'viewed' | 'partially_paid' | 'paid' | 'overdue' | 'void';
export type RecurrenceFrequency = 'days' | 'weeks' | 'months' | 'years';

// --- Address ---
export interface ZohoAddress {
  address?: string;
  city?: string;
  state?: string;
  zip?: string;
  country?: string;
}

// --- Contact ---
export interface ZohoContactCreate {
  contact_name: string;
  contact_type: ContactType;
  company_name?: string;
  email?: string;
  phone?: string;
  currency_id?: string;
  payment_terms?: number;
  billing_address?: ZohoAddress;
  shipping_address?: ZohoAddress;
  // India
  gst_no?: string;
  gst_treatment?: GstTreatment;
  place_of_contact?: string;
  pan_no?: string;
}
export interface ZohoContact extends ZohoContactCreate {
  contact_id: string;
  status: 'active' | 'inactive';
}

// --- Line Item ---
export interface ZohoLineItem {
  item_id?: string;
  name: string;
  description?: string;
  quantity: number;
  rate: number;
  unit?: string;
  tax_id?: string;
  account_id?: string;
  discount?: string;
  item_total?: number;
  // India
  hsn_or_sac?: string;
  product_type?: ProductType;
  reverse_charge_tax_id?: string;
  tds_tax_id?: string;
}

// --- Invoice ---
export interface ZohoInvoiceCreate {
  customer_id: string;
  invoice_number?: string;
  date: string;
  due_date?: string;
  currency_id?: string;
  exchange_rate?: number;
  line_items: ZohoLineItem[];
  discount?: string;
  is_discount_before_tax?: boolean;
  is_inclusive_tax?: boolean;
  payment_terms?: number;
  notes?: string;
  terms?: string;
  template_id?: string;
  // India
  place_of_supply?: string;
  gst_no?: string;
  gst_treatment?: GstTreatment;
  is_reverse_charge_applied?: boolean;
  tcs_tax_id?: string;
}
export interface ZohoInvoice extends ZohoInvoiceCreate {
  invoice_id: string;
  status: InvoiceStatus;
  total: number;
  balance: number;
  created_time: string;
  last_modified_time: string;
  // E-invoice (India)
  irn?: string;
  ack_no?: string;
  ack_date?: string;
  qr_code?: string;
  e_invoice_status?: string;
}

// --- Bill ---
export interface ZohoBillCreate {
  vendor_id: string;
  bill_number?: string;
  date: string;
  due_date?: string;
  account_id?: string;
  line_items: ZohoBillLineItem[];
  currency_id?: string;
  exchange_rate?: number;
  // India
  source_of_supply?: string;
  destination_of_supply?: string;
  gst_no?: string;
  gst_treatment?: GstTreatment;
  is_reverse_charge_applied?: boolean;
}
export interface ZohoBillLineItem {
  account_id: string;
  name: string;
  description?: string;
  quantity: number;
  rate: number;
  tax_id?: string;
  hsn_or_sac?: string;
  product_type?: ProductType;
  tds_tax_id?: string;
  reverse_charge_tax_id?: string;
}
export interface ZohoBill extends ZohoBillCreate {
  bill_id: string;
  status: string;
  total: number;
  balance: number;
}

// --- Customer Payment ---
export interface ZohoCustomerPaymentCreate {
  customer_id: string;
  amount: number;
  date: string;
  payment_mode: PaymentMode;
  paid_through_account_id?: string;
  reference_number?: string;
  invoices?: Array<{ invoice_id: string; amount_applied: number }>;
  tax_amount_withheld?: number;
}
export interface ZohoCustomerPayment extends ZohoCustomerPaymentCreate {
  payment_id: string;
}

// --- Credit Note ---
export interface ZohoCreditNoteCreate {
  customer_id: string;
  creditnote_number?: string;
  date: string;
  line_items: ZohoLineItem[];
  place_of_supply?: string;
  gst_no?: string;
  gst_treatment?: GstTreatment;
  is_reverse_charge_applied?: boolean;
}
export interface ZohoCreditNote extends ZohoCreditNoteCreate {
  creditnote_id: string;
  status: string;
  total: number;
}

// --- Journal ---
export interface ZohoJournalCreate {
  journal_date: string;
  journal_number?: string;
  reference_number?: string;
  notes?: string;
  currency_id?: string;
  exchange_rate?: number;
  line_items: ZohoJournalLineItem[];
  status?: 'draft' | 'published';
}
export interface ZohoJournalLineItem {
  account_id: string;
  debit_or_credit: 'debit' | 'credit';
  amount: number;
  description?: string;
  tax_id?: string;
}

// --- Recurring Invoice ---
export interface ZohoRecurringInvoiceCreate {
  recurrence_name: string;
  customer_id: string;
  recurrence_frequency: RecurrenceFrequency;
  repeat_every: number;
  start_date?: string;
  end_date?: string;
  line_items: ZohoLineItem[];
  place_of_supply?: string;
  gst_treatment?: GstTreatment;
  gst_no?: string;
}

// --- Tax ---
export interface ZohoTax {
  tax_id: string;
  tax_name: string;
  tax_percentage: number;
  tax_type: string;
  tax_specific_type?: string;
}

// --- Organization ---
export interface ZohoOrganization {
  organization_id: string;
  name: string;
  is_default_org: boolean;
  plan_name: string;
  currency_code: string;
  currency_id: string;
  time_zone: string;
  fiscal_year_start_month: number;
  is_org_active: boolean;
}

// --- Tenant Config (stored per connected shop) ---
export interface TenantZohoConfig {
  tokens: ZohoTokens;
  organizationId: string;
  organizationName: string;
  dc: ZohoDC;
  orgState?: string;       // India org state code
  taxIds?: ZohoTaxIdMap;   // Cached tax IDs
  accountIds?: ShopifyAccountIds; // Cached account IDs
}
```

### 12.2 Field Mapping: Shopify Order → Zoho Books Invoice

| Shopify Field | Zoho Books Field | Notes |
|---------------|-----------------|-------|
| `order.order_number` | `invoice_number` | Prefix with `SHO-` |
| `order.created_at` | `date` | Strip to `yyyy-mm-dd` |
| `customer.id` | `customer_id` | Must be Zoho contact ID |
| `line_item.name` | `line_items[].name` | Direct map |
| `line_item.price` | `line_items[].rate` | Per unit, not total |
| `line_item.quantity` | `line_items[].quantity` | Direct map |
| `line_item.sku` | `line_items[].description` | Add as description |
| `line_item.tax_lines[].rate` | Summed → `tax_id` lookup | Sum all rates, find matching tax ID |
| `line_item.properties[hsn_code]` | `line_items[].hsn_or_sac` | Custom metafield |
| `shipping_address.province_code` | `place_of_supply` | Map via `SHOPIFY_PROVINCE_TO_ZOHO_STATE` |
| `customer.metafields[gstin]` | `gst_no` | Validate 15-char GSTIN |
| `total_shipping_price_set.amount` | Extra line item `Shipping` | Add if > 0 |
| `total_discounts` | `discount` | Pass as flat amount |
| `currency` | `currency_id` | Map via `ensureCurrencies()` |
| `presentment_money.amount` | `exchange_rate` | Calculate from order currency |
| `order.name` | `notes` | Free-text reference |

---

## 13. Implementation Checklist

### India Organization — Initial Setup

- [ ] Register app at `https://api-console.zoho.in`
- [ ] Store `ZOHO_IN_CLIENT_ID`, `ZOHO_IN_CLIENT_SECRET`, `ZOHO_IN_REDIRECT_URI`
- [ ] Implement OAuth flow; save `access_token`, `refresh_token`, DC = `in`, `expires_at`
- [ ] Call `GET /organizations` → store `organization_id`
- [ ] Run `setupIndiaGstTaxes()` → cache all GST `tax_id`s (IGST 5/12/18/28 + CGST+SGST groups)
- [ ] Run `setupShopifyAccounts()` → cache all account IDs
- [ ] Create `Shopify Payments` bank account → store `account_id`
- [ ] Create `Shopify Inc.` vendor contact → store `contact_id` (for fee bills)
- [ ] Register webhooks via `registerShopifyWebhooks()`
- [ ] Configure e-invoicing in Zoho Books UI (Settings → e-Invoicing) if turnover > ₹5 crore
- [ ] Enable GSTN Online Filing in Zoho Books UI (Settings → Taxes → Online Filing)

### Singapore Organization — Initial Setup

- [ ] Register app at `https://api-console.zoho.com`
- [ ] Store `ZOHO_COM_CLIENT_ID`, `ZOHO_COM_CLIENT_SECRET`, `ZOHO_COM_REDIRECT_URI`
- [ ] Implement OAuth flow; save tokens with DC = `com`
- [ ] Call `GET /organizations` → store `organization_id`
- [ ] Run `fetchSGTaxIds()` → cache all Singapore GST tax IDs
- [ ] Run `setupShopifyAccounts()` for SGD accounts
- [ ] Create `Shopify Payments (SGD)` bank account
- [ ] Register webhooks

### Per Shopify Order (Real-time)

- [ ] On `orders/paid` webhook:
  1. `upsertContactFromShopify()` → get or create Zoho contact
  2. `mapShopifyOrderToInvoice()` → build invoice payload
  3. `createInvoice()` → store `invoice_id` mapped to Shopify `order_id`
  4. `markInvoiceSent()` → move from draft to sent
  5. `recordShopifyPayment()` → record payment against invoice

### Per Shopify Refund

- [ ] On `refunds/create` webhook:
  1. Retrieve original Zoho invoice by `order_id`
  2. `handleShopifyRefund()` → creates credit note + applies to invoice

### Payout Reconciliation (Per Payout Cycle)

- [ ] Fetch payout transactions from Shopify Payouts API
- [ ] `importShopifyPayoutStatement()` → import into Zoho bank account
- [ ] `autoReconcileShopifyPayout()` → match credits to payments
- [ ] `reconcileShopifyFees()` → categorize fee debits as expenses
- [ ] Handle any chargebacks via `handleChargeback()`

### Monthly/Quarterly Accounting

- [ ] `buildPLSummary()` → generate revenue/expense report for merchant
- [ ] (India) `buildGstSummary()` → prepare GST liability data
- [ ] (India) Merchant reviews + files GSTR-1/3B via Zoho Books UI
- [ ] (Singapore) Merchant reviews + exports F5 Return PDF via Zoho Books UI → submits to IRAS
- [ ] `createMonthEndFxAdjustment()` → update FX rates for multi-currency stores

### Recurring Subscriptions (if applicable)

- [ ] `setupShopifySubscriptionInvoice()` → create recurring invoice profile
- [ ] `stopRecurringInvoice()` when subscription cancelled in Shopify

### Error Monitoring

- [ ] Wrap all Zoho API calls in `withRetry()`
- [ ] Use `getRateLimiter()` per org to stay under 100 req/min
- [ ] Log all `ZohoApiError` instances with `code`, `message`, `orgId`, `shopId`
- [ ] Alert on code `45` (daily limit) — prompt user to upgrade Zoho plan
- [ ] Alert on persistent `1000` errors — may indicate Zoho service degradation

---

*All API endpoints and field names sourced from [Zoho Books API v3 Documentation](https://www.zoho.com/books/api/v3/) and [Zoho Books Help Center](https://www.zoho.com/in/books/help/). Last validated: March 28, 2026.*
