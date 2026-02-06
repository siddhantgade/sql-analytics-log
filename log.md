# SQL Analytics Log

Daily compressed SQL problem-solving log.

Append-only. One commit per day.

----------------------------------------------------------------------------------------------------
## 2026-01-20

Q) SQL Third Transaction

Link: https://datalemur.com/questions/sql-third-transaction

Keywords: ranking, partitioning
Constraints: ordering must be by transaction\_date, row numbering must reset per user
Decision: Select the third row per user after ranking transactions

WITH cte\_transactions AS (

SELECT

user\_id,

spend,

transaction\_date,

ROW\_NUMBER() OVER (

PARTITION BY user\_id

ORDER BY transaction\_date ASC

) AS rn

FROM transactions

)

SELECT

user\_id,

spend,

transaction\_date

FROM cte\_transactions

WHERE rn = 3;

-----------------------------------------------------------------------------------------------------
## 21 Jan 2026

Q) Second Highest Salary
Link: https://datalemur.com/questions/sql-second-highest-salary

Keywords: value ranking, elimination logic
Constraints: duplicate salaries allowed, single-value output required
Decision: Identify the highest remaining salary after excluding the maximum.

SELECT 
    MAX(salary) AS second_highest_salary
FROM employee
WHERE salary < (
    SELECT MAX(salary)
    FROM employee
);

-- not using MAX()

SELECT DISTINCT salary
FROM employee
ORDER BY salary DESC
LIMIT 1 OFFSET 1;

OFFSET 1
→ skips the highest salary

LIMIT 1
→ returns exactly one value
-----------------------------------------------------------------------------------------------------
## 22 Jan 2026

Q) time spent snaps
Link: https://datalemur.com/questions/time-spent-snaps

Keywords: age bucket aggregation, activity time distribution
Constraints: conditional aggregation must be row-level using CASE, aggregated totals must be computed before percentage calculation
Decision: Calculate send and open time percentages per age bucket after aggregating user activity time.

WITH cte_answer AS (
  SELECT 
    a.age_bucket,
    SUM(
      CASE 
        WHEN ac.activity_type = 'send' THEN ac.time_spent 
        ELSE 0 
      END
    ) AS time_spent_sending,
    SUM(
      CASE 
        WHEN ac.activity_type = 'open' THEN ac.time_spent
        ELSE 0
      END
    ) AS time_spent_opening
  FROM activities ac 
  INNER JOIN age_breakdown a
    ON ac.user_id = a.user_id
  GROUP BY a.age_bucket
) 
SELECT 
  age_bucket,
  ROUND(
    (time_spent_sending * 100.0) /
    (time_spent_sending + time_spent_opening),
    2
  ) AS send_perc,
  ROUND(
    (time_spent_opening * 100.0) /
    (time_spent_sending + time_spent_opening),
    2
  ) AS open_perc
FROM cte_answer;
-----------------------------------------------------------------------------------------------------
## 24 Jan 2026

Q) Highest Grossing Products
Link: https://datalemur.com/questions/sql-highest-grossing

Keywords: aggregation, ranking
Constraints: per-category ranking, year-based filtering
Decision: Identify the top two highest-spending products within each category for a fixed time window.

WITH cte_highest_grossing AS (
  SELECT
    category,
    product,
    SUM(spend) AS total_spend,
    ROW_NUMBER() OVER (
      PARTITION BY category
      ORDER BY SUM(spend) DESC
    ) AS rn
  FROM product_spend
  WHERE transaction_date >= '2022/01/01'
    AND transaction_date <= '2022/12/31'
  GROUP BY category, product
)
SELECT
  category,
  product,
  total_spend
FROM cte_highest_grossing
WHERE rn < 3;
-----------------------------------------------------------------------------------------------------
## 25 Jan 2026

Q) Tweets Rolling Averages
Link: https://datalemur.com/questions/rolling-average-tweets

Keywords: rolling aggregation, time series
Constraints: ordered window frame, per-user isolation
Decision: Compute a 3-day rolling average of tweet activity for each user over time.

SELECT
  user_id,
  tweet_date,
  ROUND(
    AVG(tweet_count) OVER (
      PARTITION BY user_id
      ORDER BY tweet_date
      ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ),
    2
  ) AS tweet_count
FROM tweets;
-----------------------------------------------------------------------------------------------------
## 28 Jan 2026

Q) Signup Confirmation Rate

Link: https://datalemur.com/questions/signup-confirmation-rate

Keywords: user-level aggregation, activation rate
Constraints: users may have multiple signup attempts; activation defined by at least one confirmation
Decision: Compute activation rate by collapsing multiple attempts into a single outcome per user.

WITH user_activation AS (
  SELECT
    e.user_id,
    MAX(
      CASE 
        WHEN t.signup_action = 'Confirmed' THEN 1
        ELSE 0
      END
    ) AS activated
  FROM emails e
  LEFT JOIN texts t
    ON e.email_id = t.email_id
  WHERE e.signup_date IS NOT NULL
  GROUP BY e.user_id
)
SELECT
  ROUND(AVG(activated), 2) AS activation_rate
FROM user_activation;
-----------------------------------------------------------------------------------------------------
## 29 Jan 2026

Q) Spotify Streaming History

Link: https://datalemur.com/questions/spotify-streaming-history

Keywords: historical aggregation, event accumulation
Constraints: history data has no timestamps, weekly data must be filtered up to August 4
Decision: Combine pre-aggregated historical plays with counted recent plays to compute total plays per user and song.

SELECT
  user_id,
  song_id,
  SUM(total_plays) AS song_count
FROM (
  SELECT
    user_id,
    song_id,
    song_plays AS total_plays
  FROM songs_history

  UNION ALL

  SELECT
    user_id,
    song_id,
    COUNT(*) AS total_plays
  FROM songs_weekly
  WHERE listen_time < '2022-08-05'
  GROUP BY user_id, song_id
) combined
GROUP BY user_id, song_id
ORDER BY song_count DESC;
-----------------------------------------------------------------------------------------------------
## 2 Feb 2026

Q) Supercloud Customer
Link: https://datalemur.com/questions/supercloud-customer

Keywords: group-level filtering, distinct count
Constraints: must evaluate per customer after grouping, must compare against total category count
Decision: Identify customers who have purchased from every available product category.

SELECT
  cc.customer_id
FROM
  customer_contracts cc
JOIN
  products p
  ON cc.product_id = p.product_id
GROUP BY
  cc.customer_id
HAVING
  COUNT(DISTINCT p.product_category) =
  (SELECT COUNT(DISTINCT product_category) FROM products);
-----------------------------------------------------------------------------------------------------
## 4 Feb 2026

Q) Odd vs Even Measurements

Link: https://datalemur.com/questions/odd-even-measurements

Keywords: sequence-based classification, per-day isolation
Constraints: full-date partitioning required, sequence must be based on time order
Decision: Assign an ordered position to each reading per day and aggregate values by odd and even positions.

WITH daily_measurements AS (
    SELECT
        DATE(measurement_time) AS measurement_date,
        measurement_value,
        ROW_NUMBER() OVER (
            PARTITION BY DATE(measurement_time)
            ORDER BY measurement_time ASC
        ) AS reading_sequence
    FROM measurements
)
SELECT
    measurement_date,
    SUM(
        CASE 
            WHEN reading_sequence % 2 = 1 THEN measurement_value
            ELSE 0
        END
    ) AS odd_sum,
    SUM(
        CASE 
            WHEN reading_sequence % 2 = 0 THEN measurement_value
            ELSE 0
        END
    ) AS even_sum
FROM daily_measurements
GROUP BY measurement_date;
-----------------------------------------------------------------------------------------------------
## 5 Feb 2026

Q) Swapped Food Delivery
Link: https://datalemur.com/questions/sql-swapped-food-delivery

Keywords: row pairing, source position mapping
Constraints: independent row sequence must be created, last unpaired row must remain unchanged
Decision: compute a source row for each position first, then fetch the item from that source

WITH numbered AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY id) AS rn,
        item
    FROM orders
),
helper AS (
    SELECT
        rn,
        CASE
            WHEN rn % 2 = 1
                 AND rn < (SELECT MAX(rn) FROM numbered)
            THEN rn + 1
            WHEN rn % 2 = 0
            THEN rn - 1
            ELSE rn
        END AS source_rn
    FROM numbered
)
SELECT
    h.rn,
    n.item
FROM helper h
JOIN numbered n
    ON h.source_rn = n.rn
ORDER BY h.rn;
-----------------------------------------------------------------------------------------------------

