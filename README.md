# 🧾 n8n Receipt & Invoice Scanner

Automatically scan receipts and invoices, extract structured data with AI, and log everything to Google Sheets — with separate tabs for receipts (paid) and invoices (unpaid).

Three n8n workflows included:
- **Receipt Scanner** — real-time automation triggered by Google Drive uploads or Gmail
- **Gmail Crawler** — on-demand webhook that searches your inbox for PDF attachments and imports them
- **Email Backfill** — one-time batch import of existing receipt/invoice emails from Drive

## ✨ Features

- **Three intake paths** — drop files into Google Drive, forward receipt emails, or run the crawler to sweep your inbox
- **AI-powered extraction** — Mistral OCR + Google Gemini Flash Lite for structured data extraction
- **Receipt vs. Invoice routing** — AI classifies each document; receipts go to the Receipts tab, invoices go to the Invoices tab with `Payment Status: Unpaid`
- **16 standardized categories** — consistent classification across all documents (Food & Dining, Gas & Fuel, Travel & Lodging, etc.)
- **PDF gating on Gmail path** — only emails with actual PDF attachments are processed; promotional HTML emails are skipped
- **Gmail Crawler** — retroactively import receipt PDFs from your inbox on demand via a webhook POST

## 📋 How It Works

```
Path 1 — Google Drive:
  Drive Trigger → Is Receipt File? → Download File → AI Pipeline

Path 2 — Gmail (real-time):
  Gmail Trigger → Has PDF Attachment? → Label Email → Upload to Drive → AI Pipeline

Path 3 — Gmail Crawler (on-demand):
  POST /webhook/crawl-gmail-receipts
  → Build Query (filename:pdf + invoice/receipt keywords, skips already-labeled)
  → Search Gmail → Fetch PDFs → Upload to Drive → Label Email
  (Drive Trigger picks up uploaded files → AI Pipeline)

AI Pipeline (shared):
  Extract Text (Mistral OCR)
  → AI Agent (Gemini Flash Lite — structured JSON extraction)
  → Format & Validate (category enforcement, items array normalization)
  → Receipt or Invoice?
     ├─ Receipt → Append to "Receipts" tab
     └─ Invoice → Add Payment Status: Unpaid → Append to "Invoices" tab
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
2. Rename the first tab to **Receipts** with headers: `Date · Vendor · Category · Total · Subtotal · Tax · Tip · Payment Method · Description · Items · Address · Receipt #`
3. Add a second tab named **Invoices** with headers: `Date · Vendor · Category · Total · Subtotal · Tax · Due Date · Invoice # · Payment Status · Payment Method · Description · Items · Billed To · Address`
4. Copy the spreadsheet ID from the URL: `docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`

**Google Drive folder:**
1. Create a folder (e.g. "Receipts") in Google Drive
2. Copy the folder ID from the URL: `drive.google.com/drive/folders/`**`THIS_PART`**

**Gmail label:**
1. Create a label (e.g. "Receipts") in Gmail
2. Find the label ID — it looks like `Label_3540768834604209583` — visible in Gmail settings or by inspecting the API

### 3. Create n8n Credentials

You need **5 credentials** in n8n (Settings → Credentials → Add Credential):

| Credential Type | Used By | Setup |
|----------------|---------|-------|
| **Google Drive OAuth2** | Drive Trigger, Download, Upload | OAuth2 Client ID from Google Cloud |
| **Google Sheets OAuth2** | Append to Receipts/Invoices | Same OAuth2 client or a Service Account |
| **Gmail OAuth2** | Gmail Trigger, Label, Crawler | Same OAuth2 client, enable Gmail API |
| **Mistral Cloud** | OCR extraction | API key from console.mistral.ai |
| **Google Gemini** | AI structured extraction | API key from aistudio.google.com |

> **OAuth2 Redirect URI:** Add `http://YOUR_HOST:5678/rest/oauth2-credential/callback` (self-hosted) or `https://YOUR_INSTANCE.app.n8n.cloud/rest/oauth2-credential/callback` (Cloud) to your Google Cloud OAuth2 client.

### 4. Import the Workflows

1. In n8n: **Workflows → Import from File**
2. Import [`workflows/receipt-scanner.json`](workflows/receipt-scanner.json) — the main ongoing scanner
3. Optionally import a crawler: [`workflows/gmail-invoice-crawler.json`](workflows/gmail-invoice-crawler.json) or [`workflows/gmail-receipt-crawler.json`](workflows/gmail-receipt-crawler.json)

### 5. Configure

After importing, update these values in the workflow nodes:

| Node | What to set |
|------|-------------|
| **Google Drive Trigger** | Select your Receipts folder |
| **Upload Attachment to Drive** | Set the Receipts folder ID |
| **Append to Receipts / Invoices** | Select your spreadsheet and tab |
| **Move to Receipts Label** | Set your Gmail label ID |
| All credential dropdowns | Select the credentials you created in step 3 |

### 6. Activate

Toggle the workflow **Active**. Drop a receipt PDF in your Drive folder, or forward a receipt email, to test.

---

## 📦 Repo Structure

```
workflows/
  receipt-scanner.json          # Main workflow — real-time Drive + Gmail triggers
  gmail-invoice-crawler.json    # Crawler v1: direct Gmail API, filename:pdf filter
  gmail-receipt-crawler.json    # Crawler v2: n8n-native nodes, SplitInBatches loop
  backfill-receipt-emails.json  # One-time batch import from Drive
sample-receipts/
  README.md                     # Supported file formats
LICENSE
README.md                       # This file
```

---

## 📊 Example Output

**Receipts tab:**

| Date | Vendor | Category | Total | Tax | Payment Method | Description |
|------|--------|----------|-------|-----|----------------|-------------|
| 2026-02-14 | Shell | Gas & Fuel | 52.80 | 0.00 | Visa ending 4321 | 15.2 gal Regular @ $3.47/gal, Pump 4 |
| 2026-02-20 | The Grill House | Food & Dining | 67.50 | 5.40 | Visa ending 4321 | Dinner for 2 — burger, caesar salad, 2 iced teas |

**Invoices tab:**

| Date | Vendor | Category | Total | Invoice # | Due Date | Payment Status |
|------|--------|----------|-------|-----------|----------|---------------|
| 2026-03-01 | Acme Corp | Office Supplies | 1200.00 | INV-2026-0042 | 2026-03-31 | Unpaid |

All 16 supported categories: Food & Dining · Groceries · Gas & Fuel · Office Supplies · Travel & Lodging · Utilities · Medical & Health · Entertainment · Shopping · Transportation · Subscriptions · Home & Garden · Auto & Maintenance · Personal Care · Education · Other

---

## 🔧 Customization

| What | How |
|------|-----|
| Change AI model | Open **Google Gemini Chat Model1** node → select a different model |
| Change OCR | Open **Extract text1** node → pick a different Mistral model |
| Add extraction fields | Edit the prompt in **AI Agent1** → add field instructions + update **Format Receipt Data1** FIELD_MAP |
| Different sheet tabs | Update tab names in the **Append to Receipts** / **Append to Invoices** nodes |
| Change Gmail filter | Edit `filters` on the **Gmail Trigger** node (e.g. add sender or subject filters) |

---

## 🔍 Gmail Crawler (On-Demand Import)

Two crawler variants are included — both do the same job but use different approaches:

| File | Approach | Notes |
|------|----------|-------|
| `gmail-invoice-crawler.json` | Direct Gmail REST API calls | Confirmed working; `filename:pdf` filter prevents false positives |
| `gmail-receipt-crawler.json` | n8n-native Gmail + SplitInBatches | Cleaner node graph; loops through emails one at a time |

**How to run:**

```bash
curl -X POST http://localhost:5678/webhook/crawl-gmail-receipts
```

The crawler:
1. Builds a Gmail query: `has:attachment (filename:pdf OR filename:PDF) -label:Receipts` + invoice/receipt keywords
2. Filters to emails not yet labeled `Receipts` (no duplicates)
3. Downloads PDF attachments and uploads them to the Receipts Drive folder
4. Applies the `Receipts` Gmail label to processed emails
5. The Drive Trigger on the Receipt Scanner picks up the new files and runs them through the AI pipeline

You can customize the date range and keywords in the **Build Search Query** node.

---

## 📩 Email Backfill (Optional)

Want to import old receipt files from Drive in bulk?

1. Import [`workflows/backfill-receipt-emails.json`](workflows/backfill-receipt-emails.json)
2. Set your Google Drive and OAuth2 credentials
3. Run manually — it iterates over all files in the Receipts folder and processes each through the AI pipeline

---

## 🤔 FAQ

**Why separate Receipts and Invoices tabs?**
Receipts are proof of payment (already paid). Invoices are bills awaiting payment. Separating them lets you track what's paid vs. unpaid at a glance.

**Can I use OpenAI instead of Gemini?**
Yes — swap the **Google Gemini Chat Model1** node for an OpenAI Chat Model node. The AI Agent prompt works with any LLM.

**Why does the Gmail path upload to Drive rather than going straight to OCR?**
It normalizes both intake paths through a single pipeline. Drive is used as the canonical source — the Drive Trigger fetches the file and feeds it to Mistral OCR regardless of where the file came from.

---

## 📄 License

MIT — see [LICENSE](LICENSE)
