# Fiken API v2 — Full Endpoint Reference

Complete curl examples for all Fiken API endpoints not covered in the core SKILL.md. All requests use the standard auth pattern:

```bash
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/{endpoint}" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json"
```

Rate limits: max 1 concurrent request, max 4/sec. Add `sleep 0.3` between batched calls.

---

## Products

```bash
# List products
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/products?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {productId, name, unitPrice}'

# Create product (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/products" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Consulting hour", "unitPrice": 150000, "vatType": "HIGH", "incomeAccount": "3000"}'
```

## Sales

```bash
# List sales (with payment status)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/sales?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  | jq '.[] | {saleId, date, settled, settledDate, outstandingBalance}'

# Create sale draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/sales/drafts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "date": "2026-01-15",
    "lines": [{"netPrice": 100000, "vatType": "HIGH", "account": "3000", "description": "Sale"}]
  }'
```

## Purchases

```bash
# List purchases (with payment status)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/purchases?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  | jq '.[] | {purchaseId, date, paid, paymentDate}'

# Create purchase draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/purchases/drafts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "date": "2026-01-15",
    "lines": [{"netPrice": 50000, "vatType": "HIGH", "account": "6300", "description": "Office supplies"}]
  }'
```

## Payments

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

## Credit Notes

Full credit note = reverse entire invoice. Partial = reverse specific lines/amounts.

```bash
# List credit notes
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/creditNotes?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  | jq '.[] | {creditNoteId, issueDate, net, settled: .sale.settled, outstanding: .sale.outstandingBalance}'

# Create full credit note (⚠️ REQUIRES CONFIRMATION)
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

## Inbox (Document Upload)

```bash
# List inbox documents
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/inbox?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {inboxDocumentId, name, createdDate, status}'

# Upload document to inbox (⚠️ REQUIRES CONFIRMATION) — supports PDF, PNG, JPG
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/inbox" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -F "file=@/path/to/receipt.pdf" \
  -F "name=Receipt 2026-01" \
  -F "description=Office supplies receipt"
```

## Transactions

```bash
# List transactions
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/transactions?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {transactionId, date, description}'

# Get specific transaction
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/transactions/{transactionId}" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '{transactionId, date, description, entries}'

# Delete transaction (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/transactions/{transactionId}/delete" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN"
```

## Projects

```bash
# List projects
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/projects?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {projectId, name, startDate, endDate}'

# Create project (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/projects" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Client Website Redesign", "startDate": "2026-01-01", "endDate": "2026-06-30"}'
```

## Time Tracking

```bash
# List time entries
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/timeEntries?page=0&pageSize=100" \
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

## Offers (Tilbud)

```bash
# List offers
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/offers?page=0&pageSize=100" \
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

## Order Confirmations

```bash
# List order confirmations
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/orderConfirmations?page=0&pageSize=100" \
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

# Convert to invoice draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/orderConfirmations/{confirmationId}/createInvoiceDraft" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN"
```

## Attachments

Same pattern for all resources — swap the resource path:

```bash
# List attachments on an invoice
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/{invoiceId}/attachments" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {identifier, downloadUrl}'

# Upload attachment (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/{invoiceId}/attachments" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -F "file=@/path/to/document.pdf"

# Supported resource paths for attachments:
#   invoices/{invoiceId}/attachments
#   sales/{saleId}/attachments
#   purchases/{purchaseId}/attachments
#   journalEntries/{journalEntryId}/attachments
#   contacts/{contactId}/attachments
#   invoices/drafts/{draftId}/attachments
#   sales/drafts/{draftId}/attachments
#   purchases/drafts/{draftId}/attachments
```

## Draft Management & Other Operations

```bash
# Finalize invoice draft (⚠️ REQUIRES CONFIRMATION)
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/drafts/{draftId}/createInvoice" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN"

# Send invoice via email/EHF (⚠️ REQUIRES CONFIRMATION)
# Dispatch types: email, sms, ehf, letter, vipps, efaktura
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/send" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"invoiceId": 12345, "method": "email", "emailAddress": "customer@example.no"}'

# Delete a draft (⚠️ REQUIRES CONFIRMATION)
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
