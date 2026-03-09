# n8n Expense Tracker Pro - Implementation Plan

## Context

A comprehensive n8n workflow (`Expense Tracker Pro - Multi Channel.json`) that serves as a multi-channel AI-powered expense tracker bot. Users interact via WhatsApp, Telegram, n8n Chat Widget, or generic webhook. Each user gets their own Google Spreadsheet with a dashboard and monthly sheets. The bot uses a configurable AI model for natural language parsing (text, images, PDFs) and auto-detects the user's language.

## Architecture Overview

```
ZONE A: Multi-Channel Ingress (4 triggers -> 4 normalizers)
         WhatsApp | Telegram | Webhook | Chat Widget
                       |
ZONE B: Profile Resolution
         Master Registry lookup -> Route by state -> New user onboarding or continue
                       |
ZONE C: Intent Classification
         Media detection -> Select AI Provider -> AI Agent -> Parse AI Response
                       |
ZONE D: Action Execution
         Route by Intent -> Record | Budget | Report | Split | Recurring | Help
                       |
ZONE E: Response Dispatch
         Route reply back to originating channel
```

## AI Model Configuration

Set `MODEL_PROVIDER` environment variable in n8n:

| Provider | Cost | Vision | Model | Credential |
|----------|------|--------|-------|------------|
| `gemini` (DEFAULT) | FREE tier | Yes | Gemini 2.0 Flash | Google Gemini API |
| `groq` | FREE tier | Yes | Llama 3.1 8B Instant | Groq API |
| `ollama` | FREE (self-hosted) | Yes (LLaVA) | Llama 3 | Ollama API |
| `openai` | Paid | Yes | GPT-4o | OpenAI API |

All 4 AI Agent nodes share the same structured JSON system prompt that instructs the model to:
1. Detect the user's language and respond in it
2. Return ONLY valid JSON (no markdown, no backticks)
3. Use one of 9 intent schemas (record_expense, check_budget, set_budget, get_report, split_expense, list_recurring, cancel_recurring, merge_report, help)
4. Parse amount suffixes: k=x1000, rb=x1000, jt=x1000000
5. Auto-detect category from context
6. Always include a `replyText` field with human-friendly confirmation

## Node Inventory (62 nodes)

### Zone A - Multi-Channel Ingress (8 nodes)

| Node | Type | Purpose |
|------|------|---------|
| WhatsApp Trigger | `n8n-nodes-base.whatsAppTrigger` | Receive WhatsApp messages (disabled by default) |
| Telegram Trigger | `n8n-nodes-base.telegramTrigger` | Receive Telegram messages (primary trigger) |
| Webhook Trigger | `n8n-nodes-base.webhook` | Generic HTTP POST at `/webhook/expense-tracker` |
| Chat Trigger | `@n8n/n8n-nodes-langchain.chatTrigger` | n8n built-in embeddable chat widget |
| Normalize WhatsApp | `n8n-nodes-base.code` | Extract unified schema from WhatsApp payload |
| Normalize Telegram | `n8n-nodes-base.code` | Extract unified schema from Telegram payload |
| Normalize Webhook | `n8n-nodes-base.code` | Extract unified schema from webhook body |
| Normalize Chat | `n8n-nodes-base.code` | Extract unified schema from chat widget payload |

**Unified schema output:**
```json
{
  "userId": "string",
  "userName": "string",
  "messageText": "string",
  "mediaUrl": "string|null",
  "mediaType": "text|image|pdf",
  "channel": "whatsapp|telegram|webhook|chat",
  "rawPayload": {}
}
```

### Zone B - Profile Resolution (13 nodes)

| Node | Type | Purpose |
|------|------|---------|
| Lookup User Profile | `n8n-nodes-base.googleSheets` | Read Master Registry by userId |
| Check Awaiting Profile | `n8n-nodes-base.code` | Determine user state (new/awaiting/active/creating) |
| Route User State | `n8n-nodes-base.switch` | 4-way: has spreadsheetId / isNew / isAwaiting / isCreating |
| Ask Profile Name | `n8n-nodes-base.code` | Build welcome message for new users |
| Save Awaiting State | `n8n-nodes-base.googleSheets` | Write "awaitingProfileName" state to Master Registry |
| Save Profile Name | `n8n-nodes-base.code` | Extract profile name from user's reply |
| Create User Spreadsheet | `n8n-nodes-base.httpRequest` | Google Sheets API v4 to create new spreadsheet |
| Setup Dashboard Headers | `n8n-nodes-base.code` | Prepare 8 category rows for Dashboard |
| Write Dashboard Data | `n8n-nodes-base.googleSheets` | Write categories to Dashboard sheet |
| Setup Month Sheet Headers | `n8n-nodes-base.httpRequest` | Add column headers to first month sheet |
| Register User in Master | `n8n-nodes-base.googleSheets` | Update Master Registry with spreadsheetId, state=active |
| Send Registration Complete | `n8n-nodes-base.code` | Format success message |
| Merge User Context | `n8n-nodes-base.set` | Combine message + profile data for registered users |

**New user flow:**
1. First message -> user not found -> ask for profile name -> save "awaitingProfileName" state -> reply via channel
2. Second message -> state = awaitingProfileName -> use messageText as profile name -> create spreadsheet -> setup Dashboard + month sheet -> register in Master -> confirm via channel

**Data flow for registered users:**
`Lookup User Profile` -> `Check Awaiting Profile` (adds profileData with spreadsheetId) -> `Route User State` (output 0: has spreadsheetId) -> `Merge User Context` (merges messageText + spreadsheetId + channel + userId + profileName) -> Zone C

### Zone C - Intent Classification (14 nodes)

| Node | Type | Purpose |
|------|------|---------|
| Detect Media Type | `n8n-nodes-base.switch` | Route: text (default) / image / pdf |
| Download Media | `n8n-nodes-base.httpRequest` | Download image/PDF from WhatsApp/Telegram |
| Process with Vision | `n8n-nodes-base.code` | Prepare media for AI vision processing |
| Select Model Provider | `n8n-nodes-base.switch` | Route to AI Agent based on MODEL_PROVIDER env var |
| AI Agent (OpenAI) | `@n8n/n8n-nodes-langchain.agent` | GPT-4o with structured JSON system prompt |
| AI Agent (Gemini) | `@n8n/n8n-nodes-langchain.agent` | Gemini 2.0 Flash with structured JSON system prompt |
| AI Agent (Ollama) | `@n8n/n8n-nodes-langchain.agent` | Local Ollama with structured JSON system prompt |
| AI Agent (Groq) | `@n8n/n8n-nodes-langchain.agent` | Llama 3.1 8B Instant with structured JSON system prompt |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | GPT-4o sub-node (disabled) |
| Gemini Chat Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Gemini 2.0 Flash sub-node |
| Ollama Chat Model | `@n8n/n8n-nodes-langchain.lmChatOllama` | Llama 3 sub-node (disabled) |
| Groq Chat Model | `@n8n/n8n-nodes-langchain.lmChatGroq` | Llama 3.1 8B Instant sub-node |
| Structured Output Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | Connected to AI Agent (OpenAI) |
| Parse AI Response | `n8n-nodes-base.code` | Provider-agnostic parser that merges AI output + user context |

**Parse AI Response** is the critical bridge between AI and action execution:
- Tries all 4 AI Agent sources dynamically (Groq, OpenAI, Gemini, Ollama)
- Recovers `spreadsheetId`, `channel`, `userId`, `profileName` from `Merge User Context`
- Extracts JSON from AI output (handles markdown-wrapped JSON, plain text fallback)
- Outputs merged object with both context fields AND parsed intent fields

### Zone D - Action Execution (21 nodes)

| Intent | Flow |
|--------|------|
| `record_expense` | Route by Intent -> Ensure Month Sheet -> Check/Create Month Sheet -> (Create New Month Sheet -> Add Month Sheet Headers ->) Record Expense Row -> Read Dashboard After Record -> Update Dashboard Spent -> Write Dashboard Update -> Route Reply to Channel |
| `check_budget` | Route by Intent -> Read Budget Data -> Format Budget Reply -> Route Reply to Channel |
| `set_budget` | Route by Intent -> Set Budget Value -> Format Budget Set Reply -> Route Reply to Channel |
| `get_report` | Route by Intent -> Read Month Data for Report -> Generate Report -> Route Reply to Channel |
| `split_expense` | Route by Intent -> Handle Split Expense -> Lookup Split Target Profile -> Record Split in Both Sheets -> Record Split Rows -> Route Reply to Channel |
| `help` | Route by Intent -> Format Help Reply -> Route Reply to Channel |

**Key nodes:**
- `Route by Intent`: Switch node routing on `$json.parsedIntent` (8 outputs)
- `Ensure Month Sheet`: HTTP GET to check if month sheet exists in user's spreadsheet
- `Check and Create Month Sheet`: Code node that determines month name (e.g., "Mar-2026") and checks existence
- `Create New Month Sheet`: batchUpdate API to create sheet, followed by `Add Month Sheet Headers` to write header row
- `Record Expense Row`: HTTP Request that appends row via Google Sheets API with 8 columns (Date, Description, Category, Amount, Currency, Split With, Recurring, Notes)
- `Read Dashboard After Record` -> `Update Dashboard Spent` -> `Write Dashboard Update`: 3-node pipeline — HTTP GET reads Dashboard, Code calculates row number + new "Spent This Month", HTTP PUT writes to the exact cell range
- `Prepare Budget Update` -> `Set Budget Value`: Code node maps category to row number, HTTP PUT updates the Budget column
- `Record Split Rows`: HTTP Request that appends via Sheets API, processes 2 items (one per spreadsheet) to record split entries in both users' sheets
- All dynamic-spreadsheetId write operations use HTTP Request with Google Sheets API (not the n8n Google Sheets node, which fails with dynamic IDs + defineBelow mapping)

### Zone E - Response Dispatch (5 nodes)

| Node | Type | Purpose |
|------|------|---------|
| Route Reply to Channel | `n8n-nodes-base.switch` | Switch on `$json.channel` |
| Send WhatsApp Reply | `n8n-nodes-base.httpRequest` | WhatsApp Cloud API message |
| Send Telegram Reply | `n8n-nodes-base.telegram` | Telegram sendMessage |
| Send Webhook Reply | `n8n-nodes-base.respondToWebhook` | HTTP JSON response |
| Send Chat Reply | `n8n-nodes-base.code` | Format for n8n chat widget |

## Google Sheets Structure

### Master Registry (shared spreadsheet)

**Sheet name:** `Profiles`

| Column | Type | Description |
|--------|------|-------------|
| userId | string | Phone number, chat ID, or session ID |
| profileName | string | User-chosen display name |
| spreadsheetId | string | Their personal tracker spreadsheet ID |
| createdDate | ISO date | When profile was created |
| state | string | "active" or "awaitingProfileName" |
| language | string | Auto-detected language code |

### Per-User Spreadsheet

**Dashboard sheet:**

| Category | Budget | Spent This Month | Remaining |
|----------|--------|-------------------|-----------|
| Makanan & Minuman | 0 | 0 | 0 |
| Transportasi | 0 | 0 | 0 |
| Tagihan | 0 | 0 | 0 |
| Hiburan | 0 | 0 | 0 |
| Belanja | 0 | 0 | 0 |
| Kesehatan | 0 | 0 | 0 |
| Langganan | 0 | 0 | 0 |
| Lain-lain | 0 | 0 | 0 |

**Recurring sheet:**

| Description | Amount | Category | Frequency | Next Due | Active |
|-------------|--------|----------|-----------|----------|--------|

**Month sheets (e.g., "Mar-2026", created on demand):**

| Date | Description | Category | Amount | Currency | Split With | Recurring | Notes |
|------|-------------|----------|--------|----------|------------|-----------|-------|

## Data Flow: spreadsheetId Propagation

This is the critical data path that enables writing to the correct user's spreadsheet:

```
1. Normalize* -> { userId, messageText, channel, ... }
2. Lookup User Profile -> reads Master Registry by userId -> gets spreadsheetId
3. Check Awaiting Profile -> attaches profileData { spreadsheetId, profileName, state }
4. Route User State -> output 0 (registered user with spreadsheetId)
5. Merge User Context -> combines: { messageText, spreadsheetId, channel, userId, profileName }
6. Detect Media Type -> passes through all fields
7. Select Model Provider -> passes through all fields
8. AI Agent (*) -> processes messageText, outputs { output: "JSON string" }
   *** AI Agent ONLY outputs its own fields, context is lost here ***
9. Parse AI Response -> RE-FETCHES context from $('Merge User Context').first().json
   -> merges: { ...originalContext, ...parsedAIOutput, parsedIntent }
10. Route by Intent -> routes on parsedIntent
11. For record_expense:
    Ensure Month Sheet -> Check and Create Month Sheet -> (Create New Month Sheet -> Add Month Sheet Headers ->) Record Expense Row
    -> Read Dashboard After Record -> Update Dashboard Spent (calculates new totals) -> Write Dashboard Update
    -> Route Reply to Channel
12. For split_expense:
    Handle Split Expense -> Lookup Split Target Profile -> Record Split in Both Sheets (outputs 2 items)
    -> Record Split Rows (appends to BOTH spreadsheets) -> Route Reply to Channel
```

Step 9 is the critical context recovery fix. Steps 11-12 show the complete write pipeline including Dashboard updates and split recording.

## Credentials

| Credential | ID | Required | Purpose |
|-----------|-----|----------|---------|
| Google Sheets OAuth2 | `36FAVfvsRenvGyJc` | Always | Master Registry + per-user spreadsheets |
| Google Gemini API | `CuVv1sBp2lTV5WXl` | If MODEL_PROVIDER=gemini | AI processing (FREE) |
| Groq API | `m5NPNO02GxojmMO8` | If MODEL_PROVIDER=groq | AI processing (FREE tier) |
| Ollama API | - | If MODEL_PROVIDER=ollama | AI processing (self-hosted) |
| OpenAI API | - | If MODEL_PROVIDER=openai | AI processing (paid) |
| Telegram Bot API | `ZbU40jBmvjhBVd3d` | If using Telegram | Trigger + reply |
| WhatsApp Business Cloud API | - | If using WhatsApp | Trigger + reply |

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `MASTER_REGISTRY_ID` | Yes | - | Google Spreadsheet ID for Profiles registry |
| `MODEL_PROVIDER` | No | `gemini` | AI provider: gemini/openai/ollama/groq |
| `WHATSAPP_PHONE_NUMBER_ID` | For WhatsApp | - | WhatsApp Business phone number ID |
| `WHATSAPP_ACCESS_TOKEN` | For WhatsApp | - | WhatsApp Cloud API access token |
| `TELEGRAM_BOT_TOKEN` | For Telegram | - | Telegram bot token |

## Known Issues & Fixes Applied

### Fix 1: AI Agent system prompts (all 4 providers)
**Problem:** Only Groq had a system prompt, and it was a copy of SKILLS.md (free-form Indonesian text) instead of structured JSON instructions. The other 3 agents had no system prompt at all.
**Fix:** All 4 AI Agent nodes now have identical structured JSON instruction prompts that force the AI to return valid JSON with intent, amount, category, date, description, and replyText fields.

### Fix 2: Parse AI Response provider-agnostic
**Problem:** Was hardcoded to `$('AI Agent (Groq)').all()`, failing if any other provider was used.
**Fix:** Now iterates through all 4 agent sources dynamically, using whichever one actually produced output.

### Fix 3: spreadsheetId preservation through AI Agent
**Problem:** AI Agent nodes only output their `output` field, dropping all context data (spreadsheetId, channel, userId, etc.) that was passed in from Merge User Context.
**Fix:** Parse AI Response explicitly re-fetches context from `$('Merge User Context').first().json` and merges it with the parsed AI output, ensuring downstream nodes have access to `spreadsheetId`.

### Fix 4: Downstream Google Sheets nodes
**Problem:** All action nodes use `={{ $json.spreadsheetId }}` which was empty due to Fix 3's underlying issue.
**Status:** Resolved by Fix 3 — spreadsheetId now propagates correctly through Parse AI Response.

### Fix 5: Structured Output Parser
**Note:** Currently only connected to AI Agent (OpenAI). The structured JSON system prompt serves as the primary output formatting mechanism for all providers, making the parser a secondary safeguard for OpenAI only.

### Fix 6: Dashboard never updated after recording expense (CRITICAL)
**Problem:** `Update Dashboard Spent` was a Code node that only formatted replyText. It never wrote to the Dashboard sheet, so "Spent This Month" was always 0.
**Fix:** Replaced with a 3-node pipeline:
1. `Read Dashboard After Record` — reads current Dashboard data
2. `Update Dashboard Spent` — calculates new "Spent This Month" (current + expense amount) and "Remaining" (budget - new spent)
3. `Write Dashboard Update` — writes updated values back to Dashboard using Category as matching column

### Fix 7: Record Expense Row wrote junk columns + Google Sheets node fails with dynamic IDs (CRITICAL)
**Problem:** Used `autoMapInputData` which dumped ALL 20+ fields into the month sheet. Additionally, the n8n Google Sheets node with `defineBelow` mapping cannot resolve dynamic `spreadsheetId` expressions at design time, causing "Could not get parameter" errors.
**Fix:** Converted ALL write/update operations targeting dynamic user spreadsheets from Google Sheets nodes to HTTP Request nodes using the Google Sheets API v4 directly (`values/append` for writes, `values/update` for updates). This bypasses the n8n node's schema validation. Only nodes targeting the Master Registry (with hardcoded/cached IDs) remain as Google Sheets nodes.

### Fix 8: New month sheets created without headers (HIGH)
**Problem:** `Create New Month Sheet` only created an empty sheet via batchUpdate. No header row was added.
**Fix:** Added `Add Month Sheet Headers` node between Create New Month Sheet and Record Expense Row. Uses Google Sheets API values/append to write the 8 column headers.

### Fix 9: Split expenses never recorded (HIGH)
**Problem:** `Record Split in Both Sheets` only prepared data but never appended rows to any spreadsheet.
**Fix:** Rewrote to output 2 items (one for self, one for target) and added `Record Split Rows` Google Sheets append node that processes both items, writing to the correct spreadsheet for each.

### Fix 10: executeOnce blocked new user headers (MEDIUM)
**Problem:** `Setup Month Sheet Headers` had `executeOnce: true`, so only the first user who registered got month sheet headers.
**Fix:** Removed `executeOnce` flag.

### Fix 11: Inconsistent month sheet name format (LOW)
**Problem:** `Read Month Data for Report` used `toLocaleDateString('en-US', ...)` which is locale-dependent, while other nodes used explicit month arrays.
**Fix:** Changed to `$now.format('MMM-yyyy')` Luxon expression for consistency.

## Verification Checklist

After importing the updated workflow:

1. **Record expense**: Send "makan siang 45k" via Telegram
   - Month sheet should get a row with exactly 8 columns (no junk)
   - Dashboard "Spent This Month" for "Makanan & Minuman" should show 45000
   - Dashboard "Remaining" should equal Budget - 45000
   - Reply confirms the expense

2. **New month sheet**: If current month sheet doesn't exist
   - New sheet is created with header row (Date, Description, etc.)
   - Then expense is appended as row 2

3. **Budget check**: Send "cek anggaran"
   - Should show actual "Spent This Month" values (not all zeros)

4. **Split expense**: Send "dinner 200k split with @John"
   - Row appended to your spreadsheet with Amount=100000
   - Row appended to John's spreadsheet with Amount=100000

5. **Monthly report**: Send "laporan bulan ini"
   - Should read from current month sheet and generate summary

6. **New user registration**: Register a new user
   - Month sheet headers should be created (not blocked by executeOnce)
