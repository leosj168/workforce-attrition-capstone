# Predicting Six-Month Voluntary Attrition
## Capstone Final Report

**Leo Lin · Workforce Planning & Analytics · Kaiser Permanente**

**[Final Report Notebook](https://github.com/leosj168/workforce-attrition-capstone/blob/main/Final%20Report/attrition_model_train_reg.ipynb)**

## 1. Define the Problem Statement

**Problem Statmenet:** Can monthly workforce information be used to identify employees with a higher relative likelihood of voluntary attrition during the following six months, so Kaiser Permanente HR leaders can focus timely retention support?

When employees leave voluntarily, their departures can disrupt patient care and operations, add work for colleagues, and increase hiring and training costs. I wanted to see whether monthly employee workforce data could help leaders decide where to begin a supportive retention review.

I used historical internal workforce data with masked employee identifiers to build a ranking and compare it with an untargeted employee list. I also examined where review needs were concentrated. The data cover business units in **Northern California and Executives**; use elsewhere in Kaiser Permanente would need separate evaluation.

The model gives HR a starting point. Leaders still need employee, job, manager and local business context before considering support. This project measures how well the model prioritizes review. Whether that support prevents exits or reduces costs remains a question for a future pilot.

## 2. Model Outcomes or Predictions

This is **binary classification**: each employee-month is labeled **1** for a qualifying voluntary exit within six months and **0** otherwise. Other types of departure may still occur.

Scores rank review priority. A cutoff creates the list; scores are not verified departure probabilities, and risk bands are not expected exit counts.

## 3. Data Acquisition

I built the SQL Server procedure `Attrition.Employee_Comp_Monthly` to combine workforce, pay, performance, manager-team and exit information.

The source has **3,273,117 employee-month records across 36 snapshots**, October 2023–September 2026. **2,548,054 records through January 2026** have complete outcomes. Employees can appear in several months.

Initial exploratory analysis found exits were uncommon, around **1.3%**, and shorter median tenure among employees who later left: roughly **three years**, versus nine without the target exit. These patterns guided modeling without establishing causes.

![Validation outcome balance and current workforce tenure and pay distributions](images/data_overview.png)

June 2025 validation has a **1.25%** exit rate. The tenure and pay panels describe September 2026 employees, whose complete outcomes are unknown.

**Access:** Workforce data are private and excluded from the repository. A full rerun needs authorized SQL access and `sql_connection.local.json` or `ATTRITION_SQLALCHEMY_URL`. Saved aggregate results can be reviewed without database access.

## 4. Data Preprocessing/Preparation

I checked duplicates, missing values and unusual ranges, and excluded identifiers and future termination details. Pipelines learn numeric replacements and scaling from training data. Tested categories did not improve development results enough to keep. All models use **13 numeric inputs**, including a pay–tenure interaction.

The fitted pay-position and manager-turnover flags combine missing values with negative conditions. This limitation needs a development-tested improvement before changing the model.

I excluded **725,063 recent or incomplete records**, never treating unknown outcomes as retention. Evaluation uses this subset of complete history:

| Purpose | Period | Records |
|---|---|---:|
| Training: learn patterns | October 2023–November 2024 | 1,272,081 |
| Validation: choose model and cutoff | June 2025 | 91,167 |
| Historical check: evaluate the fixed choice | January 2026 | 90,988 |

Calendar gaps allow earlier six-month outcomes to finish before the next forecast. Two earlier checks within training guide input choices and settings.

## 5. Modeling

| Model | Approach and settings tested |
|---|---|
| Logistic Regression | Weighted-score benchmark; grid search over regularization, weighting and training-only under-sampling |
| Random Forest | Combined trees; randomized search over tree complexity |
| Histogram Gradient Boosting | Sequential small trees; grid search across eight simpler configurations |

All models use the same selected inputs and development periods. Logistic Regression selects 1:5 under-sampling, no class weighting, `C=1` and L2 regularization. Boosting uses **five leaves per tree**, at least **500 records per leaf**, and an L2 penalty of **5**.

The outer checks follow calendar order. Boosting's internal early-stopping split is random within training, a limitation discussed in the notebook.

## 6. Model Evaluation

**Average precision (AP)** summarizes ranking quality and guides development. Accuracy alone is misleading: predicting no exit would be **98.75% accurate** while finding no cases. **Recall** means cases found; **precision** means the list's observed exit share. ROC-AUC adds a ranking check.

Selection requires at least **25% validation recall** and a training-to-development AP gap within **five percentage points**. Among eligible models, higher precision wins; list share and AP break ties. HR has not agreed to the resulting workload.

| Model | AP | Precision | Recall | Reviews | Development gap |
|---|---:|---:|---:|---:|---:|
| Logistic Regression | 2.67% | 3.59% | 25.0% | 7,964 | 2.15 pp |
| Random Forest | 3.39% | 5.03% | 25.0% | 5,683 | 6.14 pp |
| **Gradient Boosting** | **3.51%** | **5.03%** | **25.1%** | **5,701** | **4.52 pp** |

**Gradient Boosting remains selected.** It finds **287 of 1,142 validation cases**, versus **286** for Logistic Regression, with **5,701 versus 7,964 reviews**: **2,263 fewer (28.4%)**. Random Forest fails the development-gap rule. An equally sized untargeted list would contain about **71 cases on average**, giving boosting **4.0 times** the exit concentration. This is a calculated reference, not a trial result. See Section 13.1 of the [executed notebook](attrition_model_train_reg.ipynb).

**January is mixed:** boosting has higher precision (**5.59% versus 4.34%**) and AP (**4.09% versus 3.62%**), but lower recall (**23.4% versus 24.2%**). January has already informed development, so it is not independent confirmation. Most exit cases remain outside the list, and no retention effect has been measured.

**Current workforce:** **4,987 of 90,493 employees** are in High, averaging **3.0 years** of tenure and **$80,834** annual salary. Manager history has **7.0%** missing six-month values and **11.8%** missing twelve-month values. These profiles frame HR questions, not causes.

The corrected feature check compares the same **20,000 validation records** before and after **ten shuffles per input**. Salary and tenure show the largest ranking changes, followed by their interaction. This does not identify causes or effective retention actions.

**Next steps:**

- Verify pay definitions and remaining coverage gaps; test improved missing-flag handling on development data.
- Keep Logistic Regression as a benchmark and test later untouched outcomes.
- Agree on a limited retention-support trial; evaluate review usefulness and retention outcomes separately.
- Consider controlled deployment after successful validation and a useful trial, with ownership, access controls and monitoring. Explore Databricks as a possible platform.
