# RetentionLab — Customer Retention Analytics

Predicting customers likely to make no purchase in the next 60 days and prioritizing them for retention campaigns.

**ROC-AUC: 0.815** · **Top-10% Precision: 96.4%** · **Top-10% Lift: 1.35×** · 

## Key Results

| Metric | Result |
|---|---:|
| ROC-AUC | 0.815 |
| Accuracy | 79.4% |
| Top-10% Precision | 96.4% |
| Top-10% Lift | 1.35× |
| Top-30% Capture | 38.9% |


## Project Overview

RetentionLab is an end-to-end customer retention analytics project built using transactional e-commerce data.

The project focuses on three main questions:

1. Which customers are likely to make no purchase during the next 60 days?
2. Which customer behaviors are most informative for predicting future inactivity?
3. How can customers be prioritized when retention campaign capacity is limited?

The project covers data preparation, exploratory analysis, point-in-time feature engineering, churn modeling, model interpretation, and customer prioritization.

## Business Problem

A retention team may have thousands of customers but limited capacity for targeted campaigns. Instead of treating every customer equally, the business can use behavioral data to identify customers who are more likely to become inactive and prioritize them for retention efforts.

RetentionLab models **60-day future inactivity** as a practical short-term proxy for churn.

The target is defined as:

- `1` — customer makes no purchase during the following 60 days
- `0` — customer makes at least one purchase during the following 60 days

This is a short-term inactivity definition rather than a claim that the customer has permanently churned.

## Dataset

The project uses the **UCI Online Retail II** dataset, which contains transactional data from a UK-based online retailer.

The dataset includes transactions from December 2009 through December 2011 and contains information such as:

- Invoice number
- Product code
- Product description
- Quantity
- Invoice date
- Unit price
- Customer ID
- Country

For customer-level modeling, transactions without a `Customer ID` were excluded because they could not be reliably attributed to a specific customer.

### Data Preparation

The analysis keeps valid purchase transactions by applying the following filters:

- `Quantity > 0`
- `Price > 0`
- Exclude cancelled invoices
- Require a known `Customer ID` for customer-level modeling

Revenue was calculated as:

Revenue = Quantity × Price


The final model was evaluated on a held-out chronological test period. The classification threshold was selected using a separate chronological validation period.

For campaign prioritization, customers were ranked by predicted inactivity risk. Among the top 10% of customers by predicted risk, 96.4% subsequently made no purchase during the following 60 days.

## Exploratory Data Analysis

The exploratory analysis focused on customer purchasing behavior, revenue concentration, and recency.

### Customer Activity

Customer purchasing behavior is highly uneven:

- The median customer placed **3 orders**.
- **27.6%** of customers made only one purchase.
- Only **5.1%** of customers placed 21 or more orders.
- However, these 21+ order customers accounted for approximately **45.6% of total revenue**.

This indicates that a relatively small group of highly active customers contributes a substantial share of historical revenue.

### Customer Recency

Customer recency is strongly right-skewed:

- Median recency: **95 days**
- Mean recency: approximately **200 days**
- **50.8%** of customers had not purchased for more than 90 days.
- **40.8%** had not purchased for more than 180 days.

Recency was also one of the most informative behavioral features in the final predictive model.

### Interpurchase Behavior

The median time between purchases was approximately **25 days**, while the mean was approximately **52 days**.

About **74.4%** of observed interpurchase gaps were 60 days or less.

This analysis motivated the use of a **60-day future inactivity window** as a practical short-term proxy for churn.

## Point-in-Time Feature Engineering

To avoid data leakage, customer features were constructed using only information that would have been available at each prediction date.

For each prediction date:

- **Past window:** transactions occurring on or before the prediction date
- **Future window:** transactions occurring after the prediction date and within the following 60 days
- **Target:** whether the customer made no purchase during the future 60-day window

The target was defined as:

churn_next_60d = 1  → no purchase in the next 60 days
churn_next_60d = 0  → at least one purchase in the next 60 days

## Modeling

Several models were evaluated using the same point-in-time features and chronological data splits:

| Model | ROC-AUC | Accuracy | Purpose |
|---|---:|---:|---|
| Dummy Classifier | — | 0.716 | Baseline |
| Logistic Regression | 0.804 | 0.782 | Linear baseline |
| Random Forest | 0.815 | 0.794 | Nonlinear model |
| HistGradientBoosting | 0.814 | 0.794 | Boosting benchmark |

The models were trained using customer-level behavioral features such as recency, order frequency, revenue, product diversity, tenure, and recent 30/90-day activity.

A **Random Forest** was used as the final model because it provided strong predictive performance and allowed straightforward analysis of feature importance.

The final model was retrained using the training and validation periods, while the test period was kept for final evaluation.

To support campaign prioritization, the model produces a probability of future 60-day inactivity for each customer. Customers can then be ranked from highest to lowest predicted risk rather than treated using a single binary classification threshold.

## Validation Strategy

Because the goal is to predict future customer inactivity, the data was split chronologically rather than randomly.

| Period | Purpose |
|---|---|
| 2010-06-01 → 2011-02-01 | Training |
| 2011-03-01 → 2011-05-01 | Validation |
| 2011-06-01 → 2011-09-01 | Final test |

The validation period was used to select the classification threshold. The final model was then retrained using the training and validation data and evaluated once on the held-out test period.

This approach prevents future customer behavior from influencing model training or threshold selection.

For campaign prioritization, customers were also ranked by predicted inactivity probability. This makes it possible to evaluate performance at different campaign capacities, such as the top 10%, 20%, or 30% of customers.

## Model Interpretation

Feature importance was analyzed using permutation importance measured by the decrease in ROC-AUC when a feature's values were randomly shuffled.

The most informative features were:

| Feature | Permutation Importance |
|---|---:|
| Recency | 0.0252 |
| Orders | 0.0163 |
| Revenue | 0.0112 |
| Unique products | 0.0062 |
| Tenure | 0.0039 |

### Key Findings

- **Recency** was the strongest feature. Customers who had gone longer without purchasing provided an important signal for future inactivity.
- **Order frequency** was also highly informative, indicating that customers' historical purchasing activity helps distinguish future inactive customers.
- **Historical revenue** contributed additional predictive information beyond order frequency.
- **Customer tenure** and **product diversity** provided smaller but still measurable contributions.

Permutation importance measures how much model performance changes when a feature is shuffled. It should therefore be interpreted as a measure of predictive usefulness rather than a causal effect.

The importance of correlated behavioral features may also be distributed across several variables, so feature importance should not be interpreted as an independent contribution from each feature.

## Customer Prioritization

Instead of applying a single classification threshold to all customers, the model was also evaluated as a ranking system.

Customers were ranked from highest to lowest predicted probability of making no purchase during the following 60 days. This allows a retention team to target a specific percentage of customers based on available campaign capacity.

| Customer Segment | Customers | Precision | Lift |
|---|---:|---:|---:|
| Top 10% | 2,036 | 96.4% | 1.35× |
| Top 20% | 4,073 | 94.4% | 1.32× |
| Top 30% | 6,109 | 92.8% | 1.30× |
| Top 40% | 8,146 | 91.4% | 1.28× |
| Top 50% | 10,183 | 89.7% | 1.25× |

### Campaign Capacity

The ranking provides different targeting options depending on campaign capacity.

- Targeting the **top 10%** of customers produced **96.4% precision**.
- Targeting the **top 30%** produced **92.8% precision** and captured **38.9% of all customers who subsequently became inactive**.
- Expanding the campaign to the top 50% increased coverage while maintaining **89.7% precision**.

Lift compares the concentration of inactive customers in a targeted group with the overall inactive rate. A lift of **1.35×** means that the top 10% ranked group contained approximately 35% more inactive customers per targeted customer than random targeting.

These results show how model predictions can be converted into a flexible customer-prioritization strategy rather than relying on a single classification threshold.

## Business Insights

The analysis suggests several practical insights for customer retention:

### 1. Recency is a key risk signal

Recency was the most informative feature according to permutation importance. Customers who have gone longer without purchasing are more likely to be classified as future inactive.

This suggests that monitoring changes in customer recency can help identify customers who may require attention.

### 2. Customer activity is highly concentrated

Only **5.1% of customers placed 21 or more orders**, but these customers accounted for approximately **45.6% of total historical revenue**.

This highlights the importance of considering customer value alongside predicted inactivity risk.

### 3. Campaign capacity can be used flexibly

The model ranking allows a business to adjust its target population according to available campaign capacity.

For example:

- A smaller campaign can focus on the top 10% of predicted-risk customers.
- A larger campaign can expand toward the top 20–30%.
- The corresponding precision and capture metrics can be used to understand the trade-off between targeting breadth and concentration of inactive customers.

### 4. Predictive risk is not the same as causal impact

The model identifies customers who are more likely to become inactive. It does not determine whether a particular retention campaign will cause those customers to return.

A controlled experiment would therefore be required to measure the incremental effect of a retention intervention.

### 5. Retention decisions should combine risk and customer value

Predicted inactivity risk can identify *who may become inactive*, while historical purchasing behavior can help identify *which customers may be particularly valuable to retain*.

A practical retention system could therefore combine model risk scores with customer-value measures when allocating limited campaign resources.

## Tech Stack

- **Python** — core programming language
- **Pandas** — data cleaning, transformation, aggregation, and feature engineering
- **NumPy** — numerical operations
- **Matplotlib** — data visualization
- **Scikit-learn** — machine learning, model evaluation, and feature importance
- **Jupyter Notebook** — analysis and experimentation
- **Git & GitHub** — version control and project management

## How to Run

### 1. Clone the repository

```cmd
git clone https://github.com/farich70/retention-lab.git
cd retention-lab
```

### 2. Install the required dependencies

```cmd
pip install -r requirements.txt
```

### 3. Download the dataset

Download the **UCI Online Retail II** dataset and place the Excel file in the location expected by the notebook.

The raw dataset is not included in the repository.

### 4. Run the notebook

Start Jupyter Notebook:

```cmd
jupyter notebook
```

Then open `RetentionLab.ipynb` and run the cells sequentially.

The notebook contains the complete workflow:

- Data preparation
- Exploratory data analysis
- Point-in-time feature engineering
- Churn target construction
- Model training and evaluation
- Model interpretation
- Customer prioritization

## Project Structure
```
retention-lab/
├── data/
│   └── raw/
│       └── online_retail_II.xlsx
├── RetentionLab.ipynb
├── README.md
├── requirements.txt
└── .gitignore
```

- `data/raw/` — raw dataset used for the analysis
- `RetentionLab.ipynb` — complete analysis and modeling workflow
- `README.md` — project documentation and key results
- `requirements.txt` — Python dependencies
- `.gitignore` — files and directories excluded from version control

## Conclusion

RetentionLab demonstrates an end-to-end customer retention analytics workflow using transactional e-commerce data.

The project combines exploratory analysis, point-in-time feature engineering, predictive modeling, and customer prioritization to identify customers who are likely to make no purchase during the following 60 days.

The final Random Forest achieved a **ROC-AUC of 0.815** on the held-out test period. The model also showed strong concentration of future inactive customers among the highest-risk ranked customers, with **96.4% precision in the top 10%** and **38.9% capture in the top 30%**.

The project demonstrates how customer transaction data can be transformed into actionable retention insights while maintaining a clear distinction between predictive risk and causal business impact.