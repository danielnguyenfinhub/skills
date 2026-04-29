---
name: finhub-fixed-rate-expiry-converter
description: Fixed rate expiry urgency converter for Finance Hub. ALWAYS use when Daniel says "fixed rate expiring for [client]", "fixed rate coming up", "fixed rate expiry campaign", "[client] fixed rate ending", "fixed rate roll off", "rate expiry", "/fixed-expiry", or any request related to a client's fixed rate period ending. Calls get_fixed_rate_expiry_book or who_is to identify the expiry, calculates payment shock on revert, compares re-fix vs variable vs refinance options, produces urgency email + Mercury task. ASIC RG 234 compliant.
license: Complete terms in LICENSE.txt
---

# FinHub Fixed Rate Expiry Converter

Converts a fixed rate expiry event into an urgent client conversation before the revert rate hits.

## Triggers
`/fixed-expiry` | "fixed rate expiring for [client]" | "fixed rate coming up" | "fixed rate roll off" | "[client] fixed rate ending" | "rate expiry [client]" | "re-fix or refi [client]"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_fixed_rate_expiry_book` → `get_customer_all_loans` → `read_client_loan_products` → `create_opportunity_task` → `add_opportunity_note` → `update_opportunity_next_action`

## Execution Steps

### STEP 1 — Identify Expiry
```
IF specific client:
  Tool: who_is(query) → opp_id
  Tool: read_client_loan_products(query)
  → fixed_rate, revert_rate, fixed_expiry_date, loan_amount, term_remaining

IF bulk / no client named:
  Tool: get_fixed_rate_expiry_book(months_ahead=6)
  → returns all clients sorted by urgency
```

### STEP 2 — Calculate Payment Shock
```
days_until_expiry = fixed_expiry_date - today

current_fixed_rate = product.interest_rate
revert_rate = product.revert_rate (if stored) OR lender SVR from table:
  CBA SVR: 6.99%  |  WBC SVR: 7.09%  |  ANZ SVR: 6.99%  |  NAB SVR: 6.99%
  Firstmac: 7.24% |  St George: 7.09%

loan_amount = product.loan_amount
remaining_term_months = product.loan_term_remaining

io_monthly = loan_amount × current_fixed_rate / 100 / 12
pi_monthly = P&I formula(loan_amount, revert_rate, remaining_term_months)
payment_shock = revert_monthly - current_monthly
payment_shock_pct = payment_shock / current_monthly × 100
```

### STEP 3 — Option Analysis
```
OPTION A — Re-fix with current lender:
  re_fix_rate = current lender new fixed rate (from rate table)
  re_fix_saving = revert_rate - re_fix_rate
  
OPTION B — Refinance to competitor fixed:
  best_refi_fixed = lowest fixed rate from table below
  refi_saving_vs_revert = revert_rate - best_refi_fixed
  refinance_costs = ~$1,800
  break_even = refinance_costs / (monthly_saving × 12)

OPTION C — Go variable (revert):
  cost = payment_shock (negative — most expensive option)
  flag only if revert rate is competitive vs market

Fixed rate table (Apr 2026):
  CBA  2yr: 5.79%  3yr: 5.89%  5yr: 6.09%
  NAB  2yr: 5.79%  3yr: 5.89%  5yr: 6.09%
  WBC  2yr: 5.84%  3yr: 5.94%  5yr: 6.14%
  ANZ  2yr: 5.89%  3yr: 5.99%  5yr: 6.19%
  Firstmac 2yr: 5.69%  3yr: 5.79%
```

### STEP 4 — Urgency Classification
```
days_until_expiry:
  < 0:    EXPIRED — client already on SVR — CRITICAL
  0–14:   CRITICAL — contact today
  15–30:  URGENT — contact this week
  31–60:  HIGH — contact within 2 weeks
  61–90:  PLAN — contact within 30 days
  > 90:   WATCH — schedule for 60 days before expiry
```

### STEP 5 — Draft Urgency Email to Client
```
Apply: mutual-finhub-email-writer
Tone: warm urgency — action required, not alarm

Subject: "[Client name], your [Lender] fixed rate expires [date]"

Structure:
  1. URGENCY: "Your X.XX% fixed rate expires in [N] days"
  2. WHAT HAPPENS: "Without action, you'll move to [SVR]% — 
                    an estimated additional $XXX/month"
  3. YOUR OPTIONS: present 2-3 options clearly
  4. OUR RECOMMENDATION: state clearly which we recommend and why
  5. CTA: "Reply YES to proceed with Option [X]"
        OR "Call us on 0430 11 11 88 today"

ASIC: "Estimated" on all payment figures
      "Subject to lender assessment" on refi option
      Include comparison rate if quoting new fixed rate
```

### STEP 6 — Mercury Updates
```
Urgency level → task priority:
  CRITICAL/URGENT: priority = "high", due = today
  HIGH: priority = "high", due = today + 3 days
  PLAN: priority = "normal", due = 60 days before expiry

Tool: create_opportunity_task(
  subject = "FIXED RATE EXPIRY [date] — Action required — [Client]",
  due_date = urgency_due_date,
  priority = urgency_priority,
  notes = "Fixed rate X.XX% expires [date]. 
           Revert to [SVR]% = +$X/month payment shock.
           Options: re-fix at X.XX% OR refi to [lender] X.XX%."
)

Tool: update_opportunity_next_action(
  query = lender_ref,
  next_action_date = urgency_due_date,
  note = "Fixed rate expiry action required"
)
```

## Output Format
```
FIXED RATE EXPIRY — [CLIENT NAME]
─────────────────────────────────────────────────────
EXPIRY DETAILS
Lender: [X]  |  Fixed Rate: X.XX%  |  Expiry: [date]  |  [N] days left
Urgency: [CRITICAL / URGENT / HIGH / PLAN]

PAYMENT SHOCK ANALYSIS
Current Fixed: $X,XXX/month
Revert to SVR [X.XX%]: $X,XXX/month
Monthly Shock: +$XXX/month  |  Annual: +$X,XXX/year

OPTIONS
A) Re-fix with [Lender]: X.XX% (3yr) = $X,XXX/month  |  Saving vs SVR: $XXX
B) Refinance to [Lender]: X.XX% (3yr) = $X,XXX/month  |  Break-even: N months
C) Go Variable: [Revert SVR X.XX%] — NOT recommended

RECOMMENDATION: [Option A/B + reason]

ACTION 1 — Urgency email to client [ready — send today]
[full email]
ACTION 2 — Mercury task: "FIXED RATE EXPIRY [date] — [client]" [created]
ACTION 3 — Next action updated to [date]
```

## ASIC Compliance Rules
- All repayment figures = "estimated"
- Fixed rate quotes must include: "comparison rate available on request"
- Refinance option: "subject to credit assessment and lender approval"
- Never say "you must act now" — say "we recommend acting before [date]"
