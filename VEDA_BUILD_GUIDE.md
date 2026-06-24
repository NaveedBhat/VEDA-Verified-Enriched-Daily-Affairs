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
9. [All Code Used](#9-all-code-used)
10. [Key Design Decisions & Lessons Learned](#10-key-design-decisions--lessons-learned)
11. [Final Workflow State](#11-final-workflow-state)

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

### Constraints
- **No paid APIs** — everything must be free tier
- **No coding environment** — build entirely inside n8n's UI
- **Fully automated** — runs without human intervention

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

### Accounts Required
- [ ] **n8n Cloud account** → [app.n8n.cloud](https://app.n8n.cloud) (free tier)
- [ ] **Google account** → for Sheets + Gemini API
- [ ] **Telegram account** → for bot + delivery
- [ ] **Groq account** → [console.groq.com](https://console.groq.com) (free tier, for AI fallback)

### n8n Setup
1. Go to [app.n8n.cloud](https://app.n8n.cloud)
2. Create a free account
3. Click **"New Workflow"**
4. Name it: `VEDA`

---

## 3. Creating the Telegram Bot

A Telegram Bot is what VEDA uses to send messages to you. Think of it as VEDA's voice.

### Step-by-Step: Create Bot with BotFather

1. Open Telegram (app or web)
2. Search for **@BotFather** (official Telegram bot creator)
3. Start a chat → click **START**
4. Type: `/newbot`
5. BotFather asks: *"Alright, a new bot. How are we going to call it?"*
6. Type the display name: `VEDA`
7. BotFather asks: *"Now let's choose a username for your bot"*
8. Type a unique username ending in `bot`: `VEDA_CurrentAffairs_bot`
9. BotFather responds with your **Bot Token**.

> ⚠️ **Save this token — you need it in n8n.**

---

## 4. Getting Your Telegram Chat ID

The Chat ID tells the bot WHERE to send messages (your personal chat).

### Method: Use the Telegram API

1. Open your browser
2. First, **start a chat with your bot**: Search your new bot in Telegram → click START
3. Send any message to the bot (e.g., "hello")
4. Open this URL in browser:
   `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
5. You'll see JSON response. Locate the `chat.id` value (e.g., **`7781245199`**)

> ✅ Save this — you'll use it in the Telegram n8n node.

---

## 5. Setting Up Google Sheets Database

### Creating the Sheet

1. Go to [sheets.google.com](https://sheets.google.com)
2. Create a new spreadsheet named: **"VEDA_Current_Affairs_Database"**
3. In **Row 1**, exactly create these headers (one per column):

| A | B | C | D | E | F | G | H | I | J | K | L | M | N | O | P |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ID | Date | Category | Headline | Source | Link | Summary | Key Points | Exam Score | MCQ 1 | Options 1 | Answer 1 | MCQ 2 | Options 2 | Answer 2 | Used In Quiz |

4. Save the spreadsheet and copy the Sheet ID from the URL.

### Additional Sheet Tabs Required

#### Trends Tab
1. Click `+` at the bottom of the sheet → rename to: **`Trends`**
2. Add these headers in Row 1:
   `Week`, `Date`, `Total Articles`, `Avg Exam Score`, `Top Category`, `Category Breakdown`

#### Error Log Tab
1. Click `+` again → rename to: **`Error Log`**
2. Add these headers in Row 1:
   `Timestamp`, `Node Name`, `Error Type`, `Error Message`, `Full Error Details`, `Status`

---

## 6. Getting a Gemini API Key

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Click **"Get API Key"** and create a new key.
3. This is used in the HTTP Request node pointing to `gemini-flash-latest`.

## 7. Getting a Groq API Key (Fallback AI)

Groq is a lightning-fast AI inference platform with a generous free tier (14,400 requests/day). VEDA uses it as an automatic fallback if Gemini fails.

1. Go to [console.groq.com](https://console.groq.com)
2. Click **"API Keys"** → **"Create API Key"**
3. Name it: `VEDA-Fallback`

---

## 8. Building the n8n Workflow — Step by Step

### Phase 1: Schedule Trigger
**Node:** Schedule Trigger  
**Settings:** Runs every day at 7:00 AM IST.

### Phase 2: Add 10 RSS Feed Nodes
**Node type:** RSS Feed Read  
Add 10 sources covering Indian current affairs (The Hindu, Livemint, NDTV, etc.).

### Phase 3: Merge Nodes
n8n Merge nodes support a maximum of 10 inputs. We funnel all feeds into two Merge nodes (Append mode) and then combine them.

### Phase 4: Code Node — Filter + Dedup
**Node:** Code  
1. Takes all raw articles.
2. Filters out irrelevant news using keywords (`UPSC, supreme court, RBI, economy, election, foreign policy...`).
3. Deduplicates by URL using `$getWorkflowStaticData('global')`.
4. Pre-builds the `geminiBody`.

### Phase 5: HTTP Request Node (Gemini AI — Primary)
**Node:** HTTP Request  
**Method:** POST  
**URL:** `https://generativelanguage.googleapis.com/v1beta/models/gemini-flash-latest:generateContent?key=YOUR_KEY`  
**Settings:** Retry on Fail: ON, **On Error: Continue (use error output)** ← CRITICAL for fallback routing!

### Phase 6: Code Node — Parse Gemini Response
Extracts the JSON from the LLM text output and transforms it into the structured format for Google Sheets.

### Phase 7: Groq Fallback Chain (Error Path)
- **Groq Setup Node:** Reads articles from `staticData` and builds a Groq-compatible JSON body.
- **HTTP Request (Groq):** POST to `api.groq.com` using the `llama-3.1-8b-instant` model.
- **Normalize Groq:** Converts the Groq response to match Gemini's structure, allowing it to seamlessly merge back into the main pipeline.

### Phase 8: Filter Node (Quality Gate)
**Condition:** `Exam Score >= 7`  
Only exam-relevant articles pass this point.

### Phase 9: Google Sheets — Save Data
**Node:** Google Sheets  
**Operation:** Append Row  
**Mapping Mode:** Auto-Map Input Data (works flawlessly because headers are neatly on Row 1).

### Phase 10: Format Daily Brief & Telegram
**Format Node:** Builds a beautiful Markdown message. Appends `Answer 1` right under `MCQ 1`.
**Telegram Node:** Sends the message via `Markdown (Legacy)`.

### Phase 11: Weekly Quiz Chain
- Triggers separately on Sundays.
- **Read All Articles:** Uses the `read` operation to pull data from Google Sheets.
- **Filter:** Keeps articles from the last 7 days.
- **AI Formatting:** Groq/Gemini reviews the top MCQs and summarizes them.
- **Format Weekly Quiz:** Includes an anti-crash script to strip `*` characters generated by the AI to protect Telegram's parser.

---

## 9. All Code Used

### Code Node: Setup Groq AI Request
```javascript
const staticData = $getWorkflowStaticData('global');
const articlesText = staticData.vedaArticlesText || '';
const articles = staticData.vedaArticlesList || [];

let prompt = 'You are VEDA, curating current affairs for UPSC, SSC, Banking exam preparation.\n';
prompt += 'Analyze these articles and return ONLY a JSON array, no markdown.\n\n';
prompt += articlesText;
prompt += '\n\nReturn array of objects with: index, headline, link, category (Polity/Economy/Science & Tech/Environment/International Relations/Sports/Awards & Honours/Defense & Security/Social Issues/Appointments), exam_score (1-10), summary (2 sentences), key_points (3 by |), mcq_1, options_1, answer_1 (exact text of correct option, NOT A/B/C/D), mcq_2, options_2, answer_2 (exact text of correct option, NOT A/B/C/D).';

const groqBody = JSON.stringify({
  model: 'llama-3.1-8b-instant',
  messages: [{role: 'user', content: prompt}],
  max_tokens: 3000
});

return [{json: {groqBody, articles}}];
```

### Code Node: Format Daily Brief (Telegram)
```javascript
const items = $input.all();
if (items.length === 0) return [{json: {message: '📚 VEDA: No significant current affairs found today.'}}];

const today = new Date().toLocaleDateString('en-IN', {weekday: 'long', year: 'numeric', month: 'long', day: 'numeric', timeZone: 'Asia/Kolkata'});
const catEmoji = {'Polity':'🏛️','Economy':'💰','Science & Tech':'🔬','Environment':'🌿','International Relations':'🌍','Sports':'🏆','Awards & Honours':'🎖️','Defense & Security':'⚔️','Social Issues':'👥','Appointments':'👤'};

const getFirstKeyPoint = (kp) => {
  if (Array.isArray(kp)) return (kp[0] || '').trim();
  return (String(kp || '').split('|')[0] || '').trim();
};

const safeStr = (v) => Array.isArray(v) ? v.join(', ') : String(v || '');

let msg = '📚 *VEDA — Daily Current Affairs*\n';
msg += '📅 ' + today + '\n\n';
msg += '📊 *' + Math.min(items.length, 7) + ' Key Affairs Today*\n';
msg += '━━━━━━━━━━━━━━━━━━━━\n\n';

const emojis = ['1️⃣','2️⃣','3️⃣','4️⃣','5️⃣','6️⃣','7️⃣'];

items.slice(0, 7).forEach((item, i) => {
  const score = Number(item.json['Exam Score'] || 0);
  const fire = score >= 9 ? '🔥' : '⭐';
  const cat = safeStr(item.json.Category) || 'General';
  const emoji = catEmoji[cat] || '📰';
  msg += emojis[i] + ' *' + safeStr(item.json.Headline).substring(0, 85) + '*\n';
  msg += fire + ' ' + score + '/10 | ' + emoji + ' ' + cat + '\n';
  msg += '📝 ' + safeStr(item.json.Summary).substring(0, 130) + '\n';
  msg += '🔑 ' + getFirstKeyPoint(item.json['Key Points']) + '\n';
  msg += '❓ _' + safeStr(item.json['MCQ 1']).substring(0, 90) + '_\n';
  msg += '💡 *Ans:* ' + safeStr(item.json['Answer 1']) + '\n';
  msg += '🔗 [Read More](' + safeStr(item.json.Link) + ')\n\n';
});

msg += '━━━━━━━━━━━━━━━━━━━━\n';
msg += '📖 _Full MCQs & answers saved in VEDA Sheet_\n';
msg += '🤖 _Powered by VEDA_\n';
msg += '👨‍💻 _Developed by Naveed Bhat_';

return [{json: {message: msg}}];
```

### Code Node: Format Weekly Quiz & Anti-Crash Failsafe
```javascript
const input = $input.first();
const articles = input.json.articles || [];
const aiText = (input.json.candidates?.[0]?.content?.parts?.[0]?.text || '').substring(0, 3500);

const today = new Date().toLocaleDateString('en-IN', {weekday: 'long', year: 'numeric', month: 'long', day: 'numeric', timeZone: 'Asia/Kolkata'});

let msg = '📚 *VEDA Weekly Quiz & Current Affairs Review*\n';
msg += '📅 ' + today + '\n';
msg += '━━━━━━━━━━━━━━━━━━━━\n\n';

if (aiText) {
  msg += aiText.replace(/\*/g, ''); // Strip ALL asterisks to prevent Telegram Markdown crash
} else {
  articles.slice(0, 5).forEach((a, i) => {
    const e = ['1️⃣','2️⃣','3️⃣','4️⃣','5️⃣'][i];
    msg += e + ' *' + (a.Headline || '').substring(0, 80) + '*\n';
    msg += '⭐ ' + (a['Exam Score'] || 0) + '/10 | ' + (a.Category || '') + '\n';
    msg += '❓ ' + (a['MCQ 1'] || '') + '\n✅ ' + (a['Answer 1'] || '') + '\n\n';
  });
}

msg += '\n━━━━━━━━━━━━━━━━━━━━\n';
msg += '📖 _Complete MCQ bank in VEDA Sheet_\n';
msg += '🤖 _VEDA Weekly Intelligence_\n';
msg += '👨‍💻 _Developed by Naveed Bhat_';

return [{json: {message: msg}}];
```

---

## 10. Key Design Decisions & Lessons Learned

### Forcing AI to Answer Directly (Not A/B/C/D)
Initially, the AI responded to MCQs with standard lettering like "A" or "B". This provided a poor user experience on Telegram. By aggressively appending `(exact text of correct option, NOT A/B/C/D)` to the system prompt, we forced the LLM to output the actual text, drastically improving the utility of the Telegram message for studying students.

### Surviving Telegram's `Markdown (Legacy)`
Telegram's markdown parser is famously fragile. A single unclosed `*` or a hallucinated `***` from the LLM will crash the entire node with the error `Can't find end of the entity starting at byte offset...`. 

To prevent this in the Weekly Quiz:
1. We modified the LLM prompt: `Do NOT use markdown or asterisks (*). Plain text only.`
2. We built a failsafe Regex in the formatting node: `aiText.replace(/\*/g, '')`
This guarantees 100% stability.

### The "Auto Map" Sheets Advantage
Unlike ARIA which used tedious manual schema mapping for Google Sheets, VEDA was designed with clean headers directly on Row 1 (no merged title cells). This allowed us to utilize n8n's `autoMapInputData`, reducing the Google Sheets node configuration to practically zero and completely eliminating shift errors.

---

## 11. Final Workflow State

VEDA is a **38-node master workflow**.

It successfully combines:
- Data Aggregation (RSS)
- AI Intelligence (Gemini/Groq)
- Database Storage (Sheets)
- Notification Delivery (Telegram)
- Resilient Error Handling & Fallbacks

**Result:** A fully autonomous EdTech pipeline running $0/month.
