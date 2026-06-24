# VEDA — Setup Guide
### Verified & Enriched Daily Affairs | AI-Powered Current Affairs for Students

> Complete this guide before importing VEDA.json into n8n.  
> Estimated setup time: **15 minutes**

---

## Step 1: Get a New Gemini API Key (For VEDA)

VEDA uses its own separate Gemini key so it doesn't share quota with ARIA.

1. Go to → **https://aistudio.google.com/app/apikey**
2. Click **"Create API Key"**
3. Select **"Create API key in new project"** → name it `VEDA-Project`
4. Copy the key (starts with `AIza...`)
5. Save it somewhere safe — you'll paste it into VEDA.json

> **Free tier:** 1,500 requests/day, 15 req/min — more than enough for VEDA.

---

## Step 1.5: Add VEDA Bot to n8n as a New Telegram Credential

VEDA has its own dedicated Telegram bot (`@Veda085_bot`) — separate from ARIA's bot.

1. Open **n8n** → **Settings** → **Credentials**
2. Click **"Add Credential"** → search for **"Telegram"**
3. Name it: `VEDA Telegram Bot`
4. Paste the bot token:
   ```
   8825496944:AAH68pY28TN4QqTmjLMoEtA2t3vFNZRXeVk
   ```
5. Click **Save**
6. **Copy the Credential ID** — you'll see it in the URL or credential list (looks like `xYzAbCdEfGhI`)
7. Open `VEDA.json` → find `YOUR_VEDA_TELEGRAM_CRED_ID` → replace with that ID

> ⚠️ **Important**: Start a chat with **@Veda085_bot** on Telegram before testing so it can send you messages.

---


## Step 2: Get a New Groq API Key (For VEDA Fallback)

1. Go to → **https://console.groq.com/keys**
2. Click **"Create API Key"**
3. Name it: `VEDA-Fallback`
4. Copy the key (starts with `gsk_...`)
5. Save it — you'll paste it into VEDA.json

> **Free tier:** 14,400 req/day on Llama 3.1 8B Instant — zero cost.

---

## Step 3: Create the VEDA Google Sheet

1. Open → **https://sheets.google.com**
2. Click **"Blank spreadsheet"**
3. Name it: `VEDA — Current Affairs Database`
4. Copy the Sheet ID from the URL:
   ```
   https://docs.google.com/spreadsheets/d/THIS_PART_IS_YOUR_ID/edit
   ```

### Tab 1 — Sheet1 (rename it if needed — keep as "Sheet1")
Add these headers in Row 1 exactly:

| A | B | C | D | E | F | G | H | I | J | K | L | M | N | O | P |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ID | Date | Category | Headline | Source | Link | Summary | Key Points | Exam Score | MCQ 1 | Options 1 | Answer 1 | MCQ 2 | Options 2 | Answer 2 | Used In Quiz |

### Tab 2 — Trends
Click `+` → name it **`Trends`** → Add Row 1 headers:

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| Week | Date | Total Articles | Avg Exam Score | Top Category | Category Breakdown |

### Tab 3 — Error Log
Click `+` → name it **`Error Log`** → Add Row 1 headers:

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| Timestamp | Node Name | Error Type | Error Message | Full Error Details | Status |

---

## Step 4: Update VEDA.json with Your Credentials

Open `VEDA.json` and replace these 3 placeholders:

| Placeholder | Replace With |
|------------|-------------|
| `YOUR_VEDA_GEMINI_KEY` | Your new Gemini API key from Step 1 |
| `YOUR_VEDA_GROQ_KEY` | Your new Groq API key from Step 2 |
| `YOUR_VEDA_SHEET_ID` | Your Google Sheet ID from Step 3 |

Quick way (terminal):
```bash
# macOS
sed -i '' 's/YOUR_VEDA_GEMINI_KEY/AIzaYOURKEYHERE/g' VEDA.json
sed -i '' 's/YOUR_VEDA_GROQ_KEY/gsk_YOURGROQKEY/g' VEDA.json
sed -i '' 's/YOUR_VEDA_SHEET_ID/YOUR_SHEET_ID/g' VEDA.json
```

---

## Step 5: Import VEDA.json into n8n

1. Open your **n8n Cloud** dashboard → **https://app.n8n.cloud**
2. Click **"Add workflow"** → **"Import from file"**
3. Select `VEDA.json`
4. The workflow loads with all 38 nodes

---

## Step 6: Connect Credentials in n8n

After import, open these nodes and connect credentials:

| Node | Credential Needed |
|------|-----------------|
| All Google Sheets nodes | Google Sheets OAuth2 (same as ARIA) |
| All Telegram nodes | Telegram account (same bot as ARIA) |
| HTTP Request — Gemini | No credential (key is in URL) |
| HTTP Request1 — Groq | No credential (key is in header) |

> ⚠️ **If Google Sheets shows "credential missing"**: click each Google Sheets node → select your existing ARIA Google OAuth2 credential — it works for any Google Sheet.

---

## Step 7: Verify & Activate

1. Click **"Test Workflow"** on the Schedule Trigger node
2. Watch it flow through all nodes
3. Check your VEDA Sheet — rows should appear
4. Check Telegram — daily brief should arrive
5. Once satisfied → toggle **"Active"** to turn it on

---

## Workflow Schedule

| Trigger | When | What |
|---------|------|------|
| Schedule Trigger | Daily 7:00 AM IST | Fetches, analyzes, saves, and sends daily brief |
| Weekly Quiz Trigger | Sunday 9:00 AM IST | Sends weekly quiz + revision summary |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No articles found | RSS feeds might be slow — re-run manually |
| Gemini fails | Groq auto-activates as fallback |
| Both AIs fail | Error alert sent to Telegram + logged to Error Log tab |
| Sheet not updating | Verify Sheet ID and credential connection |
| Telegram not receiving | Confirm Chat ID matches your account |

---

*VEDA Setup Guide — Naveed Bhat — June 2026*
