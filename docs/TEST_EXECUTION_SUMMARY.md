# Test Execution Summary

Concise list of key scenarios executed against the Transfer Funds, Bill Pay,
Open New Account, and Request Loan flows, per the risk-based priorities in
the Test Strategy. This is a summary of outcomes, not a comprehensive test
case document.

| # | Scenario | Method | Outcome |
|---|---|---|---|
| 1 | Baseline transfer between two of the user's own accounts | UI | ✅ Pass — balances and transaction history updated correctly on both sides |
| 2 | Transfer Funds using a `fromAccountId` belonging to a different customer | DevTools/Postman, tampered `transfer` request | ❌ **Fail** — request succeeded (200 OK), $10,000 transferred out of an account the requester did not own. See BUG-002. |
| 3 | Open New Account using a `fromAccountId` belonging to a different customer | DevTools/Postman, tampered `createAccount` request | ❌ **Fail** — request succeeded (200 OK) with no ownership check. See BUG-003. |
| 4 | Submit a negative amount on Transfer Funds | UI + DevTools/Postman | ❌ **Fail** — request accepted; source account balance *increased* instead of being rejected or decreased. See BUG-001. |
| 5 | Submit a negative amount on Bill Pay | UI + DevTools/Postman | ❌ **Fail** — same behavior as #4: negative amount accepted, balance increased rather than debited. See BUG-001. |
| 6 | Transfer amount exceeding available source account balance | DevTools/Postman | ❌ **Fail** — transfer completed despite insufficient funds, driving the account negative. See Appendix A3. |
| 7 | Open New Account using a source account with insufficient balance (< $100 minimum deposit) | UI + API (`createAccount`) | ❌ **Fail** — account created anyway; source account driven to a negative balance. See BUG-005. |
| 8 | Self-transfer (same account as source and destination) | UI | ❌ **Fail** — self-transfer permitted; creates a redundant debit + credit pair in transaction history for a no-op operation. See Appendix A1. |
| 9 | Duplicate/rapid resubmission of the same transfer or payment | UI (repeated clicks) | ❌ **Fail** — no idempotency protection; each click processes a separate transaction (e.g., 3 clicks produced 3 separate transfers of the same amount). Confirmed on both Transfer Funds and Bill Pay. See BUG-004. |
| 10 | Request Loan with a down payment amount exceeding the down-payment account's available balance | API (SOAP/Postman, `requestLoan`) | ✅ Pass — application correctly rejects/blocks the loan when the down-payment account has insufficient funds. |
| 11 | Request Loan using a `downPaymentAccountId` belonging to a different customer | API (SOAP/Postman, `requestLoan`) | ❌ **Fail** — loan approved; down payment successfully debited from an account the requester does not own. Third confirmed instance of the same missing-ownership-validation pattern (alongside BUG-002 and BUG-003). See Appendix A4. |

## Notes

- Scenarios 4 and 5 are recorded as a single defect (BUG-001) since they
  share the same root cause (no sign/amount validation on the underlying
  debit-credit operation) rather than as two separate bugs.
- Scenario 10 is a positive/control test confirming the loan workflow
  correctly enforces balance sufficiency on the down-payment account —
  included to show the sufficiency check works, which makes the missing
  ownership check in Scenario 11 a validation gap rather than a general
  lack of input checking on that endpoint.
- Scenario 11 (loan IDOR) is the third confirmed occurrence of the same
  root cause as BUG-002 (Transfer IDOR) and BUG-003 (createAccount IDOR):
  missing account-ownership validation. Documented as Appendix A4 in the
  Top 5 Defects doc rather than a Top-5 slot, since it would otherwise
  over-represent the same vulnerability class already covered twice.
- A separate issue — plaintext credentials and unmasked PII on the login
  endpoint — was also identified during testing but falls outside the
  declared Account Operations/Money Movement scope, so it is not included
  in the numbered scenarios above. See Appendix A2 in the Top 5 Defects doc.
- All failing scenarios have corresponding API evidence (request/response
  captured via DevTools or Postman) and balance before/after data,
  detailed in the individual bug reports.