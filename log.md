```sql
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
## 8 Feb 2026

Q) Bloomberg Stock Min Max (Monthly Open)
Link: https://datalemur.com/questions/sql-bloomberg-stock-min-max-1

Keywords: window ranking, monthly aggregation
Constraints: month must be defined before comparison; row-level selection required to preserve month-value linkage
Decision: Rank month-level rows per ticker to select highest and lowest monthly opens without losing date context.

WITH cte_stock AS (
  SELECT
    ticker,
    TO_CHAR(date, 'Mon-YYYY') AS month_year,
    open,
    ROW_NUMBER() OVER (
      PARTITION BY ticker
      ORDER BY open DESC
    ) AS rn_high,
    ROW_NUMBER() OVER (
      PARTITION BY ticker
      ORDER BY open ASC
    ) AS rn_low
  FROM stock_prices
)
SELECT
  ticker,
  MAX(CASE WHEN rn_high = 1 THEN month_year END) AS highest_month,
  MAX(CASE WHEN rn_high = 1 THEN open END)       AS highest_open,
  MAX(CASE WHEN rn_low = 1 THEN month_year END)  AS lowest_month,
  MAX(CASE WHEN rn_low = 1 THEN open END)        AS lowest_open
FROM cte_stock
GROUP BY ticker
ORDER BY ticker;
-----------------------------------------------------------------------------------------------------
## 10 Feb 2026

Q) Amazon Shopping Spree
Link: https://datalemur.com/questions/amazon-shopping-spree

Keywords: streak detection, temporal sequencing
Constraints: consecutive days must be derived from date differences not row order, streak grouping must reset only on non-consecutive days
Decision: identify users who have at least one sequence of three calendar days with purchases

WITH cte_transactions AS (
    SELECT
        user_id,
        transaction_date,
        CASE
            WHEN transaction_date =
                 LAG(transaction_date) OVER (
                     PARTITION BY user_id
                     ORDER BY transaction_date
                 ) + INTERVAL '1 day'
            THEN 1
            ELSE 0
        END AS is_consecutive
    FROM transactions
),
cte_streaks AS (
    SELECT
        user_id,
        transaction_date,
        is_consecutive,
        SUM(CASE WHEN is_consecutive = 0 THEN 1 ELSE 0 END)
            OVER (PARTITION BY user_id ORDER BY transaction_date) AS streak_id
    FROM cte_transactions
),
cte_counts AS (
    SELECT
        user_id,
        streak_id,
        COUNT(*) FILTER (WHERE is_consecutive = 1) AS consecutive_count
    FROM cte_streaks
    GROUP BY user_id, streak_id
)
SELECT DISTINCT user_id
FROM cte_counts
WHERE consecutive_count >= 2
ORDER BY user_id;
-----------------------------------------------------------------------------------------------------
## 12 Feb 2026

Q) Best Selling Product Per Category
Link: https://datalemur.com/questions/best-selling-products

Keywords: Window Ranking, Top-Per-Group
Constraints: Window function result cannot be filtered in same SELECT layer, Subquery requires alias and creates derived table scope
Decision: Select the highest-selling product per category using rating as tie-breaker

SELECT 
  rp.product_name,
  rp.category_name
FROM (
  SELECT 
    p.product_name,
    p.category_name,
    ROW_NUMBER() OVER (
      PARTITION BY p.category_name
      ORDER BY ps.sales_quantity DESC, ps.rating DESC
    ) AS category_rank
  FROM products p
  INNER JOIN product_sales ps
    ON p.product_id = ps.product_id
) rp
WHERE rp.category_rank = 1
ORDER BY rp.category_name;
-----------------------------------------------------------------------------------------------------
## 13 Feb 2026

Q) Most Recent Purchase Count per User
Link: https://datalemur.com/questions/histogram-users-purchases

Keywords: layered aggregation, dataset isolation
Constraints: aggregate result must not be filtered in the same query layer, join must match on both user and derived max date
Decision: isolate each user’s latest transaction date before counting products on that date

WITH latest_transaction_per_user AS (
    SELECT
        user_id,
        MAX(transaction_date) AS latest_transaction_date
    FROM
        user_transactions
    GROUP BY
        user_id
)

SELECT
    ut.transaction_date,
    ut.user_id,
    COUNT(ut.product_id) AS product_count
FROM
    user_transactions ut
JOIN
    latest_transaction_per_user ltu
    ON ut.user_id = ltu.user_id
   AND ut.transaction_date = ltu.latest_transaction_date
GROUP BY
    ut.transaction_date,
    ut.user_id
ORDER BY
    ut.transaction_date;
-----------------------------------------------------------------------------------------------------
## 14 Feb 2026

Q) Alibaba Compressed Mode
Link: https://datalemur.com/questions/alibaba-compressed-mode

Keywords: aggregated dataset, frequency filtering
Constraints: table is pre-aggregated so no re-counting allowed; ties must return all matching rows
Decision: select item_count where order_occurrences equals the maximum frequency
Business Context: Identifying the most common order size helps analyze purchasing behavior patterns.

SELECT 
  item_count AS mode
FROM 
  items_per_order
WHERE 
  order_occurrences = (
    SELECT 
      MAX(order_occurrences)
    FROM
      items_per_order
  );
-----------------------------------------------------------------------------------------------------
## 15 Feb 2026

Q) Card Launch Success
Link: https://datalemur.com/questions/card-launch-success

Keywords: first-occurrence-per-group, chronological ordering
Constraints: ordering must include issue_year and issue_month; exactly one earliest record per card must be selected
Decision: retrieve the earliest issuance record for each card
Business Context: Used to evaluate first-month launch performance of each card product.

WITH card_issue_ranked AS (
    SELECT
        card_name,
        issued_amount,
        ROW_NUMBER() OVER (
            PARTITION BY card_name
            ORDER BY issue_year, issue_month
        ) AS row_num
    FROM monthly_cards_issued
)

SELECT
    card_name,
    issued_amount
FROM 
    card_issue_ranked
WHERE 
    row_num = 1
ORDER BY 
    issued_amount DESC;
-----------------------------------------------------------------------------------------------------
## 16 Feb 2026

Q) International Call Percentage
Link: https://datalemur.com/questions/international-call-percentage

Keywords: Self-Join, Conditional Aggregation
Constraints: phone_info must be joined twice with distinct aliases; international calls must be counted without filtering out total rows
Decision: Compute percentage of calls where caller and receiver countries differ using a single aggregation pass
Business Context: Useful for analyzing cross-border telecom traffic patterns and revenue mix.

SELECT 
  ROUND(
    (COUNT(
        CASE 
          WHEN caller_info.country_id != receiver_info.country_id 
          THEN 1 
        END
    ) * 100.0) / COUNT(*), 
  1) AS international_calls_pct
FROM phone_calls pc
INNER JOIN phone_info caller_info 
  ON pc.caller_id = caller_info.caller_id
INNER JOIN phone_info receiver_info
  ON pc.receiver_id = receiver_info.caller_id;
-----------------------------------------------------------------------------------------------------
## 17 Feb 2026

Q) Top 3 Highest Salaries Per Department
Link: 

Keywords: Top-N per group, Partitioned ranking
Constraints: Ranking must reset per department; Filtering must occur after window function evaluation
Decision: Identify employees earning within the top three distinct salary levels in each department
Business Context: Used to identify top earners per department for compensation benchmarking or performance review analysis.

WITH cte_salaries AS (
    SELECT 
        d.department_name,
        e.name,
        e.salary,
        DENSE_RANK() OVER (
            PARTITION BY d.department_id
            ORDER BY e.salary DESC
        ) AS rn
    FROM employee e
    INNER JOIN department d
        ON e.department_id = d.department_id
)
SELECT 
    department_name,
    name,
    salary
FROM cte_salaries
WHERE rn <= 3
ORDER BY 
    department_name ASC,
    salary DESC,
    name ASC;
-----------------------------------------------------------------------------------------------------
## 20 Feb 2026

Q) Salary Difference Between Marketing and Engineering
Link: https://platform.stratascratch.com/coding/10308-salaries-differences?code_type=1

Keywords: Conditional aggregation, Scalar subquery
Constraints: Correct foreign key join between employee and department tables; Department names must match exact case in filter condition
Decision: Extract maximum salary per required department and compute absolute difference
Business Context: Useful for identifying internal compensation gaps between major departments.

WITH salary_differences AS (
    SELECT 
        (
            SELECT MAX(e.salary)
            FROM db_employee e
            INNER JOIN db_dept d
                ON e.department_id = d.id
            WHERE d.department = 'marketing'
        ) AS marketing_salary,
        (
            SELECT MAX(e.salary)
            FROM db_employee e
            INNER JOIN db_dept d
                ON e.department_id = d.id
            WHERE d.department = 'engineering'
        ) AS engineering_salary
)

SELECT 
    ABS(marketing_salary - engineering_salary)
FROM salary_differences;
-----------------------------------------------------------------------------------------------------
22 Feb 2026

Q) Finding Updated Records
Link: http://platform.stratascratch.com/coding/10299-finding-updated-records?code_type=1
Keywords: latest record selection, grouped ranking
Constraints: ordering must be applied in outer query to guarantee final sort, ranking must reset per employee id
Decision: select the most recent salary record for each employee
Business Context: Useful for retrieving the current compensation details of employees from historical salary records.

WITH employee_latest_salary AS (
    SELECT 
        id,
        first_name,
        last_name,
        department_id,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY id
            ORDER BY salary DESC
        ) AS rn
    FROM ms_employee_salary
)

SELECT 
    id,
    first_name,
    last_name,
    department_id,
    salary
FROM 
    employee_latest_salary
WHERE 
    rn = 1
ORDER BY 
    id;
-----------------------------------------------------------------------------------------------------
28 Feb 2026

Q0 Workers With The Highest Salaries
Link: https://platform.stratascratch.com/coding/10353-workers-with-the-highest-salaries?code_type=1

Keywords: scalar subquery, join alignment
Constraints: maximum salary must be computed using the same join conditions as the outer query, aggregate value cannot be directly used in WHERE without subquery
Decision: filter job titles where salary equals the maximum salary among workers with titles
Business Context: Identify the job titles associated with the highest paid officially designated employees for compensation analysis. 

SELECT
    t.worker_title
FROM
    worker w
INNER JOIN
    title t
    ON w.worker_id = t.worker_ref_id
WHERE
    w.salary = (
        SELECT
            MAX(w.salary)
        FROM
            worker w
        INNER JOIN
            title t
            ON w.worker_id = t.worker_ref_id
    );
-----------------------------------------------------------------------------------------------------
01 Mar 2026

Q) Bikes Last Used
Link: https://platform.stratascratch.com/coding/10176-bikes-last-used?code_type=1

Keywords: Aggregation, Entity-level summarization
Constraints: Non-aggregated columns must appear in GROUP BY; ORDER BY must reference aggregated result via alias
Decision: Compute the most recent return timestamp per bike and rank by recency
Business Context: Helps operations teams identify recently active bikes for fleet monitoring and redistribution.

SELECT
    bike_number,
    MAX(end_time) AS last_used
FROM
    dc_bikeshare_q1_2012
GROUP BY
    bike_number
ORDER BY
    last_used DESC;
-----------------------------------------------------------------------------------------------------
05 Mar 2026

Q) Customer Details
Link: https://platform.stratascratch.com/coding/9891-customer-details?code_type=1
Keywords: Left Join, Relationship Mapping
Constraints: Non-matching records must persist, Join key must maintain referential integrity
Decision: Include all registered customers regardless of whether they have placed an order.
Business Context: Identify customer purchasing patterns while maintaining a complete registry of the user base for marketing reach.

SELECT
    c.first_name,
    c.last_name,
    c.city,
    o.order_details
FROM
    customers AS c
LEFT JOIN
    orders AS o
    ON
        c.id = o.cust_id;
-----------------------------------------------------------------------------------------------------
05 Mar 2026

Q) Average Bedrooms & Bathrooms by Property Type
Link: https://platform.stratascratch.com/coding/9622-number-of-bathrooms-and-bedrooms?code_type=1

Keywords: aggregation, categorical grouping
Constraints: aggregation must occur per city-property_type pair, non-aggregated columns must appear in GROUP BY
Decision: compute average bedrooms and bathrooms for each city and property type combination
Business Context: Helps marketplace teams understand listing composition across cities and property categories for supply planning.

SELECT
    city,
    property_type,
    AVG(bathrooms) AS n_bathrooms_avg,
    AVG(bedrooms) AS n_bedrooms_avg
FROM airbnb_search_details
GROUP BY
    city,
    property_type;
-----------------------------------------------------------------------------------------------------
6 Mar 2026

Q) Department Salary vs Department Average
Link: https://platform.stratascratch.com/coding/9917-average-salaries/discussion?code_type=1

Keywords: Department-level benchmarking, Salary distribution
Constraints: Window aggregation must preserve row-level records, GROUP BY positional references must match selected columns
Decision: Compare each employee salary against the average salary within their department.
Business Context: HR teams may compare individual salaries against department averages to detect pay imbalances.

SELECT
    department,
    first_name,
    salary,
    AVG(salary) OVER (
        PARTITION BY department
    ) AS avg_salary
FROM
    employee
GROUP BY
    1,
    2,
    3;
-----------------------------------------------------------------------------------------------------
7 Mar 2026

Q) Lyft Driver Wages
Link: https://platform.stratascratch.com/coding/10003-lyft-driver-wages?code_type=1

Keywords: Range Filtering, Conditional Logic
Constraints: Salary thresholds must be evaluated in a single WHERE clause, OR condition must combine both boundary checks
Decision: Retrieve driver records whose salaries fall outside the middle salary band.
Business Context: A ride-sharing company may review drivers earning at the lowest and highest salary brackets for compensation analysis.

select 
    index,
    start_date,
    end_date,
    yearly_salary
from 
    lyft_drivers
WHERE 
    yearly_salary >= 30000 
OR
    yearly_salary >= 70000 
;
-----------------------------------------------------------------------------------------------------
9 march 2026

Q) Artist Appearance Count
Link: https://platform.stratascratch.com/coding/9992-find-artists-that-have-been-on-spotify-the-most-number-of-times?code_type=1

Keywords: aggregation, frequency counting
Constraints: artist must be included in GROUP BY when using COUNT; ordering must reference the derived occurrence count
Decision: count how many ranking entries exist per artist and sort by that frequency
Business Context: music analytics teams track how frequently artists appear in global charts to measure sustained popularity.

SELECT
    artist,
    COUNT(position) AS n_occurences
FROM
    spotify_worldwide_daily_song_ranking
GROUP BY
    artist
ORDER BY
    n_occurences DESC;
-----------------------------------------------------------------------------------------------------
9 march 2026

Q) Customer Orders for Jill and Eva
Link: https://platform.stratascratch.com/coding/9913-order-details?code_type=1

Keywords: relational filtering, customer-order mapping
Constraints: customer id must correctly map to order cust_id, name filter must match exact stored first_name values
Decision: retrieve all orders belonging only to customers named Jill or Eva.
Business Context: A sales team may need to quickly review the purchase history of specific VIP customers.

select 
    c.first_name,
    o.order_date,
    o.order_details,
    o.total_order_cost
from 
    customers c 
inner join
    orders o
on
    c.id = o.cust_id
where 
    c.first_name in ('Jill', 'Eva')
order by
    c.id asc
;
-----------------------------------------------------------------------------------------------------
10 march 2026

Q) Average Hack Popularity by Employee Location
Link: https://platform.stratascratch.com/coding/10061-popularity-of-hack?code_type=1

Keywords: relational join, group aggregation
Constraints: employee IDs must match across both tables, aggregation must occur after grouping by location
Decision: combine employee and survey tables to compute average hack popularity per location
Business Context: Helps internal teams compare innovation engagement levels across company office locations.

SELECT
    f.location,
    AVG(fh.popularity) AS avg_popularity
FROM
    facebook_employees f
INNER JOIN
    facebook_hack_survey fh
ON
    f.id = fh.employee_id
GROUP BY
    f.location;
-----------------------------------------------------------------------------------------------------
13 march 2026

Q) Top Ranked Songs

Link: https://platform.stratascratch.com/coding/9991-top-ranked-songs?code_type=1

Keywords: aggregation logic, row-level filtering
Constraints: ranking column must already represent final song position; filtering must occur before grouping
Decision: count occurrences of rows where songs reached position 1 per track.
Business Context: A music analytics team may track how often songs reach the #1 position globally to measure sustained popularity.

SELECT
    DISTINCT trackname,
    COUNT(id) AS times_top1
FROM
    spotify_worldwide_daily_song_ranking
WHERE
    position = 1
GROUP BY 
    trackname
ORDER BY
    times_top1 desc
-----------------------------------------------------------------------------------------------------
14 march 2026

Q) Admin Department Employee Count
Link: https://platform.stratascratch.com/coding/9845-find-the-number-of-employees-working-in-the-admin-department?code_type=1
Keywords: Case-insensitive filtering, Conditional aggregation
Constraints: Department values may vary in letter casing, Filtering must occur before aggregation
Decision: Count employees in the Admin department who joined from April onward.
Business Context: HR analytics teams often measure departmental hiring patterns within specific time windows to monitor seasonal recruitment activity.

SELECT 
    COUNT(worker_id) AS admin_employee_count
FROM worker
WHERE LOWER(department) LIKE 'admin'
AND EXTRACT(MONTH FROM joining_date) >= 4;
-----------------------------------------------------------------------------------------------------
15 march 2026

Q Annual Violations Count for Roxanne Cafe
Link: https://platform.stratascratch.com/coding/9728-inspections-that-resulted-in-violations?code_type=1

Keywords: temporal aggregation, dataset filtering
Constraints: grouping must match extracted year level, aggregation must operate after restaurant filter
Decision: aggregate violation records per inspection year for a specific restaurant.
Business Context: Public health departments track yearly violation trends for individual restaurants to monitor compliance patterns over time.

SELECT
    EXTRACT(YEAR FROM inspection_date) AS inspection_year,
    COUNT(DISTINCT violation_id) AS n_violations
FROM sf_restaurant_health_violations
WHERE business_name = 'Roxanne Cafe'
GROUP BY EXTRACT(YEAR FROM inspection_date)
ORDER BY inspection_year;
-----------------------------------------------------------------------------------------------------
16 march 2026

Q) Street Churros Inspection Risk Lookup
Link: https://platform.stratascratch.com/coding/9688-churro-activity-date?code_type=1

Keywords: Row Filtering, Conditional Threshold
Constraints: Exact facility name match is required; Only inspection records with score strictly below 95 must be returned
Decision: Retrieve inspection dates and risk descriptions specifically for the STREET CHURROS facility when the inspection score falls below the defined threshold
Business Context: A compliance team may review low-scoring inspections for a specific vendor to monitor potential health risk patterns.

SELECT
    activity_date,
    pe_description
FROM los_angeles_restaurant_health_inspections
WHERE
    facility_name = 'STREET CHURROS'
AND
    score < 95;
-----------------------------------------------------------------------------------------------------
21 March 2026

Q) Count MacBook Pro User Events
Link: https://platform.stratascratch.com/coding/9653-count-the-number-of-user-events-performed-by-macbookpro-users?code_type=1

Keywords: categorical filtering, grouped aggregation
Constraints: device value must exactly match filter string, grouping must align with selected non-aggregated column
Decision: count event occurrences specifically for filtered device type
Business Context: Helps analyze feature usage patterns among MacBook Pro users for product optimization

select 
    event_name,
    COUNT(*) AS event_count
from playbook_events
WHERE
    device = 'macbook pro'
GROUP BY
    event_name
ORDER BY
    event_count desc;
-----------------------------------------------------------------------------------------------------
22 March 2026

Q) New Products Net Change
Link: https://platform.stratascratch.com/coding/10318-new-products?code_type=1

Keywords: conditional aggregation, grouped comparison
Constraints: counts must be computed per company, year-based filtering must not break grouping
Decision: compute year-wise product counts within each company and subtract to get net change
Business Context: Helps track year-over-year product launch performance across companies.

SELECT
    company_name,
    COUNT(DISTINCT CASE WHEN year = 2020 THEN product_name END) -
    COUNT(DISTINCT CASE WHEN year = 2019 THEN product_name END) AS net_products
FROM car_launches
GROUP BY company_name;
-----------------------------------------------------------------------------------------------------
23 March 2026

Q) Highest Daily Customer Order Cost
Link: https://platform.stratascratch.com/coding/9915-highest-cost-orders?code_type=1

Keywords: aggregation, ranking
Constraints: daily total must be computed before ranking, ties must be preserved per date
Decision: identify top customers per day based on aggregated order cost
Business Context: Helps identify highest revenue-generating customers on a daily basis for targeted engagement.

WITH daily_customer_totals AS (
    SELECT 
        c.first_name,
        o.order_date,
        SUM(o.total_order_cost) AS total_costs,
        RANK() OVER (
            PARTITION BY o.order_date
            ORDER BY SUM(o.total_order_cost) DESC
        ) AS rn
    FROM customers c
    INNER JOIN orders o
        ON c.id = o.cust_id
    WHERE o.order_date >= '2019-02-01'
      AND o.order_date <= '2019-05-01'
    GROUP BY 
        c.first_name,
        o.order_date
)
SELECT
    first_name,
    order_date,
    total_costs
FROM daily_customer_totals
WHERE rn = 1;
-----------------------------------------------------------------------------------------------------
24 march 2026

Q) Risky Projects
Link: https://platform.stratascratch.com/coding/10304-risky-projects?code_type=1

Keywords: cost allocation, temporal adjustment
Constraints: inclusive project duration must be applied, avoid integer truncation in division
Decision: compute prorated employee cost per project and filter where it exceeds budget
Business Context: Identify projects where actual labor cost over the project duration exceeds allocated budget to flag financial risk.

select
    lp.title,
    lp.budget,
    ROUND(SUM(le.salary * ((end_date - start_date + 1) / 365.0)),0) AS prorated_employee_expense
from 
    linkedin_projects lp
INNER JOIN
    linkedin_emp_projects lep
ON 
    lp.id = lep.project_id
INNER JOIN
    linkedin_employees le
ON
    lep.emp_id = le.id
GROUP BY
    lp.title,
    lp.budget
HAVING
    SUM(le.salary * ((end_date - start_date + 1) / 365.0)) > lp.budget;
-----------------------------------------------------------------------------------------------------
25 March 2026

Q) Email Activity Rank
Link: https://platform.stratascratch.com/coding/10351-activity-rank?code_type=1

Keywords: aggregation, window ranking
Constraints: all CTEs must be declared within a single WITH clause, ranking must be globally applied without partitioning
Decision: compute total emails per user and assign a unique rank based on descending activity with alphabetical tie-breaker
Business Context: Identify the most active users on an email platform to prioritize engagement strategies.

WITH cte_google_gmail_emails AS
(
SELECT
    from_user,
    COUNT(*) AS total_emails
FROM
    google_gmail_emails
GROUP BY
    from_user
),
cte_ranked AS (
select
    from_user,
    total_emails,
    ROW_NUMBER() OVER(
    ORDER BY
        total_emails desc,
        from_user asc
    ) AS activity_rank
from 
    cte_google_gmail_emails
)
SELECT
    from_user,
    total_emails,
    activity_rank
FROM
    cte_ranked
;
-----------------------------------------------------------------------------------------------------
27 March 2026

Q) Finding Purchases Within 7 Days
Link: https://platform.stratascratch.com/coding/10553-finding-purchases?code_type=1

Keywords: temporal comparison, row sequencing
Constraints: correct ordering required for LAG to access immediate previous row, DISTINCT must be applied after filtering to avoid multiple rows per user
Decision: compare each transaction with its immediate previous one and return unique users
Business Context: Identify users with short purchase cycles to analyze retention behavior

WITH user_transactions AS (
    SELECT
        user_id,
        created_at,
        LAG(created_at) OVER (
            PARTITION BY user_id
            ORDER BY created_at
        ) AS previous_created_at
    FROM amazon_transactions
)
SELECT DISTINCT
    user_id
FROM user_transactions
WHERE
    created_at > previous_created_at
    AND created_at < previous_created_at + INTERVAL '7 days';
-----------------------------------------------------------------------------------------------------
04 April 2026

Q) Young Hosts Apartment Count by Nationality
Link: https://platform.stratascratch.com/coding/10156-number-of-units-per-nationality?code_type=1

Keywords: join aggregation, deduplication
Constraints: duplicate rows from join required DISTINCT on unit_id, filtering conditions had to be applied before grouping to avoid inflated counts
Decision: count unique apartment units per nationality for hosts under 30
Business Context: Helps identify which nationalities of younger hosts are most active in listing apartments on the platform.

SELECT 
    ah.nationality,
    COUNT(DISTINCT au.unit_id) AS apartment_count
FROM 
    airbnb_hosts ah
INNER JOIN
    airbnb_units au
ON
    ah.host_id = au.host_id
WHERE
    ah.age < 30
    AND au.unit_type = 'Apartment'
GROUP BY
    ah.nationality
ORDER BY
    apartment_count DESC;
-----------------------------------------------------------------------------------------------------
06 April 2026

Q) Matching Hosts and Guests by Gender and Nationality
Link: https://platform.stratascratch.com/coding/10078-find-matching-hosts-and-guests-in-a-way-that-they-are-both-of-the-same-gender-and-nationality?code_type=1

Keywords: controlled pairing, group alignment
Constraints: must avoid many-to-many join explosion, must assign row numbers before joining
Decision: create one-to-one matches by aligning ranked rows within each gender and nationality group
Business Context: Used in platforms to fairly match users based on shared attributes without duplication.

WITH ranked_hosts AS (
    SELECT
        host_id,
        gender,
        nationality,
        ROW_NUMBER() OVER (
            PARTITION BY gender, nationality
            ORDER BY host_id
        ) AS row_num
    FROM airbnb_hosts
),
ranked_guests AS (
    SELECT
        guest_id,
        gender,
        nationality,
        ROW_NUMBER() OVER (
            PARTITION BY gender, nationality
            ORDER BY guest_id
        ) AS row_num
    FROM airbnb_guests
)
SELECT
    h.host_id,
    g.guest_id
FROM ranked_hosts h
INNER JOIN ranked_guests g
    ON h.gender = g.gender
   AND h.nationality = g.nationality
   AND h.row_num = g.row_num;
-----------------------------------------------------------------------------------------------------
09 April 2026

Q) Acceptance Rate By Date
Link: https://platform.stratascratch.com/coding/10285-acceptance-rate-by-date?code_type=1

Keywords: event mapping, pair matching
Constraints: join must be on sender and receiver together, accepted events must not be filtered in WHERE with sent
Decision: compute acceptance rate by mapping sent requests to accepted pairs
Business Context: Helps measure how effectively users convert connection requests into accepted relationships on a platform

WITH sent_requests AS (
    SELECT 
        date,
        user_id_sender,
        user_id_receiver
    FROM
        fb_friend_requests
    WHERE
        action = 'sent'
),
accepted_requests AS (
    SELECT
        DISTINCT 
        user_id_sender,
        user_id_receiver
    FROM
        fb_friend_requests
    WHERE
        action = 'accepted'
)
SELECT
    s.date,
    (COUNT(a.user_id_sender) * 100.0 / COUNT(*)) AS percentage_acceptance
FROM
    sent_requests s
LEFT JOIN
    accepted_requests a
ON
    s.user_id_sender = a.user_id_sender
AND
    s.user_id_receiver = a.user_id_receiver
GROUP BY
    s.date;
-----------------------------------------------------------------------------------------------------
18 April 2026

Q) Finding Returning Users Within 7 Days
Link: https://platform.stratascratch.com/coding/10322-finding-user-purchases?code_type=1

Keywords: sequential comparison, retention window
Constraints: correct pairing of first and second purchase, valid date difference calculation between two rows
Decision: identify users whose second purchase occurs within 1–7 days after their first purchase
Business Context: Helps measure early user retention by tracking quick repeat purchases after initial engagement.

WITH user_purchase_sequence AS (
    SELECT
        user_id,
        created_at,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY created_at ASC
        ) AS rn,
        LEAD(created_at) OVER (
            PARTITION BY user_id
            ORDER BY created_at ASC
        ) AS next_date
    FROM
        amazon_transactions
)

SELECT 
    user_id
FROM user_purchase_sequence
WHERE
    rn = 1 
AND
    next_date IS NOT NULL
AND
    DATEDIFF(next_date, created_at) BETWEEN 1 AND 7;
-----------------------------------------------------------------------------------------------------
10 may 2026

Q – Ranking Most Active Guests

Link: https://platform.stratascratch.com/coding/10159-ranking-most-active-guests?code_type=1

Keywords: aggregation, ranking
Constraints: avoided grouping by message-level values, ranking sequence had to be applied after aggregation
Decision: Ranked guests based on total exchanged messages.
Business Context: A hospitality platform can identify highly engaged users for retention campaigns and customer experience analysis.

WITH guest_message_summary AS (
    SELECT
        id_guest,
        SUM(n_messages) AS sum_n_messages,
        DENSE_RANK() OVER (
            ORDER BY SUM(n_messages) DESC
        ) AS ranking
    FROM airbnb_contacts
    GROUP BY id_guest
)

SELECT
    ranking,
    id_guest,
    sum_n_messages
FROM guest_message_summary;
-----------------------------------------------------------------------------------------------------
18 May 2026

Q – Income By Title And Gender
Link: https://platform.stratascratch.com/coding/10077-income-by-title-and-gender?code_type=1

Keywords: Granularity Handling, Layered Aggregation
Constraints: Salary must not be duplicated across multiple bonus rows, Aggregation must occur at employee-level before averaging
Decision: Calculate one compensation value per employee before computing grouped averages
Business Context: Used in payroll and incentive reporting to calculate average compensation across organizational categories.

WITH employee_bonus_summary AS
(
    SELECT
        worker_ref_id,
        SUM(bonus) AS total_bonus
    FROM sf_bonus
    GROUP BY worker_ref_id
),

employee_compensation AS
(
    SELECT
        e.id,
        e.employee_title,
        e.sex,
        e.salary + ebs.total_bonus AS compensation
    FROM sf_employee e
    INNER JOIN employee_bonus_summary ebs
        ON e.id = ebs.worker_ref_id
)

SELECT
    employee_title,
    sex,
    AVG(compensation) AS avg_compensation
FROM employee_compensation
GROUP BY
    employee_title,
    sex;
-----------------------------------------------------------------------------------------------------
19 May 2026

Q – Highest Target Achieved Under Manager
Link: https://platform.stratascratch.com/coding/9905-highest-target-under-manager?code_type=1

Keywords: Window Ranking, Common Table Expression
Constraints: Manager filtering must happen before ranking, Tied highest targets must return all matching employees
Decision: Identify employees with the highest target under a specific manager.
Business Context: Used to identify top-performing sales employees reporting to a specific manager for performance tracking.

WITH ranked_employee_targets AS
(
    SELECT
        first_name,
        target,
        DENSE_RANK() OVER
        (
            ORDER BY target DESC
        ) AS target_rank
    FROM 
        salesforce_employees
    WHERE
        manager_id = 13
)

SELECT
    first_name,
    target
FROM
    ranked_employee_targets
WHERE
    target_rank = 1
;
-----------------------------------------------------------------------------------------------------
21 May 2026

Q) HR Department Employees With Duplicate Output
Link: https://platform.stratascratch.com/coding/9858-find-employees-in-the-hr-department-and-output-the-result-with-one-duplicate

Keywords: Set Operations, Duplicate Preservation
Constraints: Both SELECT statements must return identical column structure, Duplicate rows must not be removed
Decision: Return HR department employees twice in the final output
Business Context: This can be used to intentionally duplicate department-level employee records for audit testing or downstream validation scenarios.

SELECT
    first_name,
    department
FROM worker
WHERE department = 'HR'

UNION ALL

SELECT
    first_name,
    department
FROM worker
WHERE department = 'HR';
-----------------------------------------------------------------------------------------------------
31 May 2026

Q) Word Frequency in Draft Contents
Link: https://platform.stratascratch.com/coding/9817-find-the-number-of-times-each-word-appears-in-drafts?code_type=1

Keywords: Text Normalization, Frequency Analysis
Constraints: Punctuation must be removed before counting, Word comparison must be case-insensitive
Decision: Count occurrences of each cleaned word across all draft contents.
Business Context: Content teams can use word frequency analysis to identify commonly used terms in draft documents.

SELECT LOWER(word) AS word,
       COUNT(*) AS occurrences
FROM google_file_store t,
     regexp_split_to_table(
         regexp_replace(t.contents, '[[:punct:]]', '', 'g'),
         E'\\s+'
     ) AS word
GROUP BY LOWER(word)
ORDER BY occurrences DESC;
-----------------------------------------------------------------------------------------------------
3 June 2026

Q – Facebook Matching User Pairs
Link: https://platform.stratascratch.com/coding/10085-facebook-matching-users-pairs?code_type=1

Keywords: Self Join, Record Pairing
Constraints: Different employee IDs, Matching location and gender
Decision: Identify employee pairs that satisfy the specified matching criteria.
Issue Faced: Initially joined records on the same ID, which made the age and seniority conditions impossible to satisfy. Also overlooked that the query could return duplicate pairs such as (A,B) and (B,A).
Business Context: Can be used to identify employee pairs for mentorship, collaboration, or demographic-based workforce analysis.

SELECT
    fe1.id,
    fe2.id
FROM
    facebook_employees fe1
JOIN
    facebook_employees fe2
ON
    fe1.id != fe2.id
WHERE
    fe1.location = fe2.location
    AND fe1.age <> fe2.age
    AND fe1.gender = fe2.gender
    AND fe1.is_senior <> fe2.is_senior;
-----------------------------------------------------------------------------------------------------
8 June 2026

Q) Percentage of Shipable Orders
Link: https://platform.stratascratch.com/coding/10090-find-the-percentage-of-shipable-orders?code_type=1

Keywords: Conditional Aggregation, Percentage Calculation
Constraints: Denominator must include all orders, Decimal arithmetic must be preserved
Decision: Calculate the percentage of orders with a valid shipping address out of all orders.
Issue Faced: Initially filtered out non-shipable orders causing an incorrect denominator, then encountered integer division truncating decimal results.
Business Context: Measure the proportion of customer orders that can be fulfilled immediately based on available shipping information.

SELECT 
    COUNT(
        CASE
            WHEN c.address IS NOT NULL THEN 1
        END
    ) * 100.0 / COUNT(*) AS percent_shipable
FROM orders o
INNER JOIN customers c
    ON o.cust_id = c.id;
-----------------------------------------------------------------------------------------------------
10 June 2026

Q) Top Cool Votes
Link: https://platform.stratascratch.com/coding/10060-top-cool-votes?code_type=1

Keywords: Ranking, Prioritization
Constraints: Sort by cool in descending order, Return only the top 2 rows
Decision: Retrieve the two reviews with the highest cool votes
Issue Faced: Identifying the correct ordering direction before limiting the result set
Business Context: A review platform can use this query to surface the most appreciated reviews for users.

SELECT
    business_name,
    review_text
FROM
    yelp_reviews
ORDER BY
    cool DESC
LIMIT 2;
-----------------------------------------------------------------------------------------------------
11 June 2026

Q) Reviews by Business Category
Link: https://platform.stratascratch.com/coding/10049-reviews-of-categories?code_type=1

Keywords: Data Normalization, Aggregation
Constraints: Categories must be split using semicolon delimiters, Review counts must be aggregated after category expansion
Decision: Expand category values and calculate total reviews per category
Issue Faced: Did not know the UNNEST function for converting array elements into separate rows
Business Context: Helps identify which business categories receive the highest volume of customer reviews across the platform

select 
    UNNEST(STRING_TO_ARRAY(categories, ';')) AS category,
    SUM(review_count) AS total_reviews
from yelp_business
GROUP BY
    1 
order by 
    2
;
-----------------------------------------------------------------------------------------------------
12 June 2026

Q) Unified Wine Variety Catalog
Link: https://platform.stratascratch.com/coding/10025-find-all-possible-varieties-which-occur-in-either-of-the-winemag-datasets?code_type=1
Keywords: Dataset Consolidation, Duplicate Elimination

Constraints: Varieties must be returned only once, Results must include records from both datasets
Decision: Return all unique varieties present in either dataset.
Business Context: Create a master wine variety list by combining entries from multiple wine review sources.

select 
    variety
from winemag_p1
union 
select 
    variety
from winemag_p2
order by variety
;
-----------------------------------------------------------------------------------------------------
```