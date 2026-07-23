# Test Strategy — Banking Transactions & Account Integrity

**Application Under Test:** ParaBank Demo (https://parabank.parasoft.com/parabank/index.htm)
**Scope:** Account Operations & Money Movement
**Tester:** Ahmed Ragab Hussien
**Date:** 22/7/2026

## 1. Objective

Validate that core financial workflows — fund transfers, bill payments, account
operations, and transaction history — behave consistently end-to-end, enforce
business rules server-side (not just in the UI), and maintain accurate account
state under both expected and adversarial inputs.

## 2. Testing Approach

Testing combined three layers, in this order:

1. **Exploratory UI testing** of the money-movement workflows, using
   risk-based test charters rather than a fixed script, to surface business
   logic and state-consistency issues a scripted checklist would miss.
2. **Backend/API validation** via browser DevTools and Postman — replaying
   captured requests with tampered parameters (amounts, account IDs, payee
   data) to check whether the server independently enforces the rules the UI
   appears to enforce.
3. **Cross-verification** of every suspected defect against transaction
   history and account balances, to confirm whether a UI-level anomaly
   reflects an actual data-integrity problem or is cosmetic.

Two user accounts were registered so that cross-account transfer manipulation
(sending funds to/from an account the authenticated session does not own)
could be tested — a common high-severity class of bug in banking apps that
single-account testing cannot reveal.

## 3. Areas Prioritized, and Why

Effort was allocated by financial and data-integrity risk, not evenly across
features:

| Priority | Area | Rationale |
|---|---|---|
| P1 | Transfer Funds | Direct, immediate money movement between accounts — highest blast radius if business rules aren't enforced |
| P1 | Bill Pay | Same balance/validation risk as transfers, plus payee-data integrity |
| P2 | Backend/API tampering on both of the above | UI validation is only half the story; server-side enforcement is where real financial risk lives |
| P2 | Transaction history & balance consistency | The "proof" layer — confirms whether a bad request actually corrupted account state |
| P3 | Account creation / Open New Account | Lower transactional risk than moving existing funds, but still checked for minimum-deposit and negative-value enforcement |
| P4 | Cosmetic/UI-only issues | Explicitly out of scope per the assessment brief; noted only if incidentally encountered |

Rationale in one line: **money movement between accounts is where an
undetected bug causes real financial loss — everything else is
secondary.**

## 4. Assumptions

- The ParaBank demo environment is shared/public; test data (balances,
  transaction history) may be altered by other testers concurrently, so
  some results were re-verified before being reported as defects.
- No formal requirements/spec document was provided, so expected behavior
  is inferred from standard banking-domain conventions (e.g., transfers
  should not be permitted below $0 resulting balance, amounts should be
  positive, users should only operate on accounts they own).
- "Backend validation" is scoped to what's observable via browser DevTools
  and Postman against the live demo's exposed endpoints — no access to
  source code, logs, or the underlying database was available.

## 5. Areas Intentionally Left Untested

- **Performance/load testing** — out of scope per the assessment brief and
  the allowed toolset (manual testing, DevTools, Postman only).
- **Cross-browser/mobile responsiveness** — cosmetic/UI concern, explicitly
  de-prioritized in favor of business-logic and data-integrity risks.
- **Authentication/session security testing** (e.g., session fixation, CSRF)
  — adjacent to but outside the stated scope of "Account Operations and
  Money Movement"; flagged as a follow-up area rather than investigated
  in depth given the timebox.
- **Full regression of every account type/loan/registration edge case** —
  time was concentrated on transfers and bill pay per the priority table
  above rather than spread thinly across all account features.
