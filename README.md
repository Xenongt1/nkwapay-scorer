# NkwaPay loan scorer

**Live app:** https://nkwapay-scorer.streamlit.app

A web app for the credit-scoring model built in ICS553 Machine Learning Essentials, Prosit 2 (Group 3, Ashesi University, November 2026). NkwaPay, a mobile-money lender in Accra, wants to score loan applications in Ghana with a model trained on Kenyan loans. The app shows what the finished scoring service does with one application.

Full analysis, report, audit, slides and tests: https://github.com/ASU-MICS-2028/mle-group-3-prosit-2

## What the app does

Enter a loan application and the applicant's earlier loans, or load one of five made-up examples, and press **Score application**. The app returns:

- **a decision**: approve, decline, or refer to a person;
- **the chance of default**, compared with the 6.2% decline threshold;
- **up to three reasons** in plain language, for declines and referrals;
- **the amount offered**, for approvals: the smaller of the request and the model's suggested limit, capped for a first loan.

### The form

| Field | Notes |
|---|---|
| Country | Kenya or Ghana. Any application outside Kenya is referred to a person (see below). |
| Product type | A dropdown. The data gives product codes only (`Type_1` to `Type_24`), not descriptions, so each option shows what the data says about it: how many Kenyan loans it had, their default rate, the typical amount and term, and how the model treats it. |
| Borrower | New or repeat loan. |
| Amount, term, date | Amount in local currency (KES for Kenya), term in days, date of the application. |
| Lender, number of lenders, share | The funding lender, how many lenders co-fund the loan and the total share they fund. |
| Earlier loans | One row per earlier loan: dates, amount, term, lender and outcome (0 repaid, 1 defaulted, blank if not yet due). Add rows with **+**; an empty table means a first-time borrower. |

### How it decides

| Decision | When |
|---|---|
| Approve | Chance of default at or below 6.2%. |
| Decline | Above 6.2% for an applicant with a repayment record, or any applicant with an earlier unpaid loan. |
| Refer | Above 6.2% for a first-time applicant (starter loan or human review instead of an automatic no), and every application from outside Kenya. |

### Lending policy and the reviewer step

The model decides the risk; the lending policy (sidebar) decides the money. Changing it needs no retraining and never changes the risk, the decision or the reasons.

| Setting | Default | Meaning |
|---|---|---|
| First-loan cap, Kenya | 5,000 KES | Largest first loan; the median repaid first loan in the data. |
| Starter loan for referrals | 10% | Share of the usual offer suggested for a referred first-time applicant. |
| First-loan cap, Ghana | not set | A cedi amount must be set before a reviewer can offer a pilot loan. |
| Maximum per customer | none | For a limit set by the Bank of Ghana (Directive s.3.5(5)). |

Approvals stay automatic. Under a referral, the reviewer step shows a suggested starter amount that is also the ceiling: the reviewer may lower it or decline (0), never raise it, and must give their name and a reason. Each decision is added to a review log that can be downloaded, so reviewers' choices can be checked for consistency. The review is a digital step because the Directive (s.2.3(1)) bars manual approval at a physical office.

The 6.2% comes from a cost matrix: on a GHS 500 loan with an 8% fee, a repaid loan earns about GHS 25 and a default loses about GHS 380, so approving pays only while the chance of default is below 25 / (25 + 380) = 6.2%.

### Product types

| Group | Products | How the model treats them |
|---|---|---|
| Learned separately | 11 types with 30 or more Kenyan loans, from `Type_1` (61,723 loans, 0.8% defaulted) to `Type_18` (30 loans) | Each has its own effect on the score. |
| Rare | 11 types with fewer than 30 Kenyan loans | Grouped together as "other". |
| Never seen | `Type_3` and `Type_8`, which occur only in Ghana, and any new product | No product information; scored as unknown. |

## Why Ghana is always referred

The model is trained on 66,520 Kenyan loans whose outcomes are known. We tested it on later Kenyan loans it had not seen, and checked that its percentages match what happened: before the market change in December 2022, loans it scored at about 1% defaulted at about 1%. That check needs real outcomes.

The 2,084 Ghana loans in the data have no outcomes, so the same check is impossible. They also look nothing like the Kenyan loans: 87% are on products never seen in Kenya and 96% are with lenders never seen in Kenya, and a classifier can tell a Ghanaian loan from a Kenyan one every time (AUC 1.0). Applied to Ghana, the model would decline 22% of applicants against 6.7% in Kenya, including almost every Type_2 and Type_8 loan. We cannot tell whether that is real risk or the model guessing outside what it knows.

So the app still computes a chance of default for a Ghana application, but a person makes the decision. The group's recommendation is a small, capped pilot in Ghana with human review, then recalibrating and validating the model on the pilot's repayment outcomes before any automated decisions.

## Model

| | |
|---|---|
| Classifier | Logistic regression, chosen from 14 candidates on time-ordered validation (train on older loans, test on newer ones) |
| Data | 66,520 Kenyan loans from the African Credit Scoring Challenge (Zindi), 1.63% defaulted |
| Performance | PR-AUC 0.145 and ROC-AUC 0.725 on later loans, about 7 times better than guessing |
| Loan limit | Ridge regression on the log amount, learned from repaid loans only; median error about 1,930 KES |
| Reasons | Exact log-odds contributions, grouped into plain-language reason families |
| Speed | About 20 ms per application |

The saved model (`models/nkwapay-v1.joblib`) holds the fitted coefficients, preprocessing averages and threshold only. It contains no loan records, and no loan data is in this repository.

## Run it on your machine

```bash
git clone https://github.com/Xenongt1/nkwapay-scorer.git
cd nkwapay-scorer
pip install -r requirements.txt
streamlit run app/app.py
```

Python 3.12 or newer.

## Repository layout

```
app/app.py         the web app
src/               scoring code shared with the analysis (features, reasons, scoring service, lending policy)
models/            the saved model, nkwapay-v1.joblib
examples/          five made-up applications: approve, decline, earlier unpaid loan, first loan, Ghana
requirements.txt   packages the app needs (scikit-learn must stay at 1.6.1 to load the saved model)
```

The live app redeploys automatically when this repository's `main` branch changes. The code here is a copy of the app, `src/`, `models/` and `examples/` from the group repository; it lives on a personal account because the course's GitHub organisation does not allow Streamlit to read its repositories.

## Limitations

- **Not a lending service.** A class project on public competition data.
- **Kenya only.** Scores for any other market are untested extrapolations.
- **The Kenyan market changed in December 2022.** After that the model ranks less well and its percentages spread too wide, so it needs recalibrating on recent outcomes, with human review after a shift.
- **The threshold depends on the fee.** At the fees actually charged on 7-day Kenyan loans (about 1.4%), break-even is 1.9%, not 6.2%, so a real deployment would set the threshold per product from its real price.
- **Only approved loans are in the data**, so the model learns from borrowers an earlier policy already accepted.
- **No credit-bureau check.** Ghana's Credit Reporting Act (Act 726, s.26(4)) requires a bureau search before deciding; the app does not do one.

## Team

ICS553 Machine Learning Essentials · MICS 2028 · Group 3: Mubarak Tijani (repository and pipeline, features, validation, classification, scoring service, web app), Eric Elikplim Sunu (loan limit, noise floor, derivations, reason codes), Clifford Yeboah (framing and cost matrix, exploratory analysis, Kenya to Ghana transfer), Adzeyena Opei (ethics and regulation, Model Card, Datasheet, report and slides).

Built with help from Claude Code; see the group repository's AI-use declaration.
