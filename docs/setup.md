# Expense Tracker Pro - Setup & Testing Guide

## Prerequisites

- n8n instance (self-hosted or cloud) running v1.24.0+
- Google account with Google Sheets API enabled
- At least one messaging channel configured (or use the built-in Chat Widget)
- One AI provider API key (Gemini is free and recommended)

## Step 1: Import the Workflow

1. Open your n8n instance
2. Go to **Workflows** -> **Import from File**
3. Select `expense-tracker.json`
4. The workflow will appear with 56 nodes across 5 zones

## Step 2: Create the Master Registry Spreadsheet

1. Go to [Google Sheets](https://sheets.google.com)
2. Create a new spreadsheet named **"Expense Tracker - Master Registry"**
3. Rename the first sheet tab to **`Profiles`**
4. Add these column headers in row 1:

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| userId | profileName | spreadsheetId | createdDate | state | language |

5. Copy the spreadsheet ID from the URL:
   ```
   https://docs.google.com/spreadsheets/d/COPY_THIS_PART/edit
   ```

## Step 3: Configure Environment Variables

In your n8n instance, set these environment variables:

### Required
```env
MASTER_REGISTRY_ID=your_spreadsheet_id_from_step_2
```

### AI Provider (choose one)
```env
# Option A: Google Gemini (FREE - recommended)
MODEL_PROVIDER=gemini

# Option B: Groq (FREE tier)
MODEL_PROVIDER=groq

# Option C: Ollama (FREE, self-hosted)
MODEL_PROVIDER=ollama

# Option D: OpenAI (paid)
MODEL_PROVIDER=openai
```

### Messaging channels (only set what you use)
```env
# WhatsApp
WHATSAPP_PHONE_NUMBER_ID=your_phone_number_id
WHATSAPP_ACCESS_TOKEN=your_access_token

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
```

**How to set env vars in n8n:**
- **Docker**: Add to `docker-compose.yml` under `environment:`
- **npm/global**: Add to your shell profile or `.env` file
- **n8n Cloud**: Settings -> Environment Variables

## Step 4: Configure Credentials

In n8n, go to **Credentials** and create:

### Google Sheets OAuth2 (required)
1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a project (or use existing)
3. Enable **Google Sheets API** and **Google Drive API**
4. Create OAuth2 credentials (Web application)
5. Add redirect URI: `https://your-n8n-url/rest/oauth2-credential/callback`
6. In n8n: Credentials -> New -> Google Sheets OAuth2
7. Enter Client ID and Client Secret, then connect

### AI Provider (based on MODEL_PROVIDER)

**Gemini (default):**
1. Go to [Google AI Studio](https://aistudio.google.com/apikey)
2. Create an API key (free)
3. In n8n: Credentials -> New -> Google Gemini API -> paste key

**Groq:**
1. Go to [Groq Console](https://console.groq.com/keys)
2. Create an API key (free)
3. In n8n: Credentials -> New -> Groq API -> paste key

**Ollama:**
1. Install [Ollama](https://ollama.ai) on your server
2. Run: `ollama pull llama3`
3. In n8n: Credentials -> New -> Ollama API -> set base URL (default: `http://localhost:11434`)

**OpenAI:**
1. Go to [OpenAI Platform](https://platform.openai.com/api-keys)
2. Create an API key
3. In n8n: Credentials -> New -> OpenAI API -> paste key

### Telegram Bot (if using Telegram)
1. Message [@BotFather](https://t.me/BotFather) on Telegram
2. Send `/newbot` and follow the prompts
3. Copy the bot token
4. In n8n: Credentials -> New -> Telegram API -> paste token

### WhatsApp Business (if using WhatsApp)
1. Set up a [Meta Developer App](https://developers.facebook.com)
2. Add WhatsApp product to your app
3. Get the Phone Number ID and Access Token from the WhatsApp dashboard
4. In n8n: Credentials -> New -> WhatsApp Business Cloud API -> configure

## Step 5: Activate the Workflow

1. Open the imported workflow
2. Click the **Active** toggle in the top-right corner
3. The workflow is now live and listening for messages

## Step 6: Test with Built-in Chat (Easiest)

The **Chat Trigger** node provides a built-in chat widget - no external setup needed!

1. In the workflow editor, click the **Chat Trigger** node
2. Click **"Chat"** button at the top, or find the **Chat URL** in the node settings
3. The chat widget opens in a new window
4. Test the following scenarios:

### Test 1: New User Registration
```
You: Hello
Bot: Welcome! What name would you like for your profile?
You: John
Bot: Profile "John" created! Your expense tracker spreadsheet is ready...
```

### Test 2: Record an Expense
```
You: Lunch 45k
Bot: Recorded: Lunch IDR 45,000 (Makanan & Minuman)
```

### Test 3: Record with Specific Category
```
You: Grabbed a taxi 35k
Bot: Recorded: Taxi IDR 35,000 (Transportasi)
```

### Test 4: Set a Budget
```
You: Set food budget 500k
Bot: Budget set: Makanan & Minuman = IDR 500,000 per month
```

### Test 5: Check Budget
```
You: Check my budget
Bot: Budget Status:
     Makanan & Minuman: Spent 45,000 / Budget 500,000 (Remaining: 455,000)
     ...
```

### Test 6: Monthly Report
```
You: Monthly report
Bot: Expense Report (This Month):
     - Makanan & Minuman: IDR 45,000 (56.3%)
     - Transportasi: IDR 35,000 (43.7%)
     Total: IDR 80,000
     Transactions: 2
```

### Test 7: Split Expense
```
You: Dinner 200k split with @John
Bot: Split recorded: Dinner IDR 200,000
     - Your share: IDR 100,000
     - John's share: IDR 100,000
```

### Test 8: Help
```
You: help
Bot: Expense Tracker Help:
     Record expense: "Lunch 45k"
     Budget: "Set food budget 500k"
     Reports: "Monthly report"
     ...
```

## Testing via Telegram

1. Open your Telegram bot
2. Send `/start` or any message
3. Follow the same test scenarios above
4. Test with images: send a photo of a receipt

## Testing via WhatsApp

1. Open WhatsApp
2. Send a message to your WhatsApp Business number
3. Follow the same test scenarios above
4. Test with documents: send a PDF invoice

## Testing via Webhook (API)

Send a POST request:

```bash
curl -X POST https://your-n8n-url/webhook/expense-tracker \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "test-user-001",
    "userName": "Test User",
    "text": "Lunch 45k"
  }'
```

Expected response:
```json
{
  "success": true,
  "reply": "Recorded: Lunch IDR 45,000 (Makanan & Minuman)",
  "data": {
    "intent": "record_expense",
    "category": "Makanan & Minuman",
    "amount": 45000
  }
}
```

## Embedding the Chat Widget on Your Website

You can embed the n8n chat widget on any website:

```html
<link href="https://cdn.jsdelivr.net/npm/@n8n/chat/dist/style.css" rel="stylesheet" />
<script type="module">
  import { createChat } from 'https://cdn.jsdelivr.net/npm/@n8n/chat/dist/chat.bundle.es.js';
  createChat({
    webhookUrl: 'https://your-n8n-url/webhook/CHAT_TRIGGER_WEBHOOK_ID/chat',
    mode: 'window',
    chatInputKey: 'chatInput',
    metadata: {},
    showWelcomeScreen: true,
    initialMessages: [
      'Hi! I\'m your expense tracker bot. Send me your expenses in natural language!'
    ],
    i18n: {
      en: {
        title: 'Expense Tracker Pro',
        subtitle: 'Track expenses with AI',
        inputPlaceholder: 'Type your expense...',
      },
    },
  });
</script>
```

Replace `CHAT_TRIGGER_WEBHOOK_ID` with the webhook ID from your Chat Trigger node.

## Troubleshooting

### "Profile not found" on first message
- Check that the Master Registry spreadsheet ID is correctly set in `MASTER_REGISTRY_ID`
- Verify the sheet tab is named exactly `Profiles` (case-sensitive)
- Ensure Google Sheets OAuth2 credential has access to the spreadsheet

### AI returns "help" for everything
- Check that `MODEL_PROVIDER` is set correctly
- Verify the AI credential is configured and working
- Try a different model provider to isolate the issue

### Month sheet not created
- Ensure Google Sheets OAuth2 has write access to the user's spreadsheet
- Check the n8n execution log for API errors

### WhatsApp/Telegram not triggering
- Verify webhook URLs are correctly configured in Meta/Telegram
- Check that the workflow is **Active** (toggle is ON)
- Review n8n execution history for errors

### Chat widget not loading
- Ensure the Chat Trigger node is in the workflow and the workflow is active
- Check browser console for CORS errors
- Verify `allowedOrigins` is set to `*` or your domain

## Supported Languages

The bot auto-detects and responds in the user's language. Tested with:
- Bahasa Indonesia
- English
- And other languages supported by the AI model

## Expense Categories

| Category | Examples |
|----------|----------|
| Makanan & Minuman | lunch, dinner, coffee, groceries, nasi goreng |
| Transportasi | taxi, grab, gojek, bus, gas/petrol |
| Tagihan | electricity, water, internet, phone bill |
| Hiburan | movies, games, Netflix, concert |
| Belanja | clothes, shoes, electronics, online shopping |
| Kesehatan | medicine, doctor, gym, supplements |
| Langganan | Netflix, Spotify, subscriptions |
| Lain-lain | everything else |

## Natural Language Examples

The AI understands flexible input formats:

```
"Makan siang 45k"              -> Makanan & Minuman, IDR 45,000
"Spent 150k on electricity"     -> Tagihan, IDR 150,000
"Kopi 25rb"                     -> Makanan & Minuman, IDR 25,000
"Grab to office Rp 35.000"     -> Transportasi, IDR 35,000
"Netflix 54k recurring monthly" -> Langganan, IDR 54,000, recurring
"Dinner 200k split with @Jane" -> Split expense, IDR 100,000 each
```
