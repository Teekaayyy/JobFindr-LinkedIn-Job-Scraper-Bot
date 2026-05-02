# 🔍 JobFindr — LinkedIn Scraper Telegram Bot

A fully automated job search bot built with **n8n**, **Apify**, and the **Telegram Bot API**. Users interact with a conversational Telegram bot to specify their job preferences step-by-step. JobFindr then fires a LinkedIn scraper in the background and delivers results as an Excel spreadsheet or inline text summary — no browser needed.

---

## 🎥 Demo

> User types `/search` → selects job role, type, location, experience, and salary via buttons → bot scrapes LinkedIn → results delivered as `.xlsx` or text summary directly in Telegram.

---

## ✨ Features

- 🤖 **Telegram-native UX** — fully button-driven, no typing required (unless the user wants a custom role or location)
- 🔄 **Stateless session design** — all conversation state is encoded inside button `callback_data`, eliminating race conditions and repeated messages
- 🌍 **Flexible filters** — job role, contract type, location (preset or custom), experience level, and salary range
- 📊 **Dual delivery formats** — Excel spreadsheet (`.xlsx`) or inline text summary, or both
- ⚡ **Async scraping** — Apify runs in the background; the bot pings the user the moment results are ready
- 🛡️ **Deduplication** — filters duplicate listings by job URL before delivery
- 🔁 **Graceful error handling** — separate failure path notifies the user if the scrape fails

---

## 🏗️ Architecture

```
Telegram (User)
     │
     ▼
📩 Telegram Trigger
     │
     ▼
🧠 Master Handler  ◄── Stateless: decodes full state from button callback_data
     │
     ├──► 📤 Send Message          (text response + inline keyboard)
     ├──► ✅ Answer Callback       (dismisses loading spinner on button tap)
     ├──► 🚀 Prepare + Start Apify Run  (fires LinkedIn scraper)
     └──► 📊 Prepare + Send Excel File  (on delivery request)

                         [Async — Apify callback]
                                  │
                         🔗 Apify Webhook
                                  │
                         🔔 Handle Apify Callback
                                  │
                         📥 Fetch Apify Results
                                  │
                         ⚙️ Process & Store Results
                                  │
                         🎉 Notify User — Results Ready
```

---

## 🛠️ Stack

| Component | Purpose |
|---|---|
| [n8n](https://n8n.io) | Workflow automation engine |
| [Telegram Bot API](https://core.telegram.org/bots/api) | User interface |
| [Apify](https://apify.com) — `cheap_scraper/linkedin-job-scraper` | LinkedIn job scraping |
| n8n Spreadsheet File node | Excel `.xlsx` generation |

---

## 📋 Prerequisites

- An [n8n](https://n8n.io) instance (cloud or self-hosted)
- A [Telegram Bot Token](https://core.telegram.org/bots#botfather) from BotFather
- An [Apify](https://apify.com) account with access to the `cheap_scraper~linkedin-job-scraper` actor
- Your n8n instance must be publicly accessible (for Apify webhooks to reach it)

---

## 🚀 Setup

### 1. Clone / Import the Workflow

Download `JobFindr_-_LinkedIn_Scraper_Bot_CLEAN.json` and import it into n8n:

**n8n UI → Workflows → Import from file**

### 2. Create Your Telegram Credential

In n8n, go to **Credentials → New → Telegram API** and paste your bot token.

Assign this credential to:
- `📩 Telegram Trigger` node
- `✅ Answer Callback` node

### 3. Set Your Secrets in Code Nodes

The following values are used inside JavaScript Code nodes. Replace each placeholder:

| Node | Placeholder | Replace with |
|---|---|---|
| `📤 Send Message` | `YOUR_TELEGRAM_BOT_TOKEN` | Your Telegram bot token |
| `🎉 Notify — Results Ready` | `YOUR_TELEGRAM_BOT_TOKEN` | Your Telegram bot token |
| `❌ Send Failure Message1` | `YOUR_TELEGRAM_BOT_TOKEN` | Your Telegram bot token |
| `📥 Fetch Apify Results` | `YOUR_APIFY_API_TOKEN` | Your Apify API token |
| `📋 Prepare Apify Input1` | `YOUR_APIFY_API_TOKEN` | Your Apify API token |
| `📋 Prepare Apify Input1` | `YOUR_N8N_SUBDOMAIN.app.n8n.cloud` | Your n8n instance URL |

> 💡 **Tip:** If you're on n8n Cloud, your base URL will look like `https://yourname.app.n8n.cloud`. For self-hosted, use your public domain.

### 4. Activate the Workflow

Toggle the workflow to **Active** in n8n. This registers the Telegram webhook automatically.

---

## 💬 Bot Commands

| Command | Description |
|---|---|
| `/start` | Start a new job search |
| `/search` | Start a new job search |
| `/status` | Check if a scrape is currently in progress |
| `/help` | Show available commands |

---

## 🗺️ Conversation Flow

```
/search
   └── Select job role (10 presets or type your own)
         └── Select contract type (Full-Time / Part-Time / Contract / Internship / Any)
               └── Select location (6 presets or type your own)
                     └── Select experience level (Entry / Mid / Senior / Director / Internship / Any)
                           └── Select salary range (5 tiers or No Preference)
                                 └── Confirm → Apify scrape fires 🚀
                                       └── Results ready → choose delivery format
                                             ├── 📊 Excel Spreadsheet (.xlsx)
                                             ├── 📋 Text Summary in Chat (top 10)
                                             └── 🔥 Both
```

---

## 📦 Excel Output Fields

Each row in the delivered `.xlsx` file contains:

| Column | Description |
|---|---|
| `#` | Row number |
| `Job Title` | Position title |
| `Company` | Hiring company |
| `Location` | Job location |
| `Contract Type` | Employment type |
| `Work Type` | Remote / Hybrid / On-site |
| `Experience` | Required experience level |
| `Salary` | Salary info (if listed) |
| `Applicants` | Number of applicants |
| `Posted` | Date posted |
| `Apply Link` | Direct LinkedIn URL |
| `Recruiter` | Recruiter name |
| `Recruiter URL` | Recruiter LinkedIn profile |
| `Job ID` | LinkedIn job ID |
| `Scraped On` | Date the data was collected |

---

## ⚙️ How the Stateless Design Works

Instead of storing conversation state server-side (which causes race conditions in n8n), all accumulated user choices are encoded directly into each button's `callback_data` string.

**Format:** `STEP~jobIndex~typeCode~locationIndex~experienceIndex~salaryIndex`

**Example:** `S~0~F~3~1~2` means:
- Step: Salary confirmation
- Job: `0` → Software Engineer
- Type: `F` → Full-Time
- Location: `3` → Nigeria
- Experience: `1` → Mid-Senior Level
- Salary: `2` → $60K–$100K

When a user taps a button, the full history travels with it. No shared state, no race conditions.

n8n's static workflow data (`$getWorkflowStaticData`) is only used for:
- `awaitingCustom[chatId]` — flag indicating the user is typing a custom value
- `customJob[chatId]` / `customLoc[chatId]` — temporary storage for typed custom text
- `scrapeJobs[chatId]` — tracking active Apify runs
- `results[chatId]` — storing scraped results pending delivery

---

## 🔒 Security Notes

- **Never commit your real credentials.** All tokens in this repo are placeholders.
- For production, consider using n8n's built-in **Credentials** store for all secrets rather than hardcoding in Code nodes.
- The Apify webhook endpoint (`/webhook/jobfindr-apify-done`) is publicly accessible by design — it must be reachable for the async callback to work.

---

## 📁 Project Structure

```
├── JobFindr_-_LinkedIn_Scraper_Bot_CLEAN.json   # n8n workflow (no credentials)
└── README.md
```

---

## 🤝 Contributing

Pull requests are welcome. For major changes, please open an issue first.

---

## 📄 License

MIT
