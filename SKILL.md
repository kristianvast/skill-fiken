---
name: fiken
description: Full Fiken accounting API v2 — invoices, credit notes, contacts, purchases, sales, inbox, journal entries, projects, time tracking, offers, and order confirmations. Use when asked about accounting, invoices, expenses, receipts, who owes money, time logging, project costs, or financial reports. NOT for tax filing, payroll, bank transfers, or Fiken UI navigation.
homepage: https://api.fiken.no/api/v2/docs/
metadata:
  {
    "openclaw":
      {
        "emoji": "💰",
        "requires": { "bins": ["curl", "jq"], "env": ["FIKEN_API_TOKEN"] },
        "primaryEnv": "FIKEN_API_TOKEN",
      },
  }
---

## When to Use

✅ USE this skill when:

- "check my balance" / "show my accounts"
- "create an invoice" / "send an invoice"
- "who owes me" / "outstanding invoices"
- "show my expenses" / "list purchases"
- "add a contact" / "find customer"
- "what did I sell" / "list sales"
- "upload a receipt" / "import a bill" / "add to inbox"
- "log my hours" / "track time" / "time entry"
- "create a credit note" / "refund an invoice"
- "create an offer" / "send a quote"
- "journal entry" / "post a transaction"
- "project costs" / "project tracking"

## When NOT to Use

❌ DON'T use this skill when:

- Tax filing (MVA return) → use Fiken UI
- Payroll → use Fiken's payroll module
- Bank transfers → use your bank directly
- Help navigating Fiken UI → use fiken.no/hjelp

## Setup

1. Activate API module: Fiken → Innstillinger → Moduler → Fiken API (99 NOK/month) — activate for **each** company
2. Generate Personal API Token: Rediger konto → API → Personlige API-nøkler → Ny nøkkel (one token covers all companies you have access to)
3. Set env var: `export FIKEN_API_TOKEN="your-token-here"`
4. Discover company slugs:
   ```bash
   curl -s https://api.fiken.no/api/v2/companies \
     -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {name, slug}'
   ```

**Multi-company note**: One API token accesses all companies you have permissions for. When the user's request doesn't make the company obvious, ask: "Which company — [name A] or [name B]?" Use the company slug directly in the URL path (`$FIKEN_COMPANY` below). There is no need to set a default company env var, but you may use `$FIKEN_COMPANY` as a convenience shorthand after confirming which company the user means.

## API Basics

- Base URL: `https://api.fiken.no/api/v2`
- Standard headers template:
  ```bash
  -H "Authorization: Bearer $FIKEN_API_TOKEN"
  -H "Content-Type: application/json"
  ```
- **RATE LIMITS (⚠️ IMPORTANT)**: Max 1 concurrent request. Max 4 requests/sec. Violation = 429 error or account ban. NEVER parallelize calls. Always wait for each response before sending next.

## Norwegian Accounting Context

- Amounts: Always integer øre. 100000 = 1000 NOK. Never use decimal kroner.
- Dates: `yyyy-MM-dd` format only
- MVA/VAT codes:
  - `HIGH` = 25% (standard goods/services)
  - `MEDIUM` = 15% (food/drink)
  - `LOW` = 12% (passenger transport, cinema)
  - `NONE` = 0% (exempt)
- Common Norsk Standard Kontoplan codes:
  - `1920` = Bank (bank account)
  - `1500` = Kundefordringer (accounts receivable)
  - `2400` = Leverandørgjeld (accounts payable)
  - `3000` = Salgsinntekt (sales revenue)
  - `4000` = Varekostnad (cost of goods)
  - `5000` = Lønn (payroll)
  - `6300` = Kontorrekvisita (office supplies)
  - `6900` = Telefon/internett (phone/internet)
  - `7700` = Avskrivninger (depreciation)

## Required Confirmations (always)

Require explicit approval for:

- Creating or sending invoices
- Creating or modifying contacts
- Creating purchases or sales entries
- Posting journal entries
- Creating or updating products
- Any DELETE operation
- Creating or sending credit notes
- Uploading documents to inbox
- Creating or modifying projects
- Creating, modifying, or deleting time entries
- Creating or sending offers
- Creating or sending order confirmations
- Posting or deleting journal entries/transactions
- Setting invoice/credit note/offer counters

**⚠️ WARNING**: There is no Fiken sandbox. All API calls affect your live accounting data. If unsure, ask.
To proceed with a write operation, show the full curl command with the complete payload to the user and wait for explicit "yes" / approval.

## Core Commands

### 1. Companies

```bash
# List companies (no companySlug needed)
curl -s https://api.fiken.no/api/v2/companies \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[].slug'

# Get company details
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '{name, organizationNumber}'
```

### 2. Contacts

```bash
# List contacts
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/contacts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {contactId, name, email}'

# Create contact (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/contacts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Ola Nordmann AS", "email": "ola@example.no", "customer": true}'
```

### 3. Invoices

Note: Draft → Invoice → Sent → Paid

```bash
# List invoices
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {invoiceId, issueDate, dueDate, net, settled}'

# Create invoice draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/drafts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "issueDate": "2026-01-15",
    "daysUntilDueDate": 14,
    "customerId": 12345,
    "lines": [{"netPrice": 100000, "vatType": "HIGH", "description": "Consulting"}]
  }'

# Get invoice details
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/{invoiceId}" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '{invoiceId, issueDate, dueDate, net, gross, settled}'
```

### 4. Products

```bash
# List products
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/products" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {productId, name, unitPrice}'

# Create product (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/products" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Consulting hour", "unitPrice": 150000, "vatType": "HIGH", "incomeAccount": "3000"}'
```

### 5. Sales

```bash
# List sales
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/sales" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {saleId, date, totalPaid}'

# Create sale draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/sales/drafts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "date": "2026-01-15",
    "lines": [{"netPrice": 100000, "vatType": "HIGH", "account": "3000", "description": "Sale"}]
  }'
```

### 6. Purchases

```bash
# List purchases
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/purchases" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {purchaseId, date, totalPaid}'

# Create purchase draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/purchases/drafts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "date": "2026-01-15",
    "lines": [{"netPrice": 50000, "vatType": "HIGH", "account": "6300", "description": "Office supplies"}]
  }'
```

### 7. Bank Accounts

```bash
# List bank accounts
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/bankAccounts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {bankAccountId, name, accountCode}'
```

### 8. Accounts & Balances

```bash
# Chart of accounts
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accounts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {code, name}'

# Account balances at a specific date
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accountBalances?date=2026-01-31" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {code, balance}'
```

### 9. Inbox (Document Upload)

```bash
# List inbox documents
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/inbox" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {inboxDocumentId, name, createdDate}'

# Upload document to inbox (⚠️ REQUIRES CONFIRMATION)
# Use multipart form — supports PDF, PNG, JPG
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/inbox" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -F "file=@/path/to/receipt.pdf" \
  -F "name=Receipt 2026-01" \
  -F "description=Office supplies receipt"
```

### 10. Payments

```bash
# List payments for a sale
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/sales/{saleId}/payments" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {paymentId, date, amount}'

# Record payment on a sale (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/sales/{saleId}/payments" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"date": "2026-01-20", "amount": 125000, "account": "1920"}'

# Record payment on a purchase (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/purchases/{purchaseId}/payments" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"date": "2026-01-20", "amount": 50000, "account": "1920"}'
```

### 11. Credit Notes

Note: Full credit note = reverse entire invoice. Partial = reverse specific lines/amounts.

```bash
# List credit notes
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/creditNotes" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {creditNoteId, issueDate, net}'

# Create full credit note for an invoice (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/creditNotes/full" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"invoiceId": 12345, "issueDate": "2026-01-20", "creditNoteText": "Full refund"}'

# Create partial credit note (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/creditNotes/partial" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"invoiceId": 12345, "issueDate": "2026-01-20", "lines": [{"netPrice": 50000, "vatType": "HIGH", "description": "Partial refund"}]}'
```

### 12. Journal Entries

```bash
# List journal entries (posteringer)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/journalEntries" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {journalEntryId, date, description}'

# Create general journal entry / fri postering (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/generalJournalEntries" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Year-end adjustment",
    "date": "2026-12-31",
    "lines": [
      {"debitAccount": "6300", "debitVatCode": "NONE", "amount": 100000},
      {"creditAccount": "2400", "creditVatCode": "NONE", "amount": 100000}
    ]
  }'
```

### 13. Transactions

```bash
# List all transactions
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/transactions" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {transactionId, date, description}'

# Get specific transaction
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/transactions/{transactionId}" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '{transactionId, date, description, entries}'

# Delete transaction by creating reverse entry (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/transactions/{transactionId}/delete" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN"
```

### 14. Projects

```bash
# List projects
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/projects" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {projectId, name, startDate, endDate}'

# Create project (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/projects" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Client Website Redesign", "startDate": "2026-01-01", "endDate": "2026-06-30"}'
```

### 15. Time Tracking

```bash
# List time entries
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/timeEntries" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {timeEntryId, date, hours, activityId}'

# Create time entry (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/timeEntries" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"date": "2026-01-15", "hours": 7.5, "activityId": 1, "projectId": 1, "description": "Frontend development"}'

# List activities
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/activities" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {activityId, name, rate}'

# Create invoice draft from time entries (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/timeEntries/createInvoiceDraft" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"timeEntryIds": [1, 2, 3], "customerId": 12345, "daysUntilDueDate": 14}'
```

### 16. Offers (Tilbud)

```bash
# List offers
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/offers" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {offerId, date, net}'

# Create offer draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/offers/drafts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 12345,
    "date": "2026-01-15",
    "daysUntilDueDate": 30,
    "lines": [{"netPrice": 200000, "vatType": "HIGH", "description": "Website development"}]
  }'

# Finalize draft into offer (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/offers/drafts/{draftId}/createOffer" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN"
```

### 17. Order Confirmations

```bash
# List order confirmations
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/orderConfirmations" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {confirmationId, date, net}'

# Create order confirmation draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/orderConfirmations/drafts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 12345,
    "date": "2026-01-15",
    "lines": [{"netPrice": 200000, "vatType": "HIGH", "description": "Confirmed order"}]
  }'

# Convert order confirmation to invoice draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/orderConfirmations/{confirmationId}/createInvoiceDraft" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN"
```

### 18. Attachments

Note: Attachments work on invoices, sales, purchases, journal entries, contacts, and all draft types. The pattern is the same everywhere.

```bash
# List attachments on an invoice
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/{invoiceId}/attachments" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {identifier, downloadUrl}'

# Upload attachment to an invoice (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/{invoiceId}/attachments" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -F "file=@/path/to/document.pdf"

# Same pattern for other resources — replace invoices/{id} with:
#   sales/{saleId}/attachments
#   purchases/{purchaseId}/attachments
#   journalEntries/{journalEntryId}/attachments
#   contacts/{contactId}/attachments
#   invoices/drafts/{draftId}/attachments
#   sales/drafts/{draftId}/attachments
#   purchases/drafts/{draftId}/attachments
```

### 19. Draft Management & Other Operations

```bash
# Finalize invoice draft into invoice (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/drafts/{draftId}/createInvoice" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN"

# Send invoice via email or EHF (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/send" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"invoiceId": 12345, "method": "email", "emailAddress": "customer@example.no"}'

# Delete a draft (⚠️ REQUIRES CONFIRMATION) — works for invoice, sale, purchase, offer, order confirmation drafts
curl -s -X DELETE "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/drafts/{draftId}" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN"

# Product sales report
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/products/salesReport" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {productId, name, totalSold}'

# Get/set invoice counter
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/counter" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.nextInvoiceNumber'

# Customer groups
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/groups" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {groupId, name}'

# Current user info
curl -s "https://api.fiken.no/api/v2/user" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '{name, email}'
```

## Notes

- **Pagination**: Add `?page=0&pageSize=100` to list endpoints. Max 100 per page. Response headers: `Fiken-Api-Page-Count` (total pages), `Fiken-Api-Result-Count` (total items).
- **Token security**: Store in env var only. Never paste `$FIKEN_API_TOKEN` as a literal string in commands or logs.
- **Full API reference**: https://api.fiken.no/api/v2/docs/ — the interactive Swagger UI for exploring all endpoints and schemas.
- **Invoice payment status**: The `settled` field on the invoice object itself may be `null` even when the invoice IS paid. Always check `invoice.sale.settled` (boolean), `invoice.sale.settledDate`, and `invoice.sale.outstandingBalance` for the actual payment status. Use this jq pattern for invoice overview:
  ```bash
  jq '[.[] | {invoiceNumber, issueDate, dueDate, gross, paid: .sale.settled, paidDate: .sale.settledDate, outstanding: .sale.outstandingBalance, customer: .customer.name}]'
  ```
