---
name: finhub-financial-passport-converter
description: Financial Passport (CDR) invitation converter for Finance Hub. ALWAYS use when Daniel says "FP invite for [client]", "send financial passport to [client]", "financial passport campaign", "CDR invite [client]", "activate FP for [client]", "financial passport [client]", "/fp", or any request to send or manage a Frollo Financial Passport invitation. Checks if FP already sent (avoids duplicate), identifies best conversation hook per client (rate/equity/expiry), drafts ASIC-compliant FP invitation email, updates Mercury status to 9.0, creates 7-day follow-up task. Integrates with the CDR campaign workflow (statuses 9.0-9.2).
license: Complete terms in LICENSE.txt
---

# FinHub Financial Passport Converter

Converts a client contact into a Financial Passport activation — with the right hook, right timing, and full Mercury workflow.

## Triggers
`/fp` | "FP invite for [client]" | "send financial passport to [client]" | "financial passport campaign" | "CDR invite [client]" | "activate FP for [client]" | "financial passport [client]"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_customer_all_loans` → `read_client_loan_products` → `get_customer_assets_liabilities` → `get_customer_contact_details` → `update_opportunity_status` → `create_opportunity_task` → `add_opportunity_note`

## Execution Steps

### STEP 1 — Load Client Context
```
Tool: who_is(query) → contact_id, opp_id, current_status, email, mobile
Tool: get_customer_all_loans(query)
→ settlement_date, lender, current_rate, loan_amount
Tool: read_client_loan_products(query)
→ fixed_expiry, io_expiry, current_rate
Tool: get_customer_assets_liabilities(query)
→ property_value, current_debt → usable_equity
```

### STEP 2 — Pre-flight Checks
```
CHECK 1 — Already invited?
  IF current_status IN ["9.0", "9.1", "9.2"]:
    STOP → "FP already sent on [date]. Status: [X].
             Do you want to send a follow-up instead?"
    → Offer: draft follow-up email + chase task

CHECK 2 — Do Not Mail?
  Tool: get_customer_contact_details(query) → doNotMail flag
  IF doNotMail = true: STOP → flag to broker

CHECK 3 — Email exists?
  IF no email on file: STOP → "No email address. 
    Update in Mercury first or send via SMS."

CHECK 4 — Time since last contact?
  IF last_updated < 30 days ago: WARN → "Recent contact [date].
    Recommend waiting before FP invite."

CHECK 5 — Eligible status?
  Ideal statuses for FP: 2.3, 8.1, 8.2 (Follow-up / Settled)
  Avoid: 12, 13, 14 (Closed / Declined / Not Proceeding)
```

### STEP 3 — Hook Identification
```
Score each hook based on client data:

HOOK A — Rate Review (strongest for settled clients):
  Trigger: rate_gap >= 0.25% vs market
  Hook: "We may be able to find you a better rate"
  Benefit: "See if there's a more competitive option for your [Lender] loan"

HOOK B — Equity Release:
  Trigger: usable_equity >= $100,000
  Hook: "See how much equity your property may have unlocked"
  Benefit: "Understand your borrowing power for your next move"

HOOK C — Fixed Rate Expiry:
  Trigger: fixed_expiry within 12 months
  Hook: "Your fixed rate expires [month] — plan ahead"
  Benefit: "Get ahead of your options before the expiry date"

HOOK D — IO Expiry:
  Trigger: io_expiry within 18 months
  Hook: "Your interest-only period ends [month]"
  Benefit: "Understand your options before your repayments change"

HOOK E — New to settled (< 3 months):
  Hook: "Now that you're settled, get your full financial picture"
  Benefit: "See your equity position and what you could do next"

HOOK F — General (no specific trigger):
  Hook: "Get a complete picture of your financial position"
  Benefit: "Our complimentary Financial Passport shows you everything in one place"

SELECT: highest-scoring hook → use in email subject + opening
```

### STEP 4 — Draft FP Invitation Email
```
Apply: mutual-finhub-email-writer (AIDA + Cialdini + Voss)
ASIC RG 234: all benefits conditional

Subject options by hook:
  Rate:    "Quick question about your [Lender] rate, [First Name]"
  Equity:  "[First Name], your property may have more equity than you think"
  Fixed:   "Your fixed rate expires [month] — let's plan ahead, [First Name]"
  IO:      "Important: your interest-only period ends [month], [First Name]"
  General: "A complimentary financial review for you, [First Name]"

Body structure:
  HOOK (1 sentence — the specific insight for this client)
  BRIDGE (what the Financial Passport does — 2 sentences)
  PROOF (what other clients discovered — social proof)
  CTA (single link OR reply to request)
  SIGNATURE (exact: Kind regards, Daniel Nguyen / Finance Hub and Networks / 
             Mob: 0430 11 11 88 / Email: daniel@finhub.net.au)

FP explanation (ASIC safe):
  "Our Financial Passport connects securely to your bank accounts 
   via Australia's Consumer Data Right (CDR) framework, giving you 
   and us a complete picture of your financial position — 
   entirely at your discretion."

NEVER: "free service" → "complimentary review"
NEVER: "save you thousands" → "may benefit from a review"
```

### STEP 5 — Mercury Workflow Updates
```
Tool: update_opportunity_status(
  query = lender_ref,
  new_status = "9.0",   ← or use correct Mercury status name
  note = "FP invitation sent [date]. Hook: [hook type]."
)

Tool: create_opportunity_task(
  subject = "Chase FP Response — [Client]",
  due_date = today + 7 days,
  priority = "normal",
  notes = "FP invite sent [date] via email [address].
           Hook used: [hook type].
           If no response in 7 days — send SMS follow-up."
)

Tool: add_opportunity_note(
  note = "[Date] Financial Passport invitation sent.
          Hook: [X]. Email: [address].
          Next: chase response [date + 7]."
)
```

### STEP 6 — Follow-up Sequence (if no response in 7 days)
```
Day 7:  SMS — "Hi [Name], just following up on the financial review 
               I sent last week. Happy to walk you through it — 
               5 mins on the phone? — Daniel 0430 11 11 88"

Day 14: Email — second touch with different hook angle

Day 21: Final touch — "Closing the loop" approach (Voss: label the resistance)
```

## Output Format
```
FINANCIAL PASSPORT INVITE — [CLIENT NAME]
─────────────────────────────────────────────────────
PRE-FLIGHT CHECKS
✅ Not previously invited  ✅ Email on file  ✅ No DNM flag
✅ Status eligible: [current status]

HOOK SELECTED: [Hook A/B/C/D/E/F] — [reason]
  "[one-line hook description]"

ACTION 1 — FP Invitation Email [ready to send]
Subject: [selected subject line]
[full email body]

ACTION 2 — Mercury status → 9.0 FP Invite Sent [updated]
ACTION 3 — Task: "Chase FP Response — [date + 7 days]" [created]
ACTION 4 — Note added to opportunity

FOLLOW-UP SEQUENCE:
  Day 7:  SMS template ready
  Day 14: Email follow-up ready
  Day 21: Final touch ready
```

## ASIC Compliance Rules
- "Complimentary Financial Passport" — never "free"
- CDR explanation required — always include data sharing consent note
- Benefits: "may help you" / "could potentially" / "subject to your situation"
- Never imply guaranteed outcome from completing the passport
