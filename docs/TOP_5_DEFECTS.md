# Top 5 Defects

---

## BUG-001: BOLA/IDOR — Unauthorized Transfer of Funds from Arbitrary Account

**Severity:** Critical
**Priority:** Highest

### Description
The Transfer Funds endpoint does not verify that the authenticated user owns or has permission over the `fromAccountId` supplied in the request. An attacker can specify any account ID — including one belonging to a different customer — as the funding source and successfully transfer money out of it.

### Endpoint
`POST /transfer`
Query Parameters: `fromAccountId`, `toAccountId`, `amount`

### Preconditions
- Attacker is authenticated as User A.
- Attacker knows (or can guess/enumerate) an account ID belonging to another customer.

### Steps to Reproduce
1. Authenticate as User A.
2. Send `POST /transfer?fromAccountId=12345&toAccountId=24555&amount=10000`, where account `12345` belongs to a different customer than the authenticated user.
3. Observe the response status and body.

### Expected Result
`403 Forbidden` (or `401 Unauthorized`), with an error indicating the authenticated user is not authorized to debit account `#12345`.

### Actual Result
`200 OK` — Response: *"Successfully transferred $10000 from account #12345 to account #24555"*. Funds are debited from an account the requester does not own.

### Supporting Evidence
Request/response pair captured via DevTools/Postman as above. [Attach screenshot of captured request, response body, and before/after balances for account #12345 in the submission repo.]

### Business Impact
Allows any authenticated user to drain funds from any other customer's account by guessing or enumerating account IDs — the most severe possible failure mode for a banking application's core transaction feature.

---

## BUG-002: IDOR on Account Creation — Funding New Account with Another Customer's Account

**Severity:** High
**Priority:** High

### Description
The account-creation endpoint accepts a `fromAccountId` used to fund the new account's opening balance without validating that this account belongs to the authenticated `customerId`.

### Endpoint
`POST /createAccount`
Query Parameters: `customerId`, `newAccountType`, `fromAccountId`

### Preconditions
- Attacker is authenticated with a valid `customerId`.
- Attacker knows an account ID belonging to a different customer.

### Steps to Reproduce
1. Log in with `customerId=42293`.
2. Send `POST /createAccount?customerId=42293&newAccountType=0&fromAccountId=12345`, where account `12345` does not belong to `customerId 42293`.
3. Check the response body.

### Expected Result
The system verifies `fromAccountId` belongs to the authenticated `customerId` before processing; if not, returns an authorization error (e.g., `403 Forbidden`).

### Actual Result
`200 OK` — a new account (`id: 74616`) is successfully created, funded from an account the requester does not own.

### Supporting Evidence
Request/response pair as above; new account ID `74616` created and linked to account `#12345`. [Attach screenshot of request and resulting account listing.]

### Business Impact
Same class of vulnerability as BUG-001 (broken access control / IDOR) applied to account creation — allows unauthorized use of another customer's funds to open new accounts.

---

## BUG-003: Lack of Input Validation — Negative Amounts Accepted on Transfer and Bill Pay

**Severity:** High
**Priority:** High

### Description
Both the Transfer Funds and Bill Pay flows accept negative values for the amount field. Rather than being rejected, a negative amount inverts the expected debit/credit direction: the source account's balance *increases* instead of decreasing, effectively creating funds from nothing.

### Endpoint
`POST /transfer` (also reproducible via the Bill Pay UI flow)
Query Parameters: `fromAccountId`, `toAccountId`, `amount`

### Preconditions
- Attacker is authenticated and has access to at least one account.

### Steps to Reproduce
1. Send `POST /transfer?fromAccountId=70731&toAccountId=59298&amount=-1000`.
2. Fetch transaction logs via `GET /accounts/59298/transactions`.
3. Separately, repeat the equivalent negative-amount submission through the Bill Pay form/API and check both accounts' balances.

### Expected Result
`400 Bad Request` — validation error requiring `amount` to be strictly positive (`> 0`).

### Actual Result
`200 OK` in both flows. The transfer/payment is processed with the negative value: a `-1000.00` entry is recorded in transaction history, and the source account's balance **increases** rather than being debited or rejected.

### Supporting Evidence
Request/response for the transfer case as above; equivalent evidence captured for Bill Pay via UI + DevTools. Balances confirmed before/after on both sides of the transaction. [Attach screenshots and transaction log excerpt.]

### Business Impact
Allows unauthorized fund creation ("money printing") by any authenticated user via a trivial parameter change, affecting both of the application's core money-movement flows.

---

## BUG-004: Business Logic Flaw — No Balance Check Permits Overdraft on Transfer

**Severity:** High
**Priority:** High

### Description
The Transfer Funds endpoint does not verify that the source account has sufficient available balance to cover the requested transfer amount, allowing the account to be driven negative.

### Endpoint
`POST /transfer`
Query Parameters: `fromAccountId`, `toAccountId`, `amount`

### Preconditions
- Source account has a known, limited balance (e.g., $100.00).

### Steps to Reproduce
1. Check balance for account `70731` (e.g., balance is `$100.00`).
2. Send `POST /transfer?fromAccountId=70731&toAccountId=59298&amount=1000`.
3. Check the response and resulting balance.

### Expected Result
`400 Bad Request` or `422 Unprocessable Entity`, with a message such as *"Insufficient funds"*.

### Actual Result
`200 OK` — the transfer completes despite the amount exceeding the account's available balance, leaving the source account negative.

### Supporting Evidence
Request/response pair as above, with before/after balance for account `#70731`. [Attach screenshot.]

### Business Impact
Permits unauthorized overdrafts with no limit enforcement, directly compromising account balance integrity — a core banking guarantee.

---

## BUG-005: Business Logic Flaw — Insufficient Funds Validation Missing on Account Creation

**Severity:** Critical
**Priority:** High

### Description
Opening a new account requires a minimum initial deposit of $100.00. The system does not verify that the selected source account has sufficient available balance to cover this deposit before processing the request, allowing the source account to be driven negative.

### Endpoint
`POST /createAccount` (also reproducible via the Open New Account UI flow)
Query Parameters: `customerId`, `newAccountType`, `fromAccountId`

### Preconditions
- User is authenticated.
- Source account balance is below the $100.00 minimum deposit (e.g., $0.00, account `14581`).

### Steps to Reproduce
1. Log in to ParaBank.
2. Confirm an existing account (`14581`) has a $0.00 balance.
3. Navigate to Open New Account, select a new account type, and select account `14581` as the funding source.
4. Submit the request and check the result.
5. Return to Accounts Overview and check balances.

### Expected Result
The system validates the source account's available balance before processing; if below $100.00, the request is rejected with a message such as *"Insufficient funds in the selected source account to complete the minimum deposit."* No account is created and no balance is deducted.

### Actual Result
`200 OK` — the new account is created with a $100.00 opening balance, and the source account balance becomes **-$100.00**. No validation error is shown.

### Supporting Evidence
Before: Account `14581` balance $0.00. After: Account `14581` balance -$100.00; new account `17562` balance $100.00. [Attach screenshot of Accounts Overview before/after, and the captured `createAccount` request/response.]

### Business Impact
Allows unauthorized overdraft creation via a routine, everyday user action (not just via deliberate parameter tampering), directly corrupting account balance integrity.

---

## Appendix: Additional Findings (Not in Top 5)

### A1. Self-Transfer Permitted, Creating Redundant Ledger Entries

**Severity:** Low
**Priority:** Low

**Description:** The Transfer Funds flow allows a user to transfer funds from an account to itself (same `fromAccountId` and `toAccountId`). The system does not block this logically meaningless operation. It correctly nets the balance to $0 (a "Funds Transfer Sent" debit and a "Funds Transfer Received" credit of the same amount are both recorded), so there is no fund-loss impact — but it clutters the transaction history with two redundant, potentially confusing ledger entries for an operation that had no real effect.

**Evidence:** Self-transfer of $1000.00 to/from account `#14898` completed successfully; Account Activity shows a `$1000.00` debit ("Funds Transfer Sent") and a `$1000.00` credit ("Funds Transfer Received") on the same date, net effect $0.

**Recommendation:** Reject same-account transfers at the UI and API level with a validation message (e.g., "Source and destination accounts must be different"), rather than processing them as a no-op.

### A2. Sensitive Data Exposure & Insecure Credential Transmission (Login)

Credentials transmitted via `GET /login/{username}/{password}` (plaintext credentials in the URL, plus unmasked PII such as SSN and address in the response). Not included in the Top 5 as it falls outside the assessment's declared scope of Account Operations and Money Movement, but flagged here as a notable adjacent finding worth a follow-up security review.