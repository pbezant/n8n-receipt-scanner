# 🧾 n8n Receipt & Invoice Scanner

Automatically scan receipts and invoices, extract structured data with AI, and log everything to Google Sheets — with separate tabs for receipts (paid) and invoices (unpaid).

Two n8n workflows included:
- **Receipt Scanner** — ongoing automation triggered by Google Drive uploads or Gmail
- **Email Backfill** — one-time batch import of existing receipt/invoice emails

## ✨ Features

- **Two intake paths** — drop files into Google Drive, or forward receipt emails to auto-process
- **AI-powered extraction** — Mistral OCR + Google Gemini for structured data extraction
- **Receipt vs. Invoice routing** — AI classifies each document; receipts go to the Receipts tab, invoices go to the Invoices tab with `payment_status: Unpaid`
- **Receipt-type aware** — gas receipts get `price_per_gallon` + `gallons`, restaurant receipts get `line_items`, hotel receipts get `check_in` / `check_out`, etc.
- **Dynamic columns** — new fields are auto-added as columns. No sheet setup needed.
- **Email backfill workflow** — one-click batch import for all receipt emails from a Gmail label

## 📋 How It Works

```
Path 1 — Google Drive:
  Google Drive Trigger → Download File → AI Pipeline → Route to Sheet Tab

Path 2 — Gmail:
  Gmail Trigger → Label & Archive → Upload to Drive → (Drive Trigger picks it up)

AI Pipeline (shared):
  Extract Text (Mistral OCR)
  → AI Agent (Gemini — structured JSON extraction)
  → Format & Flatten
  → Receipt or Invoice? (If node)
     ├─ Receipt → Sync Columns → Append to "Receipts" tab
     └─ Invoice → Add payment_status:Unpaid → Sync Columns → Append to "Invoices" tab
```

---

## 🚀 Quick Start (5 minutes)

### 1. Prerequisites

| Service | What you need | Free tier? |
|---------|--------------|------------|
| [n8n](https://n8n.io) | Self-hosted or Cloud instance | Yes (self-hosted) |
| [Google Cloud](https://console.cloud.google.com) | Project with Drive, Sheets, Gmail APIs enabled | Yes |
| [Mistral AI](https://console.mistral.ai) | API key | Yes |
| [Google AI Studio](https://aistudio.google.com) | Gemini API key | Yes |

### 2. Create Google Resources

**Google Sheet:**
1. Create a new Google Sheet
2. Rename the first tab to **Receipts**
3. Add a second tab named **Invoices**
4. Copy the spreadsheet ID from the URL: `docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`

**Google Drive folder:**
1. Create a folder (e.g. "Receipts") in Google Drive
2. Copy the folder ID from the URL: `drive.google.com/drive/folders/`**`THIS_PART`**

### 3. Create n8n Credentials

You need **5 credentials** in n8n (Settings → Credentials → Add Credential):

| Credential Type | Used By | Setup |
|----------------|---------|-------|
| **Google Drive OAuth2** | Drive Trigger, Download, Read/Write Headers | OAuth2 Client ID from Google Cloud |
| **Google Sheets OAuth2** | Append to Receipts/Invoices | Same OAuth2 client or a Service Account |
| **Gmail OAuth2** | Gmail Trigger, Move to Label | Same OAuth2 client, enable Gmail API |
| **Mistral Cloud** | OCR extraction | API key from console.mistral.ai |
| **Google Gemini** | AI structured extraction | API key from aistudio.google.com |

> **OAuth2 Redirect URI:** Add `http://YOUR_HOST:5678/rest/oauth2-credential/callback` (self-hosted) or `https://YOUR_INSTANCE.app.n8n.cloud/rest/oauth2-credential/callback` (Cloud) to your Google Cloud OAuth2 client.

### 4. Import the Workflow

1. In n8n: **Workflows → Import from File**
2. Select [`workflows/receipt-scanner.json`](workflows/receipt-scanner.json)

### 5. Configure (find & replace)

After importing, update these values in the workflow nodes:

| Find this placeholder | Replace with | Where |
|----------------------|--------------|-------|
| `YOUR_GOOGLE_SHEET_ID` | Your spreadsheet ID | Read Header Row URL, Write Headers URL, Read Invoice Header Row URL, Write Invoice Headers URL, Append nodes |
| Credential dropdowns | Select your credentials | Every node with a ⚠️ warning |

**Also configure:**
- **Google Drive Trigger** → select your Receipts folder
- **Gmail Trigger** → set email filters (sender, subject, label)
- **Move to Receipts Label** → set your Gmail label ID

### 6. Activate

Toggle the workflow **Active**. Drop a receipt image in your Drive folder to test.

---

## 📦 Repo Structure

```
workflows/
  receipt-scanner.json          # Main workflow — import this into n8n
  backfill-receipt-emails.json  # Optional: batch-import old emails
sample-receipts/
  README.md                     # Supported file formats
LICENSE
README.md                       # This file
```

---

## 📊 Example Output

**Receipts tab:**

| document_type | vendor | date | total | category | price_per_gallon | gallons | fuel_grade |
|--------------|--------|------|-------|----------|-----------------|---------|------------|
| receipt | Shell | 2026-02-14 | 52.80 | Gas | 3.52 | 15.0 | Regular |
| receipt | The Grill | 2026-02-20 | 67.50 | Food | — | — | — |

**Invoices tab:**

| document_type | vendor | date | total | invoice_number | due_date | payment_status |
|--------------|--------|------|-------|---------------|----------|---------------|
| invoice | Acme Corp | 2026-03-01 | 1200.00 | INV-2026-0042 | 2026-03-31 | Unpaid |

Columns are added dynamically — gas receipts auto-create `price_per_gallon`, hotel receipts create `check_in`/`check_out`, etc.

---

## 🔧 Customization

| What | How |
|------|-----|
| Change AI model | Open **Gemini Chat Model** node → select a different model |
| Change OCR | Open **Extract Text** node → pick a different Mistral model |
| Add extraction fields | Edit the prompt in **AI Agent** → add field instructions |
| Different sheet tabs | Update tab names in the HTTP Request URLs and Append nodes |

---

## 📩 Email Backfill (Optional)

Want to import old receipt emails in bulk?

1. Import [`workflows/backfill-receipt-emails.json`](workflows/backfill-receipt-emails.json)
2. Set your Gmail OAuth2 credential
3. Edit the query in the HTTP Request node (e.g. change the date range or label)
4. Run manually — it fetches all matching emails and uploads attachments to Drive

The main Receipt Scanner workflow picks them up from there automatically.

---

## 🤔 FAQ

**Why HTTP requests instead of the Google Sheets node for headers?**
The native Sheets node treats row 1 as immutable headers. The HTTP nodes call the Sheets API directly to read and write row 1, enabling fully dynamic column management.

**Why separate Receipts and Invoices tabs?**
Receipts are proof of payment (already paid). Invoices are bills awaiting payment. Separating them lets you track what's paid vs. unpaid at a glance.

**Can I use OpenAI instead of Gemini?**
Yes — swap the Gemini Chat Model node for an OpenAI Chat Model node. The AI Agent prompt works with any LLM.

---

## 📄 License

MIT — see [LICENSE](LICENSE)
