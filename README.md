# Expense Tracker Pro - Multi Channel

An n8n-based multi-channel expense and income tracker with AI intent parsing, Google Sheets storage, and channel-specific replies for Telegram, Webhook, Chat Widget, and WhatsApp.

---

# 📑 Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Installation](#installation)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Dependencies](#dependencies)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)
- [Contributors](#contributors)
- [License](#license)

---

## Introduction

This project provides a production-style n8n workflow (`Expense Tracker Pro - Multi Channel.json`) that:
- accepts financial messages from multiple channels,
- classifies intent with AI,
- stores expenses/income in user-specific Google Sheets,
- returns contextual responses based on user intent and sheet data.

## Features

- Multi-channel input: Telegram, Webhook, n8n Chat, WhatsApp (trigger currently disabled in JSON)
- AI intent parsing for text/image/PDF routes
- Expense, income, budget, split, balance/report intents
- Auto onboarding for new users with per-user spreadsheet creation
- Dashboard + monthly expense sheets + Income sheet support
- AI-based analysis replies for balance/report-style intents

## Tech Stack

- n8n workflow engine
- Google Sheets API (storage)
- Telegram Bot API
- Groq API (vision + analysis model path)
- Gemini/OpenAI/Ollama provider branches for intent parsing

## Installation

1. Import `Expense Tracker Pro - Multi Channel.json` into n8n.
2. Create a Google Sheets registry file with `Profiles` tab.
3. Configure environment variables and credentials.
4. Follow [docs/setup.md](docs/setup.md) for full setup and verification.

## Usage

Typical flow:
1. User sends message in one supported channel.
2. Workflow normalizes input and resolves user profile.
3. AI parser returns structured intent JSON.
4. Workflow executes action against Google Sheets.
5. Reply is routed back to the originating channel.

## Project Structure

- `Expense Tracker Pro - Multi Channel.json`: main n8n workflow
- `.env.example`: environment variable template
- `docs/setup.md`: setup and testing guide
- `docs/implementation-plan.md`: JSON-aligned implementation analysis and plan
- `skills/`: local skill artifacts

## Configuration

Core environment variables:

```env
MASTER_REGISTRY_ID=
MODEL_PROVIDER=gemini
TELEGRAM_BOT_TOKEN=
GROQ_API_KEY=
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_ACCESS_TOKEN=
```

Notes:
- `GROQ_API_KEY` is required for HTTP Groq nodes.
- WhatsApp variables are only required if WhatsApp path is enabled.
- Keep registry read/write nodes pointed to the same sheet ID.

## Dependencies

Runtime dependencies are managed by n8n node credentials and external APIs:
- Google Sheets OAuth2
- Telegram Bot token/credential
- Groq API credential
- Optional Gemini/OpenAI/Ollama credentials

## Examples

- `Lunch 45k` -> `record_expense`
- `terima gaji 5jt` -> `record_income`
- `set anggaran makanan 1jt` -> `set_budget`
- `saldo bulan ini` -> `check_balance`
- `laporan bulan ini` -> `get_report`
- `dinner 200k split with @John` -> `split_expense`

## Troubleshooting

- Registry mismatch: ensure all registry nodes use one sheet ID.
- Telegram not triggering: check credential, user filter, and workflow active state.
- Empty analysis: verify Groq credential and `GROQ_API_KEY`.
- PDF parsing issues: current Telegram media extraction is image-first and needs document fallback logic.

## Contributors

Project contributors can be listed here.

## License

Add your project license here (for example: MIT).
