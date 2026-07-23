# Top 5 Defects

Ranked using Risk-Based Testing principles (Business Impact, Financial
Impact, Data Integrity, Security Risk, Customer Impact, Probability of
Occurrence, Recoverability, Release Risk, Compliance/Banking Risk) —
**not** by Severity field alone. Order reflects overall business risk,
highest first.

---

## BUG-001: Lack of Input Validation — Negative Amounts Accepted on Transfer and Bill Pay ("Money Creation")

**Severity:** Critical
**Priority:** Highest
**Risk Score:** 10/10

### Description
Both the Transfer Funds and Bill Pay flows accept negative values for the amount field. Rather than being rejected, a negative amount inverts the expected debit/credit direction: the source account's balance *increases* instead of decreasing, effectively creating funds from nothing.

### Endpoint
`POST /transfer` (also reproducible via the Bill Pay UI flow)
Query Parameters: `fromAccountId`, `toAccountId`, `amount`

### Preconditions
- Attacker is authenticated and has access to at least one account (no other account or victim required).

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


---

## BUG-002: BOLA/IDOR — Unauthorized Transfer of Funds from Arbitrary Account

**Severity:** Critical
**Priority:** Highest
**Risk Score:** 10/10

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

---

## BUG-003: IDOR on Account Creation — Funding New Account with Another Customer's Account

**Severity:** High
**Priority:** High
**Risk Score:** 8.5/10

### Description
The account-creation endpoint accepts a `fromAccountId` used to fund the new account's opening balance without validating that this account belongs to the authenticated `customerId`.

### Endpoint
`POST /createAccount`
Query Parameters: `customerId`, `newAccountType`, `fromAccountId`

### Preconditions
- Attacker is authenticated with a valid `customerId`.
- Attacker knows an account ID belonging to another customer.

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

---

## BUG-004: Missing Idempotency Protection — Duplicate Resubmission Processes Multiple Real Transactions

**Severity:** High
**Priority:** High
**Risk Score:** 8/10

### Description
Neither the Transfer Funds nor the Bill Pay endpoint has any protection against duplicate submission. Each click of the submit button (e.g., due to a slow response, accidental double-click, or resubmission) is processed as a fully independent transaction, with no duplicate-request detection or confirmation step. This requires no malicious intent or request tampering — it reproduces from completely normal user behavior.

### Endpoint
`POST /transfer` and Bill Pay submission (UI)

### Preconditions
- User is authenticated with a funded source account.

### Steps to Reproduce
1. Navigate to Transfer Funds (or Bill Pay), enter a valid amount, and click Submit multiple times in quick succession (e.g., 3 clicks).
2. Check the source account's transaction history and balance.
3. Repeat the same test on the other flow (Bill Pay, if starting from Transfer Funds, or vice versa).

### Expected Result
Only one transaction should be processed per user intent. The system should detect and reject or ignore duplicate/rapid resubmissions (e.g., via a disabled submit button after first click, a confirmation step, or server-side idempotency keys).

### Actual Result
Each click was processed as a separate, real transaction — 3 clicks produced 3 separate transfers of the same amount, each debited independently. Confirmed on both Transfer Funds and Bill Pay.

### Supporting Evidence
Transaction history showing multiple identical "Funds Transfer Sent" entries for the same amount and date, corresponding to a single user action of repeated clicks. [Attach screenshot of transaction history and before/after balance.]

---

## BUG-005: Business Logic Flaw — Insufficient Funds Validation Missing on Account Creation

**Severity:** Critical
**Priority:** High
**Risk Score:** 7.5/10

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


---

## Appendix: Additional Findings (Not in Top 5)

### A1. Self-Transfer Permitted, Creating Redundant Ledger Entries

**Severity:** Low
**Priority:** Low

**Description:** The Transfer Funds flow allows a user to transfer funds from an account to itself (same `fromAccountId` and `toAccountId`). The system does not block this logically meaningless operation. It correctly nets the balance to $0 (a "Funds Transfer Sent" debit and a "Funds Transfer Received" credit of the same amount are both recorded), so there is no fund-loss impact — but it clutters the transaction history with two redundant, potentially confusing ledger entries for an operation that had no real effect.

**Evidence:** Self-transfer of $1000.00 to/from account `#14898` completed successfully; Account Activity shows a `$1000.00` debit ("Funds Transfer Sent") and a `$1000.00` credit ("Funds Transfer Received") on the same date, net effect $0.

**Recommendation:** Reject same-account transfers at the UI and API level with a validation message (e.g., "Source and destination accounts must be different"), rather than processing them as a no-op.

### A2. Sensitive Data Exposure & Insecure Credential Transmission (Login)

Credentials transmitted via `GET /login/{username}/{password}` (plaintext credentials in the URL, plus unmasked PII such as SSN and address in the response). Not included in the Top 5 as it falls outside the assessment's declared scope of Account Operations and Money Movement, but flagged here as a notable adjacent finding worth a follow-up security review. Note: in a pure risk-based (not scope-limited) ranking, this finding would score approximately 9/10, ahead of BUG-003 and BUG-004 above, due to its 100% occurrence rate (every login) and significant compliance exposure (GLBA-type breach notification risk).

### A3. Business Logic Flaw — No Balance Check Permits Overdraft on Transfer

**Severity:** High
**Priority:** High

**Description:** The Transfer Funds endpoint does not verify that the source account has sufficient available balance to cover the requested transfer amount, allowing the account to be driven negative. Reproduced by transferring $1000 from account `#70731` (balance $100.00) to account `#59298` — the transfer completed (`200 OK`) despite exceeding the available balance, leaving the source account negative. Not included in the Top 5 as it overlaps thematically with BUG-005 (missing balance validation), but documented here as it remains a valid, distinct finding on the transfer endpoint specifically.

### A4. IDOR on Loan Request — Down Payment Debited from Another Customer's Account
 
**Severity:** Critical
**Priority:** Highest
 
**Description:** The loan request service (`requestLoan`) correctly validates that the specified down-payment account has *sufficient funds* (confirmed via a positive control test — a loan request with an underfunded down-payment account was correctly rejected) but performs no ownership/authorization check at all on the `downPaymentAccountId`. A user can fund their own loan's down payment using an account belonging to a different customer.
 
**Endpoint:** `requestLoan(int customerId, double amount, double downPayment, int downPaymentAccountId)` (SOAP/Postman)
 
**Steps to Reproduce:**
1. Authenticate as the attacker.
2. Submit a loan request via the SOAP/API client, specifying `downPaymentAccountId` belonging to a different customer.
3. Observe the loan approval response.
4. Check the victim account's balance/transaction history to confirm the debit.
**Expected Result:** The system should verify `downPaymentAccountId` belongs to the authenticated customer before processing; if not, reject with an authorization error.
 
**Actual Result:** Loan approved; down payment successfully debited from an account the requester does not own.
 