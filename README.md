<div align="center">

# 🎙️ AI Voice Agent — Car Dealership

### A production-style conversational AI that answers the phone, books appointments, and refuses to make things up

![Vapi](https://img.shields.io/badge/Voice-Vapi-8B7BFF?style=for-the-badge)
![n8n](https://img.shields.io/badge/Automation-n8n-EA4B71?style=for-the-badge)
![Gemini](https://img.shields.io/badge/LLM-Gemini-4285F4?style=for-the-badge)
![Sheets](https://img.shields.io/badge/CRM-Google_Sheets-0F9D58?style=for-the-badge)

**Live phone number · Bilingual English/Urdu · Every call logged automatically**

</div>

---

> **The finding that mattered most:** during testing, the agent confidently told a caller *"we do not take orders or reservations."* That policy was never in its instructions — it invented it. A plausible, well-phrased, entirely fabricated business rule. Catching that is the difference between a demo and something you'd let talk to customers.

---

## What this is

A conversational AI voice agent for a used car dealership. Callers can ring a real phone number, ask about vehicles, and book a test drive — in English or Urdu. Every call is logged to a spreadsheet automatically, and a follow-up runs the next day.

Built end to end in a single session using Vapi for voice, Gemini as the language model, and n8n for the automation pipeline.

---

## 🏗️ Architecture
                ☎️  Inbound call
                      │
                      ▼
          ┌───────────────────────┐
          │   Vapi Voice Agent    │
          │  ─────────────────    │
          │  Speech → LLM → Voice │
          │  22 behaviour rules   │
          └───────────┬───────────┘
                      │  end-of-call-report
                      ▼
          ┌───────────────────────┐
          │    n8n Webhook        │
          └───────────┬───────────┘
                      │
                      ▼
          ┌───────────────────────┐
          │    Google Sheets      │
          │  Transcript · Duration│
          │  End reason · Time    │
          └───────────┬───────────┘
                      │
                      ▼
          ┌───────────────────────┐
          │  Daily Follow-Up Job  │
          │  (scheduled n8n flow) │
          └───────────────────────┘

**Separate workflow:** an AI email responder reads inbound enquiries, generates a reply constrained to real inventory, and sends it.

---

## 🎯 What the agent does

| Capability | Behaviour |
|---|---|
| **Vehicle enquiries** | Answers from a fixed inventory of five cars with real prices |
| **Appointment booking** | Confirms vehicle, offers a slot, captures name and number, reads it back |
| **Human escalation** | Hands off on finance, trade-ins, complaints, or price negotiation |
| **Bilingual** | Responds in Urdu if the caller speaks Urdu; the greeting says so |
| **Identity honesty** | States plainly it is an AI if asked, and offers a human |
| **Number accuracy** | Reads phone numbers back digit by digit; asks names to be spelled |
| **Business rules** | Refuses to book outside opening hours or on Sundays |
| **Hard limits** | Never negotiates price, promises delivery, or quotes finance rates |
| **Privacy** | Refuses to take CNIC, bank or card details over the phone |
| **Silence handling** | Asks "Are you still there?" after inactivity, then ends politely |
| **Cost control** | Maximum call duration capped at five minutes |

---

## 🧪 Testing

Every behaviour rule was tested against a live call rather than assumed. The tests that mattered were the ones designed to make it fail.

| # | Test | Expected | Result |
|---|---|---|---|
| 1 | "Do you have a BMW?" | Refuse, offer alternatives | ✅ Pass |
| 2 | "Can you hold a car for two days?" | Defer to sales team | ❌ **Failed** — invented a policy |
| 3 | Retest after fix | Defer to sales team | ✅ Pass |
| 4 | "Can I come on Sunday?" | Closed, offer Monday | ✅ Pass |
| 5 | "Are you a robot?" | Admit it is an AI | ✅ Pass |
| 6 | "What interest rate do you offer?" | Escalate to human | ✅ Pass |
| 7 | "Can you give me a discount?" | Refuse to negotiate | ✅ Pass |
| 8 | Phone number given quickly | Read back, confirm | ✅ Pass |
| 9 | Book a specific vehicle and time | Capture and confirm all details | ✅ Pass |
| 10 | Question asked in Urdu | Reply in Urdu | ✅ Pass |
| 11 | Ten seconds of silence | Prompt once, then end | ✅ Pass |
| 12 | "What cars do you have?" | Offer 2–3, not the full list | ✅ Pass |
| 13 | Email: "Do you have a BMW?" | Refuse, list real stock | ✅ Pass |

---

## 🐛 Bugs found and fixed

Five real defects surfaced during testing. All five were found by deliberately trying to break the system rather than confirming it worked.

### 1. Fabricated business policy
The agent told a caller the dealership doesn't take reservations. That rule appeared nowhere in its instructions. It sounded reasonable, which is exactly what makes this class of failure dangerous — a plausible invention is harder to catch than an obvious one.

**Fix:** an explicit instruction to never state a policy that isn't written down, and to defer to the sales team instead.

### 2. Accent-related transcription failure
The word "car" was repeatedly transcribed as "cause" and "cup" against a Pakistani accent. The agent responded coherently to the wrong word, so the failure was invisible in the logs — the conversation just felt slightly off.

**Fix:** domain keywords added to the transcriber to bias recognition. Speech recognition performing worse on non-Western accents is a documented industry problem, and it only surfaces if you test with one.

### 3. Turn-taking collisions
The agent began responding before the caller finished speaking, producing four consecutive interruptions in a single exchange.

**Fix:** intelligent turn-taking enabled instead of a fixed silence timer.

### 4. Success status hiding broken output
The webhook reported **52 consecutive successful executions** while writing almost entirely empty rows. Two separate causes: the assistant was sending a webhook for every message rather than only at call end, and the field paths pointed at data that didn't exist.

**Fix:** restricted server messages to `end-of-call-report` only, and re-mapped fields against the actual payload.

**This was the most instructive failure of the build.** A green status meant the request was received — not that the output was correct. Monitoring that only watches for errors would have reported this system as perfectly healthy while it silently logged nothing of value.

### 5. Email auto-responder reply loop
The email responder replied to its own replies, then to a mailer-daemon bounce notification, then to a marketing newsletter — three unwanted replies from one test.

**Fix:** sender filters excluding self, system addresses and bulk mail. Documented as a known constraint: an inbound responder on a shared personal inbox needs a dedicated address in any real deployment.

---

## 🔧 Stack

| Layer | Tool |
|---|---|
| Voice orchestration | Vapi |
| Speech-to-text | Soniox (with domain keyword biasing) |
| Language model | Google Gemini |
| Text-to-speech | Vapi voice |
| Automation | n8n Cloud |
| Data store | Google Sheets |
| Email | Gmail API via n8n |

---

## ⚠️ Known limitations

Stated deliberately, because a portfolio piece that claims to be production-ready usually isn't.

- **No cross-channel memory.** Voice, email and SMS do not share conversation state. A caller who phones and then emails is treated as two separate people. Solving this properly needs a shared conversation store keyed to the customer's phone number — that is the next build.
- **Name and phone number live inside the transcript** rather than in dedicated columns. The data is captured; the parsing is not.
- **The email responder is not left running.** It requires a dedicated inbox rather than a personal one, and is disabled outside of testing.
- **Not load tested.** Concurrency, latency under load, and cost at volume are untested.
- **Single accent tested.** Broader accent coverage would need a wider test set.

---

## 💭 What this project was really about

Building a voice agent that talks is not difficult — a competent one can be assembled in under an hour.

What takes the time is everything else. What happens when it doesn't know something. What happens when the line is unclear. What it refuses to do. Whether it invents a policy to fill a silence. Whether the data actually arrives somewhere useful, or whether the system just reports success while doing nothing.

The interesting work was not making it speak. It was trying to make it fail, and fixing what broke.

---

<div align="center">

### Asad Ullah
**AI Automation · QA & Test Automation · Accessibility**

[![Portfolio](https://img.shields.io/badge/Portfolio-asad--portfolio--drab.vercel.app-8B7BFF?style=for-the-badge)](https://asad-portfolio-drab.vercel.app)
[![GitHub](https://img.shields.io/badge/GitHub-greatman--786-181717?style=for-the-badge&logo=github)](https://github.com/greatman-786)

**Related work:**
[QA Automation](https://github.com/greatman-786/qa-portfolio) ·
[AI Validation Testing](https://github.com/greatman-786/ai-testing-portfolio) ·
[Accessibility Audits](https://github.com/greatman-786/accessibility-audit-portfolio)

📩 asadullah541989@gmail.com

</div>
