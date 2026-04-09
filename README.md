# DAML Loan Approval Management

A DAML smart contract project that models a bank loan workflow end to end — request, approval, disbursement, and repayment. Built across three modules that each add a layer on top of the previous one.

## How it works

A borrower submits a loan request to a bank. The bank checks if the amount is within 25% of the borrower's salary and under the bank's total lending cap. If it passes, the loan gets approved and the funds are disbursed into a token wallet in tranches. The borrower then repays over time — each payment has to be at least 5% of what was disbursed. Once fully repaid, the loan contract archives itself and the bank's lending capacity is restored.

The three modules are:

**Module 1 — LoanApproval:** Just the basics. Borrower creates a request, bank approves or rejects it, a Loan contract is created with the outcome.

**Module 2 — TokenDisbursement:** Adds the wallet and lending cap logic. Introduces `TokenWallet` for balance tracking and `LoanLimit` to enforce how much the bank can lend in total. Approved loans get disbursed into the wallet in chunks.

**Module 3 — LoanRepayment:** The full loop. Adds the `Repay` choice, minimum repayment enforcement (5% of disbursed amount), and auto-archiving when the loan is paid off. Each repayment also restores the bank's total limit so it can be lent out again.

## Prerequisites

- DAML SDK 2.10.0 (`daml version` to check)

## Getting started

```bash
git clone https://github.com/yaminidesai/DAML_LoanApprovalManagement.git
cd DAML_LoanApprovalManagement
daml build
daml start
```

Each module has its own script file you can run to walk through the workflow. To run a specific one:

```bash
daml script --dar .daml/dist/Sunflower-0.0.1.dar \
  --script-name LoanScript:testLoanApproval \
  --ledger-host localhost --ledger-port 6865
```

## Project structure

```
daml/
├── 1_LoanApproval/
│   ├── LoanApproval.daml
│   └── LoanScript.daml
├── 2_TokenDisbursement/
│   ├── TokenDisbursement.daml
│   └── TokenDisbScript.daml
└── 3_Loan Repayment Workflow/
    ├── LoanRepayment.daml
    └── LoanRepaymentscript.daml
```

## A few things worth knowing

The three modules are independent — `TokenWallet` and `LoanLimit` are redefined in each one rather than shared. That was intentional as the design evolved across modules.

The `Repay` choice returns `Optional (ContractId Loan)`. If it comes back `None`, the loan is fully paid and archived. If it comes back `Some cid`, it's a partial repayment and a new loan contract was created with the updated balance.

Contract keys are used in module 3 (`TokenWallet` keyed by owner). That's DAML 2.x style — they'd need to be removed if you ever migrate to DAML 3.x.
