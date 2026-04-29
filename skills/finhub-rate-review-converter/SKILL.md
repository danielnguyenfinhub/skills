---
name: finhub-rate-review-converter
description: Rate review and negotiation converter for Finance Hub. ALWAYS use when Daniel says "rate review for [client]", "can we do better on [client]'s rate", "what rate are they on", "negotiate [client]'s rate", "check [client]'s rate", "rate too high", "above market rate", "/rate-review", or any request to review, compare, or negotiate a client's existing interest rate. Pulls live loan data from Mercury V2, calculates rate gap vs market, produces lender negotiation email + client update email + Mercury task. ASIC RG 234 compliant — all savings framed as estimates. Works on any client identifiable by name, phone, email, or lender reference.
license: Complete terms in LICENSE.txt
---

# FinHub Rate Review Converter

Converts a rate gap finding into a complete negotiation action — lender email, client email, Mercury task — in one run.

## Triggers
`/rate-review` | "rate review for [client]" | "can we do better on [client]'s rate" | "what rate are they on" | "negotiate [client]'s rate" | "above market rate" | "rate too high for [client]"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_loan_book_health` → `get_rate_review_candidates` → `get_customer_all_loans` → `create_opportunity_task` → `add_opportunity_note`
- **Gmail / Google Calendar:** optional send + calendar follow-up

## Execution Steps

### STEP 1 — Identify Client
```
Input: name / phone / email / lender_ref
Tool: who_is(query)
Output: contact_id, active opp_id, lender, amount, current_rate, settlement_date
```
If no client specified → run `get_rate_review_candidates(rate_gap_min=0.30)` to find all candidates.

### STEP 2 — Rate Analysis
```
Tool: get_customer_all_loans(query) → pull current loan product rate
Cross-reference: current lender market rate (stored in tool knowledge base)

Calculate:
  rate_gap = current_rate - market_rate
  annual_saving = rate_gap × loan_amount / 100
  monthly_saving = annual_saving / 12
  clawback_safe = settlement_date < today - 12 months
```

Market rate reference (update quarterly):
```
CBA Variable:     6.09%    CBA Fixed 3yr:  5.89%
Westpac Variable: 6.14%    WBC Fixed 3yr:  5.94%
ANZ Variable:     6.19%    ANZ Fixed 3yr:  5.99%
NAB Variable:     6.09%    NAB Fixed 3yr:  5.89%
Firstmac:         6.29%    St George:      6.14%
```

### STEP 3 — Decision Logic
```
IF rate_gap >= 0.50% AND clawback_safe:
  RECOMMENDATION: Refinance to new lender
  → Run finhub-refinance-converter instead

IF rate_gap >= 0.25% AND NOT clawback_safe:
  RECOMMENDATION: Rate negotiation with current lender (no clawback risk)
  → Draft lender BDM email

IF rate_gap >= 0.25% AND clawback_safe:
  RECOMMENDATION: Try negotiation first, prepare refi as backup
  → Draft both emails

IF rate_gap < 0.25%:
  RECOMMENDATION: Rate is competitive — no action needed
  → Advise client, set 6-month review task
```

### STEP 4 — Draft Lender Negotiation Email
```
TO: [Lender BDM email from lender directory]
SUBJECT: Rate Review Request — [Client Name] — [Lender Ref]

Apply: mutual-finhub-email-writer rules
Tone: professional, evidence-based, non-adversarial
Include:
  - Current rate vs market comparison
  - Loan loyalty tenure (months since settlement)
  - LVR if known (lower LVR = stronger negotiation)
  - Specific rate requested
  - Response deadline (5 business days)

ASIC: No guaranteed outcome language
```

### STEP 5 — Draft Client Email
```
TO: client email from Mercury
SUBJECT: Good news — we're reviewing your [Lender] rate

Apply: mutual-finhub-email-writer rules + ASIC RG 234
Include:
  - Current rate confirmed
  - Market comparison framed as "may be able to"
  - Estimated saving framed as "could potentially save approximately"
  - What happens next (we contact lender, update you within 5 days)
  - Single CTA: reply to confirm you'd like us to proceed

NEVER use: "save you thousands", "guaranteed", "best rate"
USE instead: "you may benefit from", "estimated saving of approximately $X/month"
```

### STEP 6 — Mercury Updates
```
Tool: create_opportunity_task(
  query = lender_ref,
  subject = "Rate Review — Chase [Lender] response",
  due_date = today + 5 business days,
  assigned_to = opp_broker,
  priority = "high"
)

Tool: add_opportunity_note(
  query = lender_ref,
  note = "[Date] Rate Review initiated. Current: X%. Market: Y%. Gap: Z%. 
          Lender negotiation email sent. Client notified. 
          Follow up [date]."
)
```

## Output Format
```
RATE REVIEW — [CLIENT NAME]
─────────────────────────────────────────
LOAN SUMMARY
Lender: [X]  |  Amount: $[X]  |  Settled: [date] ([N] months)
Current Rate: X.XX%  |  Market Rate: X.XX%
Rate Gap: X.XX%  |  Annual Saving Estimate: $X,XXX  |  Monthly: $XXX

RECOMMENDATION: [Negotiate / Refinance / No action]
Clawback Risk: [SAFE / HIGH — explain]

ACTION 1 — Lender Negotiation Email [ready to send]
[full email body]

ACTION 2 — Client Notification Email [ready to send]
[full email body]

ACTION 3 — Mercury task created: "Rate Review — Chase [Lender] — [date]"
ACTION 4 — Note added to opportunity notepad
```

## ASIC Compliance Rules
- All savings = "estimated" and "subject to lender approval"
- Never state guaranteed rate reduction
- Never use "save you thousands", "massive savings"
- Use "may be able to reduce", "estimated saving of approximately"
- Comparison rate disclosure required if quoting a specific rate
