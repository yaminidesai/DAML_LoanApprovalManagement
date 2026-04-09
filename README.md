# DAML Loan Approval Management

Modeling a bank loan lifecycle on Canton Network using DAML smart contracts. Built across three incremental modules — LoanApproval, TokenDisbursement, and LoanRepayment — where each module extends the previous one without breaking it.

Each module was built incrementally — Module 1 approval, Module 2 disbursement, Module 3 repayment — to show how DAML contracts can be extended without touching what already works.

The core idea is simple: loan eligibility, disbursement limits, and repayment thresholds are enforced by the contract itself, not by application code. If the business rule lives in the contract, no developer can bypass it by changing a config file or skipping a middleware check. The contract is the policy.

---

## What it does

A borrower submits a loan request to a bank. The bank checks whether the amount is within 25% of the borrower's stated income — an implementation of the CFPB Ability-to-Repay principle under Regulation Z, which requires lenders to verify a borrower's capacity to repay based on income and existing obligations. If the request passes, it also checks against the bank's total lending capacity. If both pass, the loan is approved.

Once approved, the bank disburses funds incrementally into the borrower's token wallet. Each disbursement is checked against the approved loan amount — you can't disburse more than was approved, and the assertion runs inside the contract, not in application code.

Repayment works the same way. The borrower repays in increments, each subject to a 5% minimum threshold enforced by a `LoanRepaymentRestriction` contract fetched at runtime. When the full disbursed amount is repaid, the loan contract archives itself and the bank's lending capacity is restored.

---

## Architecture

The project is structured as three progressive modules. Each one is self-contained — they don't import from each other — but each builds conceptually on the previous.

**LoanApproval** is the foundation. It handles the request and decision flow: borrower creates a `LoanRequest`, bank approves or rejects it, a `Loan` contract is created with the resulting status. The 25% income cap is enforced here as a contract assertion.

**TokenDisbursement** adds the financial infrastructure. Introduces `TokenWallet` (account-based balance tracking), `LoanLimit` (bank-wide lending capacity), and a `Disburse` choice on the `Loan` contract. The bank can disburse in tranches; each tranche is checked against the approved amount. The bank-wide limit tracks total outstanding exposure across all borrowers simultaneously — closer in spirit to a credit concentration limit than a per-borrower limit (the per-borrower statutory limit is a separate concept governed by 12 USC 84).

**LoanRepayment** closes the loop. Adds `repaidAmount` tracking to the `Loan` contract, a `LoanRepaymentRestriction` template for the minimum payment rule, and the `Repay` choice. When `repaidAmount` equals `disbursedAmount`, the contract returns `None` and archives. Partial repayments return `Some newLoanCid` — the updated loan contract — so the caller always has the current active contract reference.

---

## Key design decisions

**25% debt-to-income cap at contract level**
The income-based eligibility check runs inside `ApproveRequest` as an assertion. It cannot be disabled by an application configuration change or a missing middleware call. This models the CFPB Ability-to-Repay requirement — the lender must verify repayment capacity, and here that verification is structurally unavoidable.

**Bank-wide lending capacity tracked on-ledger**
`LoanLimit.totalLimit` decrements with every approval and increments with every repayment. This is closer to an internal credit concentration limit than a regulatory per-borrower cap. It means the bank's total exposure is always visible and consistent on the ledger — no separate reconciliation needed between an application database and the ledger state.

**`Optional (ContractId Loan)` return type on Repay**
The `Repay` choice returns `Optional (ContractId Loan)` rather than always returning a new contract. `None` means fully repaid and archived. `Some cid` means partial repayment, here is your new active contract. This forces the caller to handle both cases explicitly — you can't accidentally use a stale contract ID after full repayment because the old one no longer exists on the ledger.

**LoanRepaymentRestriction fetched at runtime**
The minimum payment rule lives in a separate `LoanRepaymentRestriction` contract rather than being hardcoded into `Loan`. The bank creates it once and it's fetched each time `Repay` is exercised. In principle, the bank could update the restriction contract to change the policy without touching the loan contract itself.

---

## Authorization model

The bank (`issuer`) controls: loan approval, disbursement, and creating the repayment restriction. The borrower (`owner`) controls: submitting loan requests, debit from their own wallet, and repayment. Neither party can exercise the other's choices — this is enforced by the `controller` field in each choice definition, not by application-layer access control.

A few specific things this prevents:
- A borrower cannot approve their own loan request
- A borrower cannot credit their own wallet (only the bank can mint)
- The bank cannot debit a borrower's wallet directly (only the borrower can spend)
- No party can disburse more than the approved loan amount

---

## Testing

Three DAML Script files, one per module. These are workflow demonstration scripts — they walk through the full lifecycle with real party allocations and show that the contracts behave correctly. They do not use `assertMsg` to verify outcomes, so they won't fail if something unexpected happens. Think of them as executable walkthroughs, not automated tests.

**LoanScript** — approves one loan request, rejects another, leaves a third as pending (the approval is intentionally commented out to demonstrate that state).

**TokenDisbScript** — four loan requests against a shared lending limit, one disbursement into a token wallet. Shows how the `totalLimit` decrements across multiple borrowers.

**LoanRepaymentscript** — the most complete. Four loan requests, one disbursement, repayment restriction created, then three repayments ($500 → $800 → $3700) until the loan archives. Demonstrates the `Optional (ContractId Loan)` handling — each repayment saves the new contract ID before passing it to the next.

---

## Known limitations

These are worth knowing upfront.

The scripts are workflow demos, not assertion-based tests. If a contract is created with wrong data, the script won't fail — it will just continue. Adding `assertMsg` checks to verify state after each step would make them proper tests.

Each module redefines `TokenWallet` and `LoanLimit` locally. They aren't imported from a shared module. This means the three modules are not interoperable as written — the types are structurally identical but are different DAML modules. A cleaner design would extract shared types into a common module.

Contract keys are used on `TokenWallet` in the repayment module (keyed by `owner`). Contract keys were removed in DAML 3.x. If this project is migrated to 3.x, those would need to be replaced with ContractId-based lookups.

---

## Running locally

Requires DAML SDK 2.10.0. Check with `daml version`.

```bash
git clone https://github.com/yaminidesai/DAML_LoanApprovalManagement.git
cd DAML_LoanApprovalManagement
daml build
daml start
```

To run a specific script against the sandbox:

```bash
daml script --dar .daml/dist/Sunflower-0.0.1.dar \
  --script-name LoanRepaymentscript:test_Repayment \
  --ledger-host localhost --ledger-port 6865
```

Replace the `--script-name` with `LoanScript:test_loan` or `TokenDisbScript:test_token` for the other modules.
