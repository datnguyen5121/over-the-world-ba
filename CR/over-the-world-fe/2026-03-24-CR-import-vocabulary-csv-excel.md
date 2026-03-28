# CR: Import Vocabulary from CSV/Excel

**Date:** 2026-03-24  
**Module:** Word Management  
**Type:** Change Request — New Feature  

---

## 1. Overview

Allow administrators and teachers to bulk-import vocabulary entries by uploading a CSV or Excel (.xlsx) file. The system validates the file, presents a preview with error highlights, and lets the user confirm or cancel before persisting the data.

## 2. User Story

> As an **Admin/Teacher**, I want to upload a CSV or Excel file containing vocabulary data so that I can add many words at once without creating them one by one.

## 3. Acceptance Criteria

| # | Criterion |
|---|-----------|
| 1 | A new **Import** button is displayed on the Word Management page (between Delete and Export CSV) |
| 2 | Clicking it opens a dialog with a file upload area accepting `.csv` and `.xlsx` |
| 3 | A **Download Template** link provides a pre-formatted file with correct headers |
| 4 | After file upload, the backend validates every row and returns a preview (valid rows + error rows) |
| 5 | The user sees a summary: total rows, valid rows, error count with per-row messages |
| 6 | Clicking **Confirm Import** persists only the valid rows and reports the count |
| 7 | Clicking **Cancel** discards the import session |
| 8 | Duplicate detection: rows whose `word` already exists in the database are flagged as errors |
| 9 | Maximum file size: **5 MB** |
| 10 | Maximum rows per import: **500** |

## 4. File Format Specification

### Required Columns

| Column | Required | Max Length | Notes |
|--------|----------|-----------|-------|
| word | Yes | 50 | Chinese word |
| pinyin | Yes | 100 | Pronunciation |
| meaning | Yes | — | Vietnamese/English meaning |

### Optional Columns

| Column | Max Length | Notes |
|--------|-----------|-------|
| description | — | Additional description |
| partOfSpeech | 50 | e.g., noun, verb, adjective |
| topic | 100 | Topic/theme |
| categories | — | Category names separated by semicolons (`;`) |

### Template Example

```
word,pinyin,meaning,description,partOfSpeech,topic,categories
你好,nǐ hǎo,Xin chào,Lời chào cơ bản,interjection,Greetings,Basic;Conversation
谢谢,xiè xiè,Cảm ơn,,interjection,Greetings,Basic
```

## 5. Workflow

```
User clicks "Import" button
  → Dialog opens with upload area
  → User selects CSV/XLSX file (max 5 MB)
  → Frontend POSTs file to POST /vocabularies/import/validate
  → Backend parses file, validates all rows
  → Backend returns { importToken, validRows[], errorRows[], summary }
  → Dialog shows preview table (valid in green, errors in red with messages)
  → User clicks "Confirm Import"
  → Frontend POSTs importToken to POST /vocabularies/import/confirm
  → Backend persists valid rows, returns { importedCount }
  → Success toast, dialog closes, vocabulary list refreshes
```

## 6. API Endpoints

### 6.1 `GET /vocabularies/import/template`

Returns a CSV template file as a downloadable blob.

**Response:** `text/csv` blob with UTF-8 BOM header row.

### 6.2 `POST /vocabularies/import/validate`

Uploads and validates the file without persisting.

**Request:** `multipart/form-data` with field `file` (CSV or XLSX, max 5 MB)

**Response (200):**
```json
{
  "importToken": "uuid-string",
  "summary": {
    "totalRows": 50,
    "validCount": 47,
    "errorCount": 3
  },
  "validRows": [
    { "row": 1, "word": "你好", "pinyin": "nǐ hǎo", "meaning": "Xin chào", ... }
  ],
  "errorRows": [
    { "row": 5, "word": "???", "errors": ["word exceeds 50 characters"] }
  ]
}
```

### 6.3 `POST /vocabularies/import/confirm`

Confirms a previously validated import session.

**Request Body:**
```json
{ "importToken": "uuid-string" }
```

**Response (201):**
```json
{ "importedCount": 47 }
```

## 7. Validation Rules

| Rule | Error Message |
|------|---------------|
| `word` is empty | "word is required" |
| `word` length > 50 | "word exceeds 50 characters" |
| `pinyin` is empty | "pinyin is required" |
| `pinyin` length > 100 | "pinyin exceeds 100 characters" |
| `meaning` is empty | "meaning is required" |
| `partOfSpeech` length > 50 | "partOfSpeech exceeds 50 characters" |
| `topic` length > 100 | "topic exceeds 100 characters" |
| Duplicate `word` in DB | "word '{word}' already exists" |
| Duplicate `word` within file | "duplicate word in file at row {N}" |
| Row count > 500 | "File exceeds maximum of 500 rows" |
| File size > 5 MB | "File exceeds maximum size of 5 MB" |

## 8. Security & Permissions

- **Guards:** `JwtAuthGuard`, `RolesGuard`
- **Roles:** `ADMIN`, `TEACHER`
- **Audit Log:** Import action logged with entity type `VOCABULARY`

## 9. Non-Functional Requirements

- Import session data stored in-memory with a 15-minute TTL (auto-cleanup)
- File parsing is streamed to avoid large memory allocation
- Maximum 500 rows per file to prevent abuse
