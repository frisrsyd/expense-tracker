# Expense Tracker Pro - Setup and Testing (JSON-Aligned)

## Prerequisites

- n8n instance with permission to use Google Sheets, Telegram, and HTTP nodes
- Google Cloud project with Sheets API and Drive API enabled
- Telegram bot token (if Telegram channel is used)
- One intent model provider configured (`gemini`, `groq`, `openai`, or `ollama`)
- Groq API key if using image intent (`Groq Vision + Intent`) or analysis chain (`AI Agent` with Groq model)

## 1) Import Workflow

1. In n8n, import `Expense Tracker Pro - Multi Channel.json`.
2. Confirm workflow name is `Expense Tracker Pro - Multi Channel`.
3. Confirm node count is `91` after import.

## 2) Create Master Registry Sheet

1. Create a Google Sheet for registry.
2. Create sheet tab `Profiles`.
3. Add headers in row 1:
   - `userId`, `profileName`, `spreadsheetId`, `createdDate`, `state`, `language`

## 3) Configure Environment Variables

Set these in your n8n environment:

```env
MASTER_REGISTRY_ID=your_registry_spreadsheet_id
MODEL_PROVIDER=gemini
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
GROQ_API_KEY=your_groq_api_key
WHATSAPP_PHONE_NUMBER_ID=your_whatsapp_phone_number_id
WHATSAPP_ACCESS_TOKEN=your_whatsapp_access_token
```

Notes:
- `MASTER_REGISTRY_ID`, `MODEL_PROVIDER`, and `TELEGRAM_BOT_TOKEN` are expected by this project and listed in `.env.example`.
- `GROQ_API_KEY` is required for HTTP-based Groq nodes.
- WhatsApp env vars are only needed for WhatsApp reply path.

## 4) Configure Credentials in n8n

Create and attach these credentials to the imported nodes:

- `Google Sheets OAuth2 API`
- `Telegram API`
- `Groq API`
- `Google Gemini (PaLM) API` (if using `MODEL_PROVIDER=gemini`)
- Optional: OpenAI / Ollama credentials if those provider paths are used

## 5) Important Post-import Fix

The workflow currently mixes dynamic and hardcoded registry IDs.

- `Lookup User Profile` reads from `MASTER_REGISTRY_ID` env.
- `Save Awaiting State` and `Register User in Master` still point to a hardcoded spreadsheet ID from the source workspace.

Action required:
1. Open node `Save Awaiting State`.
2. Open node `Register User in Master`.
3. Replace their `documentId` with your registry sheet (`MASTER_REGISTRY_ID` target) so read/write use the same sheet.

## 6) Channel Setup

### Telegram

1. Ensure `Telegram Trigger` credential is valid.
2. Check `Telegram Trigger` additional field `userIds`.
3. Remove or adjust the current filter (`frisrsyd`) if you want all allowed users/chats to trigger.

### Webhook

- Webhook endpoint path is `POST /webhook/expense-tracker`.
- Response is returned by `Send Webhook Reply` node.

### Chat Widget

- `When chat message received` + `Chat` nodes are already wired.
- Enable public/embed according to your n8n deployment policy.

### WhatsApp

- Trigger exists but `WhatsApp Trigger` is disabled in the JSON.
- Reply node (`Send WhatsApp Reply`) is active and uses env vars.
- Enable trigger only after Meta webhook config is complete.

## 7) Activation

1. Save credentials and node parameter updates.
2. Activate workflow.
3. Run the tests below.

## 8) Test Matrix

### A. Onboarding
- Send a first message from a new user.
- Expected:
  - Bot asks profile name.
  - After next message, user spreadsheet is created with:
    - `Dashboard`
    - `Recurring`
    - current `MMM-yyyy` sheet
    - `Income`
  - User row in `Profiles` has `spreadsheetId`.

### B. Record expense (text)
- Example: `Lunch 45k`.
- Expected:
  - `record_expense` intent
  - row added to current month sheet
  - dashboard spent updated
  - reply sent to source channel

### C. Record income
- Example: `terima gaji 5jt`.
- Expected:
  - `record_income` intent
  - row added to `Income` sheet

### D. Check budget / balance / report
- `cek anggaran makanan`
- `saldo bulan ini`
- `laporan bulan ini`

Expected:
- Relevant read nodes execute (`Read Budget Data`, `Read Income Data`, `Read Month Data for Report` depending on intent)
- Combined analysis returns a single conversational reply

### E. Receipt image (Telegram)
- Send receipt photo.
- Expected:
  - `Get File from Telegram` -> `Build Telegram File URL` -> `Groq Vision + Intent`
  - parsed expense enters normal intent route

### F. PDF input (current known limitation)
- Send PDF document.
- Current behavior may fail because `Get File from Telegram` uses `photo.last().file_id`.
- Treat this as a known bug until node logic is updated for `document.file_id`.

## 9) Troubleshooting

### Profile lookup and registration mismatch
- Symptom: user found/not found inconsistently.
- Check that all registry read/write nodes use the same spreadsheet ID.

### No Telegram trigger events
- Check Telegram credential.
- Check `userIds` filter in trigger.
- Confirm workflow is active.

### Analysis reply empty or fallback
- Verify Groq credential and `GROQ_API_KEY` env.
- Inspect `Combine Report Data` output in execution data.

### Webhook returns but no stored data
- Verify `MASTER_REGISTRY_ID` and Google Sheets OAuth scope/access.
- Confirm per-user spreadsheet was created and `spreadsheetId` propagated.

## 10) Recommended Next Hardening Tasks

1. Replace hardcoded registry IDs in `Save Awaiting State` and `Register User in Master`.
2. Update `Get File from Telegram` to handle both `photo` and `document` payloads.
3. Implement true `merge_report` cross-profile aggregation logic.
4. Implement recurring list/cancel actions against `Recurring` sheet.
