# Shopify API Integration Skill
## For: Building a Shopify Accounting App (India + Singapore)
## Version: 1.0 | March 2026

---

> **Purpose:** This is a self-contained technical instruction file for building the Shopify integration layer of a multi-tenant accounting SaaS platform. Every section contains production-ready TypeScript/Node.js code. Follow sections in order during initial build; reference individual sections during ongoing development.

> **Primary Sources:** [Shopify Dev Docs](https://shopify.dev/docs) | [Shopify GraphQL API Reference](https://shopify.dev/docs/api/admin-graphql) | [Shopify Access Scopes](https://shopify.dev/docs/api/usage/access-scopes) | [Shopify Billing API](https://shopify.dev/docs/apps/billing) | [Shopify App Store Requirements](https://shopify.dev/docs/apps/launch/shopify-app-store/app-store-requirements)

---

## Table of Contents

1. [App Architecture](#1-app-architecture)
2. [OAuth 2.0 Authentication](#2-oauth-20-authentication)
3. [Order Data Extraction](#3-order-data-extraction)
4. [Transaction & Payment Data](#4-transaction--payment-data)
5. [Shopify Payments / Payout Reconciliation](#5-shopify-payments--payout-reconciliation)
6. [Refunds & Returns](#6-refunds--returns)
7. [Tax Data Extraction (India Focus)](#7-tax-data-extraction-india-focus)
8. [Product & Inventory Data](#8-product--inventory-data)
9. [Webhook Implementation](#9-webhook-implementation)
10. [GraphQL Bulk Operations](#10-graphql-bulk-operations)
11. [Billing API (Monetization)](#11-billing-api-monetization)
12. [Rate Limit Management](#12-rate-limit-management)
13. [App Bridge & Embedded UI](#13-app-bridge--embedded-ui)
14. [GDPR Compliance](#14-gdpr-compliance)
15. [Data Model Reference](#15-data-model-reference)
16. [Implementation Checklist](#16-implementation-checklist)

---

## 1. App Architecture

### App Type: Public App

Build a **Public App** for distribution via the Shopify App Store. This allows any Shopify merchant to discover and install your accounting app. Custom Apps (single-store) are not suitable for a SaaS platform.

Key App Store requirements ([source](https://shopify.dev/docs/apps/launch/shopify-app-store/app-store-requirements)):
- Must use Shopify APIs exclusively
- Must use session tokens for authentication (requirement 1.1.1)
- Must use Shopify Billing API for all charges (no Stripe/PayPal off-platform)
- Must implement App Bridge (latest version, all apps since March 13, 2024)
- Must implement all 3 mandatory GDPR compliance webhooks
- Must verify HMAC signatures on all webhooks
- Screenshots must be 1600×900px (16:9), 3–6 required

### Recommended Tech Stack

```
Framework:     Remix (preferred) or Next.js 14+
Runtime:       Node.js 20+
Language:      TypeScript (strict mode)
Database:      PostgreSQL (primary) + Redis (queues/cache)
ORM:           Prisma
Queue:         BullMQ (Redis-backed)
Shopify SDK:   @shopify/shopify-app-remix
Polaris UI:    @shopify/polaris
API Version:   2026-01 (latest stable)
```

### Project Structure

```
/
├── app/
│   ├── shopify.server.ts          # Shopify app config + session storage
│   ├── routes/
│   │   ├── auth.$.tsx             # OAuth install/callback (handled by Remix adapter)
│   │   ├── webhooks.tsx           # All webhook topics → dispatcher
│   │   ├── app._index.tsx         # Dashboard (embedded)
│   │   ├── app.orders.tsx         # Orders view (embedded)
│   │   └── app.payouts.tsx        # Payouts view (embedded)
│   ├── lib/
│   │   ├── shopify/
│   │   │   ├── client.ts          # GraphQL client factory
│   │   │   ├── orders.ts          # Order sync logic
│   │   │   ├── payouts.ts         # Payout reconciliation
│   │   │   ├── bulk-operations.ts # Bulk operation helpers
│   │   │   ├── webhooks/          # Per-topic webhook handlers
│   │   │   └── rate-limiter.ts    # Request queue + rate limiting
│   │   ├── accounting/
│   │   │   ├── journal-entries.ts # Shopify → accounting entry mapping
│   │   │   ├── tax-splitter.ts    # CGST/SGST/IGST splitting logic
│   │   │   └── reconciliation.ts  # Payout reconciliation algorithm
│   │   └── db/
│   │       └── prisma.ts          # Prisma client singleton
│   └── queues/
│       ├── order-sync.queue.ts
│       └── payout-sync.queue.ts
├── prisma/
│   └── schema.prisma
├── shopify.app.toml               # App config + scopes + webhooks
└── package.json
```

### Database Schema (Multi-Tenant)

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Shopify session storage (required by @shopify/shopify-app-remix)
model Session {
  id          String    @id
  shop        String
  state       String
  isOnline    Boolean   @default(false)
  scope       String?
  expires     DateTime?
  accessToken String
  userId      BigInt?
  firstName   String?
  lastName    String?
  email       String?
  accountOwner Boolean  @default(false)
  locale      String?
  collaborator Boolean?
  emailVerified Boolean?

  @@index([shop])
}

// One row per installed merchant
model Merchant {
  id                String   @id @default(cuid())
  shopDomain        String   @unique // e.g. "mystore.myshopify.com"
  shopName          String
  shopCurrency      String   // Base currency (INR, SGD, USD)
  shopCountryCode   String   // IN, SG, US
  shopTimezone      String   // IANA timezone
  hasShopifyPayments Boolean @default(false)
  accessToken       String   // Encrypted offline access token
  installedAt       DateTime @default(now())
  uninstalledAt     DateTime?
  lastSyncAt        DateTime?
  syncStatus        String   @default("pending") // pending | syncing | complete | error

  // Subscription
  subscriptionId    String?
  subscriptionStatus String?  // active | pending | declined | expired | frozen | cancelled

  // Relations
  orders            Order[]
  payouts           Payout[]
  webhookEvents     WebhookEvent[]

  @@index([shopDomain])
}

model Order {
  id                  String   @id              // Shopify GID e.g. "gid://shopify/Order/123"
  shopifyId           BigInt                    // Numeric Shopify order ID
  merchantId          String
  merchant            Merchant @relation(fields: [merchantId], references: [id])

  orderNumber         String                    // e.g. "#1001"
  createdAt           DateTime
  updatedAt           DateTime
  processedAt         DateTime?

  // Financial status
  financialStatus     String                    // paid | pending | refunded | voided | partially_refunded
  fulfillmentStatus   String?

  // Amounts (always shop_money / base currency)
  currency            String
  subtotalPrice       Decimal  @db.Decimal(14, 4)
  totalTax            Decimal  @db.Decimal(14, 4)
  totalShipping       Decimal  @db.Decimal(14, 4)
  totalDiscounts      Decimal  @db.Decimal(14, 4)
  totalPrice          Decimal  @db.Decimal(14, 4)
  currentTotalPrice   Decimal  @db.Decimal(14, 4)

  // Tax handling
  taxesIncluded       Boolean  @default(false)
  taxLines            Json     // Array of TaxLine objects

  // Multi-currency
  presentmentCurrency String?
  exchangeRate        Decimal? @db.Decimal(14, 8)

  // Gateway
  gateway             String?
  paymentGatewayNames String[] // Array

  // Relations
  lineItems           LineItem[]
  transactions        OrderTransaction[]
  refunds             Refund[]

  // Customer (denormalized for accounting; PII anonymized on redact)
  customerId          BigInt?
  customerEmail       String?

  // Raw payload (for reprocessing)
  rawPayload          Json?

  @@index([merchantId, createdAt])
  @@index([merchantId, financialStatus])
  @@index([shopifyId])
}

model LineItem {
  id              String   @id              // Shopify GID
  shopifyId       BigInt
  orderId         String
  order           Order    @relation(fields: [orderId], references: [id])

  title           String
  quantity        Int
  sku             String?
  variantId       BigInt?
  productId       BigInt?
  vendor          String?
  taxable         Boolean  @default(true)

  // Amounts
  unitPrice       Decimal  @db.Decimal(14, 4)
  totalPrice      Decimal  @db.Decimal(14, 4)  // unitPrice × quantity
  totalDiscount   Decimal  @db.Decimal(14, 4)
  totalTax        Decimal  @db.Decimal(14, 4)
  taxLines        Json     // Array of TaxLine objects

  @@index([orderId])
}

model OrderTransaction {
  id              String   @id              // Shopify GID
  shopifyId       BigInt
  orderId         String
  order           Order    @relation(fields: [orderId], references: [id])

  kind            String                    // authorization | capture | sale | void | refund
  status          String                    // pending | failure | success | error
  gateway         String
  amount          Decimal  @db.Decimal(14, 4)
  currency        String
  processedAt     DateTime
  parentId        BigInt?

  // Links to Shopify Payments balance transaction
  balanceTransactionId String?

  @@index([orderId])
  @@index([shopifyId])
}

model Payout {
  id              String   @id              // Shopify GID
  shopifyId       BigInt
  merchantId      String
  merchant        Merchant @relation(fields: [merchantId], references: [id])

  status          String                    // scheduled | in_transit | paid | failed | canceled
  date            DateTime
  currency        String
  amount          Decimal  @db.Decimal(14, 4)

  // Summary components
  chargesGross    Decimal  @db.Decimal(14, 4) @default(0)
  refundsGross    Decimal  @db.Decimal(14, 4) @default(0)
  adjustmentsGross Decimal @db.Decimal(14, 4) @default(0)
  feesTotal       Decimal  @db.Decimal(14, 4) @default(0)

  balanceTransactions BalanceTransaction[]

  @@index([merchantId, date])
  @@index([shopifyId])
}

model BalanceTransaction {
  id                        String   @id     // Shopify GID
  shopifyId                 BigInt
  payoutId                  String?
  payout                    Payout?  @relation(fields: [payoutId], references: [id])

  type                      String           // charge | refund | dispute | reserve | adjustment | payout
  amount                    Decimal  @db.Decimal(14, 4)
  fee                       Decimal  @db.Decimal(14, 4)
  net                       Decimal  @db.Decimal(14, 4)
  currency                  String
  processedAt               DateTime

  // Linkage back to order
  sourceOrderTransactionId  BigInt?
  sourceType                String?

  @@index([payoutId])
  @@index([sourceOrderTransactionId])
}

model Refund {
  id            String   @id              // Shopify GID
  shopifyId     BigInt
  orderId       String
  order         Order    @relation(fields: [orderId], references: [id])

  createdAt     DateTime
  note          String?
  refundAmount  Decimal  @db.Decimal(14, 4)
  taxAmount     Decimal  @db.Decimal(14, 4)
  refundLineItems Json   // Array of refund line items
  orderAdjustments Json  // Shipping adjustments etc.

  @@index([orderId])
}

model WebhookEvent {
  id            String   @id @default(cuid())
  merchantId    String
  merchant      Merchant @relation(fields: [merchantId], references: [id])

  topic         String                    // orders/create | refunds/create | etc.
  shopifyEventId String?                  // X-Shopify-Event-Id header (dedup key)
  webhookId     String?                   // X-Shopify-Webhook-Id header
  status        String   @default("pending") // pending | processed | failed | duplicate
  payload       Json?
  error         String?
  receivedAt    DateTime @default(now())
  processedAt   DateTime?

  @@unique([shopifyEventId])             // Idempotency
  @@index([merchantId, topic])
}
```

---

## 2. OAuth 2.0 Authentication

### Overview

Use the **Authorization Code grant** for the initial merchant install (offline access token). The offline token never expires and allows background syncs. For embedded app interactions, use **Token Exchange** to get short-lived tokens from App Bridge session tokens.

**CRITICAL:** Request `read_all_orders` scope — without it, you can only access orders from the last 60 days. This scope requires Shopify approval before it works in production. Apply via Partner Dashboard → Apps → [Your App] → API access → Access requests.

### shopify.app.toml Configuration

```toml
# shopify.app.toml
name = "MyMediset Accounting"
client_id = "your_api_key_here"
application_url = "https://yourapp.com"
embedded = true

[access_scopes]
scopes = "read_orders,read_all_orders,read_products,read_customers,read_inventory,read_locations,read_shipping,read_shopify_payments_payouts,read_shopify_payments_disputes,read_returns,read_draft_orders,read_markets,read_reports"

[auth]
redirect_urls = ["https://yourapp.com/auth/callback"]

[webhooks]
api_version = "2026-01"

[[webhooks.subscriptions]]
topics = [
  "orders/create",
  "orders/updated",
  "orders/paid",
  "orders/cancelled",
  "orders/fulfilled",
  "orders/edited",
  "refunds/create",
  "disputes/create",
  "disputes/update",
  "app/uninstalled",
  "app_subscriptions/update",
  "bulk_operations/finish"
]
uri = "https://yourapp.com/webhooks"

[[webhooks.subscriptions]]
compliance_topics = ["customers/data_request", "customers/redact", "shop/redact"]
uri = "https://yourapp.com/webhooks/compliance"

[pos]
embedded = false
```

### Shopify App Server Setup (Remix)

```typescript
// app/shopify.server.ts
import { shopifyApp } from "@shopify/shopify-app-remix/server";
import { PrismaSessionStorage } from "@shopify/shopify-app-session-storage-prisma";
import { prisma } from "~/lib/db/prisma";

const shopify = shopifyApp({
  apiKey: process.env.SHOPIFY_API_KEY!,
  apiSecretKey: process.env.SHOPIFY_API_SECRET!,
  apiVersion: "2026-01" as any,
  appUrl: process.env.SHOPIFY_APP_URL!,
  authPathPrefix: "/auth",
  sessionStorage: new PrismaSessionStorage(prisma),
  scopes: [
    "read_orders",
    "read_all_orders",
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
    "read_reports",
  ],
  hooks: {
    afterAuth: async ({ session }) => {
      // Called after each successful OAuth install/re-auth
      await onMerchantInstall(session);
    },
  },
  future: {
    unstable_newEmbeddedAuthStrategy: true, // Token Exchange flow
  },
});

export default shopify;
export const authenticate = shopify.authenticate;
export const unauthenticated = shopify.unauthenticated;

// Called after merchant installs or re-authorizes
async function onMerchantInstall(session: any) {
  const { shop, accessToken, scope } = session;

  // Fetch shop info for metadata
  const shopInfo = await fetchShopInfo(shop, accessToken);

  // Upsert merchant record
  await prisma.merchant.upsert({
    where: { shopDomain: shop },
    create: {
      shopDomain: shop,
      shopName: shopInfo.name,
      shopCurrency: shopInfo.currency,
      shopCountryCode: shopInfo.country_code,
      shopTimezone: shopInfo.iana_timezone,
      hasShopifyPayments: shopInfo.has_shopify_payments,
      accessToken: encryptToken(accessToken), // Always encrypt at rest
      installedAt: new Date(),
      uninstalledAt: null,
    },
    update: {
      shopName: shopInfo.name,
      shopCurrency: shopInfo.currency,
      shopCountryCode: shopInfo.country_code,
      shopTimezone: shopInfo.iana_timezone,
      hasShopifyPayments: shopInfo.has_shopify_payments,
      accessToken: encryptToken(accessToken),
      uninstalledAt: null,
    },
  });

  // Trigger historical data sync
  await enqueueBulkHistoricalSync(shop);
}

async function fetchShopInfo(shop: string, accessToken: string) {
  const response = await fetch(
    `https://${shop}/admin/api/2026-01/shop.json`,
    { headers: { "X-Shopify-Access-Token": accessToken } }
  );
  if (!response.ok) throw new Error(`Failed to fetch shop info: ${response.status}`);
  const { shop: shopData } = await response.json();
  return shopData;
}
```

### Manual OAuth Flow (without Remix adapter — for reference)

```typescript
// lib/shopify/oauth.ts
import crypto from "crypto";

const SHOPIFY_API_KEY = process.env.SHOPIFY_API_KEY!;
const SHOPIFY_API_SECRET = process.env.SHOPIFY_API_SECRET!;
const REDIRECT_URI = process.env.SHOPIFY_REDIRECT_URI!;

const SCOPES = [
  "read_orders",
  "read_all_orders",
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
  "read_reports",
].join(",");

// Step 1: Generate install URL (redirect merchant here)
export function buildInstallUrl(shop: string): { url: string; nonce: string } {
  const nonce = crypto.randomBytes(16).toString("hex");
  const url = `https://${shop}/admin/oauth/authorize?` +
    `client_id=${SHOPIFY_API_KEY}` +
    `&scope=${encodeURIComponent(SCOPES)}` +
    `&redirect_uri=${encodeURIComponent(REDIRECT_URI)}` +
    `&state=${nonce}`;
  return { url, nonce };
}

// Step 2: Verify the OAuth callback — call this in your /auth/callback route
export function verifyOAuthCallback(
  queryParams: Record<string, string>,
  savedNonce: string
): void {
  const { hmac, state, shop, ...rest } = queryParams;

  // CSRF protection: verify state matches nonce
  if (state !== savedNonce) {
    throw new Error("State mismatch — possible CSRF attack");
  }

  // Verify shop is a valid myshopify.com domain
  if (!shop.match(/^[a-zA-Z0-9][a-zA-Z0-9-]*\.myshopify\.com$/)) {
    throw new Error("Invalid shop domain");
  }

  // Verify HMAC signature
  const message = Object.entries(rest)
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([k, v]) => `${k}=${v}`)
    .join("&");

  const computed = crypto
    .createHmac("sha256", SHOPIFY_API_SECRET)
    .update(message)
    .digest("hex");

  if (!crypto.timingSafeEqual(Buffer.from(hmac), Buffer.from(computed))) {
    throw new Error("HMAC verification failed");
  }
}

// Step 3: Exchange code for permanent offline access token
export async function exchangeCodeForToken(
  shop: string,
  code: string
): Promise<string> {
  const response = await fetch(
    `https://${shop}/admin/oauth/access_token`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        client_id: SHOPIFY_API_KEY,
        client_secret: SHOPIFY_API_SECRET,
        code,
      }),
    }
  );

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Token exchange failed: ${response.status} ${error}`);
  }

  const { access_token, scope } = await response.json();

  // Validate granted scopes contain required scopes
  const grantedScopes = scope.split(",");
  const requiredScopes = ["read_orders", "read_products"];
  for (const required of requiredScopes) {
    if (!grantedScopes.includes(required)) {
      throw new Error(`Missing required scope: ${required}`);
    }
  }

  return access_token;
}

// Token encryption at rest (use AWS KMS or similar in production)
export function encryptToken(token: string): string {
  const key = Buffer.from(process.env.TOKEN_ENCRYPTION_KEY!, "hex"); // 32-byte key
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv("aes-256-cbc", key, iv);
  const encrypted = Buffer.concat([cipher.update(token, "utf8"), cipher.final()]);
  return `${iv.toString("hex")}:${encrypted.toString("hex")}`;
}

export function decryptToken(encrypted: string): string {
  const [ivHex, dataHex] = encrypted.split(":");
  const key = Buffer.from(process.env.TOKEN_ENCRYPTION_KEY!, "hex");
  const iv = Buffer.from(ivHex, "hex");
  const data = Buffer.from(dataHex, "hex");
  const decipher = crypto.createDecipheriv("aes-256-cbc", key, iv);
  return Buffer.concat([decipher.update(data), decipher.final()]).toString("utf8");
}
```

### Getting the Access Token for API Calls

```typescript
// lib/shopify/client.ts
import { prisma } from "~/lib/db/prisma";
import { decryptToken } from "./oauth";

export async function getAccessToken(shopDomain: string): Promise<string> {
  const merchant = await prisma.merchant.findUniqueOrThrow({
    where: { shopDomain },
    select: { accessToken: true, uninstalledAt: true },
  });

  if (merchant.uninstalledAt) {
    throw new Error(`App uninstalled from ${shopDomain}`);
  }

  return decryptToken(merchant.accessToken);
}
```

---

## 3. Order Data Extraction

### Critical Rules

1. **ALWAYS use `shopMoney` / `shop_money` for accounting** — never `presentmentMoney`. The shop_money field is always in the merchant's base currency regardless of what currency the customer paid in.
2. Use API version `2026-01` — REST is legacy; use GraphQL for all new code.
3. For real-time sync: use webhooks (`orders/create`, `orders/paid`, `orders/updated`).
4. For historical/initial sync: use GraphQL Bulk Operations (see Section 10).

### GraphQL Query for Orders

```typescript
// lib/shopify/orders.ts
import { getAccessToken } from "./client";

const ORDERS_QUERY = `
  query GetOrders($cursor: String, $query: String) {
    orders(first: 250, after: $cursor, query: $query, sortKey: CREATED_AT) {
      pageInfo {
        hasNextPage
        endCursor
      }
      edges {
        node {
          id
          name
          legacyResourceId
          createdAt
          updatedAt
          processedAt
          displayFinancialStatus
          displayFulfillmentStatus
          cancelledAt

          # Always use shopMoney for accounting
          subtotalPriceSet {
            shopMoney { amount currencyCode }
            presentmentMoney { amount currencyCode }
          }
          totalPriceSet {
            shopMoney { amount currencyCode }
            presentmentMoney { amount currencyCode }
          }
          totalTaxSet {
            shopMoney { amount currencyCode }
          }
          totalShippingPriceSet {
            shopMoney { amount currencyCode }
          }
          totalDiscountsSet {
            shopMoney { amount currencyCode }
          }
          currentTotalPriceSet {
            shopMoney { amount currencyCode }
          }

          taxesIncluded
          taxLines {
            title
            rate
            priceSet {
              shopMoney { amount currencyCode }
            }
            channelLiable
          }

          lineItems(first: 100) {
            edges {
              node {
                id
                name
                quantity
                sku
                variantId: variant { id }
                product { id }
                vendor
                taxable
                originalUnitPriceSet {
                  shopMoney { amount currencyCode }
                }
                discountedUnitPriceSet {
                  shopMoney { amount currencyCode }
                }
                totalDiscountSet {
                  shopMoney { amount currencyCode }
                }
                taxLines {
                  title
                  rate
                  priceSet {
                    shopMoney { amount currencyCode }
                  }
                  channelLiable
                }
              }
            }
          }

          shippingLines(first: 10) {
            edges {
              node {
                id
                title
                code
                originalPriceSet {
                  shopMoney { amount currencyCode }
                }
                discountedPriceSet {
                  shopMoney { amount currencyCode }
                }
                taxLines {
                  title rate
                  priceSet { shopMoney { amount currencyCode } }
                }
              }
            }
          }

          transactions(first: 20) {
            id
            legacyResourceId
            kind
            status
            amountSet {
              shopMoney { amount currencyCode }
            }
            gateway
            processedAt
            parentTransaction { legacyResourceId }
          }

          paymentGatewayNames

          customer {
            id
            legacyResourceId
            email
          }

          discountCodes {
            code
            amount
            type
          }

          refunds(first: 10) {
            id
            createdAt
            note
          }
        }
      }
    }
  }
`;

// GraphQL client for a specific shop
export async function shopifyGraphQL(
  shopDomain: string,
  query: string,
  variables?: Record<string, unknown>
): Promise<any> {
  const accessToken = await getAccessToken(shopDomain);

  const response = await fetch(
    `https://${shopDomain}/admin/api/2026-01/graphql.json`,
    {
      method: "POST",
      headers: {
        "X-Shopify-Access-Token": accessToken,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ query, variables }),
    }
  );

  if (!response.ok) {
    throw new Error(`GraphQL request failed: ${response.status}`);
  }

  const result = await response.json();

  if (result.errors?.length) {
    throw new Error(`GraphQL errors: ${JSON.stringify(result.errors)}`);
  }

  // Check throttle status proactively
  const throttle = result.extensions?.cost?.throttleStatus;
  if (throttle && throttle.currentlyAvailable < 200) {
    const waitMs = (200 / throttle.restoreRate) * 1000;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }

  return result.data;
}

// Sync orders incrementally — call on webhook trigger or scheduled sync
export async function syncOrdersForShop(
  shopDomain: string,
  options: {
    sinceDate?: Date;    // For incremental sync — use order updatedAt
    statusFilter?: string; // e.g. "financial_status:paid"
  } = {}
): Promise<number> {
  let cursor: string | null = null;
  let synced = 0;

  // Build query filter string
  const queryParts: string[] = [];
  if (options.sinceDate) {
    const iso = options.sinceDate.toISOString();
    queryParts.push(`updated_at:>='${iso}'`);
  }
  if (options.statusFilter) {
    queryParts.push(options.statusFilter);
  }
  const queryFilter = queryParts.join(" AND ") || undefined;

  do {
    const data = await shopifyGraphQL(shopDomain, ORDERS_QUERY, {
      cursor,
      query: queryFilter,
    });

    const { edges, pageInfo } = data.orders;

    // Process each order
    const orderPromises = edges.map((edge: any) =>
      upsertOrder(shopDomain, edge.node)
    );
    await Promise.all(orderPromises);

    synced += edges.length;
    cursor = pageInfo.hasNextPage ? pageInfo.endCursor : null;
  } while (cursor !== null);

  return synced;
}

// Upsert a single order from GraphQL response into the database
export async function upsertOrder(shopDomain: string, node: any): Promise<void> {
  const merchant = await prisma.merchant.findUniqueOrThrow({
    where: { shopDomain },
    select: { id: true },
  });

  // Extract numeric ID from GID: "gid://shopify/Order/820982911946154508"
  const shopifyId = BigInt(node.id.split("/").pop());
  const currency = node.subtotalPriceSet.shopMoney.currencyCode;

  // Calculate exchange rate if multi-currency
  let exchangeRate: number | null = null;
  const shopAmt = parseFloat(node.totalPriceSet.shopMoney.amount);
  const presAmt = parseFloat(node.totalPriceSet.presentmentMoney.amount);
  const pressCurrency = node.totalPriceSet.presentmentMoney.currencyCode;
  if (pressCurrency !== currency && presAmt > 0) {
    exchangeRate = shopAmt / presAmt;
  }

  const orderData = {
    shopifyId,
    merchantId: merchant.id,
    orderNumber: node.name,
    createdAt: new Date(node.createdAt),
    updatedAt: new Date(node.updatedAt),
    processedAt: node.processedAt ? new Date(node.processedAt) : null,
    financialStatus: node.displayFinancialStatus,
    fulfillmentStatus: node.displayFulfillmentStatus,
    currency,
    subtotalPrice: parseFloat(node.subtotalPriceSet.shopMoney.amount),
    totalTax: parseFloat(node.totalTaxSet.shopMoney.amount),
    totalShipping: parseFloat(node.totalShippingPriceSet.shopMoney.amount),
    totalDiscounts: parseFloat(node.totalDiscountsSet.shopMoney.amount),
    totalPrice: parseFloat(node.totalPriceSet.shopMoney.amount),
    currentTotalPrice: parseFloat(node.currentTotalPriceSet.shopMoney.amount),
    taxesIncluded: node.taxesIncluded,
    taxLines: node.taxLines,
    presentmentCurrency: pressCurrency !== currency ? pressCurrency : null,
    exchangeRate,
    gateway: node.paymentGatewayNames?.[0] ?? null,
    paymentGatewayNames: node.paymentGatewayNames ?? [],
    customerId: node.customer ? BigInt(node.customer.legacyResourceId) : null,
    customerEmail: node.customer?.email ?? null,
    rawPayload: node,
  };

  await prisma.order.upsert({
    where: { id: node.id },
    create: { id: node.id, ...orderData },
    update: orderData,
  });

  // Upsert line items
  for (const edge of node.lineItems.edges) {
    const li = edge.node;
    const liShopifyId = BigInt(li.id.split("/").pop());
    const liData = {
      shopifyId: liShopifyId,
      orderId: node.id,
      title: li.name,
      quantity: li.quantity,
      sku: li.sku ?? null,
      variantId: li.variantId?.id ? BigInt(li.variantId.id.split("/").pop()) : null,
      productId: li.product?.id ? BigInt(li.product.id.split("/").pop()) : null,
      vendor: li.vendor ?? null,
      taxable: li.taxable,
      unitPrice: parseFloat(li.originalUnitPriceSet.shopMoney.amount),
      totalPrice:
        parseFloat(li.originalUnitPriceSet.shopMoney.amount) * li.quantity,
      totalDiscount: parseFloat(li.totalDiscountSet?.shopMoney.amount ?? "0"),
      totalTax: li.taxLines.reduce(
        (sum: number, tl: any) => sum + parseFloat(tl.priceSet.shopMoney.amount),
        0
      ),
      taxLines: li.taxLines,
    };

    await prisma.lineItem.upsert({
      where: { id: li.id },
      create: { id: li.id, ...liData },
      update: liData,
    });
  }

  // Upsert transactions
  for (const tx of node.transactions) {
    await upsertOrderTransaction(node.id, tx);
  }
}
```

---

## 4. Transaction & Payment Data

### Transaction Types and Accounting Mapping

| `kind` | Accounting Treatment |
|--------|---------------------|
| `sale` | DR Accounts Receivable / CR Revenue + Tax Payable |
| `capture` | DR Cash/Stripe Clearing / CR Accounts Receivable |
| `authorization` | Memo only — no accounting entry |
| `void` | Reverses the authorization — no accounting entry |
| `refund` | DR Revenue (reversal) / CR Accounts Receivable or Cash |

Only process transactions with `status: "success"` for accounting entries.

### Fetching Transactions via GraphQL

```typescript
// lib/shopify/transactions.ts

const ORDER_TRANSACTIONS_QUERY = `
  query GetOrderTransactions($orderId: ID!) {
    order(id: $orderId) {
      id
      transactions(first: 50) {
        id
        legacyResourceId
        kind
        status
        amountSet {
          shopMoney { amount currencyCode }
        }
        gateway
        processedAt
        paymentId
        parentTransaction {
          id
          legacyResourceId
          kind
        }
        receiptJson
      }
    }
  }
`;

export async function fetchAndStoreTransactions(
  shopDomain: string,
  orderGid: string
): Promise<void> {
  const data = await shopifyGraphQL(shopDomain, ORDER_TRANSACTIONS_QUERY, {
    orderId: orderGid,
  });

  const transactions = data.order.transactions;

  for (const tx of transactions) {
    await upsertOrderTransaction(orderGid, tx);
  }
}

export async function upsertOrderTransaction(
  orderGid: string,
  tx: any
): Promise<void> {
  const shopifyId = BigInt(tx.id.split("/").pop());

  await prisma.orderTransaction.upsert({
    where: { id: tx.id },
    create: {
      id: tx.id,
      shopifyId,
      orderId: orderGid,
      kind: tx.kind.toLowerCase(),
      status: tx.status.toLowerCase(),
      gateway: tx.gateway,
      amount: parseFloat(tx.amountSet.shopMoney.amount),
      currency: tx.amountSet.shopMoney.currencyCode,
      processedAt: new Date(tx.processedAt),
      parentId: tx.parentTransaction
        ? BigInt(tx.parentTransaction.legacyResourceId)
        : null,
    },
    update: {
      kind: tx.kind.toLowerCase(),
      status: tx.status.toLowerCase(),
      amount: parseFloat(tx.amountSet.shopMoney.amount),
      processedAt: new Date(tx.processedAt),
    },
  });
}

// Identify the effective payment gateway for an order
export function getOrderGateway(order: {
  paymentGatewayNames: string[];
  gateway?: string | null;
}): string {
  if (order.paymentGatewayNames.includes("shopify_payments")) {
    return "shopify_payments";
  }
  return order.paymentGatewayNames[0] ?? order.gateway ?? "unknown";
}
```

---

## 5. Shopify Payments / Payout Reconciliation

### Architecture

```
Bank Deposit (Payout)
    └── BalanceTransactions (many)
            ├── type=charge  → source_order_transaction_id → OrderTransaction → Order
            ├── type=refund  → source_order_transaction_id → OrderTransaction → Order
            ├── type=dispute → links to a chargeback
            ├── type=reserve
            └── type=adjustment
```

**Payout amount** = Σ(balance_transactions.net) for all transactions with that payout_id.

### Step 1: Fetch Payouts

```typescript
// lib/shopify/payouts.ts

const PAYOUTS_QUERY = `
  query GetPayouts($cursor: String) {
    shopifyPaymentsAccount {
      payouts(first: 50, after: $cursor) {
        pageInfo {
          hasNextPage
          endCursor
        }
        edges {
          node {
            id
            legacyResourceId
            status
            issuedAt
            net { amount currencyCode }
            gross { amount currencyCode }
            fee { amount currencyCode }
            summary {
              chargesGross { amount currencyCode }
              chargesFee { amount currencyCode }
              refundsFeeGross { amount currencyCode }
              adjustmentsFeeAmount { amount currencyCode }
              adjustmentsGross { amount currencyCode }
            }
          }
        }
      }
    }
  }
`;

export async function syncPayouts(shopDomain: string): Promise<void> {
  const merchant = await prisma.merchant.findUniqueOrThrow({
    where: { shopDomain },
  });

  if (!merchant.hasShopifyPayments) return; // Non-Shopify Payments stores don't have payouts

  let cursor: string | null = null;

  do {
    const data = await shopifyGraphQL(shopDomain, PAYOUTS_QUERY, { cursor });
    const { edges, pageInfo } = data.shopifyPaymentsAccount.payouts;

    for (const edge of edges) {
      const p = edge.node;
      const shopifyId = BigInt(p.id.split("/").pop());

      const payoutData = {
        shopifyId,
        merchantId: merchant.id,
        status: p.status.toLowerCase(),
        date: new Date(p.issuedAt),
        currency: p.net.currencyCode,
        amount: parseFloat(p.net.amount),
        chargesGross: parseFloat(p.summary.chargesGross.amount),
        refundsGross: parseFloat(p.summary.refundsFeeGross.amount),
        adjustmentsGross: parseFloat(p.summary.adjustmentsGross.amount),
        feesTotal: parseFloat(p.fee.amount),
      };

      await prisma.payout.upsert({
        where: { id: p.id },
        create: { id: p.id, ...payoutData },
        update: payoutData,
      });

      // Fetch balance transactions for this payout
      await syncBalanceTransactionsForPayout(shopDomain, p.id);
    }

    cursor = pageInfo.hasNextPage ? pageInfo.endCursor : null;
  } while (cursor !== null);
}
```

### Step 2: Fetch Balance Transactions (The Reconciliation Key)

```typescript
const BALANCE_TRANSACTIONS_QUERY = `
  query GetBalanceTransactions($payoutId: ID, $cursor: String) {
    shopifyPaymentsAccount {
      balanceTransactions(first: 100, after: $cursor, payoutId: $payoutId) {
        pageInfo {
          hasNextPage
          endCursor
        }
        edges {
          node {
            id
            type
            net { amount currencyCode }
            fee { amount currencyCode }
            amount { amount currencyCode }
            processedAt
            associatedPayout { id }
            sourceOrderTransaction {
              id
              legacyResourceId
              order {
                id
                name
              }
            }
          }
        }
      }
    }
  }
`;

export async function syncBalanceTransactionsForPayout(
  shopDomain: string,
  payoutGid: string
): Promise<void> {
  // Extract numeric payout ID for the filter
  const payoutShopifyId = payoutGid.split("/").pop();
  let cursor: string | null = null;

  do {
    const data = await shopifyGraphQL(shopDomain, BALANCE_TRANSACTIONS_QUERY, {
      payoutId: payoutGid,
      cursor,
    });

    const { edges, pageInfo } =
      data.shopifyPaymentsAccount.balanceTransactions;

    for (const edge of edges) {
      const bt = edge.node;
      const shopifyId = BigInt(bt.id.split("/").pop());

      const btData = {
        shopifyId,
        payoutId: payoutGid,
        type: bt.type.toLowerCase(),
        amount: parseFloat(bt.amount.amount),
        fee: parseFloat(bt.fee.amount),
        net: parseFloat(bt.net.amount),
        currency: bt.net.currencyCode,
        processedAt: new Date(bt.processedAt),
        sourceOrderTransactionId: bt.sourceOrderTransaction
          ? BigInt(bt.sourceOrderTransaction.legacyResourceId)
          : null,
        sourceType: bt.sourceOrderTransaction ? "order_transaction" : null,
      };

      await prisma.balanceTransaction.upsert({
        where: { id: bt.id },
        create: { id: bt.id, ...btData },
        update: btData,
      });

      // Link balance transaction back to order transaction
      if (bt.sourceOrderTransaction) {
        const orderTxShopifyId = BigInt(bt.sourceOrderTransaction.legacyResourceId);
        await prisma.orderTransaction.updateMany({
          where: { shopifyId: orderTxShopifyId },
          data: { balanceTransactionId: bt.id },
        });
      }
    }

    cursor = pageInfo.hasNextPage ? pageInfo.endCursor : null;
  } while (cursor !== null);
}
```

### Step 3: Reconciliation Algorithm

```typescript
// lib/accounting/reconciliation.ts

export interface ReconciliationResult {
  payoutId: string;
  payoutAmount: number;
  sumOfBalanceTransactionNets: number;
  difference: number;
  isReconciled: boolean;
  lineItems: ReconciliationLineItem[];
}

export interface ReconciliationLineItem {
  balanceTransactionId: string;
  type: string;
  orderId?: string;
  orderName?: string;
  amount: number;
  fee: number;
  net: number;
}

export async function reconcilePayout(
  payoutGid: string
): Promise<ReconciliationResult> {
  const payout = await prisma.payout.findUniqueOrThrow({
    where: { id: payoutGid },
    include: {
      balanceTransactions: true,
    },
  });

  const lineItems: ReconciliationLineItem[] = [];
  let sumNets = 0;

  for (const bt of payout.balanceTransactions) {
    let orderId: string | undefined;
    let orderName: string | undefined;

    // If this balance transaction links to an order transaction, find the order
    if (bt.sourceOrderTransactionId) {
      const orderTx = await prisma.orderTransaction.findFirst({
        where: { shopifyId: bt.sourceOrderTransactionId },
        include: { order: { select: { id: true, orderNumber: true } } },
      });
      orderId = orderTx?.order?.id;
      orderName = orderTx?.order?.orderNumber;
    }

    const net = Number(bt.net);
    sumNets += net;

    lineItems.push({
      balanceTransactionId: bt.id,
      type: bt.type,
      orderId,
      orderName,
      amount: Number(bt.amount),
      fee: Number(bt.fee),
      net,
    });
  }

  const payoutAmount = Number(payout.amount);
  const difference = Math.abs(payoutAmount - sumNets);
  const isReconciled = difference < 0.01; // Within 1 cent tolerance

  return {
    payoutId: payoutGid,
    payoutAmount,
    sumOfBalanceTransactionNets: sumNets,
    difference,
    isReconciled,
    lineItems,
  };
}

// Generate accounting entries from a payout
export async function generatePayoutJournalEntries(payoutGid: string) {
  const result = await reconcilePayout(payoutGid);
  const entries = [];

  // Payout received entry
  entries.push({
    date: new Date(),
    description: `Shopify Payments payout received`,
    debit: { account: "Bank / Current Account", amount: result.payoutAmount },
    credit: { account: "Shopify Payments Clearing", amount: result.payoutAmount },
    reference: payoutGid,
  });

  // Fee entries (from balance transactions)
  const totalFees = result.lineItems.reduce((sum, li) => sum + li.fee, 0);
  if (totalFees > 0) {
    entries.push({
      date: new Date(),
      description: "Shopify payment processing fees",
      debit: { account: "Payment Gateway Fees (Expense)", amount: totalFees },
      credit: { account: "Shopify Payments Clearing", amount: totalFees },
      reference: payoutGid,
    });
  }

  return entries;
}
```

---

## 6. Refunds & Returns

### Refund Data Structure

A refund on an order contains:
- `refundLineItems`: which products were returned and quantities
- `transactions`: the actual money returned to the customer (kind = `refund`)
- `orderAdjustments`: shipping refunds or manual adjustments

### Fetching Refunds

```typescript
// lib/shopify/refunds.ts

const ORDER_REFUNDS_QUERY = `
  query GetOrderRefunds($orderId: ID!) {
    order(id: $orderId) {
      refunds {
        id
        createdAt
        note
        totalRefundedSet {
          shopMoney { amount currencyCode }
        }
        refundLineItems(first: 50) {
          edges {
            node {
              quantity
              restockType
              subtotalSet {
                shopMoney { amount currencyCode }
              }
              totalTaxSet {
                shopMoney { amount currencyCode }
              }
              lineItem {
                id
                name
                sku
              }
            }
          }
        }
        orderAdjustments {
          id
          kind
          reason
          amountSet {
            shopMoney { amount currencyCode }
          }
          taxAmountSet {
            shopMoney { amount currencyCode }
          }
        }
        transactions {
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
`;

export async function syncRefundsForOrder(
  shopDomain: string,
  orderGid: string
): Promise<void> {
  const data = await shopifyGraphQL(shopDomain, ORDER_REFUNDS_QUERY, {
    orderId: orderGid,
  });

  const refunds = data.order.refunds;

  for (const refund of refunds) {
    // Sum the actual money returned via successful refund transactions
    const refundTransactions = refund.transactions.filter(
      (tx: any) => tx.kind.toLowerCase() === "refund" && tx.status.toLowerCase() === "success"
    );
    const refundAmount = refundTransactions.reduce(
      (sum: number, tx: any) => sum + parseFloat(tx.amountSet.shopMoney.amount),
      0
    );

    // Sum tax refunded from line items
    const taxAmount = refund.refundLineItems.edges.reduce(
      (sum: number, edge: any) =>
        sum + parseFloat(edge.node.totalTaxSet.shopMoney.amount),
      0
    );

    const refundData = {
      shopifyId: BigInt(refund.id.split("/").pop()),
      orderId: orderGid,
      createdAt: new Date(refund.createdAt),
      note: refund.note ?? null,
      refundAmount,
      taxAmount,
      refundLineItems: refund.refundLineItems.edges.map((e: any) => e.node),
      orderAdjustments: refund.orderAdjustments,
    };

    await prisma.refund.upsert({
      where: { id: refund.id },
      create: { id: refund.id, ...refundData },
      update: refundData,
    });
  }
}

// Generate accounting journal entry for a refund
export function buildRefundJournalEntry(refund: {
  orderId: string;
  orderNumber: string;
  refundAmount: number;
  taxAmount: number;
  currency: string;
  shippingRefund?: number;
}) {
  const revenueReversal = refund.refundAmount - refund.taxAmount - (refund.shippingRefund ?? 0);

  return {
    date: new Date(),
    description: `Refund on order ${refund.orderNumber}`,
    entries: [
      {
        account: "Revenue (Sales)",
        debit: revenueReversal,
        credit: 0,
        note: "Revenue reversal",
      },
      {
        account: "Tax Payable (GST/VAT)",
        debit: refund.taxAmount,
        credit: 0,
        note: "Tax reversal",
      },
      ...(refund.shippingRefund
        ? [{ account: "Shipping Revenue", debit: refund.shippingRefund, credit: 0 }]
        : []),
      {
        account: "Accounts Receivable / Shopify Clearing",
        debit: 0,
        credit: refund.refundAmount,
        note: "Refund cleared",
      },
    ],
    reference: refund.orderId,
  };
}
```

---

## 7. Tax Data Extraction (India Focus)

### How Shopify Stores Indian GST

For Indian stores, Shopify records GST components as separate `tax_lines` entries:
- Intra-state sales: two `tax_lines` entries — `title: "CGST"` and `title: "SGST"`, each at 9% for an 18% GST item
- Inter-state sales: one `tax_lines` entry — `title: "IGST"` at the full rate (e.g., 18%)

**Important:** Shopify does NOT store HSN codes in orders or GSTIN on customer profiles. These must be stored as metafields and joined at report time.

### Tax Extraction and Splitting Logic

```typescript
// lib/accounting/tax-splitter.ts

export interface TaxLine {
  title: string;          // "CGST", "SGST", "IGST", "GST", "TAX", etc.
  rate: number;           // Decimal rate e.g. 0.09
  price: number;          // Amount in shop currency
  channelLiable: boolean; // If true, marketplace remits — do NOT record as your liability
}

export interface SplitTaxResult {
  cgst: number;
  sgst: number;
  igst: number;
  otherTax: number;       // VAT, customs, etc.
  total: number;
  isInterState: boolean;
  channelLiableAmount: number; // Excluded from your liability
}

// For Indian (INR) stores: split tax_lines into CGST/SGST/IGST
export function splitIndianTax(taxLines: TaxLine[]): SplitTaxResult {
  let cgst = 0;
  let sgst = 0;
  let igst = 0;
  let otherTax = 0;
  let channelLiableAmount = 0;

  for (const line of taxLines) {
    // Exclude marketplace-liable taxes from your accounting
    if (line.channelLiable) {
      channelLiableAmount += line.price;
      continue;
    }

    const titleUpper = line.title.toUpperCase().trim();

    if (titleUpper.includes("CGST")) {
      cgst += line.price;
    } else if (titleUpper.includes("SGST") || titleUpper.includes("UTGST")) {
      sgst += line.price;
    } else if (titleUpper.includes("IGST")) {
      igst += line.price;
    } else if (titleUpper.includes("GST") || titleUpper.includes("TAX")) {
      // Shopify sometimes stores as generic "GST" — need to split based on place of supply
      // This requires your own logic comparing seller state to buyer state
      // Default: treat as ambiguous, add to otherTax for manual review
      otherTax += line.price;
    } else {
      otherTax += line.price;
    }
  }

  const isInterState = igst > 0 && cgst === 0;
  const total = cgst + sgst + igst + otherTax;

  return { cgst, sgst, igst, otherTax, total, isInterState, channelLiableAmount };
}

// For orders where Shopify stores combined tax (not split CGST/SGST)
// Use buyer state vs seller state to determine intra/inter-state
export function inferGSTType(
  sellerGSTIN: string,       // Your GSTIN e.g. "27AAAA..."
  buyerStateCode: string,    // 2-char state code from shipping address
  taxAmount: number,
  taxRate: number            // Combined rate e.g. 0.18
): SplitTaxResult {
  const sellerStateCode = sellerGSTIN.substring(0, 2); // GSTIN starts with state code

  const isInterState = sellerStateCode !== buyerStateCode;

  if (isInterState) {
    return {
      cgst: 0,
      sgst: 0,
      igst: taxAmount,
      otherTax: 0,
      total: taxAmount,
      isInterState: true,
      channelLiableAmount: 0,
    };
  } else {
    const half = parseFloat((taxAmount / 2).toFixed(4));
    return {
      cgst: half,
      sgst: taxAmount - half, // Handle rounding by giving remainder to SGST
      igst: 0,
      otherTax: 0,
      total: taxAmount,
      isInterState: false,
      channelLiableAmount: 0,
    };
  }
}

// Handle taxes_included = true (price INCLUDES tax — extract it)
export function extractIncludedTax(
  totalPriceInclTax: number,
  taxRate: number
): { priceExcluding: number; taxAmount: number } {
  // total = price_excl + (price_excl × rate) = price_excl × (1 + rate)
  const priceExcluding = totalPriceInclTax / (1 + taxRate);
  const taxAmount = totalPriceInclTax - priceExcluding;
  return {
    priceExcluding: parseFloat(priceExcluding.toFixed(4)),
    taxAmount: parseFloat(taxAmount.toFixed(4)),
  };
}

// Build the full tax summary for an order (used when generating GST reports)
export function buildOrderTaxSummary(order: {
  taxesIncluded: boolean;
  totalTax: number;
  subtotalPrice: number;
  taxLines: TaxLine[];
  lineItems: Array<{ taxLines: TaxLine[]; totalTax: number }>;
}) {
  // Use line-item-level tax_lines for HSN-level breakdown (GST filing)
  const lineItemTaxBreakdown = order.lineItems.map((li) => ({
    taxes: splitIndianTax(li.taxLines),
    amount: li.totalTax,
  }));

  // Order-level aggregated split
  const orderTaxSplit = splitIndianTax(order.taxLines);

  return {
    taxesIncluded: order.taxesIncluded,
    totalTax: order.totalTax,
    orderLevelSplit: orderTaxSplit,
    lineItemBreakdown: lineItemTaxBreakdown,
    // Warning flag if Shopify hasn't fully separated CGST/SGST
    requiresManualSplit:
      orderTaxSplit.otherTax > 0 || (orderTaxSplit.cgst === 0 && orderTaxSplit.igst === 0),
  };
}
```

### Indian State Codes for GSTIN Matching

```typescript
// lib/accounting/india-states.ts
export const INDIA_STATE_CODES: Record<string, string> = {
  "01": "Jammu & Kashmir",
  "02": "Himachal Pradesh",
  "03": "Punjab",
  "04": "Chandigarh",
  "05": "Uttarakhand",
  "06": "Haryana",
  "07": "Delhi",
  "08": "Rajasthan",
  "09": "Uttar Pradesh",
  "10": "Bihar",
  "11": "Sikkim",
  "12": "Arunachal Pradesh",
  "13": "Nagaland",
  "14": "Manipur",
  "15": "Mizoram",
  "16": "Tripura",
  "17": "Meghalaya",
  "18": "Assam",
  "19": "West Bengal",
  "20": "Jharkhand",
  "21": "Odisha",
  "22": "Chhattisgarh",
  "23": "Madhya Pradesh",
  "24": "Gujarat",
  "25": "Daman & Diu",
  "26": "Dadra & Nagar Haveli",
  "27": "Maharashtra",
  "28": "Andhra Pradesh (Old)",
  "29": "Karnataka",
  "30": "Goa",
  "31": "Lakshadweep",
  "32": "Kerala",
  "33": "Tamil Nadu",
  "34": "Pondicherry",
  "35": "Andaman & Nicobar",
  "36": "Telangana",
  "37": "Andhra Pradesh",
  "38": "Ladakh",
};

// Map Shopify province_code to GSTIN state code
// Shopify uses ISO 3166-2 codes (IN-MH, IN-KA etc.)
export const SHOPIFY_PROVINCE_TO_GSTIN_STATE: Record<string, string> = {
  "IN-MH": "27", // Maharashtra
  "IN-KA": "29", // Karnataka
  "IN-TN": "33", // Tamil Nadu
  "IN-DL": "07", // Delhi
  "IN-GJ": "24", // Gujarat
  "IN-RJ": "08", // Rajasthan
  "IN-UP": "09", // Uttar Pradesh
  "IN-WB": "19", // West Bengal
  "IN-TS": "36", // Telangana
  "IN-AP": "37", // Andhra Pradesh
  "IN-KL": "32", // Kerala
  "IN-HR": "06", // Haryana
  "IN-PB": "03", // Punjab
  "IN-BR": "10", // Bihar
  "IN-OR": "21", // Odisha
  "IN-AS": "18", // Assam
  "IN-JH": "20", // Jharkhand
  "IN-CG": "22", // Chhattisgarh
  "IN-HP": "02", // Himachal Pradesh
  "IN-UK": "05", // Uttarakhand
  "IN-GA": "30", // Goa
  "IN-MP": "23", // Madhya Pradesh
  "IN-CH": "04", // Chandigarh
  "IN-JK": "01", // J&K
  "IN-PY": "34", // Puducherry
  "IN-MN": "14", // Manipur
  "IN-TR": "16", // Tripura
  "IN-ML": "17", // Meghalaya
  "IN-MZ": "15", // Mizoram
  "IN-NL": "13", // Nagaland
  "IN-AR": "12", // Arunachal Pradesh
  "IN-SK": "11", // Sikkim
  "IN-DD": "25", // Daman & Diu
  "IN-DN": "26", // Dadra & Nagar Haveli
  "IN-LD": "31", // Lakshadweep
  "IN-AN": "35", // Andaman & Nicobar
  "IN-LA": "38", // Ladakh
};
```

---

## 8. Product & Inventory Data

### Syncing Products and Variants

```typescript
// lib/shopify/products.ts

const PRODUCTS_QUERY = `
  query GetProducts($cursor: String) {
    products(first: 250, after: $cursor) {
      pageInfo {
        hasNextPage
        endCursor
      }
      edges {
        node {
          id
          legacyResourceId
          title
          vendor
          productType
          status
          variants(first: 100) {
            edges {
              node {
                id
                legacyResourceId
                sku
                barcode
                price
                compareAtPrice
                inventoryQuantity
                taxable
                inventoryManagement
                weight
                weightUnit
                inventoryItem {
                  id
                  unitCost { amount currencyCode }  # Cost of goods (Shopify Plus only)
                  tracked
                }
              }
            }
          }
        }
      }
    }
  }
`;

export async function syncProducts(shopDomain: string): Promise<number> {
  let cursor: string | null = null;
  let count = 0;

  do {
    const data = await shopifyGraphQL(shopDomain, PRODUCTS_QUERY, { cursor });
    const { edges, pageInfo } = data.products;

    for (const edge of edges) {
      const product = edge.node;
      // Store in your products table
      // For COGS: store inventoryItem.unitCost if available (Shopify Plus)
      // For non-Plus: require merchants to provide COGS separately
      await upsertProduct(shopDomain, product);
      count++;
    }

    cursor = pageInfo.hasNextPage ? pageInfo.endCursor : null;
  } while (cursor !== null);

  return count;
}

async function upsertProduct(shopDomain: string, product: any) {
  // Implementation: store product + variants in your DB
  // Key fields for accounting:
  // - sku: for line item → product matching
  // - unitCost: for COGS calculation
  // - taxable: to verify tax was correctly applied
  console.log(`Upserted product ${product.id} for ${shopDomain}`);
}
```

### Inventory Levels

```typescript
const INVENTORY_QUERY = `
  query GetInventoryLevels($cursor: String) {
    inventoryItems(first: 250, after: $cursor) {
      pageInfo {
        hasNextPage
        endCursor
      }
      edges {
        node {
          id
          sku
          tracked
          unitCost { amount currencyCode }
          inventoryLevels(first: 10) {
            edges {
              node {
                id
                available
                location {
                  id
                  name
                  address { city province country }
                }
              }
            }
          }
        }
      }
    }
  }
`;

export async function syncInventoryLevels(shopDomain: string): Promise<void> {
  let cursor: string | null = null;

  do {
    const data = await shopifyGraphQL(shopDomain, INVENTORY_QUERY, { cursor });
    const { edges, pageInfo } = data.inventoryItems;

    for (const edge of edges) {
      const item = edge.node;
      // Store inventory per SKU per location
      // Used for: stock valuation (qty × cost), low-stock alerts in accounting app
      for (const levelEdge of item.inventoryLevels.edges) {
        const level = levelEdge.node;
        // Upsert: inventory_item_id, location_id, available_qty, unit_cost
        console.log(`SKU ${item.sku} at ${level.location.name}: ${level.available} units`);
      }
    }

    cursor = pageInfo.hasNextPage ? pageInfo.endCursor : null;
  } while (cursor !== null);
}
```

---

## 9. Webhook Implementation

### Webhook Dispatcher (Remix Route)

```typescript
// app/routes/webhooks.tsx
import { authenticate } from "~/shopify.server";
import type { ActionFunctionArgs } from "@remix-run/node";
import { handleOrderCreate } from "~/lib/shopify/webhooks/orders-create";
import { handleOrderPaid } from "~/lib/shopify/webhooks/orders-paid";
import { handleOrderUpdated } from "~/lib/shopify/webhooks/orders-updated";
import { handleRefundCreate } from "~/lib/shopify/webhooks/refunds-create";
import { handleDisputeCreate } from "~/lib/shopify/webhooks/disputes-create";
import { handleAppUninstalled } from "~/lib/shopify/webhooks/app-uninstalled";
import { handleBulkOperationFinish } from "~/lib/shopify/webhooks/bulk-finish";

export const action = async ({ request }: ActionFunctionArgs) => {
  const { topic, shop, session, payload, webhookId } =
    await authenticate.webhook(request);

  // The @shopify/shopify-app-remix adapter handles HMAC verification automatically.
  // If HMAC fails, it throws before reaching this code.

  // Idempotency check using webhookId
  const existing = await prisma.webhookEvent.findUnique({
    where: { shopifyEventId: webhookId },
  });
  if (existing && existing.status === "processed") {
    return new Response("Already processed", { status: 200 });
  }

  // Log the event
  const eventRecord = await prisma.webhookEvent.upsert({
    where: { shopifyEventId: webhookId ?? `${topic}-${Date.now()}` },
    create: {
      merchantId: (await prisma.merchant.findUnique({ where: { shopDomain: shop } }))?.id ?? "unknown",
      topic,
      shopifyEventId: webhookId,
      webhookId,
      status: "pending",
      payload,
    },
    update: {},
  });

  // Dispatch to appropriate handler (async — return 200 immediately)
  setImmediate(async () => {
    try {
      switch (topic) {
        case "ORDERS_CREATE":
          await handleOrderCreate(shop, payload);
          break;
        case "ORDERS_PAID":
          await handleOrderPaid(shop, payload);
          break;
        case "ORDERS_UPDATED":
          await handleOrderUpdated(shop, payload);
          break;
        case "REFUNDS_CREATE":
          await handleRefundCreate(shop, payload);
          break;
        case "DISPUTES_CREATE":
          await handleDisputeCreate(shop, payload);
          break;
        case "APP_UNINSTALLED":
          await handleAppUninstalled(shop, payload);
          break;
        case "BULK_OPERATIONS_FINISH":
          await handleBulkOperationFinish(shop, payload);
          break;
        default:
          console.warn(`Unhandled webhook topic: ${topic}`);
      }

      await prisma.webhookEvent.update({
        where: { id: eventRecord.id },
        data: { status: "processed", processedAt: new Date() },
      });
    } catch (error: any) {
      await prisma.webhookEvent.update({
        where: { id: eventRecord.id },
        data: { status: "failed", error: error.message },
      });
      console.error(`Webhook processing failed for ${topic}:`, error);
    }
  });

  // Return 200 immediately — Shopify requires response within 5 seconds
  return new Response("OK", { status: 200 });
};
```

### HMAC Verification (Manual Implementation)

If not using the Remix adapter, implement HMAC verification yourself:

```typescript
// lib/shopify/webhooks/verify-hmac.ts
import crypto from "crypto";

export function verifyShopifyWebhook(
  rawBody: Buffer,
  hmacHeader: string,
  clientSecret: string
): boolean {
  if (!hmacHeader) return false;

  const computed = crypto
    .createHmac("sha256", clientSecret)
    .update(rawBody) // MUST be raw Buffer — before JSON parsing
    .digest("base64");

  try {
    return crypto.timingSafeEqual(
      Buffer.from(hmacHeader, "base64"),
      Buffer.from(computed, "base64")
    );
  } catch {
    return false; // Buffer lengths differ — invalid
  }
}

// Express middleware example (if not using Remix)
export function webhookMiddleware(
  req: express.Request,
  res: express.Response,
  next: express.NextFunction
) {
  // CRITICAL: Use express.raw() before this middleware, NOT express.json()
  const hmac = req.get("X-Shopify-Hmac-Sha256");
  if (!hmac) return res.status(401).send("Missing HMAC");

  const isValid = verifyShopifyWebhook(
    req.body, // Must be raw Buffer
    hmac,
    process.env.SHOPIFY_CLIENT_SECRET!
  );

  if (!isValid) return res.status(401).send("HMAC verification failed");

  next();
}
```

### Order Webhook Handlers

```typescript
// lib/shopify/webhooks/orders-create.ts
import { upsertOrder, fetchAndStoreTransactions } from "../orders";

export async function handleOrderCreate(
  shopDomain: string,
  payload: any
): Promise<void> {
  // Webhook payload is the REST order object format
  // Convert to GraphQL format by fetching from API for consistency
  const orderGid = `gid://shopify/Order/${payload.id}`;

  // Fetch full order via GraphQL (more complete than webhook payload)
  const data = await shopifyGraphQL(shopDomain, SINGLE_ORDER_QUERY, {
    orderId: orderGid,
  });

  await upsertOrder(shopDomain, data.order);

  // Generate accrual accounting entry if financial_status is "paid"
  if (payload.financial_status === "paid") {
    await generateRevenueEntry(shopDomain, orderGid);
  }
}

const SINGLE_ORDER_QUERY = `
  query GetSingleOrder($orderId: ID!) {
    order(id: $orderId) {
      id
      name
      legacyResourceId
      createdAt
      updatedAt
      processedAt
      displayFinancialStatus
      displayFulfillmentStatus
      subtotalPriceSet { shopMoney { amount currencyCode } presentmentMoney { amount currencyCode } }
      totalPriceSet { shopMoney { amount currencyCode } presentmentMoney { amount currencyCode } }
      totalTaxSet { shopMoney { amount currencyCode } }
      totalShippingPriceSet { shopMoney { amount currencyCode } }
      totalDiscountsSet { shopMoney { amount currencyCode } }
      currentTotalPriceSet { shopMoney { amount currencyCode } }
      taxesIncluded
      taxLines { title rate priceSet { shopMoney { amount currencyCode } } channelLiable }
      paymentGatewayNames
      lineItems(first: 100) {
        edges {
          node {
            id name quantity sku taxable
            variantId: variant { id }
            product { id }
            vendor
            originalUnitPriceSet { shopMoney { amount currencyCode } }
            totalDiscountSet { shopMoney { amount currencyCode } }
            taxLines { title rate priceSet { shopMoney { amount currencyCode } } channelLiable }
          }
        }
      }
      transactions(first: 20) {
        id legacyResourceId kind status
        amountSet { shopMoney { amount currencyCode } }
        gateway processedAt
        parentTransaction { legacyResourceId }
      }
      customer { id legacyResourceId email }
      refunds(first: 10) { id createdAt note }
    }
  }
`;

// lib/shopify/webhooks/app-uninstalled.ts
export async function handleAppUninstalled(
  shopDomain: string,
  payload: any
): Promise<void> {
  await prisma.merchant.update({
    where: { shopDomain },
    data: { uninstalledAt: new Date() },
  });

  // Cancel active subscription
  // Stop all sync queues for this shop
  // Schedule data deletion job (runs after 48h when shop/redact arrives)
  console.log(`App uninstalled from ${shopDomain}`);
}
```

### Registering Webhooks Programmatically (GraphQL)

```typescript
// lib/shopify/webhooks/register.ts
// Use this if you need to register additional webhooks beyond what's in shopify.app.toml

const REGISTER_WEBHOOK_MUTATION = `
  mutation webhookSubscriptionCreate($topic: WebhookSubscriptionTopic!, $url: URL!) {
    webhookSubscriptionCreate(
      topic: $topic
      webhookSubscription: {
        format: JSON
        callbackUrl: $url
      }
    ) {
      webhookSubscription {
        id
        topic
        endpoint { __typename }
      }
      userErrors { field message }
    }
  }
`;

export async function registerWebhook(
  shopDomain: string,
  topic: string,
  url: string
): Promise<void> {
  const data = await shopifyGraphQL(shopDomain, REGISTER_WEBHOOK_MUTATION, {
    topic,
    url,
  });

  if (data.webhookSubscriptionCreate.userErrors.length > 0) {
    throw new Error(
      `Failed to register webhook: ${JSON.stringify(data.webhookSubscriptionCreate.userErrors)}`
    );
  }
}
```

---

## 10. GraphQL Bulk Operations

### When to Use Bulk Operations

| Scenario | Use |
|----------|-----|
| Initial historical sync (all orders since store creation) | Bulk Operations |
| Importing 1 year+ of order history | Bulk Operations |
| Exporting all products/inventory | Bulk Operations |
| Daily incremental sync (new/updated orders) | Paginated GraphQL queries |
| Real-time single order | Webhook → single GraphQL fetch |

Bulk operations run asynchronously, bypass per-request rate limits, and return JSONL files downloaded from a URL.

### Complete Bulk Operation Workflow

```typescript
// lib/shopify/bulk-operations.ts
import https from "https";
import fs from "fs";
import readline from "readline";
import path from "path";
import os from "os";

const BULK_OPERATION_MUTATION = `
  mutation bulkOperationRunQuery($query: String!) {
    bulkOperationRunQuery(query: $query) {
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
`;

const BULK_OPERATION_STATUS_QUERY = `
  query GetBulkOperation($id: ID!) {
    node(id: $id) {
      ... on BulkOperation {
        id
        status
        errorCode
        objectCount
        fileSize
        url
        partialDataUrl
        createdAt
        completedAt
      }
    }
  }
`;

export type BulkOperationStatus =
  | "CREATED"
  | "RUNNING"
  | "COMPLETED"
  | "FAILED"
  | "CANCELED"
  | "CANCELING"
  | "EXPIRED";

// Step 1: Start a bulk operation
export async function startBulkOperation(
  shopDomain: string,
  innerQuery: string
): Promise<string> {
  // innerQuery is the GraphQL query INSIDE the bulk operation
  // NOTE: Do NOT include 'first:' args — they are ignored in bulk mode
  const data = await shopifyGraphQL(shopDomain, BULK_OPERATION_MUTATION, {
    query: innerQuery,
  });

  const { bulkOperation, userErrors } = data.bulkOperationRunQuery;

  if (userErrors.length > 0) {
    throw new Error(`Bulk operation failed: ${JSON.stringify(userErrors)}`);
  }

  console.log(`Bulk operation started: ${bulkOperation.id}`);
  return bulkOperation.id;
}

// Step 2: Poll until completion
export async function pollBulkOperation(
  shopDomain: string,
  operationId: string,
  pollIntervalMs = 10000
): Promise<{ url: string; objectCount: number }> {
  while (true) {
    const data = await shopifyGraphQL(shopDomain, BULK_OPERATION_STATUS_QUERY, {
      id: operationId,
    });

    const op = data.node;
    const status: BulkOperationStatus = op.status;

    console.log(`Bulk operation ${operationId}: ${status} (${op.objectCount ?? 0} objects)`);

    if (status === "COMPLETED") {
      if (!op.url) throw new Error("Bulk operation completed but no URL provided");
      return { url: op.url, objectCount: parseInt(op.objectCount ?? "0") };
    }

    if (status === "FAILED") {
      throw new Error(`Bulk operation failed: ${op.errorCode}`);
    }

    if (status === "CANCELED" || status === "EXPIRED") {
      throw new Error(`Bulk operation ended with status: ${status}`);
    }

    // Still running — wait and poll again
    await new Promise((resolve) => setTimeout(resolve, pollIntervalMs));
  }
}

// Step 3: Download and process JSONL file
export async function downloadAndProcessBulkResult(
  url: string,
  onRecord: (record: any) => Promise<void>
): Promise<number> {
  // Download to temp file (bulk results can be gigabytes)
  const tmpFile = path.join(os.tmpdir(), `shopify-bulk-${Date.now()}.jsonl`);

  await downloadFile(url, tmpFile);

  // Process line by line
  let count = 0;
  const rl = readline.createInterface({
    input: fs.createReadStream(tmpFile),
    crlfDelay: Infinity,
  });

  // Build parent-child map for nested objects
  // JSONL format: child objects have __parentId linking to parent
  const parentMap = new Map<string, any>(); // id → record
  const allRecords: any[] = [];

  for await (const line of rl) {
    if (!line.trim()) continue;
    const record = JSON.parse(line);
    allRecords.push(record);
    if (record.id) {
      parentMap.set(record.id, record);
    }
    count++;
  }

  rl.close();
  fs.unlinkSync(tmpFile); // Clean up temp file

  // Reconstruct parent-child relationships
  // Records without __parentId are top-level (orders)
  // Records with __parentId are children (line items, transactions)
  const topLevel = allRecords.filter((r) => !r.__parentId);
  const childrenByParent = new Map<string, any[]>();

  for (const record of allRecords) {
    if (record.__parentId) {
      if (!childrenByParent.has(record.__parentId)) {
        childrenByParent.set(record.__parentId, []);
      }
      childrenByParent.get(record.__parentId)!.push(record);
    }
  }

  // Attach children to parents and call onRecord for each top-level item
  for (const parent of topLevel) {
    const children = childrenByParent.get(parent.id) ?? [];
    // Group children by type (line items vs transactions vs refunds)
    parent._children = children;
    await onRecord(parent);
  }

  return count;
}

function downloadFile(url: string, destPath: string): Promise<void> {
  return new Promise((resolve, reject) => {
    const file = fs.createWriteStream(destPath);
    https
      .get(url, (response) => {
        response.pipe(file);
        file.on("finish", () => {
          file.close();
          resolve();
        });
      })
      .on("error", (err) => {
        fs.unlinkSync(destPath);
        reject(err);
      });
  });
}

// Complete historical order sync using bulk operations
export async function runHistoricalOrderSync(
  shopDomain: string,
  sinceDate?: Date
): Promise<void> {
  const dateFilter = sinceDate
    ? `query: "created_at:>='${sinceDate.toISOString()}'"`
    : "";

  // NOTE: Bulk operations have constraints:
  // - Max 5 connections per query
  // - Max 2 levels of nesting for connections
  // - 'first:' arg on connections is ignored
  const innerQuery = `
    {
      orders(${dateFilter}) {
        edges {
          node {
            id
            name
            legacyResourceId
            createdAt
            updatedAt
            processedAt
            displayFinancialStatus
            displayFulfillmentStatus
            taxesIncluded
            subtotalPriceSet { shopMoney { amount currencyCode } presentmentMoney { amount currencyCode } }
            totalPriceSet { shopMoney { amount currencyCode } presentmentMoney { amount currencyCode } }
            totalTaxSet { shopMoney { amount currencyCode } }
            totalShippingPriceSet { shopMoney { amount currencyCode } }
            totalDiscountsSet { shopMoney { amount currencyCode } }
            currentTotalPriceSet { shopMoney { amount currencyCode } }
            taxLines { title rate priceSet { shopMoney { amount currencyCode } } channelLiable }
            paymentGatewayNames
            lineItems {
              edges {
                node {
                  id
                  name
                  quantity
                  sku
                  taxable
                  originalUnitPriceSet { shopMoney { amount currencyCode } }
                  totalDiscountSet { shopMoney { amount currencyCode } }
                  taxLines { title rate priceSet { shopMoney { amount currencyCode } } channelLiable }
                }
              }
            }
            transactions {
              id
              legacyResourceId
              kind
              status
              amountSet { shopMoney { amount currencyCode } }
              gateway
              processedAt
            }
          }
        }
      }
    }
  `;

  const operationId = await startBulkOperation(shopDomain, innerQuery);

  // Poll for completion (or listen to bulk_operations/finish webhook)
  const { url, objectCount } = await pollBulkOperation(shopDomain, operationId);

  console.log(`Bulk operation complete: ${objectCount} objects. Downloading...`);

  let processedOrders = 0;
  await downloadAndProcessBulkResult(url, async (orderNode) => {
    await upsertOrder(shopDomain, orderNode);
    processedOrders++;
    if (processedOrders % 100 === 0) {
      console.log(`Processed ${processedOrders} orders for ${shopDomain}`);
    }
  });

  // Update merchant sync status
  await prisma.merchant.update({
    where: { shopDomain },
    data: { lastSyncAt: new Date(), syncStatus: "complete" },
  });

  console.log(`Historical sync complete for ${shopDomain}: ${processedOrders} orders`);
}
```

### Handling the `bulk_operations/finish` Webhook

```typescript
// lib/shopify/webhooks/bulk-finish.ts
export async function handleBulkOperationFinish(
  shopDomain: string,
  payload: any
): Promise<void> {
  const { admin_graphql_api_id: operationId, status, url } = payload;

  if (status !== "completed") {
    console.error(`Bulk operation ${operationId} finished with status: ${status}`);
    return;
  }

  // Look up what this bulk operation was for
  // (You should store the operation ID when you start it, tagging it with type)
  // Then process accordingly
  console.log(`Bulk operation ${operationId} completed. Download URL: ${url}`);
}
```

---

## 11. Billing API (Monetization)

### Subscription Plans for an Accounting App

```typescript
// lib/shopify/billing.ts

export interface Plan {
  name: string;
  price: number;
  interval: "EVERY_30_DAYS" | "ANNUAL";
  trialDays: number;
  features: string[];
}

export const PLANS: Record<string, Plan> = {
  starter: {
    name: "Starter Plan",
    price: 9.99,
    interval: "EVERY_30_DAYS",
    trialDays: 14,
    features: ["Up to 500 orders/month", "Basic financial reports", "GST summary"],
  },
  pro: {
    name: "Pro Plan",
    price: 29.99,
    interval: "EVERY_30_DAYS",
    trialDays: 14,
    features: [
      "Unlimited orders",
      "Payout reconciliation",
      "CGST/SGST/IGST split",
      "CSV export",
      "Priority support",
    ],
  },
  enterprise: {
    name: "Enterprise Plan",
    price: 99.99,
    interval: "EVERY_30_DAYS",
    trialDays: 14,
    features: [
      "All Pro features",
      "Multi-store",
      "Tally/Zoho integration",
      "Custom HSN mapping",
      "Dedicated support",
    ],
  },
};

const CREATE_SUBSCRIPTION_MUTATION = `
  mutation AppSubscriptionCreate(
    $name: String!
    $returnUrl: URL!
    $test: Boolean
    $trialDays: Int
    $lineItems: [AppSubscriptionLineItemInput!]!
  ) {
    appSubscriptionCreate(
      name: $name
      returnUrl: $returnUrl
      test: $test
      trialDays: $trialDays
      lineItems: $lineItems
    ) {
      appSubscription {
        id
        status
        currentPeriodEnd
      }
      confirmationUrl
      userErrors {
        field
        message
      }
    }
  }
`;

// Create a subscription and return the Shopify-hosted confirmation URL
export async function createSubscription(
  shopDomain: string,
  planKey: keyof typeof PLANS,
  returnUrl: string,
  isTest = false
): Promise<{ confirmationUrl: string; subscriptionId: string }> {
  const plan = PLANS[planKey];
  if (!plan) throw new Error(`Unknown plan: ${planKey}`);

  const data = await shopifyGraphQL(shopDomain, CREATE_SUBSCRIPTION_MUTATION, {
    name: plan.name,
    returnUrl,
    test: isTest,
    trialDays: plan.trialDays,
    lineItems: [
      {
        plan: {
          appRecurringPricingDetails: {
            price: { amount: plan.price, currencyCode: "USD" },
            interval: plan.interval,
          },
        },
      },
    ],
  });

  const { appSubscription, confirmationUrl, userErrors } =
    data.appSubscriptionCreate;

  if (userErrors.length > 0) {
    throw new Error(`Subscription creation failed: ${JSON.stringify(userErrors)}`);
  }

  // Store pending subscription
  await prisma.merchant.update({
    where: { shopDomain },
    data: {
      subscriptionId: appSubscription.id,
      subscriptionStatus: "pending",
    },
  });

  return {
    confirmationUrl,
    subscriptionId: appSubscription.id,
  };
}

// Usage-based billing (for pay-per-sync model)
const CREATE_USAGE_RECORD_MUTATION = `
  mutation AppUsageRecordCreate(
    $subscriptionLineItemId: ID!
    $price: MoneyInput!
    $description: String!
    $idempotencyKey: String
  ) {
    appUsageRecordCreate(
      subscriptionLineItemId: $subscriptionLineItemId
      price: $price
      description: $description
      idempotencyKey: $idempotencyKey
    ) {
      appUsageRecord { id createdAt }
      userErrors { field message }
    }
  }
`;

export async function recordUsage(
  shopDomain: string,
  subscriptionLineItemId: string,
  description: string,
  amountUSD: number,
  idempotencyKey: string
): Promise<void> {
  const data = await shopifyGraphQL(shopDomain, CREATE_USAGE_RECORD_MUTATION, {
    subscriptionLineItemId,
    price: { amount: amountUSD, currencyCode: "USD" },
    description,
    idempotencyKey,
  });

  if (data.appUsageRecordCreate.userErrors.length > 0) {
    throw new Error(
      `Usage record failed: ${JSON.stringify(data.appUsageRecordCreate.userErrors)}`
    );
  }
}

// Check and handle subscription status from webhook (APP_SUBSCRIPTIONS_UPDATE)
export async function handleSubscriptionUpdate(
  shopDomain: string,
  payload: any
): Promise<void> {
  const { app_subscription } = payload;
  const status = app_subscription.status.toLowerCase();

  await prisma.merchant.update({
    where: { shopDomain },
    data: {
      subscriptionId: `gid://shopify/AppSubscription/${app_subscription.id}`,
      subscriptionStatus: status,
    },
  });

  if (status === "cancelled" || status === "declined" || status === "expired") {
    // Downgrade merchant to free tier or block access
    console.log(`Subscription ${status} for ${shopDomain} — restricting access`);
  }

  if (status === "frozen") {
    // Merchant's Shopify plan was downgraded or they have unpaid bills
    // Pause syncing — do not lose data
    console.log(`Subscription frozen for ${shopDomain}`);
  }
}
```

### Billing Route (Remix)

```typescript
// app/routes/app.billing.tsx
import { authenticate } from "~/shopify.server";
import { createSubscription, PLANS } from "~/lib/shopify/billing";
import type { LoaderFunctionArgs, ActionFunctionArgs } from "@remix-run/node";

export const loader = async ({ request }: LoaderFunctionArgs) => {
  const { session } = await authenticate.admin(request);
  return json({ plans: PLANS });
};

export const action = async ({ request }: ActionFunctionArgs) => {
  const { session } = await authenticate.admin(request);
  const formData = await request.formData();
  const planKey = formData.get("plan") as string;

  const returnUrl = `${process.env.SHOPIFY_APP_URL}/auth/callback?shop=${session.shop}`;

  const { confirmationUrl } = await createSubscription(
    session.shop,
    planKey as any,
    returnUrl,
    process.env.NODE_ENV !== "production"
  );

  // Redirect merchant to Shopify billing consent page
  return redirect(confirmationUrl);
};
```

---

## 12. Rate Limit Management

### GraphQL Rate Limiter with Token Bucket

```typescript
// lib/shopify/rate-limiter.ts

interface ThrottleStatus {
  maximumAvailable: number;
  currentlyAvailable: number;
  restoreRate: number;
}

class GraphQLRateLimiter {
  private available: number = 1000; // Conservative starting point
  private restoreRate: number = 50;
  private lastUpdated: number = Date.now();

  // Update bucket state from response extensions
  update(throttleStatus: ThrottleStatus): void {
    this.available = throttleStatus.currentlyAvailable;
    this.restoreRate = throttleStatus.restoreRate;
    this.lastUpdated = Date.now();
  }

  // Estimate current availability (accounting for time elapsed since last update)
  currentEstimate(): number {
    const elapsed = (Date.now() - this.lastUpdated) / 1000;
    return Math.min(1000, this.available + elapsed * this.restoreRate);
  }

  // Wait if needed before a query estimated to cost N points
  async waitForCapacity(estimatedCost: number): Promise<void> {
    const current = this.currentEstimate();
    if (current < estimatedCost) {
      const pointsNeeded = estimatedCost - current;
      const waitMs = Math.ceil((pointsNeeded / this.restoreRate) * 1000) + 100; // +100ms buffer
      console.log(`Rate limiter: waiting ${waitMs}ms for ${pointsNeeded} points`);
      await new Promise((resolve) => setTimeout(resolve, waitMs));
    }
  }
}

// Per-shop rate limiters
const limiters = new Map<string, GraphQLRateLimiter>();

export function getRateLimiter(shopDomain: string): GraphQLRateLimiter {
  if (!limiters.has(shopDomain)) {
    limiters.set(shopDomain, new GraphQLRateLimiter());
  }
  return limiters.get(shopDomain)!;
}

// Enhanced GraphQL client with rate limiting
export async function shopifyGraphQLWithRateLimit(
  shopDomain: string,
  query: string,
  variables?: Record<string, unknown>,
  maxRetries = 3
): Promise<any> {
  const limiter = getRateLimiter(shopDomain);
  const accessToken = await getAccessToken(shopDomain);

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    // Proactive throttle: wait if we estimate low availability
    await limiter.waitForCapacity(100); // Conservative cost estimate

    const response = await fetch(
      `https://${shopDomain}/admin/api/2026-01/graphql.json`,
      {
        method: "POST",
        headers: {
          "X-Shopify-Access-Token": accessToken,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ query, variables }),
      }
    );

    // Handle rate limit (429)
    if (response.status === 429) {
      const retryAfter = parseInt(response.headers.get("Retry-After") ?? "2");
      console.warn(`Rate limited on ${shopDomain}. Waiting ${retryAfter}s...`);
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
      continue;
    }

    if (!response.ok) {
      throw new Error(`GraphQL request failed: ${response.status}`);
    }

    const result = await response.json();

    // Update rate limiter from response extensions
    if (result.extensions?.cost?.throttleStatus) {
      limiter.update(result.extensions.cost.throttleStatus);
    }

    // Handle GraphQL-level throttle errors
    if (result.errors?.some((e: any) => e.extensions?.code === "THROTTLED")) {
      const delay = 2000 * (attempt + 1);
      console.warn(`GraphQL throttled on ${shopDomain}. Waiting ${delay}ms...`);
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }

    if (result.errors?.length) {
      throw new Error(`GraphQL errors: ${JSON.stringify(result.errors)}`);
    }

    return result.data;
  }

  throw new Error(`Max retries exceeded for ${shopDomain}`);
}
```

### BullMQ Queue for Async Sync Jobs

```typescript
// app/queues/order-sync.queue.ts
import { Queue, Worker, QueueEvents } from "bullmq";
import IORedis from "ioredis";
import { syncOrdersForShop } from "~/lib/shopify/orders";
import { syncPayouts } from "~/lib/shopify/payouts";

const connection = new IORedis(process.env.REDIS_URL!, {
  maxRetriesPerRequest: null,
});

export const orderSyncQueue = new Queue("shopify-order-sync", { connection });
export const payoutSyncQueue = new Queue("shopify-payout-sync", { connection });

// Worker: incremental order sync
export const orderSyncWorker = new Worker(
  "shopify-order-sync",
  async (job) => {
    const { shopDomain, sinceDate } = job.data;
    const since = sinceDate ? new Date(sinceDate) : undefined;
    const count = await syncOrdersForShop(shopDomain, { sinceDate: since });
    return { ordersProcessed: count };
  },
  {
    connection,
    concurrency: 5,          // Process up to 5 shops simultaneously
    limiter: {
      max: 10,               // 10 jobs per second max
      duration: 1000,
    },
  }
);

orderSyncWorker.on("completed", (job, result) => {
  console.log(`Order sync complete for ${job.data.shopDomain}: ${result.ordersProcessed} orders`);
});

orderSyncWorker.on("failed", (job, err) => {
  console.error(`Order sync failed for ${job?.data.shopDomain}:`, err);
});

// Schedule incremental syncs every 6 hours per merchant
export async function scheduleIncrementalSync(shopDomain: string): Promise<void> {
  await orderSyncQueue.add(
    "incremental-sync",
    {
      shopDomain,
      sinceDate: new Date(Date.now() - 6 * 60 * 60 * 1000).toISOString(), // 6h ago
    },
    {
      repeat: { every: 6 * 60 * 60 * 1000 }, // Every 6 hours
      jobId: `sync-${shopDomain}`,            // Prevents duplicates
      removeOnComplete: 100,
      removeOnFail: 50,
    }
  );
}
```

---

## 13. App Bridge & Embedded UI

### Setup

```html
<!-- public/index.html or _document.tsx — MUST be first script -->
<script src="https://cdn.shopify.com/shopifycloud/app-bridge.js"></script>
```

### Remix + Polaris Layout

```typescript
// app/routes/app.tsx (root layout for embedded app)
import { AppProvider } from "@shopify/shopify-app-remix/react";
import { NavMenu } from "@shopify/app-bridge-react";
import polarisStyles from "@shopify/polaris/build/esm/styles.css";
import { authenticate } from "~/shopify.server";
import type { LoaderFunctionArgs } from "@remix-run/node";

export const links = () => [{ rel: "stylesheet", href: polarisStyles }];

export const loader = async ({ request }: LoaderFunctionArgs) => {
  await authenticate.admin(request);
  return json({ apiKey: process.env.SHOPIFY_API_KEY });
};

export default function App() {
  const { apiKey } = useLoaderData<typeof loader>();

  return (
    <AppProvider isEmbeddedApp apiKey={apiKey}>
      {/* Navigation within embedded app */}
      <NavMenu>
        <a href="/app" rel="home">Dashboard</a>
        <a href="/app/orders">Orders</a>
        <a href="/app/payouts">Payouts</a>
        <a href="/app/reports">Reports</a>
        <a href="/app/settings">Settings</a>
      </NavMenu>
      <Outlet />
    </AppProvider>
  );
}
```

### Accounting Dashboard Page (Polaris)

```typescript
// app/routes/app._index.tsx
import {
  Page,
  Layout,
  Card,
  DataTable,
  Badge,
  Text,
  DatePicker,
  Filters,
  Button,
  Banner,
} from "@shopify/polaris";

export default function Dashboard() {
  const { metrics, recentOrders } = useLoaderData<typeof loader>();

  return (
    <Page
      title="Accounting Dashboard"
      subtitle="Financial overview for your Shopify store"
      primaryAction={
        <Button variant="primary" url="/app/reports/export">
          Export to CSV
        </Button>
      }
    >
      <Layout>
        {/* KPI Summary Cards */}
        <Layout.Section>
          <InlineGrid columns={4} gap="400">
            <Card>
              <Text as="h3" variant="headingSm">Total Revenue</Text>
              <Text as="p" variant="headingXl">₹{metrics.totalRevenue.toLocaleString("en-IN")}</Text>
              <Badge tone="success">+12% vs last month</Badge>
            </Card>
            <Card>
              <Text as="h3" variant="headingSm">Tax Collected</Text>
              <Text as="p" variant="headingXl">₹{metrics.totalTax.toLocaleString("en-IN")}</Text>
            </Card>
            <Card>
              <Text as="h3" variant="headingSm">Refunds</Text>
              <Text as="p" variant="headingXl">₹{metrics.totalRefunds.toLocaleString("en-IN")}</Text>
            </Card>
            <Card>
              <Text as="h3" variant="headingSm">Net Payout</Text>
              <Text as="p" variant="headingXl">₹{metrics.netPayout.toLocaleString("en-IN")}</Text>
            </Card>
          </InlineGrid>
        </Layout.Section>

        {/* Recent Orders Table */}
        <Layout.Section>
          <Card>
            <DataTable
              columnContentTypes={["text", "text", "text", "numeric", "numeric", "text"]}
              headings={["Order", "Date", "Status", "Amount (₹)", "Tax (₹)", "Gateway"]}
              rows={recentOrders.map((order: any) => [
                order.orderNumber,
                new Date(order.createdAt).toLocaleDateString("en-IN"),
                <Badge key={order.id} tone={order.financialStatus === "PAID" ? "success" : "attention"}>
                  {order.financialStatus}
                </Badge>,
                parseFloat(order.totalPrice).toLocaleString("en-IN", { minimumFractionDigits: 2 }),
                parseFloat(order.totalTax).toLocaleString("en-IN", { minimumFractionDigits: 2 }),
                order.gateway,
              ])}
            />
          </Card>
        </Layout.Section>
      </Layout>
    </Page>
  );
}
```

### Key UI Patterns for Accounting Apps

| Pattern | Polaris Component | Use Case |
|---------|------------------|----------|
| Order list | `ResourceList` + `ResourceItem` | Browsable order history |
| Financial table | `DataTable` | Transactions, payouts |
| Date range filter | `DatePicker` + `Filters` | Report date selection |
| Status indicators | `Badge` | Order status, payout status |
| Metric summary | `Card` + `Text` | KPIs on dashboard |
| Loading state | `SkeletonPage` + `SkeletonBodyText` | Async data loading |
| Error/warning | `Banner` | Reconciliation mismatches |
| Confirmation dialogs | `Modal` | Before bulk actions |
| Empty state | `EmptyState` | No data yet / first sync |

---

## 14. GDPR Compliance

### Three Mandatory Webhooks

All three must be registered via `compliance_topics` in `shopify.app.toml` (cannot use Admin API).

```typescript
// app/routes/webhooks.compliance.tsx
import type { ActionFunctionArgs } from "@remix-run/node";
import crypto from "crypto";

export const action = async ({ request }: ActionFunctionArgs) => {
  // Manual HMAC verification for compliance webhooks
  const rawBody = await request.arrayBuffer();
  const body = Buffer.from(rawBody);
  const hmacHeader = request.headers.get("X-Shopify-Hmac-Sha256");
  const topic = request.headers.get("X-Shopify-Topic");

  // Verify HMAC
  const computed = crypto
    .createHmac("sha256", process.env.SHOPIFY_CLIENT_SECRET!)
    .update(body)
    .digest("base64");

  if (!hmacHeader || !crypto.timingSafeEqual(
    Buffer.from(hmacHeader, "base64"),
    Buffer.from(computed, "base64")
  )) {
    return new Response("Unauthorized", { status: 401 });
  }

  const payload = JSON.parse(body.toString("utf8"));

  // Process asynchronously — return 200 immediately
  setImmediate(async () => {
    switch (topic) {
      case "customers/data_request":
        await handleCustomerDataRequest(payload);
        break;
      case "customers/redact":
        await handleCustomerRedact(payload);
        break;
      case "shop/redact":
        await handleShopRedact(payload);
        break;
    }
  });

  return new Response("OK", { status: 200 });
};

// customers/data_request — Customer wants to see their stored data
async function handleCustomerDataRequest(payload: {
  shop_id: number;
  shop_domain: string;
  customer: { id: number; email: string; phone: string };
  orders_requested: number[];
}): Promise<void> {
  // Find all data you have about this customer
  const customerId = BigInt(payload.customer.id);
  const orders = await prisma.order.findMany({
    where: {
      customerId,
      merchant: { shopDomain: payload.shop_domain },
    },
    select: {
      id: true,
      orderNumber: true,
      createdAt: true,
      totalPrice: true,
    },
  });

  // Log for audit purposes — Shopify does not require you to send this data anywhere
  // The merchant is responsible for providing it to the customer
  console.log(
    `Data request for customer ${payload.customer.email} on ${payload.shop_domain}:`,
    { orderCount: orders.length, orderIds: orders.map((o) => o.id) }
  );

  // In practice: store this in an audit log table and notify the merchant via email
}

// customers/redact — Customer requested deletion
async function handleCustomerRedact(payload: {
  shop_id: number;
  shop_domain: string;
  customer: { id: number; email: string };
  orders_to_redact: number[];
}): Promise<void> {
  const customerId = BigInt(payload.customer.id);
  const shopDomain = payload.shop_domain;

  // Anonymize PII but keep financial records (needed for accounting/audit)
  await prisma.order.updateMany({
    where: {
      customerId,
      merchant: { shopDomain },
    },
    data: {
      customerEmail: null,  // Remove PII
      customerId: null,     // Remove PII linkage
      // Keep: orderNumber, amounts, tax, dates — needed for accounting
    },
  });

  console.log(
    `Redacted customer ${customerId} data on ${shopDomain}. Orders anonymized: ${payload.orders_to_redact.length}`
  );
}

// shop/redact — Fired 48h after app uninstall; delete all shop data
async function handleShopRedact(payload: {
  shop_id: number;
  shop_domain: string;
}): Promise<void> {
  const shopDomain = payload.shop_domain;

  const merchant = await prisma.merchant.findUnique({
    where: { shopDomain },
  });

  if (!merchant) {
    console.log(`shop/redact: No merchant record for ${shopDomain}`);
    return;
  }

  // Delete in dependency order
  await prisma.$transaction([
    prisma.lineItem.deleteMany({
      where: { order: { merchantId: merchant.id } },
    }),
    prisma.orderTransaction.deleteMany({
      where: { order: { merchantId: merchant.id } },
    }),
    prisma.refund.deleteMany({
      where: { order: { merchantId: merchant.id } },
    }),
    prisma.order.deleteMany({ where: { merchantId: merchant.id } }),
    prisma.balanceTransaction.deleteMany({
      where: { payout: { merchantId: merchant.id } },
    }),
    prisma.payout.deleteMany({ where: { merchantId: merchant.id } }),
    prisma.webhookEvent.deleteMany({ where: { merchantId: merchant.id } }),
    prisma.merchant.delete({ where: { id: merchant.id } }),
  ]);

  // Also delete Shopify Session record
  await prisma.session.deleteMany({ where: { shop: shopDomain } });

  console.log(`shop/redact: All data deleted for ${shopDomain}`);
}
```

### Data Retention Policy

| Data Type | Retention | Reason |
|-----------|-----------|--------|
| Order amounts, tax, dates | 7 years | GST/financial audit requirement |
| Customer email/name | Delete on `customers/redact` | GDPR/DPDP compliance |
| Customer ID linkage | Delete on `customers/redact` | GDPR/DPDP compliance |
| Access tokens | Delete on `shop/redact` | No longer needed |
| Webhook event logs | 90 days | Debugging/replay |
| Payout reconciliation records | 7 years | Financial audit |

---

## 15. Data Model Reference

### Complete TypeScript Interfaces

```typescript
// lib/shopify/types.ts

export interface MoneySet {
  shopMoney: Money;
  presentmentMoney: Money;
}

export interface Money {
  amount: string;  // Always a string from Shopify API — parse with parseFloat()
  currencyCode: string;
}

export interface TaxLine {
  title: string;
  rate: number;         // Decimal e.g. 0.09 = 9%
  price: string;        // Shop currency amount as string
  priceSet: MoneySet;
  channelLiable: boolean;
}

export interface LineItem {
  id: string;           // GID
  legacyResourceId?: string;
  name: string;
  quantity: number;
  sku: string | null;
  variant: { id: string } | null;
  product: { id: string } | null;
  vendor: string | null;
  taxable: boolean;
  originalUnitPriceSet: MoneySet;
  discountedUnitPriceSet: MoneySet;
  totalDiscountSet: MoneySet;
  taxLines: TaxLine[];
}

export interface ShippingLine {
  id: string;
  title: string;
  code: string;
  originalPriceSet: MoneySet;
  discountedPriceSet: MoneySet;
  taxLines: TaxLine[];
}

export interface OrderTransaction {
  id: string;           // GID
  legacyResourceId: string;
  kind: "AUTHORIZATION" | "CAPTURE" | "SALE" | "VOID" | "REFUND";
  status: "PENDING" | "FAILURE" | "SUCCESS" | "ERROR";
  amountSet: MoneySet;
  gateway: string;
  processedAt: string;  // ISO datetime
  parentTransaction: { legacyResourceId: string } | null;
}

export interface ShopifyOrder {
  id: string;           // GID: "gid://shopify/Order/123"
  legacyResourceId: string;
  name: string;         // "#1001"
  createdAt: string;
  updatedAt: string;
  processedAt: string | null;
  cancelledAt: string | null;
  displayFinancialStatus:
    | "AUTHORIZED"
    | "PENDING"
    | "PAID"
    | "PARTIALLY_PAID"
    | "REFUNDED"
    | "VOIDED"
    | "PARTIALLY_REFUNDED"
    | "EXPIRED";
  displayFulfillmentStatus:
    | "UNFULFILLED"
    | "PARTIALLY_FULFILLED"
    | "FULFILLED"
    | "RESTOCKED";
  subtotalPriceSet: MoneySet;
  totalPriceSet: MoneySet;
  totalTaxSet: MoneySet;
  totalShippingPriceSet: MoneySet;
  totalDiscountsSet: MoneySet;
  currentTotalPriceSet: MoneySet;
  taxesIncluded: boolean;
  taxLines: TaxLine[];
  lineItems: { edges: Array<{ node: LineItem }> };
  shippingLines: { edges: Array<{ node: ShippingLine }> };
  transactions: OrderTransaction[];
  paymentGatewayNames: string[];
  customer: {
    id: string;
    legacyResourceId: string;
    email: string;
  } | null;
  refunds: Array<{ id: string; createdAt: string; note: string | null }>;
  discountCodes: Array<{ code: string; amount: string; type: string }>;
}

export interface ShopifyPayout {
  id: string;           // GID
  legacyResourceId: string;
  status: "SCHEDULED" | "IN_TRANSIT" | "PAID" | "FAILED" | "CANCELED";
  issuedAt: string;     // ISO datetime
  net: Money;
  gross: Money;
  fee: Money;
  summary: {
    chargesGross: Money;
    chargesFee: Money;
    refundsFeeGross: Money;
    adjustmentsFeeAmount: Money;
    adjustmentsGross: Money;
  };
}

export interface ShopifyBalanceTransaction {
  id: string;           // GID
  type:
    | "CHARGE"
    | "REFUND"
    | "DISPUTE"
    | "RESERVE"
    | "ADJUSTMENT"
    | "CREDIT"
    | "DEBIT"
    | "PAYOUT"
    | "PAYOUT_FAILURE"
    | "PAYOUT_CANCELLATION";
  net: Money;
  fee: Money;
  amount: Money;
  processedAt: string;
  associatedPayout: { id: string; status: string; issuedAt: string } | null;
  sourceOrderTransaction: {
    id: string;
    legacyResourceId: string;
    order: { id: string; name: string };
  } | null;
}

export interface ShopifyRefund {
  id: string;
  createdAt: string;
  note: string | null;
  totalRefundedSet: MoneySet;
  refundLineItems: {
    edges: Array<{
      node: {
        quantity: number;
        restockType: "RETURN" | "CANCEL" | "DECLINE" | "NO_RESTOCK" | "LEGACY_RESTOCK";
        subtotalSet: MoneySet;
        totalTaxSet: MoneySet;
        lineItem: { id: string; name: string; sku: string | null };
      };
    }>;
  };
  orderAdjustments: Array<{
    id: string;
    kind: "SHIPPING_REFUND" | "REFUND_DISCREPANCY";
    reason: string;
    amountSet: MoneySet;
    taxAmountSet: MoneySet;
  }>;
  transactions: OrderTransaction[];
}

export interface ShopifyProduct {
  id: string;
  legacyResourceId: string;
  title: string;
  vendor: string;
  productType: string;
  status: "ACTIVE" | "ARCHIVED" | "DRAFT";
  variants: {
    edges: Array<{
      node: {
        id: string;
        legacyResourceId: string;
        sku: string | null;
        barcode: string | null;
        price: string;
        compareAtPrice: string | null;
        inventoryQuantity: number;
        taxable: boolean;
        inventoryItem: {
          id: string;
          unitCost: Money | null; // Only available on Shopify Plus
          tracked: boolean;
        };
      };
    }>;
  };
}

export interface ShopInfo {
  id: number;
  name: string;
  email: string;
  domain: string;
  myshopify_domain: string;
  currency: string;          // Base/shop currency (INR, SGD, USD)
  timezone: string;
  iana_timezone: string;     // e.g. "Asia/Kolkata"
  country_code: string;      // "IN", "SG"
  country_name: string;
  has_shopify_payments: boolean;
  taxes_included: boolean;
  plan_name: string;
}

// Accounting journal entry shape (your accounting module's input)
export interface JournalEntry {
  date: Date;
  description: string;
  reference: string;         // Shopify GID
  currency: string;
  lines: JournalLine[];
  tags?: Record<string, string>;
}

export interface JournalLine {
  account: string;           // Account name/code
  debit: number;
  credit: number;
  note?: string;
}

// Accounting mapping from a Shopify order
export function orderToJournalEntries(
  order: ShopifyOrder,
  taxSplit: { cgst: number; sgst: number; igst: number }
): JournalEntry[] {
  const currency = order.totalPriceSet.shopMoney.currencyCode;
  const totalPrice = parseFloat(order.totalPriceSet.shopMoney.amount);
  const totalTax = parseFloat(order.totalTaxSet.shopMoney.amount);
  const totalShipping = parseFloat(order.totalShippingPriceSet.shopMoney.amount);
  const totalDiscounts = parseFloat(order.totalDiscountsSet.shopMoney.amount);
  const subtotal = parseFloat(order.subtotalPriceSet.shopMoney.amount);

  const entries: JournalEntry[] = [];

  if (order.displayFinancialStatus === "PAID") {
    // Revenue recognition entry
    entries.push({
      date: new Date(order.createdAt),
      description: `Sale: Order ${order.name}`,
      reference: order.id,
      currency,
      lines: [
        { account: "Accounts Receivable / Shopify Clearing", debit: totalPrice, credit: 0 },
        { account: "Revenue (Sales)", debit: 0, credit: subtotal + totalShipping - totalDiscounts },
        ...(taxSplit.cgst > 0
          ? [{ account: "CGST Payable", debit: 0, credit: taxSplit.cgst }]
          : []),
        ...(taxSplit.sgst > 0
          ? [{ account: "SGST Payable", debit: 0, credit: taxSplit.sgst }]
          : []),
        ...(taxSplit.igst > 0
          ? [{ account: "IGST Payable", debit: 0, credit: taxSplit.igst }]
          : []),
      ],
    });
  }

  return entries;
}
```

---

## 16. Implementation Checklist

### Phase 1: Foundation (Week 1–2)

#### App Setup
- [ ] Create Shopify Partner account and app at partners.shopify.com
- [ ] Initialize Remix project with `@shopify/shopify-app-remix` adapter
- [ ] Configure `shopify.app.toml` with all scopes and webhook topics
- [ ] Set up PostgreSQL database and run `prisma migrate dev`
- [ ] Set up Redis for BullMQ queues
- [ ] Deploy to staging (Fly.io, Railway, or Render)
- [ ] Configure environment variables: `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET`, `SHOPIFY_APP_URL`, `DATABASE_URL`, `REDIS_URL`, `TOKEN_ENCRYPTION_KEY`
- [ ] Register ngrok tunnel for local development

#### OAuth
- [ ] Test OAuth install flow on development store
- [ ] Verify HMAC validation on OAuth callback
- [ ] Verify offline access token is stored encrypted in DB
- [ ] Test `afterAuth` hook fires and creates Merchant record
- [ ] Apply for `read_all_orders` scope approval in Partner Dashboard

### Phase 2: Data Sync (Week 3–4)

#### Orders
- [ ] Implement `syncOrdersForShop()` with cursor pagination
- [ ] Implement `upsertOrder()` — always use `shopMoney`
- [ ] Implement `upsertLineItem()` with tax line storage
- [ ] Test with development store orders in INR
- [ ] Verify `taxes_included` handling
- [ ] Verify `exchangeRate` calculation for multi-currency orders

#### Historical Sync
- [ ] Implement `runHistoricalOrderSync()` with bulk operations
- [ ] Test bulk operation start → poll → download → process
- [ ] Handle `__parentId` JSONL reconstruction for line items
- [ ] Verify bulk operation handles 10k+ orders correctly

#### Webhooks
- [ ] Deploy compliance webhooks (GDPR) — test with Shopify Partner tools
- [ ] Implement `orders/create` handler
- [ ] Implement `orders/paid` handler
- [ ] Implement `orders/updated` handler (idempotent upsert)
- [ ] Implement `refunds/create` handler
- [ ] Implement `app/uninstalled` handler
- [ ] Implement `bulk_operations/finish` handler
- [ ] Verify HMAC signature check on all webhook routes
- [ ] Verify idempotency using `X-Shopify-Webhook-Id`
- [ ] Verify all webhooks return 200 within 5 seconds

### Phase 3: Financial Features (Week 5–6)

#### Transactions
- [ ] Implement `fetchAndStoreTransactions()` for each order
- [ ] Map transaction `kind` to accounting entry type
- [ ] Store `balanceTransactionId` link on order transactions

#### Payouts (Shopify Payments stores only)
- [ ] Implement `syncPayouts()` with GraphQL
- [ ] Implement `syncBalanceTransactionsForPayout()`
- [ ] Verify `source_order_transaction_id` → order transaction linking
- [ ] Implement `reconcilePayout()` — verify sum(net) = payout.amount
- [ ] Build reconciliation report UI (flag mismatches with Banner)

#### Tax
- [ ] Implement `splitIndianTax()` for CGST/SGST/IGST extraction
- [ ] Implement `inferGSTType()` for stores with combined tax
- [ ] Build Indian state code mapping table
- [ ] Implement `taxes_included` extraction logic
- [ ] Add `channel_liable` exclusion logic
- [ ] Test with orders to: Maharashtra (intra-state), Karnataka (inter-state)

### Phase 4: Billing & Compliance (Week 7)

#### Billing
- [ ] Implement `createSubscription()` for all plan tiers
- [ ] Build plan selection UI (Polaris `Card` list)
- [ ] Implement `handleSubscriptionUpdate()` webhook handler
- [ ] Test subscription in development store (set `test: true`)
- [ ] Implement access gate: block non-paying merchants from features
- [ ] Implement free trial logic (14 days, then prompt to upgrade)

#### GDPR
- [ ] Implement `handleCustomerDataRequest()` — log and notify merchant
- [ ] Implement `handleCustomerRedact()` — anonymize PII, keep amounts
- [ ] Implement `handleShopRedact()` — delete all shop data
- [ ] Test all three with Shopify Partner compliance tools
- [ ] Document data retention policy

### Phase 5: UI & Export (Week 8)

#### Embedded Dashboard
- [ ] Set up App Bridge with `@shopify/app-bridge-react`
- [ ] Build Dashboard with Polaris KPI cards
- [ ] Build Orders list with `DataTable` and `Filters`
- [ ] Build Payouts view with reconciliation status
- [ ] Build Tax Summary view (CGST/SGST/IGST breakdown)
- [ ] Add date range filter with `DatePicker`

#### Exports
- [ ] CSV export: orders with tax split
- [ ] CSV export: payout reconciliation
- [ ] GSTR-1 compatible export format (for Indian merchants)

### Phase 6: App Store Submission

- [ ] All scopes in TOML match what you actually use
- [ ] App Bridge latest version is the FIRST script loaded
- [ ] All compliance webhooks tested and passing
- [ ] No 404/500 errors in any embedded routes
- [ ] Screenshots prepared: 1600×900px, 3–6 shots
- [ ] App listing text: describe India + Singapore market focus
- [ ] Privacy policy URL live
- [ ] Support email configured
- [ ] Submit for Shopify review

---

## Appendix A: Environment Variables

```bash
# .env
SHOPIFY_API_KEY=your_api_key
SHOPIFY_API_SECRET=your_api_secret
SHOPIFY_APP_URL=https://yourapp.com

# Database
DATABASE_URL=postgresql://user:pass@host:5432/shopify_accounting

# Redis (for BullMQ)
REDIS_URL=redis://localhost:6379

# Encryption (generate: openssl rand -hex 32)
TOKEN_ENCRYPTION_KEY=64-char-hex-string

# App
NODE_ENV=production
PORT=3000
```

## Appendix B: Key Shopify API Endpoints Reference

```
# GraphQL (use this for all new code — REST is legacy)
POST https://{shop}.myshopify.com/admin/api/2026-01/graphql.json

# REST (legacy — reference only)
GET  https://{shop}.myshopify.com/admin/api/2026-01/shop.json
GET  https://{shop}.myshopify.com/admin/api/2026-01/orders.json
GET  https://{shop}.myshopify.com/admin/api/2026-01/orders/{id}/transactions.json
GET  https://{shop}.myshopify.com/admin/api/2026-01/orders/{id}/refunds.json
GET  https://{shop}.myshopify.com/admin/api/2026-01/shopify_payments/payouts.json
GET  https://{shop}.myshopify.com/admin/api/2026-01/shopify_payments/balance/transactions.json
GET  https://{shop}.myshopify.com/admin/api/2026-01/shopify_payments/disputes.json
POST https://{shop}.myshopify.com/admin/api/2026-01/webhooks.json
GET  https://{shop}.myshopify.com/admin/oauth/access_scopes.json
```

## Appendix C: Critical Rules (Never Violate These)

1. **ALWAYS use `shopMoney` for accounting** — never `presentmentMoney`
2. **ALWAYS verify HMAC** before processing any webhook payload
3. **ALWAYS encrypt access tokens** at rest in your database
4. **ALWAYS use cursor-based pagination** — offset pagination is deprecated
5. **ALWAYS return `200 OK` within 5 seconds** from webhook endpoint — process async
6. **ALWAYS use `crypto.timingSafeEqual`** for HMAC comparisons — never `===`
7. **NEVER use REST API for new code** — it's legacy as of October 2024; use GraphQL
8. **NEVER store customer PII** beyond what is needed — anonymize on `customers/redact`
9. **NEVER bill outside Shopify Billing API** — grounds for App Store removal
10. **NEVER parse `tax_lines` by array index** — always match by `title` field

---

*Sources: [Shopify Dev Docs](https://shopify.dev/docs) | [GraphQL Admin API](https://shopify.dev/docs/api/admin-graphql) | [Bulk Operations Guide](https://shopify.dev/docs/api/usage/bulk-operations/queries) | [Shopify Access Scopes](https://shopify.dev/docs/api/usage/access-scopes) | [Shopify Billing API](https://shopify.dev/docs/apps/billing) | [App Store Requirements](https://shopify.dev/docs/apps/launch/shopify-app-store/app-store-requirements) | [Privacy Compliance](https://shopify.dev/docs/apps/build/compliance/privacy-law-compliance) | [API Rate Limits](https://shopify.dev/docs/api/usage/limits) | [Shopify Webhook HTTPS Docs](https://shopify.dev/docs/apps/build/webhooks/subscribe/https) | [Shopify Payouts REST Reference](https://shopify.dev/docs/api/admin-rest/latest/resources/payouts)*
