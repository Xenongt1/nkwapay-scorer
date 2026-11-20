# NkwaPay loan scorer

Web app for the NkwaPay credit-scoring model built in ICS553 Machine Learning Essentials, Prosit 2 (Group 3, Ashesi University). Fill in a loan application and the applicant's earlier loans, or load one of five made-up examples, and the app returns the decision (approve, decline or refer), the chance of default against the 6.2% cost threshold, up to three plain-language reasons and the amount offered.

The model is a logistic regression trained on 66,520 Kenyan loans from the African Credit Scoring Challenge (Zindi). Applications from outside Kenya are always referred to a person, because the model is not validated there. This is a class project on public competition data, not a lending service.

Full analysis, report and tests: https://github.com/ASU-MICS-2028/mle-group-3-prosit-2

## Run locally

```bash
pip install -r requirements.txt
streamlit run app/app.py
```

`models/nkwapay-v1.joblib` holds the fitted coefficients, preprocessing averages and threshold only; it contains no loan records.
