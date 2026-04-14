# Morning Skill

[![Author](https://img.shields.io/badge/Author-Daniel_Rudaev-000000?style=flat)](https://github.com/daniel-rudaev)
[![Studio](https://img.shields.io/badge/Studio-D1DX-000000?style=flat)](https://d1dx.com)
[![Morning](https://img.shields.io/badge/Morning_(חשבונית_ירוקה)-Skill-00B37E?style=flat&logoColor=white)](https://www.greeninvoice.co.il)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](./LICENSE)

Full Morning (Green Invoice / חשבונית ירוקה) API skill for AI agents. Covers the complete API surface: expenses, file upload, classifications, clients, suppliers, items, documents (invoices, receipts, quotes), payments, business settings, and reference data. Expenses and file upload are documented with raw API patterns; all other endpoints use [morning-cli](https://github.com/Jango-AI-com/morning-cli) as the execution layer.

**This skill uses Morning's official public API** ([Apiary spec](https://jsapi.apiary.io/apis/greeninvoice.json)) — not a reverse-engineered or unofficial API.

## When to Use What

| Method | When |
|--------|------|
| **Morning UI** | One-off entry, draft review, classification setup |
| **morning-cli** (this skill, sections 10–18) | Clients, suppliers, items, documents, payments, business, reference data |
| **Raw API / Python** (this skill, sections 1–9) | Expense bulk imports, file uploads with OCR, automation scripts |
| **Morning + n8n/Make** | Webhook-driven flows (e.g., auto-create expense when a file lands in Google Drive) |

## What's Included

**Sections 1–9 — Raw API (Python)**

| Topic | What it covers |
|-------|---------------|
| Authentication | Bearer token via `/account/token`, production vs sandbox base URLs |
| Expense Endpoints | All expense and classification endpoints in one table |
| Create Expense | Full request structure, document types, payment types, statuses — with Hebrew labels |
| File Upload (Two-Step) | Presigned S3 URL flow: create expense → get presigned URL → upload to S3 → poll for attachment |
| File Upload Gotchas | requests library required, field ordering, async processing (5–15s), file type limits |
| Upload-First Flow | Upload file → create draft (Morning runs OCR and creates draft automatically) |
| Search Expenses | Filtered search by date, supplier, amount, status, classification |
| Classifications | List all accounting classifications with codes and VAT settings |
| Webhooks | expense-draft/parsed, expense-file/updated, expense-draft/declined, file/infected |
| Common Errors | Error codes 3306, 3310, 3311, 3312 with Hebrew descriptions and fixes |
| Bulk Upload Pattern | Complete Python pattern for uploading many expenses with files from external sources |

**Sections 10–18 — morning-cli**

| Section | What it covers |
|---------|---------------|
| 10 | morning-cli setup and routing guide |
| 11 | Clients — CRUD, search, balance, associate, merge |
| 12 | Suppliers — CRUD, search, merge |
| 13 | Documents — types, statuses, search, preview, create, download |
| 14 | Items — price list CRUD |
| 15 | Payments — tokens, charge, payment form |
| 16 | Business — profile, numbering sequences, logo/signature upload |
| 17 | Partners — partner-tier account operations |
| 18 | Reference data — currencies (live rates), countries, cities, occupations |

## Install

### Claude Code

```bash
git clone https://github.com/D1DX/morning-skill.git
cp -r morning-skill ~/.claude/skills/morning
```

Or as a git submodule:

```bash
git submodule add https://github.com/D1DX/morning-skill.git path/to/skills/morning
```

### Other AI Agents

Copy `SKILL.md` into your agent's prompt or knowledge directory. The skill is structured markdown — works with any LLM agent that reads reference files.

## Structure

```
morning-skill/
└── SKILL.md    — Full API skill (18 sections: expenses + file upload via raw API; all other endpoints via morning-cli)
```

## Key Gotcha: supplier placement

The most common error (code 3311) is placing `supplier` inside a `data` object. It must be at the root level of the expense JSON. The skill documents this and all other critical payload rules.

## Sources

- **Sections 1–9:** Verified against the [Morning/Green Invoice Apiary specification](https://jsapi.apiary.io/apis/greeninvoice.json) and live production usage (March 2026).
- **Sections 10–18:** Verified against live sandbox via [morning-cli](https://github.com/Jango-AI-com/morning-cli) (April 2026).

## Credits

Built by [Daniel Rudaev](https://github.com/daniel-rudaev) at [D1DX](https://d1dx.com).

## License

MIT License — Copyright (c) 2026 Daniel Rudaev @ D1DX
