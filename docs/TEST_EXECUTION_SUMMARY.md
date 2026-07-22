# Test Execution Summary

Concise list of key scenarios executed against the Transfer Funds, Bill Pay,
and Open New Account flows, per the risk-based priorities in the Test
Strategy. This is a summary of outcomes, not a comprehensive test case
document.

| # | Scenario | Method | Outcome |
|---|---|---|---|
| 1 | Baseline transfer between two of the user's own accounts | UI | ✅ Pass — balances and transaction history updated correctly on both sides |
| 2 | Transfer Funds using a `fromAccountId` belonging to a different customer | DevTools/Postman, tampered `transfer` request | ❌ **Fail** — request succeeded (200 OK), $10,000 transferred out of an account the requester did not own. See BUG-001. |
| 3 | Open New Account using a `fromAccountId` belonging to a different customer | DevTools/Postman, tampered `createAccount` request | ❌ **Fail** — request succeeded (200 OK) with no ownership check. See BUG-002. |
| 4 | Submit a negative amount on Transfer Funds | UI + DevTools/Postman | ❌ **Fail** — request accepted; source account balance *increased* instead of being rejected or decreased. See BUG-003. |
| 5 | Submit a negative amount on Bill Pay | UI + DevTools/Postman | ❌ **Fail** — same behavior as #4: negative amount accepted, balance increased rather than debited. See BUG-003. |
| 6 | Transfer amount exceeding available source account balance | DevTools/Postman | ❌ **Fail** — transfer completed despite insufficient funds, driving the account negative. See BUG-004. |
| 7 | Open New Account using a source account with insufficient balance (< $100 minimum deposit) | UI + API (`createAccount`) | ❌ **Fail** — account created anyway; source account driven to a negative balance. See BUG-005. |

## Still pending / not yet executed

- Self-transfer (same account as source and destination)
- Duplicate/rapid-resubmission of the same transfer or payment
- Bill Pay payee data validation (empty/invalid payee fields)
- Broader response code/status consistency check beyond the endpoints already tested above

## Notes

- Scenarios 4 and 5 are recorded as a single defect (BUG-003) since they
  share the same root cause (no sign/amount validation on the underlying
  debit-credit operation) rather than as two separate bugs.
- A sixth issue — plaintext credentials and unmasked PII on the login
  endpoint — was also identified during testing but falls outside the
  declared Account Operations/Money Movement scope, so it is not included
  in the numbered scenarios above. See the note in the Top 5 Defects doc.
- All failing scenarios have corresponding API evidence (request/response
  captured via DevTools or Postman) and balance before/after data,
  detailed in the individual bug reports.
