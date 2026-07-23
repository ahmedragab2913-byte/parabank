# Banking QA — Account Operations & Money Movement

Risk-based exploratory testing, API/backend validation, and defect
reporting for a banking demo application's account operations and fund
transfer workflows.

## Scope

Testing covered the application's core money-movement functionality:
- Fund transfers between accounts
- Bill payment
- Account creation and opening deposits
- Loan requests and down-payment handling
- Transaction history and account balance integrity
- Server-side validation of business rules and request parameters (via
  browser DevTools and Postman)

Full details on approach, prioritization, and assumptions are in the Test
Strategy document below.

## Contents

| Document | Description |
|---|---|
| [`docs/01-test-strategy.md`](docs/01-test-strategy.md) | Testing approach, prioritization rationale, assumptions, and areas intentionally left untested |
| [`docs/02-test-execution-summary.md`](docs/02-test-execution-summary.md) | Concise list of key scenarios executed and their outcomes |
| [`docs/03-top-5-defects.md`](docs/03-top-5-defects.md) | The five most significant defects found, ranked using risk-based testing criteria and written as Jira-style reports, plus an appendix of additional findings |
| [`docs/04-release-recommendation.md`](docs/04-release-recommendation.md) | Executive summary and release/no-release recommendation |

## Evidence

Supporting screenshots and captured network traffic for each defect are in
[`Evidences/`](Evidences/), organized by bug ID:

```
evidence/
├── bug-001-negative-amount/
├── bug-002-transfer-idor/
├── bug-003-createaccount-idor/
├── bug-004-duplicate-resubmission/
├── bug-005-insufficient-funds/
└── additional bug self transfer/
└──loans/
└──accounts overview
└──customer id
```

## Bonus: Automated Test

[If completed] An automated Selenium test covering one positive end-to-end
user journey (fund transfer between two owned accounts) is included in
[`automation/`](automation/), with its own README explaining how to run it.

## Summary of Findings

Testing identified five critical/high-severity defects affecting the
application's core money-movement flows. Most notably, the same root
cause — missing account-ownership validation — was found independently
on three separate endpoints (fund transfers, account creation, and loan
down payments), pointing to a systemic gap in server-side authorization
rather than isolated defects. Additional findings include an input-
validation gap allowing negative transaction amounts to inflate account
balances, and a missing idempotency check that allows normal user
behavior (e.g., a double-click) to process duplicate real transactions.
See the Release Recommendation for the overall release/no-release call
and supporting rationale.