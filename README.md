# VEDA — Verified & Enriched Daily Affairs

<div align="center">
  <strong>🎓 Daily Current Affairs for Competitive Exam Preparation</strong><br/>
  UPSC · SSC · Banking · State PCS
</div>

## Overview
VEDA is an n8n automation that curates, analyzes, and delivers daily current affairs for competitive exam aspirants. It aggregates news from 10 Indian news RSS feeds, scores articles by exam relevance using AI, generates MCQs, and delivers a daily brief to Telegram.

## Features
- 📰 10 Indian news RSS feeds (The Hindu, India Today, Livemint, etc.)
- 🤖 Dual-AI analysis (Gemini → Groq fallback)
- 🎯 Exam relevance scoring (1–10)
- ❓ Auto-generated MCQs for each article
- 📱 Daily Telegram brief via @Veda085_bot
- 📊 Weekly quiz generation
- 📈 Trend tracking via Google Sheets
- 🛡️ Error handling & alert system

## Categories Covered
`Polity` · `Economy` · `Science & Tech` · `Environment` · `International Relations` · `Sports` · `Awards & Honours` · `Defense & Security` · `Social Issues` · `Appointments`

## Tech Stack
- **Automation**: n8n
- **Primary AI**: Google Gemini 2.0 Flash
- **Fallback AI**: Groq (Llama 3.1 8b Instant)
- **Storage**: Google Sheets
- **Delivery**: Telegram Bot API

## Setup
1. Import `VEDA.json` into your n8n instance
2. Add credentials:
   - Google Gemini API key (get from [aistudio.google.com](https://aistudio.google.com/apikey))
   - Groq API key (get from [console.groq.com](https://console.groq.com))
   - Google Sheets OAuth2
   - Telegram Bot token
3. Set `headerRow: 2` in all Google Sheets nodes
4. Activate workflow — runs daily at 7 AM IST

## Google Sheet Structure
| Column | Description |
|--------|-------------|
| ID | Unique article ID |
| Date | Publication date |
| Category | Exam category |
| Headline | Article headline |
| Source | News source |
| Link | Article URL |
| Summary | AI-generated summary |
| Key Points | 3 key facts |
| Exam Score | Relevance score (1–10) |
| MCQ 1 & 2 | Practice questions |

---
*Developed by Naveed Bhat*
