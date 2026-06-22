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
WITH purchase_baskets AS (
    SELECT
        client_id,
        session_id,
        event_time,
        SUM(price) AS basket_revenue,
        AVG(price) AS basket_avg_price,
        COUNT(DISTINCT product_id) AS basket_products
    FROM events
    WHERE event_type = 'purchase'
      AND product_category = 'wine'
      AND client_id IS NOT NULL
      AND session_id IS NOT NULL
      AND event_time IS NOT NULL
      AND price IS NOT NULL
      AND product_id IS NOT NULL
    GROUP BY client_id, session_id, event_time
),

purchase_order AS (
    SELECT
        client_id,
        session_id,
        event_time,
        basket_revenue,
        basket_avg_price,
        basket_products,
        ROW_NUMBER() OVER (
            PARTITION BY client_id
            ORDER BY event_time ASC
        ) AS purchase_number
    FROM purchase_baskets
),

first_vs_later AS (
    SELECT
        client_id,

        MAX(CASE WHEN purchase_number = 1 THEN basket_revenue END) AS first_basket_revenue,
        MAX(CASE WHEN purchase_number = 1 THEN basket_avg_price END) AS first_basket_avg_price,

        COUNT(CASE WHEN purchase_number > 1 THEN 1 END) AS later_purchases,
        SUM(CASE WHEN purchase_number > 1 THEN basket_revenue ELSE 0 END) AS later_revenue,
        AVG(CASE WHEN purchase_number > 1 THEN basket_avg_price END) AS later_avg_price,
        SUM(CASE WHEN purchase_number > 1 THEN basket_products ELSE 0 END) AS later_products_count,

        AVG(CASE WHEN purchase_number > 1 THEN basket_avg_price END)
            / NULLIF(MAX(CASE WHEN purchase_number = 1 THEN basket_avg_price END), 0)
            AS avg_price_growth_ratio
    FROM purchase_order
    GROUP BY client_id
)

SELECT
    client_id,
    ROUND(first_basket_revenue, 2) AS first_basket_revenue,
    ROUND(first_basket_avg_price, 2) AS first_basket_avg_price,
    later_purchases,
    ROUND(later_revenue, 2) AS later_revenue,
    ROUND(later_avg_price, 2) AS later_avg_price,
    later_products_count,
    ROUND(avg_price_growth_ratio, 4) AS avg_price_growth_ratio
FROM first_vs_later
WHERE later_purchases >= 2
  AND later_revenue >= 300
  AND avg_price_growth_ratio >= 1.3
ORDER BY avg_price_growth_ratio DESC, later_revenue DESC;
```


## Result

| customer_id  | first_purchase_revenue | first_avg_price | later_purchases | later_revenue | later_avg_price | later_distinct_products | avg_price_growth_ratio |
| ------------ | ---------------------: | --------------: | --------------: | ------------: | --------------: | ----------------------: | ---------------------: |
| client_04552 |                  10.00 |           10.00 |              13 |      7,097.40 |          545.95 |                       9 |                54.5954 |
| client_03059 |                  15.00 |           15.00 |              23 |     11,277.00 |          490.30 |                      23 |                32.6870 |
| client_02345 |                  76.00 |           19.00 |              23 |     12,404.60 |          539.33 |                      22 |                28.3858 |
| client_01526 |                  22.00 |           22.00 |              34 |     19,043.00 |          560.09 |                      34 |                25.4586 |
| client_03638 |                  18.00 |           18.00 |              14 |      5,995.10 |          428.22 |                      13 |                23.7901 |
| client_03768 |                  70.00 |           23.33 |              16 |      7,850.00 |          490.63 |                      16 |                21.0268 |
| client_04449 |                  14.00 |           14.00 |              16 |      3,548.00 |          221.75 |                      16 |                15.8393 |
| client_00066 |                  55.00 |           13.75 |              17 |      3,565.00 |          209.71 |                      17 |                15.2513 |
| client_00989 |                  22.00 |            7.33 |               6 |        634.10 |          105.68 |                       6 |                14.4114 |
| client_02338 |                  27.00 |           13.50 |               6 |      1,129.00 |          188.17 |                       6 |                13.9383 |

## Visualization

![First vs Later Average Purchase Price](images/First_vs_Later_Average_Purchase_Price.png)

## Local Conclusion

The analysis shows that several customers started with low or moderate first purchases but later generated much higher purchase value. For example, the strongest cases had later average purchase prices from 13.9x to 54.6x higher than their first purchase average price.

This suggests that first purchase value alone is a weak indicator of long-term customer potential. Some customers who initially look low-value can later become high-value repeat buyers with significantly higher revenue and broader product activity.

However, the highest growth ratios should be interpreted carefully, because very small first purchases can inflate the ratio. For this reason, later revenue, repeat purchase count, and distinct product count should be considered together with the growth ratio.

---

# Analysis 2: High-Value Revenue Threshold Crossing

## Business Question

At what point does a customer become high-value?

This analysis tracks cumulative customer revenue and identifies the purchase where a customer crosses a selected revenue threshold.

The goal is to understand how many purchases it takes for customers to become valuable and whether high-value behavior appears early or only after repeated purchases.

## SQL Logic

The query calculates basket revenue, purchase order number, cumulative revenue, previous cumulative revenue, and the number of days from the first purchase.

```sql
WITH purchase_baskets AS (
    SELECT
        client_id,
        session_id,
        event_time,
        SUM(price) AS basket_revenue
    FROM events
    WHERE event_type = 'purchase'
      AND product_category = 'wine'
      AND client_id IS NOT NULL
      AND session_id IS NOT NULL
      AND event_time IS NOT NULL
      AND price IS NOT NULL
    GROUP BY client_id, session_id, event_time
),

purchase_sequence AS (
    SELECT
        client_id,
        session_id,
        event_time,
        basket_revenue,
        ROW_NUMBER() OVER (
            PARTITION BY client_id
            ORDER BY event_time ASC
        ) AS purchase_number,
        SUM(basket_revenue) OVER (
            PARTITION BY client_id
            ORDER BY event_time ASC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS cumulative_revenue,
        MIN(event_time) OVER (
            PARTITION BY client_id
        ) AS first_purchase_time
    FROM purchase_baskets
),

threshold_crossing AS (
    SELECT
        client_id,
        session_id,
        event_time,
        purchase_number,
        basket_revenue,
        cumulative_revenue,
        cumulative_revenue - basket_revenue AS previous_cumulative_revenue,
        DATE_DIFF('day', first_purchase_time, event_time) AS days_from_first_purchase
    FROM purchase_sequence
)

SELECT
    client_id,
    session_id,
    event_time,
    purchase_number,
    ROUND(basket_revenue, 2) AS basket_revenue,
    ROUND(cumulative_revenue, 2) AS cumulative_revenue,
    ROUND(previous_cumulative_revenue, 2) AS previous_cumulative_revenue,
    days_from_first_purchase
FROM threshold_crossing
WHERE cumulative_revenue >= 500
  AND previous_cumulative_revenue < 500
  AND purchase_number >= 2
ORDER BY cumulative_revenue DESC;
```

## Result

| customer_id  | purchase_number | basket_revenue | cumulative_revenue | previous_cumulative_revenue | days_from_first_purchase |
| ------------ | --------------: | -------------: | -----------------: | --------------------------: | -----------------------: |
| client_01526 |               3 |      18,518.00 |          18,573.00 |                       55.00 |                        0 |
| client_02345 |               5 |      11,615.00 |          11,847.00 |                      232.00 |                      239 |
| client_03059 |               2 |      10,747.00 |          10,762.00 |                       15.00 |                        0 |
| client_02038 |               4 |       9,920.00 |          10,087.00 |                      167.00 |                        0 |
| client_03342 |               5 |       6,580.00 |           6,930.00 |                      350.00 |                      577 |
| client_03638 |               4 |       5,790.00 |           5,841.00 |                       51.00 |                        0 |
| client_01760 |               3 |       4,435.00 |           4,466.00 |                       31.00 |                      882 |
| client_04568 |               2 |       3,900.00 |           4,215.00 |                      315.00 |                      117 |
| client_04449 |               6 |       3,149.00 |           3,361.00 |                      212.00 |                      208 |
| client_03401 |               2 |       3,190.00 |           3,215.00 |                       25.00 |                        0 |
| client_00066 |               3 |       3,120.00 |           3,202.00 |                       82.00 |                       31 |
| client_00481 |               2 |       2,850.00 |           2,899.00 |                       49.00 |                      331 |
| client_00556 |               5 |       2,767.00 |           2,879.00 |                      112.00 |                        0 |
| client_04552 |               4 |       2,770.00 |           2,801.80 |                       31.80 |                       89 |
| client_03768 |               3 |       2,600.00 |           2,705.00 |                      105.00 |                        0 |
| client_01942 |               2 |       2,670.00 |           2,703.00 |                       33.00 |                      390 |
| client_02340 |               2 |       2,315.00 |           2,685.00 |                      370.00 |                      516 |
| client_02656 |               2 |       2,575.00 |           2,613.00 |                       38.00 |                        6 |
| client_01880 |               2 |       2,420.00 |           2,585.00 |                      165.00 |                        0 |
| client_00598 |              10 |       2,060.00 |           2,528.00 |                      468.00 |                      514 |

## Visualization

![High Value Threshold Crossing](images/high_value_threshold_crossing.png)

## Local Conclusion

The chart shows that most customers who crossed the €500 cumulative revenue threshold did so relatively early, especially on the 2nd and 3rd purchase. The number of new threshold crossings gradually declines as purchase number increases, although a long tail of later crossings still exists.

This suggests that early repeat purchases are an important signal for identifying customers with high-value potential. Customers who approach or cross the threshold within the first few purchases may be strong candidates for CRM follow-up, loyalty actions, or premium product recommendations.

The result table adds another layer: in many top cases, the threshold was crossed through a single large basket after relatively low previous cumulative revenue. Therefore, the business should distinguish between customers who become high-value gradually and customers who cross the threshold suddenly due to one premium purchase.


---

# Analysis 3: Spend Uplift Detection

## Business Question

Which customers suddenly increased their basket value compared to their own previous behavior?

This analysis detects spend uplift by comparing each basket against the customer’s previous basket history.

Instead of comparing customers to the global average, the query compares each customer against their own historical baseline.

## SQL Logic

The query uses window functions to calculate previous basket revenue, rolling average basket revenue, rolling maximum basket revenue, and spend uplift ratio.

```sql
WITH purchase_baskets AS (
    SELECT
        client_id,
        session_id,
        event_time AS purchase_time,
        SUM(price) AS basket_revenue,
        COUNT(*) AS basket_items
    FROM events
    WHERE event_type = 'purchase'
      AND product_category = 'wine'
      AND client_id IS NOT NULL
      AND session_id IS NOT NULL
      AND event_time IS NOT NULL
      AND price IS NOT NULL
    GROUP BY client_id, session_id, event_time
),

purchase_history AS (
    SELECT
        client_id,
        session_id,
        purchase_time,
        basket_revenue,
        basket_items,
        ROW_NUMBER() OVER (
            PARTITION BY client_id
            ORDER BY purchase_time ASC
        ) AS purchase_number
    FROM purchase_baskets
),

previous_behavior AS (
    SELECT
        client_id,
        session_id,
        purchase_time,
        purchase_number,
        basket_revenue,
        basket_items,

        LAG(purchase_time) OVER (
            PARTITION BY client_id
            ORDER BY purchase_number ASC
        ) AS previous_purchase_time,

        LAG(basket_revenue) OVER (
            PARTITION BY client_id
            ORDER BY purchase_number ASC
        ) AS previous_basket_revenue,

        AVG(basket_revenue) OVER (
            PARTITION BY client_id
            ORDER BY purchase_number ASC
            ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING
        ) AS avg_previous_3_basket_revenue,

        MAX(basket_revenue) OVER (
            PARTITION BY client_id
            ORDER BY purchase_number ASC
            ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING
        ) AS max_previous_3_basket_revenue,

        purchase_number - 1 AS previous_baskets_count
    FROM purchase_history
),

spend_uplift AS (
    SELECT
        client_id,
        purchase_time,
        purchase_number,
        basket_revenue,
        basket_items,
        DATE_DIFF('day', previous_purchase_time, purchase_time) AS days_since_last_purchase,
        previous_basket_revenue,
        avg_previous_3_basket_revenue,
        max_previous_3_basket_revenue,
        previous_baskets_count,

        basket_revenue * 1.0
            / NULLIF(avg_previous_3_basket_revenue, 0) AS spend_uplift_ratio,

        CASE
            WHEN basket_revenue > max_previous_3_basket_revenue THEN 1
            ELSE 0
        END AS is_new_high_value_basket
    FROM previous_behavior
)

SELECT
    client_id,
    purchase_time,
    purchase_number,
    ROUND(basket_revenue, 2) AS basket_revenue,
    basket_items,
    days_since_last_purchase,
    ROUND(previous_basket_revenue, 2) AS previous_basket_revenue,
    ROUND(avg_previous_3_basket_revenue, 2) AS avg_previous_3_basket_revenue,
    ROUND(max_previous_3_basket_revenue, 2) AS max_previous_3_basket_revenue,
    previous_baskets_count,
    ROUND(spend_uplift_ratio, 4) AS spend_uplift_ratio,
    is_new_high_value_basket
FROM spend_uplift
WHERE previous_baskets_count >= 3
  AND basket_revenue >= 200
  AND spend_uplift_ratio >= 3
  AND days_since_last_purchase <= 60
ORDER BY spend_uplift_ratio DESC, basket_revenue DESC;
```

## Result

| customer_id  | purchase_number | basket_revenue | basket_items | avg_previous_3_basket_revenue | max_previous_3_basket_revenue | stake_jump_ratio | days_since_last_purchase |
| ------------ | --------------: | -------------: | -----------: | ----------------------------: | ----------------------------: | ---------------: | -----------------------: |
| client_04636 |             146 |       7,045.00 |            7 |                         12.00 |                         15.00 |           587.08 |                        1 |
| client_01785 |              16 |      14,800.00 |            3 |                         30.33 |                         73.00 |           487.91 |                       20 |
| client_03638 |               4 |       5,790.00 |            5 |                         17.00 |                         18.00 |           340.59 |                        0 |
| client_04552 |               4 |       2,770.00 |            3 |                         10.60 |                         12.00 |           261.32 |                       28 |
| client_03768 |              15 |       5,000.00 |            1 |                         21.33 |                         33.00 |           234.38 |                        0 |
| client_02345 |               5 |      11,615.00 |            2 |                         52.00 |                         67.00 |           223.37 |                       28 |
| client_02038 |               4 |       9,920.00 |            3 |                         55.67 |                         85.00 |           178.20 |                        0 |
| client_00556 |               5 |       2,767.00 |            5 |                         15.67 |                         20.00 |           176.62 |                        0 |
| client_01685 |              23 |       2,410.00 |            2 |                         13.67 |                         15.00 |           176.34 |                        8 |
| client_04441 |              81 |       2,939.00 |            8 |                         17.00 |                         29.00 |           172.88 |                       55 |

## Visualization

![Spend Uplift Detection](images/spend_uplift_detection.png)

## Local Conclusion

The analysis identifies customers whose current basket value sharply exceeded their own recent purchase baseline. In the strongest cases, basket revenue was more than 170x higher than the average of the previous 3 baskets, and every selected case also created a new high-value basket compared to the customer’s recent maximum.

This suggests that sudden premium-intent behavior can appear even after a long history of low-value baskets. These customers may be useful for CRM follow-up, premium product recommendations, or manual review of high-value purchasing patterns.

However, the highest uplift ratios should be interpreted carefully. Many ratios are extremely large because the previous 3 baskets were very small, not only because the current basket was large. For this reason, `basket_revenue`, `avg_previous_3_basket_revenue`, `max_previous_3_basket_revenue`, and `basket_items` should be reviewed together instead of relying only on `stake_jump_ratio`.

---

# Optional Analysis 4: Spend Rhythm Break by Purchase Blocks

## Business Question

Can customer spend growth be detected more reliably by grouping purchases into blocks?

Single baskets can be noisy. This optional analysis groups customer purchases into blocks of three and compares each block against previous blocks.

This gives a more stable view of customer value progression.

## SQL Logic

```sql
WITH purchase_baskets AS (
    SELECT
        client_id,
        event_time AS purchase_time,
        SUM(price) AS basket_revenue,
        COUNT(*) AS basket_items
    FROM events
    WHERE event_type = 'purchase'
      AND product_category = 'wine'
      AND client_id IS NOT NULL
      AND event_time IS NOT NULL
      AND product_id IS NOT NULL
      AND price IS NOT NULL
    GROUP BY client_id, event_time
),

purchase_blocks AS (
    SELECT
        client_id,
        purchase_time,
        basket_revenue,
        basket_items,
        FLOOR(
            (ROW_NUMBER() OVER (
                PARTITION BY client_id
                ORDER BY purchase_time ASC
            ) - 1) / 3
        ) AS purchase_block
    FROM purchase_baskets
),

block_stats AS (
    SELECT
        client_id,
        purchase_block,
        COUNT(*) AS block_purchases,
        MIN(purchase_time) AS block_start,
        MAX(purchase_time) AS block_end,
        SUM(basket_items) AS block_items,
        SUM(basket_revenue) AS block_revenue,
        MIN(basket_revenue) AS min_basket_revenue_in_block,
        MAX(basket_revenue) AS max_basket_revenue_in_block,
        AVG(basket_revenue) AS avg_basket_revenue_in_block
    FROM purchase_blocks
    GROUP BY client_id, purchase_block
),

block_comparison AS (
    SELECT
        client_id,
        purchase_block,
        block_purchases,
        block_start,
        block_end,
        block_items,
        block_revenue,
        min_basket_revenue_in_block,
        max_basket_revenue_in_block,
        avg_basket_revenue_in_block,

        DATE_DIFF(
            'day',
            LAG(block_end) OVER previous_block,
            block_start
        ) AS days_since_last_block,

        LAG(block_revenue) OVER previous_block AS previous_block_revenue,

        AVG(block_revenue) OVER previous_2_blocks AS avg_previous_2_blocks_revenue,
        MAX(block_revenue) OVER previous_2_blocks AS max_previous_2_blocks_revenue,

        block_revenue * 1.0
            / NULLIF(AVG(block_revenue) OVER previous_2_blocks, 0)
            AS block_jump_ratio,

        CASE
            WHEN block_revenue > MAX(block_revenue) OVER previous_2_blocks
            THEN 1
            ELSE 0
        END AS is_new_high_value_block

    FROM block_stats

    WINDOW
        previous_block AS (
            PARTITION BY client_id
            ORDER BY purchase_block ASC
        ),

        previous_2_blocks AS (
            PARTITION BY client_id
            ORDER BY purchase_block ASC
            ROWS BETWEEN 2 PRECEDING AND 1 PRECEDING
        )
)

SELECT
    client_id,
    purchase_block,
    block_purchases,
    block_start,
    block_end,
    block_items,
    ROUND(block_revenue, 2) AS block_revenue,
    ROUND(avg_previous_2_blocks_revenue, 2) AS avg_previous_2_blocks_revenue,
    ROUND(max_previous_2_blocks_revenue, 2) AS max_previous_2_blocks_revenue,
    ROUND(block_jump_ratio, 4) AS block_jump_ratio,
    days_since_last_block,
    is_new_high_value_block
FROM block_comparison
WHERE purchase_block >= 2
  AND block_purchases = 3
  AND avg_previous_2_blocks_revenue IS NOT NULL
  AND block_revenue >= 300
  AND block_jump_ratio >= 2
ORDER BY block_jump_ratio DESC, block_revenue DESC
LIMIT 10;
```

## Result

| customer_id  | block | block_purchases | block_items | block_revenue | avg_previous_2_blocks_revenue | max_previous_2_blocks_revenue | block_jump_ratio | days_since_last_block | is_block_high |
| ------------ | ----: | --------------: | ----------: | ------------: | ----------------------------: | ----------------------------: | ---------------: | --------------------: | ------------: |
| client_01785 |     5 |               3 |           5 |     14,828.00 |                         91.00 |                         91.00 |           162.95 |                    20 |             1 |
| client_03768 |     4 |               3 |           3 |      5,049.00 |                         57.50 |                         72.00 |            87.81 |                     0 |             1 |
| client_04636 |    48 |               3 |           9 |      7,066.00 |                        126.25 |                        130.00 |            55.97 |                     0 |             1 |
| client_04441 |    26 |               3 |          11 |      2,979.00 |                         82.50 |                        111.00 |            36.11 |                     0 |             1 |
| client_01878 |    11 |               3 |          12 |      2,339.00 |                         69.00 |                         69.00 |            33.90 |                    31 |             1 |
| client_03958 |     3 |               3 |           6 |     12,072.00 |                        377.00 |                        393.00 |            32.02 |                     0 |             1 |
| client_03131 |     5 |               3 |           5 |      1,589.00 |                         53.00 |                         58.00 |            29.98 |                     0 |             1 |
| client_03645 |    10 |               3 |           5 |      3,087.00 |                        126.25 |                        151.50 |            24.45 |                     0 |             1 |
| client_01743 |     2 |               3 |          33 |      2,125.50 |                         91.00 |                        117.00 |            23.36 |                     0 |             1 |
| client_03942 |     2 |               3 |          26 |     11,313.00 |                        517.00 |                        761.00 |            21.88 |                     0 |             1 |

## Visualization

![Spend Rhythm by Blocks](images/spend_rhythm_by_blocks.png)


## Local Conclusion

This analysis groups customer purchase history into blocks of 3 purchases and compares each block against the previous 2 blocks. The selected cases show a clear break from the customer’s previous spending rhythm: every listed block is at least 21x higher than the recent 2-block average and also exceeds the previous high-value block level.

The strongest case is `client_01785`, whose current 3-purchase block reached €14,828 compared with a previous 2-block average of only €91. Similar patterns appear for several other customers, where block revenue jumps from a low historical baseline to several thousand euros.

This suggests that some customers do not grow gradually. Instead, they can remain low-value for multiple purchase blocks and then suddenly generate a much higher-value block. These cases are useful for identifying changes in customer value, premium-intent behavior, or moments where CRM follow-up may be justified.

However, the result should be interpreted carefully. In many cases, the block-level jump is driven by one very large basket inside the 3-purchase block, rather than three consistently high purchases. Therefore, this analysis is best understood as a broader spend-rhythm break detector, not as proof of stable long-term customer upgrade.

---

## Overall Conclusion

This project analyzed customer spend progression using SQL window functions, customer-level purchase history, cumulative revenue tracking, and rolling behavioral baselines.

The analysis shows that customer value is not always visible from the first purchase. Some customers start with low or moderate basket values but later generate much higher revenue. Several high-value customers crossed the €500 cumulative revenue threshold within the first few repeat purchases, while others showed sudden spend uplift after a longer low-value history.

The strongest pattern across the analyses is that customer value often changes non-linearly. Customers may remain low-value for multiple purchases or purchase blocks, then suddenly generate a high-value basket or a high-value 3-purchase block. This makes simple first-purchase segmentation insufficient for detecting future high-value potential.

At the same time, the results should be interpreted carefully. Some extreme uplift ratios are caused by very small previous baselines, and several block-level jumps are driven by one large basket rather than a stable long-term increase. For this reason, revenue, purchase frequency, basket size, and rolling baselines should be reviewed together.

---

## Business Implications

Early repeat purchases are important signals for identifying customers with high-value potential. Customers who cross meaningful revenue thresholds within their first few purchases may be suitable for CRM follow-up, loyalty actions, or premium product recommendations.

Sudden uplift events should be monitored separately from gradual customer growth. A customer who suddenly moves from small baskets to a large premium basket may require a different retention or recommendation strategy than a customer whose spend increases steadily over time.

Customer segmentation should not rely only on first purchase value. A more useful approach is to combine cumulative revenue, purchase number, recent basket behavior, and rolling spend baselines to detect both early high-value customers and late-emerging premium-intent customers.


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
---
