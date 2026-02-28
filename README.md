# skill-fiken

[Fiken](https://fiken.no) accounting API skill for AI agents.

Gives your AI agent full access to the Fiken API v2: invoices, credit notes, contacts, purchases, sales, inbox (document upload), journal entries, projects, time tracking, offers, order confirmations, and more.

## Install

Copy the skill into your project's skills directory:

```bash
mkdir -p skills/fiken
curl -sL https://raw.githubusercontent.com/kristianvast/skill-fiken/main/SKILL.md -o skills/fiken/SKILL.md
```

Or clone the whole repo:

```bash
git clone https://github.com/kristianvast/skill-fiken.git skills/fiken
```

## Requirements

- An AI agent that supports skill files (e.g. [OpenCode](https://github.com/sst/opencode), [Claude Code](https://docs.anthropic.com/en/docs/claude-code), or any skill-compatible gateway)
- `curl` and `jq` installed
- A [Fiken](https://fiken.no) account with the API module activated

## Setup

### 1. Activate the Fiken API module

In Fiken: **Innstillinger → Moduler → Fiken API** (99 NOK/month). Activate for each company you want to access.

### 2. Generate an API token

**Rediger konto → API → Personlige API-nøkler → Ny nøkkel**

One token gives access to all companies you have permissions for.

### 3. Set the environment variable

Make `FIKEN_API_TOKEN` available to your agent process:

```bash
export FIKEN_API_TOKEN="your-token-here"
```

For systemd-managed services, add to your service file:

```ini
Environment=FIKEN_API_TOKEN=your-token-here
```

### 4. Verify

Ask your agent: *"List my Fiken companies"* — it should return the company names and slugs.

## What's covered

| Category | Read | Write |
|---|---|---|
| Companies | ✅ | — |
| Contacts | ✅ | ✅ |
| Invoices & Drafts | ✅ | ✅ |
| Credit Notes | ✅ | ✅ |
| Products | ✅ | ✅ |
| Sales & Payments | ✅ | ✅ |
| Purchases & Payments | ✅ | ✅ |
| Bank Accounts & Balances | ✅ | — |
| Accounts & Balances | ✅ | — |
| Inbox (Document Upload) | ✅ | ✅ |
| Journal Entries | ✅ | ✅ |
| Transactions | ✅ | ✅ |
| Projects | ✅ | ✅ |
| Time Tracking | ✅ | ✅ |
| Offers | ✅ | ✅ |
| Order Confirmations | ✅ | ✅ |
| Attachments | ✅ | ✅ |
| Draft Management | ✅ | ✅ |

## Safety

All write operations (POST/PUT/DELETE) require explicit user confirmation before execution. The agent will show you the full curl command and payload and wait for your "yes" before running it.

There is no Fiken sandbox — all API calls affect live accounting data.

## Multi-company support

One API token accesses all companies you have permissions for. When the context is ambiguous, the agent will ask which company you mean.

## Norwegian accounting context

The skill includes Norwegian accounting conventions built in:

- Amounts in øre (integer cents — `100000` = 1000.00 NOK)
- MVA/VAT codes (HIGH 25%, MEDIUM 15%, LOW 12%, NONE 0%)
- Common Norsk Standard Kontoplan account codes

## Rate limits

Fiken enforces strict rate limits: max 1 concurrent request, max 4 requests/second. Violation results in 429 errors or account ban. The skill instructs the agent to never parallelize calls.

## License

MIT
