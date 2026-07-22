# Banking QA Assessment — Account Operations & Money Movement

QA assessment of a banking demo application, focused on risk-based
exploratory testing, API/backend validation, and defect reporting for
account operations and fund transfer workflows.

## Scope

Testing covered the application's core money-movement functionality:
- Fund transfers between accounts
- Bill payment
- Account creation and opening deposits
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
| [`docs/03-top-5-defects.md`](docs/03-top-5-defects.md) | The five most significant defects found, written in Jira-style reports, plus an appendix of additional lower-severity/out-of-scope findings |
| [`docs/04-release-recommendation.md`](docs/04-release-recommendation.md) | Executive summary and release/no-release recommendation |

## Evidence

Supporting screenshots and captured network traffic for each defect are in
[`evidence/`](evidence/), organized by bug ID:

```
evidence/
├── bug-001-transfer-idor/
├── bug-002-createaccount-idor/
├── bug-003-negative-amount/
├── bug-004-overdraft/
├── bug-005-insufficient-funds/
└── captured-traffic.har
```

