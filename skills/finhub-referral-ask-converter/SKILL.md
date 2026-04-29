---
name: finhub-referral-ask-converter
description: Referral request converter for Finance Hub. ALWAYS use when Daniel says "ask [client] for a referral", "referral ask [client]", "request referral from settled client", "referral email [client]", "can [client] refer someone", "/referral", or any request to ask a settled client for a referral or Google Review. Confirms client is in settled status (8.1/8.2), checks if Google Review already sent (avoids duplicate), identifies referral hook (FHB/investor/refinancer), produces referral email + SMS follow-up + Mercury task + category tag. ASIC RG 234 compliant.
license: Complete terms in LICENSE.txt
---

# FinHub Referral Ask Converter

Turns a happy settled client into a referral source — with the right ask, right timing, and full Mercury tracking.

## Triggers
`/referral` | "ask [client] for a referral" | "referral ask [client]" | "request referral from settled client" | "referral email [client]" | "can [client] refer someone"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_customer_all_loans` → `get_customer_contact_details` → `get_customer_notes` → `get_customer_audit_trail` → `create_opportunity_task` → `add_opportunity_note` → `tag_contact_category`

## Execution Steps

### STEP 1 — Load Client Context
```
Tool: who_is(query) → contact_id, opp_id, status, settlement_date
Tool: get_customer_all_loans(query)
→ settled loan details, lender, loan_amount, transaction_type
Tool: get_customer_contact_details(query)
→ email, mobile, doNotMail flag
Tool: get_customer_notes(query)
→ any referral already requested? any client relationship notes?
Tool: get_customer_audit_trail(query)
→ scan for Google Review request (auto-sent at status 8.1)
```

### STEP 2 — Pre-flight Checks
```
CHECK 1 — Correct status?
  MUST BE: Settled (status 8.1/8.2/Settled/Post-Settlement)
  IF not settled: STOP → "Client not yet settled. 
    Schedule referral ask for 30 days post-settlement."

CHECK 2 — Settlement timing (optimal window)?
  IDEAL: 2–6 months post-settlement (honeymoon period)
  ALSO GOOD: 12-month anniversary (major milestone)
  TOO EARLY: < 60 days (still in post-settlement admin)
  LATE BUT OK: > 12 months (still worth asking, adjust tone)

CHECK 3 — Google Review already sent?
  Scan audit_trail for status change to 8.1 → 
  Mercury auto-fires Google Review request at 8.1
  IF found: WARN → "Google Review auto-sent on [date]. 
    Referral ask is separate — proceed."
  Flag to NOT ask for Google Review again in referral email
  (already done)

CHECK 4 — Referral already requested?
  Scan notes for "referral" keyword
  IF previous referral ask found: WARN → "Referral requested [date]. 
    Gap it by 6 months OR personalise the follow-up."

CHECK 5 — Do Not Mail / Do Not Contact?
  IF doNotMail: STOP and flag to broker
```

### STEP 3 — Referral Hook Identification
```
Based on client loan profile:

FHB (First Home Buyer):
  Hook: "Friends planning to buy their first home?"
  Script: "Many of my clients refer friends who are saving for their first home"

INVESTOR:
  Hook: "Other investors in your network?"
  Script: "Investors often know other investors looking to grow their portfolio"

REFINANCER:
  Hook: "Friends on a high rate with another lender?"
  Script: "Many people are still paying too much — a quick introduction could help"

UPGRADER / NEXT HOME:
  Hook: "Friends or family thinking about upsizing?"
  Script: "Life changes quickly — a lot of families are thinking about their next move"

VIETNAMESE COMMUNITY (if Vietnamese name/background):
  Hook: Bilingual approach — offer EN + VI message
  Reference: "cộng đồng của chúng ta" (our community)

DEFAULT:
  Hook: "Anyone in your circle thinking about property?"
```

### STEP 4 — Draft Referral Request Email
```
Apply: mutual-finhub-email-writer (AIDA + Cialdini: reciprocity + liking)
Tone: personal, appreciative, low-pressure

Subject options:
  FHB:       "[Name], do you know anyone saving for their first home?"
  Investor:  "[Name], know any investors looking for their next property?"
  General:   "[Name], a quick favour — do you know anyone thinking about property?"

Structure:
  1. APPRECIATION (2 sentences): genuine thanks for their business
  2. MILESTONE: celebrate what they achieved (settled, own their home)
  3. THE ASK (soft): "If you know anyone in a similar situation..."
  4. WHAT WE DO: "We'll look after them the same way we looked after you"
  5. HOW EASY IT IS: "Just forward this email or share my number"
  6. OPTIONAL INCENTIVE: (if referral program exists)
  7. CTA: single — "Reply with their name and I'll reach out"
  
AVOID:
  "Can you refer us?" — sounds transactional
  "We'd appreciate your business" — too formal
  
USE:
  "I'd love to help someone you care about"
  "It would mean a lot to us"
```

### STEP 5 — Draft SMS Follow-up (Day 7)
```
SMS (160 chars max):
  "Hi [Name], just following up on my recent email — 
   do you know anyone thinking about property? Happy to help. 
   — Daniel 0430 11 11 88"

Bilingual option (Vietnamese):
  "Chào [Name], nếu bạn biết ai đang tính mua nhà, 
   mình rất vui được giúp. — Daniel 0430 11 11 88"
```

### STEP 6 — Mercury Updates
```
Tool: create_opportunity_task(
  subject = "Referral Ask — [Client] — follow up [date+7]",
  due_date = today + 7 days,
  priority = "normal",
  notes = "Referral email sent [date].
           Hook: [type]. 
           SMS follow-up due [date+7] if no response."
)

Tool: add_opportunity_note(
  note = "[Date] Referral request sent.
          Hook: [X]. Email sent to [address].
          Follow-up SMS due [date+7]."
)

Tool: tag_contact_category(
  contacts = [contact_id],
  category_name = "Referral Source — Actioned"
)
```

## Output Format
```
REFERRAL ASK — [CLIENT NAME]
─────────────────────────────────────────────────────
SETTLED [N] months ago  |  [Lender] $[X]  |  [transaction type]
Settlement: [date]  |  Timing: [IDEAL / GOOD / LATE]

PRE-FLIGHT
✅ Settled status confirmed  
✅ Google Review: auto-sent [date] ← NOT asking again
✅ No previous referral ask on file
✅ Email on file: [address]

HOOK SELECTED: [type] — [description]

ACTION 1 — Referral email [ready to send]
Subject: [selected subject]
[full email body]

ACTION 2 — SMS follow-up [ready for Day 7]
[SMS text]

ACTION 3 — Mercury task: "Referral Ask follow-up — [date+7]" [created]
ACTION 4 — Note added to opportunity
ACTION 5 — Contact tagged: "Referral Source — Actioned"
```

## ASIC Compliance Rules
- Referral requests: no financial incentive implied unless formal referral program exists
- If referral fee/gift: must be disclosed to both referrer and referee
- Client contact: no unsolicited contact if doNotMail = true
- Community targeting (Vietnamese): same ASIC standards apply in all languages
