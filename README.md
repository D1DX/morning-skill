# Morning Skill

[![Author](https://img.shields.io/badge/Author-Daniel_Rudaev-000000?style=flat)](https://github.com/daniel-rudaev)
[![Studio](https://img.shields.io/badge/Studio-D1DX-000000?style=flat)](https://d1dx.com)
[![Morning](https://img.shields.io/badge/Morning_(חשבונית_ירוקה)-Skill-00B37E?style=flat&logoColor=white)](https://www.greeninvoice.co.il)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](./LICENSE)

Morning (Green Invoice / חשבונית ירוקה) skill for AI agents. Covers expense management, file upload via presigned S3 URLs, accounting classifications, expense search, and webhooks. Built from production expense import automation.

## What's Included

| Topic | What it covers |
|-------|---------------|
| Authentication | Bearer token via `/account/token`, production vs sandbox base URLs |
| Endpoints Quick Reference | All expense and classification endpoints in one table |
| Create Expense | Full request structure, document types, payment types, expense statuses — with Hebrew labels |
| File Upload (Two-Step) | Presigned S3 URL flow: create expense → get presigned URL → upload to S3 → poll for attachment |
| File Upload Gotchas | requests library required, field ordering, async processing (5–15s), file type limits |
| Upload-First Flow | Upload file → create draft (Morning runs OCR and creates draft automatically) |
| Search Expenses | Filtered search by date, supplier, amount, status, classification |
| Classifications | List all accounting classifications with codes and VAT settings |
| Webhooks | expense-draft/parsed, expense-file/updated, expense-draft/declined, file/infected |
| Common Errors | Error codes 3306, 3310, 3311, 3312 with Hebrew descriptions and fixes |
| Bulk Upload Pattern | Complete Python pattern for uploading many expenses with files from external sources |
| Rate Limits | API ~10 req/s, presigned URL expiry (~1 min), sequential S3 upload recommendation |

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
└── SKILL.md    — Main skill (auth, expense CRUD, file upload flow, classifications, error codes)
```

## Key Gotcha: supplier placement

The most common error (code 3311) is placing `supplier` inside a `data` object. It must be at the root level of the expense JSON. The skill documents this and all other critical payload rules.

## Sources

- **Morning API:** Verified against the [Morning/Green Invoice Apiary specification](https://jsapi.apiary.io/apis/greeninvoice.json) and live production usage (March 2026).
- **File upload flow:** Verified from live S3 presigned URL uploads and polling behavior.

## Credits

Built by [Daniel Rudaev](https://github.com/daniel-rudaev) at [D1DX](https://d1dx.com).

## License

MIT License — Copyright (c) 2026 Daniel Rudaev @ D1DX
