# Fiken Workflows Reference

Detailed workflow patterns for complex accounting operations via the Fiken API v2.

## 1. Konsernbidrag (Group Contributions)

Konsernbidrag is a tax-motivated transfer between Norwegian AS companies in a group (>90% ownership). The parent gives profit to the subsidiary (or vice versa) to equalize taxable results across the group.

### Account Mapping

| Account | Name                   | Role                                                                     |
| ------- | ---------------------- | ------------------------------------------------------------------------ |
| 8935    | Avsatt konsernbidrag   | Giver — expense (reduces giver's profit)                                 |
| 8931    | Mottatt konsernbidrag  | Receiver — income (increases receiver's profit)                          |
| 2500    | Betalbar skatt         | Tax payable (adjusted in both companies)                                 |
| 1560    | Konsernfordring        | Intercompany receivable (receiver books this)                            |
| 2920    | Gjeld konsern          | Intercompany payable (giver books this)                                  |
| 1301    | Aksjer i datterselskap | Investment in subsidiary (giver — DO NOT use for konsernbidrag postings) |
| 8301    | Betalbar skatt         | Tax expense line (giver — zero if giving away all profit)                |

### Tax Effect

The konsernbidrag is always posted at **FULL GROSS** on both 8935 and 8931 — the same amount on both sides. The 22% tax effect is a separate consequence: the giver's tax payable (2500) decreases by gross × 22%, the receiver's increases by the same. This is handled by the overall tax calculation, NOT by splitting the konsernbidrag entry 78/22. Net tax effect for the group = zero.

### Journal Entry Patterns

**Giver (parent company posting konsernbidrag expense):**

```bash
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/generalJournalEntries" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Konsernbidrag 2025 — avgitt til [datterselskap]",
    "date": "2025-12-31",
    "lines": [
      {"debitAccount": "8935", "debitVatCode": "NONE", "amount": 30273000},
      {"creditAccount": "2920", "creditVatCode": "NONE", "amount": 30273000}
    ]
  }'
```

**Receiver (subsidiary posting konsernbidrag income):**

```bash
curl -s -X POST "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/generalJournalEntries" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Konsernbidrag 2025 — mottatt fra [morselskap]",
    "date": "2025-12-31",
    "lines": [
      {"debitAccount": "1560", "debitVatCode": "NONE", "amount": 30273000},
      {"creditAccount": "8931", "creditVatCode": "NONE", "amount": 30273000}
    ]
  }'
```

Note: The intercompany receivable (1560) and payable (2920) are settled when the actual bank transfer happens — that's a separate payment posting, not part of the konsernbidrag journal entry.

### Verification Checklist

```bash
# Verify konsernbidrag balances for both companies
# ⚠️ Use individual account lookups — page=0&pageSize=100 will MISS 89xx accounts
for ACCT in 8935 8931 2500 1560 2920 1301; do
  curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accountBalances/$ACCT?date=2025-12-31" \
    -H "Authorization: Bearer $FIKEN_API_TOKEN" \
    | jq '{code, name, balance}'
  sleep 0.3
done
```

Run for EACH company, then verify:

- ✅ Giver's 8935 (debit) = Receiver's 8931 (credit) in absolute amount — these MUST be equal. If they differ, the posting is WRONG. Do NOT rationalize by adding 8931 + 2500 to "match" 8935.
- ✅ Tax effect = gross × 22% reflected in 2500 (this is separate from the konsernbidrag amounts)
- ✅ Intercompany 1560 (receiver) = 2920 (giver) — nets to zero after settlement
- ✅ 1301 unchanged from original investment amount
- ⚠️ Check if giver has negative EK after konsernbidrag — legal but board must be aware

### Common Mistakes

- Do NOT split the konsernbidrag amount 78/22 between 8931 and 2500 — the FULL GROSS goes on 8931 (receiver) and 8935 (giver). The tax effect on 2500 is a separate calculation, not a split of the konsernbidrag posting
- Do NOT post konsernbidrag to account 2030 (Annen innskutt EK) — that's for equity contributions, not konsernbidrag
- Do NOT post to 8320 (Endring utsatt skatt) — konsernbidrag affects betalbar skatt (2500), not utsatt skatt
- Do NOT debit 1301 (Aksjer i datterselskap) for the konsernbidrag amount — 1301 stays at original investment value
- Do NOT use 80/20 split — the correct Norwegian tax rate is 22%, giving a 78/22 split
- Always date the journal entry 31.12 of the fiscal year, even if the bank transfer happens in January
- Do NOT query accountBalances with `page=0&pageSize=100` for verification — use the single-account endpoint

### Timing

Konsernbidrag must be decided by the board BEFORE annual accounts are filed (typically before 30 June for calendar-year companies). The journal entry date is always 31.12 of the fiscal year.

---

## 2. Skattemelding Prep (Tax Filing Readiness)

Step-by-step checklist to verify accounts are ready for skattemelding (tax return).

1. **Verify konsernbidrag** (if multi-company) — run the verification checklist from section 1 above
2. **Check all account balances** — use non-zero balance query:
   ```bash
   curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accountBalances?date=2025-12-31&page=0&pageSize=100" \
     -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '[.[] | select(.balance != 0) | {code, name, balance}]'
   ```
   ⚠️ Check `Fiken-Api-Page-Count` header — if >1, use `fromAccount`/`toAccount` ranges to get all data.
3. **Reconcile bank** — compare Fiken bank balance vs actual bank statement:
   ```bash
   curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/bankAccounts?page=0&pageSize=100" \
     -H "Authorization: Bearer $FIKEN_API_TOKEN" | jq '.[] | {name, accountCode, balance: .balance}'
   ```
   Ask user to confirm each balance matches their bank.
4. **Check MVA status** — see MVA Status Check section below
5. **Verify no stale intercompany balances** — 1560/2920 should be zero if all payments settled
6. **Check for negative EK** — if total equity < 0, flag for board awareness
7. **Confirm all invoices accounted for** — check for unpaid invoices that should be written off or followed up

After running through this checklist, tell the user: "Regnskapet er klart for skattemeldingen" or flag any issues found.

---

## 3. Multi-company Reconciliation

```bash
# Run for EACH company, compare results:
FIKEN_COMPANY="company-a-slug"
for ACCT in 1560 2920 1301 2500; do
  curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accountBalances/$ACCT?date=2025-12-31" \
    -H "Authorization: Bearer $FIKEN_API_TOKEN" \
    | jq '{code, name, balance}'
  sleep 0.3
done
```

Verification rules:

- Company A's 1560 (receivable) should equal Company B's 2920 (payable) in absolute value
- 1301 should equal original share capital investment (e.g. 30,000 NOK = 3000000 øre)
- If balances don't match, check for unposted payments or timing differences

Note: Always check BOTH driftskonto AND folio accounts — companies may have multiple bank accounts.

---

## 4. MVA Status Check

```bash
# Check MVA-related account balances (2700-2799)
curl -s "https://api.fiken.no/api/v2/companies/$FIKEN_COMPANY/accountBalances?date=$(date +%Y-%m-%d)&fromAccount=2700&toAccount=2799&pageSize=100" \
  -H "Authorization: Bearer $FIKEN_API_TOKEN" \
  | jq '[.[] | select(.balance != 0) | {code, name, balance}]'
```

Key accounts:

- 2700 = Utgående MVA (output VAT — what you owe)
- 2710 = Inngående MVA (input VAT — what you can deduct)
- 2740 = Oppgjørskonto MVA (settlement account)

Netto MVA å betale = 2700 + 2710 (2710 is typically negative/debit). Report this to the user.

Note: MVA-melding (VAT return) must be filed via Fiken UI or Altinn — the API cannot submit VAT returns. But you CAN verify the numbers are correct before the user files.
