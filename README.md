# Shopify Accounting Platform — Integration Skills

Technical integration guides for building a Shopify accounting platform focused on India and Singapore markets.

## Skills

| Skill | File | Size | What It Covers |
|-------|------|------|----------------|
| **Zoho Inventory** | `zoho-inventory-integration.md` | ~104KB | OAuth 2.0, Items/Orders/Invoices CRUD, India GST fields, Zoho Books sync, multi-warehouse, rate limiting |
| **Zoho Books** | `zoho-books-integration.md` | ~100KB | Full accounting pipeline, GST tax setup, invoice generation, bank reconciliation, Singapore GST, e-invoicing, TDS/TCS |
| **Shopify APIs** | `shopify-api-integration.md` | ~110KB | Public app OAuth, order extraction, payout reconciliation, webhooks, Billing API, GraphQL bulk ops, GDPR compliance |
| **Indian GST Engine** | `indian-gst-compliance.md` | ~133KB | Place of Supply engine, tax rate calculation, GSTIN validation, GSTR-1/3B data prep, e-invoicing (IRP), HSN codes, TCS/TDS |

## Usage with Claude Code / Paperclip

Reference any skill file as context when prompting:

```
@zoho-inventory-integration.md Implement the OAuth 2.0 auth flow and item sync module
```

```
@indian-gst-compliance.md Build the Place of Supply engine and CGST/SGST/IGST determination logic
```

```
@shopify-api-integration.md Set up the webhook handler with HMAC verification for order events
```

## Research Files

Raw research data is in the `research/` directory — use these for additional API reference detail.

## Tech Stack

All code examples use **TypeScript / Node.js**. Designed for a Remix or Next.js app embedded in Shopify Admin.

## Markets

- **India**: Full GST compliance (CGST/SGST/IGST), GSTR-1/3B, e-invoicing, HSN codes, TCS
- **Singapore**: 9% GST, InvoiceNow/PEPPOL, IRAS F5 returns

---

*Generated March 2026*
