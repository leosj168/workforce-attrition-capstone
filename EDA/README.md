# Six-Month Voluntary Attrition: Initial Analysis

## Question

Can information known at a monthly workforce snapshot help identify employees with higher relative risk of voluntary attrition in the following six months? I am exploring this question for Workforce Analytics at Kaiser Permanente so HR leaders can focus review and retention planning where it may be most useful. A risk score is not a prediction of certainty or a reason to make an individual employment decision.

## Data and Approach

I used employee-month records from the internal `Attrition.Employee_Comp_Monthly` stored procedure in SQL Server. Each row is one employee at one month end, so an employee can appear more than once. The notebook reviews about 3.3 million records across 36 snapshots; only snapshots with a complete six-month outcome window are used for comparisons with attrition. Identifiers and future termination details are not used as model predictors.

I checked data types, missing values, duplicate employee-months, numeric ranges, categorical groups, and unusual values. I then compared observed attrition across time, tenure, pay position, performance, manager context, job families, and departments. A simple Logistic Regression baseline uses tenure, the previous year's performance rating, and broad job and organizational groups. It trains on earlier snapshots and validates on later mature ones, with a six-month gap between their outcome windows. Its missing-value replacements are learned from training data only.

## Initial Findings

- Voluntary attrition is uncommon: about 1.3% of fully observed employee-month records have a positive six-month outcome. Accuracy alone would therefore be misleading.
- Employees who later left had about three years of median tenure, versus nearly nine years for those retained. Pay-position differences were smaller.
- Some named job families and departments have higher observed rates, but specialized work and repeated monthly records require context. These group rates are not individual risk scores or evidence of cause.
- Unusual salary, pay-position, and tenure values remain visible for review; I did not assume that every extreme value is an error.
- The baseline's validation average precision is about **3.1%**, versus an attrition rate of about **1.3%**. This is roughly 2.4 times a random ranking, but the absolute score is still modest. Average precision summarizes how well true cases are ranked near the top across review cutoffs.

## Next Step and Limits

The baseline is a comparison point, not the final model. In the separate modeling notebook, I compare feature sets and algorithms using calendar-based validation and a final held-out test period. These are associations in historical data; they do not explain why a person leaves. The underlying workforce data are internal and are not included in this repository.

## Notebook

[Exploratory Data Analysis notebook](REPLACE_WITH_GITHUB_NOTEBOOK_URL)
