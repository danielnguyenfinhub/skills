---
name: finhub-winback-converter
description: Declined and not-proceeding client win-back converter for Finance Hub. ALWAYS use when Daniel says "win back [client]", "re-engage [client]", "client went with someone else", "declined client reactivation", "not proceeding client [name]", "second attempt [client]", "try again with [client]", "/winback", or any request to re-approach a client whose application was declined or who chose not to proceed. Checks months since decline, assesses changed market conditions, recommends resubmission readiness, drafts win-back email, creates new Mercury opportunity if viable. ASIC RG 234 compliant.
license: Complete terms in LICENSE.txt
---

# FinHub Win-Back Converter

Turns a previously declined or lost client into a second-attempt opportunity — when timing, rates, or policy has changed.

## Triggers
`/winback` | "win back [client]" | "re-engage [client]" | "client went with someone else" | "declined client reactivation" | "not proceeding client [name]" | "second attempt [client]" | "try again with [client]"

## MCP Tools Used
- **FINHUB-MERCURY CRM V2:** `who_is` → `get_customer_all_loans` → `get_customer_audit_trail` → `get_customer_employment_income` → `get_customer_assets_liabilities` → `create_opportunity` → `create_opportunity_task` → `add_opportunity_note`

## Execution Steps

### STEP 1 — Load Decline Context
```
Tool: who_is(query) → contact_id, closed opp_id, current_status
Tool: get_customer_all_loans(query)
→ find loan with status "Declined" or "Not Proceeding"
→ decline_date, lender, original_amount, decline_reason (if in notepad)

Tool: get_customer_audit_trail(query)
→ find exact date of status change to Declined/Not Proceeding
→ scan for decline reason in notes or status notes

Tool: get_customer_notes(query)
→ broker notes explaining decline reason
```

### STEP 2 — Viability Assessment
```
months_since_decline = (today - decline_date) / 30

MINIMUM WAIT PERIODS by reason:
  Serviceability:     6 months (income change needed)
  Credit history:     12 months (credit improvement time)
  Deposit/LVR:        6-12 months (savings time)
  Employment (casual/contract): 6 months (history needed)
  Policy change:      3 months (new lender policy review)
  Client not ready:   3 months (re-engagement window)
  Client chose other broker: 12 months (respecting choice)

IF months_since_decline < minimum_wait:
  OUTPUT: "Too soon to re-engage. Schedule for [minimum_wait_date]."
  Tool: create_opportunity_task(
    subject = "Win-Back Opportunity — [Client] — recontact [date]",
    due_date = minimum_wait_date,
    notes = "Original decline: [reason]. Minimum wait: [N] months."
  )
  STOP

IF months_since_decline >= minimum_wait:
  PROCEED to changed conditions check
```

### STEP 3 — Changed Conditions Analysis
```
Check what has changed since decline:

MARKET CHANGES:
  Rate environment: rates moved significantly since decline?
  Lender policy: relevant policy changes for this scenario?
  New lenders: new lenders with better policy for this profile?

CLIENT CHANGES (infer from time elapsed):
  Serviceability: 6+ months more income history
  Employment: possible stability improvement
  Credit: possible credit score improvement
  Savings: possible deposit growth
  Debt reduction: possible liability paydown

SCORE each change category:
  STRONG IMPROVEMENT (2 pts): clear improvement likely
  POSSIBLE IMPROVEMENT (1 pt): may have improved
  NO CHANGE (0 pts): unlikely to have changed

Total score >= 3: STRONG CASE — recommend re-submission
Total score 1-2:  MODERATE CASE — re-engage conversation
Total score 0:    WEAK CASE — wait longer
```

### STEP 4 — Lender Strategy
```
Based on original decline reason → suggest better-fit lender:

Serviceability tight:
  → Try: Firstmac (higher HEM allowances), ANZ (flex policy)
  → Avoid: CBA, WBC (conservative HEM)

Credit history issue:
  → Try: Pepper, La Trobe, Resimac (non-conforming)
  → After 12mo clean: try mainstream again

Self-employed income:
  → Try: Firstmac (1yr ABN), ANZ (alt doc), ING
  → Avoid: NAB, WBC (strict 2yr ABN)

Deposit/LVR:
  → Try: Firstmac (85% no LMI for some), family guarantee lenders
  → Consider: FHBG if eligible (first home buyer, under $900K)

Do NOT recommend: same lender that declined (without policy change evidence)
```

### STEP 5 — Draft Win-Back Re-engagement Email
```
Apply: mutual-finhub-email-writer
Tone: warm, non-pushy, curiosity-led (Voss: open-ended questions)

TIMING: send 30 days after minimum wait period

Subject options:
  Rate change: "Things have changed since we last spoke, [Name]"
  Policy change: "There may be a new option for your situation, [Name]"
  General: "Checking in — how are things, [Name]?"

DO NOT: mention "declined" or "rejected" — use "when we last spoke"
DO NOT: promise approval — this is a conversation starter only

Structure:
  1. RECONNECT: warmth, reference to when we last spoke
  2. WHAT CHANGED: specific change that opens the door
     "Since we last spoke, [lender] has updated their policy..."
     "Rates have moved in your favour..."
  3. POSSIBILITY: "There may now be an option worth exploring"
  4. LOW PRESSURE CTA: "Would you be open to a quick chat?"

ASIC:
  "Subject to formal assessment"
  "Cannot guarantee outcome — every application assessed individually"
```

### STEP 6 — Create New Opportunity
```
IF score >= 3 (STRONG CASE):
  Tool: create_opportunity(
    client = contact_id,
    transaction_type = original_transaction_type,
    purpose = original_purpose,
    loan_amount = original_amount,
    lender = recommended_new_lender,
    assigned_broker = original_broker
  )
  
  Opportunity name format:
  "[TransCode] [PurpCode] [CLIENT NAME] — APP 2"
  e.g., "NP OO JOHN NGUYEN AND THI SMITH — APP 2"

Tool: create_opportunity_task(
  subject = "Win-Back — [Client] — Initial call",
  due_date = today + 5 days,
  priority = "high",
  notes = "Original decline: [date] — [reason].
           Changed conditions: [list].
           Recommended lender: [X].
           New opp created: APP 2."
)

Tool: add_opportunity_note(on NEW opp,
  note = "[Date] Win-back opportunity. 
          Original decline: [lender] [date] — [reason].
          Changed conditions: [list].
          Strategy: [lender] — [approach]."
)
```

## Output Format
```
WIN-BACK ANALYSIS — [CLIENT NAME]
─────────────────────────────────────────────────────
ORIGINAL DECLINE
Lender: [X]  |  Declined: [date] ([N] months ago)
Amount: $[X]  |  Reason: [decline reason]
Status: [Declined / Not Proceeding]

VIABILITY CHECK
Minimum wait: [N] months  |  Elapsed: [N] months  |  [READY / TOO SOON]

CHANGED CONDITIONS
✅ [Improvement 1]: [detail]
✅ [Improvement 2]: [detail]
⚠️ [Possible improvement]: [detail]
Score: [N]/6 → [STRONG CASE / MODERATE / WEAK]

RECOMMENDED LENDER: [X] — [reason for switch from original]

ACTION 1 — Win-back email [ready to send]
[full email]

ACTION 2 — New Mercury opp: "[TransCode] [Purpose] [CLIENT] — APP 2" [created]
ACTION 3 — Task: "Win-Back — Initial call — [date]" [created]
ACTION 4 — Note added to new opportunity
```

## ASIC Compliance Rules
- Never reference "declined" in client communication — "when we last spoke"
- Never imply guaranteed approval on second attempt
- Any credit product discussion: "subject to formal assessment"
- If alternative lender is non-conforming (Pepper/La Trobe): disclose this clearly
