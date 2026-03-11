# n8n Expense Tracker Pro - Implementation Plan (JSON-Aligned)

## Scope and Source of Truth

This plan is based on direct analysis of:
- `Expense Tracker Pro - Multi Channel.json`

Workflow snapshot from the JSON:
- Workflow name: `Expense Tracker Pro - Multi Channel`
- Status: `active=true`
- Node count: `91`
- Main-graph connections: `86`
- Disabled nodes: `4` (`WhatsApp Trigger`, `OpenAI Chat Model`, `Ollama Chat Model`, `Simple Memory`)

## Current Architecture

### Zone A: Multi-channel ingress and normalization

Input channels:
- Telegram (`Telegram Trigger`)
- Webhook (`Webhook Trigger`, path `expense-tracker`)
- Chat widget (`When chat message received` + `Chat` response node)
- WhatsApp trigger exists but is currently disabled

Normalization code nodes:
- `Normalize Telegram`
- `Normalize WhatsApp`
- `Normalize Webhook`
- `Normalize Chat`

Unified payload shape produced by normalizers:
- `userId`, `userName`, `messageText`, `mediaUrl`, `mediaType`, `channel`, `rawPayload`

### Zone B: user lookup and onboarding

Core nodes:
- `Lookup User Profile` (Google Sheets, `Profiles` sheet, uses `MASTER_REGISTRY_ID` env)
- `Check Awaiting Profile` (resolves user by multiple candidate keys)
- `Route User State` (registered vs new vs awaiting profile name)

Onboarding path (new user):
1. `Ask Profile Name`
2. `Save Awaiting State`
3. `Route Reply to Channel`
4. On next message: `Save Profile Name` -> `Create User Spreadsheet`
5. Initialize sheets:
   - `Setup Dashboard Headers` -> `Write Dashboard Data`
   - `Setup Income Sheet` -> `Write Income Headers`
   - `Setup Month Sheet Headers`
6. `Register User in Master`
7. `Send Registration Complete`

### Zone C: intent parsing (text/image/pdf)

Media router:
- `Detect Media Type`
  - output 2 (`text`) -> `Select Model Provider`
  - output 0 (`image`) and output 1 (`pdf`) both currently go to `Get File from Telegram`

Provider router:
- `Select Model Provider` via `MODEL_PROVIDER`
  - `openai` -> `AI Agent (OpenAI)`
  - `ollama` -> `AI Agent (Ollama)`
  - `groq` -> `AI Agent (Groq)`
  - `gemini` -> `AI Agent (Gemini)`

Vision path:
- `Get File from Telegram` -> `Build Telegram File URL` -> `Groq Vision + Intent` -> `Extract Groq Vision Output`

Parsing:
- `Parse AI Response` merges parsed intent with original user context from `Merge User Context`
- Supports object or array JSON outputs and normalizes to `parsedIntent` + `replyText`

### Zone D: action execution

Intent router:
- `Route by Intent` with 10 outputs

Active write flows:
- `record_expense`: `Ensure Month Sheet` -> `Check and Create Month Sheet` -> conditional create/header -> `Record Expense Row` -> dashboard refresh/update -> `Aggregate Expense Reply`
- `record_income`: `Record Income Row`
- `set_budget`: `Prepare Budget Update` -> `Set Budget Value` -> `Format Budget Set Reply`
- `split_expense`: `Handle Split Expense` -> `Lookup Split Target Profile` -> `Record Split in Both Sheets` -> `Record Split Rows`

Active read/analysis flows:
- `check_budget`: `Read Budget Data` -> `Prepare Budget Analysis`
- `check_balance`: `Read Budget Data` + `Read Income Data` + `Read Month Data for Report` -> prepare nodes -> `Merge` (3 inputs) -> `Combine Report Data` -> `AI Agent` -> `Extract Report Analysis`
- `get_report` and `merge_report`: currently routed to same 3-data analysis chain as balance

Notes:
- `AI Agent` (generic) + `Groq Chat Model` currently drive multi-source analysis replies.
- Legacy nodes `AI Analysis Agent (Budget)`, `Format Budget Reply`, and `Generate Report` exist but are not connected from active intent routes.

### Zone E: response dispatch

`Route Reply to Channel` sends to:
- WhatsApp: `Send WhatsApp Reply`
- Telegram: merge path with loading/deletion UX (`Merge2`/`Combine for delete message`/`Delete a chat message`/`Merge3`) -> `Combined message to send` -> `Send Telegram Reply`
- Webhook: `Send Webhook Reply`
- Chat: `Send Chat Reply`

## Intent Routing Map (from JSON)

- output 0: `record_expense` -> `Ensure Month Sheet`
- output 1: `record_income` -> `Record Income Row`
- output 2: `check_balance` -> `Read Budget Data` + `Read Income Data` + `Read Month Data for Report`
- output 3: `check_budget` -> `Read Budget Data`
- output 4: `set_budget` -> `Prepare Budget Update`
- output 5: `get_report` OR `merge_report` -> `Read Month Data for Report` + `Read Income Data` + `Read Budget Data`
- output 6: `split_expense` -> `Handle Split Expense`
- output 7: `help` -> `Format Help Reply`
- output 8: `list_recurring` OR `cancel_recurring` -> direct reply fallback
- output 9: default fallback -> direct reply fallback

## Data Model in Google Sheets

Master registry (`Profiles`):
- `userId`, `profileName`, `spreadsheetId`, `createdDate`, `state`, `language`, plus extra fields currently written by update nodes

Per-user spreadsheet:
- `Dashboard`: category, budget, spent, remaining
- `Recurring`: created at spreadsheet creation
- Monthly sheet (`MMM-yyyy`): expense rows
- `Income`: date, description, source, amount, currency, recurring, notes, month, year

## Credentials and Environment

Credentials found in workflow JSON:
- `googleSheetsOAuth2Api` (id `36FAVfvsRenvGyJc`)
- `googlePalmApi` (Gemini)
- `groqApi` (id `m5NPNO02GxojmMO8`)
- `telegramApi` (id `ZbU40jBmvjhBVd3d`)

Environment variables referenced by nodes:
- `MASTER_REGISTRY_ID`
- `MODEL_PROVIDER`
- `TELEGRAM_BOT_TOKEN`
- `GROQ_API_KEY`
- `WHATSAPP_PHONE_NUMBER_ID`
- `WHATSAPP_ACCESS_TOKEN`

## Findings and Risks

1. Master registry ID inconsistency
- `Lookup User Profile` uses `MASTER_REGISTRY_ID` env.
- `Save Awaiting State` and `Register User in Master` still target a hardcoded spreadsheet ID.
- Impact: read/write may hit different registries.

2. PDF media path is incomplete
- `Detect Media Type` sends `pdf` to `Get File from Telegram`.
- `Get File from Telegram` uses `rawPayload.message.photo.last().file_id`, which fails for document-only messages.

3. `merge_report` is not true cross-profile merge yet
- `merge_report` shares the same read path as `get_report` without explicit profile lookup/aggregation.

4. Dead/legacy nodes still present
- `AI Analysis Agent (Budget)`, `Format Budget Reply`, `Generate Report` are still connected to reply node but unused by active intent routing.

5. Telegram trigger is user-filtered
- `Telegram Trigger` has `additionalFields.userIds = "frisrsyd"`.
- This may limit who can trigger the workflow.

## Implementation Priorities

### Priority 1: correctness and stability
- Replace hardcoded registry IDs in write nodes with `MASTER_REGISTRY_ID`.
- Fix Telegram PDF path (`document.file_id` fallback) in `Get File from Telegram`.
- Add explicit guard/error reply when media file ID is missing.

### Priority 2: complete intent behavior
- Implement real `merge_report` by resolving multiple profiles and combining their sheet data.
- Implement `list_recurring` and `cancel_recurring` with Recurring sheet reads/writes.

### Priority 3: cleanup and maintainability
- Remove or archive unused legacy nodes.
- Standardize analysis path naming (`Extract Report Analysis` currently used for budget/balance/report combined path).
- Add execution-level logging without `console.log` dependency.

## Validation Checklist

- New user onboarding creates Dashboard, Income, and current month sheet; registry row includes spreadsheetId.
- Text expense updates monthly sheet and dashboard spent.
- Telegram receipt image reaches Groq vision and records expense.
- Telegram PDF either processes correctly or returns explicit unsupported/error guidance.
- `record_income` appends to Income sheet.
- `check_budget` and `check_balance` return data-grounded analysis.
- `get_report` returns period report from combined reads.
- Telegram loading message is deleted before final reply.
