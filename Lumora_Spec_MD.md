# Lumora --- Trust Scoring Model: Technical Specification (v1)

**Purpose:** A build-ready spec for the scoring model engineer. Answers
each of the engineer\'s questions with concrete file types, data
structures, JSON schemas, a dummy-data plan, a recommended model
approach, and a deployment plan. **Status:** Early development. Some
items are flagged as *DECISION NEEDED* --- listed at the end.

## Quick orientation (read this first)

Three facts that shape everything below:

1.  **There are two human users of the product, plus Lumora as the layer
    between them.** Investors (lenders) and Entrepreneurs (borrowers).
    The model\'s job is to score the entrepreneur so the investor can
    decide. In the model objective doc, \"vendor\" = the entrepreneur
    being evaluated. (Confirm this --- see DECISION 1.)

2.  **Open Finance Brasil and PIX are NOT files. They are a JSON REST
    API** (FAPI-secured, OAuth2 + mutual-TLS, consent-gated). You never
    download a \"PIX.csv.\" You make authenticated API calls and get
    back JSON. Practically, nobody integrates with the raw central-bank
    network directly --- they use an aggregator (Belvo or Pluggy) that
    wraps it behind one clean JSON API **and ships a sandbox full of
    dummy data**. This solves both the integration problem and the
    training-data problem for now.

3.  **We cannot train an accurate bespoke model yet, and that\'s
    expected.** A \"will they repay?\" model is supervised --- it needs
    rows of {borrower features → repaid or defaulted}. We have zero loan
    outcomes (no one has borrowed on Lumora yet), and our friends\' Open
    Finance data is unlabeled and off-target. So v1 is a **transparent
    baseline scorer** trained/validated on public proxy data, that
    **learns from real outcomes as we issue loans**. More in §7.

## 1. The two users and what each wants the software to do

  --------------------------------------------------------------------------
                          **Investor (lender)**      **Entrepreneur
                                                     (borrower /
                                                     \"vendor\")**
  ----------------------- -------------------------- -----------------------
  **Role in the model**   *Consumes* the output      *Provides* the input,
                                                     *receives* a score +
                                                     portfolio

  **Wants the software    See vetted entrepreneurs   Submit their data once
  to...**                 with a payback-probability (financial + business +
                          score, a plain-language    personal), get a trust
                          explanation of *why*, a    score, see *why* and
                          business portfolio (story, *how to improve it*,
                          photos, financials), a     get an auto-built
                          recommended/eligible loan  business profile to
                          size; fund someone; track  show investors, receive
                          repayments; get a return.  funds, and repay over
                                                     time.

  **Trust signals they    Transparent scoring        A clear benefit in
  need** (from            methodology,               exchange for sharing
  interviews)             transaction-history proof, data; a path from
                          photos of the              \"invisible\" to
                          business/person,           fundable.
                          community-benefit/impact   
                          framing, borrower          
                          accountability.            
  --------------------------------------------------------------------------

**Lumora itself** is the third actor: the **scoring + routing + payments
layer** that sits between the two portals. The model is the engine of
that layer.

## 2. What problem the product solves

- **Business problem:** Informal women entrepreneurs in Brazil (MEI
  holders, autônomas, domestic-service providers) generate steady income
  but are invisible to banks --- no formal credit file, no collateral,
  grant/investor info is word-of-mouth. They can\'t access
  micro-capital. Investors who *want* to fund them have no trustworthy,
  vetted way to find and assess them.

- **Technical problem (the actual ML problem):** *Estimate the
  probability that an informal borrower with little or no formal credit
  history will repay a loan, using \"alternative\" / thin-file data
  (PIX + Open Finance transactions, plus qualitative signals), and
  explain the estimate well enough that an investor will trust it.* This
  is a **thin-file / alternative-data credit-default problem** --- a
  well-studied field, which is good news (see §7 for base models).

## 3. Data flow (one picture in words)

ENTREPRENEUR PORTAL SCORING SERVICE INVESTOR PORTAL

───────────────────── ────────────── ───────────────

structured form ─┐

financial API │ (raw inputs) ┌─ feature extraction ─┐ score +
explanation

doc scans (PDF) ├──────────────────► │ (OCR / NLP / vision) │ ─────►
business portfolio

photos / video │ │ → tabular features │ loan-size band

online presence ─┘ └─ scoring model (P_repay)┘ (all JSON)

│

writes to DB + object storage

Everything between services moves as **JSON**. Files
(scans/photos/video) are uploaded to **object storage**, and the system
passes around **URLs/IDs**, not the bytes.

## 4. Complete input list (no vagueness)

Inputs fall into 6 groups. \"Format\" = how the data actually arrives.
\"When\" = onboarding (one-time form/upload) vs API (pulled live after
consent).

### A. Quantitative --- financial transaction & account data

**Source:** Open Finance Brasil + PIX, accessed via an aggregator (Belvo
or Pluggy). **Format: JSON over REST API.** Consent-gated (LGPD).
Available only at *inference* time for real users; for *training* use
sandbox/synthetic JSON (see §6).

Representative normalized transaction object (Belvo-style; the native
OFB schema is similar with fields like transactionId, creditDebitType,
type, transactionName, amount, transactionDateTime):

{

\"id\": \"tx_8f3a\...\",

\"account_id\": \"acc_22b\...\",

\"type\": \"INFLOW\", // INFLOW \| OUTFLOW

\"method\": \"PIX\", // PIX \| TED \| BOLETO \| CARD \| CASH_DEPOSIT\...

\"amount\": 150.00,

\"currency\": \"BRL\",

\"value_date\": \"2026-05-12\",

\"description\": \"PIX RECEBIDO - cliente\",

\"counterparty_doc\": \"\*\*\*\", // masked CPF/CNPJ

\"category\": \"sales_income\",

\"balance_after\": 842.10

}

Account/balance object (one per linked account): account_id, type
(CHECKING/SAVINGS/PREPAID), balance_available, balance_current,
currency, institution, opened_date.

> **Engineer note:** start against the **Belvo or Pluggy sandbox** ---
> it returns this exact JSON shape with dummy users, so you can build
> the whole pipeline before we ever touch a real consent. Belvo also
> publishes field-by-field data dictionaries (CSV/XLSX) on their public
> GitHub, and a Python SDK (pip install belvo-python).

### B. Quantitative --- engineered features (what the model actually eats)

Derived from group A by our feature code. These are the columns of the
training table. Examples (not exhaustive --- finalize together):

- Income: avg monthly inflow, inflow volatility (std/CV), \# distinct
  paying counterparties, months with positive inflow, trend slope.

- Cash management: avg end-of-day balance, \# negative-balance days,
  inflow/outflow ratio.

- Stability/tenure: account age (months), transaction history length,
  active days/month.

- PIX behavior: PIX inflow share, recurring-payer count.

**Structure:** a single flat row of numbers per borrower → one record in
a table (pandas.DataFrame row / a row in a Postgres table). **This flat
numeric row is the model\'s real input.**

### C. Qualitative --- structured profile (form → JSON)

**Source:** entrepreneur fills a form in the app. **Format: JSON**
(validated against a schema).

- Business: type/sector, description, age of business, \# employees,
  location (city/state/neighborhood), online sales channels.

- Demographic: age, gender, residence city/state, **\# of times moved /
  years at current address**, household info. (We focus on women but
  capture gender as a field.)

> ✅ **RESOLVED --- demographics are NOT score inputs.** Gender,
> location, and residential-moves are collected for **portfolio display
> and bias-monitoring only**; they must **not** feed the scoring model.
> Rationale: under LGPD the \"credit protection\" legal basis cannot be
> used for sensitive data, and the non-discrimination principle bars
> using personal data for discriminatory purposes (gender is
> increasingly treated as sensitive). Technically these fields add
> overfitting on small/unlabeled data rather than real accuracy --- the
> predictive signal lives in behavior (income stability, history,
> repayment), which the scorecard already uses. Engineer: keep these
> columns in the profile table, exclude them from the feature set passed
> to the scorer, and use them to *check* the model isn\'t
> discriminating. (Get a Brazilian data/credit lawyer to confirm before
> launch.)

### D. Qualitative --- documents (file uploads → OCR → JSON)

**Source:** entrepreneur uploads. **Format: PDF, JPEG, PNG**
(scans/photos of paper). Stored in object storage; an OCR/parse step
turns them into structured JSON fields.

- Bills, receipts, informal financial statements/ledgers, ID, proof of
  address, MEI registration.

- **Output of this step:** extracted fields (e.g.,
  monthly_revenue_declared, business_name, cnpj_mei) as JSON + a
  confidence score per field.

### E. Qualitative --- media

**Source:** entrepreneur uploads. **Format: JPEG/PNG (photos), MP4/MOV
(video).** Stored in object storage. Used mainly for the
**investor-facing portfolio** (trust signal). Optionally, a vision model
derives light features (e.g., \"storefront present: yes/no\") --- keep
optional for v1.

### F. Qualitative --- online presence / reputation

**Source:** links the entrepreneur provides (Instagram, WhatsApp
Business, Facebook, marketplace pages). **Format:** scraped/queried →
JSON/text. Possible features: follower count, posting recency, review
counts/ratings, years active.

### Training-only input: LABELS

For *any* supervised model you also need the target column:

outcome: { \"repaid_on_time\" \| \"repaid_late\" \| \"defaulted\" } //
or a 0/1 default flag

We have **none of these yet**. This is the central constraint --- see
§7.

**Summary of file types you\'ll touch:** JSON (financial API, forms,
model I/O) · PDF/JPEG/PNG (document scans) · MP4 (video) · CSV/Parquet
(training tables, exports). You will **not** receive a \"PIX file.\"

## 5. Complete output list & structure

There is **one core model output** (the score object) plus **two derived
products** (portfolio, suggestions). All JSON. \"Output structure\" =
the JSON schema the scoring API returns; both portals render from it.

### 5.1 Trust score object (the model\'s primary output)

{

\"borrower_id\": \"ent_4471\",

\"model_version\": \"0.3.1\",

\"scored_at\": \"2026-06-16T10:00:00Z\",

\"payback_probability\": 0.86, // P(repay) in \[0,1\] --- the real model
output

\"trust_score\": 720, // 300--850 display scale mapped from probability

\"risk_band\": \"B\", // A best ... E worst

\"confidence\": 0.74, // how much data backed this (data completeness)

\"recommended_loan_brl\": 1200,

\"max_eligible_loan_brl\": 2000,

\"explanation\": { // transparency --- investors require this

\"top_positive_factors\": \[

{\"feature\": \"stable_monthly_inflow\", \"impact\": 0.21, \"plain\":
\"Steady income for 9+ months\"},

{\"feature\": \"many_distinct_payers\", \"impact\": 0.14, \"plain\":
\"Broad, recurring customer base\"}

\],

\"top_negative_factors\": \[

{\"feature\": \"high_inflow_volatility\", \"impact\": -0.09, \"plain\":
\"Income varies a lot month to month\"}

\],

\"method\": \"gradient-boosted scorecard + SHAP attributions\"

},

\"data_completeness\": { // drives confidence + improvement tips

\"financial_api\": true, \"documents\": true, \"photos\": false,
\"online_presence\": false

}

}

> **Metric note for the engineer (important):** the investor who said
> \"≥97% accuracy\" is using the wrong yardstick. With repayment
> that\'s, say, 90%+, a model that predicts \"everyone repays\" is \~90%
> \"accurate\" and useless. **Use AUC-ROC, Gini, and KS** for the
> default model, and calibration (Brier/reliability) for the
> probability. Report those, not \"accuracy.\"

### 5.2 Business portfolio object (derived; for investor display + given back to entrepreneur)

{

\"borrower_id\": \"ent_4471\",

\"display_name\": \"Maria\'s Salgados\",

\"sector\": \"food / catering\",

\"city\": \"Recife, PE\",

\"story\": \"...short narrative...\",

\"media\": \[\"s3://\.../store1.jpg\", \"s3://\.../intro.mp4\"\],

\"financial_summary\": {

\"avg_monthly_revenue_brl\": 1800, \"months_of_history\": 11,
\"active_channels\": \[\"WhatsApp\",\"Instagram\"\]

},

\"verification\": {\"identity\": \"verified\", \"address\":
\"verified\", \"mei\": \"verified\"}

}

### 5.3 Improvement suggestions (derived; entrepreneur-facing)

{ \"borrower_id\": \"ent_4471\",

\"suggestions\": \[

{\"action\": \"Add 2 photos of your storefront\", \"expected_lift\":
\"small\", \"reason\": \"Investors trust visual proof\"},

{\"action\": \"Connect your bank for the full 12 months\",
\"expected_lift\": \"medium\", \"reason\": \"More history sharpens your
score\"}

\]}

**So:** investor sees 5.1 (score + explanation) and 5.2 (portfolio).
Entrepreneur sees a friendly version of 5.1, plus 5.2 and 5.3.

## 6. What the dummy data looks like + how to make it

Goal: enough realistic, *labeled* records to build and test the pipeline
before real data exists. Three layers:

1.  **Financial inputs → Belvo/Pluggy sandbox.** Free dummy users return
    the exact JSON of §4-A through real API calls, so the whole
    ingestion path is built against the real shape.

2.  **A labeled training table → public proxy dataset.** Use the
    **Kaggle \"Home Credit Default Risk\"** dataset --- it\'s literally
    alternative-data default prediction (thin-file borrowers, no/low
    bureau history, default labels). Train and validate the baseline
    pipeline on it to prove the modeling works end-to-end. (Caveat:
    it\'s not Brazilian and not our exact features --- it\'s a
    *scaffold*, not the final model.)

3.  **Synthetic Lumora-shaped records → generate with Faker (pt_BR
    locale).** Produce N borrower JSONs matching our combined schema,
    each with a *simulated* outcome label, so the API/portal/score flow
    can be demoed with our own field names. One record:

{

\"borrower_id\": \"ent_demo_001\",

\"profile\": {\"sector\": \"food\", \"business_age_months\": 14,
\"city\": \"Salvador, BA\",

\"age\": 34, \"gender\": \"F\", \"years_at_address\": 3},

\"financial_features\": {\"avg_monthly_inflow_brl\": 1650,
\"inflow_cv\": 0.38,

\"distinct_payers\": 27, \"account_age_months\": 22,
\"neg_balance_days\": 4},

\"documents\": \[{\"type\": \"mei_certificate\", \"status\":
\"verified\"}\],

\"media_count\": {\"photos\": 3, \"videos\": 1},

\"label\": {\"outcome\": \"repaid_on_time\"} // simulated, for testing
only

}

> Keep synthetic labels clearly tagged. They\'re for plumbing/demo,
> never for claiming accuracy.

## 7. The model itself --- recommended approach

**Don\'t build one giant model. Build a small pipeline:**

extractors (OCR, NLP on descriptions, optional vision)

→ produce features

→ FEATURE STORE (one flat table)

→ scorer: gradient-boosted trees (XGBoost or LightGBM) → P(repay)

→ SHAP for the per-borrower explanation (§5.1)

Why gradient-boosted trees and not deep learning: they\'re the industry
standard for tabular credit scoring, work with modest data, handle
missing values, and are **explainable** (SHAP gives the \"why,\" which
investors demand). Deep models need data we don\'t have and hurt
transparency.

**Base models / repos to start from (your \"take a good model and adapt
it\" plan):**

- deburky/boosting-scorecards --- interpretable XGBoost credit
  *scorecards* (WOE-style, the bank-standard format). Closest to what
  you want as a skeleton.

- github.com/topics/credit-scoring --- includes an **\"alternative
  credit scoring for MSMEs using mobile-money + behavioral data
  (FastAPI + ML)\"** repo that mirrors our architecture almost exactly.

- mourarthur/awesome-credit-modeling --- curated papers/resources on
  alternative-data and P2P credit risk.

- Kaggle Home Credit notebooks --- battle-tested feature engineering for
  thin-file default.

**About Kiva (reality check worth internalizing):** Kiva\'s very low
default rate is driven *substantially by its field-partner network and
group/social accountability*, not by a magic algorithm. This matches
your own Week 2 finding (solidarity groups → 97--98% repayment) and what
the MFI experts said. **Takeaway:** the scoring model is necessary but
not sufficient. The biggest repayment lever is structural --- group
lending, community partners, staged loan sizes, relationships. Build the
score, but don\'t expect it alone to deliver Kiva-level performance;
design the *product* (group features, partner channels, small first
loans) for repayment too.

**The cold-start problem and how we get to \"accurate\" (phased):**

- **Phase 0 (now):** ship a **transparent rules-based scorecard** ---
  hand-weighted, sensible features (income stability, history length,
  verification status). No training labels needed, fully explainable,
  defensible to investors. In parallel, stand up the XGBoost pipeline on
  Kaggle data to prove it works.

- **Phase 1 (first loans):** every loan you issue (your Week-4 plan to
  pick 2--3 entrepreneurs) becomes a **labeled row** once it
  repays/defaults. This is your real training set forming.

- **Phase 2:** once you have enough outcomes (rough order: low hundreds
  of completed loans), retrain the XGBoost model on *real Lumora data*
  and let it replace/augment the rules. The \"optimize and train for
  accuracy over time\" requirement is this loop.

> **This is the honest version of \"AI scoring,\" and ML is the explicit
> goal --- not the scorecard.** The scorecard is the on-ramp: every loan
> it approves becomes a labeled training row once it repays or defaults,
> so the rules-based version is literally the machine that manufactures
> the data the ML model needs to exist. Seed the scorecard\'s rules from
> established informal-credit practice (microfinance cash-flow
> underwriting, character-based lending, progressive lending) and from
> the team\'s MFI interviews --- see §10 for the starter scorecard.

## 8. How to ship it (deployment for someone who only knows the basics)

You don\'t need deep AWS/DevOps. Target shape:

\[ PWA front-end \] ──HTTPS/JSON──► \[ REST API (FastAPI) \] ──► \[
scoring service \]

web + installable auth, validation loads model, returns JSON

on cheap phones │

├──► \[ Postgres \] (profiles, scores, loans)

└──► \[ Object storage \] (scans, photos, video)

**Front-end → build a PWA (Progressive Web App), not a native app
first.** A PWA is a website that installs to the home screen and runs in
any mobile browser. This directly answers \"must run on the most basic
phones\": no app-store approval, tiny footprint, works on low-end
Android, one codebase for web + mobile. Go native later only if you need
camera/offline depth.

**Back-end → wrap the model in FastAPI** (Python, same language as the
model --- no glue code). One endpoint POST /score takes a borrower JSON,
returns the §5.1 object.

**Hosting given \"I only know basic services\" → use a managed PaaS so
there\'s no server to babysit.** Two equally fine routes:

- **Simplest:** Render, Railway, or Fly.io --- git push deploys the
  FastAPI app; managed Postgres included; generous free/cheap tiers.

- **Stay in AWS:** **AWS App Runner** (point it at a container/repo, it
  runs the API --- no EC2/Kubernetes) + **RDS Postgres** + **S3** for
  file storage. App Runner is the \"I don\'t know AWS\" path; avoid raw
  EC2/EKS for now.

**Model packaging:** save the trained model as a file (joblib/pickle for
XGBoost, or ONNX), load it on API startup, keep it in memory. For v1 the
model is small enough to live inside the API process --- no separate ML
infra needed.

**Security/compliance (don\'t skip):** consent capture per LGPD before
any Open Finance pull; encrypt files at rest; never log raw
CPF/financial data; the aggregator handles the FAPI/mTLS/cert burden so
you don\'t have to.

## 9. Decisions --- RESOLVED

1.  **\"Vendor\" = the entrepreneur/borrower being scored.** Confirmed.
    Two users only: investor + borrower.

2.  **Aggregator: Belvo for development.** Build against Belvo\'s
    sandbox + Python SDK + data dictionaries. Evaluate Pluggy
    (Brazil-native) for production later. Not locked in --- shapes are
    similar.

3.  **v1 scope: rules-based scorecard first, ML pipeline proven on
    Kaggle in parallel.** ML is the stated end goal; the scorecard is
    the on-ramp that generates its training labels (§7, §10).

4.  **Demographics are NOT score inputs.**
    Gender/location/residential-moves are display + bias-monitoring
    only; behavioral/financial features drive the score (§4-C). Lawyer
    to confirm pre-launch.

5.  **\"Lock the phone/app on default\" is OUT of v1.** Not to be built
    pending a legal/ethics review (consumer-protection + LGPD + mission
    risk).

6.  **v1 scores individuals.** Design the data model so group lending
    can be added later without a rebuild (it\'s the biggest repayment
    lever, per the team\'s research and the Kiva finding).

## 10. Starter rules-based scorecard (v1) --- build this first

A transparent, hand-weighted scorecard the engineer can implement
immediately --- no training data needed. Criteria are drawn from
established informal-credit evaluation: microfinance **cash-flow
underwriting** (can the business service the loan?),
**character/behavior** (tenure, consistency), **verification** (trust
signals), and **progressive lending** (everyone starts small). Refine
the weights with the team\'s MFI interview notes.

**How it works:** each borrower earns points across the factors below;
points sum to a raw score, mapped to a 300--850 display score, a risk
band, and a starting loan size. Every input is a behavioral/financial or
verification signal --- **no demographic fields**.

  ------------------------------------------------------------------------------------
  **\#**         **Factor (signal)**  **Source**     **Why it matters** **Indicative
                                                                        weight**
  -------------- -------------------- -------------- ------------------ --------------
  1              **Debt-service       Open Finance / Core MFI test: can **25%**
                 capacity** = avg     PIX (§4-A,B)   the cash flow      
                 monthly inflow ÷                    cover the loan?    
                 proposed monthly                                       
                 repayment                                              

  2              **Income stability** §4-B           Steady beats       **20%**
                 (lower inflow                       high-but-erratic   
                 volatility / CV; \#                                    
                 months with positive                                   
                 inflow)                                                

  3              **Business/account   §4-A,B         Longevity strongly **15%**
                 tenure** (account                   predicts survival  
                 age, months of                      & repayment        
                 transaction history)                                   

  4              **Customer base      §4-B           Diversified        **10%**
                 breadth** (#                        revenue = lower    
                 distinct paying                     shock risk         
                 counterparties,                                        
                 recurring payers)                                      

  5              **Cash management**  §4-B           Buffer behavior    **10%**
                 (avg end-of-day                     signals discipline 
                 balance; \#                                            
                 negative-balance                                       
                 days)                                                  

  6              **Verification /     §4-C,D         Reduces fraud;     **10%**
                 trust** (ID +                       investors require  
                 address + MEI                       it                 
                 verified; documents                                    
                 present)                                               

  7              **Profile            §4-E,F         Trust signal for   **10%**
                 completeness**                      investors;         
                 (photos, business                   correlates with    
                 description, online                 engagement         
                 presence linked)                                       
  ------------------------------------------------------------------------------------

> Weights are a starting point to calibrate, not gospel. They sum to
> 100%. Capacity + stability + tenure (factors 1--3) deliberately carry
> 60% --- that\'s where microfinance practitioners put the weight.

**Scoring logic (pseudocode the engineer can implement directly):**

def score_borrower(features) -\> dict:

pts = 0

\# 1. Debt-service capacity (ratio of income to proposed repayment)

dscr = features.avg_monthly_inflow /
max(features.proposed_monthly_repayment, 1)

pts += band(dscr, \[(3.0, 25), (2.0, 18), (1.5, 10), (1.0, 4)\],
default=0)

\# 2. Income stability (coefficient of variation --- lower is better)

pts += band(features.inflow_cv, \[(0.2, 20), (0.4, 14), (0.6, 7)\],
default=2, reverse=True)

\# 3. Tenure (months of history)

pts += band(features.account_age_months, \[(24, 15), (12, 10), (6, 5)\],
default=1)

\# 4. Customer breadth

pts += band(features.distinct_payers, \[(20, 10), (10, 6), (5, 3)\],
default=0)

\# 5. Cash management (fewer negative-balance days is better)

pts += band(features.neg_balance_days, \[(0, 10), (2, 6), (5, 3)\],
default=0, reverse=True)

\# 6. Verification (count of verified items)

pts += features.verified_count \* 3 \# cap at 10

\# 7. Profile completeness (0--1)

pts += round(features.profile_completeness \* 10)

raw = min(pts, 100)

display = 300 + round(raw / 100 \* 550) \# → 300--850

band_label = (\"A\" if raw\>=80 else \"B\" if raw\>=65 else

\"C\" if raw\>=50 else \"D\" if raw\>=35 else \"E\")

\# Progressive lending: start small, grow with repayment history

base_loan =
{\"A\":2000,\"B\":1500,\"C\":1000,\"D\":600,\"E\":300}\[band_label\]

return {\"trust_score\": display, \"risk_band\": band_label,

\"recommended_loan_brl\": base_loan,

\"factor_points\": {\...}} \# keep per-factor points → the §5.1
explanation

*(band() = helper returning the points for the first threshold the value
clears; reverse=True for \"lower is better\" factors.)*

**Why this also satisfies the explainability requirement:** because you
compute and keep factor_points, the per-borrower \"why\" (§5.1
explanation) falls out for free --- no SHAP needed for the scorecard
stage. When you later swap in the XGBoost model, SHAP produces the same
shape of explanation, so the investor-facing UI doesn\'t change.

**Progressive lending (bake in from day one):** cap first loans small
for everyone regardless of band, and raise the ceiling only after
on-time repayments. This is the safest lever for a thin-file book and is
what keeps default low while the ML model is still learning.

*Prepared for the Lumora scoring build. All six decisions in §9 are
resolved; §10 is the concrete v1 to build now. Open legal items to
confirm before launch: LGPD use of Open Finance data for scoring,
demographic-data handling, and the (currently excluded)
default-enforcement mechanism.*
