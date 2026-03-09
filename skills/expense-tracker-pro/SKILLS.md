---
name: expense-tracker-pro
description: Multi-channel AI expense tracker with natural language parsing, budgets, reports, and expense splitting
author: clawd-team
version: 2.0.0
triggers:
  - "catat pengeluaran"
  - "lacak pengeluaran"
  - "record expense"
  - "track expense"
  - "cek anggaran"
  - "check budget"
  - "laporan pengeluaran"
  - "expense report"
  - "split expense"
  - "help"
---

# Expense Tracker Pro

Track expenses through natural language chat. Supports WhatsApp, Telegram, n8n Chat Widget, and webhook API. Each user gets their own Google Spreadsheet with dashboard and monthly sheets.

## Features

- Natural language expense recording (text, image, PDF)
- Auto-categorization with 8 categories
- Budget management per category
- Monthly/weekly/yearly reports
- Expense splitting between profiles
- Recurring expense tracking
- Multilingual (auto-detects user language)
- Multi-channel: WhatsApp, Telegram, Chat Widget, Webhook

## AI Intent System

The AI parses user messages into structured JSON intents:

### Intents

| Intent | Description | Example Input |
|--------|-------------|---------------|
| `record_expense` | Log an expense | "Lunch 45k", "Makan siang 45rb" |
| `check_budget` | View budget status | "Check my budget", "Cek anggaran" |
| `set_budget` | Set category budget | "Set food budget 500k" |
| `get_report` | Generate expense report | "Monthly report", "Laporan bulan ini" |
| `split_expense` | Split between profiles | "Dinner 200k split with @John" |
| `list_recurring` | List recurring expenses | "Show recurring", "Lihat berulang" |
| `cancel_recurring` | Cancel a recurring expense | "Cancel Netflix recurring" |
| `merge_report` | Consolidated multi-profile report | "Merge report with @John" |
| `help` | Show usage help | "Help", "Bantuan" |

### JSON Schemas

**Record Expense:**
```json
{
  "intent": "record_expense",
  "date": "2026-03-09",
  "description": "Makan siang",
  "amount": 45000,
  "currency": "IDR",
  "category": "Makanan & Minuman",
  "splitWith": null,
  "recurring": null,
  "notes": "",
  "replyText": "Tercatat: Makan siang IDR 45.000 (Makanan & Minuman)"
}
```

**Set Budget:**
```json
{
  "intent": "set_budget",
  "category": "Makanan & Minuman",
  "amount": 500000,
  "currency": "IDR",
  "period": "monthly",
  "replyText": "Budget set: Makanan & Minuman = IDR 500,000/month"
}
```

**Split Expense:**
```json
{
  "intent": "split_expense",
  "date": "2026-03-09",
  "description": "Dinner",
  "amount": 200000,
  "currency": "IDR",
  "category": "Makanan & Minuman",
  "splitWith": "John",
  "splitType": "equal",
  "replyText": "Split recorded: Dinner IDR 200,000 - Your share: IDR 100,000"
}
```

## Categories

Auto-detected from context (use exact names):

| Category | Examples |
|----------|----------|
| Makanan & Minuman | lunch, dinner, coffee, groceries, nasi goreng, kopi |
| Transportasi | taxi, grab, gojek, bus, gas, bensin |
| Tagihan | electricity, water, internet, phone bill, listrik |
| Hiburan | movies, games, Netflix, concert, bioskop |
| Belanja | clothes, shoes, electronics, online shopping |
| Kesehatan | medicine, doctor, gym, supplements, obat |
| Langganan | Netflix, Spotify, subscriptions |
| Lain-lain | everything else |

## Amount Parsing Rules

- `k` suffix = x1,000 (45k = 45,000)
- `rb` / `ribu` suffix = x1,000 (25rb = 25,000)
- `jt` / `juta` suffix = x1,000,000 (1.5jt = 1,500,000)
- `Rp` prefix is optional
- Default currency: IDR

## Natural Language Examples

```
"Makan siang 45k"                -> record_expense, Makanan & Minuman, IDR 45,000
"Spent 150k on electricity"      -> record_expense, Tagihan, IDR 150,000
"Kopi 25rb"                      -> record_expense, Makanan & Minuman, IDR 25,000
"Grab to office Rp 35.000"       -> record_expense, Transportasi, IDR 35,000
"Netflix 54k recurring monthly"  -> record_expense, Langganan, IDR 54,000, recurring
"Dinner 200k split with @Jane"   -> split_expense, Makanan & Minuman, IDR 100,000 each
"Set food budget 500k"           -> set_budget, Makanan & Minuman, IDR 500,000
"Check my budget"                -> check_budget
"Monthly report"                 -> get_report, period: month
"Help"                           -> help
```

## Tips

- Include amounts clearly for accurate recording
- Use "berulang" / "recurring" for subscriptions: "Netflix 54k berulang tiap bulan"
- Use "@ProfileName" or "split with ProfileName" for splitting
- Ask "tren pengeluaran" / "spending trends" to see patterns over time
- Send receipt photos or PDF invoices for automatic parsing
