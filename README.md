# Customer Spend Progression Analysis

## Project Overview

This project analyzes anonymized real-world e-commerce purchase data to identify customers who become more valuable over time.

The goal is to detect customers whose purchasing behavior changes after the first order: higher basket value, repeated purchases, faster revenue accumulation, or sudden spend uplift.

This type of analysis can support CRM targeting, loyalty campaigns, premium product recommendations, and customer value segmentation.

---

## Business Problem

Not all valuable customers start with a high-value first purchase.

A customer may begin with a small or moderate basket, but later become important for the business through repeated purchases, higher average basket value, or stronger product diversity.

The business question is:

> Which customers show signs of becoming high-value customers over time, and how can we detect this behavior from purchase history?

---

## Dataset

The analysis is based on anonymized e-commerce purchase data.

The original dataset included customer-level purchase events, product attributes, timestamps, prices, sessions, and product identifiers.

---

## Tools Used

* SQL / DuckDB
* Python / pandas
* Tableau

---

## Analytical Approach

The analysis was performed at customer and basket level.

Main techniques used:

* Customer-level aggregation
* Basket-level revenue calculation
* First purchase vs later purchase comparison
* Cumulative revenue tracking
* Revenue threshold detection
* Window functions
* Rolling historical baselines
* Spend uplift detection
* Business interpretation of customer behavior

---

## Key Findings



---

# Analysis 1: First Purchase vs Later Behavior

## Business Question

Do some customers become more valuable after their first purchase?

This analysis compares the first purchase with later customer behavior. It checks whether customers increased their average basket value, total revenue, product diversity, and number of repeat purchases after the first purchase.

## SQL Logic

The query separates each customer’s first purchase from later purchases, then compares early and later behavior.

```sql

```

Full query: [`sql/01_first_vs_later_behavior.sql`](sql/01_first_vs_later_behavior.sql)

## Result


| customer_id  | first_purchase_revenue | first_avg_price | later_purchases | later_revenue | later_avg_price | later_distinct_products | avg_price_growth_ratio |
| ------------ | ---------------------: | --------------: | --------------: | ------------: | --------------: | ----------------------: | ---------------------: |
| customer_001 |                [value] |         [value] |         [value] |       [value] |         [value] |                 [value] |                [value] |

## Visualization

![First Purchase vs Later Revenue](images/first_vs_later_revenue.png)

## Local Conclusion




---

# Analysis 2: High-Value Revenue Threshold Crossing

## Business Question

At what point does a customer become high-value?

This analysis tracks cumulative customer revenue and identifies the purchase where a customer crosses a selected revenue threshold.

The goal is to understand how many purchases it takes for customers to become valuable and whether high-value behavior appears early or only after repeated purchases.

## SQL Logic

The query calculates basket revenue, purchase order number, cumulative revenue, previous cumulative revenue, and the number of days from the first purchase.

```sql
[INSERT SHORT SQL SNIPPET HERE]

-- Recommended snippet:
-- ROW_NUMBER() over customer purchase history
-- SUM(revenue) OVER cumulative revenue
-- previous_cumulative_revenue
-- threshold filter
```

Full query: [`sql/02_high_value_threshold_crossing.sql`](sql/02_high_value_threshold_crossing.sql)

## Result

[INSERT RESULT TABLE HERE]

Recommended columns:

| customer_id  | purchase_number | basket_revenue | cumulative_revenue | previous_cumulative_revenue | days_from_first_purchase |
| ------------ | --------------: | -------------: | -----------------: | --------------------------: | -----------------------: |
| customer_001 |         [value] |        [value] |            [value] |                     [value] |                  [value] |

## Visualization

![High Value Threshold Crossing](images/high_value_threshold_crossing.png)

## Local Conclusion

[INSERT LOCAL CONCLUSION HERE]

Example:

The threshold analysis identifies customers who became high-value after multiple purchases rather than immediately. This can help the business separate one-time high spenders from customers who gradually build value through repeat behavior.

---

# Analysis 3: Spend Uplift Detection

## Business Question

Which customers suddenly increased their basket value compared to their own previous behavior?

This analysis detects spend uplift by comparing each basket against the customer’s previous basket history.

Instead of comparing customers to the global average, the query compares each customer against their own historical baseline.

## SQL Logic

The query uses window functions to calculate previous basket revenue, rolling average basket revenue, rolling maximum basket revenue, and spend uplift ratio.

```sql
[INSERT SHORT SQL SNIPPET HERE]

-- Recommended snippet:
-- LAG(previous purchase time)
-- LAG(previous basket revenue)
-- AVG(basket_revenue) over previous 3 baskets
-- MAX(basket_revenue) over previous 3 baskets
-- stake_jump_ratio
```

Full query: [`sql/03_spend_uplift_detection.sql`](sql/03_spend_uplift_detection.sql)

## Result

[INSERT RESULT TABLE HERE]

Recommended columns:

| customer_id  | purchase_number | basket_revenue | avg_previous_3_basket_revenue | max_previous_3_basket_revenue | stake_jump_ratio | days_since_last_purchase |
| ------------ | --------------: | -------------: | ----------------------------: | ----------------------------: | ---------------: | -----------------------: |
| customer_001 |         [value] |        [value] |                       [value] |                       [value] |          [value] |                  [value] |

## Visualization

![Spend Uplift Detection](images/spend_uplift_detection.png)

## Local Conclusion

[INSERT LOCAL CONCLUSION HERE]

Example:

The spend uplift logic highlights customers whose current basket is significantly larger than their own recent baseline. These customers may be strong candidates for premium offers, loyalty actions, or personalized recommendations.

---

# Optional Analysis 4: Spend Rhythm Break by Purchase Blocks

## Business Question

Can customer spend growth be detected more reliably by grouping purchases into blocks?

Single baskets can be noisy. This optional analysis groups customer purchases into blocks of three and compares each block against previous blocks.

This gives a more stable view of customer value progression.

## SQL Logic

```sql
[INSERT SHORT SQL SNIPPET HERE]

-- Recommended snippet:
-- FLOOR((ROW_NUMBER() - 1) / 3) AS block
-- block-level revenue
-- previous block revenue
-- rolling average of previous blocks
-- block_jump_ratio
```

Full query: [`sql/04_spend_rhythm_by_blocks.sql`](sql/04_spend_rhythm_by_blocks.sql)

## Result

[INSERT RESULT TABLE HERE]

Recommended columns:

| customer_id  |   block | block_purchases | block_revenue | avg_previous_2_blocks_revenue | block_jump_ratio | days_since_last_block |
| ------------ | ------: | --------------: | ------------: | ----------------------------: | ---------------: | --------------------: |
| customer_001 | [value] |         [value] |       [value] |                       [value] |          [value] |               [value] |

## Visualization

![Spend Rhythm by Blocks](images/spend_rhythm_by_blocks.png)

## Local Conclusion

[INSERT LOCAL CONCLUSION HERE]

Example:

Block-level analysis reduces the noise of individual baskets and shows broader shifts in customer behavior. This is useful when the business wants to detect sustained value growth rather than isolated high-value purchases.

---

# Overall Business Conclusion

[INSERT FINAL CONCLUSION HERE]

Example:

The analysis shows that customer value progression cannot be understood only from the first purchase. Some customers start with moderate baskets but later become high-value through repeat purchases, higher basket value, or sudden spend uplift.

From a business perspective, these customers are important because they may be missed by simple first-order segmentation. Tracking cumulative revenue, purchase number, spend uplift, and basket progression can help the business identify customers suitable for CRM campaigns, loyalty incentives, premium recommendations, and retention actions.

---

# Business Recommendations

[INSERT 3–5 RECOMMENDATIONS HERE]

Example:

* Track customers whose later basket value grows significantly compared to their first purchase.
* Create a CRM segment for customers approaching the high-value revenue threshold.
* Monitor sudden spend uplift as a signal for premium product interest.
* Use customer-level spend progression as an input for recommendation and loyalty campaigns.
* Avoid judging long-term customer value only by the first purchase.

---

# Limitations

* The dataset contains anonymized historical purchase data.
* Raw commercial data is not included in this repository.
* The analysis is based on observed purchase behavior, not full customer intent.
* The project does not include marketing exposure, acquisition channel, or customer demographics.
* Revenue thresholds are analytical assumptions and should be adjusted based on real business context.
* The analysis identifies behavioral patterns but does not prove causality.

---

# Project Structure

```text
customer-spend-progression-analysis/
│
├── README.md
├── sql/
│   ├── 01_first_vs_later_behavior.sql
│   ├── 02_high_value_threshold_crossing.sql
│   ├── 03_spend_uplift_detection.sql
│   └── 04_spend_rhythm_by_blocks.sql
│
├── images/
│   ├── first_vs_later_revenue.png
│   ├── high_value_threshold_crossing.png
│   ├── spend_uplift_detection.png
│   └── spend_rhythm_by_blocks.png
│
├── notebooks/
│   └── analysis_validation.ipynb
│
└── data_sample/
    └── anonymized_sample.csv
```

---

# Skills Demonstrated

* SQL data analysis
* Customer-level segmentation
* Basket-level revenue analysis
* Window functions
* Cumulative revenue tracking
* Rolling baseline calculation
* Business KPI interpretation
* Data anonymization awareness
* Analytical storytelling
* Portfolio-ready documentation

---

# Summary

This project demonstrates how SQL can be used to identify customer value progression in real-world e-commerce purchase data.

The analysis focuses on customers who become more valuable over time, cross high-value revenue thresholds, or show sudden spend uplift compared to their own previous behavior.

The main business value is the ability to support CRM, retention, loyalty, and recommendation strategies with customer-level behavioral analysis.
