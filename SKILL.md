---
name: morning
description: "Morning (Green Invoice) expense management — create expenses, upload files, manage classifications, search expenses/drafts. Auto-triggers on: Green Invoice API, Morning API, expense upload, חשבונית ירוקה, expense file attachment."
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
    """Upload single Airtable record to Morning with file."""
    headers = {'Authorization': f'Bearer {token}'}

    # 1. Create expense
    clause = record['fields'].get('Clause', '')
    classification = classification_map.get(clause, {})

    expense = {
        'documentType': 305,
        'date': record['date'],
        'reportingDate': record['date'][:8] + '01',
        'number': record['fields'].get('Source ID', record['fields'].get('ID', '')),
        'currency': record['currency'],
        'amount': record['amount'],
        'amountExcludeVat': record['amount'],  # Foreign = no VAT split
        'vat': 0,
        'supplier': {
            'name': record['fields'].get('Entity Name', ['Unknown'])[0],
            'taxId': '',
            'country': 'US'  # Adjust per record
        },
        'accountingClassification': classification,
    }

    resp = requests.post(f'{base_url}/expenses', json=expense, headers=headers)
    resp.raise_for_status()
    expense_id = resp.json()['id']

    # 2. Upload file
    file_url = record['file_url'][0]['url']
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

