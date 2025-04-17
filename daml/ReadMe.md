# Loan Workflow System – End-to-End Design & Approach

## Setup & Execution Guide

To run this DAML-based loan system locally, follow these steps:

### Prerequisites

- Install the latest version of the [DAML SDK](https://docs.daml.com/getting-started/installation.html) for your operating system (Windows, macOS, or Linux).
- Recommended: Install VS Code with the DAML extension for better contract development and script execution.

### Getting Started

```bash
# Step 1: Install DAML (if not already installed)
daml install latest

# Step 2: Set up your project
mkdir loan-system && cd loan-system
daml new loan-workflow --template=create-daml-app
cd loan-workflow

# Step 3: Replace contents with this project's DAML files
# (LoanApproval.daml, TokenDisbursement.daml, LoanRepayment.daml, etc.)

# Step 4: Build and compile DAML modules
daml build

# Step 5: Start sandbox and run scripts
daml start  # Or use: daml script --script-name Main:test_YourScript
```

### Files Structure

- `LoanApproval.daml` – Defines loan request and approval flow
- `TokenDisbursement.daml` – Manages token minting and disbursement
- `LoanRepayment.daml` – Handles partial repayments and contract archiving
- `*.daml` scripts – Validate each flow through test cases

### Assumptions & Design Decisions

- The bank has full control over approval, disbursal, and repayment rules
- Loan approvals are based on borrower salary capped at 25%
- TokenWallet uses an account-based model for balance updates
- Repayment logic enforces a 5% minimum threshold to prevent microtransactions
- Upon full repayment, loans are archived and `LoanLimit` is replenished

---

## Document Structure

This document presents a comprehensive design for a three-part DAML-based loan system:

1. **Loan Approval Workflow** – Models loan requests and approval/rejection flow
2. **Token Disbursement Workflow** – Introduces tokenized fund disbursement with bank-defined limits
3. **Repayment Workflow** – Supports partial repayments, enforces minimum repayments, and enables loan closure

Each section contains design insights, workflow logic, and example use cases to demonstrate the system’s integrity, scalability, and enterprise readiness.

---

# 1️ Loan Approval Workflow – Design & Approach

## Overview

Automates borrower-bank loan interactions with DAML smart contracts.

## Objective

- Eliminate manual approval
- Enforce programmatic rules
- Improve transparency and auditability

## Key Templates

- `LoanRequest`: Raised by the borrower
- `Loan`: Created on approval/rejection

## Access Control

- `Borrower`: Signatory of request
- `Bank`: Controller of decisions
- Both: Sign the final loan

## Workflow

1. Borrower submits a request
2. Bank approves/rejects
3. Loan contract is created accordingly

## Sample Case

- Yami → $1000 → Approved
- Akhil → $1,000,000 → Rejected
- Bheema → $100 → Pending

## DAML Summary

| Action             | Role         |
| ------------------ | ------------ |
| Create LoanRequest | Borrower     |
| Approve/Reject     | Bank         |
| Sign Loan          | Both Parties |

## Summary

Establishes a secure and modular base for digital loan processing.

---

# 2️ Token Disbursement Workflow – Design & Approach

## Overview

Disburses approved loans through a token-based wallet system under a central loan limit.

## Objective

- Enforce lending caps per bank policy
- Automate token creation & transfers
- Ensure disbursements respect loan approvals

## Key Templates

- `TokenWallet`: Account-based balance tracking
- `LoanLimit`: Tracks global and individual caps
- `LoanRequest`: Validates requested amount
- `Loan`: Manages disbursal

## Workflow

1. Bank sets loan limit
2. Borrower requests → validated
3. Approved loan → tokens minted to wallet

## Sample Cases

- Yami → $10k → Rejected (exceeds cap)
- Akhil → $5k → Approved → Disbursed
- Shashi → $93k → Final request, limit exhausted

## DAML Summary

| Action             | Role            |
| ------------------ | --------------- |
| Create TokenWallet | Issuer, Owner   |
| Credit/Debit Token | Issuer / Owner  |
| Approve Loan       | Bank            |
| Disburse Funds     | Bank / Borrower |

## Summary

Implements real-world token disbursal using DAML ledger and smart contract validation.

---

# 3️ Repayment Workflow – Design & Approach

## Overview

Adds structured repayment logic with contract auto-archiving and limit recovery.

## Objective

- Allow incremental loan repayments
- Enforce minimum repayment requirement
- Archive loan on full repayment

## Key Templates

- `LoanRepaymentRestriction`: 5% minimum payment logic
- `Loan`: Manages repayment status and closure

## Workflow

1. Restriction defines `minimumPay`
2. Borrower repays via wallet
3. Contract updates or archives on full repayment

## Sample Case

- Akhil: $500 → $800 → $3700 → Fully repaid and archived

## DAML Summary

| Action                  | Role     |
| ----------------------- | -------- |
| Create Restriction      | Bank     |
| Fetch Minimum Repayment | Bank     |
| Repay Loan              | Borrower |
| Update LoanLimit        | Bank     |

## 📝 Summary

Closes the loop on lending logic by supporting rule-based repayments and financial audit trails.
