# Predicting Six-Month Voluntary Attrition
## Capstone Final Report

**Leo Lin · Workforce Planning & Analytics · Kaiser Permanente**

**[Final Report notebook](https://github.com/leosj168/workforce-attrition-capstone/blob/main/Final%20Report/attrition_model_train_reg.ipynb)**

## 1. Define the Problem Statement

**Research question:** Can monthly workforce information be used to identify employees with a higher relative likelihood of voluntary attrition during the following six months, so Kaiser Permanente HR leaders can focus timely retention support?

When employees leave voluntarily, their departures can disrupt patient care and operations, add work for colleagues, and increase hiring and training costs. I wanted to see whether monthly HR records could help leaders decide where to begin a supportive retention review.

I used historical internal HR data with masked employee identifiers to build a ranking and compare it with an untargeted employee list. I also examined where review needs were concentrated. The data cover **Northern California and Executives**; use elsewhere in Kaiser Permanente would need separate evaluation.

The model gives HR a starting point. Leaders still need employee, job, manager and local business context before considering support. This project measures how well the model prioritizes review. Whether that support prevents exits or reduces costs remains a question for a future pilot.

## 2. Model Outcomes or Predictions

The task is **supervised binary classification**: learning from records with known outcomes to distinguish between two possibilities. For each employee at a monthly snapshot, the target is **1** for a qualifying voluntary exit within the next six months and **0** when no such exit is recorded. Other types of departure may still occur.

The model assigns a score to help order the review list. A selected cutoff determines who appears on that list. Scores show relative review priority; they do not give a verified probability that someone will leave. Likewise, risk-band counts describe review workload rather than expected departures.

## 3. Data Acquisition

I created the SQL Server stored procedure `Attrition.Employee_Comp_Monthly` to bring workforce, pay, performance and manager-team information together with exit outcomes. All data came through **one internal extract**, with no external dataset added.

The source contains **3,273,117 employee-month records across 36 snapshots**, from October 2023 to September 2026. Of these, **2,548,054 records through January 2026** have complete six-month outcomes. An employee can appear in several months, so these totals do not represent different people throughout the history.

The initial exploratory data analysis (EDA) found that voluntary exits were uncommon, around **1.3%**. Employees who later had a qualifying exit had a median tenure of roughly **three years**, compared with nine years among those without that outcome. Pay-position differences were smaller. These patterns helped guide the modeling, while unusual pay values and department differences raised questions to check with HR. They did not establish why employees left.

![Validation outcome balance and current workforce tenure and pay distributions](images/data_overview.png)

The left panel shows June 2025 validation, when **1.25%** of records had a qualifying voluntary exit in the following six months. The other panels describe September 2026 employees, whose complete six-month outcomes are not yet known.

**Data access and reproduction:** The workforce data are private and are not included in the repository. Rerunning the full notebook requires authorized SQL Server access and either a local `sql_connection.local.json` file or an `ATTRITION_SQLALCHEMY_URL` environment setting. Readers can review the saved aggregate results without database access.

## 4. Data Preprocessing/Preparation

Before modeling, I checked for duplicate records, missing values and unusual ranges. I excluded identifiers and future termination details from the inputs so the model could use only information available at the forecast date.

Preparation happens inside each modeling pipeline. Invalid numeric values are treated as missing and replaced with medians learned from training data. Inputs are scaled where appropriate. I also tested categorical inputs, filling missing categories and converting them to indicator columns through one-hot encoding, but they did not improve development results enough to keep. The final models use **13 numeric inputs**, including combined pay-and-tenure information.

An unknown outcome was never treated as evidence that an employee stayed. Removing whole recent or incomplete snapshots excluded **725,063 records** and left no missing outcomes in the retained history.

| Purpose | Period | Records |
|---|---|---:|
| Training | October 2023–November 2024 | 1,272,081 |
| Validation | June 2025 | 91,167 |
| Historical check | January 2026 | 90,988 |

These groups use part of the history with complete outcomes. Training helps the model learn patterns. Validation helps me choose the model and review cutoff. The historical check shows how that fixed choice performed later. Gaps between periods allow earlier six-month outcomes to finish before the next forecast date. Two earlier calendar checks within training guide input choices and model settings.

## 5. Modeling

I compared three classification models using the same selected inputs:

| Model | How it works | Settings tested |
|---|---|---|
| Logistic Regression | Combines inputs into a weighted score and provides an interpretable benchmark | Grid search over regularization, class weighting and training-only under-sampling |
| Random Forest | Combines decision trees to capture patterns a linear model may miss | Focused randomized search over tree complexity |
| Histogram Gradient Boosting | Builds small trees in sequence to improve the ranking | Grid search across eight simpler configurations |

All searches use the same two calendar-based development checks. Regularization limits overly specific patterns, while class weighting and under-sampling address the uncommon exit outcome during training.

The selected Gradient Boosting model limits each tree to **five leaves**, requires at least **1,000 records per leaf**, and uses an L2 regularization penalty of **10**. These controls reduce the chance of fitting details that do not carry forward. Its internal early-stopping check uses a random portion of training data, so that step does not follow calendar order. The notebook explains this limitation.

## 6. Model Evaluation

I used **average precision (AP)** as the main development measure because voluntary exits are uncommon. AP summarizes how well the ranking concentrates exit cases as the review list expands. Predicting no exit for everyone would be 98.75% accurate on validation while finding no exit cases, so accuracy alone would tell an incomplete story.

For HR, two measures make the tradeoff easier to understand. **Recall** is the share of later exit cases found on the list. **Precision** is the share of listed records that had a qualifying exit. At similar recall, higher precision means fewer reviews to find the same number of cases. ROC-AUC provides an additional check of ranking quality.

To qualify for selection, a model had to capture at least **25% of validation exit cases** and keep its training AP within **five percentage points** of development AP. This gap is a check for overfitting. Among models meeting both conditions, I chose the one with higher validation precision, using list share and AP to break ties. The 25% target is a comparison assumption, not an agreed HR workload.

| Model | AP | Precision | Recall | Reviews | Development gap |
|---|---:|---:|---:|---:|---:|
| Logistic Regression | 3.36% | 4.72% | 25.0% | 6,056 | 2.17 pp |
| Random Forest | 3.56% | 5.08% | 25.0% | 5,627 | 6.60 pp |
| **Gradient Boosting** | **3.75%** | **5.30%** | **25.0%** | **5,399** | **3.77 pp** |

Gradient Boosting was selected under this rule. It found the same **286 of 1,142 validation exit cases** as Logistic Regression with **657 fewer reviews, a 10.8% reduction**. Random Forest exceeded the allowed development gap. Section 13.1 of the [executed notebook](attrition_model_train_reg.ipynb) shows the review-workload comparison.

The January historical check gave a more cautious result. Gradient Boosting achieved **23.7% recall and 5.74% precision**, compared with **25.2% and 5.87%** for Logistic Regression. Because January had already been reviewed during development, it cannot serve as independent confirmation. Most exit cases remain outside the list, and the project has not measured whether retention support prevents departures.

**Findings and next steps:**

- **Check whether the improvement holds up.** The validation result suggests less review work for the same case coverage. Keep Logistic Regression as a benchmark and test a later period that has not informed development.
- **Resolve data questions with HR and data owners.** Confirm salary definitions, work arrangements and missing manager history before interpreting patterns or recommending support.
- **Agree on a manageable pilot.** The current model places **4,478 of 90,493 employees** in the High review-priority band. Ask HR what review capacity is realistic, then consider a supportive pilot after the data and performance checks. Measure whether the list helps reviewers and assess later retention outcomes separately.
- **Consider deployment after a successful pilot.** If further validation and an HR pilot show useful results, establish a controlled scoring workflow with clear review ownership, access controls and ongoing performance checks. Explore Databricks as a possible platform.
