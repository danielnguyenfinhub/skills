---
name: finhub-upsell-investment-converter
description: Investment loan upsell converter for Finance Hub. ALWAYS use when Daniel says "investment opportunity for [client]", "can [client] buy an investment property", "upsell investment loan [client]", "property investor conversation [client]", "cross-sell investment [client]", "investment loan [client]", "next property for [client]", "/invest", or any request to identify and convert a settled client into an investment loan applicant. Pulls equity + income + property portfolio from Mercury, runs investment capacity analysis, identifies target property range, drafts investment conversation email, creates new Mercury opportunity, produces 1-page broker brief. ASIC RG 234 compliant.
license: Complete terms in LICENSE.txt
---

# FinHub Upsell Investment Converter

Converts equity + income into a structured investment property conversation — brief, email, opportunity, and task in one run.

## Triggers
`/invest` | "investment opportunity for [client]" | "can [client] buy an investment property" | "upsell investment loan [client]" | "property investor conversation [client]" | "cross-sell investment [client]" | "next property for [client]"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_customer_assets_liabilities` → `get_customer_employment_income` → `get_customer_living_expenses` → `get_customer_security_properties` → `read_client_loan_products` → `create_opportunity` → `create_opportunity_task` → `add_opportunity_note`
- **V2 Loan Book:** `get_equity_release_candidates` (for bulk identification)

## Execution Steps

### STEP 1 — Load Full Financial Position
```
Tool: who_is(query) → contact_id, opp_id, settlement_date
Tool: get_customer_assets_liabilities(query)
→ all real estate (value, debt, rental income, LVR per property)
→ savings, super, vehicles

Tool: get_customer_employment_income(query)
→ gross_annual_income (all sources: PAYG, rental, other)
→ employment_type, employer, stability

Tool: get_customer_living_expenses(query)
→ declared_monthly_expenses
→ HEM comparison

Tool: get_customer_security_properties(query)
→ detailed property portfolio (usable equity per property)
```

### STEP 2 — Investment Capacity Analysis
```
EQUITY CALCULATION:
  For each property:
    usable_equity_80 = (value × 0.80) - existing_debt
    usable_equity_90 = (value × 0.90) - existing_debt (with LMI)
  
  total_usable_equity = sum(usable_equity_80 across all properties)
  best_equity_property = property with highest usable_equity_80

BORROWING CAPACITY ESTIMATE (rough):
  gross_monthly_income = annual_income / 12
  buffer_rate = current_rate + 3.00% (APRA buffer)
  
  existing_debt_service = sum(all mortgage repayments)
  existing_card_limit_service = sum(credit_limits × 3.8% / 12)
  total_existing_commitments = existing_debt + living_expenses + card_service
  
  available_for_new_debt = (gross_monthly_income × 0.30) - total_existing_commitments
  
  IF available_for_new_debt > 0:
    estimated_capacity = available_for_new_debt / (buffer_rate/100/12)
    cap_by_equity = total_usable_equity
    recommended_loan = min(estimated_capacity, cap_by_equity)
  ELSE:
    serviceable = false → flag to broker

TARGET PROPERTY RANGE:
  If equity provides 20% deposit + costs:
    purchase_price_range = recommended_loan × 1.25 (80% LVR)
    lower_bound = recommended_loan × 0.90
    upper_bound = recommended_loan × 1.15
```

### STEP 3 — Investment Profile Assessment
```
ASSESS suitability:

GREEN LIGHTS (score +1 each):
  - Gross income >= $100,000 (combined if couple)
  - Existing property(s) — proven property investor mindset
  - Stable employment (PAYG or established SE)
  - Total usable equity >= $100,000
  - LVR on existing properties <= 70%
  - Clawback safe (settled >= 12 months)
  - Age <= 55 (long investment horizon)

AMBER FLAGS (score -1 each, note to broker):
  - Self-employed < 3 years
  - Single income household < $80K
  - High existing debt-to-income ratio
  - Age 56-65 (exit strategy needed)

RED FLAGS (STOP — flag to broker only):
  - Credit impairment on file
  - Income < $70K combined
  - Usable equity < $80K (may not cover deposit + costs)
  - Age > 65 (exit strategy critical — not suitable for standard investment)

Score >= 4 GREEN: STRONG INVESTMENT CANDIDATE → draft email
Score 2-3 GREEN: MODERATE → present as exploratory conversation
Score < 2: NOT YET → schedule for future review
```

### STEP 4 — Market Context & Suburb Suggestions
```
Based on recommended_loan range and Finance Hub client base:
(Update quarterly — these are indicative examples)

$400K-$600K range (House):
  Western Sydney: Fairfield, Cabramatta, Canley Vale, Liverpool
  Melbourne: Springvale, Footscray, Sunshine
  Adelaide: Elizabeth, Salisbury, Davoren Park

$600K-$800K range (House/Townhouse):
  Sydney: Blacktown, Penrith, Campbelltown, Bankstown
  Melbourne: Werribee, Hoppers Crossing, Craigieburn
  Brisbane: Logan, Ipswich, North Lakes

$800K-$1M range (Established suburbs):
  Sydney: Parramatta, Liverpool, Homebush
  Melbourne: St Albans, Deer Park, Broadmeadows

Rental yield target: >= 4.5% for near-neutral gearing
Property type preference: Established house (simpler for lender)
```

### STEP 5 — Draft Investment Conversation Email
```
Apply: mutual-finhub-email-writer (AIDA + Cialdini: social proof + scarcity)
ASIC RG 234: investment language must be conditional + general only

Subject options:
  Equity-led:  "[Name], your property may have unlocked an investment opportunity"
  Income-led:  "[Name], based on your profile, you may be in a strong position to invest"
  Milestone:   "[Name], it's been [N] months — have you thought about your next property?"

Structure:
  1. HOOK: "Since settling your [Lender] loan, your property may have grown..."
  2. THE OPPORTUNITY: "With [equity estimate] potentially available, 
                       you may be in a position to..."
  3. WHAT THIS MEANS: investment borrowing capacity range (estimated)
  4. MARKET CONTEXT: brief — why now is worth exploring
  5. PROCESS: "Here's how it works — 3 steps..."
  6. INVESTMENT DISCLAIMER (mandatory):
     "This is general information only. Investment property carries risks 
      including vacancy, maintenance costs, and market fluctuation. 
      We recommend seeking independent financial advice from a 
      qualified financial planner before proceeding."
  7. CTA: "Reply YES for a 15-min investment strategy call"
       OR "Book a call: 0430 11 11 88"

ASIC mandatory inclusions:
  "General information only — not personal financial advice"
  "Subject to formal credit assessment and lender approval"
  "We recommend seeking independent financial advice"
```

### STEP 6 — Produce Broker Brief (1-Page)
```
INVESTMENT OPPORTUNITY BRIEF — [CLIENT] — [DATE]

CLIENT POSITION:
  Gross Income: $X pa  |  Employment: [type]
  Properties: [list with values and debt]
  Total Equity @ 80% LVR: $X

INVESTMENT CAPACITY:
  Estimated Borrowing: ~$X (rough — formal assessment needed)
  Available Equity for Deposit: $X
  Target Property Range: $X–$X
  Loan Structure: INV [IO/PI], [lender]

RECOMMENDED LENDER:
  [Lender] INV IO [X.XX%] — reason for selection

CONVERSATION AGENDA:
  1. Goals: "Are you thinking about property investment?"
  2. Equity: "Your property has grown — you may have $X available"
  3. Capacity: "Based on your income, you may be able to borrow ~$X"
  4. Suburbs: suggest 2-3 suburbs matching their budget + lifestyle
  5. Next steps: fact find, pre-approval

RISK FLAGS:
  [Any amber flags to mention in broker conversation]

DISCLAIMERS TO COVER:
  [ ] Vacancy risk
  [ ] Maintenance costs
  [ ] Market fluctuation
  [ ] Independent financial advice recommended
  [ ] Negative gearing not a primary reason to invest
```

### STEP 7 — Create Mercury Opportunity and Tasks
```
Tool: create_opportunity(
  client = contact_id,
  transaction_type = "NP",
  purpose = "INV",
  loan_amount = recommended_loan,
  lender = recommended_lender,
  assigned_broker = current_broker
)
Opportunity name: "NP INV [CLIENT FULL NAME]"

Tool: create_opportunity_task(
  subject = "Investment Strategy Call — [Client]",
  due_date = today + 5 days,
  priority = "high",
  notes = "Investment opportunity identified.
           Equity: $X  |  Capacity: ~$X  |  Range: $X–$X
           Recommended: [lender] INV IO.
           Email sent [date]. Awaiting response."
)

Tool: add_opportunity_note(
  note = "[Date] Investment upsell initiated.
          Profile: [score]/7 GREEN — [STRONG/MODERATE].
          Equity available: $X. Estimated capacity: ~$X.
          New opp created: NP INV [CLIENT]."
)
```

## Output Format
```
INVESTMENT OPPORTUNITY — [CLIENT NAME]
─────────────────────────────────────────────────────
FINANCIAL POSITION
Income: $X pa ([employment type])
Properties: [list — value / debt / LVR / equity]
Total Usable Equity @ 80% LVR: $X

INVESTMENT CAPACITY
Estimated Borrowing: ~$X
Deposit Available from Equity: $X
Target Property Range: $X–$X
Profile: [N]/7 GREEN → [STRONG / MODERATE / NOT YET]

RISK FLAGS: [list any amber flags]

RECOMMENDED STRUCTURE
Lender: [X]  |  Product: INV IO  |  Rate: X.XX%  |  Period: 5yr IO
Rationale: [1 sentence]

SUGGESTED SUBURBS (budget match)
$[range]: [suburb 1], [suburb 2], [suburb 3]
Rental yield target: >= 4.5%

ACTION 1 — Broker investment brief [above]
ACTION 2 — Investment conversation email [ready to send]
[full email]
ACTION 3 — New Mercury opp: "NP INV [CLIENT]" [created]
ACTION 4 — Task: "Investment Strategy Call — [date]" [created]
ACTION 5 — Note added to existing opportunity
```

## ASIC Compliance Rules (Investment — Strict)
- MANDATORY disclaimer: "General information only — not personal financial advice"
- MANDATORY: "We recommend seeking independent financial advice from a licensed financial planner"
- NEVER: imply guaranteed rental income or capital growth
- NEVER: use negative gearing as primary reason to invest
- NEVER: recommend a specific property or suburb as an investment (general area only)
- Capacity estimate: ALWAYS "estimated" and "subject to formal assessment"
- Vacancy risk, maintenance costs, market risk MUST be noted in broker brief
