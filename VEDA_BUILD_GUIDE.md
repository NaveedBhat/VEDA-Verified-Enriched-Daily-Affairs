# VEDA — Complete Build Guide & Implementation Log

> **Developed by Naveed Bhat**  
> A full step-by-step record of how VEDA was built from scratch — every decision, every fix, every piece of code.

---

## 📋 Table of Contents

1. [Project Vision](#1-project-vision)
2. [Pre-requisites & Accounts Setup](#2-pre-requisites--accounts-setup)
3. [Creating the Telegram Bot](#3-creating-the-telegram-bot)
4. [Getting Your Telegram Chat ID](#4-getting-your-telegram-chat-id)
5. [Setting Up Google Sheets Database](#5-setting-up-google-sheets-database)
6. [Getting a Gemini API Key](#6-getting-a-gemini-api-key)
7. [Getting a Groq API Key (Fallback AI)](#7-getting-a-groq-api-key-fallback-ai)
8. [Building the n8n Workflow — Step by Step](#8-building-the-n8n-workflow--step-by-step)
9. [Key AI Prompts & Formatting](#9-key-ai-prompts--formatting)
10. [Final Workflow State](#10-final-workflow-state)

---

## 1. Project Vision

### What We Wanted to Build
An autonomous AI system for competitive exam aspirants (UPSC, SSC, Banking) that:
- Monitors top Indian news sites from 10 RSS sources 24/7
- Analyzes each article using Gemini AI (with Groq as automatic fallback)
- Grades articles by Exam Relevance (1-10) and filters out noise (<7)
- Automatically generates 2 practice MCQs per article based on the text
- Deduplicates articles across runs (never saves the same article twice)
- Saves everything to a structured Google Sheet database
- Sends a formatted morning brief to Telegram at 7 AM
- Sends a weekly Quiz every Sunday with the week's best MCQs
- Costs $0 to run

### Tools Chosen
| Tool | Reason |
|------|--------|
| n8n Cloud | Free automation platform, visual workflow |
| Google Gemini Flash Latest | Primary AI — best free AI API, 1,500 req/day |
| Groq (Llama 3.1) | Fallback AI — 14,400 req/day free, ultra fast |
| Google Sheets | Free database, easy to read/write |
| Telegram | Free bot API, perfect for morning briefs and quizzes |

---

## 2. Pre-requisites & Accounts Setup

Before building VEDA, you need these accounts:
- **n8n Cloud account** → [app.n8n.cloud](https://app.n8n.cloud) (free tier)
- **Google account** → for Sheets + Gemini API
- **Telegram account** → for bot + delivery
- **Groq account** → [console.groq.com](https://console.groq.com) (free tier, for AI fallback)

---

## 3. Creating the Telegram Bot

1. Open Telegram and search for **@BotFather**.
2. Type `/newbot` and name it `VEDA`.
3. Give it a username like `VEDA_CurrentAffairs_bot`.
4. Save the **Bot Token** provided by BotFather (e.g., `8825496944:AAH6...`).

## 4. Getting Your Telegram Chat ID

1. Start a chat with your new bot and send a message.
2. Open `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` in your browser.
3. Find `"chat": {"id": 7781245199}` in the JSON response.
4. Save the **Chat ID**.

---

## 5. Setting Up Google Sheets Database

1. Create a new spreadsheet named **"VEDA_Current_Affairs_Database"**.
2. In **Row 1**, exactly create these headers:
   `ID`, `Date`, `Category`, `Headline`, `Source`, `Link`, `Summary`, `Key Points`, `Exam Score`, `MCQ 1`, `Options 1`, `Answer 1`, `MCQ 2`, `Options 2`, `Answer 2`, `Used In Quiz`
3. Create two additional tabs: **`Trends`** and **`Error Log`**.
4. In n8n, connect your Google account using the **Google Sheets OAuth2 API** credential.

---

## 6. Getting a Gemini API Key

1. Go to [aistudio.google.com](https://aistudio.google.com).
2. Create an API key. 
3. This is used in the HTTP Request node pointing to `gemini-flash-latest`.

## 7. Getting a Groq API Key (Fallback AI)

1. Go to [console.groq.com](https://console.groq.com).
2. Create an API key.
3. This is used for the `llama-3.1-8b-instant` model if Gemini hits rate limits.

---

## 8. Building the n8n Workflow — Step by Step

### Phase 1: Schedule Trigger & RSS Feeds
- **Trigger:** Set to run every day at 7:00 AM IST.
- **RSS Feeds:** 10 HTTP/RSS nodes reading from The Hindu, Livemint, NDTV, India Today, etc.
- **Merge Nodes:** 2 layers of Merge nodes (Append mode) combine all articles into one list.

### Phase 2: AI Filter & Deduplication
- A Code node filters articles using keywords (`UPSC, supreme court, RBI, economy, election...`).
- It uses `$getWorkflowStaticData('global')` to track URLs already processed so it never repeats news.
- Prepares the `geminiBody` for the API.

### Phase 3: The Primary AI (Gemini)
- **HTTP Request** node sends the aggregated news to Gemini.
- The prompt asks Gemini to extract headlines, summarize, assign an `Exam Score` out of 10, and generate 2 MCQs with options and text answers.
- **Error Node routing:** Configured to "Continue On Error" to trigger the fallback!

### Phase 4: The Fallback AI (Groq)
- If Gemini errors out, the workflow routes to a **Groq Setup** node which reads the exact same articles from `$getWorkflowStaticData`.
- Groq executes the prompt and a **Normalize** node converts Groq's output format to match Gemini's.

### Phase 5: Quality Gate & Database
- **Filter Node:** Drops any article where `Exam Score` < 7.
- **Google Sheets Node:** Uses `autoMapInputData` to drop the JSON exactly into the matching columns in Row 2.

### Phase 6: Telegram Daily Brief
- A Code node formats the top 7 articles into a clean Telegram Markdown message.
- Emojis, scores, and the `Answer 1` are neatly formatted.
- The **Telegram Node** fires it off to your phone!

### Phase 7: The Sunday Weekly Quiz
- A separate Schedule Trigger runs on Sundays.
- **Google Sheets (Read):** Pulls all articles.
- **Filter:** Keeps only articles from the last 7 days.
- **AI Formatting:** Groq/Gemini takes the top MCQs and generates a Weekly Review text.
- **Anti-Crash Failsafe:** The code completely strips Markdown asterisks (`*`) from the AI output before sending to Telegram to prevent strict parsing crashes.

---

## 9. Key AI Prompts & Formatting

### The MCQ Generation Prompt
To get perfect multiple-choice questions without Markdown breaking, the AI is prompted strictly:
> `answer_1 (exact text of correct option, NOT A/B/C/D), mcq_2, options_2, answer_2 (exact text of correct option, NOT A/B/C/D).`

### Telegram Crash Failsafe
Telegram's `Markdown (Legacy)` parse mode instantly crashes if it receives a single unclosed `*`. To prevent the Weekly Quiz from failing when the AI hallucinates bold tags, we scrubbed them out in the formatting node:
```javascript
// Strip ALL asterisks to prevent Telegram Markdown crash
msg += aiText.replace(/\*/g, '');
```

---

## 10. Final Workflow State

VEDA is a **38-node master workflow**.

It successfully combines:
- Data Aggregation (RSS)
- AI Intelligence (Gemini/Groq)
- Database Storage (Sheets)
- Notification Delivery (Telegram)
- Resilient Error Handling & Fallbacks

**Result:** A fully autonomous EdTech pipeline running $0/month.
