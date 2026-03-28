# Indian GST & Tax Laws: Comprehensive Reference for E-Commerce / Shopify Sellers

> **As of March 2026** — Reflects GST 2.0 rate rationalization (effective 22 Sep 2025), 30-day IRN upload limit (effective 1 Apr 2025), and all current compliance requirements.

---

## Table of Contents

1. [GST Structure](#1-gst-structure)
2. [E-Commerce Specific GST Rules](#2-e-commerce-specific-gst-rules)
3. [GST Invoice Requirements (Rule 46)](#3-gst-invoice-requirements-rule-46)
4. [HSN / SAC Codes](#4-hsn--sac-codes)
5. [GST Return Filing](#5-gst-return-filing)
6. [E-Invoicing](#6-e-invoicing)
7. [Input Tax Credit (ITC)](#7-input-tax-credit-itc)
8. [TDS / TCS in GST](#8-tds--tcs-in-gst)
9. [Income Tax for E-Commerce](#9-income-tax-for-e-commerce)
10. [Recent Changes & Upcoming](#10-recent-changes--upcoming)

---

## 1. GST Structure

### 1.1 Four Components of GST

| Tax | Full Name | Levied By | Applicability |
|-----|-----------|-----------|---------------|
| **CGST** | Central Goods & Services Tax | Central Government | Intra-state supplies (alongside SGST/UTGST) |
| **SGST** | State Goods & Services Tax | State Government | Intra-state supplies (alongside CGST) |
| **IGST** | Integrated Goods & Services Tax | Central Government | Inter-state supplies, imports, exports (shared later between Centre and consuming state) |
| **UTGST** | Union Territory Goods & Services Tax | Union Territory Administration | Intra-UT supplies in UTs without a legislature (e.g., Chandigarh, Dadra & Nagar Haveli, Daman & Diu, Lakshadweep, Andaman & Nicobar) |

**Key Rules:**
- GST is a **destination-based consumption tax**.
- For **intra-state** supplies: CGST + SGST (each at half the applicable rate). Max CGST rate: 14%.
- For **inter-state** supplies: IGST only (= CGST + SGST combined rate).
- UTs **with** legislature (Delhi, Puducherry, J&K, Ladakh) levy SGST, not UTGST.

**ITC Cross-Utilization Order:**
1. IGST credit → set off IGST first, then CGST, then SGST/UTGST
2. CGST credit → set off CGST first, then IGST
3. SGST/UTGST credit → set off SGST/UTGST first, then IGST
- CGST credit **cannot** offset SGST and vice versa.

### 1.2 Place of Supply Rules

#### Section 10 IGST Act — Place of Supply for Goods (Domestic)

| Rule | Scenario | Place of Supply |
|------|----------|-----------------|
| 10(1)(a) | Supply involves movement of goods | Location where movement **terminates** for delivery to recipient |
| 10(1)(b) | Bill-to-ship-to model (goods delivered to 3rd party on buyer's instructions) | **Principal place of business of the buyer** (not delivery point) |
| 10(1)(c) | No movement of goods | Location of goods **at time of delivery** to recipient |
| 10(1)(ca) | Supply to **unregistered person** | **Address of delivery** as per recipient's address on record *(key for e-commerce B2C)* |
| 10(1)(d) | Goods assembled/installed at site | Place of installation/assembly |
| 10(1)(e) | Goods on board conveyance | Location where goods are taken on board |

> **E-commerce critical rule:** Section 10(1)(ca) — for B2C deliveries to unregistered persons, the place of supply is the **delivery address**. This determines whether CGST+SGST or IGST applies.

**Practical Example — E-Commerce:**
- Seller (Gujarat) ships to customer (Maharashtra) → Place of supply = Maharashtra → **IGST** charged
- Seller (Maharashtra) ships to customer (Maharashtra, same state) → **CGST + SGST** charged

#### Section 11 IGST Act — Imports/Exports
- **Imports**: Place of supply = location of importer
- **Exports**: Place of supply = location outside India (zero-rated)

#### Section 12 IGST Act — Place of Supply for Services (Both Parties in India)
- **General rule**: Location of the registered recipient; if unregistered, address on record; if no address, location of supplier.
- **Specific overrides** for services like telecom, banking, transport, etc.

#### Section 13 IGST Act — Place of Supply for Services (Cross-Border)
- Used when supplier or recipient is outside India.
- General rule: Location of the recipient.

### 1.3 Intra-State vs Inter-State Determination

```
Determine: Is "Location of Supplier" state = "Place of Supply" state?

YES → INTRA-STATE → Charge CGST + SGST/UTGST
NO  → INTER-STATE → Charge IGST

Special case: Supply to/from SEZ → Always IGST (treated as inter-state)
Special case: Import of goods → IGST + Basic Customs Duty
```

### 1.4 GST Rate Slabs

**Post-GST 2.0 Rationalization (56th GST Council Meeting, effective 22 September 2025):**

| Rate | Category | Examples |
|------|----------|---------|
| **0%** | Exempt / Nil rated | Fresh milk, curd, paneer, bread, salt, eggs, fresh vegetables, books, contraceptives, health & life insurance (post-reform) |
| **5%** | Essential / Merit goods | Basic food items (biscuits, chocolate, hair oil, shampoo, soaps, mineral water), medicines, educational materials, footwear >₹2,500/pair, some services |
| **18%** | Standard rate (most goods & services) | Electronics (mobile phones, laptops, TVs, AC), most FMCG, clothing >₹1,000/piece, software, restaurants, professional services |
| **40%** | Luxury & sin goods | Luxury cars (engine >1500cc/1200cc or length >4000mm), tobacco, cigarettes, pan masala, aerated drinks, non-alcoholic beer, yachts, casinos |
| **3%** | Precious metals | Gold, silver, diamonds |
| **0.25%** | Rough diamonds | Diamonds (rough/semi-polished) |

**Legacy rates still applicable to specific goods:**
- 12%: Some goods not yet reclassified (check CBIC notifications)
- 28% + Cess: Certain remaining luxury/sin goods with cess

> **Important for coding:** Always validate against the CBIC GST Rates master — rates for specific HSN codes can vary from chapter-level rates. The 56th Council rationalized rates; 12% and 28% slabs were largely absorbed into 5% and 18% respectively.

**E-Commerce Specific Rate Examples:**

| Category | Rate | Notes |
|----------|------|-------|
| Apparel ≤₹1,000/piece | 5% | Chapter 61/62 |
| Apparel >₹1,000/piece | 12% | Chapter 61/62 |
| Mobile phones | 18% | HSN 8517 |
| Laptops/computers | 18% | HSN 8471 |
| Beauty/cosmetics | 18% | HSN 3304/3305 |
| Footwear ≤₹1,000 | 5% | HSN 6401-6405 |
| Footwear >₹1,000 (≤₹2,500) | 12% | HSN 6401-6405 |
| Footwear >₹2,500 | 5% | Post-rationalization |
| Home furniture/furnishings | 18% | Chapter 94 |
| Books | 0% | HSN 4901 |
| Handicrafts | Varies 5%-12% | Specific HSN |

### 1.5 Composition Scheme

**Eligibility:**
| Category | Turnover Limit (General States) | Turnover Limit (Special States) |
|----------|--------------------------------|--------------------------------|
| Manufacturers & Traders of goods | ≤ ₹1.5 crore | ≤ ₹75 lakh |
| Service providers | ≤ ₹50 lakh | ≤ ₹50 lakh (same) |
| Mixed (goods + services) | ≤ ₹1.5 crore (services ≤10% or ₹5L whichever higher) | ≤ ₹75 lakh |

**Special category states:** Himachal Pradesh, Uttarakhand, Arunachal Pradesh, Assam, Manipur, Meghalaya, Mizoram, Nagaland, Sikkim, Tripura

**Composition Scheme Tax Rates:**
| Business Type | Rate on Turnover |
|--------------|-----------------|
| Manufacturer | 1% (0.5% CGST + 0.5% SGST) |
| Trader | 1% (0.5% CGST + 0.5% SGST) |
| Restaurant (non-alcohol) | 5% (2.5% CGST + 2.5% SGST) |
| Service provider (SAHAJ scheme) | 6% (3% CGST + 3% SGST) |

**Composition Scheme Restrictions:**
- ❌ Cannot make **inter-state supplies** (auto-disqualifier for most e-commerce)
- ❌ Cannot supply through **e-commerce operators** (Section 10(2)(d)) — **disqualifies Amazon/Flipkart/Shopify marketplace sellers**
- ❌ Cannot issue tax invoices (must issue Bill of Supply)
- ❌ Cannot claim ITC
- ❌ Cannot collect GST from customers
- ✅ File quarterly GSTR-4 return (annually from FY 2019-20 via GSTR-4A)
- ✅ Pay tax quarterly via CMP-08 challan

**Filing for composition:** Form CMP-02 by 31st March of FY in which scheme is opted.

---

## 2. E-Commerce Specific GST Rules

### 2.1 Mandatory GST Registration for E-Commerce Sellers

Under Section 24 of the CGST Act, **GST registration is compulsory regardless of turnover** for:
- Any person making **inter-state taxable supply**
- Any person supplying goods/services **through an e-commerce operator** (e.g., Amazon, Flipkart, Meesho, Myntra, Nykaa)
- E-commerce operators themselves

**Practical implication:** Any seller listing on a marketplace is mandatorily required to obtain GST registration even if turnover is ₹0.

**Limited Exception (Notification 34/2023-CT):** Small sellers making **only intra-state supplies** through platforms may qualify for exemption if below the turnover threshold and complete PAN-based enrolment — applicable to a very narrow class.

**Standard Registration Thresholds (for own-website sellers not using marketplaces):**
| State Category | Goods | Services |
|---------------|-------|---------|
| General states | ₹40 lakh | ₹20 lakh |
| Special category states (Manipur, Mizoram, Nagaland, Tripura) | ₹20 lakh | ₹10 lakh |
| Other NE + Himachal Pradesh + Uttarakhand | ₹40 lakh | ₹20 lakh |

### 2.2 Section 52 — Tax Collected at Source (TCS) by E-Commerce Operators

**Who collects:** E-Commerce Operators (ECOs) — Amazon, Flipkart, Myntra, Meesho, Nykaa, Snapdeal, etc.

**Rate:**
| Supply Type | Rate | Components |
|-------------|------|-----------|
| Intra-state | 1% | 0.5% CGST + 0.5% SGST |
| Inter-state | 1% | 1% IGST |

**Mechanism:**
1. ECO collects payment from buyer
2. ECO deducts TCS @ 1% of **net taxable value** (gross sales minus returns/cancellations for that month)
3. ECO pays balance (minus commission) to seller
4. ECO deposits TCS with government by **10th of the following month**
5. ECO files **GSTR-8** monthly with supplier-GSTIN-wise breakdown
6. TCS credit automatically appears in seller's **Electronic Cash Ledger** (not credit ledger)
7. Seller claims TCS credit in **GSTR-3B** (Table 6.2 — TDS/TCS credit)

**Key Rules:**
- TCS is on the **net value** of taxable supplies — cancellations and returns reduce the base
- Both ECO and seller must be GST-registered
- TCS is NOT an additional tax — it is an advance payment that seller adjusts against output liability
- ECO also files annual return in **Form GSTR-9B**
- If ECO is NOT collecting consideration (e.g., seller collects directly), TCS provisions may not apply

**Seller's GSTR-3B Treatment:**
```
TCS Credit received from ECO (GSTR-8 reflected in Electronic Cash Ledger)
→ Credited automatically to Electronic Cash Ledger
→ Can be used to offset GST payable
→ Report in Table 6.2 of GSTR-3B
```

### 2.3 Section 9(5) — E-Commerce Operator as Deemed Supplier

For certain **notified services**, the ECO itself is treated as the supplier (not the actual service provider), and must pay GST regardless of whether the actual supplier is registered.

**Notified services under Section 9(5) (as of 2026):**

| Service | Notes |
|---------|-------|
| Passenger transport (app-based cabs) | Ola, Uber, etc. |
| Hotel/accommodation services | OYO, MakeMyTrip listings |
| Housekeeping services (plumbing, carpentry, etc.) | Urban Company, etc. |
| **Restaurant services** (effective 1 Jan 2022) | Swiggy, Zomato — pays 5% GST |

**Key differences from Section 52:**

| Parameter | Section 52 (TCS) | Section 9(5) (Deemed Supplier) |
|-----------|-----------------|-------------------------------|
| Scope | All taxable supplies through ECO | Only notified service categories |
| Tax liability | Seller pays; ECO collects TCS | **ECO pays the full GST** |
| Supplier registration | Both ECO and supplier mandatory | Supplier can be unregistered |
| ITC | Normal flow | ECO cannot use ITC for Section 9(5) liability (must pay in cash) |
| GSTR-3B reporting | Table 3.1 for supplier | Table 3.1.1(i) for ECO; Table 3.1.1(ii) for supplier |

**GSTR-3B Table 3.1.1 (added from July 2022):**
- 3.1.1(i): ECO fills — taxable value and tax on Section 9(5) supplies
- 3.1.1(ii): Supplier fills — shows supplies under Section 9(5) (where tax is ECO's liability)
- Suppliers must NOT include Section 9(5) supplies in Table 3.1

### 2.4 Place of Supply for E-Commerce Sales

For **B2C e-commerce** (supply to unregistered persons):
- Per Section 10(1)(ca): Place of supply = **delivery address** of the customer
- This is the fundamental rule driving CGST/SGST vs IGST determination for online orders

**Coding Logic:**
```
If seller_state == delivery_state:
    Apply CGST + SGST
Else:
    Apply IGST
```

**Note:** For **B2B e-commerce** (supply to registered businesses):
- Section 10(1)(a): Place of supply = where movement of goods terminates for delivery
- Typically the buyer's state (registered place of business)

---

## 3. GST Invoice Requirements (Rule 46 CGST Rules)

### 3.1 Mandatory Fields — Tax Invoice (16 Fields)

All 16 fields under Rule 46 of the CGST Rules, 2017:

| # | Field | Details |
|---|-------|---------|
| 1 | **Supplier Name, Address, GSTIN** | Legal name as per PAN; complete address |
| 2 | **Invoice Number** | Unique, consecutive number; max **16 characters**; alphanumeric with `/`, `-` allowed; unique per GSTIN per **financial year** |
| 3 | **Date of Issue** | Date the invoice was issued (DD/MM/YYYY) |
| 4 | **Recipient Details (Registered)** | Name, Address, GSTIN/UIN of registered buyer |
| 5 | **Recipient Details (Unregistered, value ≥ ₹50,000)** | Name, address of delivery, state name, state code (even if value <₹50,000 if recipient requests) |
| 6 | **HSN/SAC Code** | Harmonized System of Nomenclature code for goods / Service Accounting Code for services |
| 7 | **Description of Goods/Services** | Clear description |
| 8 | **Quantity (for goods)** | Number + Unit of Measurement (UQC: KGS, NOS, MTR, etc.) |
| 9 | **Total Value of Supply** | Total value inclusive of all taxes |
| 10 | **Taxable Value** | Value after discounts/abatements (pre-tax) |
| 11 | **Rate of Tax** | CGST rate, SGST rate, IGST rate, UTGST rate, Cess rate separately |
| 12 | **Amount of Tax Charged** | CGST amount, SGST amount, IGST amount, UTGST amount, Cess amount separately |
| 13 | **Place of Supply** | State name + state code — **mandatory for inter-state transactions**; penalty up to ₹25,000 for omission (Circular No. 9/09/2019-GST) |
| 14 | **Delivery Address** | If delivery address differs from place of supply (bill-to-ship-to scenarios) |
| 15 | **Reverse Charge Indicator** | "Yes/No" — whether GST is payable on reverse charge basis |
| 16 | **Signature** | Authorized signatory's signature or digital signature (not required for electronic invoices with IRN) |

### 3.2 Invoice Numbering Rules

- **Maximum length:** 16 characters
- **Format:** Alphanumeric; may include `/` and `-`
- **Sequencing:** Consecutive (sequential) for each financial year per GSTIN
- **Uniqueness:** Must be unique for a GSTIN within a financial year
- **Case:** From June 2025 — IRP treats invoice numbers as **case-insensitive** (ABC101 = abc101 = Abc101); auto-converted to uppercase before IRN generation
- **Series:** Multiple series allowed (e.g., "TAX/2025-26/001", "INV-001") but must be consecutive within each series

### 3.3 Time Limits for Invoice Issuance

| Supply Type | Time Limit |
|------------|-----------|
| Goods (general) | Before or at the time of removal/delivery |
| Continuous supply of goods | Before or at time of issue of statement of accounts |
| Services (general) | Within **30 days** of supply |
| Services by Banks/FIs/NBFCs | Within **45 days** |
| Reverse charge supplies | Date of supply or 30 days from date of supplier's invoice |

### 3.4 Credit Notes (Section 34)

**When to issue:**
- Tax charged on invoice exceeds actual tax payable
- Goods returned by recipient
- Goods/services found deficient

**Mandatory fields in Credit Note:**
1. Document type: "Credit Note"
2. Sequential credit note number (up to 16 chars)
3. Date of issue
4. Original invoice number and date
5. Supplier and recipient details (same as invoice)
6. HSN/SAC
7. Description, quantity
8. Taxable value reduction
9. Tax rate and reduced tax amount
10. Signature

**Time limit:** Must be reported in GSTR-1 **by the earlier of:**
- 30th November of the FY following the FY in which supply was made
- Date of furnishing of annual return (GSTR-9)

### 3.5 Debit Notes (Section 34)

**When to issue:**
- Tax charged on invoice is less than the actual tax payable
- Additional consideration becomes payable

**Time limit for ITC on debit note:** Buyer can claim ITC up to:
- 30th November following end of FY to which debit note pertains, or
- Date of filing GSTR-9, whichever is earlier

### 3.6 Bill of Supply (Rule 49)

**When to issue (instead of tax invoice):**
- Supplies of **exempt goods/services**
- Supplies by a **composition dealer**

**Mandatory fields:**
1. Supplier name, address, GSTIN
2. Serial number (up to 16 chars, consecutive)
3. Date of issue
4. Recipient details (name, address, GSTIN if registered)
5. HSN/SAC code
6. Description, quantity
7. Total value (inclusive of tax where applicable)
8. Signature
9. Must state: **"Composition taxable person, not eligible to collect tax on supplies"** (for composition dealers)

### 3.7 Delivery Challan (Rule 55)

**When to issue (goods transported without tax invoice):**
- Goods transported for **job work**
- Goods transported for **testing/inspection/approval**
- Goods transported for **exhibition or stock transfer**
- Goods transported **on approval basis** (supply not yet finalized)
- Liquid gas transport (quantity unknown at dispatch)
- Goods transported by pipeline

**Mandatory fields in Delivery Challan:**
1. Date and sequential challan number
2. Consignor: Name, address, GSTIN
3. Consignee: Name, address, GSTIN (if registered) or delivery address (if unregistered)
4. Place of supply
5. Description and quantity of goods
6. HSN code
7. Taxable value and applicable tax rates (if tax applicable)
8. E-way bill number (if goods value >₹50,000)
9. Signature

---

## 4. HSN / SAC Codes

### 4.1 HSN Digit Requirements by Turnover

| Annual Aggregate Turnover (AATO) | GSTR-1 | E-Invoice | E-Way Bill (B2B + Export) |
|----------------------------------|--------|-----------|--------------------------|
| Up to ₹5 crore | 4-digit HSN | 4-digit (min) | 4-digit (min) |
| Above ₹5 crore | 6-digit HSN | 6-digit (min) | 6-digit (min) |
| Export (all turnovers) | 8-digit | 8-digit preferred | 8-digit required |

**Phase III (from January 2025):** Manual HSN entry in GSTR-1 Table 12 is **not allowed** — must select from dropdown. System has split Table 12 into B2B and B2C tabs.

**Rule:** If AATO exceeds ₹5 crore in any FY from 2017-18 onwards, 6-digit HSN is mandatory in all subsequent years.

### 4.2 Common HSN Codes for E-Commerce Categories

#### Apparel & Clothing (Chapter 61 — Knitted; Chapter 62 — Non-knitted)

| HSN Code | Description | GST Rate |
|----------|-------------|---------|
| 6101 | Men's outer garments (knitted) | 5% (≤₹1,000), 12% (>₹1,000) |
| 6103 | Men's suits, trousers (knitted) | 5% (≤₹1,000), 12% (>₹1,000) |
| 6104 | Women's suits, dresses (knitted) | 5% (≤₹1,000), 12% (>₹1,000) |
| 6105 | Men's shirts (knitted) | 12% |
| 6106 | Women's blouses (knitted) | 5%/12% |
| 6109 | T-shirts, singlets (knitted) | 5% (≤₹1,000), 12% (>₹1,000) |
| 6201 | Men's overcoats (non-knitted) | 5% (≤₹1,000), 12% (>₹1,000) |
| 6203 | Men's suits, trousers (non-knitted) | 5%/12% |
| 6204 | Women's suits, dresses (non-knitted) | 5%/12% |
| 6211 | Track suits, swimwear | 5%/12% |

> Apparel cut-off: ≤₹1,000 per piece = 5%; >₹1,000 per piece = 12% (post-rationalization retained)

#### Electronics (Chapter 84, 85)

| HSN Code | Description | GST Rate |
|----------|-------------|---------|
| 8471 | Computers, laptops | 18% |
| 84713020 | Laptops (8-digit) | 18% |
| 8517 | Mobile phones | 18% |
| 85171200 | Mobile phones (8-digit) | 18% |
| 8518 | Headphones, earphones, speakers | 18% |
| 8528 | TV monitors/sets | 18% (post-rationalization from 28%) |
| 8415 | Air conditioners | 18% (post-rationalization from 28%) |
| 8509 | Kitchen appliances (mixer, grinder) | 18% |
| 8516 | Electric irons, hair dryers | 18% |

#### Beauty & Personal Care (Chapter 33)

| HSN Code | Description | GST Rate |
|----------|-------------|---------|
| 3304 | Beauty/make-up preparations, skin care | 18% |
| 33041000 | Lip make-up | 18% |
| 33042000 | Eye make-up | 18% |
| 33043000 | Manicure/pedicure preparations | 18% |
| 33049110 | Face powder | 18% |
| 33049920 | Nail polish | 18% |
| 33049930 | Moisturising lotion | 18% |
| 3305 | Hair preparations (shampoo, hair oil, conditioner) | 5% (post-rationalization from 18%) |
| 3301-3303 | Essential oils, perfumery | 18% |
| 3306 | Oral hygiene products (toothpaste) | 5% (post-rationalization) |
| 3307 | Shaving preparations, deodorants | 18% |

#### Home Goods & Furniture (Chapter 94)

| HSN Code | Description | GST Rate |
|----------|-------------|---------|
| 9401 | Seating furniture (chairs, sofas) | 18% |
| 9403 | Other furniture (tables, wardrobes, beds) | 18% |
| 9404 | Mattresses, pillows, quilts | 12% |
| 9405 | Lamps, lighting fittings | 12% |
| 6302 | Bed linen, table linen, kitchen linen | 5% (≤₹1,000), 12% (>₹1,000) |
| 6303 | Curtains, blinds | 5%/12% |
| 3924 | Plasticware (kitchen containers) | 18% |
| 7013 | Glassware (household) | 18% |
| 6911-6914 | Ceramics (tableware, kitchenware) | 12% |

### 4.3 HSN Summary in GSTR-1 (Table 12)

GSTR-1 Table 12 requires HSN-wise summary of all outward supplies:
- **Table 12A (B2B tab):** HSN-wise summary for B2B supplies — mandatory if any B2B supply exists; if no B2B supplies, enter one row with a HSN code and all zeros
- **Table 12B (B2C tab):** HSN-wise summary for B2C supplies

**Fields required in Table 12:**
1. HSN Code
2. UQC (Unit Quantity Code)
3. Total Quantity
4. Total Value
5. Total Taxable Value
6. Integrated Tax (IGST)
7. Central Tax (CGST)
8. State/UT Tax (SGST/UTGST)
9. Cess

### 4.4 SAC Codes (Services)

Service Accounting Codes are 6-digit codes starting with "99":

| SAC Code | Service Category |
|----------|----------------|
| 9983 | IT services, software development |
| 9984 | Telecom services |
| 9985 | Support services (logistics, warehousing) |
| 9986 | Agriculture support services |
| 9987 | Maintenance/repair services |
| 9988 | Manufacturing on job work |
| 9992 | Education services |
| 9993 | Healthcare services |
| 9997 | Other services |

---

## 5. GST Return Filing

### 5.1 GSTR-1 — Outward Supply Details

**Who files:** All registered taxpayers (except composition dealers, ISD, TDS deductors, TCS collectors)

**Frequency & Due Dates:**

| Turnover | Filing Frequency | Due Date |
|---------|-----------------|---------|
| >₹5 crore (AATO) | Monthly | **11th of next month** |
| ≤₹5 crore (QRMP scheme opted) | Quarterly | **13th of month after quarter** |
| ≤₹5 crore (QRMP, first 2 months) | IFF (optional) | **13th of following month** |

**GSTR-1 Table Structure:**

| Table | Description | When to Use |
|-------|-------------|-------------|
| 1, 2, 3 | GSTIN, legal/trade name, previous year turnover | Always |
| **4** | Taxable outward supplies to **registered persons** (B2B) — incl. debit/credit notes, exports to registered | All B2B invoices |
| **5** | Taxable inter-state outward supplies to **unregistered persons** where invoice value >₹2.5 lakh | B2C Large invoices |
| **6** | Zero-rated supplies and deemed exports | Exports, SEZ supplies |
| **7** | Taxable supplies to unregistered persons (other than Table 5) — net of debit/credit notes | B2C Small (state-wise summary) |
| **8** | Nil-rated, exempt, and non-GST outward supplies | Nil/exempt supplies |
| **9** | Amendments to B2B supplies from prior periods | Corrections |
| **10** | Amendments to B2C Large from prior periods | Corrections |
| **11** | Advances received / adjusted against invoices | Advance receipts |
| **12** | HSN-wise summary (Tab 12A = B2B, Tab 12B = B2C) | Mandatory always |
| **13** | Documents issued (invoices, credit/debit notes, delivery challans) | Document count |
| **14** | For **suppliers** — ECO-GSTIN-wise sales through e-commerce operators (TCS/Section 9(5)) | E-commerce sellers |
| **14A** | Amendments to Table 14 | Corrections |
| **15** | For **ECOs** — B2B and B2C supplier-GSTIN-wise sales (TCS + Section 9(5)) | E-commerce operators |
| **15A** | Amendments to Table 15 | Corrections |

**B2C Large vs B2C Small:**
- **B2C Large (Table 5):** Inter-state B2C invoices with value **>₹2.5 lakh** (invoice-level reporting)
- **B2C Small (Table 7):** All other B2C (intra-state all amounts, inter-state ≤₹2.5 lakh) — **consolidated state-wise reporting**

### 5.2 Invoice Furnishing Facility (IFF) — For QRMP Scheme

QRMP scheme quarterly filers can use IFF to upload B2B invoices for the **first two months** of each quarter (so buyers can claim ITC sooner):
- IFF is **optional** (not mandatory)
- Due date: **13th of the following month**
- Only B2B invoices can be uploaded via IFF (not B2C)
- The quarterly GSTR-1 filed on 13th of month after quarter-end covers remaining invoices (including any not uploaded via IFF, and Month 3 invoices)

### 5.3 GSTR-3B — Summary Return

**Who files:** All regular registered taxpayers (monthly or quarterly under QRMP)

**Due Dates:**

| Category | Due Date |
|----------|---------|
| Monthly filers (turnover >₹5 crore) | **20th of following month** |
| Quarterly filers, Category A states (QRMP) | **22nd of month following quarter** |
| Quarterly filers, Category B states (QRMP) | **24th of month following quarter** |

> Category A/B state classification is based on principal place of business — check GST portal for current list.

**GSTR-3B Table Structure:**

#### Table 3.1 — Outward Supplies and Inward Supplies on Reverse Charge

| Sub-table | Description | What to Enter |
|-----------|-------------|--------------|
| 3.1(a) | Outward taxable supplies (other than zero-rated, nil, exempt) | All normal taxable supplies; Value of invoices + debit notes - credit notes + advances (net); IGST, CGST, SGST, UTGST, Cess |
| 3.1(b) | Outward taxable supplies (zero-rated) | Exports and SEZ supplies; tax paid on these (IGST only) |
| 3.1(c) | Other outward supplies (nil rated, exempt) | Exempt and nil-rated supply value only |
| 3.1(d) | Inward supplies liable to reverse charge | Purchases from unregistered dealers, RCM imports — self-assessed tax |
| 3.1(e) | Non-GST outward supplies | Alcohol, petroleum, etc. — value only |

#### Table 3.1.1 — Section 9(5) E-Commerce Supplies (from August 2022)

| Sub-table | Who fills | What |
|-----------|-----------|------|
| 3.1.1(i) | **E-commerce operator** | Taxable value and tax for all Section 9(5) services (restaurant, cab, accommodation, housekeeping) — ECO pays tax in cash |
| 3.1.1(ii) | **Service supplier** | Same supplies — shows value but does NOT include tax (tax is ECO's liability) |

**Critical:** Do NOT include Section 9(5) supplies in Table 3.1 — double reporting risk.

#### Table 3.2 — Inter-State Supplies to Special Categories

Breakdown of supplies in Table 3.1(a) and 3.1.1(i) made to:
- Unregistered persons (state-wise)
- Composition taxable persons (state-wise)
- UIN holders (embassies, UN bodies)

> **From November 2025 onwards:** Values in Table 3.2 are **auto-populated** from GSTR-1/IFF and are **non-editable** in GSTR-3B. Must be corrected via GSTR-1A.

#### Table 4 — Eligible ITC

| Sub-table | Description |
|-----------|-------------|
| 4A(1) | ITC on import of goods |
| 4A(2) | ITC on import of services |
| 4A(3) | ITC on inward supplies on reverse charge (other than imports) |
| 4A(4) | ITC from Input Service Distributor (ISD) |
| 4A(5) | All other ITC (domestic purchases) |
| 4B(1) | ITC reversed as per Rules 38, 42, 43 and Section 17(5) |
| 4B(2) | Other ITC reversals |
| 4C | Net ITC available = 4A minus 4B (auto-calculated) |
| 4D(1) | ITC reclaimed (previously reversed under 4B(2)) |
| 4D(2) | Ineligible ITC under Section 16(4) and PoS restrictions |

**From July 2025:** Auto-populated ITC in Table 4A is **hard-locked** (cannot be manually edited upward, only downward); based on GSTR-2B.

#### Table 5 — Exempt, Nil-Rated, Non-GST Inward Supplies

- Purchases from composition dealers (inter-state / intra-state)
- Exempt inward supplies
- Nil-rated inward supplies
- Non-GST inward supplies (petroleum, alcohol)

#### Table 6 — Payment of Tax

| Sub-table | Description |
|-----------|-------------|
| 6.1 | Tax payable (matches 3.1(a) and 3.1.1) — shown per IGST, CGST, SGST, UTGST, Cess |
| 6.1 (ITC credit column) | ITC offset from Table 4C |
| 6.1 (Balance) | Cash to be paid |
| **6.2** | **TDS/TCS Credit received** (from ECO's GSTR-8; reflects in Electronic Cash Ledger) |

**Late Fees:**
| Category | Late Fee |
|----------|---------|
| General | ₹50/day (₹25 CGST + ₹25 SGST) |
| Nil return | ₹20/day (₹10 CGST + ₹10 SGST) |
| Interest on late tax payment | 18% per annum |
| 3-year filing restriction | Cannot file GSTR-3B after 3 years from due date (from July 2025 onwards) |

### 5.4 GSTR-2A vs GSTR-2B

| Feature | GSTR-2A | GSTR-2B |
|---------|---------|---------|
| Nature | Dynamic, real-time | **Static** — locked on generation date |
| When generated | Continuously updated when suppliers file | **14th of every month** (for monthly filers) |
| Purpose | Reference only | **Official basis for ITC claims** |
| ITC validity | Cannot use as final ITC basis | **Mandatory source for ITC in GSTR-3B** |
| For GSTR-9 | No longer used (replaced) | **Used for ITC reconciliation in GSTR-9 (from FY 2023-24)** |

**GSTR-2B Generation Timeline:**
- Generated on 14th of Month M+1 for Month M
- Contains all supplier GSTR-1 filings from 12:00AM on 14th of Month M to 11:59PM on 13th of Month M+1
- Example: GSTR-2B for March 2026 → generated on 14th April 2026 → contains supplier filings from 14 March 2026 to 13 April 2026

**GSTR-2B for quarterly filers:** Generated on 14th of the month following the quarter end.

**Rule 36(4):** ITC cannot be claimed on invoices NOT appearing in GSTR-2B (provisional ITC no longer permitted from FY 2022-23 onwards).

### 5.5 GSTR-9 — Annual Return

**Applicability:**

| Turnover | GSTR-9 | GSTR-9C (Reconciliation) |
|---------|--------|--------------------------|
| ≤₹2 crore | Optional (exempt) | Not applicable |
| >₹2 crore and ≤₹5 crore | **Mandatory** | Optional (self-certified) |
| >₹5 crore | **Mandatory** | **Mandatory** (self-certified) |

**Due Date:** 31st December of the following financial year (e.g., FY 2025-26 → 31 Dec 2026)

**Not required to file GSTR-9:** Composition dealers (file GSTR-4), casual taxable persons, ISD, non-resident taxable persons, TDS deductors (Section 51), TCS collectors (Section 52)

**Late Fee:** ₹200/day (₹100 CGST + ₹100 SGST), capped at 0.25% of turnover

**Key Tables in GSTR-9:**
- Tables 4/5/6: Outward supply details (from GSTR-1)
- Tables 6/7/8: ITC details — Table 8A auto-populated from **GSTR-2B** (as of FY 2023-24 onwards per Notification 20/2024)
- Table 9: Tax paid as declared in GSTR-3B
- Tables 10-14: Amendments, reversals

### 5.6 QRMP Scheme Summary

| Feature | Details |
|---------|---------|
| Eligibility | AATO ≤₹5 crore in current and preceding FY |
| Returns | Quarterly GSTR-1 (13th) + Quarterly GSTR-3B (22nd/24th) |
| Tax payment | **Monthly PMT-06 challan** by 25th of each month (for first 2 months of quarter) |
| IFF | Optional B2B invoice upload for months 1 and 2 of quarter (due 13th) |
| Auto-assignment | System assigns QRMP to eligible taxpayers; can opt-out |

---

## 6. E-Invoicing

### 6.1 Current Threshold & Scope

| Parameter | Details |
|-----------|---------|
| **Current threshold** | AATO > **₹5 crore** in any FY from 2017-18 (effective 1 August 2023) |
| **Applicable transactions** | B2B invoices, B2G invoices, exports, credit notes, debit notes, RCM invoices |
| **NOT applicable** | B2C invoices, composition dealers, SEZ units, insurance companies, banking companies, NBFCs, Goods Transport Agencies, passenger transport services |
| **Archiving** | 72 months (6 years) — signed JSON is the legally valid document (not the PDF) |

**Threshold history:**
- Oct 2020: ₹500 crore
- Jan 2021: ₹100 crore
- Apr 2021: ₹50 crore
- Apr 2022: ₹20 crore
- Oct 2022: ₹10 crore
- Aug 2023: **₹5 crore (current)**

### 6.2 E-Invoice Process Flow

```
1. Seller creates invoice in ERP/billing system
   ↓
2. Seller's system generates JSON in prescribed schema (INV-1 v1.1)
   ↓
3. JSON uploaded to IRP via API / web portal / mobile / GSP
   ↓
4. IRP validates: GSTIN check, duplicate check, mandatory fields
   ↓
5. IRP generates:
   - IRN (64-char hash)
   - Digitally signed invoice (JWT)
   - QR code (signed JWT)
   - Acknowledgement Number + Date
   ↓
6. Signed JSON returned to seller
   ↓
7. Seller prints IRN + QR code on physical invoice
   ↓
8. Auto-population to GSTR-1 (within T+2 days)
   ↓
9. E-Way Bill auto-generated if required (for goods movement >₹50,000)
```

### 6.3 IRP (Invoice Registration Portal)

**Production portals (multiple IRPs available from March 2021):**

| IRP | URL | Operator |
|-----|-----|---------|
| IRP-1 (NIC) | https://einvoice1.gst.gov.in | NIC |
| IRP-2 (NIC) | https://einvoice2.gst.gov.in | NIC |
| IRP-6 (IRIS) | https://einvoice6.gst.gov.in | IRIS Business |
| IRP-7 (ClearTax) | https://einvoice7.gst.gov.in | ClearTax |

**Sandbox/Testing:**
- https://einv-apisandbox.nic.in (NIC sandbox)

### 6.4 IRN (Invoice Reference Number)

**Algorithm:** Hash computed by IRP using:
```
SHA-256 hash of: Supplier GSTIN + Financial Year + Document Type + Document Number
Result: 64-character hexadecimal string
```

**Example IRN:** `11f8ef701fe294d4a14aad0b12457e62775d0fdc41a0acf05b74fbb2ddc47acb`

**Properties:**
- Unique across all IRPs
- Once generated, invoice cannot be modified (only cancelled within 24 hours)
- Cancellation window: **24 hours** from IRN generation
- After 24 hours, adjustments only via credit notes

### 6.5 E-Invoice JSON Schema (INV-1 Version 1.1)

**Top-level structure:**
```json
{
  "Version": "1.1",
  "Irn": "64-char hash (filled by IRP)",
  "TranDtls": { ... },    // Transaction details
  "DocDtls": { ... },     // Document details
  "SellerDtls": { ... },  // Supplier/seller details
  "BuyerDtls": { ... },   // Buyer/recipient details
  "DispDtls": { ... },    // Dispatch-from details (optional)
  "ShipDtls": { ... },    // Ship-to details (optional)
  "ItemList": [ ... ],    // Line items array (max 1000; 5000 on request)
  "ValDtls": { ... },     // Value/totals
  "PayDtls": { ... },     // Payment details (optional)
  "RefDtls": { ... },     // Reference details (optional)
  "AddlDocDtls": { ... }, // Additional document info (optional)
  "ExpDtls": { ... },     // Export details (optional)
  "EwbDtls": { ... }      // E-Way Bill details (optional)
}
```

**30 Mandatory Fields:**

| # | Field | Max Length | Specification |
|---|-------|-----------|---------------|
| 1 | Document Type Code (TranDtls.SupTyp) | 10 | `INV`, `CRN`, `DBN` |
| 2 | Supply Type (TranDtls.SupTyp) | 10 | `B2B`, `SEZWP`, `SEZWOP`, `EXPWP`, `EXPWOP`, `DEXP` |
| 3 | Supplier Legal Name (SellerDtls.LglNm) | 100 | As per PAN |
| 4 | Supplier GSTIN (SellerDtls.Gstin) | 15 | Alphanumeric |
| 5 | Supplier Address (SellerDtls.Addr1) | 100 | Building, street, locality |
| 6 | Supplier Place/City (SellerDtls.Loc) | 50 | City/town/village |
| 7 | Supplier State Code (SellerDtls.Stcd) | 2 | State code from master list |
| 8 | Supplier Pincode (SellerDtls.Pin) | 6 | 6-digit code |
| 9 | Document Number (DocDtls.No) | 16 | Sequential, unique per GSTIN per FY |
| 10 | Document Date (DocDtls.Dt) | 10 | DD/MM/YYYY format |
| 11 | Preceding Invoice Reference (PrecDocDtls.InvNo) | 16 | For credit/debit notes |
| 12 | Recipient Legal Name (BuyerDtls.LglNm) | 100 | As per PAN |
| 13 | Recipient GSTIN (BuyerDtls.Gstin) | 15 | Or "URP" for unregistered |
| 14 | Recipient Address (BuyerDtls.Addr1) | 100 | |
| 15 | Recipient State Code (BuyerDtls.Stcd) | 2 | State code |
| 16 | Place of Supply State Code (BuyerDtls.Pos) | 2 | Enumerated state codes |
| 17 | Recipient Pincode (BuyerDtls.Pin) | 6 | 6-digit |
| 18 | Recipient Place (BuyerDtls.Loc) | 100 | City/town/village |
| 19 | Ship-To GSTIN (ShipDtls.Gstin) | 15 | GSTIN of delivery recipient |
| 20 | Ship-To State/Pincode/State Code (ShipDtls) | 100/6/2 | Delivery location |
| 21 | Dispatch From Details (DispDtls) | 100 each | Name, address, place, pincode |
| 22 | Is Service (ItemList[n].IsServc) | 1 | Y/N |
| 23 | Supply Type Code | 10 | B2B/B2C/SEZWP/SEZWOP etc. |
| 24 | Item Description (ItemList[n].PrdDesc) | 300 | |
| 25 | HSN Code (ItemList[n].HsnCd) | 8 | 4 or 6 digit per turnover rules |
| 26 | Item Unit Price (ItemList[n].UnitPrice) | Decimal(12,3) | Per unit, ex-GST |
| 27 | Assessable Value (ItemList[n].AssAmt) | Decimal(13,2) | Net price after discount |
| 28 | GST Rate (ItemList[n].GstRt) | Decimal(3,2) | In percentage (e.g., 18) |
| 29 | IGST/CGST/SGST Amounts (ItemList[n]) | Decimal(11,2) each | Per item, separately |
| 30 | Total Invoice Value (ValDtls.TotInvVal) | Decimal(11,2) | Grand total incl. GST |

**TranDtls key fields:**
```json
"TranDtls": {
  "TaxSch": "GST",                    // Always "GST"
  "SupTyp": "B2B",                    // B2B/SEZWP/SEZWOP/EXPWP/EXPWOP/DEXP
  "RegRev": "N",                      // Y if reverse charge
  "EcmGstin": "29AAACK1234Z1Z1",      // GSTIN of e-commerce operator (optional)
  "IgstOnIntra": "N"                  // Y if IGST on intra-state supply
}
```

### 6.6 IRP API Authentication (NIC API v1.03)

**Base URL (Sandbox):** `https://einv-apisandbox.nic.in`  
**Base URL (Production):** `https://einvoice1.gst.gov.in` (IRP-1)

**Authentication Endpoint:**
```
POST /api/auth
Content-Type: application/json
Headers:
  client_id: [provided by E-Invoice System]
  client_secret: [provided by E-Invoice System]

Body:
{
  "data": {
    "UserName": "taxpayer_username",
    "Password": "RSA-encrypted(password, IRP_PublicKey)",
    "AppKey": "RSA-encrypted(32_char_random_key, IRP_PublicKey)",
    "ForceRefreshAccessToken": false
  }
}
```

**Auth Response:**
```json
{
  "Status": "1",
  "Data": {
    "ClientId": "...",
    "UserName": "...",
    "AuthToken": "Bearer_token_here",
    "Sek": "AES256_encrypted_session_key",
    "TokenExpiry": "2026-03-28 20:00:00"
  }
}
```
- Token valid: **360 minutes** (6 hours) in production; 60 minutes in sandbox
- Session Encryption Key (Sek): AES-256 encrypted with AppKey; used to encrypt invoice JSON data

**Generate IRN Endpoint:**
```
POST /api/Invoice/Generate
Headers:
  client_id: [client_id]
  client_secret: [client_secret]
  Gstin: [supplier GSTIN]
  user_name: [username]
  AuthToken: [token from auth]

Body:
{
  "Data": "[Base64_encoded_AES256_encrypted_invoice_JSON]"
}
```

**IRN Response (Success):**
```json
{
  "Status": "1",
  "Data": {
    "AckNo": 112010000002315,
    "AckDt": "2026-03-28 15:18:00",
    "Irn": "11f8ef701fe294d4a14aad0b12457e62775d0fdc41a0acf05b74fbb2ddc47acb",
    "SignedInvoice": "[JWT_signed_invoice]",
    "SignedQRCode": "[JWT_signed_QR_data]",
    "Status": "ACT",
    "EwbNo": 191008688443,
    "EwbDt": "2026-03-28 15:18:00",
    "EwbValidTill": "2026-03-29 23:59:00"
  }
}
```

### 6.7 QR Code Requirements

**For B2B e-invoices:** QR code generated by IRP (not the supplier). Self-generated QR codes on B2B e-invoices are **void**.

**QR Code data elements (minimum):**
1. GSTIN of supplier (`SellerGstin`)
2. GSTIN of buyer (`BuyerGstin`)
3. Invoice number (`DocNo`)
4. Invoice type (`DocTyp`)
5. Invoice date (`DocDt`)
6. Total invoice value (`TotInvVal`)
7. Number of line items (`ItemCnt`)
8. HSN code of main commodity (`MainHsnCode`)
9. IRN (`Irn`)

**Format:** The signed QR code is a JWT (JSON Web Token) signed with IRP's RSA private key; verifiable using IRP's public key.

**For B2C invoices >₹500 (mandatory QR):** Self-generated QR code (not IRP-generated) with payment link; must include same invoice details except buyer GSTIN (replaced with name).

### 6.8 Auto-Population of GSTR-1 from E-Invoices

- Happens within **T+2 days** of IRN generation
- Incremental population — each e-invoice auto-populates the respective GSTR-1 table:

| E-Invoice Type | GSTR-1 Table Auto-Populated |
|---------------|----------------------------|
| B2B taxable (non-RCM) | Table 4A |
| B2B reverse charge | Table 4B |
| Exports with payment | Table 6A |
| Exports without payment | Table 6B |
| Credit notes (registered) | Table 9B |

- If taxpayer edits auto-populated data in GSTR-1, the `Source`, `IRN`, and `IRN Date` fields reset to blank
- Cancellation of IRN deletes the entry from GSTR-1

### 6.9 30-Day IRN Upload Limit (from 1 April 2025)

| Turnover | 30-Day Rule | Effective Date |
|---------|------------|----------------|
| AATO > ₹100 crore | Must report invoices within 30 days | 1 November 2023 |
| AATO > ₹10 crore | Must report invoices within 30 days | **1 April 2025** |
| AATO > ₹5 crore but ≤₹10 crore | No time limit yet (expected to reduce) | TBD |

**Consequence of missing 30-day window:** IRP rejects the invoice — IRN cannot be generated; buyer loses ITC.

---

## 7. Input Tax Credit (ITC)

### 7.1 Conditions for Claiming ITC (Section 16)

All four conditions must be met simultaneously:
1. **Possession of tax invoice** or debit note issued by supplier
2. **Receipt of goods or services** (delivery completed)
3. **Tax actually paid** by supplier to the government
4. **GSTR-3B filed** for the period

**Additional requirement (Rule 36(4) as amended):** Invoice must appear in **GSTR-2B** — provisional ITC beyond GSTR-2B is no longer permitted.

**Time limit for ITC claims (Section 16(4)):**  
ITC for any invoice/debit note cannot be claimed after the **earlier of:**
- 30th November following the end of the FY to which the invoice pertains
- Date of filing GSTR-9 for that FY

### 7.2 ITC Matching with GSTR-2B

**Monthly Workflow:**
```
14th: GSTR-2B generated (supplier-reported ITC locked)
14th-20th: Reconcile GSTR-2B vs Purchase Register
→ Category 1: Match (same GSTIN, invoice no., value) → Claim ITC
→ Category 2: In GSTR-2B not in books → Book invoice, claim ITC
→ Category 3: In books not in GSTR-2B → Follow up supplier; do NOT claim
→ Category 4: Amount mismatch → Supplier amendment needed; claim confirmed amount only
→ Category 5: Blocked credit (Section 17(5)) → Reverse ITC; document reason
20th: File GSTR-3B with reconciled ITC
```

### 7.3 Blocked ITC — Section 17(5) Complete List

| Clause | Category | ITC NOT Allowed On | Exceptions |
|--------|----------|-------------------|-----------|
| (a)/(aa)/(ab) | Motor vehicles | Vehicles ≤13 seats (cars, 2W, 3W), ships, aircraft insurance, repairs | Allowed if used for transportation business, driving school, vehicle manufacturing/selling |
| (b) | Food, catering, personal use | Food & beverages, outdoor catering, health/beauty treatment, cosmetic surgery, renting vehicles, life/health insurance, club membership, LTC | Allowed if part of composite supply, mandatory under law (e.g., Factories Act canteen), or resold |
| (c)/(d) | Construction | Building construction/renovation, materials capitalized in books | Allowed for builders reselling property; allowed for plant & machinery |
| (e) | Composition | Composition scheme taxpayers | N/A |
| (f) | Non-resident | Non-resident taxable person's domestic purchases | Allowed on IGST paid on goods import |
| (g) | Personal use | Goods/services used for personal purposes | ITC allowed proportionately for business use |
| (h) | Lost/destroyed | Goods lost, stolen, destroyed, written off, free samples, gifts | No exception |
| (i) | Fraud (up to FY 2023-24) | Tax paid due to fraud, misstatement, suppression of facts | No exception |

**Reporting blocked ITC in GSTR-3B:** Table 4(B)(1) — reversal of blocked credits.

### 7.4 ITC Reversal — Rule 42 (Inputs/Input Services)

**Applicable when:** Inputs/input services used for **both taxable and exempt supplies**.

**Monthly provisional reversal formula (Rule 42(1)):**
```
Step 1: Separate ITC into:
  T1 = ITC exclusively for non-business use → full reversal
  T2 = ITC exclusively for exempt supplies → full reversal
  T3 = ITC as per Section 17(5) blocked credit → full reversal
  T4 = ITC exclusively for taxable supplies → no reversal
  C1 = Total ITC - T1 - T2 - T3 (common credit)
  T5 = ITC exclusively for taxable supplies (out of C1) → retain
  C2 = C1 - T5 (remaining common credit)

Step 2: Monthly reversal
  D1 = C2 × (E/F)
  where E = total exempt turnover for period
        F = total turnover for period
  D2 = C2 - D1 (eligible ITC from common)

Step 3: Annual true-up (by 20th November)
  Compare: Σ D1 monthly vs. Annual D1 (using full FY ratios)
  If excess ITC claimed → pay back with interest
  If excess reversed → reclaim in October return
```

### 7.5 ITC Reversal — Rule 43 (Capital Goods)

**Applicable when:** Capital goods used for **both taxable and exempt supplies**.

**Method:**
```
Useful life: 60 months (5 years)

Monthly notional credit = Total ITC on CG / 60 = Tc/60

Monthly reversal = (Tc/60) × (E/F)
where E = exempt turnover for month
      F = total turnover for month

Annual true-up: similar to Rule 42
```

### 7.6 ITC on E-Commerce Purchases (Seller's Perspective)

| Purchase Type | ITC Available |
|--------------|--------------|
| Inventory (goods for resale) | ✅ Yes — claim in GSTR-3B Table 4A(5) |
| Packaging materials | ✅ Yes |
| Marketplace fees/commissions | ✅ Yes — as input service |
| Logistics/shipping costs | ✅ Yes |
| Office expenses (non-blocked) | ✅ Yes |
| Motor vehicle (personal use) | ❌ No — blocked under Section 17(5) |
| Food/entertainment | ❌ No — blocked |
| Personal use items | ❌ No |

---

## 8. TDS / TCS in GST

### 8.1 TCS under Section 52 (E-Commerce Operators)

| Parameter | Details |
|-----------|---------|
| **Who deducts** | E-Commerce Operators (Amazon, Flipkart, Meesho, etc.) |
| **On what** | Net value of taxable supplies made by third-party sellers through the platform |
| **Rate** | 1% (0.5% CGST + 0.5% SGST for intra-state; 1% IGST for inter-state) |
| **Base** | Net taxable value = gross sales − returns/cancellations for the month |
| **When** | At time of credit or payment to seller, whichever is earlier |
| **Government deposit deadline** | **10th of the following month** |
| **Return by ECO** | **GSTR-8** (monthly, by 10th) — supplier GSTIN-wise breakdown |
| **Annual return by ECO** | GSTR-9B (currently on hold) |
| **Credit to seller** | Appears in seller's **Electronic Cash Ledger** |
| **Seller claims credit** | In GSTR-3B Table 6.2 (TDS/TCS Credit) |

**Electronic Cash Ledger vs Electronic Credit Ledger:**
- TCS credit → **Electronic Cash Ledger** (can only offset cash liability, not ITC)
- Regular ITC → **Electronic Credit Ledger**

**Reconciliation requirement (seller):**
1. Download GSTR-2A/Electronic Cash Ledger data
2. Match with settlement statements from marketplace
3. Report TCS received in GSTR-3B Table 6.2
4. Apply TCS credit against GST payable

### 8.2 TDS under Section 51 (Government Entities)

| Parameter | Details |
|-----------|---------|
| **Who deducts** | Central/State Government departments, local authorities, government agencies, PSUs with ≥51% government equity, certain notified bodies |
| **On what** | Taxable supply under a single contract exceeding **₹2.5 lakh** (excluding GST) |
| **Rate** | 2% (1% CGST + 1% SGST for intra-state; 2% IGST for inter-state) |
| **When** | At time of payment to supplier |
| **Government deposit** | Within **10 days** of end of month in which deduction was made |
| **Return by deductor** | **GSTR-7** (monthly) |
| **Certificate to supplier** | Form GSTR-7A |
| **No TDS when** | Location of supplier and place of supply are in different states from state of registration of recipient |

**Note:** Section 51 TDS does not directly apply to most e-commerce B2C transactions — relevant when selling to government departments.

### 8.3 Electronic Ledgers in GST

| Ledger | What Goes In | Used For |
|--------|-------------|---------|
| **Electronic Tax Liability Ledger** | GST payable per return | Tracks total liability |
| **Electronic Credit Ledger** | ITC claimed in GSTR-3B | Offset against output liability |
| **Electronic Cash Ledger** | Cash deposited + TCS/TDS credits received | Cash payment of remaining liability |

**Settlement order for liability:**
1. IGST from IGST credit
2. IGST from CGST credit
3. IGST from SGST credit
4. CGST from CGST credit
5. SGST from SGST credit
6. CGST from IGST credit
7. SGST from IGST credit
8. Balance from Electronic Cash Ledger

---

## 9. Income Tax for E-Commerce

### 9.1 Section 194-O — Income Tax TDS by E-Commerce Operators

| Parameter | Details |
|-----------|---------|
| **Who deducts** | E-Commerce operators (same as GST TCS entities) |
| **Rate (with PAN)** | **0.1%** (reduced from 1% effective **1 October 2024**) |
| **Rate (without PAN)** | **5%** (Section 206AA) |
| **Rate (without PAN, non-resident)** | Exempt from Section 194-O |
| **Base** | Gross amount of sale of goods or provision of services facilitated |
| **Threshold** | ₹5 lakh per year per e-commerce operator (applies to individual/HUF sellers only) |
| **No threshold for** | Companies and partnership firms — TDS from first payment |
| **Non-residents** | Exempt from Section 194-O |
| **Filing form** | **Form 26QE** by e-commerce operator |
| **Quarterly filing deadline** | 31 Jul / 31 Oct / 31 Jan / 31 May |
| **Seller visibility** | Form 26AS Part A / AIS |

**Key distinction from GST TCS:**
- GST TCS (Section 52): 1% of net taxable value → credited to GST Electronic Cash Ledger
- Income Tax TDS (Section 194-O): 0.1% of gross sales → credited to seller's income tax account (Form 26AS); claimed in ITR

**Note on interplay:** Both GST TCS and Income Tax TDS are deducted by the e-commerce operator independently. They serve different purposes — GST TCS for GST compliance; Income Tax TDS for income tax compliance.

### 9.2 Section 44AD — Presumptive Taxation for E-Commerce Sellers

| Parameter | Details |
|-----------|---------|
| **Who can use** | Resident individuals, HUFs, partnerships (not LLPs) in eligible businesses |
| **Turnover limit** | ≤ ₹2 crore (≤ ₹3 crore if cash receipts ≤5% of total receipts — from AY 2024-25) |
| **Presumptive profit (cash transactions)** | **8%** of turnover |
| **Presumptive profit (digital/banking transactions)** | **6%** of turnover |
| **No books required** | No detailed P&L required if opting for presumptive taxation |
| **Return form** | ITR-4 (Sugam) |
| **Opt-out penalty** | If assessee declares less than 8%/6% in any year, cannot use 44AD for the **next 5 years** |
| **Advance tax** | Single installment by **15th March** (instead of quarterly) — special benefit |

**Not eligible for 44AD:** LLPs, companies, professionals, commission agents, specified businesses (agency, brokerage, etc.)

**E-commerce seller applicability:** Most small Shopify/D2C sellers below ₹2 crore turnover can opt for 44AD — significantly reduces compliance burden.

### 9.3 Section 44ADA — Presumptive Taxation for Professionals

| Parameter | Details |
|-----------|---------|
| **Who** | Resident professionals (doctors, lawyers, accountants, engineers, architects, consultants) |
| **Turnover limit** | ≤ ₹50 lakh (≤ ₹75 lakh if 95%+ receipts via banking) |
| **Presumptive profit** | **50%** of gross receipts |

### 9.4 Advance Tax for E-Commerce Sellers

**Quarterly installments (normal taxpayers):**

| Installment | Due Date | % of Annual Tax |
|------------|---------|----------------|
| 1st | 15th June | 15% |
| 2nd | 15th September | 45% (cumulative) |
| 3rd | 15th December | 75% (cumulative) |
| 4th | 15th March | 100% (cumulative) |

**Section 44AD benefit:** Only one installment — 100% by **15th March** of the relevant FY.

**Threshold:** Advance tax not required if total tax liability <₹10,000.

---

## 10. Recent Changes & Upcoming

### 10.1 GST 2.0 Rate Rationalization (56th GST Council, Effective 22 September 2025)

**Key structural changes:**
- Simplified from 5-slab system (0%, 5%, 12%, 18%, 28%) to effectively 3 slabs: 0%, 5%, 18%, 40%
- 12% slab absorbed mostly into 5% (for essential goods) and 18% (for standard goods)
- 28% slab absorbed mostly into 18% (consumer durables like AC, TV) with 40% for genuinely luxury/sin goods
- Compensation cess abolished for most goods (replaced by 40% rate for sin goods)

**Key product shifts relevant to e-commerce:**

| Category | Old Rate | New Rate |
|----------|----------|---------|
| ACs, TVs >32 inch, dishwashers, white goods | 28% | 18% |
| Cement | 28% | 18% |
| Chocolates, biscuits, hair oil, shampoo, soaps, mineral water | 18% | 5% |
| Mobile phones | 18% | **18% (unchanged)** |
| Footwear >₹2,500/pair | 12% | 5% |
| Some beauty services | 18% | 5% |
| Health & life insurance (individual) | 18% | **0% (exempt)** |
| Paneer, pizza bread, khakhra | 5% | **0% (exempt)** |

### 10.2 B2C E-Invoicing Pilot (2026-27)

- **September 2024:** 56th GST Council recommended B2C e-invoicing
- **Status:** Pilot expected in select states and sectors
- **Expected launch:** Voluntary B2C pilot in FY 2026-27; full rollout FY 2027-28
- **Mechanism:** B2C invoices will use self-generated QR codes (not IRP-validated IRN)
- **E-Way Bill integration:** B2C e-way bills to be integrated with e-invoicing system

### 10.3 HSN Dropdown Mandate on GST Portal (Phase III — January 2025)

- **Manual entry of HSN codes in GSTR-1 Table 12 is NO LONGER ALLOWED** from January 2025
- Must select from predefined dropdown list
- System auto-fills correct description when HSN selected
- Table 12 split into B2B and B2C tabs
- Validation checks introduced (in warning mode initially — not blocking filing)

### 10.4 30-Day IRN Upload Limit (from 1 April 2025)

- Businesses with AATO > ₹10 crore: must report e-invoices to IRP within **30 days** of invoice date
- Previously this limit only applied to ₹100 crore+ businesses (from 1 November 2023)
- Expected to extend to ₹5 crore+ businesses in FY 2026-27

### 10.5 GSTR-3B Hard Locking (from July 2025)

- Auto-populated liability in GSTR-3B is **hard-locked** (non-editable) from the July 2025 tax period
- From November 2025: Table 3.2 values (inter-state to unregistered) also become non-editable
- Corrections must be made in GSTR-1A/IFF before GSTR-3B filing

### 10.6 2FA (Two-Factor Authentication) for IRP (2026)

- Mandatory 2FA for e-invoice portal access for businesses with AATO > ₹10 crore
- Extends verification security to prevent unauthorized IRN generation

### 10.7 Section 194-O Rate Cut (1 October 2024)

Income Tax TDS under Section 194-O reduced from 1% to **0.1%** for e-commerce participants with valid PAN — significant compliance relief for sellers.

### 10.8 IMS (Invoice Management System) — 2024-25

- New system enabling recipients to **accept, reject, or mark pending** individual invoices from GSTR-2B
- Actions taken on IMS flow into GSTR-3B auto-population
- Reduces ITC mismatches and supplier-buyer disputes
- GSTR-9 for FY 2024-25 includes new fields for IMS-based ITC (per Notification 16/2025)

---

## Appendix A: GST State Codes

| State/UT | Code | State/UT | Code |
|----------|------|----------|------|
| Jammu & Kashmir | 01 | Himachal Pradesh | 02 |
| Punjab | 03 | Chandigarh | 04 |
| Uttarakhand | 05 | Haryana | 06 |
| Delhi | 07 | Rajasthan | 08 |
| Uttar Pradesh | 09 | Bihar | 10 |
| Sikkim | 11 | Arunachal Pradesh | 12 |
| Nagaland | 13 | Manipur | 14 |
| Mizoram | 15 | Tripura | 16 |
| Meghalaya | 17 | Assam | 18 |
| West Bengal | 19 | Jharkhand | 20 |
| Odisha | 21 | Chhattisgarh | 22 |
| Madhya Pradesh | 23 | Gujarat | 24 |
| Daman & Diu | 25 | Dadra & Nagar Haveli | 26 |
| Maharashtra | 27 | Andhra Pradesh (new) | 28 |
| Karnataka | 29 | Goa | 30 |
| Lakshadweep | 31 | Kerala | 32 |
| Tamil Nadu | 33 | Puducherry | 34 |
| Andaman & Nicobar | 35 | Telangana | 36 |
| Andhra Pradesh (residual) | 37 | Ladakh | 38 |
| Other Territory | 97 | Centre Jurisdiction | 99 |

---

## Appendix B: Key Compliance Calendar for E-Commerce Sellers

| Date | Obligation |
|------|-----------|
| **10th** | ECO deposits GST TCS (GSTR-8) |
| **11th** | GSTR-1 due (monthly filers, turnover >₹5 crore) |
| **13th** | GSTR-1 / IFF due (quarterly QRMP filers) |
| **14th** | GSTR-2B generated for prior month |
| **20th** | GSTR-3B due (monthly filers) |
| **22nd / 24th** | GSTR-3B due (quarterly QRMP filers, by state category) |
| **25th** | PMT-06 (monthly tax payment by QRMP filers for months 1 & 2 of quarter) |
| **30 Nov** | Last date for credit note issuance; ITC time limit for prior FY invoices |
| **31 Dec** | GSTR-9 annual return due |
| **15 Jun, 15 Sep, 15 Dec, 15 Mar** | Advance income tax installments |
| **15 Mar** | Advance tax (single installment for Section 44AD assessees) |
| **31 Jul** | Income tax return for non-audit assessees |
| **31 Oct** | Income tax return for audit cases |

---

## Appendix C: Penalties Reference

| Violation | Penalty |
|-----------|---------|
| Not obtaining GST registration (when mandatory) | ₹10,000 or tax due, whichever higher |
| Failure to issue invoice | ₹10,000 or tax amount, whichever higher |
| Omission of place of supply on invoice | Up to ₹25,000 |
| Not filing e-invoice / generating IRN | ₹10,000 or tax amount, whichever higher |
| Goods transported without e-way bill | Detention; penalty up to 100% of tax |
| Late filing of GSTR-3B | ₹50/day (₹20/day for nil return) + 18% p.a. interest |
| Late filing of GSTR-9 | ₹200/day capped at 0.25% of turnover |
| Wrong ITC claims | Tax + interest + penalty up to 100% |
| Fraud, deliberate misstatement | 100% penalty + criminal prosecution |

---

## Sources

1. [ClearTax — CGST/SGST/IGST/UTGST Explained](https://cleartax.in/s/what-is-sgst-cgst-igst)
2. [ClearTax — GST Rate Slabs 2026 (GST 2.0)](https://cleartax.in/s/gst-rates)
3. [ClearTax — Place of Supply of Goods](https://cleartax.in/s/place-of-supply-of-goods)
4. [CBIC Tax Information — Section 10 IGST Act](https://taxinformation.cbic.gov.in/content/html/tax_repository/gst/acts/2017_IGST_Act/active/chapterv/section10_v1.00.html)
5. [ClearTax — Composition Scheme](https://cleartax.in/s/gst-composition-scheme)
6. [GST Pro — E-Commerce GST Registration 2026](https://gstproindia.com/blogs/blog/gst-registration-for-ecommerce-sellers-do-you-need-it-2026)
7. [ClearTax — Section 9(5) Notified Services](https://cleartax.in/s/gst-on-notified-services-ecommerce-operators-95)
8. [ClearTax — TCS Section 52 Multiple ECO](https://cleartax.in/s/gst-tcs-multiple-ecommerce-operators-in-single-transaction)
9. [TaxGuru — Rule 46 Invoice Requirements](https://taxguru.in/goods-and-service-tax/tax-invoice-requirements-section-31-cgst-act-gst-rule-46.html)
10. [ClearTax — GST Invoice Details](https://cleartax.in/s/gst-invoice-details)
11. [ClearTax — Mandatory HSN Code Reporting in GSTR-1](https://cleartax.in/s/mandatory-hsn-code-reporting-gstr1-1a)
12. [ClearTax — GSTR-1 Filing](https://cleartax.in/s/gstr-1)
13. [ClearTax — GSTR-3B Details](https://cleartax.in/s/details-mentioned-form-gstr-3b)
14. [GST Portal — GSTR-3B FAQs](https://tutorial.gst.gov.in/userguide/returns/GSTR3B.htm)
15. [ClearTax — E-Invoicing 5 Crore Threshold](https://cleartax.in/s/e-invoicing-businesses-above-rs-5-crore-turnover)
16. [ClearTax — E-Invoice Format & Schema](https://cleartax.in/s/e-invoice-format-schema-template)
17. [NIC E-Invoice API Sandbox — Authentication](https://einv-apisandbox.nic.in/version1.03/authentication.html)
18. [NIC E-Invoice API Sandbox — Generate IRN](https://einv-apisandbox.nic.in/version1.03/generate-irn.html)
19. [ClearTax — Auto-Population of E-Invoice into GSTR-1](https://cleartax.in/s/auto-population-e-invoice-gstr-1)
20. [TallySolutions — E-Invoicing Threshold](https://tallysolutions.com/gst/e-invoicing-threshold/)
21. [Mark IT Solutions — 30-Day E-Invoice Rule April 2025](https://www.markitsolutions.in/blog-details/e-invoicing-threshold-reduced-new-rules-april-2025)
22. [TallySolutions — Section 17(5) Blocked Credits](https://tallysolutions.com/gst/blocked-credits-under-gst-section-175-what-you-cannot-claim/)
23. [ClearTax — ITC Reversal under GST](https://cleartax.in/s/itc-reversal-gst)
24. [TaxGuru — Rule 42/43 ITC Reversal](https://taxguru.in/goods-and-service-tax/gst-rule-42-43-reversal-itc-inputs-input-services-capital-goods.html)
25. [DisyTax — TDS under Section 51](https://disytax.com/tds-under-gst-section-51-applicability-gstr7/)
26. [ClearTax — Section 194-O TDS E-Commerce](https://cleartax.in/s/section-194o)
27. [Terra Insight — Section 194-O Reconciliation](https://www.terra-insight.com/insights/tds-section-194o-ecommerce-reconciliation/)
28. [ClearTax — Section 44AD Presumptive Taxation](https://cleartax.in/s/section-44ad-presumptive-scheme)
29. [ClearTax — GSTR-9 Annual Return](https://cleartax.in/s/gstr-9-annual-return)
30. [ClearTax — GSTR-2B Guide](https://cleartax.in/s/gstr-2b)
31. [Binary Semantics — GSTR-2B Replaces GSTR-2A for GSTR-9](https://www.binarysemantics.com/blogs/gstr-2b-replaces-gstr-2a-for-itc-reconciliation-in-gstr-9/)
32. [Vertex Inc. — India E-Invoicing Compliance](https://www.vertexinc.com/resources/resource-library/indias-e-invoicing-regulations-explained-scope-formats-and-penalties)
33. [VATcalc — India B2B E-Invoicing 30-Day Reporting](https://www.vatcalc.com/india/india-b2b-e-invoicing-threshold-drops-to-%E2%82%B95-january-2023-faqs-update/)
34. [Fiscal Requirements — India GST E-Invoicing B2C Expansion](https://www.fiscal-requirements.com/news/3818)
35. [56th GST Council Meeting Press Release (CBIC)](https://gstcouncil.gov.in/sites/default/files/2025-09/press_release_press_information_bureau_0.pdf)
36. [Kotak MF — GST 2.0 New Slabs Effective 22 Sep 2025](https://www.kotakmf.com/Information/blogs/gst-2-point-0_)
37. [TaxGuru — Credit Notes GST Deadline](https://taxguru.in/goods-and-service-tax/credit-notes-gst-hidden-deadline-business.html)
38. [CBIC FAQ — Composition Levy](https://cbic-gst.gov.in/pdf/faq-manual/faq-composition-levy-revised.pdf)
39. [ClearTax — HSN Cosmetics GST Rates](https://cleartax.in/s/beauty-make-up-preparations-skin-gst-rates-hsn-code-3304)
40. [ClearTax — GST on Mobile Phones 2026](https://cleartax.in/s/gst-mobile-phones)
