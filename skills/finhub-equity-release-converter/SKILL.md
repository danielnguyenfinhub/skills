---
name: finhub-equity-release-converter
description: Equity release opportunity converter for Finance Hub. ALWAYS use when Daniel says "equity release for [client]", "how much equity does [client] have", "can [client] buy another property", "renovation loan for [client]", "investment loan using equity", "access equity", "equity check", "/equity", or any request to calculate usable equity and convert it into a new lending opportunity. Calls get_equity_release_candidates or who_is, calculates usable equity at 80% LVR, runs serviceability sense-check, produces client opportunity email + new Mercury opportunity stub + task. ASIC RG 234 compliant.
license: Complete terms in LICENSE.txt
---

# FinHub Equity Release Converter

Turns equity in an existing property into a new loan conversation — investment, renovation, or debt consolidation.

## Triggers
`/equity` | "equity release for [client]" | "how much equity does [client] have" | "can [client] buy another property" | "renovation loan" | "investment loan using equity" | "access equity" | "equity check [client]"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_equity_release_candidates` → `get_customer_assets_liabilities` → `get_customer_employment_income` → `create_opportunity` → `create_opportunity_task` → `add_opportunity_note`

## Execution Steps

### STEP 1 — Identify Client or Find Candidates
```
IF specific client named:
  Tool: who_is(query) → contact_id, active opp_id
  Tool: get_customer_assets_liabilities(query)
  → pull all real estate assets + existing mortgage balances

IF no client named (bulk run):
  Tool: get_equity_release_candidates(min_equity=100000, max_lvr=80)
  → returns ranked list of all clients with usable equity
```

### STEP 2 — Equity Calculation
```
For each property:
  property_value = asset.value
  existing_debt = sum of linked mortgage balances
  max_borrowing = property_value × 0.80 (80% LVR cap)
  usable_equity = max_borrowing - existing_debt

Portfolio equity:
  total_usable_equity = sum across all properties
  best_property = highest usable equity, lowest LVR

Display:
  Current LVR: existing_debt / property_value × 100
  Equity at 80% LVR: usable_equity
  Equity at 90% LVR (with LMI): (property_value × 0.90) - existing_debt
```

### STEP 3 — Purpose Identification
```
Determine best equity release purpose from client context:

IF existing property portfolio >= 2:
  purpose = "Investment Property Purchase"
  
IF single OO property, no investment:
  purpose = "Investment Property Purchase" OR "Renovation"
  
IF high consumer debt on file:
  purpose = "Debt Consolidation"

IF income suggests renovation stage of life:
  purpose = "Renovation / Upgrade"
```

### STEP 4 — Quick Serviceability Sense-Check
```
Tool: get_customer_employment_income(query)
→ gross_annual_income, employment_type

Quick capacity estimate:
  gross_monthly_income = annual / 12
  estimated_max_additional_debt = gross_annual_income × 4.5 (conservative)
  can_service = usable_equity <= estimated_max_additional_debt

Flag if:
  - Self-employed income (needs 2yr tax returns)
  - Income < $80,000 combined (limits investment capacity)
  - Age > 55 (exit strategy required)
```

### STEP 5 — Lender Recommendation
```
For investment equity release:
  Preferred: CBA Investment Variable 6.49% | Westpac Investment 6.54%
  IO available: Westpac IO 6.64% | CBA IO 6.59%

For renovation (OO):
  Preferred: CBA Wealth Package 6.09% | NAB Choice Package 6.09%

For debt consolidation:
  Preferred: CBA Extra Home Loan | NAB Base Variable
```

### STEP 6 — Draft Client Opportunity Email
```
Apply: mutual-finhub-email-writer (AIDA + Cialdini + Voss)
ASIC RG 234: all benefit language conditional

Hook options by purpose:
  Investment: "Your property may have significant untapped equity..."
  Renovation: "Your home may be working harder for you..."
  Debt consolidation: "There may be a smarter structure for your finances..."

Include:
  - Equity position (estimated, subject to valuation)
  - What they could potentially do with it
  - How we access it (refinance or top-up)
  - Single CTA: book a 15-minute call

NEVER: "guaranteed equity", "definitely can borrow", "free money"
USE: "subject to valuation and lender assessment", "may be able to access approximately"
```

### STEP 7 — Create Mercury Opportunity Stub
```
IF client confirms interest (or skill set to auto-create):

Tool: create_opportunity(
  client = contact_id,
  transaction_type = "RE" (refinance for equity) or "CO" (cashout),
  purpose = "INV" or "OO",
  loan_amount = usable_equity (estimated),
  lender = recommended_lender,
  assigned_broker = current broker
)

Tool: create_opportunity_task(
  subject = "Equity Release Discussion — [client]",
  due_date = today + 3 days,
  priority = "high",
  notes = "Usable equity: $X. Purpose: investment/renovation. 
           Call to confirm goals and book fact find."
)

Tool: add_opportunity_note(source opp,
  note = "[Date] Equity release opportunity identified. 
          Property value: $X. Usable equity at 80% LVR: $X.
          New opp created: [opp_name]."
)
```

## Output Format
```
EQUITY RELEASE OPPORTUNITY — [CLIENT NAME]
─────────────────────────────────────────────────────
PROPERTY PORTFOLIO
[Address]  |  Value: $X  |  Debt: $X  |  LVR: X%  |  Equity @ 80%: $X
[Address]  |  Value: $X  |  Debt: $X  |  LVR: X%  |  Equity @ 80%: $X

TOTAL USABLE EQUITY: $X,XXX,XXX (at 80% LVR)

SERVICEABILITY: [LIKELY SERVICEABLE / FLAGGED — explain]
RECOMMENDED PURPOSE: [Investment / Renovation / Debt Consolidation]
RECOMMENDED LENDER: [X] at X.XX%

ACTION 1 — Client Opportunity Email [ready to send]
[full email body]

ACTION 2 — New Mercury opportunity created: "[CO/RE] INV [CLIENT]"
ACTION 3 — Task created: "Equity Release Discussion — [date]"
ACTION 4 — Note added to source opportunity
```

## ASIC Compliance Rules
- Equity figures = "estimated, subject to formal valuation"
- Serviceability = "subject to lender assessment and full application"
- Never state client "can borrow" — always "may be able to access"
- Investment recommendations require: "This is general information only. Consider whether it's appropriate for your situation."
