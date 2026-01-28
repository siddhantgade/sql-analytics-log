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
