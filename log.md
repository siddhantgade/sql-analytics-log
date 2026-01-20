\# SQL Analytics Log



Daily compressed SQL problem-solving log.

Append-only. One commit per day.



----------------------------------------------------------------------------------------------------

\## 2026-01-20



Q – SQL Third Transaction

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



