# n8n Expense Tracker Pro - Implementation Plan

## Context

A comprehensive n8n workflow JSON (`expense-tracker.json`) that serves as a multi-channel AI-powered expense tracker bot. Users interact via WhatsApp, Telegram, n8n Chat Widget, or generic webhook. Each user gets their own Google Spreadsheet with a dashboard and monthly sheets. The bot uses a configurable AI model for natural language parsing (text, images, PDFs) and auto-detects the user's language.

## AI Model Selection

Set `MODEL_PROVIDER` environment variable in n8n to choose:

| Provider | Cost | Vision | Model |
|----------|------|--------|-------|
| `gemini` (DEFAULT) | FREE tier | Yes | Gemini 2.0 Flash |
| `ollama` | FREE (self-hosted) | Yes (LLaVA) | Llama 3 |
| `groq` | FREE tier | Yes | Llama 3.3 70B |
| `openai` | Paid | Yes | GPT-4o |

## Architecture Overview

```
ZONE A: Multi-Channel Ingress (4 triggers -> Normalize)
         WhatsApp | Telegram | Webhook | Chat Widget
                       |
ZONE B: Profile Resolution
         Master Registry lookup -> New user onboarding or continue
                       |
ZONE C: Intent Classification
         Media detection -> AI Agent (configurable model) -> Parse response
                       |
ZONE D: Action Execution
         Record | Budget | Report | Split | Recurring | Help
                       |
ZONE E: Response Dispatch
         Route reply back to originating channel
```

## Node Inventory (56 nodes)

### Zone A - Multi-Channel Ingress (8 nodes)

| Node | Type | Purpose |
|------|------|---------|
| WhatsApp Trigger | `n8n-nodes-base.whatsAppTrigger` | Receive WhatsApp messages (text, image, document) |
| Telegram Trigger | `n8n-nodes-base.telegramTrigger` | Receive Telegram messages (text, photo, document) |
| Webhook Trigger | `n8n-nodes-base.webhook` | Generic HTTP POST endpoint for custom integrations |
| Chat Trigger | `@n8n/n8n-nodes-langchain.chatTrigger` | n8n built-in chat widget (embeddable in websites) |
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

### Zone B - Profile Resolution (12 nodes)

| Node | Type | Purpose |
|------|------|---------|
| Lookup User Profile | `n8n-nodes-base.googleSheets` | Read Master Registry by userId |
| Check Awaiting Profile | `n8n-nodes-base.code` | Check if user state = "awaitingProfileName" |
| Route User State | `n8n-nodes-base.switch` | 3-way: New user / Awaiting name / Registered |
| Ask Profile Name | `n8n-nodes-base.code` | Build welcome message for new users |
| Save Awaiting State | `n8n-nodes-base.googleSheets` | Write "awaitingProfileName" state to Master Registry |
| Create User Spreadsheet | `n8n-nodes-base.httpRequest` | POST Google Sheets API to create new spreadsheet |
| Setup Dashboard Headers | `n8n-nodes-base.code` | Prepare 8 category rows for Dashboard |
| Write Dashboard Data | `n8n-nodes-base.googleSheets` | Write categories to Dashboard sheet |
| Setup Month Sheet Headers | `n8n-nodes-base.httpRequest` | Add column headers to first month sheet |
| Register User in Master | `n8n-nodes-base.googleSheets` | Update Master Registry with spreadsheetId, state=active |
| Send Registration Complete | `n8n-nodes-base.code` | Format success message |
| Merge User Context | `n8n-nodes-base.set` | Combine message with profile data for registered users |

**New user flow:**
1. First message -> user not found -> ask for profile name -> save "awaitingProfileName" state
2. Second message -> state = awaitingProfileName -> use messageText as profile name -> create spreadsheet -> register -> confirm

### Zone C - Intent Classification (15 nodes)

| Node | Type | Purpose |
|------|------|---------|
| Detect Media Type | `n8n-nodes-base.switch` | Route: text (fallback) / image / pdf |
| Download Media | `n8n-nodes-base.httpRequest` | Download image/PDF from WhatsApp/Telegram |
| Process with Vision | `n8n-nodes-base.code` | Prepare media for AI vision processing |
| Select Model Provider | `n8n-nodes-base.switch` | Route to AI Agent based on MODEL_PROVIDER env var |
| AI Agent (OpenAI) | `@n8n/n8n-nodes-langchain.agent` | AI Agent using GPT-4o |
| AI Agent (Gemini) | `@n8n/n8n-nodes-langchain.agent` | AI Agent using Gemini Flash (default) |
| AI Agent (Ollama) | `@n8n/n8n-nodes-langchain.agent` | AI Agent using local Ollama |
| AI Agent (Groq) | `@n8n/n8n-nodes-langchain.agent` | AI Agent using Groq |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | GPT-4o sub-node |
| Gemini Chat Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Gemini 2.0 Flash sub-node |
| Ollama Chat Model | `@n8n/n8n-nodes-langchain.lmChatOllama` | Llama 3 sub-node |
| Groq Chat Model | `@n8n/n8n-nodes-langchain.lmChatGroq` | Llama 3.3 70B sub-node |
| Structured Output Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | Force JSON output |
| Parse AI Response | `n8n-nodes-base.code` | Validate and extract structured JSON from AI |

**AI Agent intents:**
- `record_expense` - Log an expense
- `check_budget` - View budget status
- `set_budget` - Set category budget
- `get_report` / `merge_report` - Generate expense reports
- `split_expense` - Split expense between profiles
- `list_recurring` / `cancel_recurring` - Manage recurring expenses
- `help` - Show usage instructions
- `profile_name_response` - Handle profile name input

**Categories:**
Makanan & Minuman, Transportasi, Tagihan, Hiburan, Belanja, Kesehatan, Langganan, Lain-lain

### Zone D - Action Execution (16 nodes)

| Intent | Flow |
|--------|------|
| Record expense | Route by Intent -> Ensure Month Sheet -> Check/Create -> Record Expense Row -> Update Dashboard -> Reply |
| Check budget | Route by Intent -> Read Budget Data -> Format Budget Reply -> Reply |
| Set budget | Route by Intent -> Set Budget Value -> Format Budget Set Reply -> Reply |
| Get report | Route by Intent -> Read Month Data -> Generate Report -> Reply |
| Split expense | Route by Intent -> Handle Split -> Lookup Target Profile -> Record Both Sheets -> Reply |
| Help | Route by Intent -> Format Help Reply -> Reply |

### Zone E - Response Dispatch (5 nodes)

| Node | Type | Purpose |
|------|------|---------|
| Route Reply to Channel | `n8n-nodes-base.switch` | Switch on channel: whatsapp/telegram/webhook/chat |
| Send WhatsApp Reply | `n8n-nodes-base.httpRequest` | WhatsApp Cloud API message |
| Send Telegram Reply | `n8n-nodes-base.telegram` | Telegram sendMessage |
| Send Webhook Reply | `n8n-nodes-base.respondToWebhook` | HTTP JSON response |
| Send Chat Reply | `n8n-nodes-base.code` | Format response for n8n chat widget |

## Google Sheets Structure

### Master Registry (1 shared spreadsheet)

Sheet name: `Profiles`

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

## Merge/Split Features

1. **Expense splitting**: "dinner 200k split with @ProfileB" -> each profile gets their share recorded in their own spreadsheet
2. **Consolidated report**: Aggregates data from multiple profiles into one view (merge_report intent)

## Key Implementation Details

- **State management**: Master Registry "state" column persists user onboarding state across stateless workflow executions
- **Spreadsheet creation**: HTTP Request to Google Sheets API v4 with OAuth2 credentials creates a new spreadsheet per user
- **Month sheet auto-creation**: Before recording, check if month sheet exists; create via batchUpdate if missing
- **Language auto-detect**: AI system prompt instructs the model to mirror the user's language
- **Media handling**: Each normalizer resolves media to a URL; shared Download Media node fetches the binary
- **Model switching**: `MODEL_PROVIDER` env var routes to the correct AI Agent + LLM sub-node pair via a Switch node

## Credentials Required

| Credential | Required | Purpose |
|-----------|----------|---------|
| Google Sheets OAuth2 | Always | Master Registry + per-user spreadsheets |
| Google Gemini API | If MODEL_PROVIDER=gemini (default) | AI processing (FREE) |
| Groq API | If MODEL_PROVIDER=groq | AI processing (FREE tier) |
| Ollama API | If MODEL_PROVIDER=ollama | AI processing (self-hosted, FREE) |
| OpenAI API | If MODEL_PROVIDER=openai | AI processing (paid) |
| Telegram Bot API | If using Telegram | Trigger + reply |
| WhatsApp Business Cloud API | If using WhatsApp | Trigger + reply |

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `MASTER_REGISTRY_ID` | Yes | - | Google Spreadsheet ID for Profiles registry |
| `MODEL_PROVIDER` | No | `gemini` | AI provider: gemini/openai/ollama/groq |
| `WHATSAPP_PHONE_NUMBER_ID` | For WhatsApp | - | WhatsApp Business phone number ID |
| `WHATSAPP_ACCESS_TOKEN` | For WhatsApp | - | WhatsApp Cloud API access token |
| `TELEGRAM_BOT_TOKEN` | For Telegram | - | Telegram bot token (also in credentials) |
