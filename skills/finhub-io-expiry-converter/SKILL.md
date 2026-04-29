---
name: finhub-io-expiry-converter
description: Interest-only expiry converter for Finance Hub. ALWAYS use when Daniel says "IO expiry for [client]", "interest only ending", "[client] IO converting to P&I", "IO expiry campaign", "interest-only period ending", "IO roll off", "/io-expiry", or any request related to an interest-only period expiring. Calls get_io_expiry_book or who_is, calculates P&I payment shock, assesses serviceability risk, recommends extend IO / new IO lender / convert to P&I, produces urgency email + Mercury task + new opp if refinance recommended. ASIC RG 234 compliant.
license: Complete terms in LICENSE.txt
---

# FinHub IO Expiry Converter

Turns an IO expiry into a structured conversation before the P&I payment shock hits.

## Triggers
`/io-expiry` | "IO expiry for [client]" | "interest only ending" | "[client] IO converting to P&I" | "IO expiry campaign" | "interest-only period ending" | "IO roll off [client]"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_io_expiry_book` → `get_customer_all_loans` → `read_client_loan_products` → `get_customer_employment_income` → `get_customer_living_expenses` → `create_opportunity` → `create_opportunity_task` → `add_opportunity_note`

## Execution Steps

### STEP 1 — Identify IO Expiry
```
IF specific client:
  Tool: who_is(query) → opp_id, contact_id
  Tool: read_client_loan_products(query)
  → io_expiry_date, current_rate, loan_amount, remaining_term

IF bulk / campaign:
  Tool: get_io_expiry_book(months_ahead=12)
  → sorted list by urgency
```

### STEP 2 — Payment Shock Calculation
```
io_expiry_date = product.io_expiry
days_until_expiry = io_expiry_date - today
remaining_term_years = product.loan_term_remaining / 12
loan_amount = product.loan_amount
rate = product.interest_rate

Current IO monthly = loan_amount × rate / 100 / 12

New P&I monthly = PMT formula:
  rate_monthly = rate / 100 / 12
  n = remaining_term_years × 12
  pmt = loan_amount × (rate_monthly × (1+rate_monthly)^n) / ((1+rate_monthly)^n - 1)

payment_shock_monthly = pmt - io_monthly
payment_shock_pct = payment_shock_monthly / io_monthly × 100

Risk levels:
  shock_pct < 15%:   LOW risk — client can likely absorb
  shock_pct 15-30%:  MEDIUM risk — serviceability check needed
  shock_pct > 30%:   HIGH risk — urgent action required
```

### STEP 3 — Serviceability Assessment
```
Tool: get_customer_employment_income(query)
→ gross_annual_income, employment_type

Tool: get_customer_living_expenses(query)
→ total_monthly_expenses

rough_net_income_monthly = gross_annual_income / 12 × 0.70 (tax estimate)
available_for_debt = rough_net_income_monthly - total_monthly_expenses
can_service_pi = pmt <= available_for_debt × 0.40

Flag HIGH RISK if:
  - payment_shock_pct > 30%
  - can_service_pi = false
  - Self-employed or contractor income
  - Age > 60 (retirement proximity)
```

### STEP 4 — Option Analysis
```
OPTION A — Extend IO with current lender:
  → Check if lender allows IO extension (most allow 1x extension)
  → New IO rate (usually higher than original: +0.20–0.50%)
  → Extends P&I pressure by 1–5 years

OPTION B — Refinance to new IO product:
  Best IO lenders (INV IO Apr 2026):
    Firstmac INV IO: 6.39%
    CBA INV IO:      6.59%
    NAB INV IO:      6.54%
    WBC INV IO:      6.64%
  → New IO period: 5 years (most lenders)
  → Refinance costs: ~$1,800
  → Must check: max IO period not exceeded (usually 10yr)

OPTION C — Convert to P&I:
  → Accept payment shock
  → Builds equity faster
  → Appropriate if: risk is LOW, client has capacity, investment cycle ending

OPTION D — Sell investment property:
  → If serviceability is HIGH RISK and equity exists
  → Avoid financial stress
  → Flag to broker for conversation — do not recommend in email
```

### STEP 5 — Draft IO Expiry Alert Email
```
Apply: mutual-finhub-email-writer
Tone: proactive advisor, not alarmist

Subject: "Your interest-only period ends [date] — here's what to do"

Structure:
  1. WHAT'S HAPPENING: IO period expires [date]
  2. IMPACT: estimated repayment moving from $X to $X/month (+$X)
  3. OPTIONS: present 2–3 relevant options only
  4. RECOMMENDATION: clear single recommendation
  5. TIME SENSITIVITY: "We need to act [N] weeks before expiry 
                        to avoid automatic conversion"
  6. CTA: "Reply to book a 15-min call" OR "Reply YES to proceed"

ASIC:
  "Estimated repayment change of approximately $X/month"
  "Subject to lender assessment"
  "We recommend seeking independent financial advice on 
   investment property strategy"
```

### STEP 6 — Mercury Updates
```
Urgency classification:
  days < 30:  CRITICAL — task due today, priority high
  days 30-60: URGENT — task due this week
  days 60-90: HIGH — task due in 2 weeks
  days > 90:  PLAN — task due 60 days before expiry

Tool: create_opportunity_task(
  subject = "IO EXPIRY [date] — Action required — [Client]",
  due_date = calculated_urgency_date,
  priority = urgency_priority,
  notes = "IO period expires [date]. Shock: +$X/month ([X]%).
           Risk level: [HIGH/MEDIUM/LOW].
           Recommended: [Option A/B/C].
           Serviceability: [can/cannot service P&I]."
)

IF OPTION B recommended:
  Tool: create_opportunity(
    transaction_type = "RE",
    purpose = "INV",
    lender = best_io_lender,
    loan_amount = current_balance
  )
```

## Output Format
```
IO EXPIRY ALERT — [CLIENT NAME]
─────────────────────────────────────────────────────
IO DETAILS
Lender: [X]  |  Rate: X.XX% IO  |  Balance: $X
IO Expiry: [date]  |  [N] days remaining
Remaining Loan Term: [N] years

PAYMENT SHOCK
Current IO: $X,XXX/month
New P&I:    $X,XXX/month
Monthly Shock: +$XXX/month (+X%)
Risk Level: [HIGH ⚠️ / MEDIUM / LOW ✅]
Serviceability: [CAN / CANNOT service P&I]

OPTIONS
A) Extend IO with [Lender]: X.XX% IO — new $X,XXX/month [defer shock N years]
B) Refinance to [Lender] IO: X.XX% — new $X,XXX/month [new 5yr IO period]
C) Convert to P&I: X.XX% — $X,XXX/month [build equity, accept shock]

RECOMMENDATION: [Option + reason]

ACTION 1 — IO expiry alert email [ready to send]
[full email]
ACTION 2 — Mercury task: "IO EXPIRY [date] — [client]" [created]
ACTION 3 — New refi opp created [if applicable]: "RE INV [CLIENT]"
```

## ASIC Compliance Rules
- Repayment figures always "estimated"
- IO extension subject to "lender assessment and policy"
- Investment strategy: always include "seek independent financial advice"
- Serviceability assessment: "indicative only — formal assessment required"
