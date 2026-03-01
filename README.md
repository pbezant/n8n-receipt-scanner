# 🧾 n8n Receipt Scanner → Google Sheets

An n8n workflow that automatically scans receipts uploaded to Google Drive, extracts all data using AI (Mistral OCR + Google Gemini), and logs everything to a Google Sheet with **dynamically managed columns** — new fields like `price_per_gallon`, `line_items`, or `room_rate` are added as new columns automatically the first time they're seen.

## ✨ Features

- **Fully automatic** — drop a receipt image into a Google Drive folder and it processes within a minute
- **AI-powered extraction** — Mistral AI reads the receipt text; Gemini analyzes and structures it
- **Receipt-type aware** — gas receipts get `price_per_gallon` + `gallons`, restaurant receipts get `line_items`, hotel receipts get `check_in` / `check_out` / `room_rate`, etc.
- **Dynamic columns** — Google Sheet headers are auto-managed; new fields are appended as new columns, existing fields map to their existing column
- **One row per receipt** — clean, consistent spreadsheet layout

## 📋 Workflow Overview

```
Google Drive Trigger
  → Download File (Google Drive)
  → Extract Text (Mistral AI — OCR)
  → AI Agent (Gemini — structured extraction)
  → Format Receipt Data (Code — flatten to key/value)
  → Read Header Row (Sheets API — get current columns)
  → Sync Columns (Code — merge new fields into header list)
  → Write Headers (Sheets API — update row 1)
  → Prepare Row Data (Code — strip internal fields)
  → Append to Sheet (Google Sheets — add row)
```

---

## 🛠️ Prerequisites

- A running **n8n** instance (self-hosted or n8n Cloud)
- A **Google Cloud project** with Drive API and Sheets API enabled
- A **Mistral AI** account (free tier works) — [console.mistral.ai](https://console.mistral.ai)
- A **Google AI Studio** API key for Gemini — [aistudio.google.com](https://aistudio.google.com)

---

## 🚀 Setup Guide

### Step 1 — Google Cloud Setup

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create (or select) a project
2. Enable these two APIs:
   - **Google Drive API**
   - **Google Sheets API**

---

### Step 2 — Create Credentials in n8n

You need **four** credentials configured in n8n before importing the workflow.

#### 2a. Google Drive OAuth2 API *(used by 4 nodes)*

> Used for: Drive Trigger, Download File, Read Header Row, Write Headers

1. In your Google Cloud project: **APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID**
2. Application type: **Web application**
3. Add your n8n redirect URI as an Authorized Redirect URI:
   - Self-hosted: `http://YOUR_N8N_HOST:5678/rest/oauth2-credential/callback`
   - n8n Cloud: `https://YOUR_INSTANCE.app.n8n.cloud/rest/oauth2-credential/callback`
4. Copy the **Client ID** and **Client Secret**
5. In n8n: **Credentials → New → Google Drive OAuth2 API**
   - Paste Client ID and Client Secret
   - Click **Connect my account** and sign in with Google
   - Name it: `Your Google Drive OAuth2`

#### 2b. Google Gemini (PaLM) API *(used by Gemini Chat Model)*

1. Go to [aistudio.google.com/apikey](https://aistudio.google.com/apikey) and create an API key
2. In n8n: **Credentials → New → Google Gemini(PaLM) Api**
   - Paste your API key
   - Name it: `Your Google Gemini API Key`

#### 2c. Mistral Cloud API *(used by Extract Text)*

1. Go to [console.mistral.ai](https://console.mistral.ai), create an account, generate an API key
2. In n8n: **Credentials → New → Mistral Cloud API**
   - Paste your API key
   - Name it: `Your Mistral Cloud Account`

#### 2d. Google Sheets Service Account *(used by Append to Sheet)*

> A Service Account bypasses OAuth2 redirect requirements for writing to Sheets.

1. In Google Cloud: **APIs & Services → Credentials → Create Credentials → Service Account**
2. Give it a name (e.g. `n8n-receipt-writer`), click **Done**
3. Click the service account → **Keys tab → Add Key → JSON** — download the file
4. In n8n: **Credentials → New → Google API (Service Account)**
   - Upload the JSON key file
   - Name it: `Your Google Service Account`
5. **Share your Google Sheet** with the service account email (found in the JSON file under `client_email`) — give it **Editor** access

---

### Step 3 — Create Your Google Drive Folder and Sheet

**Google Drive folder:**
1. Create a folder named **Receipts** (or any name you prefer) in Google Drive
2. Copy the folder ID from the URL: `drive.google.com/drive/folders/`**`THIS_PART`**

**Google Sheet:**
1. Create a new blank Google Sheet (no headers needed — the workflow manages them)
2. Copy the spreadsheet ID from the URL: `docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`
3. Share the sheet with your service account email (see Step 2d above)

---

### Step 4 — Import the Workflow

1. In n8n: **Workflows → Import from File**
2. Select `workflow.json` from this repo
3. The workflow opens with all nodes visible

---

### Step 5 — Configure the Workflow

Open each of these nodes and update them:

#### Google Drive Trigger
- Click the **Folder to Watch** field → select your Receipts folder (or paste the folder ID)
- Set **Poll interval** to your preference (default: every minute)
- Select your **Google Drive OAuth2** credential

#### Download File
- Select your **Google Drive OAuth2** credential

#### Extract Text (Mistral)
- Select your **Mistral Cloud Account** credential

#### Gemini Chat Model
- Select your **Google Gemini API Key** credential

#### Read Header Row
- In the URL, replace `YOUR_SPREADSHEET_ID` with your actual spreadsheet ID

#### Write Headers
- In the URL, replace `YOUR_SPREADSHEET_ID` with your actual spreadsheet ID

#### Append to Sheet
- Click the **Document** field and paste your spreadsheet URL (or replace `YOUR_SPREADSHEET_ID` in the existing URL)
- Select your **Google Service Account** credential

---

### Step 6 — Activate

Toggle the workflow **Active** in the top-right corner. Upload a receipt image to your Google Drive Receipts folder to test it.

---

## 📊 Example Output

For a gas receipt, the sheet will get columns like:

| vendor | date | total | category | price_per_gallon | gallons | fuel_grade | payment_method |
|--------|------|-------|----------|-----------------|---------|------------|----------------|
| Shell | 2026-02-14 | 52.80 | Gas | 3.52 | 15.0 | Regular | Credit |

For a restaurant receipt:

| vendor | date | total | subtotal | tax | tip | line_items | server |
|--------|------|-------|----------|-----|-----|------------|--------|
| The Grill | 2026-02-20 | 67.50 | 55.00 | 4.95 | 7.55 | Burger x1 @ $18 = $18 \| Fries x2 @ $5 = $10 | Sarah |

New columns are added automatically — you never need to set up the sheet manually.

---

## 🔧 Customization

**Change the AI model:** Open the **Gemini Chat Model** node and select a different model from the dropdown (e.g. `gemini-pro`, `gemini-flash`).

**Change the OCR model:** The **Extract Text** node uses Mistral's default OCR model. You can swap it for any Mistral vision model.

**Add more extraction fields:** Edit the prompt in the **AI Agent** node to instruct the model to extract additional fields for specific receipt types.

**Sheet name:** If your sheet tab isn't named `Sheet1`, update the URL in Read Header Row, Write Headers, and the Sheet Name in Append to Sheet.

---

## 🤔 Why HTTP requests for Read/Write Headers?

The native Google Sheets n8n node always treats row 1 as column headers — you can't read or overwrite the headers themselves through it. The two HTTP nodes (`Read Header Row` and `Write Headers`) call the Sheets REST API directly using the same Google Drive OAuth2 credential, bypassing this limitation to enable fully dynamic column management.

---

## 📄 License

MIT — see [LICENSE](LICENSE)
