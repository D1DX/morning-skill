---
name: morning
description: "Morning (Green Invoice) full API reference — expenses, file upload, classifications, clients, suppliers, items, documents (invoices/receipts), payments, business settings, reference data. Auto-triggers on: Green Invoice API, Morning API, expense upload, invoice creation, חשבונית ירוקה, morning-cli."
disable-model-invocation: false
user-invocable: true
argument-hint: "task description"
---

# Morning (Green Invoice) Expenses API Skill

Complete reference for the Morning/Green Invoice Expenses API including file uploads, classifications, drafts, and bulk operations.

**As of March 2026.** Source: Apiary spec at `https://jsapi.apiary.io/apis/greeninvoice.json`.

---

## 1. Authentication

```python
import requests, json

resp = requests.post("https://api.greeninvoice.co.il/api/v1/account/token",
                     json={'id': API_KEY, 'secret': API_SECRET})
token = resp.json()['token']
headers = {'Authorization': f'Bearer {token}'}
```

**Base URLs:**
- Production: `https://api.greeninvoice.co.il/api/v1`
- Sandbox: `https://sandbox.d.greeninvoice.co.il/api/v1`
- File upload gateway: `https://apigw.greeninvoice.co.il/file-upload/v1/url`

---

## 2. Endpoints Quick Reference

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/expenses` | Create expense |
| `GET` | `/expenses/{id}` | Get expense |
| `PUT` | `/expenses/{id}` | Update expense |
| `DELETE` | `/expenses/{id}` | Delete expense |
| `POST` | `/expenses/search` | Search expenses |
| `POST` | `/expenses/{id}/open` | Reopen reported expense |
| `POST` | `/expenses/{id}/close` | Report/close expense |
| `GET` | `/expenses/statuses` | Allowed statuses |
| `POST` | `/expenses/drafts/search` | Search expense drafts |
| `GET` | `/accounting/classifications/search` | List classifications (POST with `{}` body) |
| `GET` | `/accounting/classifications/map` | Classification map |
| `PUT` | `/accounting/classifications/{id}` | Update classification |
| `GET` | `apigw.../file-upload/v1/url` | Get presigned S3 upload URL |

---

## 3. Create Expense

```python
expense = {
    'documentType': 305,           # See document types below
    'date': '2024-09-18',          # Document date
    'reportingDate': '2024-09-01', # First day of reporting month
    'number': 'INV-001',           # Document number / source ID
    'currency': 'USD',             # ILS, USD, EUR, THB, etc.
    'amount': 100.00,              # Total including VAT
    'amountExcludeVat': 85.00,     # Amount before VAT
    'vat': 15.00,                  # VAT amount
    'supplier': {                  # MUST be at root level, NOT inside data
        'name': 'Vendor Name',
        'taxId': '',               # Israeli ח.פ. or empty for foreign
        'country': 'US'            # ISO country code
    },
    'accountingClassification': {
        'id': 'd0c61a4a-...',      # Morning classification UUID
        'title': 'הוצ\' כלי עבודה',
        'income': 100,
        'vat': 100
    },
    'description': '',             # Optional
    'remarks': '',                 # Optional
    'paymentType': 3,              # Optional, see payment types
}

resp = requests.post(f'{base}/expenses', json=expense, headers=headers)
created = resp.json()
expense_id = created['id']  # UUID
```

**Critical:** `supplier` goes at the root level of the JSON, NOT inside a `data` object. Putting it in `data.supplier` causes error 3311.

### Document Types
| Value | Hebrew | Translation |
|-------|--------|-------------|
| 20 | חשבון / אישור תשלום | Invoice / Payment Confirmation |
| 305 | חשבונית מס | Tax Invoice |
| 320 | חשבונית מס / קבלה | Tax Invoice / Receipt |
| 330 | חשבונית זיכוי | Credit Invoice |
| 400 | קבלה | Receipt |
| 405 | קבלה על תרומה | Donation Receipt |

### Payment Types
| Value | Hebrew | Translation |
|-------|--------|-------------|
| -1 | לא שולם | Unpaid |
| 1 | מזומן | Cash |
| 2 | צ'ק | Check |
| 3 | כרטיס אשראי | Credit Card |
| 4 | העברה בנקאית | Bank Transfer |
| 5 | פייפאל | PayPal |
| 10 | אפליקציית תשלום | Payment App |
| 11 | אחר | Other |

### Expense Statuses
| Value | Hebrew | Translation |
|-------|--------|-------------|
| 10 | הוצאה פתוחה | Open (draft) |
| 20 | הוצאה מדווחת | Reported |
| 30 | סומנה ידנית כפתוחה | Manually reopened |
| 100 | הוצאה מחוקה | Deleted |

---

## 4. File Upload (Two-Step Presigned S3 URL)

This is the **critical flow** for attaching PDFs to expenses. Morning does OCR on the uploaded file.

### Flow: Create Expense → Upload File

```python
import requests, json, uuid

# Step 1: Create expense (no file yet)
expense = { ... }  # See section 3
resp = requests.post(f'{base}/expenses', json=expense, headers=headers)
expense_id = resp.json()['id']

# Step 2: Get presigned S3 URL with expense ID
data_param = json.dumps({"source": 5, "id": expense_id, "state": "expense"})
resp = requests.get(
    'https://apigw.greeninvoice.co.il/file-upload/v1/url',
    params={'context': 'expense', 'data': data_param},
    headers=headers
)
presigned = resp.json()

# Step 3: Download file (e.g. from Airtable)
pdf_data = requests.get(airtable_file_url).content

# Step 4: Upload to S3 — MUST use requests library for proper multipart
filename = f"expense_{uuid.uuid4().hex[:8]}.pdf"
upload_fields = {}
for key, value in presigned['fields'].items():
    upload_fields[key] = (None, value)
upload_fields['file'] = (filename, pdf_data, 'application/pdf')

resp = requests.post(presigned['url'], files=upload_fields)
assert resp.status_code == 204  # S3 returns 204 No Content on success

# Step 5: Poll for file attachment (~5-15 seconds)
import time
for _ in range(6):
    time.sleep(5)
    resp = requests.get(f'{base}/expenses/{expense_id}', headers=headers)
    if resp.json().get('data', {}).get('fileKey'):
        break  # File attached!
```

### Presigned URL `data` Parameter

| Scenario | `data` value |
|----------|-------------|
| New expense draft (no expense yet) | `{"source": 5}` |
| Attach file to existing expense | `{"source": 5, "id": "<expense_id>", "state": "expense"}` |

**`source: 5`** is mandatory in all cases (means "API upload").

### Critical Gotchas

1. **Use `requests` library** for S3 multipart upload. Manual boundary construction with `urllib` does NOT work — the multipart encoding is subtly different and Morning's backend silently ignores the file.
2. **All presigned `fields` must be included** in the multipart upload, appended BEFORE the file.
3. **The `file` field must be last** in the multipart form.
4. **Filenames must be unique** per upload to avoid S3 deduplication.
5. **Async processing:** File attachment takes 5-15 seconds after S3 upload. Poll the expense to confirm.
6. **File types:** PDF, PNG, JPG, GIF, SVG. Max 16 MB.
7. **Reported expenses (status=20) cannot have files updated.**

### Flow: Upload File → Create Draft (no expense yet)

```python
# Get presigned URL WITHOUT expense ID
data_param = json.dumps({"source": 5})
resp = requests.get(
    'https://apigw.greeninvoice.co.il/file-upload/v1/url',
    params={'context': 'expense', 'data': data_param},
    headers=headers
)
# Upload to S3 (same as above)
# Morning creates an expense DRAFT asynchronously
# Poll via POST /expenses/drafts/search
```

**Note:** Draft creation is fully async and may take longer. The draft must be manually approved to become an actual expense.

---

## 5. Search Expenses

```python
resp = requests.post(f'{base}/expenses/search', json={
    'fromDate': '2024-01-01',
    'toDate': '2024-12-31',
    'page': 1,
    'pageSize': 25,
    # Optional filters:
    'supplierName': 'Amazon',
    'minAmount': 10,
    'maxAmount': 500,
    'reported': True,
    'accountingClassificationId': '...',
}, headers=headers)

result = resp.json()
# result = {total, page, pageSize, pages, items: [...]}
```

**Note:** `page` and `pageSize` are required. Empty body `{}` works for fetching all.

---

## 6. Classifications

### List All Classifications
```python
resp = requests.post(f'{base}/accounting/classifications/search',
                     json={}, headers=headers)
for c in resp.json()['items']:
    print(f"{c['id']}: {c['title']} (code={c['code']}, key={c['key']}, "
          f"income={c['income']}, vat={c['vat']})")
```

---

## 7. Webhooks

Configure via Morning GUI only (Settings → Developer Tools).

| Event | Fires When |
|-------|------------|
| `expense-draft/parsed` | Draft created from uploaded file |
| `expense-file/updated` | Existing expense file replaced |
| `expense-draft/declined` | Draft rejected |
| `file/infected` | Uploaded file has virus |

---

## 8. Common Errors

| Code | Hebrew | Cause | Fix |
|------|--------|-------|-----|
| 3306 | מספר מסמך הוצאה לא תקין | Missing `number` field | Add document number |
| 3310 | חודש דיווח לא תקין | Missing `reportingDate` | Add first-of-month date |
| 3311 | נא למלא פרטי ספק | Supplier missing or in wrong location | Move `supplier` to root level (not `data.supplier`) |
| 3312 | נא למלא פרטי סוג הוצאה | Missing `accountingClassification` | Add with valid `id` from classifications list |
| 1100 | שדה {id} לא תקין | Invalid UUID in path | Check expense ID is valid UUID |

---

## 9. Bulk Upload Pattern

For uploading many expenses from Airtable to Morning:

```python
import requests, json, time, uuid

def upload_expense(record, token, base_url, classification_map):
    """Upload single record to Morning with file."""
    headers = {'Authorization': f'Bearer {token}'}

    # 1. Create expense
    category = record.get('category', '')
    classification = classification_map.get(category, {})

    expense = {
        'documentType': 305,
        'date': record['date'],
        'reportingDate': record['date'][:8] + '01',
        'number': record.get('document_number', ''),
        'currency': record['currency'],
        'amount': record['amount'],
        'amountExcludeVat': record['amount'],  # Foreign = no VAT split
        'vat': 0,
        'supplier': {
            'name': record.get('supplier_name', 'Unknown'),
            'taxId': '',
            'country': 'US'  # Adjust per record
        },
        'accountingClassification': classification,
    }

    resp = requests.post(f'{base_url}/expenses', json=expense, headers=headers)
    resp.raise_for_status()
    expense_id = resp.json()['id']

    # 2. Upload file
    file_url = record['file_url']
    pdf_data = requests.get(file_url).content

    data_param = json.dumps({"source": 5, "id": expense_id, "state": "expense"})
    presigned = requests.get(
        'https://apigw.greeninvoice.co.il/file-upload/v1/url',
        params={'context': 'expense', 'data': data_param},
        headers=headers
    ).json()

    upload_fields = {k: (None, v) for k, v in presigned['fields'].items()}
    upload_fields['file'] = (f"doc_{uuid.uuid4().hex[:8]}.pdf",
                             pdf_data, 'application/pdf')

    resp = requests.post(presigned['url'], files=upload_fields)
    assert resp.status_code == 204

    # 3. Wait for file attachment
    for _ in range(6):
        time.sleep(5)
        exp = requests.get(f'{base_url}/expenses/{expense_id}',
                          headers=headers).json()
        if exp.get('data', {}).get('fileKey'):
            return expense_id, True

    return expense_id, False  # File didn't attach in time
```

### Rate Limits
- API: ~10 requests/second (undocumented, be conservative)
- S3 uploads: No strict limit but process sequentially to avoid presigned URL expiry
- Presigned URLs expire in ~1 minute (check Policy `expiration` field)

---

## 10. Other Endpoints via morning-cli

Sections 11–18 document these command groups. All use **morning-cli** as the execution layer:

```bash
pip install morning-cli
morning-cli auth init --env sandbox   # or production
```

All commands accept `--env sandbox|production` and `--json` for machine-readable output.
Use `morning-cli <group> --help` for the full subcommand list.

**Do NOT use morning-cli for expense file uploads** — use the two-step S3 presigned flow in Section 4.

---

## 11. Clients (`morning-cli client`)

| Subcommand | What it does |
|------------|-------------|
| `search` | List/filter clients with pagination and text search |
| `get` | Fetch a single client by UUID |
| `add` | Create a new client |
| `update` | Patch fields on an existing client |
| `balance` | Get balance data for a client |
| `assoc` | Associate a client with other entity IDs |
| `merge` | Merge this client into a target client (destructive) |
| `delete` | Delete a client by ID |

### Key commands

**Search clients:**
```bash
morning-cli --env sandbox --json client search
morning-cli --env sandbox --json client search --data '{"searchText":"acme","pageSize":25,"page":1}'
```
Response shape:
```json
{
  "ok": true,
  "op": "client.search",
  "data": {
    "pageSize": 25, "page": 1, "total": 1, "pages": 1,
    "items": [
      {
        "id": "b6a1da5e-fdb6-4edd-87b8-e45b611d4370",
        "name": "Test Client",
        "active": true, "send": true,
        "taxId": "", "accountingKey": "b5984494",
        "city": "Tel Aviv", "country": "IL",
        "emails": ["test@example.com"],
        "incomeAmount": 0, "paymentAmount": 0, "balanceAmount": 0,
        "creationDate": 1776155807, "lastUpdateDate": 1776155816,
        "self": false
      }
    ]
  }
}
```

**Get client:**
```bash
morning-cli --env sandbox --json client get <CLIENT_ID>
```

**Create client:**
```bash
morning-cli --env sandbox --json client add --data '{"name":"Acme Ltd","country":"IL","emails":["billing@acme.com"]}'
```

**Update client:**
```bash
morning-cli --env sandbox --json client update <CLIENT_ID> --data '{"city":"Tel Aviv","remarks":"VIP"}'
```

**Get balance:**
```bash
morning-cli --env sandbox --json client balance <CLIENT_ID> --data '{}'
```
Returns `{}` when no transactions exist.

**Associate:**
```bash
morning-cli --env sandbox --json client assoc <CLIENT_ID> --data '{"ids":["<OTHER_CLIENT_ID>"]}'
```

**Merge (destructive):**
```bash
morning-cli --env sandbox --json client merge <SOURCE_ID> --data '{"id":"<TARGET_ID>"}'
```

**Delete:**
```bash
morning-cli --env sandbox --json client delete <CLIENT_ID> --yes
```

### Gotchas
- `client balance` requires `--data '{}'` — the flag is mandatory even with an empty body
- `client assoc` requires `ids` in body — omitting returns error 2402
- `send: true` is set automatically on creation (clients receive documents); suppliers default `send: false`
- Dates are Unix timestamps (seconds), not ISO strings
- `--file FILE` accepted as alternative to `--data` on add/update/search

---

## 12. Suppliers (`morning-cli supplier`)

| Subcommand | What it does |
|------------|-------------|
| `search` | List/filter suppliers with pagination and text search |
| `get` | Fetch a single supplier by UUID |
| `add` | Create a new supplier |
| `update` | Patch fields on an existing supplier |
| `merge` | Merge this supplier into a target supplier (destructive) |
| `delete` | Delete a supplier by ID |

### Key commands

**Search suppliers:**
```bash
morning-cli --env sandbox --json supplier search
morning-cli --env sandbox --json supplier search --data '{"searchText":"acme","pageSize":25,"page":1}'
```
Response shape mirrors clients — key differences: has `businessId`, `fax`, `mobile`, `accountingClassificationId`; lacks `nameAliases`, `bankName/Branch/Account`, `self`.

**Create supplier:**
```bash
morning-cli --env sandbox --json supplier add --data '{"name":"Acme Vendor","country":"IL","taxId":"514756428"}'
```

**Update supplier:**
```bash
morning-cli --env sandbox --json supplier update <SUPPLIER_ID> --data '{"city":"Tel Aviv"}'
```

**Merge (destructive):**
```bash
morning-cli --env sandbox --json supplier merge <SOURCE_ID> --data '{"id":"<TARGET_ID>"}'
```

**Delete:**
```bash
morning-cli --env sandbox --json supplier delete <SUPPLIER_ID> --yes
```

### Gotchas
- No `assoc` or `balance` subcommands — those are client-only
- `accountingClassificationId` links to an expense category — set this to auto-classify expenses from this supplier
- `send: false` is the default for suppliers
- Aggregations shape differs from client search — no `byName.buckets` key

---

## 13. Documents (`morning-cli document`)

| Subcommand | What it does |
|------------|-------------|
| `types` | List all document type IDs and names (Hebrew) |
| `statuses` | List all document status IDs and names |
| `templates` | List visual templates |
| `info` | Get defaults + next document number for a given type |
| `search` | Search/list documents with filters |
| `payments-search` | Search payment rows across documents |
| `get` | Fetch a single document by ID |
| `linked` | Fetch documents linked to a given document |
| `create` | Create a new document |
| `preview` | Preview without creating |
| `open` | Reopen a closed document |
| `close` | Close an open document |
| `download` | Download document PDF |

### Document type IDs
| ID | English |
|----|---------|
| 10 | Quote |
| 20 | Proforma / Payment confirmation |
| 300 | Transaction invoice |
| 305 | Tax invoice |
| 320 | Tax invoice + receipt (most common) |
| 330 | Credit invoice |
| 400 | Receipt |
| 500 | Purchase order |

### Document status IDs
| ID | Hebrew |
|----|--------|
| 0 | Open |
| 1 | Closed |
| 2 | Manually closed |
| 3 | Cancelling |
| 4 | Cancelled |

### Key commands

**Get document types:**
```bash
morning-cli --env sandbox --json document types
morning-cli --env sandbox --json document types --lang en
```

**Get info/defaults for a type:**
```bash
morning-cli --env sandbox --json document info --type 320
```
Returns: next document number, VAT rate, allowed VAT types, max backdating days, `token` (required by some create paths), currency/language defaults, feature flags.

**Search documents:**
```bash
morning-cli --env sandbox --json document search --data '{"type":320,"page":1}'
```
Useful filters: `type`, `status`, `clientId`, `fromDate`, `toDate`, `page`, `pageSize`.

**Preview document (non-destructive):**
```bash
morning-cli --env sandbox --json document preview --data '{
  "type": 320,
  "client": {"id": "<CLIENT_ID>"},
  "currency": "ILS",
  "vatType": 0,
  "date": "2026-04-14",
  "incomeRows": [{"description": "Service", "quantity": 1, "price": 1000, "currency": "ILS", "vatType": 0}],
  "paymentRows": [{"date": "2026-04-14", "type": 1, "price": 1180, "currency": "ILS"}]
}'
```

**Create document (same payload as preview — irreversible):**
```bash
morning-cli --env sandbox --json document create --data '{ ... same as preview ... }'
```
Required fields: `type`, `client.id`, `currency`, `vatType`, `date`, at least one `incomeRows` entry with `description` + `price`.

### Payment type IDs (for `paymentRows[].type`)
| ID | Type |
|----|------|
| 1 | Bank transfer |
| 2 | Check |
| 3 | Cash |
| 4 | Credit card |
| 5 | PayPal |
| 10 | Other |

### Gotchas
- **Always preview before create** — creation is irreversible and increments the document number sequence
- `document info` returns a `token` — some create paths require passing it to confirm the next number; fetch fresh immediately before create
- `vatType: 0` = standard VAT included; `vatType: 1` = zero VAT (exempt)
- `maxDaysBack` enforced by API — backdating beyond it will fail (~59 days in sandbox)
- `signed: true` in the payload finalizes (issues) the document; omit for draft if account supports it (`unsignedEnabled` in `document info`)

---

## 14. Items (`morning-cli item`)

| Subcommand | What it does |
|------------|-------------|
| `search` | Search/list price list items |
| `get` | Fetch a single item by UUID |
| `add` | Create a new price list item |
| `update` | Update an existing item |
| `delete` | Delete an item |

### Key commands

**List items:**
```bash
morning-cli --env sandbox --json item search
morning-cli --env sandbox --json item search --data '{"page":1,"pageSize":25}'
```
Response shape:
```json
{
  "ok": true, "op": "item.search",
  "data": {
    "items": [
      {
        "id": "e0b445f5-9a02-4da7-bac1-c641a5d26d8c",
        "name": "Test Item", "description": "",
        "price": 200, "currency": "ILS",
        "creationDate": 1776155844
      }
    ]
  }
}
```

**Create item:**
```bash
morning-cli --env sandbox --json item add --data '{"name":"Item Name","description":"Item description","price":100,"currency":"ILS"}'
```
Required fields: `name`, `description`, `price`, `currency`

**Update item:**
```bash
morning-cli --env sandbox --json item update <ITEM_ID> --data '{"name":"Updated","description":"Updated desc","price":150,"currency":"ILS"}'
```

**Delete:**
```bash
morning-cli --env sandbox --json item delete <ITEM_ID> --yes
```

### Gotchas
- `update` is a **full replace** — omitting any field clears it. Always send all fields you want to keep.
- Error 2300 = missing name, error 2301 = missing description

---

## 15. Payments (`morning-cli payment`)

| Subcommand | What it does |
|------------|-------------|
| `tokens-search` | Search saved payment tokens (stored cards) |
| `charge` | Charge a saved payment token |
| `form` | Create a hosted payment form |

### Key commands

**Search tokens:**
```bash
morning-cli --env sandbox --json payment tokens-search
```

**Charge a token:**
```bash
morning-cli --env sandbox --json payment charge <TOKEN_ID> --data '{"amount":100,"currency":"ILS"}'
```

**Create payment form:**
```bash
morning-cli --env sandbox --json payment form --data '{"documentType":<type>,"client":{...},"income":[...]}'
```

### Gotchas
- `payment form` returns error 2403 in sandbox — "Document type not supported for this business type." This is a sandbox account tier restriction, not a payload error.
- `payment charge` requires a real token UUID from `tokens-search` — passing any non-existent UUID returns error 1100
- `tokens-search` aggregations returns `[]` (empty array) not an object — shape differs from other search endpoints

---

## 16. Business (`morning-cli business`)

| Subcommand | What it does |
|------------|-------------|
| `current` | Active business profile |
| `list` | All businesses under the account |
| `get` | Specific business by UUID |
| `types` | Legal entity types |
| `footer` | Document footer configuration |
| `numbering-get` | Document numbering sequences per type |
| `numbering-update` | Update numbering sequences |
| `update` | Update business profile |
| `file-upload` | Upload logo/signature (`--type 0`=logo, `1`=signature) |
| `file-delete` | Delete uploaded file by type |

### Key commands

**Get current business:**
```bash
morning-cli --env sandbox --json business current
```
Response includes: `id`, `type`, `taxId`, `name`, `address`, `cityId`, `category`, `subCategory`, `exemption`, `inboundEmail`, `settings` (currency, lang, vatType, taxAuthorityConnected).

**Get numbering sequences:**
```bash
morning-cli --env sandbox --json business numbering-get
```
Returns one entry per document type: `type`, `number`, `nextNumber`, `lastDocumentDate`, `active`.

**Get business types:**
```bash
morning-cli --env sandbox --json business types --lang en
```
| ID | English |
|----|---------|
| 1 | Licensed dealer |
| 2 | Ltd. company |
| 3 | Exempt dealer |
| 4 | Non-profit / NGO |
| 5 | Public benefit company |
| 6 | Partnership |

### Gotchas
- `business current` and `business list` return identical data — `current` is shorthand for the single credentialed business
- `business footer` returns `{}` when no footer is configured — not an error
- `business get` requires the UUID, not the taxId

---

## 17. Partners (`morning-cli partner`)

| Subcommand | What it does |
|------------|-------------|
| `users` | List all users connected to the partner account |
| `get` | Get a specific user by email |
| `connect` | Connect a business to the partner account |
| `disconnect` | Disconnect a business |

### Gotchas
- **All partner commands return HTTP 401 with regular credentials.** Partner API requires an accountant/partner-tier Morning account. Not usable with standard business API keys.

---

## 18. Reference Data (`morning-cli tools`)

| Subcommand | What it does |
|------------|-------------|
| `currencies` | All supported currencies with live ILS exchange rates |
| `countries` | All countries with alpha2/alpha3 codes and calling codes |
| `cities` | Cities for a given country (`--country` required, ISO alpha2) |
| `occupations` | Full occupation/category tree for business classification |

### Key commands

**Get currencies (live rates):**
```bash
morning-cli --env sandbox --json tools currencies
```
Returns 31 currencies vs ILS base. Each has `spot.current` (real-time) and `closing.current` (previous business day official rate).

Supported: `ILS, AED, AUD, BGN, BRL, CAD, CHF, CNY, CZK, DKK, EUR, GBP, HKD, HUF, IDR, INR, JPY, KRW, MXN, NOK, NZD, PHP, PLN, RON, RUB, SEK, SGD, THB, TRY, USD, ZAR`

**Get countries:**
```bash
morning-cli --env sandbox --json tools countries --locale en
```

**Get cities:**
```bash
morning-cli --env sandbox --json tools cities --country IL
```
City `id` maps to `business.cityId`.

**Get occupations tree:**
```bash
morning-cli --env sandbox --json tools occupations --locale en
```
Two-level tree: parent `category` + child `subCategory`. IDs map to `business.category` / `business.subCategory`.

### Gotchas
- `tools cities --country` requires ISO alpha2 (`IL`, `US`) — not alpha3
- All tools default to `--locale he` (Hebrew). Pass `--locale en` for English.
- `currencies` returns **live rates** — not static. Spot and closing rates differ.

