---
name: finhub-refinance-converter
description: Refinance analysis and conversion skill for Finance Hub. ALWAYS use when Daniel says "should [client] refinance", "refinance analysis for [client]", "is [client] better off refinancing", "refi options for [client]", "run a refi on [client]", "compare lenders for [client]", "refi this client", "/refi", or any request to analyse whether a client should move to a new lender. Runs full break-even analysis, clawback check, lender comparison, drafts refinance proposal email, creates new Mercury opportunity. ASIC RG 234 compliant. Uses get_rate_review_candidates + who_is + get_customer_all_loans.
license: Complete terms in LICENSE.txt
---

# FinHub Refinance Converter

Full refinance feasibility analysis → lender recommendation → proposal email → new Mercury opportunity in one run.

## Triggers
`/refi` | "should [client] refinance" | "refinance analysis for [client]" | "is [client] better off refinancing" | "refi options for [client]" | "run a refi on [client]" | "compare lenders for [client]"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_customer_all_loans` → `get_customer_assets_liabilities` → `get_customer_employment_income` → `create_opportunity` → `create_opportunity_task` → `add_opportunity_note`
- **V2 Loan Book:** `get_rate_review_candidates` (for bulk identification)

## Execution Steps

### STEP 1 — Pull Current Loan Position
```
Tool: who_is(query) → contact_id, active opp_id
Tool: get_customer_all_loans(query)
→ current lender, rate, amount, settlement_date, loan_type, IO/fixed details

Tool: get_customer_assets_liabilities(query)
→ property value, existing mortgage balance, LVR
```

### STEP 2 — Refinance Feasibility
```
Calculate:
  current_monthly_repayment = (rate × amount) / 12 (IO) 
                            or P&I formula (P&I)
  new_rate = best_available_rate for profile (see rate table)
  new_monthly_repayment = recalculate at new_rate
  monthly_saving = current - new
  annual_saving = monthly_saving × 12

  refinance_costs = $1,500 (discharge) + $300 (new loan setup) = $1,800
  break_even_months = refinance_costs / monthly_saving

  clawback_months_remaining = max(0, 24 - months_since_settlement)
  clawback_safe = months_since_settlement >= 12
```

### STEP 3 — Strategy Selection
```
IF break_even_months <= 6 AND clawback_safe:
  STRATEGY: STRONG REFINANCE CASE
  → Draft full proposal, create opp

IF break_even_months 7–18 AND clawback_safe:
  STRATEGY: MODERATE REFINANCE CASE
  → Try rate negotiation first (run rate-review-converter)
  → If fails, proceed to refinance

IF NOT clawback_safe (< 12 months settled):
  STRATEGY: WAIT — clawback at risk
  → Schedule review task for clawback_safe_date
  → Advise client to wait

IF break_even_months > 18:
  STRATEGY: NOT VIABLE
  → Rate negotiation only
```

### STEP 4 — Lender Comparison (Top 3)
```
Based on: loan_amount, LVR, loan_purpose (OO/INV), loan_type (IO/PI)

Rate table (update quarterly — Apr 2026):
  OO P&I Variable:
    CBA Wealth Package:     6.09%  (offset, no annual fee with package)
    NAB Choice Package:     6.09%  (offset)
    Westpac Flexi First:    6.14%
    ANZ Simplicity Plus:    6.19%
    
  OO Fixed 3yr:
    CBA:  5.89% | NAB: 5.89% | WBC: 5.94% | ANZ: 5.99%

  INV IO Variable:
    CBA:  6.59% | NAB: 6.54% | WBC: 6.64% | Firstmac: 6.39%

Select top 3 by: lowest rate + features match (offset/redraw/IO)
```

### STEP 5 — Draft Current Lender Negotiation Email
```
Always try negotiation before refinance — send this first.

TO: [current lender BDM]
SUBJECT: Retention Rate Request — [Client Name] — [Lender Ref]

Key points:
  - Years of loyalty
  - Current rate vs market
  - Specific rate requested (target new lender rate)
  - Willing to stay if competitive
  - Response needed within 5 business days
```

### STEP 6 — Draft Client Refinance Proposal
```
Apply: mutual-finhub-email-writer rules
ASIC: all figures estimated, conditional language

Structure:
  1. Current position (rate, repayment)
  2. What we found (market comparison)
  3. What's possible (3 lender options with estimated savings)
  4. Our recommendation (single lender)
  5. Process overview (4 steps, ~4 weeks)
  6. CTA: Reply YES to proceed OR book a 15-min call

ASIC language:
  "estimated saving of approximately $X/month"
  "subject to formal assessment and lender approval"
  "savings may vary depending on your circumstances"
```

### STEP 7 — Create New Opportunity
```
Tool: create_opportunity(
  client = contact_id,
  transaction_type = "RE",
  purpose = opp_purpose (OO/INV),
  loan_amount = current_balance,
  lender = recommended_lender,
  assigned_broker = current_broker
)

Tool: create_opportunity_task(
  subject = "Refinance — [Client] — chase lender response",
  due_date = today + 5 days,
  priority = "high"
)

Tool: add_opportunity_note(original opp,
  note = "[Date] Refinance analysis completed. 
          Rate gap: X%. Break-even: N months.
          Strategy: [negotiation first / direct refi].
          New opp created: RE [PURPOSE] [CLIENT]."
)
```

## Output Format
```
REFINANCE ANALYSIS — [CLIENT NAME]
─────────────────────────────────────────────────────
CURRENT POSITION
Lender: [X]  |  Rate: X.XX%  |  Balance: $X  |  Type: [PI/IO]
Monthly Repayment: $X  |  Settled: [date] ([N] months)

REFINANCE FEASIBILITY
Best New Rate: X.XX% ([Lender])  |  New Repayment: $X
Monthly Saving: $X  |  Annual Saving: ~$X
Refinance Costs: ~$1,800  |  Break-even: [N] months
Clawback Status: [SAFE ✅ / RISK ⚠️ — N months remaining]

RECOMMENDATION: [STRONG REFI / NEGOTIATE FIRST / WAIT]

TOP 3 LENDERS
1. [Lender] X.XX% — [key features]
2. [Lender] X.XX% — [key features]
3. [Lender] X.XX% — [key features]

ACTION 1 — Retention email to [current lender BDM] [ready]
ACTION 2 — Refinance proposal to client [ready]
ACTION 3 — New opp created: "RE [OO/INV] [CLIENT]"
ACTION 4 — Task: "Chase lender response — [date]"
```

## ASIC Compliance Rules
- Monthly/annual savings = always "estimated" and "approximately"
- Never guarantee approval or rate match
- Include: "Subject to formal credit assessment and lender approval"
- Rate comparison must note: "Comparison rate may differ — ask us for details"
