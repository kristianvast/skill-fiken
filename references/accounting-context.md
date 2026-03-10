# Norwegian Accounting Context

Reference material for Norwegian accounting standards used with the Fiken API.

## Norsk Standard Kontoplan (Account Codes)

Common account codes used in Norwegian bookkeeping. Fiken uses 4-digit codes. Reskontro (subledger) accounts use format `1500:10001`.

### Assets (1xxx)

| Code | Name                   | Description                      |
| ---- | ---------------------- | -------------------------------- |
| 1301 | Aksjer i datterselskap | Investment in subsidiary         |
| 1500 | Kundefordringer        | Accounts receivable              |
| 1560 | Konsernfordring        | Intercompany receivable          |
| 1920 | Bank (driftskonto)     | Main operating bank account      |
| 1921 | Bank (folio)           | Secondary bank / savings account |

### Liabilities (2xxx)

| Code | Name              | Description                                  |
| ---- | ----------------- | -------------------------------------------- |
| 2030 | Annen innskutt EK | Other paid-in equity (NOT for konsernbidrag) |
| 2400 | Leverandørgjeld   | Accounts payable                             |
| 2500 | Betalbar skatt    | Tax payable                                  |
| 2700 | Utgående MVA      | Output VAT (what you owe)                    |
| 2710 | Inngående MVA     | Input VAT (what you can deduct)              |
| 2740 | Oppgjørskonto MVA | VAT settlement account                       |
| 2920 | Gjeld konsern     | Intercompany payable                         |

### Revenue (3xxx)

| Code | Name                | Description             |
| ---- | ------------------- | ----------------------- |
| 3000 | Salgsinntekt        | Sales revenue           |
| 3440 | Offentlige tilskudd | Public grants/subsidies |

### Cost of Goods (4xxx)

| Code | Name        | Description   |
| ---- | ----------- | ------------- |
| 4000 | Varekostnad | Cost of goods |

### Payroll (5xxx)

| Code | Name | Description |
| ---- | ---- | ----------- |
| 5000 | Lønn | Payroll     |

### Operating Expenses (6xxx–7xxx)

| Code | Name              | Description     |
| ---- | ----------------- | --------------- |
| 6300 | Kontorrekvisita   | Office supplies |
| 6900 | Telefon/internett | Phone/internet  |
| 7700 | Avskrivninger     | Depreciation    |

### Financial Items & Tax (8xxx)

| Code | Name                  | Description                                     |
| ---- | --------------------- | ----------------------------------------------- |
| 8301 | Betalbar skatt        | Tax expense (giver — zero if giving all profit) |
| 8320 | Endring utsatt skatt  | Deferred tax change (NOT for konsernbidrag)     |
| 8931 | Mottatt konsernbidrag | Group contribution received (income)            |
| 8935 | Avsatt konsernbidrag  | Group contribution given (expense)              |

---

## VAT/MVA Codes

Fiken uses string VAT type identifiers. Amounts in the API are always **net** (before VAT) unless documented otherwise.

### Standard Types (most common)

| Code     | Rate | Description                                        |
| -------- | ---- | -------------------------------------------------- |
| `HIGH`   | 25%  | Standard rate — most goods and services            |
| `MEDIUM` | 15%  | Food and drink                                     |
| `LOW`    | 12%  | Passenger transport, cinema, sports events         |
| `NONE`   | 0%   | Exempt — no VAT (e.g., financial services, health) |

### Extended Types

| Code            | Rate   | Description                                       |
| --------------- | ------ | ------------------------------------------------- |
| `RAW_FISH`      | 11.11% | Raw fish (special Norwegian rate)                 |
| `EXEMPT`        | 0%     | Outside VAT system entirely (not just zero-rated) |
| `OUTSIDE`       | 0%     | Reverse charge — EU/international services        |
| `EXEMPT_IMPORT` | 0%     | Import of services from abroad (reverse charge)   |
| `HIGH_DIRECT`   | 25%    | Direct to consumer (rare in B2B)                  |

### Numeric Internal Codes

Fiken's OpenAPI spec also references numeric VAT codes internally:

| Numeric | String Equivalent | Description          |
| ------- | ----------------- | -------------------- |
| 0 / 7   | —                 | No VAT               |
| 1 / 3   | HIGH              | 25% standard         |
| 11 / 31 | MEDIUM            | 15% food/drink       |
| 12 / 32 | RAW_FISH          | 11.11% raw fish      |
| 13 / 33 | LOW               | 12% transport/cinema |
| 14      | —                 | 25% direct           |
| 21      | —                 | Exempt import        |

In API requests, always use the **string** types (`HIGH`, `MEDIUM`, etc.), not numeric codes.

---

## Norwegian Accounting Terms (Glossary)

| Norwegian            | English                    | Notes                               |
| -------------------- | -------------------------- | ----------------------------------- |
| Bilag                | Voucher / document         | Supporting document for a posting   |
| Bokføring            | Bookkeeping                | Recording transactions              |
| Driftsregnskap       | Operating accounts         |                                     |
| Fri postering        | Free/general journal entry | Manual posting via API              |
| Hovedbok             | General ledger             |                                     |
| Konsernbidrag        | Group contribution         | Tax-motivated intercompany transfer |
| Kontoplan            | Chart of accounts          |                                     |
| Kundefordringer      | Accounts receivable        | Account 1500                        |
| Leverandørgjeld      | Accounts payable           | Account 2400                        |
| MVA / Merverdiavgift | VAT / Value Added Tax      |                                     |
| MVA-melding          | VAT return                 | Filed via Altinn                    |
| Resultatregnskap     | Income statement / P&L     |                                     |
| Reskontro            | Subledger                  | Format: 1500:10001                  |
| Saldobalanse         | Trial balance              |                                     |
| Skattemelding        | Tax return                 | Filed via Altinn                    |
| Årsregnskap          | Annual accounts            | Must be filed with Brønnøysund      |
| Øre                  | Cent (1/100 of NOK)        | All API amounts in øre              |

---

## Fiscal Year Conventions

- Norwegian fiscal year = calendar year (1 Jan – 31 Dec) for most companies
- Konsernbidrag and year-end entries are dated **31.12** of the fiscal year
- Annual accounts must be filed with Brønnøysundregistrene
- Skattemelding (tax return) deadline: typically 31 May
- Konsernbidrag must be decided by the board BEFORE annual accounts are filed (before 30 June)
