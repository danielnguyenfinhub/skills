---
name: finhub-annual-review-converter
description: Annual mortgage review converter for Finance Hub. ALWAYS use when Daniel says "annual review for [client]", "12 month review", "mortgage review [client]", "review call [client]", "[client] anniversary", "6 month review", "18 month review", "24 month review", "post-settlement review", "/review", or any request to conduct or prepare a structured mortgage review for a settled client. Pulls full client 360 from Mercury, identifies rate gaps + equity + expiries + new opportunities, produces broker meeting brief + client invitation email + calendar event + Mercury task. Covers 6/12/18/24/36 month milestones.
license: Complete terms in LICENSE.txt
---

# FinHub Annual Review Converter

Transforms a review trigger into a fully prepared broker meeting — client brief, invitation, calendar, task — with multiple conversion opportunities identified.

## Triggers
`/review` | "annual review for [client]" | "12 month review" | "mortgage review [client]" | "review call [client]" | "[client] anniversary" | "6 month review" | "18 month review" | "24 month review" | "post-settlement review"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_customer_all_loans` → `get_customer_assets_liabilities` → `get_customer_employment_income` → `get_customer_living_expenses` → `read_client_loan_products` → `get_customer_notes` → `create_opportunity_task` → `update_opportunity_next_action` → `add_opportunity_note`
- **Google Calendar:** `create_event` (optional)

## Execution Steps

### STEP 1 — Load Client Full 360
```
Tool: who_is(query) → contact_id, active opp_id, settlement_date
Tool: get_customer_all_loans(query) → all loan history
Tool: get_customer_assets_liabilities(query) → properties, equity
Tool: get_customer_employment_income(query) → income, employment
Tool: read_client_loan_products(query) → current rate, IO/fixed details
Tool: get_customer_notes(query) → any existing broker notes / flags
```

### STEP 2 — Calculate Review Milestone
```
months_settled = (today - settlement_date) / 30
review_type = match closest milestone:
  3-7 months:    "6 Month Review"
  10-14 months:  "12 Month Review"
  16-20 months:  "18 Month Review"
  22-26 months:  "2 Year Review"
  34-38 months:  "3 Year Review"

upcoming_review = next milestone date for scheduling
```

### STEP 3 — Opportunity Scan (7 checks)
```
Run all 7 checks and flag each as: ✅ No action | ⚠️ Monitor | 🔴 Action needed

CHECK 1 — Rate Competitiveness:
  Compare current_rate vs market_rate for same lender
  Flag 🔴 if gap >= 0.30%
  → Link: finhub-rate-review-converter

CHECK 2 — Equity Available:
  usable_equity = (property_value × 0.80) - current_debt
  Flag 🔴 if usable_equity >= $100,000
  → Link: finhub-equity-release-converter

CHECK 3 — Fixed Rate Expiry:
  Check fixed_rate_expiry within next 12 months
  Flag 🔴 if expiring within 90 days
  → Link: finhub-fixed-rate-expiry-converter

CHECK 4 — IO Expiry:
  Check io_expiry within next 12 months
  Flag 🔴 if expiring within 180 days
  → Link: finhub-io-expiry-converter

CHECK 5 — Clawback Safe:
  months_settled >= 12 → ✅ SAFE to discuss refinance
  months_settled < 12  → ⚠️ Clawback risk — no refi discussion

CHECK 6 — Life Changes:
  Check employment type change since settlement
  Check if income increased (capacity for investment)
  Flag ⚠️ if significant change noted in last notes

CHECK 7 — Investment Opportunity:
  IF equity >= $100K AND gross_income >= $100K:
  Flag 🔴 as investment loan candidate
  → Link: finhub-upsell-investment-converter
```

### STEP 4 — Draft Broker Meeting Brief
```
One-page internal brief for Daniel / broker to use on call

FORMAT:
  CLIENT SNAPSHOT: name, loan, lender, rate, settled date
  
  OPPORTUNITY ALERTS:
    [list each 🔴 finding with $ figure and recommended action]
  
  CONVERSATION AGENDA (15 min):
    1. Check-in: "How has the year been? Any life changes?"
    2. [🔴 Finding 1]: "I noticed your rate may be X.XX% above market..."
    3. [🔴 Finding 2]: "Your property may have $X in usable equity..."
    4. [🔴 Finding 3]: "Your fixed rate expires in N months..."
    5. Next steps: book follow-up for any actions
  
  NOTES FROM FILE:
    [relevant broker notes from Mercury]
  
  DATA QUALITY FLAGS:
    [missing income / no email / outdated ID etc]
```

### STEP 5 — Draft Client Review Invitation Email
```
Apply: mutual-finhub-email-writer
Timing: send 2 weeks before target review date

Subject: "It's been [N] months — time for your complimentary mortgage review"

Structure:
  1. MILESTONE: celebrate N months / X years
  2. WHY NOW: market has changed, rates moved, equity grown
  3. WHAT WE'LL COVER: rate check, equity position, goals
  4. VALUE: "clients who do regular reviews typically save..."
     ASIC: "may benefit from" not "save you thousands"
  5. CTA: book 15-min call OR reply to confirm time
  
Personalise with: settlement milestone + known life context
```

### STEP 6 — Mercury Updates
```
Tool: create_opportunity_task(
  subject = "[N]-Month Review — [Client] — [date]",
  due_date = review_due_date,
  priority = "normal",
  notes = "[Opportunity alerts from Step 3]
           Rate gap: [X]  |  Equity: $[X]  |  Fixed expiry: [date]
           Conversation agenda: [3 bullet points]"
)

Tool: update_opportunity_next_action(
  query = lender_ref,
  next_action_date = review_due_date,
  note = "[N]-Month review scheduled"
)

Tool: add_opportunity_note(
  note = "[Date] [N]-Month review initiated.
          Opportunity flags: [list].
          Invitation email sent to [email]."
)

Optional — Google Calendar:
  Google Calendar:create_event(
    title = "[N]-Month Review — [Client]",
    date = review_date,
    description = "Loan: $X [Lender] | Rate gap: X% | Equity: $X"
  )
```

## Output Format
```
MORTGAGE REVIEW — [CLIENT NAME] — [N]-MONTH MILESTONE
─────────────────────────────────────────────────────
LOAN SUMMARY
[Lender] $[X] @ X.XX%  |  Settled [date] ([N] months)
[Property address]  |  Est. value: $X  |  Current LVR: X%

OPPORTUNITY SCAN
🔴 Rate gap: X.XX% above market → $X,XXX/yr estimated saving
🔴 Usable equity: $X,XXX → investment or renovation possible
⚠️ Fixed rate: expiring [date] — plan in [N] months
✅ IO: Variable — no expiry concern
✅ Clawback: SAFE — [N] months settled

BROKER BRIEF — CONVERSATION AGENDA
1. Check-in (2 min)
2. Rate review — X.XX% vs X.XX% market ($X saving) (5 min)
3. Equity conversation — $X available (5 min)
4. Next steps + book follow-ups (3 min)

ACTION 1 — Broker meeting brief [above]
ACTION 2 — Client review invitation email [ready to send]
[full email]
ACTION 3 — Mercury task: "[N]-Month Review — [date]" [created]
ACTION 4 — Calendar event created [if confirmed]
ACTION 5 — Next action updated in Mercury
```

## ASIC Compliance Rules
- "Complimentary mortgage review" — never "free service"
- Savings: "may benefit from", "estimated saving of approximately"
- Rate comparison: conditional and non-guaranteed language throughout
- Investment discussion: "general information only — consider your circumstances"
