# Indian GST & Tax Compliance Engine Skill
## For: Shopify Accounting Platform
## Version: 1.0 | March 2026

> **Scope:** Complete implementation guide for building a GST-compliant tax engine for a Shopify-based accounting platform. Covers GST calculations, invoice generation, GSTR-1/3B return data preparation, e-invoicing with IRP API integration, TCS/TDS handling, and ITC management. All code is TypeScript. Rules reflect GST 2.0 post-rationalization (effective 22 Sep 2025) and 30-day IRN upload limit (effective 1 Apr 2025).

---

## Table of Contents

1. [GST Architecture Overview](#1-gst-architecture-overview)
2. [State/UT Code Master Data](#2-stateut-code-master-data)
3. [Place of Supply Engine](#3-place-of-supply-engine)
4. [Tax Rate Engine](#4-tax-rate-engine)
5. [GSTIN Validation](#5-gstin-validation)
6. [Invoice Generation Engine](#6-invoice-generation-engine)
7. [HSN Code Management](#7-hsn-code-management)
8. [GSTR-1 Data Preparation](#8-gstr-1-data-preparation)
9. [GSTR-3B Data Preparation](#9-gstr-3b-data-preparation)
10. [E-Invoicing Module](#10-e-invoicing-module)
11. [TCS/TDS Handling](#11-tcstds-handling)
12. [Input Tax Credit (ITC) Engine](#12-input-tax-credit-itc-engine)
13. [Financial Year & Period Management](#13-financial-year--period-management)
14. [Complete Data Models](#14-complete-data-models)
15. [Implementation Checklist](#15-implementation-checklist)

---

## 1. GST Architecture Overview

### 1.1 The Engine's Role

The GST engine is the tax brain of the Shopify accounting platform. Every Shopify order that flows into the platform must pass through this engine to:

1. Determine tax jurisdiction (which state's tax applies)
2. Select the correct tax type (CGST+SGST, IGST, or CGST+UTGST)
3. Calculate tax amounts at the correct rate
4. Generate a legally compliant GST invoice (with IRN if e-invoicing applies)
5. Accumulate data for GSTR-1 and GSTR-3B return filing
6. Track TCS credits from marketplace operators (Amazon, Flipkart, etc.)
7. Manage Input Tax Credit eligibility and GSTR-2B matching

### 1.2 Data Flow

```
Shopify Order Webhook (order/created, order/updated, refunds/create)
         │
         ▼
┌─────────────────────────────┐
│  Order Normalization Layer  │  → Extract: line items, shipping address,
└─────────────────────────────┘    billing address, customer, amounts
         │
         ▼
┌─────────────────────────────┐
│  Place of Supply Engine     │  → Determine: seller state vs buyer state
│  (Section 10 IGST Act)      │    Output: PlaceOfSupplyResult
└─────────────────────────────┘    { place_of_supply_code, is_interstate,
         │                           tax_type: CGST_SGST | IGST | CGST_UTGST }
         ▼
┌─────────────────────────────┐
│  Tax Rate Engine            │  → Input: HSN code + PlaceOfSupplyResult
│  (GST Rate Slabs)           │    Output: TaxCalculationResult
└─────────────────────────────┘    { cgst, sgst, igst, cess amounts }
         │
         ▼
┌─────────────────────────────┐
│  Invoice Generation Engine  │  → Build Rule-46-compliant GSTInvoice
│  (Rule 46 CGST Rules)       │    Assign sequential invoice number
└─────────────────────────────┘    Handle B2B / B2C / Credit Note / Bill of Supply
         │
         ▼
┌─────────────────────────────┐
│  E-Invoicing Module         │  → If AATO > ₹5 crore AND B2B/export:
│  (IRP API — INV-1 v1.1)     │    Generate JSON → POST to IRP → Get IRN + QR
└─────────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│  Return Data Accumulator    │  → Classify invoice into GSTR-1 tables
│  (GSTR-1 / GSTR-3B)         │    (B2B Table 4, B2C Large Table 5,
└─────────────────────────────┘     B2C Small Table 7, CDN Table 9)
         │
         ▼
┌─────────────────────────────┐
│  ITC & TCS Reconciliation   │  → Match GSTR-2B purchases
│  Engine                     │    Record TCS from marketplace settlements
└─────────────────────────────┘    Compute ITC reversals (Rule 42/43)
```

### 1.3 Key Decisions Per Transaction

For every Shopify order, the engine must answer:

| Decision | How | Law |
|----------|-----|-----|
| Intra-state or inter-state? | Seller GSTIN state vs delivery address state | Section 10(1)(ca) IGST Act |
| CGST+SGST or IGST or CGST+UTGST? | Based on above + UT check | CGST/IGST/UTGST Acts |
| B2B or B2C? | Does buyer have valid GSTIN? | Rule 46 CGST Rules |
| What GST rate? | HSN code + item value thresholds | CBIC Rate Schedule |
| Does e-invoicing apply? | Is AATO > ₹5 crore AND B2B/export/CDN? | E-Invoice Notification |
| Is invoice within 30-day window? | Invoice date vs today (for e-invoicing) | IRP 30-day rule |
| Credit note or debit note? | Shopify refund → credit note; price revision up → debit note | Section 34 CGST Act |
| TCS applicable? | Is order fulfilled via ECO (marketplace)? | Section 52 CGST Act |

---

## 2. State/UT Code Master Data

### 2.1 TypeScript Constants

```typescript
// src/gst/master/stateCodes.ts

export type TaxType = 'CGST_SGST' | 'IGST' | 'CGST_UTGST';

export interface StateEntry {
  code: string;          // Two-digit GST state code (e.g., "27")
  name: string;          // Full state name
  shortCode: string;     // ISO 3166-2 province code used by Shopify (e.g., "MH")
  isUT: boolean;         // true = UTGST applies; false = SGST applies
  hasLegislature: boolean; // UTs with legislature use SGST, not UTGST
  isSpecialCategory: boolean; // Special category states (lower composition threshold)
}

// Complete 37 States + UTs + special codes
export const STATE_MASTER: Record<string, StateEntry> = {
  '01': { code: '01', name: 'Jammu & Kashmir',          shortCode: 'JK', isUT: true,  hasLegislature: true,  isSpecialCategory: false },
  '02': { code: '02', name: 'Himachal Pradesh',         shortCode: 'HP', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '03': { code: '03', name: 'Punjab',                   shortCode: 'PB', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '04': { code: '04', name: 'Chandigarh',               shortCode: 'CH', isUT: true,  hasLegislature: false, isSpecialCategory: false },
  '05': { code: '05', name: 'Uttarakhand',              shortCode: 'UK', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '06': { code: '06', name: 'Haryana',                  shortCode: 'HR', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '07': { code: '07', name: 'Delhi',                    shortCode: 'DL', isUT: true,  hasLegislature: true,  isSpecialCategory: false },
  '08': { code: '08', name: 'Rajasthan',                shortCode: 'RJ', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '09': { code: '09', name: 'Uttar Pradesh',            shortCode: 'UP', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '10': { code: '10', name: 'Bihar',                    shortCode: 'BR', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '11': { code: '11', name: 'Sikkim',                   shortCode: 'SK', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '12': { code: '12', name: 'Arunachal Pradesh',        shortCode: 'AR', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '13': { code: '13', name: 'Nagaland',                 shortCode: 'NL', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '14': { code: '14', name: 'Manipur',                  shortCode: 'MN', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '15': { code: '15', name: 'Mizoram',                  shortCode: 'MZ', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '16': { code: '16', name: 'Tripura',                  shortCode: 'TR', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '17': { code: '17', name: 'Meghalaya',                shortCode: 'ML', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '18': { code: '18', name: 'Assam',                    shortCode: 'AS', isUT: false, hasLegislature: false, isSpecialCategory: true  },
  '19': { code: '19', name: 'West Bengal',              shortCode: 'WB', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '20': { code: '20', name: 'Jharkhand',                shortCode: 'JH', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '21': { code: '21', name: 'Odisha',                   shortCode: 'OD', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '22': { code: '22', name: 'Chhattisgarh',             shortCode: 'CG', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '23': { code: '23', name: 'Madhya Pradesh',           shortCode: 'MP', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '24': { code: '24', name: 'Gujarat',                  shortCode: 'GJ', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '25': { code: '25', name: 'Daman & Diu',              shortCode: 'DD', isUT: true,  hasLegislature: false, isSpecialCategory: false },
  '26': { code: '26', name: 'Dadra & Nagar Haveli',     shortCode: 'DN', isUT: true,  hasLegislature: false, isSpecialCategory: false },
  '27': { code: '27', name: 'Maharashtra',              shortCode: 'MH', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '28': { code: '28', name: 'Andhra Pradesh (new)',     shortCode: 'AP', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '29': { code: '29', name: 'Karnataka',                shortCode: 'KA', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '30': { code: '30', name: 'Goa',                      shortCode: 'GA', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '31': { code: '31', name: 'Lakshadweep',              shortCode: 'LD', isUT: true,  hasLegislature: false, isSpecialCategory: false },
  '32': { code: '32', name: 'Kerala',                   shortCode: 'KL', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '33': { code: '33', name: 'Tamil Nadu',               shortCode: 'TN', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '34': { code: '34', name: 'Puducherry',               shortCode: 'PY', isUT: true,  hasLegislature: true,  isSpecialCategory: false },
  '35': { code: '35', name: 'Andaman & Nicobar Islands',shortCode: 'AN', isUT: true,  hasLegislature: false, isSpecialCategory: false },
  '36': { code: '36', name: 'Telangana',                shortCode: 'TS', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '37': { code: '37', name: 'Andhra Pradesh (residual)',shortCode: 'AP', isUT: false, hasLegislature: false, isSpecialCategory: false },
  '38': { code: '38', name: 'Ladakh',                   shortCode: 'LA', isUT: true,  hasLegislature: false, isSpecialCategory: false },
  '97': { code: '97', name: 'Other Territory',          shortCode: 'OT', isUT: true,  hasLegislature: false, isSpecialCategory: false },
  '99': { code: '99', name: 'Centre Jurisdiction',      shortCode: 'CJ', isUT: false, hasLegislature: false, isSpecialCategory: false },
};

// Lookup by Shopify province_code (ISO 3166-2:IN short code)
export const SHOPIFY_PROVINCE_TO_GST_CODE: Record<string, string> = {
  'JK': '01', 'HP': '02', 'PB': '03', 'CH': '04', 'UK': '05',
  'HR': '06', 'DL': '07', 'RJ': '08', 'UP': '09', 'BR': '10',
  'SK': '11', 'AR': '12', 'NL': '13', 'MN': '14', 'MZ': '15',
  'TR': '16', 'ML': '17', 'AS': '18', 'WB': '19', 'JH': '20',
  'OD': '21', 'CG': '22', 'MP': '23', 'GJ': '24', 'DD': '25',
  'DN': '26', 'MH': '27', 'AP': '28', 'KA': '29', 'GA': '30',
  'LD': '31', 'KL': '32', 'TN': '33', 'PY': '34', 'AN': '35',
  'TS': '36', 'LA': '38',
};

// Helper: get state entry by Shopify province code
export function getStateByProvinceCode(provinceCode: string): StateEntry | null {
  const gstCode = SHOPIFY_PROVINCE_TO_GST_CODE[provinceCode.toUpperCase()];
  if (!gstCode) return null;
  return STATE_MASTER[gstCode] ?? null;
}

// Helper: get state entry by GST state code
export function getStateByGSTCode(gstCode: string): StateEntry | null {
  return STATE_MASTER[gstCode] ?? null;
}

// Helper: extract state code from GSTIN (first 2 digits)
export function getStateCodeFromGSTIN(gstin: string): string {
  return gstin.substring(0, 2);
}

// Helper: determine which tax component applies in the destination state
export function determineTaxType(
  supplierStateCode: string,
  placeOfSupplyCode: string
): TaxType {
  if (supplierStateCode === placeOfSupplyCode) {
    const state = STATE_MASTER[placeOfSupplyCode];
    if (!state) return 'CGST_SGST';
    // UTs without legislature → UTGST; UTs with legislature (Delhi, Puducherry, J&K) → SGST
    if (state.isUT && !state.hasLegislature) {
      return 'CGST_UTGST';
    }
    return 'CGST_SGST';
  }
  return 'IGST';
}
```

---

## 3. Place of Supply Engine

### 3.1 Design Principles

Place of Supply (PoS) determines whether a supply is intra-state or inter-state, which governs the tax type. For e-commerce:
- **B2C (unregistered buyer):** Section 10(1)(ca) — PoS = delivery address state
- **B2B (registered buyer, goods move):** Section 10(1)(a) — PoS = where movement terminates = buyer's state
- **Bill-to-ship-to:** Section 10(1)(b) — PoS = principal place of business of the buyer (billing address state)

### 3.2 Complete Implementation

```typescript
// src/gst/engine/placeOfSupply.ts

import {
  STATE_MASTER,
  SHOPIFY_PROVINCE_TO_GST_CODE,
  getStateCodeFromGSTIN,
  determineTaxType,
  TaxType,
} from '../master/stateCodes';

export type POSRule =
  | 'SECTION_10_1_A'   // Goods movement → delivery terminates
  | 'SECTION_10_1_B'   // Bill-to-ship-to → buyer's principal place of business
  | 'SECTION_10_1_CA'  // B2C unregistered → delivery address
  | 'SECTION_10_1_C'   // No movement → location of goods
  | 'SECTION_11_EXPORT'; // Export → outside India

export interface PlaceOfSupplyInput {
  sellerGSTIN: string;                 // Seller's GSTIN (first 2 digits = seller state)
  buyerGSTIN?: string | null;          // Buyer's GSTIN (if registered); null/undefined for B2C
  shippingProvinceCode?: string | null; // Shopify shipping_address.province_code (ISO 3166-2)
  billingProvinceCode?: string | null;  // Shopify billing_address.province_code
  shippingCountryCode?: string | null;  // Shopify shipping_address.country_code
  isBillToShipTo?: boolean;            // True when billing and shipping addresses differ AND buyer is B2B
  isSEZ?: boolean;                     // True when buyer is an SEZ unit/developer
  isExport?: boolean;                  // True when shipping outside India
}

export interface PlaceOfSupplyResult {
  placeOfSupplyCode: string;   // GST 2-digit state code
  placeOfSupplyName: string;   // State name
  supplierStateCode: string;   // Seller's state code
  isInterState: boolean;       // true = IGST; false = CGST+SGST/UTGST
  taxType: TaxType;
  posRule: POSRule;
  isSEZ: boolean;
  isExport: boolean;
  buyerStateCode?: string;     // For B2B: buyer's GSTIN state
  deliveryStateCode?: string;  // Actual delivery state (ship-to)
  errors: string[];
}

export function determinePlaceOfSupply(input: PlaceOfSupplyInput): PlaceOfSupplyResult {
  const errors: string[] = [];
  const supplierStateCode = getStateCodeFromGSTIN(input.sellerGSTIN);

  // --- EXPORT: Always IGST ---
  if (input.isExport || (input.shippingCountryCode && input.shippingCountryCode !== 'IN')) {
    return {
      placeOfSupplyCode: '96', // Outside India
      placeOfSupplyName: 'Outside India',
      supplierStateCode,
      isInterState: true,
      taxType: 'IGST',
      posRule: 'SECTION_11_EXPORT',
      isSEZ: false,
      isExport: true,
      errors,
    };
  }

  // --- SEZ: Always IGST (treated as inter-state) ---
  if (input.isSEZ) {
    const buyerStateCode = input.buyerGSTIN
      ? getStateCodeFromGSTIN(input.buyerGSTIN)
      : (input.shippingProvinceCode ? SHOPIFY_PROVINCE_TO_GST_CODE[input.shippingProvinceCode.toUpperCase()] : supplierStateCode);
    return {
      placeOfSupplyCode: buyerStateCode ?? supplierStateCode,
      placeOfSupplyName: STATE_MASTER[buyerStateCode ?? supplierStateCode]?.name ?? 'Unknown',
      supplierStateCode,
      isInterState: true, // SEZ always inter-state
      taxType: 'IGST',
      posRule: 'SECTION_10_1_A',
      isSEZ: true,
      isExport: false,
      errors,
    };
  }

  // --- B2B: Registered Buyer ---
  if (input.buyerGSTIN && input.buyerGSTIN.length === 15) {
    const buyerStateCode = getStateCodeFromGSTIN(input.buyerGSTIN);

    // Bill-to-ship-to (Section 10(1)(b)):
    // B2B where goods are delivered to a 3rd party (ship-to != bill-to)
    // PoS = buyer's principal place of business = billing state = GSTIN state
    if (input.isBillToShipTo) {
      const isInterState = supplierStateCode !== buyerStateCode;
      const taxType = determineTaxType(supplierStateCode, buyerStateCode);
      const deliveryStateCode = input.shippingProvinceCode
        ? SHOPIFY_PROVINCE_TO_GST_CODE[input.shippingProvinceCode.toUpperCase()]
        : buyerStateCode;
      return {
        placeOfSupplyCode: buyerStateCode,
        placeOfSupplyName: STATE_MASTER[buyerStateCode]?.name ?? 'Unknown',
        supplierStateCode,
        isInterState,
        taxType,
        posRule: 'SECTION_10_1_B',
        isSEZ: false,
        isExport: false,
        buyerStateCode,
        deliveryStateCode,
        errors,
      };
    }

    // Standard B2B (Section 10(1)(a)): PoS = where movement terminates = delivery state
    // Typically the same as buyer's GSTIN state
    const deliveryStateCode = input.shippingProvinceCode
      ? SHOPIFY_PROVINCE_TO_GST_CODE[input.shippingProvinceCode.toUpperCase()]
      : buyerStateCode;

    // If delivery state differs from buyer GSTIN state, use delivery state
    const posCode = deliveryStateCode ?? buyerStateCode;
    const isInterState = supplierStateCode !== posCode;
    const taxType = determineTaxType(supplierStateCode, posCode);

    return {
      placeOfSupplyCode: posCode,
      placeOfSupplyName: STATE_MASTER[posCode]?.name ?? 'Unknown',
      supplierStateCode,
      isInterState,
      taxType,
      posRule: 'SECTION_10_1_A',
      isSEZ: false,
      isExport: false,
      buyerStateCode,
      deliveryStateCode: posCode,
      errors,
    };
  }

  // --- B2C: Unregistered Buyer ---
  // Section 10(1)(ca): PoS = delivery address
  if (input.shippingProvinceCode) {
    const deliveryStateCode = SHOPIFY_PROVINCE_TO_GST_CODE[input.shippingProvinceCode.toUpperCase()];
    if (!deliveryStateCode) {
      errors.push(`Unknown Shopify province code: ${input.shippingProvinceCode}`);
      // Fallback to seller's state
      const taxType = determineTaxType(supplierStateCode, supplierStateCode);
      return {
        placeOfSupplyCode: supplierStateCode,
        placeOfSupplyName: STATE_MASTER[supplierStateCode]?.name ?? 'Unknown',
        supplierStateCode,
        isInterState: false,
        taxType,
        posRule: 'SECTION_10_1_CA',
        isSEZ: false,
        isExport: false,
        deliveryStateCode: supplierStateCode,
        errors,
      };
    }
    const isInterState = supplierStateCode !== deliveryStateCode;
    const taxType = determineTaxType(supplierStateCode, deliveryStateCode);
    return {
      placeOfSupplyCode: deliveryStateCode,
      placeOfSupplyName: STATE_MASTER[deliveryStateCode]?.name ?? 'Unknown',
      supplierStateCode,
      isInterState,
      taxType,
      posRule: 'SECTION_10_1_CA',
      isSEZ: false,
      isExport: false,
      deliveryStateCode,
      errors,
    };
  }

  // --- Fallback: No shipping address; PoS = seller's state ---
  errors.push('No shipping address found; defaulting place of supply to seller state');
  const taxType = determineTaxType(supplierStateCode, supplierStateCode);
  return {
    placeOfSupplyCode: supplierStateCode,
    placeOfSupplyName: STATE_MASTER[supplierStateCode]?.name ?? 'Unknown',
    supplierStateCode,
    isInterState: false,
    taxType,
    posRule: 'SECTION_10_1_C',
    isSEZ: false,
    isExport: false,
    errors,
  };
}
```

---

## 4. Tax Rate Engine

### 4.1 GST Rate Slabs (Post-GST 2.0, Effective 22 Sep 2025)

| Rate | Category |
|------|----------|
| 0% | Exempt/Nil: fresh food, books, contraceptives, health/life insurance |
| 5% | Essential: basic food, medicines, footwear >₹2,500, hair products, shampoo, soaps |
| 12% | Retained: apparel >₹1,000, footwear ₹1,000–₹2,500, some home goods |
| 18% | Standard: electronics, clothing >₹1,000 (clothing uses 12%), most FMCG, services |
| 40% | Luxury/sin: tobacco, aerated drinks, luxury cars, pan masala |
| 3% | Precious metals: gold, silver |
| 0.25% | Rough diamonds |

> **Developer Note:** Always use the HSN-level rate, not just the chapter-level rate. Item value matters for apparel (≤₹1,000 vs >₹1,000) and footwear thresholds.

### 4.2 Complete Tax Rate Engine

```typescript
// src/gst/engine/taxRate.ts

import { PlaceOfSupplyResult } from './placeOfSupply';

export interface TaxRateEntry {
  hsnCode: string;
  description: string;
  gstRate: number;          // Total GST rate (e.g., 18 for 18%)
  cessRate: number;         // Cess rate (e.g., 0 or additional %)
  valueThresholds?: {       // For rate-by-value items like apparel
    upToValue: number;      // Item value in INR
    rateUpTo: number;       // GST rate for items ≤ upToValue
    rateAbove: number;      // GST rate for items > upToValue
  };
}

// Post-GST 2.0 HSN Rate Table
// Format: hsnPrefix → TaxRateEntry
export const HSN_RATE_TABLE: Record<string, TaxRateEntry> = {
  // --- Apparel (Chapter 61 Knitted, 62 Non-knitted) ---
  '6101': { hsnCode: '6101', description: "Men's outer garments (knitted)", gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6103': { hsnCode: '6103', description: "Men's suits/trousers (knitted)", gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6104': { hsnCode: '6104', description: "Women's suits/dresses (knitted)", gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6105': { hsnCode: '6105', description: "Men's shirts (knitted)", gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6106': { hsnCode: '6106', description: "Women's blouses (knitted)", gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6109': { hsnCode: '6109', description: 'T-shirts/singlets (knitted)', gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6201': { hsnCode: '6201', description: "Men's overcoats (non-knitted)", gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6203': { hsnCode: '6203', description: "Men's suits/trousers (non-knitted)", gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6204': { hsnCode: '6204', description: "Women's suits/dresses (non-knitted)", gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6211': { hsnCode: '6211', description: 'Track suits/swimwear', gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  // --- Footwear (Chapter 64) ---
  '6401': { hsnCode: '6401', description: 'Waterproof footwear', gstRate: 5, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 5 } }, // All footwear >₹2500 → 5% post-GST 2.0
  '6402': { hsnCode: '6402', description: 'Other footwear outer soles/uppers of rubber/plastics', gstRate: 5, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6403': { hsnCode: '6403', description: 'Footwear with outer soles of rubber/leather', gstRate: 5, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6404': { hsnCode: '6404', description: 'Footwear with outer soles of rubber/textile', gstRate: 5, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6405': { hsnCode: '6405', description: 'Other footwear', gstRate: 5, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  // --- Electronics (Chapter 84, 85) ---
  '8471': { hsnCode: '8471', description: 'Computers/Laptops', gstRate: 18, cessRate: 0 },
  '8517': { hsnCode: '8517', description: 'Mobile phones/handsets', gstRate: 18, cessRate: 0 },
  '8518': { hsnCode: '8518', description: 'Headphones/earphones/speakers', gstRate: 18, cessRate: 0 },
  '8528': { hsnCode: '8528', description: 'TV monitors/sets', gstRate: 18, cessRate: 0 }, // Reduced from 28% in GST 2.0
  '8415': { hsnCode: '8415', description: 'Air conditioners', gstRate: 18, cessRate: 0 }, // Reduced from 28% in GST 2.0
  '8509': { hsnCode: '8509', description: 'Kitchen appliances (mixer/grinder)', gstRate: 18, cessRate: 0 },
  '8516': { hsnCode: '8516', description: 'Electric irons/hair dryers', gstRate: 18, cessRate: 0 },
  // --- Beauty & Personal Care (Chapter 33) ---
  '3304': { hsnCode: '3304', description: 'Beauty/skin care preparations', gstRate: 18, cessRate: 0 },
  '3305': { hsnCode: '3305', description: 'Hair preparations (shampoo/hair oil/conditioner)', gstRate: 5, cessRate: 0 }, // Reduced from 18% in GST 2.0
  '3306': { hsnCode: '3306', description: 'Oral hygiene (toothpaste)', gstRate: 5, cessRate: 0 }, // Reduced from 18% in GST 2.0
  '3307': { hsnCode: '3307', description: 'Shaving preparations/deodorants', gstRate: 18, cessRate: 0 },
  '3301': { hsnCode: '3301', description: 'Essential oils', gstRate: 18, cessRate: 0 },
  '3303': { hsnCode: '3303', description: 'Perfumes and toilet waters', gstRate: 18, cessRate: 0 },
  // --- Home Goods & Furniture (Chapter 94) ---
  '9401': { hsnCode: '9401', description: 'Seating furniture (chairs/sofas)', gstRate: 18, cessRate: 0 },
  '9403': { hsnCode: '9403', description: 'Other furniture (tables/wardrobes/beds)', gstRate: 18, cessRate: 0 },
  '9404': { hsnCode: '9404', description: 'Mattresses/pillows/quilts', gstRate: 12, cessRate: 0 },
  '9405': { hsnCode: '9405', description: 'Lamps/lighting fittings', gstRate: 12, cessRate: 0 },
  '6302': { hsnCode: '6302', description: 'Bed linen/table linen', gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '6303': { hsnCode: '6303', description: 'Curtains/blinds', gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateUpTo: 5, rateAbove: 12 } },
  '3924': { hsnCode: '3924', description: 'Plasticware (kitchen containers)', gstRate: 18, cessRate: 0 },
  '7013': { hsnCode: '7013', description: 'Glassware (household)', gstRate: 18, cessRate: 0 },
  '6911': { hsnCode: '6911', description: 'Ceramics (tableware/kitchenware)', gstRate: 12, cessRate: 0 },
  // --- Food & Beverages (Chapter 09, 19, 21) ---
  '0901': { hsnCode: '0901', description: 'Coffee (roasted/ground)', gstRate: 5, cessRate: 0 },
  '1901': { hsnCode: '1901', description: 'Malt extract/food preparations', gstRate: 18, cessRate: 0 },
  '1905': { hsnCode: '1905', description: 'Bread/pastry/biscuits', gstRate: 5, cessRate: 0 }, // Reduced from 18% in GST 2.0
  '2106': { hsnCode: '2106', description: 'Food preparations NES (nutrition supplements)', gstRate: 18, cessRate: 0 },
  '1806': { hsnCode: '1806', description: 'Chocolate and cocoa products', gstRate: 5, cessRate: 0 }, // Reduced from 18% in GST 2.0
  '2201': { hsnCode: '2201', description: 'Mineral water/packaged drinking water', gstRate: 5, cessRate: 0 }, // Reduced from 18% in GST 2.0
  // --- Books & Educational ---
  '4901': { hsnCode: '4901', description: 'Books (printed)', gstRate: 0, cessRate: 0 },
  '4902': { hsnCode: '4902', description: 'Newspapers/journals', gstRate: 0, cessRate: 0 },
  // --- Precious Metals ---
  '7108': { hsnCode: '7108', description: 'Gold', gstRate: 3, cessRate: 0 },
  '7106': { hsnCode: '7106', description: 'Silver', gstRate: 3, cessRate: 0 },
  '7102': { hsnCode: '7102', description: 'Rough diamonds', gstRate: 0.25, cessRate: 0 },
  // --- Default ---
  'DEFAULT': { hsnCode: 'DEFAULT', description: 'General goods (default 18%)', gstRate: 18, cessRate: 0 },
};

export interface TaxCalculationResult {
  hsnCode: string;
  taxableValue: number;
  gstRate: number;        // Total GST rate applied
  cessRate: number;
  cgstRate: number;
  cgstAmount: number;
  sgstRate: number;
  sgstAmount: number;
  igstRate: number;
  igstAmount: number;
  utgstRate: number;
  utgstAmount: number;
  cessAmount: number;
  totalTax: number;
  totalValue: number;     // taxableValue + totalTax
  taxType: string;        // 'CGST_SGST' | 'IGST' | 'CGST_UTGST'
}

// Resolve the effective GST rate for an HSN code considering value thresholds
function resolveGSTRate(entry: TaxRateEntry, itemValuePerUnit: number): number {
  if (entry.valueThresholds) {
    return itemValuePerUnit <= entry.valueThresholds.upToValue
      ? entry.valueThresholds.rateUpTo
      : entry.valueThresholds.rateAbove;
  }
  return entry.gstRate;
}

// Find best-matching HSN entry: try 8-digit, then 6, then 4, then 2
function lookupHSNRate(hsnCode: string): TaxRateEntry {
  const code = hsnCode.replace(/\s/g, '');
  for (let len = Math.min(code.length, 8); len >= 2; len -= 2) {
    const prefix = code.substring(0, len);
    if (HSN_RATE_TABLE[prefix]) return HSN_RATE_TABLE[prefix];
  }
  return HSN_RATE_TABLE['DEFAULT'];
}

export function calculateTax(
  hsnCode: string,
  pos: PlaceOfSupplyResult,
  taxableValue: number,           // Pre-tax value in INR (decimal)
  itemValuePerUnit: number = taxableValue, // For threshold-based rates
  taxesIncluded: boolean = false  // If Shopify price already includes tax
): TaxCalculationResult {
  const entry = lookupHSNRate(hsnCode);
  const gstRate = resolveGSTRate(entry, itemValuePerUnit);
  const cessRate = entry.cessRate;

  // If taxes are included in price, back-calculate taxable value
  let effectiveTaxable = taxableValue;
  if (taxesIncluded) {
    const totalRate = (gstRate + cessRate) / 100;
    effectiveTaxable = taxableValue / (1 + totalRate);
  }

  effectiveTaxable = Math.round(effectiveTaxable * 100) / 100;
  const cessAmount = Math.round(effectiveTaxable * cessRate) / 100;

  let cgstRate = 0, cgstAmount = 0;
  let sgstRate = 0, sgstAmount = 0;
  let igstRate = 0, igstAmount = 0;
  let utgstRate = 0, utgstAmount = 0;

  switch (pos.taxType) {
    case 'CGST_SGST':
      cgstRate = gstRate / 2;
      sgstRate = gstRate / 2;
      cgstAmount = Math.round(effectiveTaxable * cgstRate) / 100;
      sgstAmount = Math.round(effectiveTaxable * sgstRate) / 100;
      break;
    case 'IGST':
      igstRate = gstRate;
      igstAmount = Math.round(effectiveTaxable * igstRate) / 100;
      break;
    case 'CGST_UTGST':
      cgstRate = gstRate / 2;
      utgstRate = gstRate / 2;
      cgstAmount = Math.round(effectiveTaxable * cgstRate) / 100;
      utgstAmount = Math.round(effectiveTaxable * utgstRate) / 100;
      break;
  }

  const totalTax = cgstAmount + sgstAmount + igstAmount + utgstAmount + cessAmount;
  const totalValue = Math.round((effectiveTaxable + totalTax) * 100) / 100;

  return {
    hsnCode: entry.hsnCode,
    taxableValue: effectiveTaxable,
    gstRate,
    cessRate,
    cgstRate, cgstAmount,
    sgstRate, sgstAmount,
    igstRate, igstAmount,
    utgstRate, utgstAmount,
    cessAmount,
    totalTax,
    totalValue,
    taxType: pos.taxType,
  };
}
```

---

## 5. GSTIN Validation

### 5.1 GSTIN Structure

```
Position: 1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
          [S][S][P][P][P][P][P][P][P][P][P][P][E][Z][C]

SS  = State Code (01–38, 97, 99) — 2 digits
PPPPPPPPPP = PAN (10 chars: 5 letters + 4 digits + 1 letter)
E   = Entity number (1–9, A–Z) — distinguishes multiple GSTINs under same PAN
Z   = Always literal 'Z'
C   = Check digit (Luhn mod-36 algorithm)
```

### 5.2 Complete Validation Implementation

```typescript
// src/gst/validation/gstin.ts

import { STATE_MASTER } from '../master/stateCodes';

export interface GSTINValidationResult {
  isValid: boolean;
  gstin: string;
  stateCode: string;
  stateName: string;
  pan: string;
  entityNumber: string;
  errors: string[];
}

// Luhn mod-36 checksum (GST Suvidha Provider / GSTN algorithm)
function computeGSTINChecksum(gstin14: string): string {
  const CHARSET = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ';
  const M = 36; // modulus
  let factor = 2;
  let sum = 0;

  for (let i = gstin14.length - 1; i >= 0; i--) {
    let codePoint = CHARSET.indexOf(gstin14[i]);
    if (codePoint < 0) return '';
    let addend = factor * codePoint;
    factor = factor === 2 ? 1 : 2;
    addend = Math.floor(addend / M) + (addend % M);
    sum += addend;
  }

  const remainder = sum % M;
  const checkIndex = (M - remainder) % M;
  return CHARSET[checkIndex];
}

export function validateGSTIN(rawGSTIN: string): GSTINValidationResult {
  const errors: string[] = [];
  const gstin = rawGSTIN.trim().toUpperCase();

  // Basic format check
  if (gstin.length !== 15) {
    errors.push(`GSTIN must be 15 characters; got ${gstin.length}`);
    return { isValid: false, gstin, stateCode: '', stateName: '', pan: '', entityNumber: '', errors };
  }

  // Regex pattern: 2 digits + 5 letters + 4 digits + 1 letter + 1 alphanumeric + Z + 1 alphanumeric
  const GSTIN_REGEX = /^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}Z[0-9A-Z]{1}$/;
  if (!GSTIN_REGEX.test(gstin)) {
    errors.push('GSTIN does not match pattern: 2-digit-state + 10-char-PAN + entity + Z + checkdigit');
    return { isValid: false, gstin, stateCode: '', stateName: '', pan: '', entityNumber: '', errors };
  }

  const stateCode = gstin.substring(0, 2);
  const pan = gstin.substring(2, 12);
  const entityNumber = gstin.substring(12, 13);
  const zChar = gstin.charAt(13);
  const checkChar = gstin.charAt(14);

  // State code validation
  if (!STATE_MASTER[stateCode]) {
    errors.push(`Invalid state code: ${stateCode}`);
  }

  // Z character check
  if (zChar !== 'Z') {
    errors.push(`Position 14 must be 'Z'; got '${zChar}'`);
  }

  // Checksum verification
  const expectedCheck = computeGSTINChecksum(gstin.substring(0, 14));
  if (expectedCheck === '') {
    errors.push('Checksum computation failed — invalid characters in GSTIN');
  } else if (expectedCheck !== checkChar) {
    errors.push(`Checksum mismatch: expected '${expectedCheck}', got '${checkChar}'`);
  }

  const isValid = errors.length === 0;
  return {
    isValid,
    gstin,
    stateCode,
    stateName: STATE_MASTER[stateCode]?.name ?? 'Unknown',
    pan,
    entityNumber,
    errors,
  };
}

// Quick boolean validator (use in runtime checks)
export function isValidGSTIN(gstin: string | null | undefined): boolean {
  if (!gstin) return false;
  return validateGSTIN(gstin).isValid;
}
```

---

## 6. Invoice Generation Engine

### 6.1 Invoice Number Generation

```typescript
// src/gst/invoice/invoiceNumber.ts

export interface InvoiceSeriesConfig {
  prefix: string;       // e.g., "INV" or "TAX"
  separator: string;    // e.g., "/" or "-"
  includeFY: boolean;   // e.g., "INV/2025-26/001"
  padWidth: number;     // Zero-pad the sequence number (e.g., 4 → 0001)
  maxLength: number;    // Max 16 chars (Rule 46)
}

export const DEFAULT_INVOICE_SERIES: InvoiceSeriesConfig = {
  prefix: 'INV',
  separator: '/',
  includeFY: true,
  padWidth: 5,
  maxLength: 16,
};

// Generates FY string from a date, e.g., "2025-26"
export function getFinancialYear(date: Date): string {
  const year = date.getFullYear();
  const month = date.getMonth() + 1; // 0-indexed
  if (month >= 4) {
    return `${year}-${String(year + 1).slice(2)}`;
  } else {
    return `${year - 1}-${String(year).slice(2)}`;
  }
}

export function generateInvoiceNumber(
  sequence: number,
  date: Date,
  config: InvoiceSeriesConfig = DEFAULT_INVOICE_SERIES
): string {
  const fy = config.includeFY ? getFinancialYear(date) : '';
  const seqPadded = String(sequence).padStart(config.padWidth, '0');
  const parts = [config.prefix, fy, seqPadded].filter(Boolean);
  const invoiceNumber = parts.join(config.separator);

  if (invoiceNumber.length > config.maxLength) {
    throw new Error(
      `Invoice number "${invoiceNumber}" exceeds ${config.maxLength} characters (Rule 46). ` +
      `Reduce prefix, separator, or padding.`
    );
  }

  // Convert to uppercase (IRP requirement from June 2025)
  return invoiceNumber.toUpperCase();
}
```

### 6.2 Shopify Order to GST Invoice

```typescript
// src/gst/invoice/invoiceGenerator.ts

import { determinePlaceOfSupply, PlaceOfSupplyInput } from '../engine/placeOfSupply';
import { calculateTax } from '../engine/taxRate';
import { isValidGSTIN } from '../validation/gstin';
import { generateInvoiceNumber, getFinancialYear } from './invoiceNumber';
import {
  GSTInvoice, GSTInvoiceLine, GSTCreditNote, GSTBillOfSupply,
  InvoiceType, SupplyType
} from '../../types/gstInvoice';

export interface SellerProfile {
  gstin: string;
  legalName: string;
  tradeName?: string;
  address1: string;
  address2?: string;
  city: string;
  stateCode: string;
  pincode: string;
  phone?: string;
  email?: string;
  pan: string;
  aato: number;           // Annual Aggregate Turnover (INR crore)
  isComposition: boolean;
  ecoGSTIN?: string;      // If selling via marketplace, ECO's GSTIN
}

// Shopify-shaped input (subset of Shopify Order object)
export interface ShopifyOrderInput {
  id: number;
  name: string;             // Order name e.g., "#1001"
  created_at: string;       // ISO8601 date
  financial_status: string;
  customer?: {
    id: number;
    email?: string;
    first_name?: string;
    last_name?: string;
    phone?: string;
    // Custom GST metafields
    gstin?: string;
    legalName?: string;
  };
  shipping_address?: {
    name?: string;
    address1?: string;
    address2?: string;
    city?: string;
    province_code?: string; // ISO 3166-2 e.g., "MH"
    country_code?: string;
    zip?: string;
    phone?: string;
  };
  billing_address?: {
    name?: string;
    address1?: string;
    address2?: string;
    city?: string;
    province_code?: string;
    country_code?: string;
    zip?: string;
  };
  line_items: Array<{
    id: number;
    title: string;
    variant_title?: string;
    sku?: string;
    quantity: number;
    price: string;          // Per unit price as string (Shopify returns strings)
    total_discount: string;
    taxable: boolean;
    tax_lines: Array<{
      title: string;
      price: string;
      rate: number;
    }>;
    // Custom metafield from product
    hsn_code?: string;
    uqc?: string;           // Unit Quantity Code: NOS, KGS, MTR etc.
  }>;
  taxes_included: boolean;
  total_price: string;
  total_tax: string;
  subtotal_price: string;
  discount_codes?: Array<{ code: string; amount: string }>;
  note?: string;
}

export interface GenerateInvoiceOptions {
  invoiceSequence: number;
  invoiceDate?: Date;
  isBillToShipTo?: boolean;
  isSEZ?: boolean;
  forceB2C?: boolean;
}

export function generateGSTInvoice(
  order: ShopifyOrderInput,
  seller: SellerProfile,
  opts: GenerateInvoiceOptions
): GSTInvoice | GSTBillOfSupply {
  const invoiceDate = opts.invoiceDate ?? new Date(order.created_at);
  const buyerGSTIN = order.customer?.gstin && isValidGSTIN(order.customer.gstin)
    ? order.customer.gstin
    : null;

  // --- If composition dealer: must issue Bill of Supply (no tax collected) ---
  if (seller.isComposition) {
    return buildBillOfSupply(order, seller, opts, invoiceDate, buyerGSTIN);
  }

  // --- Determine Place of Supply ---
  const posInput: PlaceOfSupplyInput = {
    sellerGSTIN: seller.gstin,
    buyerGSTIN: buyerGSTIN ?? undefined,
    shippingProvinceCode: order.shipping_address?.province_code ?? null,
    billingProvinceCode: order.billing_address?.province_code ?? null,
    shippingCountryCode: order.shipping_address?.country_code ?? null,
    isBillToShipTo: opts.isBillToShipTo,
    isSEZ: opts.isSEZ,
    isExport: order.shipping_address?.country_code
      ? order.shipping_address.country_code !== 'IN'
      : false,
  };
  const pos = determinePlaceOfSupply(posInput);

  // --- Build Line Items ---
  const lines: GSTInvoiceLine[] = order.line_items.map((item, idx) => {
    const hsnCode = item.hsn_code ?? 'DEFAULT';
    const unitPrice = parseFloat(item.price);
    const quantity = item.quantity;
    const discount = parseFloat(item.total_discount);
    const grossValue = unitPrice * quantity;
    const taxableValue = grossValue - discount;

    const tax = item.taxable
      ? calculateTax(hsnCode, pos, taxableValue, unitPrice, order.taxes_included)
      : {
          hsnCode, taxableValue, gstRate: 0, cessRate: 0,
          cgstRate: 0, cgstAmount: 0, sgstRate: 0, sgstAmount: 0,
          igstRate: 0, igstAmount: 0, utgstRate: 0, utgstAmount: 0,
          cessAmount: 0, totalTax: 0, totalValue: taxableValue, taxType: pos.taxType,
        };

    return {
      slNo: idx + 1,
      productDescription: item.variant_title
        ? `${item.title} - ${item.variant_title}`
        : item.title,
      hsnCode,
      uqc: item.uqc ?? 'NOS',
      quantity,
      unitPrice,
      grossValue,
      discount,
      taxableValue: tax.taxableValue,
      gstRate: tax.gstRate,
      cgstRate: tax.cgstRate, cgstAmount: tax.cgstAmount,
      sgstRate: tax.sgstRate, sgstAmount: tax.sgstAmount,
      igstRate: tax.igstRate, igstAmount: tax.igstAmount,
      utgstRate: tax.utgstRate, utgstAmount: tax.utgstAmount,
      cessRate: tax.cessRate, cessAmount: tax.cessAmount,
      totalTaxAmount: tax.totalTax,
      totalAmount: tax.totalValue,
      sku: item.sku,
    };
  });

  // --- Totals ---
  const totalTaxableValue = lines.reduce((s, l) => s + l.taxableValue, 0);
  const totalCGST = lines.reduce((s, l) => s + l.cgstAmount, 0);
  const totalSGST = lines.reduce((s, l) => s + l.sgstAmount, 0);
  const totalIGST = lines.reduce((s, l) => s + l.igstAmount, 0);
  const totalUTGST = lines.reduce((s, l) => s + l.utgstAmount, 0);
  const totalCess = lines.reduce((s, l) => s + l.cessAmount, 0);
  const totalTax = totalCGST + totalSGST + totalIGST + totalUTGST + totalCess;
  const grandTotal = totalTaxableValue + totalTax;

  // --- Determine invoice type ---
  const invoiceType: InvoiceType = 'TAX_INVOICE';
  const supplyType: SupplyType = buyerGSTIN
    ? (opts.isSEZ ? 'SEZWP' : 'B2B')
    : (pos.isExport ? 'EXPWP' : 'B2C');

  const invoiceNumber = generateInvoiceNumber(opts.invoiceSequence, invoiceDate);

  const invoice: GSTInvoice = {
    // Identification
    invoiceNumber,
    invoiceDate,
    financialYear: getFinancialYear(invoiceDate),
    invoiceType,
    supplyType,
    shopifyOrderId: String(order.id),
    shopifyOrderName: order.name,

    // Supplier
    supplierGSTIN: seller.gstin,
    supplierLegalName: seller.legalName,
    supplierTradeName: seller.tradeName,
    supplierAddress1: seller.address1,
    supplierAddress2: seller.address2,
    supplierCity: seller.city,
    supplierStateCode: seller.stateCode,
    supplierPincode: seller.pincode,

    // Recipient
    recipientGSTIN: buyerGSTIN ?? 'URP', // URP = Unregistered Person
    recipientName: order.customer
      ? (order.customer.legalName ?? `${order.customer.first_name ?? ''} ${order.customer.last_name ?? ''}`.trim())
      : (order.billing_address?.name ?? 'Consumer'),
    recipientAddress1: order.billing_address?.address1 ?? order.shipping_address?.address1 ?? '',
    recipientAddress2: order.billing_address?.address2,
    recipientCity: order.billing_address?.city ?? order.shipping_address?.city ?? '',
    recipientStateCode: pos.buyerStateCode ?? pos.placeOfSupplyCode,
    recipientPincode: order.billing_address?.zip ?? order.shipping_address?.zip ?? '',

    // Delivery (if different from billing)
    deliveryAddress1: order.shipping_address?.address1,
    deliveryAddress2: order.shipping_address?.address2,
    deliveryCity: order.shipping_address?.city,
    deliveryStateCode: pos.deliveryStateCode,
    deliveryPincode: order.shipping_address?.zip,

    // Place of Supply
    placeOfSupplyCode: pos.placeOfSupplyCode,
    placeOfSupplyName: pos.placeOfSupplyName,
    isInterState: pos.isInterState,
    taxType: pos.taxType,

    // Reverse Charge
    isReverseCharge: false, // Set true if supplier is unregistered and RCM applies

    // ECO (if sold via marketplace)
    ecoGSTIN: seller.ecoGSTIN,

    // Line items
    lineItems: lines,

    // Totals
    totalTaxableValue: Math.round(totalTaxableValue * 100) / 100,
    totalCGST: Math.round(totalCGST * 100) / 100,
    totalSGST: Math.round(totalSGST * 100) / 100,
    totalIGST: Math.round(totalIGST * 100) / 100,
    totalUTGST: Math.round(totalUTGST * 100) / 100,
    totalCess: Math.round(totalCess * 100) / 100,
    totalTax: Math.round(totalTax * 100) / 100,
    roundOff: 0,
    grandTotal: Math.round(grandTotal * 100) / 100,

    // E-invoicing (populated later by e-invoice module)
    irn: undefined,
    ackNumber: undefined,
    ackDate: undefined,
    signedQRCode: undefined,

    // Meta
    status: 'DRAFT',
    isCancelled: false,
    createdAt: new Date(),
    updatedAt: new Date(),
  };

  // Add note for B2C invoices with value ≥ ₹50,000 requiring address
  if (!buyerGSTIN && grandTotal >= 50000 && !invoice.recipientAddress1) {
    invoice.warnings = ['B2C invoice value ≥ ₹50,000: recipient address is mandatory per Rule 46(e)'];
  }

  return invoice;
}

// --- Credit Note Generation (from Shopify Refund) ---
export interface ShopifyRefundInput {
  id: number;
  order_id: number;
  created_at: string;
  note?: string;
  refund_line_items: Array<{
    line_item_id: number;
    quantity: number;
    subtotal: number;
    total_tax: number;
    line_item: {
      title: string;
      price: string;
      hsn_code?: string;
      uqc?: string;
    };
  }>;
  transactions: Array<{
    amount: string;
    kind: string;
  }>;
}

export function generateCreditNote(
  refund: ShopifyRefundInput,
  originalInvoice: GSTInvoice,
  seller: SellerProfile,
  creditNoteSequence: number
): GSTCreditNote {
  const creditDate = new Date(refund.created_at);
  const creditNoteNumber = `CN/${getFinancialYear(creditDate)}/${String(creditNoteSequence).padStart(5, '0')}`;

  const lines = refund.refund_line_items.map((ri, idx) => {
    const hsnCode = ri.line_item.hsn_code ?? 'DEFAULT';
    // Re-use same PoS as original invoice for consistent tax type
    const posInput: PlaceOfSupplyInput = {
      sellerGSTIN: seller.gstin,
      buyerGSTIN: originalInvoice.recipientGSTIN !== 'URP' ? originalInvoice.recipientGSTIN : undefined,
      shippingProvinceCode: originalInvoice.deliveryStateCode,
    };
    const pos = determinePlaceOfSupply(posInput);
    const tax = calculateTax(
      hsnCode, pos,
      ri.subtotal,
      parseFloat(ri.line_item.price),
      false
    );
    return {
      slNo: idx + 1,
      productDescription: ri.line_item.title,
      hsnCode,
      uqc: ri.line_item.uqc ?? 'NOS',
      quantity: ri.quantity,
      unitPrice: parseFloat(ri.line_item.price),
      taxableValue: tax.taxableValue,
      cgstRate: tax.cgstRate, cgstAmount: tax.cgstAmount,
      sgstRate: tax.sgstRate, sgstAmount: tax.sgstAmount,
      igstRate: tax.igstRate, igstAmount: tax.igstAmount,
      utgstRate: tax.utgstRate, utgstAmount: tax.utgstAmount,
      cessRate: tax.cessRate, cessAmount: tax.cessAmount,
      totalTaxAmount: tax.totalTax,
      totalAmount: tax.totalValue,
    };
  });

  const totalTaxable = lines.reduce((s, l) => s + l.taxableValue, 0);
  const totalCGST = lines.reduce((s, l) => s + l.cgstAmount, 0);
  const totalSGST = lines.reduce((s, l) => s + l.sgstAmount, 0);
  const totalIGST = lines.reduce((s, l) => s + l.igstAmount, 0);
  const totalUTGST = lines.reduce((s, l) => s + l.utgstAmount, 0);
  const totalCess = lines.reduce((s, l) => s + l.cessAmount, 0);
  const totalTax = totalCGST + totalSGST + totalIGST + totalUTGST + totalCess;

  return {
    creditNoteNumber: creditNoteNumber.toUpperCase(),
    creditNoteDate: creditDate,
    financialYear: getFinancialYear(creditDate),
    originalInvoiceNumber: originalInvoice.invoiceNumber,
    originalInvoiceDate: originalInvoice.invoiceDate,
    shopifyRefundId: String(refund.id),
    shopifyOrderId: String(refund.order_id),
    supplierGSTIN: seller.gstin,
    supplierLegalName: seller.legalName,
    supplierStateCode: seller.stateCode,
    recipientGSTIN: originalInvoice.recipientGSTIN,
    recipientName: originalInvoice.recipientName,
    recipientStateCode: originalInvoice.recipientStateCode,
    placeOfSupplyCode: originalInvoice.placeOfSupplyCode,
    isInterState: originalInvoice.isInterState,
    taxType: originalInvoice.taxType,
    lineItems: lines,
    totalTaxableValue: Math.round(totalTaxable * 100) / 100,
    totalCGST: Math.round(totalCGST * 100) / 100,
    totalSGST: Math.round(totalSGST * 100) / 100,
    totalIGST: Math.round(totalIGST * 100) / 100,
    totalUTGST: Math.round(totalUTGST * 100) / 100,
    totalCess: Math.round(totalCess * 100) / 100,
    totalTax: Math.round(totalTax * 100) / 100,
    grandTotal: Math.round((totalTaxable + totalTax) * 100) / 100,
    reason: refund.note ?? 'Goods returned by customer',
    status: 'DRAFT',
    createdAt: new Date(),
  };
}

function buildBillOfSupply(
  order: ShopifyOrderInput,
  seller: SellerProfile,
  opts: GenerateInvoiceOptions,
  invoiceDate: Date,
  buyerGSTIN: string | null
): GSTBillOfSupply {
  const bos: GSTBillOfSupply = {
    documentNumber: `BOS/${getFinancialYear(invoiceDate)}/${String(opts.invoiceSequence).padStart(5,'0')}`.toUpperCase(),
    documentDate: invoiceDate,
    financialYear: getFinancialYear(invoiceDate),
    shopifyOrderId: String(order.id),
    shopifyOrderName: order.name,
    supplierGSTIN: seller.gstin,
    supplierLegalName: seller.legalName,
    supplierStateCode: seller.stateCode,
    recipientGSTIN: buyerGSTIN ?? 'URP',
    recipientName: order.customer
      ? `${order.customer.first_name ?? ''} ${order.customer.last_name ?? ''}`.trim()
      : 'Consumer',
    recipientStateCode: order.shipping_address?.province_code ?? seller.stateCode,
    lineItems: order.line_items.map((item, idx) => ({
      slNo: idx + 1,
      productDescription: item.title,
      hsnCode: item.hsn_code ?? 'DEFAULT',
      uqc: item.uqc ?? 'NOS',
      quantity: item.quantity,
      unitPrice: parseFloat(item.price),
      taxableValue: parseFloat(item.price) * item.quantity - parseFloat(item.total_discount),
    })),
    totalValue: parseFloat(order.total_price),
    compositionNote: 'Composition taxable person, not eligible to collect tax on supplies',
    status: 'DRAFT',
    createdAt: new Date(),
  };
  return bos;
}
```

---

## 7. HSN Code Management

### 7.1 HSN Hierarchy & Digit Requirements

```
HSN Structure:
  Chapter  (2 digits): e.g., 61 = Knitted apparel
  Heading  (4 digits): e.g., 6109 = T-shirts
  Sub-heading (6 digits): e.g., 610910 = T-shirts, cotton
  Tariff Item (8 digits): e.g., 61091000 = T-shirts, cotton, knitted

Turnover-based requirements:
  AATO ≤ ₹5 crore  → minimum 4-digit HSN in GSTR-1
  AATO > ₹5 crore  → minimum 6-digit HSN in GSTR-1
  Exports            → 8-digit HSN preferred
  E-invoicing        → matches GSTR-1 requirement (4 or 6 digit)
```

### 7.2 HSN Lookup Utility

```typescript
// src/gst/hsn/hsnLookup.ts

export interface HSNEntry {
  code: string;         // Full HSN code (2–8 digits)
  level: 2 | 4 | 6 | 8;
  description: string;
  chapter: string;
  chapterDescription: string;
  gstRate: number;      // Total GST rate
  cessRate: number;
  valueThresholds?: { upToValue: number; rateBelow: number; rateAbove: number };
  uqc: string;          // Default Unit Quantity Code
  isService: boolean;
  sacCode?: string;     // SAC code if service
}

// Common e-commerce HSN directory
export const HSN_DIRECTORY: Record<string, HSNEntry> = {
  // APPAREL
  '61': { code: '61', level: 2, description: 'Knitted/crocheted clothing', chapter: '61',
    chapterDescription: 'Knitted or crocheted clothing', gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateBelow: 5, rateAbove: 12 }, uqc: 'NOS', isService: false },
  '6109': { code: '6109', level: 4, description: 'T-shirts, singlets, other vests', chapter: '61',
    chapterDescription: 'Knitted or crocheted clothing', gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateBelow: 5, rateAbove: 12 }, uqc: 'NOS', isService: false },
  '610910': { code: '610910', level: 6, description: 'T-shirts, singlets of cotton', chapter: '61',
    chapterDescription: 'Knitted or crocheted clothing', gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateBelow: 5, rateAbove: 12 }, uqc: 'NOS', isService: false },
  '61091000': { code: '61091000', level: 8, description: 'T-shirts of cotton, knitted', chapter: '61',
    chapterDescription: 'Knitted or crocheted clothing', gstRate: 12, cessRate: 0,
    valueThresholds: { upToValue: 1000, rateBelow: 5, rateAbove: 12 }, uqc: 'NOS', isService: false },
  // ELECTRONICS
  '8471': { code: '8471', level: 4, description: 'Automatic data processing machines (computers/laptops)', chapter: '84',
    chapterDescription: 'Nuclear reactors, boilers, machinery', gstRate: 18, cessRate: 0, uqc: 'NOS', isService: false },
  '84713020': { code: '84713020', level: 8, description: 'Laptops', chapter: '84',
    chapterDescription: 'Nuclear reactors, boilers, machinery', gstRate: 18, cessRate: 0, uqc: 'NOS', isService: false },
  '8517': { code: '8517', level: 4, description: 'Telephone sets, mobile phones', chapter: '85',
    chapterDescription: 'Electrical machinery', gstRate: 18, cessRate: 0, uqc: 'NOS', isService: false },
  '85171200': { code: '85171200', level: 8, description: 'Mobile phones (smartphones)', chapter: '85',
    chapterDescription: 'Electrical machinery', gstRate: 18, cessRate: 0, uqc: 'NOS', isService: false },
  '8518': { code: '8518', level: 4, description: 'Microphones, loudspeakers, headphones', chapter: '85',
    chapterDescription: 'Electrical machinery', gstRate: 18, cessRate: 0, uqc: 'NOS', isService: false },
  // BEAUTY
  '3304': { code: '3304', level: 4, description: 'Beauty/make-up preparations, skin care', chapter: '33',
    chapterDescription: 'Essential oils, resinoids, perfumery', gstRate: 18, cessRate: 0, uqc: 'NOS', isService: false },
  '3305': { code: '3305', level: 4, description: 'Hair preparations', chapter: '33',
    chapterDescription: 'Essential oils, resinoids, perfumery', gstRate: 5, cessRate: 0, uqc: 'NOS', isService: false },
  // HOME
  '9401': { code: '9401', level: 4, description: 'Seats (chairs, sofas, recliners)', chapter: '94',
    chapterDescription: 'Furniture, bedding, mattresses', gstRate: 18, cessRate: 0, uqc: 'NOS', isService: false },
  '9403': { code: '9403', level: 4, description: 'Other furniture', chapter: '94',
    chapterDescription: 'Furniture, bedding, mattresses', gstRate: 18, cessRate: 0, uqc: 'NOS', isService: false },
  '9404': { code: '9404', level: 4, description: 'Mattresses, bed articles (quilts, pillows)', chapter: '94',
    chapterDescription: 'Furniture, bedding, mattresses', gstRate: 12, cessRate: 0, uqc: 'NOS', isService: false },
  // FOOD
  '0901': { code: '0901', level: 4, description: 'Coffee (roasted, ground)', chapter: '09',
    chapterDescription: 'Coffee, tea, spices', gstRate: 5, cessRate: 0, uqc: 'KGS', isService: false },
  '4901': { code: '4901', level: 4, description: 'Printed books', chapter: '49',
    chapterDescription: 'Books, newspapers, pictures', gstRate: 0, cessRate: 0, uqc: 'NOS', isService: false },
  // SERVICES
  '9983': { code: '9983', level: 6, description: 'IT services, software development', chapter: '99',
    chapterDescription: 'Services', gstRate: 18, cessRate: 0, uqc: 'OTH', isService: true, sacCode: '998314' },
  '9985': { code: '9985', level: 6, description: 'Support services (logistics, warehousing)', chapter: '99',
    chapterDescription: 'Services', gstRate: 18, cessRate: 0, uqc: 'OTH', isService: true, sacCode: '998510' },
};

// Validate HSN code format and get minimum required digits by AATO
export function getRequiredHSNDigits(aato: number): 4 | 6 | 8 {
  if (aato > 500) return 6;  // >₹5 crore → 6 digit (aato in crore)
  return 4;
}

// Expand short HSN to minimum required digits
export function padHSNCode(hsnCode: string, requiredDigits: 4 | 6 | 8): string {
  const code = hsnCode.replace(/\s/g, '');
  if (code.length >= requiredDigits) return code;
  // Pad with zeros to minimum length
  return code.padEnd(requiredDigits, '0');
}

export function lookupHSN(code: string): HSNEntry | null {
  const normalized = code.replace(/\s/g, '');
  // Try exact match first, then progressively shorter prefixes
  for (let len = normalized.length; len >= 2; len -= 2) {
    const key = normalized.substring(0, len);
    if (HSN_DIRECTORY[key]) return HSN_DIRECTORY[key];
  }
  return null;
}
```

---

## 8. GSTR-1 Data Preparation

### 8.1 Table Classification Logic

```
For each invoice, classify into GSTR-1 tables:

B2B invoice (buyer has GSTIN)
  → Table 4A (regular B2B)
  → Table 4B (reverse charge B2B)

B2C invoice (no buyer GSTIN)
  → Inter-state AND invoice value > ₹2.5 lakh → Table 5A (B2C Large)
  → Everything else (intra-state any amount; inter-state ≤₹2.5L) → Table 7 (B2C Small, state-wise)

Credit/Debit notes
  → Table 9B (amendments to registered person invoices)
  → Table 10 (amendments to B2C Large)

HSN Summary
  → Table 12A (B2B)
  → Table 12B (B2C)
```

### 8.2 Complete GSTR-1 Implementation

```typescript
// src/gst/returns/gstr1.ts

import { GSTInvoice, GSTCreditNote } from '../../types/gstInvoice';

// --- Table 4A/4B: B2B Invoices ---
export interface GSTR1_B2BEntry {
  ctin: string;          // Buyer GSTIN
  invoices: Array<{
    inum: string;        // Invoice number
    idt: string;         // Invoice date (DD-MM-YYYY)
    val: number;         // Total invoice value
    pos: string;         // Place of supply (2-digit state code)
    rchrg: 'Y' | 'N';   // Reverse charge
    inv_typ: 'R' | 'SEWP' | 'SEWOP' | 'DE'; // Regular/SEZ with payment/SEZ without payment/Deemed export
    etin?: string;       // E-commerce operator GSTIN
    cflag?: 'Y' | 'N';  // Amendment flag
    itms: Array<{
      num: number;       // Item serial number
      itm_det: {
        txval: number;   // Taxable value
        rt: number;      // GST rate
        camt?: number;   // CGST amount
        samt?: number;   // SGST amount
        iamt?: number;   // IGST amount
        csamt?: number;  // Cess amount
      };
    }>;
  }>;
}

// --- Table 5A/5B: B2C Large (Inter-state, >₹2.5 lakh, unregistered) ---
export interface GSTR1_B2CLEntry {
  pos: string;          // Place of supply (2-digit state code)
  ospvl: string;        // Origin state province value (for e-commerce: ECO GSTIN)
  etin?: string;        // E-commerce operator GSTIN
  inv: Array<{
    inum: string;
    idt: string;
    val: number;
    itms: Array<{
      num: number;
      itm_det: {
        txval: number;
        rt: number;
        iamt: number;
        csamt?: number;
      };
    }>;
  }>;
}

// --- Table 7: B2C Small (consolidated state-wise) ---
export interface GSTR1_B2CSEntry {
  sply_ty: 'INTER' | 'INTRA';
  pos: string;          // Place of supply
  etin?: string;        // E-commerce operator GSTIN
  txval: number;        // Taxable value
  rt: number;           // GST rate
  iamt?: number;        // IGST (if inter-state)
  camt?: number;        // CGST (if intra-state)
  samt?: number;        // SGST (if intra-state)
  csamt?: number;       // Cess
  typ?: 'E'; // E = e-commerce
}

// --- Table 9B: Credit/Debit Notes ---
export interface GSTR1_CDNREntry {
  ctin: string;         // Buyer GSTIN (for registered)
  nt: Array<{
    ntty: 'C' | 'D';   // C = Credit Note, D = Debit Note
    nt_num: string;
    nt_dt: string;      // DD-MM-YYYY
    rsn?: string;       // Reason
    p_gst: 'Y' | 'N'; // Pre-GST indicator
    val: number;
    itms: Array<{
      num: number;
      itm_det: {
        txval: number;
        rt: number;
        camt?: number;
        samt?: number;
        iamt?: number;
        csamt?: number;
      };
    }>;
  }>;
}

// --- Table 12: HSN Summary ---
export interface GSTR1_HSNSummaryEntry {
  num: number;          // Serial number
  hsn_sc: string;       // HSN/SAC code
  desc?: string;        // Description (auto-filled from HSN dropdown)
  uqc: string;          // Unit Quantity Code
  qty: number;          // Total quantity
  val: number;          // Total value
  txval: number;        // Total taxable value
  iamt: number;         // IGST
  camt: number;         // CGST
  samt: number;         // SGST/UTGST
  csamt: number;        // Cess
}

// --- Complete GSTR-1 Report ---
export interface GSTR1Report {
  gstin: string;
  fp: string;           // Filing period: MMYYYY e.g., "032026"
  gt: number;           // Gross turnover
  cur_gt: number;       // Current period turnover
  b2b: GSTR1_B2BEntry[];
  b2cl: GSTR1_B2CLEntry[];
  b2cs: GSTR1_B2CSEntry[];
  cdnr: GSTR1_CDNREntry[];
  hsn: { data: GSTR1_HSNSummaryEntry[] };
  doc_issue?: {
    doc_det: Array<{
      doc_num: number;
      docs: Array<{
        num: number;
        from: string;
        to: string;
        totnum: number;
        cancel: number;
        net_issue: number;
      }>;
    }>;
  };
}

// --- Aggregation Engine ---
export function aggregateToGSTR1(
  invoices: GSTInvoice[],
  creditNotes: GSTCreditNote[],
  period: { month: number; year: number }, // month: 1-12
  sellerGSTIN: string
): GSTR1Report {
  const fp = `${String(period.month).padStart(2, '0')}${period.year}`;

  const b2bMap: Map<string, GSTR1_B2BEntry> = new Map();
  const b2clList: GSTR1_B2CLEntry[] = [];
  const b2csMap: Map<string, GSTR1_B2CSEntry> = new Map(); // key: `${pos}_${rt}_${sply_ty}`
  const cdnrMap: Map<string, GSTR1_CDNREntry> = new Map();
  const hsnMap: Map<string, GSTR1_HSNSummaryEntry> = new Map();

  let grossTurnover = 0;

  for (const inv of invoices) {
    // Only include invoices for this period
    const invPeriod = getPeriodKey(inv.invoiceDate);
    if (invPeriod !== fp) continue;

    grossTurnover += inv.grandTotal;

    // B2B Invoices (registered buyer)
    if (inv.recipientGSTIN && inv.recipientGSTIN !== 'URP') {
      const buyerGSTIN = inv.recipientGSTIN;
      if (!b2bMap.has(buyerGSTIN)) {
        b2bMap.set(buyerGSTIN, { ctin: buyerGSTIN, invoices: [] });
      }
      const entry = b2bMap.get(buyerGSTIN)!;
      entry.invoices.push({
        inum: inv.invoiceNumber,
        idt: formatDate(inv.invoiceDate),
        val: inv.grandTotal,
        pos: inv.placeOfSupplyCode,
        rchrg: inv.isReverseCharge ? 'Y' : 'N',
        inv_typ: mapInvType(inv.supplyType),
        etin: inv.ecoGSTIN,
        itms: buildB2BItems(inv),
      });
    } else {
      // B2C
      if (inv.isInterState && inv.grandTotal > 250000) {
        // B2C Large (Table 5)
        b2clList.push({
          pos: inv.placeOfSupplyCode,
          ospvl: sellerGSTIN,
          etin: inv.ecoGSTIN,
          inv: [{
            inum: inv.invoiceNumber,
            idt: formatDate(inv.invoiceDate),
            val: inv.grandTotal,
            itms: buildB2CLItems(inv),
          }],
        });
      } else {
        // B2C Small (Table 7) — consolidate by POS + rate + supply type
        for (const line of inv.lineItems) {
          const supplyType = inv.isInterState ? 'INTER' : 'INTRA';
          const key = `${inv.placeOfSupplyCode}_${line.gstRate}_${supplyType}`;
          if (!b2csMap.has(key)) {
            b2csMap.set(key, {
              sply_ty: supplyType,
              pos: inv.placeOfSupplyCode,
              etin: inv.ecoGSTIN,
              txval: 0, rt: line.gstRate,
              iamt: 0, camt: 0, samt: 0, csamt: 0,
              typ: inv.ecoGSTIN ? 'E' : undefined,
            });
          }
          const cs = b2csMap.get(key)!;
          cs.txval += line.taxableValue;
          cs.iamt = (cs.iamt ?? 0) + line.igstAmount;
          cs.camt = (cs.camt ?? 0) + line.cgstAmount;
          cs.samt = (cs.samt ?? 0) + line.sgstAmount;
          cs.csamt = (cs.csamt ?? 0) + line.cessAmount;
        }
      }
    }

    // HSN Summary (for all invoices)
    accumulateHSN(inv.lineItems, hsnMap, inv.isInterState);
  }

  // Credit Notes
  for (const cn of creditNotes) {
    const cnPeriod = getPeriodKey(cn.creditNoteDate);
    if (cnPeriod !== fp) continue;

    if (cn.recipientGSTIN && cn.recipientGSTIN !== 'URP') {
      const buyerGSTIN = cn.recipientGSTIN;
      if (!cdnrMap.has(buyerGSTIN)) {
        cdnrMap.set(buyerGSTIN, { ctin: buyerGSTIN, nt: [] });
      }
      const entry = cdnrMap.get(buyerGSTIN)!;
      entry.nt.push({
        ntty: 'C',
        nt_num: cn.creditNoteNumber,
        nt_dt: formatDate(cn.creditNoteDate),
        rsn: cn.reason,
        p_gst: 'N',
        val: cn.grandTotal,
        itms: cn.lineItems.map((l, idx) => ({
          num: idx + 1,
          itm_det: {
            txval: l.taxableValue,
            rt: (l.cgstRate + l.sgstRate + l.igstRate + l.utgstRate) * 2
              || l.igstRate,
            camt: l.cgstAmount || undefined,
            samt: l.sgstAmount || undefined,
            iamt: l.igstAmount || undefined,
            csamt: l.cessAmount || undefined,
          },
        })),
      });
    }
  }

  return {
    gstin: sellerGSTIN,
    fp,
    gt: Math.round(grossTurnover * 100) / 100,
    cur_gt: Math.round(grossTurnover * 100) / 100,
    b2b: Array.from(b2bMap.values()),
    b2cl: b2clList,
    b2cs: Array.from(b2csMap.values()).map(cs => ({
      ...cs,
      txval: Math.round(cs.txval * 100) / 100,
      iamt: Math.round((cs.iamt ?? 0) * 100) / 100,
      camt: Math.round((cs.camt ?? 0) * 100) / 100,
      samt: Math.round((cs.samt ?? 0) * 100) / 100,
      csamt: Math.round((cs.csamt ?? 0) * 100) / 100,
    })),
    cdnr: Array.from(cdnrMap.values()),
    hsn: { data: Array.from(hsnMap.values()) },
  };
}

// --- Helpers ---
function getPeriodKey(date: Date): string {
  return `${String(date.getMonth() + 1).padStart(2, '0')}${date.getFullYear()}`;
}

function formatDate(date: Date): string {
  const d = String(date.getDate()).padStart(2, '0');
  const m = String(date.getMonth() + 1).padStart(2, '0');
  const y = date.getFullYear();
  return `${d}-${m}-${y}`;
}

function mapInvType(supplyType: string): 'R' | 'SEWP' | 'SEWOP' | 'DE' {
  switch (supplyType) {
    case 'SEZWP': return 'SEWP';
    case 'SEZWOP': return 'SEWOP';
    case 'DEXP': return 'DE';
    default: return 'R';
  }
}

function buildB2BItems(inv: GSTInvoice) {
  // Group by GST rate (GSTR-1 groups line items by rate within an invoice)
  const rateGroups: Map<number, { txval: number; camt: number; samt: number; iamt: number; csamt: number }> = new Map();
  for (const line of inv.lineItems) {
    const key = line.gstRate;
    if (!rateGroups.has(key)) rateGroups.set(key, { txval: 0, camt: 0, samt: 0, iamt: 0, csamt: 0 });
    const g = rateGroups.get(key)!;
    g.txval += line.taxableValue;
    g.camt += line.cgstAmount;
    g.samt += line.sgstAmount;
    g.iamt += line.igstAmount;
    g.csamt += line.cessAmount;
  }
  return Array.from(rateGroups.entries()).map(([rt, g], idx) => ({
    num: idx + 1,
    itm_det: {
      txval: Math.round(g.txval * 100) / 100,
      rt,
      camt: g.camt ? Math.round(g.camt * 100) / 100 : undefined,
      samt: g.samt ? Math.round(g.samt * 100) / 100 : undefined,
      iamt: g.iamt ? Math.round(g.iamt * 100) / 100 : undefined,
      csamt: g.csamt ? Math.round(g.csamt * 100) / 100 : undefined,
    },
  }));
}

function buildB2CLItems(inv: GSTInvoice) {
  const rateGroups: Map<number, { txval: number; iamt: number; csamt: number }> = new Map();
  for (const line of inv.lineItems) {
    const key = line.gstRate;
    if (!rateGroups.has(key)) rateGroups.set(key, { txval: 0, iamt: 0, csamt: 0 });
    const g = rateGroups.get(key)!;
    g.txval += line.taxableValue;
    g.iamt += line.igstAmount;
    g.csamt += line.cessAmount;
  }
  return Array.from(rateGroups.entries()).map(([rt, g], idx) => ({
    num: idx + 1,
    itm_det: {
      txval: Math.round(g.txval * 100) / 100,
      rt,
      iamt: Math.round(g.iamt * 100) / 100,
      csamt: g.csamt ? Math.round(g.csamt * 100) / 100 : undefined,
    },
  }));
}

function accumulateHSN(
  lineItems: GSTInvoice['lineItems'],
  hsnMap: Map<string, GSTR1_HSNSummaryEntry>,
  isInterState: boolean
) {
  for (const line of lineItems) {
    const key = line.hsnCode;
    if (!hsnMap.has(key)) {
      hsnMap.set(key, {
        num: hsnMap.size + 1,
        hsn_sc: line.hsnCode,
        uqc: line.uqc,
        qty: 0, val: 0, txval: 0,
        iamt: 0, camt: 0, samt: 0, csamt: 0,
      });
    }
    const h = hsnMap.get(key)!;
    h.qty += line.quantity;
    h.val += line.totalAmount;
    h.txval += line.taxableValue;
    h.iamt += line.igstAmount;
    h.camt += line.cgstAmount;
    h.samt += line.sgstAmount + line.utgstAmount;
    h.csamt += line.cessAmount;
  }
}
```

---

## 9. GSTR-3B Data Preparation

```typescript
// src/gst/returns/gstr3b.ts

import { GSTInvoice, GSTCreditNote } from '../../types/gstInvoice';
import { ITCRecord } from '../../types/itc';

export interface GSTR3B_OutwardSupplies {
  osup_det: {          // 3.1(a) — Normal taxable supplies
    txval: number;
    iamt: number;
    camt: number;
    samt: number;
    csamt: number;
  };
  osup_zero: {         // 3.1(b) — Zero-rated (exports, SEZ)
    txval: number;
    iamt: number;
    camt: number;
    samt: number;
    csamt: number;
  };
  osup_nil_exmp: {     // 3.1(c) — Nil/exempt supplies
    txval: number;
  };
  isup_rev: {          // 3.1(d) — Inward supplies on reverse charge
    txval: number;
    iamt: number;
    camt: number;
    samt: number;
    csamt: number;
  };
  osup_nongst: {       // 3.1(e) — Non-GST outward supplies
    txval: number;
  };
}

// Table 3.1.1 — Section 9(5) e-commerce supplies (ECO fills)
export interface GSTR3B_EcomSupplies {
  eco_taxable: {       // 3.1.1(i) — ECO's liability
    txval: number;
    iamt: number;
    camt: number;
    samt: number;
    csamt: number;
  };
  eco_by_supplier: {   // 3.1.1(ii) — Supplier's portion (for supplier's return)
    txval: number;
  };
}

// Table 3.2 — Inter-state supplies (state-wise breakdown)
export interface GSTR3B_InterStateSupply {
  pos: string;         // Place of supply state code
  txval: number;
  iamt: number;
}

// Table 4 — ITC
export interface GSTR3B_ITC {
  itc_avl: {           // 4A — Available ITC
    iamt: number;      // 4A(1) Import of goods
    camt: number;      // 4A(1) Import of goods (CGST component)
    samt: number;
    csamt: number;
    others: {          // 4A(5) — All other ITC (domestic purchases)
      iamt: number;
      camt: number;
      samt: number;
      csamt: number;
    };
  };
  itc_rev: {           // 4B — ITC reversed
    rule42_43: {       // 4B(1) — Rule 38, 42, 43 + Section 17(5)
      iamt: number;
      camt: number;
      samt: number;
      csamt: number;
    };
    others: {          // 4B(2) — Other reversals
      iamt: number;
      camt: number;
      samt: number;
      csamt: number;
    };
  };
  net_itc: {           // 4C — Net ITC = 4A - 4B (auto-calculated)
    iamt: number;
    camt: number;
    samt: number;
    csamt: number;
  };
  itc_ineligible: {    // 4D(2) — Ineligible ITC (Section 16(4), PoS restrictions)
    iamt: number;
    camt: number;
    samt: number;
    csamt: number;
  };
}

// Table 5 — Exempt, nil, non-GST supplies received
export interface GSTR3B_ExemptInward {
  inter: number;       // Inter-state exempt/nil
  intra: number;       // Intra-state exempt/nil
}

// Table 6 — Tax payment summary
export interface GSTR3B_TaxPayment {
  tax: {
    igst: { paid_itc: number; paid_cash: number; total: number };
    cgst: { paid_itc: number; paid_cash: number; total: number };
    sgst: { paid_itc: number; paid_cash: number; total: number };
    cess: { paid_itc: number; paid_cash: number; total: number };
  };
  tds_tcs_credit: {    // 6.2 — TCS/TDS from ECO (Electronic Cash Ledger)
    igst: number;
    cgst: number;
    sgst: number;
    cess: number;
  };
}

export interface GSTR3BReport {
  gstin: string;
  fp: string;          // Filing period MMYYYY
  ret_period: string;
  inward_sup: GSTR3B_ExemptInward;
  sup_details: GSTR3B_OutwardSupplies;
  itc_elg: GSTR3B_ITC;
  intr_dtls: GSTR3B_InterStateSupply[]; // Table 3.2
  tax_pay: GSTR3B_TaxPayment;
}

export function computeGSTR3B(
  invoices: GSTInvoice[],
  creditNotes: GSTCreditNote[],
  purchaseITC: ITCRecord[],
  tcsCredits: { igst: number; cgst: number; sgst: number; cess: number },
  period: { month: number; year: number },
  sellerGSTIN: string
): GSTR3BReport {
  const fp = `${String(period.month).padStart(2, '0')}${period.year}`;
  const periodInvoices = invoices.filter(inv => {
    const d = inv.invoiceDate;
    return d.getMonth() + 1 === period.month && d.getFullYear() === period.year;
  });
  const periodCNs = creditNotes.filter(cn => {
    const d = cn.creditNoteDate;
    return d.getMonth() + 1 === period.month && d.getFullYear() === period.year;
  });

  // Compute net outward supplies (invoices minus credit notes)
  let osup_txval = 0, osup_igst = 0, osup_cgst = 0, osup_sgst = 0, osup_cess = 0;
  let zero_txval = 0, zero_igst = 0;
  let nil_txval = 0;
  const interStateMap: Map<string, { txval: number; iamt: number }> = new Map();

  for (const inv of periodInvoices) {
    if (inv.supplyType === 'EXPWP' || inv.supplyType === 'EXPWOP' ||
        inv.supplyType === 'SEZWP' || inv.supplyType === 'SEZWOP') {
      zero_txval += inv.totalTaxableValue;
      zero_igst += inv.totalIGST;
    } else {
      const hasGST = inv.totalTax > 0;
      if (hasGST) {
        osup_txval += inv.totalTaxableValue;
        osup_igst += inv.totalIGST;
        osup_cgst += inv.totalCGST;
        osup_sgst += inv.totalSGST + inv.totalUTGST;
        osup_cess += inv.totalCess;
        // Table 3.2: inter-state to unregistered (B2C inter-state)
        if (inv.isInterState && (!inv.recipientGSTIN || inv.recipientGSTIN === 'URP')) {
          if (!interStateMap.has(inv.placeOfSupplyCode)) {
            interStateMap.set(inv.placeOfSupplyCode, { txval: 0, iamt: 0 });
          }
          const m = interStateMap.get(inv.placeOfSupplyCode)!;
          m.txval += inv.totalTaxableValue;
          m.iamt += inv.totalIGST;
        }
      } else {
        nil_txval += inv.totalTaxableValue;
      }
    }
  }

  // Subtract credit notes
  for (const cn of periodCNs) {
    osup_txval -= cn.totalTaxableValue;
    osup_igst -= cn.totalIGST;
    osup_cgst -= cn.totalCGST;
    osup_sgst -= cn.totalSGST + cn.totalUTGST;
    osup_cess -= cn.totalCess;
  }

  // ITC from purchase register (GSTR-2B matched)
  const itcAvail = purchaseITC
    .filter(r => r.isEligible && !r.isReversed)
    .reduce(
      (acc, r) => ({
        iamt: acc.iamt + r.igstAmount,
        camt: acc.camt + r.cgstAmount,
        samt: acc.samt + r.sgstAmount,
        csamt: acc.csamt + r.cessAmount,
      }),
      { iamt: 0, camt: 0, samt: 0, csamt: 0 }
    );

  const itcReversed = purchaseITC
    .filter(r => r.isReversed)
    .reduce(
      (acc, r) => ({
        iamt: acc.iamt + r.igstAmount,
        camt: acc.camt + r.cgstAmount,
        samt: acc.samt + r.sgstAmount,
        csamt: acc.csamt + r.cessAmount,
      }),
      { iamt: 0, camt: 0, samt: 0, csamt: 0 }
    );

  const netITC = {
    iamt: itcAvail.iamt - itcReversed.iamt,
    camt: itcAvail.camt - itcReversed.camt,
    samt: itcAvail.samt - itcReversed.samt,
    csamt: itcAvail.csamt - itcReversed.csamt,
  };

  // ITC utilization: IGST first → CGST → SGST
  const { igstPayable, cgstPayable, sgstPayable, cessPayable } =
    computeTaxPayable(osup_igst, osup_cgst, osup_sgst, osup_cess, netITC);

  return {
    gstin: sellerGSTIN,
    fp,
    ret_period: fp,
    sup_details: {
      osup_det: {
        txval: round2(osup_txval), iamt: round2(osup_igst),
        camt: round2(osup_cgst), samt: round2(osup_sgst), csamt: round2(osup_cess),
      },
      osup_zero: {
        txval: round2(zero_txval), iamt: round2(zero_igst),
        camt: 0, samt: 0, csamt: 0,
      },
      osup_nil_exmp: { txval: round2(nil_txval) },
      isup_rev: { txval: 0, iamt: 0, camt: 0, samt: 0, csamt: 0 },
      osup_nongst: { txval: 0 },
    },
    itc_elg: {
      itc_avl: {
        iamt: round2(itcAvail.iamt), camt: round2(itcAvail.camt),
        samt: round2(itcAvail.samt), csamt: round2(itcAvail.csamt),
        others: {
          iamt: round2(itcAvail.iamt), camt: round2(itcAvail.camt),
          samt: round2(itcAvail.samt), csamt: round2(itcAvail.csamt),
        },
      },
      itc_rev: {
        rule42_43: {
          iamt: round2(itcReversed.iamt), camt: round2(itcReversed.camt),
          samt: round2(itcReversed.samt), csamt: round2(itcReversed.csamt),
        },
        others: { iamt: 0, camt: 0, samt: 0, csamt: 0 },
      },
      net_itc: {
        iamt: round2(netITC.iamt), camt: round2(netITC.camt),
        samt: round2(netITC.samt), csamt: round2(netITC.csamt),
      },
      itc_ineligible: { iamt: 0, camt: 0, samt: 0, csamt: 0 },
    },
    intr_dtls: Array.from(interStateMap.entries()).map(([pos, v]) => ({
      pos,
      txval: round2(v.txval),
      iamt: round2(v.iamt),
    })),
    inward_sup: { inter: 0, intra: 0 },
    tax_pay: {
      tax: {
        igst: { paid_itc: round2(igstPayable.itc), paid_cash: round2(igstPayable.cash), total: round2(osup_igst) },
        cgst: { paid_itc: round2(cgstPayable.itc), paid_cash: round2(cgstPayable.cash), total: round2(osup_cgst) },
        sgst: { paid_itc: round2(sgstPayable.itc), paid_cash: round2(sgstPayable.cash), total: round2(osup_sgst) },
        cess: { paid_itc: round2(cessPayable.itc), paid_cash: round2(cessPayable.cash), total: round2(osup_cess) },
      },
      tds_tcs_credit: tcsCredits,
    },
  };
}

// ITC Utilization Order: IGST credit → IGST, then CGST, then SGST
// CGST credit → CGST, then IGST
// SGST credit → SGST, then IGST
function computeTaxPayable(
  igstLiability: number,
  cgstLiability: number,
  sgstLiability: number,
  cessLiability: number,
  netITC: { iamt: number; camt: number; samt: number; csamt: number }
) {
  let igstITC = netITC.iamt;
  let cgstITC = netITC.camt;
  let sgstITC = netITC.samt;
  let cessITC = netITC.csamt;

  // 1. Use IGST ITC → offset IGST first
  const igstFromIGST = Math.min(igstITC, igstLiability);
  igstITC -= igstFromIGST;
  let igstRemaining = igstLiability - igstFromIGST;

  // 2. Use CGST ITC → offset CGST first
  const cgstFromCGST = Math.min(cgstITC, cgstLiability);
  cgstITC -= cgstFromCGST;
  let cgstRemaining = cgstLiability - cgstFromCGST;

  // 3. Use SGST ITC → offset SGST first
  const sgstFromSGST = Math.min(sgstITC, sgstLiability);
  sgstITC -= sgstFromSGST;
  let sgstRemaining = sgstLiability - sgstFromSGST;

  // 4. Use remaining IGST ITC → offset CGST then SGST
  const cgstFromIGST = Math.min(igstITC, cgstRemaining);
  igstITC -= cgstFromIGST;
  cgstRemaining -= cgstFromIGST;

  const sgstFromIGST = Math.min(igstITC, sgstRemaining);
  igstITC -= sgstFromIGST;
  sgstRemaining -= sgstFromIGST;

  // 5. Use remaining CGST ITC → offset remaining IGST
  const igstFromCGST = Math.min(cgstITC, igstRemaining);
  cgstITC -= igstFromCGST;
  igstRemaining -= igstFromCGST;

  // 6. Use remaining SGST ITC → offset remaining IGST
  const igstFromSGST = Math.min(sgstITC, igstRemaining);
  sgstITC -= igstFromSGST;
  igstRemaining -= igstFromSGST;

  // 7. Cess: only from cess ITC
  const cessFromCess = Math.min(cessITC, cessLiability);
  const cessRemaining = cessLiability - cessFromCess;

  return {
    igstPayable: {
      itc: igstFromIGST + igstFromCGST + igstFromSGST,
      cash: Math.max(0, igstRemaining),
    },
    cgstPayable: {
      itc: cgstFromCGST + cgstFromIGST,
      cash: Math.max(0, cgstRemaining),
    },
    sgstPayable: {
      itc: sgstFromSGST + sgstFromIGST,
      cash: Math.max(0, sgstRemaining),
    },
    cessPayable: {
      itc: cessFromCess,
      cash: Math.max(0, cessRemaining),
    },
  };
}

function round2(n: number): number {
  return Math.round(n * 100) / 100;
}
```

---

## 10. E-Invoicing Module

### 10.1 When E-Invoicing Applies

```
AATO > ₹5 crore AND:
  - B2B invoice (buyer has GSTIN)
  - B2G invoice (buyer is government)
  - Export invoice (EXPWP, EXPWOP)
  - SEZ supply (SEZWP, SEZWOP)
  - Credit notes against above
  - Debit notes against above

NOT applicable:
  - B2C invoices
  - Composition dealers
  - SEZ units (as supplier)
  - Insurance, banking, NBFC companies
  - Goods Transport Agencies
```

### 10.2 IRN Generation & IRP API Integration

```typescript
// src/gst/einvoice/einvoice.ts

import * as crypto from 'crypto';

export interface IRPCredentials {
  clientId: string;
  clientSecret: string;
  username: string;
  password: string; // plain text — will be RSA encrypted before sending
  irpBaseUrl: string; // e.g., 'https://einvoice1.gst.gov.in' (production)
}

export interface IRPAuthToken {
  authToken: string;
  sek: string;       // AES-256 encrypted session key (decrypt with AppKey)
  tokenExpiry: Date;
  appKey: string;    // 32-char random key used to decrypt SEK
}

export interface EInvoiceJSON {
  Version: '1.1';
  TranDtls: {
    TaxSch: 'GST';
    SupTyp: 'B2B' | 'SEZWP' | 'SEZWOP' | 'EXPWP' | 'EXPWOP' | 'DEXP';
    RegRev: 'Y' | 'N';
    EcmGstin?: string;
    IgstOnIntra?: 'Y' | 'N';
  };
  DocDtls: {
    Typ: 'INV' | 'CRN' | 'DBN';
    No: string;       // Invoice number (max 16 chars, uppercase)
    Dt: string;       // DD/MM/YYYY
  };
  SellerDtls: {
    Gstin: string;
    LglNm: string;
    TrdNm?: string;
    Addr1: string;
    Addr2?: string;
    Loc: string;
    Pin: number;
    Stcd: string;     // 2-digit state code
    Ph?: string;
    Em?: string;
  };
  BuyerDtls: {
    Gstin: string;    // 'URP' for unregistered
    LglNm: string;
    TrdNm?: string;
    Pos: string;      // Place of supply state code
    Addr1: string;
    Addr2?: string;
    Loc: string;
    Pin: number;
    Stcd: string;
    Ph?: string;
    Em?: string;
  };
  DispDtls?: {        // Dispatch from (if different from seller)
    Nm: string;
    Addr1: string;
    Addr2?: string;
    Loc: string;
    Pin: number;
    Stcd: string;
  };
  ShipDtls?: {        // Ship to (if different from buyer)
    Gstin?: string;
    LglNm?: string;
    TrdNm?: string;
    Addr1: string;
    Addr2?: string;
    Loc: string;
    Pin: number;
    Stcd: string;
  };
  ItemList: Array<{
    SlNo: string;
    PrdDesc: string;
    IsServc: 'Y' | 'N';
    HsnCd: string;
    Qty?: number;
    Unit?: string;       // UQC
    UnitPrice: number;
    TotAmt: number;      // Gross amount (qty × unitPrice)
    Discount?: number;
    AssAmt: number;      // Assessable/taxable amount
    GstRt: number;       // Total GST rate
    IgstAmt?: number;
    CgstAmt?: number;
    SgstAmt?: number;
    CesRt?: number;
    CesAmt?: number;
    TotItemVal: number;  // AssAmt + all taxes
  }>;
  ValDtls: {
    AssVal: number;      // Total assessable value
    CgstVal: number;
    SgstVal: number;
    IgstVal: number;
    CesVal: number;
    Discount?: number;
    OthChrg?: number;
    RndOffAmt?: number;
    TotInvVal: number;   // Grand total
    TotInvValFc?: number; // Foreign currency total (for exports)
  };
  PrecDocDtls?: Array<{  // For credit/debit notes
    InvNo: string;
    InvDt: string;       // DD/MM/YYYY
    OthRefNo?: string;
  }>;
  EwbDtls?: {            // E-way bill details
    TransId?: string;    // Transporter GSTIN
    TransName?: string;
    TransMode?: '1' | '2' | '3' | '4'; // 1=Road, 2=Rail, 3=Air, 4=Ship
    TransDocDt?: string; // DD/MM/YYYY
    TransDocNo?: string;
    VehNo?: string;
    VehType?: 'R' | 'O'; // R=Regular, O=ODC
    Distance: number;    // KMs
  };
  // Filled by IRP:
  Irn?: string;
  AckNo?: number;
  AckDt?: string;
  SignedInvoice?: string;
  SignedQRCode?: string;
}

// Build e-invoice JSON from internal GSTInvoice
export function buildEInvoiceJSON(inv: GSTInvoice): EInvoiceJSON {
  const supTyp = mapSupplyTypeToEInv(inv.supplyType);
  const docTyp = mapInvoiceTypeToEInv(inv.invoiceType);

  const einv: EInvoiceJSON = {
    Version: '1.1',
    TranDtls: {
      TaxSch: 'GST',
      SupTyp: supTyp,
      RegRev: inv.isReverseCharge ? 'Y' : 'N',
      EcmGstin: inv.ecoGSTIN,
      IgstOnIntra: 'N',
    },
    DocDtls: {
      Typ: docTyp,
      No: inv.invoiceNumber.toUpperCase(),
      Dt: formatDateDDMMYYYY(inv.invoiceDate),
    },
    SellerDtls: {
      Gstin: inv.supplierGSTIN,
      LglNm: inv.supplierLegalName.substring(0, 100),
      TrdNm: inv.supplierTradeName?.substring(0, 100),
      Addr1: inv.supplierAddress1.substring(0, 100),
      Addr2: inv.supplierAddress2?.substring(0, 100),
      Loc: inv.supplierCity.substring(0, 50),
      Pin: parseInt(inv.supplierPincode) || 0,
      Stcd: inv.supplierStateCode,
    },
    BuyerDtls: {
      Gstin: inv.recipientGSTIN,
      LglNm: inv.recipientName.substring(0, 100),
      Pos: inv.placeOfSupplyCode,
      Addr1: inv.recipientAddress1.substring(0, 100),
      Addr2: inv.recipientAddress2?.substring(0, 100),
      Loc: inv.recipientCity.substring(0, 100),
      Pin: parseInt(inv.recipientPincode) || 0,
      Stcd: inv.recipientStateCode,
    },
    ItemList: inv.lineItems.map((line, idx) => ({
      SlNo: String(idx + 1),
      PrdDesc: line.productDescription.substring(0, 300),
      IsServc: 'N',
      HsnCd: line.hsnCode,
      Qty: line.quantity,
      Unit: line.uqc ?? 'NOS',
      UnitPrice: round2(line.unitPrice),
      TotAmt: round2(line.grossValue),
      Discount: line.discount > 0 ? round2(line.discount) : undefined,
      AssAmt: round2(line.taxableValue),
      GstRt: line.gstRate,
      IgstAmt: line.igstAmount > 0 ? round2(line.igstAmount) : undefined,
      CgstAmt: line.cgstAmount > 0 ? round2(line.cgstAmount) : undefined,
      SgstAmt: line.sgstAmount > 0 ? round2(line.sgstAmount) : undefined,
      CesRt: line.cessRate > 0 ? line.cessRate : undefined,
      CesAmt: line.cessAmount > 0 ? round2(line.cessAmount) : undefined,
      TotItemVal: round2(line.totalAmount),
    })),
    ValDtls: {
      AssVal: round2(inv.totalTaxableValue),
      CgstVal: round2(inv.totalCGST),
      SgstVal: round2(inv.totalSGST),
      IgstVal: round2(inv.totalIGST),
      CesVal: round2(inv.totalCess),
      RndOffAmt: inv.roundOff || undefined,
      TotInvVal: round2(inv.grandTotal),
    },
  };

  // Ship-to details when delivery differs from billing
  if (inv.deliveryAddress1 && inv.deliveryStateCode) {
    einv.ShipDtls = {
      Addr1: inv.deliveryAddress1.substring(0, 100),
      Addr2: inv.deliveryAddress2?.substring(0, 100),
      Loc: inv.deliveryCity?.substring(0, 100) ?? '',
      Pin: parseInt(inv.deliveryPincode ?? '0') || 0,
      Stcd: inv.deliveryStateCode,
    };
  }

  return einv;
}

// Compute IRN locally for pre-validation (IRP computes authoritative IRN)
export function computeLocalIRN(
  supplierGSTIN: string,
  financialYear: string,   // e.g., "2025-26"
  documentType: 'INV' | 'CRN' | 'DBN',
  documentNumber: string
): string {
  const input = `${supplierGSTIN}${financialYear}${documentType}${documentNumber.toUpperCase()}`;
  return crypto.createHash('sha256').update(input).digest('hex');
}

// IRP API integration
export class IRPClient {
  private authToken: IRPAuthToken | null = null;
  private readonly creds: IRPCredentials;

  constructor(creds: IRPCredentials) {
    this.creds = creds;
  }

  // Authenticate with IRP and get session token
  async authenticate(): Promise<IRPAuthToken> {
    // Generate 32-char random app key for AES session key exchange
    const appKey = crypto.randomBytes(16).toString('hex'); // 32 hex chars

    // In production: RSA-encrypt password and appKey with IRP's public key
    // For simplicity shown as plaintext here — wrap with node-forge or node-rsa
    const encPassword = this.rsaEncrypt(this.creds.password, IRP_PUBLIC_KEY);
    const encAppKey = this.rsaEncrypt(appKey, IRP_PUBLIC_KEY);

    const response = await fetch(`${this.creds.irpBaseUrl}/api/auth`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'client_id': this.creds.clientId,
        'client_secret': this.creds.clientSecret,
      },
      body: JSON.stringify({
        data: {
          UserName: this.creds.username,
          Password: encPassword,
          AppKey: encAppKey,
          ForceRefreshAccessToken: false,
        },
      }),
    });

    const json: any = await response.json();
    if (json.Status !== '1') {
      throw new Error(`IRP Auth failed: ${JSON.stringify(json.ErrorDetails)}`);
    }

    // Decrypt SEK (Session Encryption Key) using appKey
    const sek = this.aesDecrypt(json.Data.Sek, appKey);

    const token: IRPAuthToken = {
      authToken: json.Data.AuthToken,
      sek,
      tokenExpiry: new Date(json.Data.TokenExpiry),
      appKey,
    };
    this.authToken = token;
    return token;
  }

  // Generate IRN by uploading e-invoice JSON to IRP
  async generateIRN(
    einvJSON: EInvoiceJSON,
    sellerGSTIN: string
  ): Promise<{
    irn: string;
    ackNo: number;
    ackDt: string;
    signedInvoice: string;
    signedQRCode: string;
    ewbNo?: number;
  }> {
    if (!this.authToken || new Date() > this.authToken.tokenExpiry) {
      await this.authenticate();
    }

    const token = this.authToken!;

    // Encrypt invoice JSON with SEK (AES-256)
    const invoiceJSON = JSON.stringify(einvJSON);
    const encryptedData = this.aesEncrypt(invoiceJSON, token.sek);
    const encodedData = Buffer.from(encryptedData).toString('base64');

    const response = await fetch(`${this.creds.irpBaseUrl}/api/Invoice/Generate`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'client_id': this.creds.clientId,
        'client_secret': this.creds.clientSecret,
        'Gstin': sellerGSTIN,
        'user_name': this.creds.username,
        'AuthToken': token.authToken,
      },
      body: JSON.stringify({ Data: encodedData }),
    });

    const json: any = await response.json();
    if (json.Status !== '1') {
      throw new Error(`IRN generation failed: ${JSON.stringify(json.ErrorDetails)}`);
    }

    return {
      irn: json.Data.Irn,
      ackNo: json.Data.AckNo,
      ackDt: json.Data.AckDt,
      signedInvoice: json.Data.SignedInvoice,
      signedQRCode: json.Data.SignedQRCode,
      ewbNo: json.Data.EwbNo,
    };
  }

  // Cancel an IRN (within 24 hours of generation)
  async cancelIRN(
    irn: string,
    cancelReason: 1 | 2 | 3 | 4, // 1=Duplicate, 2=Order cancelled, 3=Data entry error, 4=Others
    cancelRemarks: string,
    sellerGSTIN: string
  ): Promise<{ cancelDate: string }> {
    if (!this.authToken || new Date() > this.authToken.tokenExpiry) {
      await this.authenticate();
    }
    const token = this.authToken!;

    const payload = JSON.stringify({ Irn: irn, CnlRsn: cancelReason, CnlRem: cancelRemarks });
    const encPayload = this.aesEncrypt(payload, token.sek);
    const encodedPayload = Buffer.from(encPayload).toString('base64');

    const response = await fetch(`${this.creds.irpBaseUrl}/api/Invoice/Cancel`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'client_id': this.creds.clientId,
        'client_secret': this.creds.clientSecret,
        'Gstin': sellerGSTIN,
        'user_name': this.creds.username,
        'AuthToken': token.authToken,
      },
      body: JSON.stringify({ Data: encodedPayload }),
    });

    const json: any = await response.json();
    if (json.Status !== '1') {
      throw new Error(`IRN cancellation failed: ${JSON.stringify(json.ErrorDetails)}`);
    }
    return { cancelDate: json.Data.CancelDate };
  }

  // Get IRN details by document (for lookup/verification)
  async getIRNByDocument(
    documentNumber: string,
    documentType: 'INV' | 'CRN' | 'DBN',
    documentDate: string, // DD/MM/YYYY
    sellerGSTIN: string
  ): Promise<EInvoiceJSON & { Irn: string }> {
    if (!this.authToken || new Date() > this.authToken.tokenExpiry) {
      await this.authenticate();
    }
    const token = this.authToken!;

    const response = await fetch(
      `${this.creds.irpBaseUrl}/api/Invoice/GetIRNByDocDetails` +
      `?DocType=${documentType}&DocNo=${documentNumber}&DocDate=${documentDate}`,
      {
        method: 'GET',
        headers: {
          'client_id': this.creds.clientId,
          'client_secret': this.creds.clientSecret,
          'Gstin': sellerGSTIN,
          'user_name': this.creds.username,
          'AuthToken': token.authToken,
        },
      }
    );

    const json: any = await response.json();
    if (json.Status !== '1') {
      throw new Error(`GetIRN failed: ${JSON.stringify(json.ErrorDetails)}`);
    }
    return json.Data;
  }

  // Stubs for crypto operations (replace with actual RSA/AES library calls)
  private rsaEncrypt(data: string, publicKey: string): string {
    // Use: import { publicEncrypt } from 'crypto'; or 'node-forge' for RSA PKCS1v15
    // const encrypted = crypto.publicEncrypt({ key: publicKey, padding: crypto.constants.RSA_PKCS1_PADDING }, Buffer.from(data));
    // return encrypted.toString('base64');
    throw new Error('Implement RSA encryption with IRP public key (PKCS#1 v1.5)');
  }

  private aesDecrypt(encryptedBase64: string, key: string): string {
    // SEK is AES-256-ECB encrypted with AppKey
    // const decipher = crypto.createDecipheriv('aes-256-ecb', Buffer.from(key, 'hex'), null);
    // const decrypted = Buffer.concat([decipher.update(Buffer.from(encryptedBase64, 'base64')), decipher.final()]);
    // return decrypted.toString('utf8');
    throw new Error('Implement AES-256 decryption for SEK');
  }

  private aesEncrypt(data: string, sek: string): string {
    // Invoice data encrypted with SEK (AES-256-ECB or CBC per IRP spec)
    throw new Error('Implement AES-256 encryption for invoice data');
  }
}

// IRP Production Public Key (NIC IRP-1) — for RSA encryption
// Fetch current key from: https://einvoice1.gst.gov.in/Others/OtherService
const IRP_PUBLIC_KEY = `-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAmu2RnFO3TuI/SKc0Fxl6
... (fetch current key from IRP portal)
-----END PUBLIC KEY-----`;

// 30-day upload limit validation
export function validateIRNUploadWindow(invoiceDate: Date, currentDate: Date = new Date()): {
  isWithinWindow: boolean;
  daysSinceInvoice: number;
  daysRemaining: number;
} {
  const msPerDay = 24 * 60 * 60 * 1000;
  const daysSince = Math.floor((currentDate.getTime() - invoiceDate.getTime()) / msPerDay);
  const daysRemaining = Math.max(0, 30 - daysSince);
  return {
    isWithinWindow: daysSince <= 30,
    daysSinceInvoice: daysSince,
    daysRemaining,
  };
}

// 24-hour cancellation window check
export function canCancelIRN(irnGenerationDate: Date, currentDate: Date = new Date()): boolean {
  const msPerHour = 60 * 60 * 1000;
  const hoursSince = (currentDate.getTime() - irnGenerationDate.getTime()) / msPerHour;
  return hoursSince <= 24;
}

// --- Helpers ---
function mapSupplyTypeToEInv(supplyType: string): EInvoiceJSON['TranDtls']['SupTyp'] {
  const map: Record<string, EInvoiceJSON['TranDtls']['SupTyp']> = {
    B2B: 'B2B', SEZWP: 'SEZWP', SEZWOP: 'SEZWOP',
    EXPWP: 'EXPWP', EXPWOP: 'EXPWOP', DEXP: 'DEXP',
  };
  return map[supplyType] ?? 'B2B';
}

function mapInvoiceTypeToEInv(invoiceType: string): 'INV' | 'CRN' | 'DBN' {
  if (invoiceType === 'CREDIT_NOTE') return 'CRN';
  if (invoiceType === 'DEBIT_NOTE') return 'DBN';
  return 'INV';
}

function formatDateDDMMYYYY(date: Date): string {
  const d = String(date.getDate()).padStart(2, '0');
  const m = String(date.getMonth() + 1).padStart(2, '0');
  return `${d}/${m}/${date.getFullYear()}`;
}

function round2(n: number): number {
  return Math.round(n * 100) / 100;
}
```

---

## 11. TCS/TDS Handling

### 11.1 Section 52 TCS (by E-Commerce Operators)

```typescript
// src/gst/tcs/tcsEngine.ts

export interface TCSRecord {
  id: string;
  marketplaceGSTIN: string;    // ECO's GSTIN (e.g., Amazon India GSTIN)
  marketplaceName: string;     // e.g., 'Amazon', 'Flipkart', 'Meesho'
  sellerGSTIN: string;
  month: number;               // 1-12
  year: number;
  settlementPeriod: string;    // e.g., "March 2026" from settlement statement
  grossSalesValue: number;     // Gross taxable value before returns for the month
  returnsValue: number;        // Returns/cancellations value
  netTaxableValue: number;     // grossSalesValue - returnsValue
  tcsRate: number;             // 1% (0.5% CGST + 0.5% SGST or 1% IGST)
  tcsCGST: number;             // 0.5% of net (for intra-state sales)
  tcsSGST: number;             // 0.5% of net (for intra-state sales)
  tcsIGST: number;             // 1% of net (for inter-state sales)
  totalTCS: number;            // tcsCGST + tcsSGST + tcsIGST
  creditedToCashLedger: boolean;
  gstr8Reference?: string;     // ECO's GSTR-8 filing reference
  reconciliationStatus: 'PENDING' | 'MATCHED' | 'DISCREPANCY';
  discrepancyNotes?: string;
  createdAt: Date;
}

export function computeTCSRecord(
  settlement: {
    marketplaceGSTIN: string;
    marketplaceName: string;
    sellerGSTIN: string;
    month: number;
    year: number;
    grossSales: { intraState: number; interState: number };
    returns: { intraState: number; interState: number };
  }
): TCSRecord {
  const netIntra = Math.max(0, settlement.grossSales.intraState - settlement.returns.intraState);
  const netInter = Math.max(0, settlement.grossSales.interState - settlement.returns.interState);
  const netTotal = netIntra + netInter;

  // TCS: 0.5% CGST + 0.5% SGST on intra-state; 1% IGST on inter-state
  const tcsCGST = round2(netIntra * 0.005);
  const tcsSGST = round2(netIntra * 0.005);
  const tcsIGST = round2(netInter * 0.01);

  return {
    id: `TCS-${settlement.marketplaceGSTIN}-${settlement.month}-${settlement.year}`,
    marketplaceGSTIN: settlement.marketplaceGSTIN,
    marketplaceName: settlement.marketplaceName,
    sellerGSTIN: settlement.sellerGSTIN,
    month: settlement.month,
    year: settlement.year,
    settlementPeriod: `${settlement.month}/${settlement.year}`,
    grossSalesValue: settlement.grossSales.intraState + settlement.grossSales.interState,
    returnsValue: settlement.returns.intraState + settlement.returns.interState,
    netTaxableValue: netTotal,
    tcsRate: 1,
    tcsCGST, tcsSGST, tcsIGST,
    totalTCS: tcsCGST + tcsSGST + tcsIGST,
    creditedToCashLedger: false, // Updated when ECO files GSTR-8 and credit appears
    reconciliationStatus: 'PENDING',
    createdAt: new Date(),
  };
}

// Reconcile TCS credits: compare seller's records vs ECO's GSTR-8 data
export function reconcileTCS(
  sellerTCSRecords: TCSRecord[],
  gstr8Data: Array<{
    marketplaceGSTIN: string;
    sellerGSTIN: string;
    period: string; // MMYYYY
    igstTCS: number;
    cgstTCS: number;
    sgstTCS: number;
  }>
): TCSRecord[] {
  return sellerTCSRecords.map(record => {
    const period = `${String(record.month).padStart(2, '0')}${record.year}`;
    const match = gstr8Data.find(
      g => g.marketplaceGSTIN === record.marketplaceGSTIN &&
           g.sellerGSTIN === record.sellerGSTIN &&
           g.period === period
    );

    if (!match) {
      return {
        ...record,
        reconciliationStatus: 'DISCREPANCY' as const,
        discrepancyNotes: 'Not found in GSTR-8 data from ECO',
      };
    }

    const tolerance = 1; // ₹1 rounding tolerance
    const cgstMatch = Math.abs(match.cgstTCS - record.tcsCGST) <= tolerance;
    const sgstMatch = Math.abs(match.sgstTCS - record.tcsSGST) <= tolerance;
    const igstMatch = Math.abs(match.igstTCS - record.tcsIGST) <= tolerance;

    if (cgstMatch && sgstMatch && igstMatch) {
      return {
        ...record,
        creditedToCashLedger: true,
        reconciliationStatus: 'MATCHED' as const,
      };
    } else {
      return {
        ...record,
        reconciliationStatus: 'DISCREPANCY' as const,
        discrepancyNotes: `Mismatch: Seller CGST=${record.tcsCGST}, GSTR-8 CGST=${match.cgstTCS}; ` +
          `Seller SGST=${record.tcsSGST}, GSTR-8 SGST=${match.sgstTCS}; ` +
          `Seller IGST=${record.tcsIGST}, GSTR-8 IGST=${match.igstTCS}`,
      };
    }
  });
}

// --- Section 194-O Income Tax TDS ---
export interface TDSRecord {
  id: string;
  ecoGSTIN: string;            // E-commerce operator's GSTIN
  ecoName: string;
  sellerPAN: string;
  financialYear: string;       // e.g., "2025-26"
  quarter: 1 | 2 | 3 | 4;
  grossSalesValue: number;     // Total gross sales facilitated in quarter
  tdsRate: number;             // 0.1% (with PAN); 5% (without PAN)
  tdsAmount: number;           // tdsRate × grossSalesValue / 100
  tdsDeductedDate?: Date;
  form26QEReference?: string;  // ECO's quarterly TDS return reference
  creditInForm26AS: boolean;   // Whether TDS appears in seller's 26AS
  reconciliationStatus: 'PENDING' | 'MATCHED' | 'DISCREPANCY';
  createdAt: Date;
}

export function computeIncomeTaxTDS(
  ecoGSTIN: string,
  ecoName: string,
  sellerPAN: string,
  hasPAN: boolean,
  grossSales: number,
  quarter: 1 | 2 | 3 | 4,
  financialYear: string
): TDSRecord {
  // Section 194-O: 0.1% with PAN (effective 1 Oct 2024), 5% without PAN
  // Threshold: ₹5 lakh/FY for individuals/HUF; no threshold for companies
  const tdsRate = hasPAN ? 0.1 : 5;
  const tdsAmount = round2(grossSales * tdsRate / 100);

  return {
    id: `TDS-${ecoGSTIN}-${sellerPAN}-Q${quarter}-${financialYear}`,
    ecoGSTIN, ecoName, sellerPAN, financialYear, quarter,
    grossSalesValue: grossSales,
    tdsRate, tdsAmount,
    creditInForm26AS: false,
    reconciliationStatus: 'PENDING',
    createdAt: new Date(),
  };
}

function round2(n: number): number {
  return Math.round(n * 100) / 100;
}
```

---

## 12. Input Tax Credit (ITC) Engine

### 12.1 Section 16 Four Conditions

All four must be satisfied to claim ITC on any purchase:
1. **Tax invoice or debit note** from supplier (Rule 46/47 compliant)
2. **Receipt of goods or services** (delivery confirmed)
3. **Tax paid by supplier** to government (confirmed via GSTR-2B)
4. **Own GSTR-3B filed** for the period

### 12.2 GSTR-2B Matching & ITC Engine

```typescript
// src/gst/itc/itcEngine.ts

export interface PurchaseRecord {
  id: string;
  invoiceNumber: string;
  invoiceDate: Date;
  supplierGSTIN: string;
  supplierName: string;
  igstAmount: number;
  cgstAmount: number;
  sgstAmount: number;
  cessAmount: number;
  taxableValue: number;
  category: ITCCategory;
  description: string;
  isReceivedInBooks: boolean;     // Goods/services actually received
  isTaxPaidBySupplier?: boolean;  // Per GSTR-2B
  isInGSTR2B: boolean;
  gstr2bPeriod?: string;          // MMYYYY when appeared in GSTR-2B
}

export type ITCCategory =
  | 'INVENTORY'           // Goods for resale → eligible
  | 'PACKAGING'           // Packaging materials → eligible
  | 'MARKETPLACE_FEE'     // Amazon/Flipkart commission → eligible
  | 'LOGISTICS'           // Shipping/courier → eligible
  | 'OFFICE_SUPPLIES'     // Office stationery → eligible
  | 'CAPITAL_GOODS'       // Machinery, equipment → eligible (Rule 43)
  | 'MOTOR_VEHICLE'       // Cars/2W/3W (personal) → BLOCKED Section 17(5)(a)
  | 'FOOD_ENTERTAINMENT'  // Food, club membership → BLOCKED Section 17(5)(b)
  | 'CONSTRUCTION'        // Building work → BLOCKED Section 17(5)(c)/(d)
  | 'PERSONAL_USE'        // Personal items → BLOCKED Section 17(5)(g)
  | 'FREE_SAMPLES'        // Gifts/free samples → BLOCKED Section 17(5)(h)
  | 'HEALTH_BEAUTY'       // Health/beauty treatment → BLOCKED Section 17(5)(b)
  | 'MIXED_USE';          // Used for both taxable & exempt → partial (Rule 42)

export const BLOCKED_ITC_CATEGORIES: ITCCategory[] = [
  'MOTOR_VEHICLE', 'FOOD_ENTERTAINMENT', 'CONSTRUCTION',
  'PERSONAL_USE', 'FREE_SAMPLES', 'HEALTH_BEAUTY',
];

export interface ITCRecord {
  purchaseId: string;
  supplierGSTIN: string;
  invoiceNumber: string;
  invoiceDate: Date;
  igstAmount: number;
  cgstAmount: number;
  sgstAmount: number;
  cessAmount: number;
  category: ITCCategory;
  isEligible: boolean;
  isBlocked: boolean;
  blockingSection?: string;    // e.g., "Section 17(5)(b)"
  isReversed: boolean;
  reversalReason?: string;     // Rule 42/43 reversal, or other
  reversalAmount?: number;
  isInGSTR2B: boolean;
  claimedInPeriod?: string;    // MMYYYY when claimed in GSTR-3B
  itcTimeLimitExpiry?: Date;   // 30 Nov following FY end
  notes?: string;
}

// Determine ITC eligibility for a purchase
export function assessITCEligibility(purchase: PurchaseRecord): ITCRecord {
  // Check blocked categories first
  if (BLOCKED_ITC_CATEGORIES.includes(purchase.category)) {
    return {
      purchaseId: purchase.id,
      supplierGSTIN: purchase.supplierGSTIN,
      invoiceNumber: purchase.invoiceNumber,
      invoiceDate: purchase.invoiceDate,
      igstAmount: purchase.igstAmount,
      cgstAmount: purchase.cgstAmount,
      sgstAmount: purchase.sgstAmount,
      cessAmount: purchase.cessAmount,
      category: purchase.category,
      isEligible: false,
      isBlocked: true,
      blockingSection: getBlockingSection(purchase.category),
      isReversed: false,
      isInGSTR2B: purchase.isInGSTR2B,
    };
  }

  // Must appear in GSTR-2B (Rule 36(4) — no provisional ITC)
  if (!purchase.isInGSTR2B) {
    return {
      purchaseId: purchase.id,
      supplierGSTIN: purchase.supplierGSTIN,
      invoiceNumber: purchase.invoiceNumber,
      invoiceDate: purchase.invoiceDate,
      igstAmount: purchase.igstAmount,
      cgstAmount: purchase.cgstAmount,
      sgstAmount: purchase.sgstAmount,
      cessAmount: purchase.cessAmount,
      category: purchase.category,
      isEligible: false,
      isBlocked: false,
      isReversed: false,
      isInGSTR2B: false,
      notes: 'Not in GSTR-2B — cannot claim per Rule 36(4). Follow up with supplier.',
    };
  }

  // Must be received in books
  if (!purchase.isReceivedInBooks) {
    return {
      purchaseId: purchase.id,
      supplierGSTIN: purchase.supplierGSTIN,
      invoiceNumber: purchase.invoiceNumber,
      invoiceDate: purchase.invoiceDate,
      igstAmount: purchase.igstAmount,
      cgstAmount: purchase.cgstAmount,
      sgstAmount: purchase.sgstAmount,
      cessAmount: purchase.cessAmount,
      category: purchase.category,
      isEligible: false,
      isBlocked: false,
      isReversed: false,
      isInGSTR2B: purchase.isInGSTR2B,
      notes: 'Goods/services not yet received — Section 16(2)(b) condition not met',
    };
  }

  // Check time limit (30 Nov following FY end)
  const itcDeadline = getITCDeadline(purchase.invoiceDate);
  if (new Date() > itcDeadline) {
    return {
      purchaseId: purchase.id,
      supplierGSTIN: purchase.supplierGSTIN,
      invoiceNumber: purchase.invoiceNumber,
      invoiceDate: purchase.invoiceDate,
      igstAmount: purchase.igstAmount,
      cgstAmount: purchase.cgstAmount,
      sgstAmount: purchase.sgstAmount,
      cessAmount: purchase.cessAmount,
      category: purchase.category,
      isEligible: false,
      isBlocked: false,
      isReversed: false,
      isInGSTR2B: purchase.isInGSTR2B,
      itcTimeLimitExpiry: itcDeadline,
      notes: `ITC time limit expired: Section 16(4). Deadline was ${itcDeadline.toDateString()}`,
    };
  }

  // Eligible
  return {
    purchaseId: purchase.id,
    supplierGSTIN: purchase.supplierGSTIN,
    invoiceNumber: purchase.invoiceNumber,
    invoiceDate: purchase.invoiceDate,
    igstAmount: purchase.igstAmount,
    cgstAmount: purchase.cgstAmount,
    sgstAmount: purchase.sgstAmount,
    cessAmount: purchase.cessAmount,
    category: purchase.category,
    isEligible: true,
    isBlocked: false,
    isReversed: false,
    isInGSTR2B: purchase.isInGSTR2B,
    itcTimeLimitExpiry: itcDeadline,
  };
}

// Rule 42 ITC reversal for inputs used in exempt supplies
export function computeRule42Reversal(
  totalCommonITC: { igst: number; cgst: number; sgst: number },
  exemptTurnover: number,
  totalTurnover: number
): { reversalIGST: number; reversalCGST: number; reversalSGST: number } {
  if (totalTurnover === 0) return { reversalIGST: 0, reversalCGST: 0, reversalSGST: 0 };
  const ratio = exemptTurnover / totalTurnover;
  return {
    reversalIGST: round2(totalCommonITC.igst * ratio),
    reversalCGST: round2(totalCommonITC.cgst * ratio),
    reversalSGST: round2(totalCommonITC.sgst * ratio),
  };
}

// Rule 43 ITC reversal for capital goods used in exempt supplies
export function computeRule43Reversal(
  capitalGoodsITC: number, // Total ITC on capital goods
  exemptTurnover: number,
  totalTurnover: number
): number {
  // Monthly notional credit = total CG ITC / 60 months (5 years useful life)
  const monthlyNotional = capitalGoodsITC / 60;
  if (totalTurnover === 0) return 0;
  return round2(monthlyNotional * (exemptTurnover / totalTurnover));
}

// Calculate ITC time limit deadline (30 Nov following FY end)
function getITCDeadline(invoiceDate: Date): Date {
  const month = invoiceDate.getMonth() + 1;
  const year = invoiceDate.getFullYear();
  // FY ends March 31. If invoice is Apr-Mar FY, deadline is 30 Nov of the following year
  const fyEndYear = month >= 4 ? year + 1 : year;
  return new Date(fyEndYear, 10, 30); // Month 10 = November (0-indexed)
}

function getBlockingSection(category: ITCCategory): string {
  const map: Record<ITCCategory, string> = {
    MOTOR_VEHICLE: 'Section 17(5)(a)/(aa)',
    FOOD_ENTERTAINMENT: 'Section 17(5)(b)',
    CONSTRUCTION: 'Section 17(5)(c)/(d)',
    PERSONAL_USE: 'Section 17(5)(g)',
    FREE_SAMPLES: 'Section 17(5)(h)',
    HEALTH_BEAUTY: 'Section 17(5)(b)',
    INVENTORY: '',
    PACKAGING: '',
    MARKETPLACE_FEE: '',
    LOGISTICS: '',
    OFFICE_SUPPLIES: '',
    CAPITAL_GOODS: '',
    MIXED_USE: '',
  };
  return map[category] ?? '';
}

function round2(n: number): number {
  return Math.round(n * 100) / 100;
}
```

---

## 13. Financial Year & Period Management

```typescript
// src/gst/period/fyManager.ts

export interface IndianFinancialYear {
  label: string;          // e.g., "2025-26"
  startDate: Date;        // April 1, 2025
  endDate: Date;          // March 31, 2026
  gstr1DueMonthly: Date[];  // 11th of each month for monthly filers
  gstr3bDueMonthly: Date[]; // 20th of each month for monthly filers
}

export interface FilingPeriod {
  periodKey: string;      // MMYYYY e.g., "032026"
  month: number;
  year: number;
  isQuarterly: boolean;
  quarterNumber?: 1 | 2 | 3 | 4;
  gstr1Due: Date;
  gstr3bDueMonthly: Date;   // 20th for monthly (>₹5 crore)
  gstr3bDueCategoryA: Date; // 22nd for quarterly Category A
  gstr3bDueCategoryB: Date; // 24th for quarterly Category B
  iffDue?: Date;            // IFF due (13th) for QRMP months 1 & 2 of quarter
  pmt06Due?: Date;          // PMT-06 (25th) for QRMP monthly payments
  gstr2bAvailable: Date;    // 14th of M+1
  advanceTaxDue?: Date;     // If applicable
}

export function getCurrentFinancialYear(date: Date = new Date()): IndianFinancialYear {
  const month = date.getMonth() + 1;
  const year = date.getFullYear();
  const fyStartYear = month >= 4 ? year : year - 1;
  const label = `${fyStartYear}-${String(fyStartYear + 1).slice(2)}`;
  return {
    label,
    startDate: new Date(fyStartYear, 3, 1),     // April 1
    endDate: new Date(fyStartYear + 1, 2, 31),   // March 31
    gstr1DueMonthly: generateMonthlyDates(fyStartYear, 11),  // 11th each month
    gstr3bDueMonthly: generateMonthlyDates(fyStartYear, 20), // 20th each month
  };
}

export function getFilingPeriod(month: number, year: number, isQRMP: boolean): FilingPeriod {
  const periodKey = `${String(month).padStart(2, '0')}${year}`;

  // Next month for due dates
  const nextMonth = month === 12 ? 1 : month + 1;
  const nextYear = month === 12 ? year + 1 : year;

  // GSTR-1 due: 11th of next month (monthly) or 13th of month after quarter (QRMP)
  const gstr1Due = isQRMP
    ? getQRMPGSTR1Due(month, year)
    : new Date(nextYear, nextMonth - 1, 11);

  // GSTR-3B due
  const gstr3bDueMonthly = new Date(nextYear, nextMonth - 1, 20);
  const gstr3bDueCategoryA = new Date(nextYear, nextMonth - 1, 22);
  const gstr3bDueCategoryB = new Date(nextYear, nextMonth - 1, 24);

  // GSTR-2B available: 14th of next month
  const gstr2bAvailable = new Date(nextYear, nextMonth - 1, 14);

  // IFF for QRMP: available for months 1 & 2 of quarter only
  const isQRMPMonthWithIFF = isQRMP && !isQuarterEndMonth(month);
  const iffDue = isQRMPMonthWithIFF ? new Date(nextYear, nextMonth - 1, 13) : undefined;

  // PMT-06 for QRMP: 25th of next month (for months 1 & 2 of quarter)
  const pmt06Due = isQRMP && !isQuarterEndMonth(month)
    ? new Date(nextYear, nextMonth - 1, 25)
    : undefined;

  return {
    periodKey,
    month, year,
    isQuarterly: isQRMP,
    quarterNumber: getQuarterNumber(month),
    gstr1Due,
    gstr3bDueMonthly,
    gstr3bDueCategoryA,
    gstr3bDueCategoryB,
    iffDue,
    pmt06Due,
    gstr2bAvailable,
  };
}

// Determine if seller qualifies for QRMP scheme
export function isQRMPEligible(aato: number): boolean {
  return aato <= 500; // ≤₹5 crore AATO (aato in lakh; 500 lakh = ₹5 crore)
}

// Get current open filing period
export function getCurrentFilingPeriod(aato: number): FilingPeriod {
  const today = new Date();
  const month = today.getMonth() + 1;
  const year = today.getFullYear();
  // Current period is the previous month (filing happens for the prior period)
  const filingMonth = month === 1 ? 12 : month - 1;
  const filingYear = month === 1 ? year - 1 : year;
  return getFilingPeriod(filingMonth, filingYear, isQRMPEligible(aato));
}

function getQuarterNumber(month: number): 1 | 2 | 3 | 4 {
  // Indian FY quarters: Q1=Apr-Jun, Q2=Jul-Sep, Q3=Oct-Dec, Q4=Jan-Mar
  if (month >= 4 && month <= 6) return 1;
  if (month >= 7 && month <= 9) return 2;
  if (month >= 10 && month <= 12) return 3;
  return 4;
}

function isQuarterEndMonth(month: number): boolean {
  return [3, 6, 9, 12].includes(month);
}

function getQRMPGSTR1Due(month: number, year: number): Date {
  // QRMP quarterly filers: GSTR-1 due on 13th of month after quarter end
  const quarterEndMonth = Math.ceil(month / 3) * 3;
  const duemonth = quarterEndMonth === 12 ? 1 : quarterEndMonth + 1;
  const dueYear = quarterEndMonth === 12 ? year + 1 : year;
  return new Date(dueYear, duemonth - 1, 13);
}

function generateMonthlyDates(fyStartYear: number, dayOfMonth: number): Date[] {
  const dates: Date[] = [];
  for (let m = 4; m <= 15; m++) { // April (month 4) to March next year (month 15→3)
    const actualMonth = m > 12 ? m - 12 : m;
    const actualYear = m > 12 ? fyStartYear + 1 : fyStartYear;
    // Due date is in next month
    const dueMonth = actualMonth === 12 ? 1 : actualMonth + 1;
    const dueYear = actualMonth === 12 ? actualYear + 1 : actualYear;
    dates.push(new Date(dueYear, dueMonth - 1, dayOfMonth));
  }
  return dates;
}

// Compliance calendar: all due dates for the current month
export function getMonthlyComplianceCalendar(
  aato: number,
  currentDate: Date = new Date()
): Array<{ date: Date; obligation: string; isOverdue: boolean }> {
  const month = currentDate.getMonth() + 1;
  const year = currentDate.getFullYear();
  const prevMonth = month === 1 ? 12 : month - 1;
  const prevYear = month === 1 ? year - 1 : year;
  const isQRMP = isQRMPEligible(aato);
  const period = getFilingPeriod(prevMonth, prevYear, isQRMP);

  const calendar = [
    { date: new Date(year, month - 1, 10), obligation: 'ECO deposits GST TCS (GSTR-8)' },
    { date: new Date(year, month - 1, 11), obligation: 'GSTR-1 filing — monthly filers (AATO >₹5 crore)' },
    { date: new Date(year, month - 1, 13), obligation: 'GSTR-1/IFF filing — QRMP quarterly filers' },
    { date: new Date(year, month - 1, 14), obligation: `GSTR-2B generated for ${prevMonth}/${prevYear}` },
    { date: new Date(year, month - 1, 20), obligation: 'GSTR-3B due — monthly filers' },
    { date: new Date(year, month - 1, 22), obligation: 'GSTR-3B due — QRMP Category A states' },
    { date: new Date(year, month - 1, 24), obligation: 'GSTR-3B due — QRMP Category B states' },
    { date: new Date(year, month - 1, 25), obligation: 'PMT-06 — QRMP monthly tax payment (months 1 & 2 of quarter)' },
  ];

  return calendar.map(item => ({
    ...item,
    isOverdue: item.date < currentDate,
  }));
}
```

---

## 14. Complete Data Models

```typescript
// src/types/gstInvoice.ts

export type InvoiceType =
  | 'TAX_INVOICE'
  | 'CREDIT_NOTE'
  | 'DEBIT_NOTE'
  | 'BILL_OF_SUPPLY'
  | 'DELIVERY_CHALLAN';

export type SupplyType =
  | 'B2B'     // Business to registered business
  | 'B2C'     // Business to unregistered consumer
  | 'SEZWP'   // SEZ with payment of tax
  | 'SEZWOP'  // SEZ without payment
  | 'EXPWP'   // Export with payment
  | 'EXPWOP'  // Export without payment
  | 'DEXP';   // Deemed export

export type InvoiceStatus = 'DRAFT' | 'SUBMITTED' | 'IRN_GENERATED' | 'CANCELLED' | 'AMENDED';

export interface GSTInvoiceLine {
  slNo: number;
  productDescription: string;
  hsnCode: string;
  uqc: string;
  quantity: number;
  unitPrice: number;
  grossValue: number;
  discount: number;
  taxableValue: number;
  gstRate: number;
  cgstRate: number; cgstAmount: number;
  sgstRate: number; sgstAmount: number;
  igstRate: number; igstAmount: number;
  utgstRate: number; utgstAmount: number;
  cessRate: number; cessAmount: number;
  totalTaxAmount: number;
  totalAmount: number;
  sku?: string;
}

export interface GSTInvoice {
  // Identity
  invoiceNumber: string;
  invoiceDate: Date;
  financialYear: string;
  invoiceType: InvoiceType;
  supplyType: SupplyType;
  shopifyOrderId: string;
  shopifyOrderName: string;

  // Supplier
  supplierGSTIN: string;
  supplierLegalName: string;
  supplierTradeName?: string;
  supplierAddress1: string;
  supplierAddress2?: string;
  supplierCity: string;
  supplierStateCode: string;
  supplierPincode: string;

  // Recipient
  recipientGSTIN: string;
  recipientName: string;
  recipientAddress1: string;
  recipientAddress2?: string;
  recipientCity: string;
  recipientStateCode: string;
  recipientPincode: string;

  // Delivery (ship-to, if different from billing)
  deliveryAddress1?: string;
  deliveryAddress2?: string;
  deliveryCity?: string;
  deliveryStateCode?: string;
  deliveryPincode?: string;

  // Tax
  placeOfSupplyCode: string;
  placeOfSupplyName: string;
  isInterState: boolean;
  taxType: 'CGST_SGST' | 'IGST' | 'CGST_UTGST';
  isReverseCharge: boolean;
  ecoGSTIN?: string;

  // Line items
  lineItems: GSTInvoiceLine[];

  // Totals
  totalTaxableValue: number;
  totalCGST: number;
  totalSGST: number;
  totalIGST: number;
  totalUTGST: number;
  totalCess: number;
  totalTax: number;
  roundOff: number;
  grandTotal: number;

  // E-invoice
  irn?: string;
  ackNumber?: number;
  ackDate?: Date;
  signedQRCode?: string;
  eInvoiceJSON?: string; // Stored serialized JSON

  // Status
  status: InvoiceStatus;
  isCancelled: boolean;
  cancellationDate?: Date;
  cancellationReason?: string;
  amendedInvoiceNumber?: string;

  // Metadata
  warnings?: string[];
  createdAt: Date;
  updatedAt: Date;
}

export interface GSTCreditNote {
  creditNoteNumber: string;
  creditNoteDate: Date;
  financialYear: string;
  originalInvoiceNumber: string;
  originalInvoiceDate: Date;
  shopifyRefundId: string;
  shopifyOrderId: string;
  supplierGSTIN: string;
  supplierLegalName: string;
  supplierStateCode: string;
  recipientGSTIN: string;
  recipientName: string;
  recipientStateCode: string;
  placeOfSupplyCode: string;
  isInterState: boolean;
  taxType: 'CGST_SGST' | 'IGST' | 'CGST_UTGST';
  lineItems: Omit<GSTInvoiceLine, 'grossValue' | 'discount' | 'gstRate'>[];
  totalTaxableValue: number;
  totalCGST: number;
  totalSGST: number;
  totalIGST: number;
  totalUTGST: number;
  totalCess: number;
  totalTax: number;
  grandTotal: number;
  reason: string;
  irn?: string;
  ackNumber?: number;
  status: 'DRAFT' | 'SUBMITTED' | 'IRN_GENERATED' | 'CANCELLED';
  createdAt: Date;
}

export interface GSTDebitNote {
  debitNoteNumber: string;
  debitNoteDate: Date;
  financialYear: string;
  originalInvoiceNumber: string;
  originalInvoiceDate: Date;
  shopifyOrderId: string;
  supplierGSTIN: string;
  supplierLegalName: string;
  recipientGSTIN: string;
  recipientName: string;
  placeOfSupplyCode: string;
  isInterState: boolean;
  taxType: 'CGST_SGST' | 'IGST' | 'CGST_UTGST';
  additionalTaxableValue: number;  // Additional amount now payable
  additionalCGST: number;
  additionalSGST: number;
  additionalIGST: number;
  additionalCess: number;
  totalAdditional: number;
  reason: string;
  irn?: string;
  status: 'DRAFT' | 'SUBMITTED' | 'IRN_GENERATED' | 'CANCELLED';
  createdAt: Date;
}

export interface GSTBillOfSupply {
  documentNumber: string;
  documentDate: Date;
  financialYear: string;
  shopifyOrderId: string;
  shopifyOrderName: string;
  supplierGSTIN: string;
  supplierLegalName: string;
  supplierStateCode: string;
  recipientGSTIN: string;
  recipientName: string;
  recipientStateCode: string;
  lineItems: Array<{
    slNo: number;
    productDescription: string;
    hsnCode: string;
    uqc: string;
    quantity: number;
    unitPrice: number;
    taxableValue: number;
  }>;
  totalValue: number;
  compositionNote: string; // "Composition taxable person, not eligible to collect tax on supplies"
  status: 'DRAFT' | 'SUBMITTED';
  createdAt: Date;
}

// src/types/gstrReports.ts

export interface GSTR1Report {
  gstin: string;
  fp: string;
  gt: number;
  cur_gt: number;
  b2b: any[];
  b2cl: any[];
  b2cs: any[];
  cdnr: any[];
  hsn: { data: any[] };
  doc_issue?: any;
}

export interface GSTR3BReport {
  gstin: string;
  fp: string;
  ret_period: string;
  sup_details: any;
  itc_elg: any;
  intr_dtls: any[];
  inward_sup: any;
  tax_pay: any;
}

// src/types/einvoice.ts — re-exported from module 10

// src/types/itc.ts

export interface ITCRecord {
  purchaseId: string;
  supplierGSTIN: string;
  invoiceNumber: string;
  invoiceDate: Date;
  igstAmount: number;
  cgstAmount: number;
  sgstAmount: number;
  cessAmount: number;
  category: string;
  isEligible: boolean;
  isBlocked: boolean;
  blockingSection?: string;
  isReversed: boolean;
  reversalReason?: string;
  reversalAmount?: number;
  isInGSTR2B: boolean;
  claimedInPeriod?: string;
  itcTimeLimitExpiry?: Date;
  notes?: string;
}

export interface HSNEntry {
  code: string;
  level: 2 | 4 | 6 | 8;
  description: string;
  chapter: string;
  chapterDescription: string;
  gstRate: number;
  cessRate: number;
  valueThresholds?: { upToValue: number; rateBelow: number; rateAbove: number };
  uqc: string;
  isService: boolean;
  sacCode?: string;
}

export interface TCSRecord {
  id: string;
  marketplaceGSTIN: string;
  marketplaceName: string;
  sellerGSTIN: string;
  month: number;
  year: number;
  settlementPeriod: string;
  grossSalesValue: number;
  returnsValue: number;
  netTaxableValue: number;
  tcsRate: number;
  tcsCGST: number;
  tcsSGST: number;
  tcsIGST: number;
  totalTCS: number;
  creditedToCashLedger: boolean;
  gstr8Reference?: string;
  reconciliationStatus: 'PENDING' | 'MATCHED' | 'DISCREPANCY';
  discrepancyNotes?: string;
  createdAt: Date;
}

export interface TDSRecord {
  id: string;
  ecoGSTIN: string;
  ecoName: string;
  sellerPAN: string;
  financialYear: string;
  quarter: 1 | 2 | 3 | 4;
  grossSalesValue: number;
  tdsRate: number;
  tdsAmount: number;
  tdsDeductedDate?: Date;
  form26QEReference?: string;
  creditInForm26AS: boolean;
  reconciliationStatus: 'PENDING' | 'MATCHED' | 'DISCREPANCY';
  createdAt: Date;
}

export interface PlaceOfSupplyResult {
  placeOfSupplyCode: string;
  placeOfSupplyName: string;
  supplierStateCode: string;
  isInterState: boolean;
  taxType: 'CGST_SGST' | 'IGST' | 'CGST_UTGST';
  posRule: string;
  isSEZ: boolean;
  isExport: boolean;
  buyerStateCode?: string;
  deliveryStateCode?: string;
  errors: string[];
}

export interface TaxCalculationResult {
  hsnCode: string;
  taxableValue: number;
  gstRate: number;
  cessRate: number;
  cgstRate: number; cgstAmount: number;
  sgstRate: number; sgstAmount: number;
  igstRate: number; igstAmount: number;
  utgstRate: number; utgstAmount: number;
  cessAmount: number;
  totalTax: number;
  totalValue: number;
  taxType: string;
}

export interface EInvoice extends EInvoiceJSON {
  // Extended with internal tracking fields
  localIRN: string;       // Pre-computed SHA-256 for deduplication
  invoiceId: string;      // Internal GSTInvoice ID
  submittedAt?: Date;
  irpStatus: 'PENDING' | 'SUBMITTED' | 'ACK' | 'CANCELLED' | 'ERROR';
  irpError?: string;
}
```

---

## 15. Implementation Checklist

### Phase 1: Foundation (Week 1–2)
- [ ] **Implement `stateCodes.ts`** with complete 37-state master and Shopify province mapping
- [ ] **Implement `gstin.ts`** GSTIN validation with Luhn mod-36 checksum
- [ ] **Unit test** GSTIN validator with known valid/invalid GSTINs
- [ ] **Implement `placeOfSupply.ts`** with all Section 10 scenarios
- [ ] **Unit test** PoS engine: intra-state, inter-state, UT, bill-to-ship-to, export, SEZ

### Phase 2: Tax Calculation (Week 2–3)
- [ ] **Implement `taxRate.ts`** with full HSN rate table (post-GST 2.0 rates)
- [ ] **Implement `hsnLookup.ts`** with common e-commerce HSN codes
- [ ] **Unit test** tax calculation for each scenario: CGST+SGST, IGST, CGST+UTGST
- [ ] **Test apparel/footwear value threshold logic** (₹1,000 cut-off)
- [ ] **Verify GST 2.0 rate changes**: TV/AC → 18%, shampoo/biscuits → 5%, etc.
- [ ] **Handle `taxes_included` flag** from Shopify correctly

### Phase 3: Invoice Generation (Week 3–4)
- [ ] **Implement `invoiceNumber.ts`** with FY-aware sequential numbering, max 16 chars
- [ ] **Implement `invoiceGenerator.ts`** with full Shopify order → GSTInvoice mapping
- [ ] **Implement credit note generation** from Shopify refunds
- [ ] **Implement debit note generation** for price adjustments
- [ ] **Implement Bill of Supply** for composition dealers
- [ ] **Validate all 16 Rule 46 mandatory fields** are present
- [ ] **Test B2B vs B2C branching** correctly

### Phase 4: E-Invoicing (Week 4–5)
- [ ] **Implement `einvoice.ts`** — build EInvoiceJSON from GSTInvoice
- [ ] **Implement `computeLocalIRN`** using SHA-256 for deduplication
- [ ] **Implement `IRPClient`** with authenticate + generateIRN + cancelIRN
- [ ] **Implement RSA encryption** for password/appKey (use `node-forge` or built-in `crypto`)
- [ ] **Implement AES-256 encryption/decryption** for SEK and invoice data
- [ ] **Test in IRP sandbox** (`https://einv-apisandbox.nic.in`)
- [ ] **Implement 30-day upload window validation**
- [ ] **Implement 24-hour cancellation window check**
- [ ] **Store signed QR code** from IRP response on invoice record
- [ ] **Implement AATO-based e-invoice trigger** (only if AATO > ₹5 crore)

### Phase 5: GSTR-1 (Week 5–6)
- [ ] **Implement `gstr1.ts`** aggregation engine
- [ ] **Verify B2B (Table 4), B2C Large (Table 5), B2C Small (Table 7)** classification logic
- [ ] **Implement CDN (Table 9)** from credit notes
- [ ] **Implement HSN summary (Table 12)** aggregation for both B2B and B2C tabs
- [ ] **Test B2C Large threshold**: inter-state, value > ₹2.5 lakh → Table 5
- [ ] **Validate JSON output** against GST portal GSTR-1 schema
- [ ] **Implement GSTR-1A** (amendment flow for locked GSTR-3B)

### Phase 6: GSTR-3B (Week 6–7)
- [ ] **Implement `gstr3b.ts`** summary computation
- [ ] **Implement ITC utilization order** (IGST → CGST → SGST cross-utilization)
- [ ] **Implement Table 3.2** inter-state supply state-wise breakdown (auto-populated from Nov 2025)
- [ ] **Test nil return scenario** (zero turnover period)
- [ ] **Implement late fee calculation** (₹50/day; ₹20/day nil return)
- [ ] **Implement interest computation** (18% p.a. on late tax payment)

### Phase 7: TCS/TDS (Week 7–8)
- [ ] **Implement `tcsEngine.ts`** TCS computation from settlement data
- [ ] **Implement TCS reconciliation** against ECO's GSTR-8
- [ ] **Implement Electronic Cash Ledger tracking** (TCS credits)
- [ ] **Implement Section 194-O TDS tracking** at 0.1% (with PAN) / 5% (without PAN)
- [ ] **Build reconciliation UI data with merchant

### Phase 8: ITC Engine (Week 8–9)
- [ ] **Implement `itcEngine.ts`** with GSTR-2B matching logic
- [ ] **Implement Section 17(5) blocked credit** filtering — motor vehicles, food, personal use, construction
- [ ] **Implement Rule 42 reversal** for mixed-use inputs (monthly D1 + annual true-up)
- [ ] **Implement Rule 43 reversal** for capital goods (60-month useful life)
- [ ] **Implement ITC cross-utilization order**: IGST→CGST→SGST/UTGST
- [ ] **Implement IMS integration** (accept/reject/pending invoice workflow)
- [ ] **Restrict provisional ITC** beyond GSTR-2B (as per Rule 36(4))

### Phase 9: FY & Period Management (Week 9–10)
- [ ] **Implement `fyPeriod.ts`** with April–March detection
- [ ] **Implement QRMP vs monthly classification** based on AATO
- [ ] **Implement due date calendar** for GSTR-1, GSTR-3B, IFF, PMT-06
- [ ] **Implement IFF eligibility check** (QRMP filers, first 2 months of quarter)
- [ ] **Implement advance income tax** due date reminders
- [ ] **Implement invoice archival policy** (72-month retention flag)
- [ ] **Implement period lock** once GSTR-3B filed (prevent backdating)

### Phase 10: Testing & Production Readiness (Week 10–12)
- [ ] **End-to-end test**: Shopify order → invoice → e-invoice → GSTR-1 row → GSTR-3B
- [ ] **Stress test**: 10,000 orders in a month → verify GSTR-1 Table 7 consolidation
- [ ] **Test all 37 state codes** and their CGST+SGST/IGST/CGST+UTGST classification
- [ ] **Test edge cases**: SEZ supplies, exports, bill-to-ship-to, unregistered UTs
- [ ] **Test composition scheme** restrictions (no inter-state, no marketplace sales)
- [ ] **Validate GSTIN checksums** against known valid GSTINs from all states
- [ ] **Pen-test IRP API client** with real NIC sandbox credentials
- [ ] **Performance test**: Invoice generation < 200ms; GSTR-1 aggregation < 5s for 1,000 orders
- [ ] **Security review**: Encrypt stored GSTINs; mask PAN in logs
- [ ] **Compliance review**: Validate against latest CBIC circulars (check cbic.gov.in monthly)

---

## Appendix: Key Compliance References

| Reference | Details |
|-----------|---------|
| CGST Act, 2017 | Primary legislation — Sections 9, 10, 16, 17, 24, 31, 34, 52 |
| IGST Act, 2017 | Sections 10, 11, 12, 13 — place of supply |
| CGST Rules, 2017 | Rules 36, 42, 43, 46, 49, 55, 89 |
| E-Invoice Schema | INV-1 v1.1 — download from einvoice6.gst.gov.in |
| GST State Code Master | Appendix to CGST Rules, Schedule III |
| 56th GST Council | Rate rationalization effective 22 Sep 2025 |
| Notification 34/2023-CT | Small seller exemption for intra-state e-commerce |
| Circular 9/09/2019-GST | ₹25,000 penalty for missing place of supply |
| Section 194-O IT Act | E-commerce TDS at 0.1% (effective 1 Oct 2024) |
| IRP NIC Sandbox | https://einv-apisandbox.nic.in |
| CBIC GST Rates | https://cbic-gst.gov.in/gst-goods-services-rates.html |

---

*End of Skill Document — Version 1.0, March 2026*
