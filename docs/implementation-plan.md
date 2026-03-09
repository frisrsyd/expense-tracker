# n8n Expense Tracker Pro - Implementation Plan

## Context

A multi-channel AI-powered expense **and income** tracker bot built on n8n. Users interact via WhatsApp, Telegram, n8n Chat Widget, or generic webhook. Each user gets their own Google Spreadsheet with a Dashboard, Income sheet, and monthly expense sheets. The bot uses a configurable AI model for natural language parsing (text, images, PDFs) and auto-detects the user's language.

**Two AI roles:**
- **Intent AI** (Zone C)  classifies user message into structured JSON intent
- **Analysis AI** (Zone D)  reads spreadsheet data and generates a contextual, conversational response per user request

## Architecture Overview

```
ZONE A: Multi-Channel Ingress
         WhatsApp | Telegram | Webhook | Chat Widget
                       |
ZONE B: Profile Resolution
         Master Registry  Route by state  New user onboarding or continue
                       |
ZONE C: Intent Classification
         Detect Media  Select AI Provider  Intent AI Agent  Parse AI Response
         (image path): Get File from Telegram  Build URL  Groq Vision+Intent  Extract  Parse AI Response
                       |
ZONE D: Action Execution
    Write intents: record_expense | record_income | set_budget | split_expense
    Read intents:  check_budget | check_balance | get_report | list_recurring | help
                    Read Sheet Data  Prepare Analysis Context  AI Analysis Agent (Groq)  Extract  reply
                       |
ZONE E: Response Dispatch
         Route reply  channel (WhatsApp | Telegram | Webhook | Chat)
```

## AI Model Configuration

### Intent AI  Model Selection

Set `MODEL_PROVIDER` env var. The `Select Model Provider` Switch routes to the active agent.

| Provider | Cost | Vision | Model | Credential |
|----------|------|--------|-------|------------|
| `gemini` (DEFAULT) | FREE | Yes | Gemini 2.0 Flash | `CuVv1sBp2lTV5WXl` |
| `groq` | FREE | Yes | Llama 3.1 8B Instant | `m5NPNO02GxojmMO8` |
| `ollama` | FREE (self-hosted) | Yes (LLaVA) | Llama 3 | Ollama API |
| `openai` | Paid | Yes | GPT-4o | OpenAI API |

### Intent AI System Prompt (all 4 agents  identical)

Returns ONLY valid JSON. 11 intents:

1. `record_expense`  date, description, amount, currency, category, splitWith, recurring, notes
2. `record_income`  date, description, source (salary/freelance/investment/business/other), amount, currency, recurring
3. `check_budget`  category (optional)
4. `set_budget`  category, amount, period
5. `get_report`  period (week/month/year)
6. `check_balance`  period (this_month/last_month/this_year)
7. `split_expense`  amount, description, category, splitWith
8. `list_recurring`  type
9. `cancel_recurring`  description
10. `merge_report`  profiles, period
11. `help`  topic

Categories: Makanan & Minuman, Transportasi, Tagihan, Hiburan, Belanja, Kesehatan, Langganan, Lain-lain

Income detection phrases: "terima gaji", "dapat freelance", "pendapatan", "income", "masuk uang", "dividen"
Balance detection phrases: "saldo", "net balance", "cashflow", "income vs pengeluaran", "berapa sisa"

### Image OCR  Groq Vision (single-call, Telegram only)

Model: `meta-llama/llama-4-scout-17b-16e-instruct`
Endpoint: `POST https://api.groq.com/openai/v1/chat/completions`
Auth: `Authorization: Bearer $env.GROQ_API_KEY`
System prompt: extract expense JSON directly from receipt image
Result flows into `Parse AI Response` via `Extract Groq Vision Output`

### Analysis AI  Groq HTTP Direct

Model: `llama-3.1-8b-instant`
Same endpoint and auth. Separate per-intent agent nodes (`AI Analysis Agent (Budget)`, `AI Analysis Agent (Report)`).
System prompt: financial analyst  reads raw spreadsheet data, answers user question conversationally in user's detected language, 300 words, emoji + bullet format, no tables.

## Node Inventory (73 nodes)

### Zone A  Multi-Channel Ingress (8 nodes)

| Node | Purpose |
|------|---------|
| WhatsApp Trigger | disabled by default |
| Telegram Trigger | primary trigger |
| Webhook Trigger | `/webhook/expense-tracker` |
| Chat Trigger | n8n embedded chat widget |
| Normalize WhatsApp | unified schema |
| Normalize Telegram | unified schema |
| Normalize Webhook | unified schema |
| Normalize Chat | unified schema |

**Unified schema**  `userId`, `userName`, `messageText`, `mediaUrl`, `mediaType`, `channel`, `rawPayload`

### Zone B  Profile Resolution (16 nodes)

| Node | Purpose |
|------|---------|
| Lookup User Profile | Google Sheets read of Master Registry by userId |
| Check Awaiting Profile | determine user state; getId() helper with 10 candidate field names |
| Route User State | 4-way switch |
| Ask Profile Name | welcome message for new users |
| Save Awaiting State | write "awaitingProfileName" to Master Registry |
| Save Profile Name | extract profileName from user reply |
| Create User Spreadsheet | Sheets API create spreadsheet |
| Setup Dashboard Headers | Code: prepare 8 category rows |
| Write Dashboard Data | HTTP POST: write Dashboard rows |
| **Setup Income Sheet** *(new)* | HTTP POST: batchUpdate to create "Income" sheet |
| **Write Income Headers** *(new)* | HTTP POST: append 9-column header row to Income |
| Setup Month Sheet Headers | HTTP POST: write expense sheet header row |
| Register User in Master | Google Sheets append/update Profiles |
| Send Registration Complete | format confirmation reply |
| Merge User Context | Set node  merge message + profile data |
| (Save Awaiting State  Route Reply to Channel) | profile name waiting flow |

**New user flow:**
1. Unknown user  ask for profile name  save "awaitingProfileName"  reply
2. Next message  create spreadsheet  setup Dashboard + **Income sheet** + month sheet  register  confirm

### Zone C  Intent Classification (14 nodes)

| Node | Purpose |
|------|---------|
| Detect Media Type | Switch: text (output 2) / image (output 0) / pdf (output 1) |
| **Get File from Telegram** *(was Download Media)* | Telegram `getFile` op  fileId from `rawPayload.message.photo[-1].file_id` |
| **Build Telegram File URL** *(was Process with Vision)* | Code: construct `https://api.telegram.org/file/bot{token}/{file_path}`, re-merge user context |
| **Groq Vision + Intent** *(new)* | HTTP POST to Groq with image URL + receipt extraction system prompt  returns JSON intent directly |
| **Extract Groq Vision Output** *(new)* | Code: extract `choices[0].message.content`, normalize to `output` field for Parse AI Response |
| Select Model Provider | Switch on `MODEL_PROVIDER` env var  routes to one of 4 AI agents |
| AI Agent (OpenAI) | Intent classification, disabled |
| AI Agent (Gemini) | Intent classification |
| AI Agent (Ollama) | Intent classification, disabled |
| AI Agent (Groq) | Intent classification, active |
| OpenAI/Gemini/Ollama/Groq Chat Models | sub-nodes |
| Structured Output Parser | sub-node for OpenAI |
| Parse AI Response | provider-agnostic: iterates all 5 sources (4 agents + Extract Groq Vision Output), re-fetches context from `Merge User Context`, merges into single item |

**Image path flow:**
```
Detect Media Type (output 0)
   Get File from Telegram (fileId = photo[-1].file_id)
   Build Telegram File URL (constructs CDN URL, re-merges context)
   Groq Vision + Intent (POST with image_url, returns JSON intent)
   Extract Groq Vision Output (normalize to `output` field)
   Parse AI Response (same node as text path)
```

`photo[photo.length - 1].file_id` gives highest resolution. Change to `photo[0].file_id` for thumbnail.

### Zone D  Action Execution (29 nodes)

#### Write Intents

| Intent | Flow |
|--------|------|
| `record_expense` | Route by Intent  Ensure Month Sheet  Check and Create Month Sheet  (Create New Month Sheet  Add Month Sheet Headers ) Record Expense Row  Read Dashboard After Record  Update Dashboard Spent  Write Dashboard Update  Route Reply |
| `record_income` *(new)* | Route by Intent  **Record Income Row**  Route Reply |
| `set_budget` | Route by Intent  Prepare Budget Update  Set Budget Value  Format Budget Set Reply  Route Reply |
| `split_expense` | Route by Intent  Handle Split Expense  Lookup Split Target Profile  Record Split in Both Sheets  Record Split Rows  Route Reply |

#### Read Intents (AI Analysis)

| Intent | Flow |
|--------|------|
| `check_budget` | Route by Intent  Read Budget Data  **Prepare Budget Analysis**  **AI Analysis Agent (Budget)**  **Extract Budget Analysis**  Route Reply |
| `check_balance` *(new)* | Route by Intent  Read Budget Data  **Prepare Budget Analysis**  **AI Analysis Agent (Budget)**  **Extract Budget Analysis**  Route Reply |
| `get_report` | Route by Intent  Read Month Data for Report  **Prepare Report Analysis**  **AI Analysis Agent (Report)**  **Extract Report Analysis**  Route Reply |
| `help` | Route by Intent  Format Help Reply  Route Reply |
| `list_recurring` / `cancel_recurring` | Route by Intent  Route Reply (fallthrough) |

**New nodes in Zone D:**

- **`Record Income Row`**  HTTP POST to `Income!A1:append`. 9 columns: Date, Description, Source, Amount, Currency, Recurring, Notes, Month (`$now.format('MMM')`), Year (`$now.format('yyyy')`)
- **`Prepare Budget Analysis`**  Code: packages `{ userRequest, intent, profileName, period, spreadsheetData }` for Analysis AI
- **`AI Analysis Agent (Budget)`**  HTTP POST to Groq with analysis system prompt + raw Dashboard data
- **`Extract Budget Analysis`**  Code: extracts `choices[0].message.content`  sets `replyText`; re-merges `rawContext` for routing fields
- **`Prepare Report Analysis`**  same Code pattern as Prepare Budget Analysis
- **`AI Analysis Agent (Report)`**  HTTP POST to Groq with analysis system prompt + raw month sheet data
- **`Extract Report Analysis`**  same extract pattern

**Kept but bypassed** (still in workflow graph, no active outgoing connections):
- `Format Budget Reply`  replaced by AI Analysis chain
- `Generate Report`  replaced by AI Analysis chain

### Route by Intent  10 outputs

| Output | Intent | Next |
|--------|--------|------|
| 0 | `record_expense` | Ensure Month Sheet |
| 1 | `record_income` | Record Income Row |
| 2 | `check_balance` | Read Budget Data |
| 3 | `check_budget` | Read Budget Data |
| 4 | `set_budget` | Prepare Budget Update |
| 5 | `get_report` | Read Month Data for Report |
| 6 | `split_expense` | Handle Split Expense |
| 7 | `help` | Format Help Reply |
| 8 | `list_recurring` / `cancel_recurring` | Route Reply (fallthrough) |
| 9 | catchall | Route Reply (fallthrough) |

### Zone E  Response Dispatch (5 nodes, unchanged)

Route Reply to Channel  Send WhatsApp Reply / Send Telegram Reply / Send Webhook Reply / Send Chat Reply

## Google Sheets Structure

### Master Registry

Sheet: `Profiles`  `userId`, `profileName`, `spreadsheetId`, `createdDate`, `state`, `language`

### Per-User Spreadsheet

**Dashboard:**
- Rows 18: Category | Budget | Spent This Month | Remaining
- Categories: Makanan & Minuman, Transportasi, Tagihan, Hiburan, Belanja, Kesehatan, Langganan, Lain-lain

**Income sheet *(new)*:**
- Columns: Date | Description | Source | Amount | Currency | Recurring | Notes | Month | Year
- `Source` values: salary, freelance, investment, business, other
- `Month`/`Year` columns for fast period filtering

**Recurring sheet:**
- Columns: Description | Amount | Category | Frequency | Next Due | Active

**Month sheets** (e.g. `Mar-2026`, created on demand):
- Columns: Date | Description | Category | Amount | Currency | Split With | Recurring | Notes

## Data Flow: spreadsheetId Propagation

```
1.  Normalize*  { userId, messageText, channel, mediaUrl, rawPayload }
2.  Lookup User Profile  gets spreadsheetId from Master Registry
3.  Check Awaiting Profile  getId() helper (10 candidate fields)  attaches profileData
4.  Route User State  output 0: registered user with spreadsheetId
5.  Merge User Context  { messageText, spreadsheetId, channel, userId, profileName, ... }
6.  Detect Media Type:
    text   Select Model Provider  Intent AI Agent  Parse AI Response
    image  Get File from Telegram  Build Telegram File URL (re-merges ctx)  Groq Vision+Intent  Extract Groq Vision Output  Parse AI Response
7.  Parse AI Response re-fetches $('Merge User Context').first().json  merges with AI output
8.  Route by Intent  10 outputs
9.  Action node(s)  for read intents: Prepare Analysis Context  AI Analysis Agent (Groq HTTP)  Extract  Route Reply
```

## Credentials

| Credential | ID | Purpose |
|-----------|-----|---------|
| Google Sheets OAuth2 | `36FAVfvsRenvGyJc` | Master Registry + per-user spreadsheets |
| Google Gemini API | `CuVv1sBp2lTV5WXl` | Intent AI (Gemini provider) |
| Groq API | `m5NPNO02GxojmMO8` | Intent AI (Groq provider) |
| Telegram Bot API | `ZbU40jBmvjhBVd3d` | Trigger + reply + Get File from Telegram |

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `MASTER_REGISTRY_ID` | Yes | Spreadsheet ID for Profiles sheet |
| `MODEL_PROVIDER` | No (default: `gemini`) | Intent AI provider: gemini/groq/openai/ollama |
| `GROQ_API_KEY` | Yes (Groq + Vision + Analysis) | Raw Groq API key for HTTP Request headers |
| `TELEGRAM_BOT_TOKEN` | For Telegram | Used in Get File from Telegram URL construction |
| `WHATSAPP_PHONE_NUMBER_ID` | For WhatsApp | WhatsApp Cloud API |
| `WHATSAPP_ACCESS_TOKEN` | For WhatsApp | WhatsApp Cloud API |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | Docker | Set to `false` to allow `$env.*` in Code nodes |

**Important:** `GROQ_API_KEY` must be set as a plain env var (not only as n8n credential) because the Groq Vision + Intent and AI Analysis Agent nodes use it via `{{ $env.GROQ_API_KEY }}` in HTTP Header expressions.

## Known Issues & Fixes Applied

### Fix 1  AI Agent system prompts
All 4 agents now have identical structured JSON instruction prompts (were missing or using free-form SKILLS.md text).

### Fix 2  Parse AI Response provider-agnostic
Iterates all 5 sources dynamically: 4 AI agents + `Extract Groq Vision Output`.

### Fix 3  spreadsheetId lost through AI Agent
AI Agent only outputs its own `output` field. Parse AI Response re-fetches `$('Merge User Context').first().json` and merges it back.

### Fix 4  Google Sheets node fails with dynamic spreadsheetId
All dynamic-ID writes use HTTP Request + Sheets API v4 directly (`values/append`, `values/update`, `batchUpdate`).

### Fix 5  Structured Output Parser
Only connected to OpenAI. JSON system prompt is the primary mechanism for all providers.

### Fix 6  Dashboard never updated after recording expense
3-node pipeline: Read Dashboard After Record  Update Dashboard Spent (Code  calculates new totals)  Write Dashboard Update (HTTP PUT  writes exact cell range).

### Fix 7  Record Expense Row wrote junk columns
Switched from `autoMapInputData` to explicit 8-column HTTP Request payload.

### Fix 8  New month sheets created without headers
Added `Add Month Sheet Headers` node after `Create New Month Sheet`.

### Fix 9  Split expenses never written
`Record Split in Both Sheets` outputs 2 items; `Record Split Rows` HTTP Request processes both items.

### Fix 10  executeOnce blocked new user headers
Removed `executeOnce` flag from `Setup Month Sheet Headers`.

### Fix 11  Inconsistent month sheet name
All nodes use `$now.format('MMM-yyyy')` Luxon expression.

### Fix 12  console.log not visible in n8n UI
`console.log` writes to Docker stdout, not the UI output panel. For debugging, use `return [{ json: { debug: ... } }]` temporarily.

### Fix 13  userRow always undefined in Check Awaiting Profile
`getId()` helper function tries 10 candidate field names (`userId`, `user_id`, `id`, `telegramId`, `chatId`, etc.) and coerces to trimmed string before comparison.

## Verification Checklist

1. **Text expense**  "makan siang 45k" via Telegram
   - Month sheet: row with 8 clean columns
   - Dashboard: Makanan & Minuman Spent updated
   - Reply confirms with replyText from AI parse

2. **Image receipt**  send struk/receipt photo via Telegram
   - `Get File from Telegram` resolves file_path
   - `Build Telegram File URL` constructs CDN URL
   - `Groq Vision + Intent` returns JSON with amount + category
   - Normal expense flow continues from Parse AI Response

3. **Record income**  "terima gaji 5jt"
   - `record_income` intent
   - Income sheet gets new row: date, "gaji/salary", 5000000, IDR, month, year
   - Reply confirms income recorded

4. **Check balance**  "saldo bulan ini" or "cashflow"
   - `check_balance` intent
   - Reads Dashboard (expense totals)
   - AI Analysis Agent generates conversational balance summary

5. **Budget check**  "cek anggaran makanan"
   - `check_budget` intent
   - Reads Dashboard via `Read Budget Data`
   - AI Analysis Agent replies with budget breakdown, flags overspend

6. **Monthly report**  "laporan bulan ini"
   - `get_report` intent
   - Reads month sheet via `Read Month Data for Report`
   - AI Analysis Agent replies with top categories, totals, tip

7. **New user registration**
   - Dashboard sheet created with 8 category rows
   - Income sheet created with 9-column header row
   - Month expense sheet created with 8-column header row
   - User registered in Master Registry

8. **Set budget**  "set anggaran makanan 1jt"
   - `set_budget` intent  Prepare Budget Update  Set Budget Value  Format Budget Set Reply  reply

9. **Split expense**  "dinner 200k split with @John"
   - Rows appended to both spreadsheets

## Pending Improvements

- **`check_balance` full implementation**: currently reuses `Read Budget Data` path; ideally should also read the Income sheet to compute actual net balance (income  total expenses). Add `Read Income Data` HTTP GET node + `Merge Balance Data` Code node before `Prepare Budget Analysis`.
- **`list_recurring` / `cancel_recurring`**: currently fall through to Route Reply without reading the Recurring sheet. Add `Read Recurring Sheet` HTTP GET  `Prepare Recurring Analysis`  `AI Analysis Agent` chain.
- **Dashboard summary row**: add a row 1 with Income This Month / Total Expenses / Net Balance, updated after each `record_expense` and `record_income` event.
- **PDF path**: `Build Telegram File URL` handles the image path. PDF from Telegram uses the same `getFile` op; adapt the Groq Vision call for PDF inputs (PDFs need page extraction before vision call).
- **AI Analysis for set_budget + help**: `Format Budget Set Reply` and `Format Help Reply` are still old Code nodes. Wire them through an AI Analysis chain for more natural replies.
