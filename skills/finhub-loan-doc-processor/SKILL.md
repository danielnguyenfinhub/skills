---
name: finhub-loan-doc-processor
description: >
  FinHub document processor — renames supporting documents to the FinHub standard
  naming convention AND outputs a Mercury_Input_Package JSON with pre-filled API
  parameters for all 13 Mercury CRM input tools. Run this BEFORE running any
  input_identity_document, input_payslip_income, input_bank_statement_asset, or
  other Mercury input tools. The JSON package pre-fills every parameter so input
  tools can run without re-opening files. ALWAYS use when Daniel says:
  "process docs for [client]", "rename and package [folder]", "prepare docs for
  Mercury input", "rename and extract", "doc processor", "/doc-process",
  "process client documents", "rename docs and extract data", or any request to
  rename supporting documents AND prepare them for Mercury CRM input. Also
  triggers when running loan-doc-rename AND planning Mercury input in the same
  session. For rename ONLY (no API input planned) use loan-doc-rename instead.
  Produces two outputs: (1) renamed files in FinHub standard naming convention,
  (2) Mercury_Input_Package_{date}.json saved to the same folder, ready for the
  Mercury input tools to consume in the next step.
license: FinHub internal
---

# FinHub Loan Document Processor

Renames client supporting documents to the FinHub standard naming convention
AND extracts structured Mercury API input parameters from each document in one
pass. Outputs a Mercury_Input_Package JSON that pre-fills all 13 Mercury input
tool parameters so the broker never has to re-open a document.

---

## When to use this skill

Use this skill when the broker wants to:
1. Rename a folder of client documents to the FinHub standard
2. AND then input that data into Mercury CRM

If the broker only wants to rename (no Mercury input planned) → use
`loan-doc-rename` instead.

---

## Triggers

- `/doc-process [folder path]`
- "Process docs for [client name]"
- "Rename and package [folder]"
- "Prepare docs for Mercury input"
- "Rename and extract client docs"
- "Run the doc processor for [client]"

---

## Two outputs

### Output 1 — Renamed files
Same naming convention as `loan-doc-rename`:
```
[Category] [Document Name] [Number] [Date] [Optional Info] [Client Name] [Quality Flag].[ext]
```
Examples:
```
ID Passport RA6502622 Expiry 28.01.2036 Neelika Chowdhury.jpg
PAYG Income Payslip Fujifilm period end 28.02.2026 Shuva Chowdhury.pdf
Savings and Equity CBA Bank Statement account 5180 period end 31.01.2026 Neelika Chowdhury.pdf
Loan Statement Mortgage NAB account 7892 period end 31.03.2026 Shuva Chowdhury.pdf
Property Contract of Sale 42 Day Street Marrickville 01.04.2026 Cong Nguyen.pdf
```
Quality flags appended at END of filename before extension:
- `- BLURRY` — cannot read document clearly
- `- EXPIRED` — ID document expiry date is in the past
- `- BLURRY AND EXPIRED` — both

### Output 2 — Mercury_Input_Package JSON
Saved to the same folder as the renamed documents.
Contains pre-filled api_params for every document, ordered input sequence,
missing document checklist, and action_required items.

---

## Execution steps

### STEP 1 — Confirm inputs
Before doing anything, confirm:
1. Folder path (where the documents are)
2. Client name(s) — primary + co-applicant
3. Opportunity reference — lender ref or opp name fragment for Mercury lookup

### STEP 2 — Read every file
List the folder. Open and read every file to extract metadata:
- Images (jpg/png): visually read ID document fields
- PDFs: extract text with pymupdf (fitz)

Extract per document type:

| Document | Extract |
|---|---|
| Driver Licence | Licence number, state, expiry DD/MM/YYYY, name |
| Passport | Passport number, country, expiry DD/MM/YYYY, name |
| Medicare | Card number, reference position, expiry MM/YYYY, name |
| Payslip | Employer, name, period end, gross income, frequency, basis |
| Tax Return | Taxpayer name, FY, taxable income / net profit |
| BAS | Business name, ABN, period, turnover |
| Bank Statement | Bank, account name, last 4 digits, closing balance, period end |
| Mortgage Statement | Lender, last 4 digits, outstanding balance, repayment, frequency |
| Credit Card Statement | Provider, credit LIMIT (not balance), closing balance, min repayment |
| Personal/Car/HECS Statement | Institution, balance, repayment, type |
| Contract of Sale | Address, purchase price, date |
| Council Rates | Address, value if shown |
| Lease Agreement | Address, weekly rent, dates |
| Centrelink Letter | Benefit type, amount, frequency |
| Super Statement | Fund name, last 4 member number, balance, date |

### STEP 3 — Classify and rename
Map each document to a FinHub category and build the new filename.

Category mapping (abbreviated):
- Passport / Driver Licence / Medicare → `ID`
- Payslip / Employment Contract / ATO Income Statement → `PAYG Income`
- Tax Return / BAS / NOA / Accountant Letter (SE) → `Self Employed Income`
- Savings / transaction account → `Savings and Equity`
- Lease / rental income → `Rental Income`
- Centrelink / pension → `Government Income`
- Mortgage statement → `Loan Statement Mortgage`
- Credit card → `Loan Statement Credit Card`
- Personal / car / HECS → `Loan Statement [type]`
- Contract of sale / council rates / valuation → `Property`
- Super / shares / vehicle → `Assets`

Apply renames via Python os.rename() — plan all renames first, then execute.

### STEP 4 — Map to Mercury API tools

| Renamed file starts with | API Tool |
|---|---|
| `ID Passport` | input_identity_document → type: Passport |
| `ID Driver Licence` | input_identity_document → type: DriversLicenceAust |
| `ID Medicare` | input_identity_document → type: MedicareCard |
| `ID Not Applicable` | SKIP |
| `PAYG Income Payslip` | input_payslip_income |
| `PAYG Income Employment Contract` | input_payslip_income |
| `PAYG Income ATO` | input_payslip_income (annual income) |
| `Self Employed Income` | input_self_employed_income |
| `Savings and Equity Bank Statement` | input_bank_statement_asset |
| `Rental Income` | input_other_income → Rental |
| `Government Income` | input_other_income → Centrelink/Pension |
| `Loan Statement Mortgage` | input_mortgage_liability |
| `Loan Statement Credit Card` | input_credit_card_liability |
| `Loan Statement Personal Loan` | input_personal_loan_liability |
| `Loan Statement Car Loan` | input_personal_loan_liability → Car Loan |
| `Loan Statement HECS` | input_personal_loan_liability → HECS/HELP |
| `Property Contract of Sale` | input_property_asset → Being Purchased |
| `Property Council Rates` | input_property_asset → Existing Owned |
| `Property Valuation` | input_property_asset → Bank Valuation |
| `Assets Superannuation` | input_other_assets → Superannuation |
| `Assets Share Portfolio` | input_other_assets → Shares |

### STEP 5 — Build api_params for each document
For each non-skip document, build the api_params object with extracted values.
Mark any required field that could not be extracted as `NEEDS_BROKER_INPUT`.

Set ready: true only if:
- api_tool is not null
- No BLURRY flag
- No NEEDS_BROKER_INPUT in required fields

### STEP 6 — Build input_sequence (ordered correctly)
Priority order for Mercury input:
1. input_identity_document (all applicants — establishes person)
2. input_applicant_address (residential address evidence)
3. input_payslip_income (PAYG employment + income)
4. input_self_employed_income (SE employment + income)
5. input_other_income (rental, Centrelink, dividends)
6. input_bank_statement_asset (savings accounts)
7. input_property_asset (real estate assets)
8. input_other_assets (super, shares, vehicle)
9. input_mortgage_liability (link to property assets after writing)
10. input_credit_card_liability
11. input_personal_loan_liability

### STEP 7 — Identify missing documents
Check against standard mortgage checklist. Flag missing:
- At least 1 photo ID per applicant
- Minimum 2 most recent payslips per PAYG applicant
- 2yr tax returns for self-employed applicants
- 3 months bank statements for deposit account
- Statement for every declared liability
- Contract of sale or council rates for security property
- Living expenses declaration (usually from fact-find, not a document)

### STEP 8 — Save Mercury_Input_Package JSON
Save to the same folder:
`Mercury_Input_Package_{DDMMYYYY}.json`

Contents:
```json
{
  "generated": "DD/MM/YYYY HH:MM",
  "broker": "Daniel Nguyen",
  "opportunity_search": {
    "reference": "lender_ref or opp_name",
    "applicants": [{"name": "...", "role": "Primary"}, ...]
  },
  "summary": {
    "total_files": N,
    "ready": N,
    "flagged": N,
    "skipped": N
  },
  "input_sequence": ["1. tool — applicant | document", ...],
  "documents": [
    {
      "filename": "renamed filename",
      "api_tool": "input_identity_document",
      "ready": true,
      "quality_flags": [],
      "api_params": { ... pre-filled fields ... }
    }
  ],
  "missing_documents": ["..."],
  "action_required": ["filename — action text"]
}
```

### STEP 9 — Print broker report
```
═══════════════════════════════════════════════════
 FINHUB DOC PROCESSOR — COMPLETE
 Client: {names}  |  Opportunity: {ref}
═══════════════════════════════════════════════════

FILES RENAMED:        {N}
✅ Ready for input:   {N}
⚠️  Flagged:          {N}
⏭️  Skipped (N/A):    {N}

INPUT SEQUENCE:
  1. input_identity_document — Applicant | Doc type
  2. input_payslip_income — Applicant | Employer
  ...

FLAGGED — ACTION REQUIRED:
  filename — action description

MISSING DOCUMENTS:
  - item

JSON PACKAGE SAVED:
  Mercury_Input_Package_{date}.json

NEXT STEP:
  In Claude chat: "Input Mercury docs for {client}"
═══════════════════════════════════════════════════
```

---

## Important rules

1. Never rename without showing the complete plan first — confirm before executing
2. Never guess a dollar amount — mark as NEEDS_BROKER_INPUT if cannot read
3. Credit cards: capture the CREDIT LIMIT not the closing balance — this is what lenders use
4. Ownership Joint/Sole: if both applicant names appear on document → Joint; single name → Sole
5. Expired ID: write to Mercury anyway with EXPIRED flag — do not skip the input
6. NOT APPLICABLE files: rename correctly, mark api_tool: null in JSON, do not input to Mercury
7. Self-employed: flag if only 1 year of tax returns — most lenders require 2
8. The JSON package is the handoff — it must be complete and accurate for the input tools to use

---

## How the broker uses this skill

**Step 1 — In Cowork or Claude chat:**
"Process docs for CONG TRIEU NGUYEN — folder: /path/to/folder"
→ Files renamed
→ Mercury_Input_Package saved to folder

**Step 2 — In Claude chat:**
"Input Mercury docs for Cong Trieu Nguyen using the package"
→ Claude reads the JSON
→ Runs all input tools in sequence
→ Logs each to Mercury activity log

---

## Related skills and tools

- `loan-doc-rename` — rename only (no Mercury extraction)
- `mercury-crm-assistant` — write individual records to Mercury
- Mercury input tools: input_identity_document, input_payslip_income,
  input_self_employed_income, input_bank_statement_asset,
  input_mortgage_liability, input_credit_card_liability,
  input_personal_loan_liability, input_property_asset,
  input_other_income, input_other_assets, input_applicant_address,
  input_living_expenses, input_document_log
