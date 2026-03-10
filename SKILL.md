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
- "konsernbidrag" / "skattemelding" / "MVA check"

## When NOT to Use

❌ DON'T use this skill when:

- Tax filing (MVA return) → use Fiken UI or Altinn
- Payroll → use Fiken's payroll module
- Bank transfers → use your bank directly
- Help navigating Fiken UI → use fiken.no/hjelp

## Reference Files

This skill includes reference files in the `references/` subdirectory. Read them when you need full endpoint documentation, detailed workflows, or accounting context.

- **workflows.md**: Konsernbidrag (group contributions) — full journal entry patterns, verification checklist, and common mistakes. Skattemelding prep. Multi-company reconciliation. MVA status check.
- **api-endpoints.md**: Complete endpoint catalog — products, sales, purchases, credit notes, payments, inbox, projects, time tracking, offers, order confirmations, attachments, drafts, transactions.
- **accounting-context.md**: Norsk Standard Kontoplan account codes, full VAT/MVA type table, Norwegian accounting terms.

Read the relevant reference file BEFORE attempting complex operations (konsernbidrag, multi-company reconciliation, unfamiliar endpoints).

## Setup

1. Activate API module: Fiken → Innstillinger → Moduler → Fiken API (99 NOK/month) — activate for **each** company
2. Generate Personal API Token: Rediger konto → API → Personlige API-nøkler → Ny nøkkel (one token covers all companies you have access to)
3. Set env var: `export FIKEN_API_TOKEN="your-token-here"`
4. Discover company slugs:
   ```bash
   curl -s https://api.fiken.no/api/v2/companies \
     -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {name, slug}'
   ```

**Multi-company note**: One API token accesses all companies you have permissions for. When the user's request doesn't make the company obvious, ask: "Which company — [name A] or [name B]?"

## API Rules

**Base URL**: `https://api.fiken.no/api/v2`

**Standard curl template** (all requests follow this pattern):

```bash
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/{endpoint}" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json"
```

**Rate limits** — violation = 429 error or account ban:

- Max **1 concurrent request** — NEVER parallelize calls
- Max **4 requests/second** — add `sleep 0.3` between batched calls

**On 429 error**: Wait 2 seconds and retry. If still 429, double the wait (up to 30s max). Never retry more than 3 times.

**Amounts**: Always integer øre. `100000` = 1,000 NOK. Never use decimal kroner.

**Dates**: `yyyy-MM-dd` format only. Date range filters: `dateGe` (≥), `dateGt` (>), `dateLe` (≤), `dateLt` (<).

**Pagination**: Default page size is **25**. Always add `?page=0&pageSize=100`. Max 100 per page. Response headers: `Fiken-Api-Page-Count` (total pages), `Fiken-Api-Result-Count` (total items).

- **⚠️ accountBalances is special**: Returns ALL accounts including zero-balance. Companies with >100 accounts will have data BEYOND page 0. Use `fromAccount`/`toAccount` filters or the single-account endpoint — never rely on `page=0&pageSize=100` alone for balance checks.
- For contacts, invoices, purchases, sales: page 0 is usually sufficient unless you need a complete export.

**POST/create responses**: Return `201 Created` with a `Location` header but **no JSON body**. To get the created resource ID, parse the `Location` header:

```bash
# Extract resource ID from Location header
LOCATION=$(curl -si -X POST "..." -d '...' | grep -i '^location:' | tr -d '\r' | awk '{print $2}')
RESOURCE_ID=$(basename "$LOCATION")
```

**Draft UUID idempotency**: When creating drafts (invoices, sales, purchases), pass a `uuid` field (any UUID v4) to prevent duplicates on retry.

**Error responses**: `400` = bad request (missing field). `401` = bad/expired token. `403` = no access to company. `404` = resource not found. `429` = rate limited.

## Required Confirmations

⚠️ **All write operations require explicit user approval.** There is no Fiken sandbox — API calls affect live accounting data.

Before any POST/PUT/DELETE: show the full curl command with payload and wait for explicit "yes" / approval. This applies to invoices, contacts, journal entries, purchases, sales, credit notes, products, drafts, offers, payments, time entries, and any deletion.

## Efficiency

The embedded agent has a **limited runtime and context window**. Fiken API calls accumulate quickly.

1. **Plan before querying**: State what data you need and why BEFORE making calls
2. **Filter with jq**: Only extract fields you need. Use `select()` to skip zero-balance accounts
3. **Never re-fetch**: If data is already in context, reference it — don't call the API again
4. **Summarize after each batch**: Write a text summary of findings, then reference it later instead of re-reading raw JSON
5. **Know when to stop**: Balance check → 2–3 calls max. Full audit → confirm scope with user first
6. **Batch with rate limit**: Combine sequential calls with `sleep 0.3` between them

## Core Operations

### Companies

```bash
# List company slugs
curl -s https://api.fiken.no/api/v2/companies \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[].slug'

# Get company details
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '{name, organizationNumber}'
```

### Account Balances

```bash
# ⚠️ PREFERRED: Filter by account code range (avoids pagination issues)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accountBalances?date=2025-12-31&fromAccount=8000&toAccount=9999&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '[.[] | select(.balance != 0) | {code, name, balance}]'

# Single account balance (most reliable for specific accounts)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accountBalances/8931?date=2025-12-31" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '{code, name, balance}'

# All non-zero balances (⚠️ may need pagination for companies with >100 accounts)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accountBalances?date=2025-12-31&page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '[.[] | select(.balance != 0) | {code, name, balance}]'

# Chart of accounts
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accounts?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {code, name}'
```

NOTE: The nested `account` object in balance responses may have null code/name — always use top-level fields.

### Journal Entries

```bash
# List journal entries
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/journalEntries?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {journalEntryId, date, description}'

# Create general journal entry (fri postering) — ⚠️ REQUIRES CONFIRMATION
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/generalJournalEntries" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Year-end adjustment",
    "date": "2025-12-31",
    "lines": [
      {"debitAccount": "6300", "debitVatCode": "NONE", "amount": 100000},
      {"creditAccount": "2400", "creditVatCode": "NONE", "amount": 100000}
    ]
  }'
```

### Invoices

**⚠️ CRITICAL — Invoice payment status**: The `settled` field on the invoice object is a **date** field that may be `null` even when the invoice IS fully paid. **Never** use `invoice.settled` alone. Instead check:

- `invoice.sale.settled` — **boolean**, true if paid
- `invoice.sale.outstandingBalance` — **integer (øre)**, `0` = fully paid (most reliable)

```bash
# List invoices with correct payment status
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  | jq '[.[] | {invoiceNumber, issueDate, dueDate, gross, paid: .sale.settled, outstanding: .sale.outstandingBalance, customer: .customer.name}]'

# Unpaid invoices only
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices?settled=false&page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  | jq '[.[] | {invoiceNumber, dueDate, gross, outstanding: .sale.outstandingBalance, customer: .customer.name}]'

# Overdue unpaid invoices (due before today)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices?settled=false&dueDateLt=$(date +%Y-%m-%d)&page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  | jq '[.[] | {invoiceNumber, dueDate, gross, outstanding: .sale.outstandingBalance, customer: .customer.name}]'

# Create invoice draft — ⚠️ REQUIRES CONFIRMATION
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/invoices/drafts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "uuid": "'$(uuidgen)'",
    "issueDate": "2026-01-15",
    "daysUntilDueDate": 14,
    "customerId": 12345,
    "bankAccountCode": "1920",
    "cash": false,
    "lines": [{"netPrice": 100000, "vatType": "HIGH", "description": "Consulting"}]
  }'
```

### Contacts

```bash
# List contacts
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/contacts?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {contactId, name, email}'

# Search by organization number (for deduplication before creating)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/contacts?organizationNumber=123456789" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {contactId, name}'

# Create contact — ⚠️ REQUIRES CONFIRMATION
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/contacts" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Ola Nordmann AS", "email": "ola@example.no", "customer": true}'
```

### Bank Accounts

```bash
# List bank accounts
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/bankAccounts?page=0&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {bankAccountId, name, accountCode}'
```

## Common Workflows

### Konsernbidrag (Group Contributions)

Tax-motivated transfer between Norwegian AS companies in a group (>90% ownership). Key accounts:

| Account | Name                  | Role                               |
| ------- | --------------------- | ---------------------------------- |
| 8935    | Avsatt konsernbidrag  | Giver — expense                    |
| 8931    | Mottatt konsernbidrag | Receiver — income                  |
| 1560    | Konsernfordring       | Intercompany receivable (receiver) |
| 2920    | Gjeld konsern         | Intercompany payable (giver)       |
| 2500    | Betalbar skatt        | Tax payable (adjusted both sides)  |

**⚠️ CRITICAL RULES**:

- 8935 and 8931 are ALWAYS posted at **FULL GROSS** — the same amount on both sides
- Do NOT split konsernbidrag 78/22 between 8931 and 2500 — the 22% tax effect is a separate consequence, not a split of the posting
- 8935 (giver) MUST equal 8931 (receiver) in absolute amount — if they differ, the posting is WRONG
- Always date 31.12 of the fiscal year
- Do NOT post to 2030 (equity contributions) or 1301 (stays at investment value)

**For full journal entry patterns, verification checklist, and all common mistakes → read `references/workflows.md`**

### Skattemelding Prep (Tax Filing Readiness)

1. Verify konsernbidrag (if multi-company) — run verification from workflows.md
2. Check all account balances (non-zero)
3. Reconcile bank balances vs actual statements
4. Check MVA status (accounts 2700-2799)
5. Verify intercompany balances (1560/2920 should net to zero if settled)
6. Check for negative equity — flag for board awareness
7. Confirm all invoices accounted for

**For detailed commands and checklists → read `references/workflows.md`**

### MVA Status Check

```bash
# Quick MVA check (accounts 2700-2799)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accountBalances?date=$(date +%Y-%m-%d)&fromAccount=2700&toAccount=2799&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  | jq '[.[] | select(.balance != 0) | {code, name, balance}]'
```

Netto MVA = 2700 (output VAT) + 2710 (input VAT, typically negative). MVA-melding must be filed via Fiken UI or Altinn — the API cannot submit returns.

## Known Limitations

- Cannot change the contact/customer on a posted sale or invoice
- Cannot edit lines on a finalized invoice — issue a credit note instead
- Cannot merge or split existing transactions
- Cannot submit MVA/VAT returns via API — use Fiken UI or Altinn
- No webhooks or event notifications — polling required
- No batch/bulk operations — all requests are individual
- No year-end closing entries, depreciation, or accruals via API
- Bank sync may lag by months — always verify against actual bank statements

If the user needs any of these, tell them directly: "This requires the Fiken UI — the API doesn't support [X]."

## Notes

- **Token security**: Store in env var only. Never paste `$FIKEN_API_TOKEN` as a literal string.
- **Full API reference**: https://api.fiken.no/api/v2/docs/ — interactive Swagger UI for all endpoints and schemas.
- **Common MVA/VAT codes**: `HIGH` = 25%, `MEDIUM` = 15%, `LOW` = 12%, `NONE` = 0%. For the full VAT type table → read `references/accounting-context.md`.
- **Common account codes**: 1920=Bank, 3000=Salgsinntekt, 6300=Kontorrekvisita. For the full Kontoplan → read `references/accounting-context.md`.
- **For endpoints not listed above** (products, sales, purchases, credit notes, payments, inbox, offers, time tracking, etc.) → read `references/api-endpoints.md`.
