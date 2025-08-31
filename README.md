# Case-Study
# A. Customer Nodes Exploration
### 1. How many unique nodes are there on the Data Bank system?
```sql
    SELECT COUNT(DISTINCT node_id) unique_nodes 
    FROM customer_nodes	
```
### 2. What is the number of nodes per region? & 3. How many customers are allocated to each region?
    SELECT r.region_name
        , COUNT(DISTiNCT c.node_id) num_of_nodes
        , COUNT(DISTINCT c.customer_id) total_customers
    FROM customer_nodes c
    LEFT JOIN regions r
    ON c.region_id = r.region_id
    GROUP BY r.region_name 
    ORDER BY r.region_name ASC

## 4. How many days on average are customers reallocated to a different node?
### check outlier is 9999-12-31
        SELECT Distinct end_date FROM customer_nodes
        ORDER BY end_date DESC

### average number of day
        SELECT ROUND(AVG(DATEDIFF(day, start_date, end_date)), 0) AS avg_num_of_day
        FROM customer_nodes
        WHERE end_date <> '9999-12-31'

## 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region? 

    WITH cte AS(
        SELECT 
            c.region_id
            , r.region_name
            , DATEDIFF(day, c.start_date, c.end_date) reallocation_days
        FROM customer_nodes c
        LEFT JOIN regions r
            ON r.region_id = c.region_id          
        WHERE c.end_date <> '9999-12-31'          
    ),
    stats AS(
        SELECT
            region_name
            , PERCENTILE_CONT(0.5)  WITHIN GROUP (ORDER BY reallocation_days)
                OVER (PARTITION BY region_name) median
            , PERCENTILE_CONT(0.80) WITHIN GROUP (ORDER BY reallocation_days)
                OVER (PARTITION BY region_name) percentile_80
            , PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY reallocation_days)
                OVER (PARTITION BY region_name) percentile_95
        FROM cte
    )
    SELECT DISTINCT
        region_name, median, percentile_80, percentile_95
    FROM stats
    ORDER BY region_name


# B. Customer Transactions
## 1. What is the unique count and total amount for each transaction type?
    SELECT txn_type
        , COUNT(txn_type) AS unique_count
        , SUM(txn_amount) AS total_amount
    FROM customer_transactions 
    GROUP BY txn_type

## 2. What is the average total historical deposit counts and amounts for all customers?
    WITH cte AS (
        SELECT 
            customer_id
            , txn_type
            , COUNT(*) deposit_count
            , SUM(txn_amount) deposit_amount
        FROM customer_transactions
        WHERE txn_type = 'deposit'
        GROUP BY customer_id, txn_type
    )
    SELECT 
        'deposit' txn_type
        , ROUND(AVG(CAST(deposit_count AS float)), 0) count
        , ROUND(AVG(CAST(deposit_amount AS float)), 0) amount
    FROM cte

## 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
    WITH cus_count AS (
        SELECT 
            customer_id
            , YEAR(txn_date) [Year]
            , MONTH(txn_date) [Month]
            , DATENAME(MONTH, txn_date) month_name  
            , SUM(CASE WHEN txn_type = 'deposit'    THEN 1 ELSE 0 END) deposit_count
            , SUM(CASE WHEN txn_type = 'purchase'   THEN 1 ELSE 0 END) purchase_count
            , SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) withdrawal_count
        FROM customer_transactions
        GROUP BY 
            customer_id
            , YEAR(txn_date)
            , MONTH(txn_date)
            , DATENAME(MONTH, txn_date)
    )
    SELECT 
        [Year] 
        , [Month]
        , month_name
        , COUNT(DISTINCT customer_id) customer_count
    FROM cus_count
    WHERE deposit_count > 1 
        AND (purchase_count = 1 
        OR withdrawal_count = 1)
    GROUP BY [Year],[Month], month_name
    ORDER BY [year], [Month] ASC

## 4. What is the closing balance for each customer at the end of the month?
    WITH customer_monthly_balance AS (
        SELECT 
            customer_id
            , YEAR(txn_date) [Year]
            , MONTH(txn_date) [Month]
            , DATENAME(MONTH, txn_date) AS month_name   
            , SUM(CASE WHEN txn_type = 'deposit' 
                       THEN txn_amount 
                       ELSE -txn_amount END) AS net_amount
        FROM customer_transactions
        GROUP BY customer_id
            , YEAR(txn_date)
            , MONTH(txn_date)
            , DATENAME(MONTH, txn_date)
    )
    SELECT 
        customer_id
        , [year]
        , month_name
        , net_amount
        , SUM(net_amount) OVER (
            PARTITION BY customer_id 
            ORDER BY [Month]  
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
          ) AS closing_balance
    FROM customer_monthly_balance

## 5. What is the percentage of customers who increase their closing balance by more than 5%?
    WITH cus_monthly AS (
        SELECT
            customer_id
            , MONTH(txn_date) [Month]
            , DATENAME(MONTH, txn_date) month_name
            , SUM(CASE WHEN txn_type = 'deposit'
                       THEN txn_amount
                       ELSE -txn_amount END) net_amount
        FROM customer_transactions
        GROUP BY customer_id
            , MONTH(txn_date)
            , DATENAME(MONTH, txn_date)
    )
    , cus_balances AS (
        SELECT
            customer_id
            , [Month]
            , month_name
            , net_amount
            , SUM(net_amount) OVER (
                PARTITION BY customer_id
                ORDER BY [Month]
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
              ) closing_balance
        FROM cus_monthly
    )
    , first_last AS (
        SELECT DISTINCT
            customer_id
            , FIRST_VALUE(closing_balance) OVER (
                PARTITION BY customer_id
                ORDER BY [Month]
                ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
              ) first_balance
            , LAST_VALUE(closing_balance)  OVER (
                PARTITION BY customer_id
                ORDER BY [Month]
                ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
              ) last_balance
        FROM cus_balances
    )
    SELECT
        CAST(
            100.0 * COUNT(DISTINCT CASE
                WHEN NULLIF(first_balance, 0) IS NOT NULL
                 AND (last_balance - first_balance) / NULLIF(first_balance, 0) > 0.05
                THEN customer_id END)
            / NULLIF(COUNT(DISTINCT customer_id), 0)
        AS decimal(5,2)
        ) [percentage]
    FROM first_last

# C. Data Allocation Challenge
## Opt 1. running customer balance column that includes the impact each transaction
    SELECT customer_id
        , txn_date
        , txn_type
        , txn_amount
        , SUM(CASE WHEN txn_type = 'deposit' tHEN txn_amount ELSE -txn_amount END) 
            OVER (PARTITION BY customer_id 
                  ORDER BY txn_date 
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
              ) running_balance
    FROM customer_transactions

=> Summary: phù hợp xem chi tiết sao kê và nắm bắt nhanh được số dư cuối tháng. 
Tuy nhiên có thể không phản ánh chính xác nhu cầu thực tế: 
    + nếu khách hàng có số dư cao giữa tháng nhưng rút gần hết trước cuối tháng → ngân hàng sẽ dự trù thiếu
    + Rủi ro không đủ dung lượng để đáp ứng khi khách tăng sử dụng trong tháng 


## Opt 2. customer balance at the end of each month
    WITH cus_monthly AS (
        SELECT customer_id
            , MONTH(txn_date) [Month]
            , DATENAME(MONTH, txn_date) month_name
            , SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) net_amount
        FROM customer_transactions
        GROUP BY
            customer_id
            , MONTH(txn_date)
            , DATENAME(MONTH, txn_date)
    )
    SELECT customer_id
        , [Month]
        , month_name
        , net_amount
        , SUM(net_amount) OVER (
              PARTITION BY customer_id
              ORDER BY [Month]
              ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
          ) closing_balance
    FROM cus_monthly
    ORDER BY customer_id
        , [Month]

=> Summary: có thế xem được xu hướng tiêu dùng của khách hàng trong ngắn hạn, giảm rủi ro thiếu dự trù hơn Option 1


## Opt 3. minimum, average and maximum values of the running balance for each customer
    WITH cus_running_balance AS (
        SELECT customer_id
            , txn_date
            , txn_type
            , txn_amount
            , SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) 
                OVER (PARTITION BY customer_id 
                      ORDER BY txn_date 
                      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
                  ) running_balance
        FROM customer_transactions
    )
    SELECT
        customer_id
        , MIN(running_balance) min_running_balance
        , AVG(running_balance) avg_running_balance
        , MAX(running_balance) max_running_balance
    FROM cus_running_balance
    GROUP BY 
        customer_id

=> Summary: Cung cấp cái nhìn tổng quan nhất: biết mức thấp nhất, trung bình và cao nhất của số dư.
Giúp Data Bank chuẩn bị dung lượng đủ cho tình huống xấu nhất (max balance) → an toàn cho hệ thống.
Tuy nhiên có thể dẫn tới dự trù dư thừa (overallocation), gây lãng phí tài nguyên nếu lấy max làm chuẩn.


# D. Extra Challenge
Data Bank wants to try another option which is a bit more difficult to implement - they want to calculate data growth using an interest calculation, just like in a traditional savings account you might have with a bank.
If the annual interest rate is set at 6% and the Data Bank team wants to reward its customers by increasing their data allocation based off the interest calculated on a daily basis at the end of each day, how much data would be required for this option on a monthly basis?

### không tính lãi kép
    WITH RunningBalance AS (
        SELECT
            customer_id
          , txn_date
          , CASE 
                WHEN txn_type = 'deposit' THEN txn_amount
                WHEN txn_type IN ('withdrawal', 'purchase') THEN -txn_amount
                ELSE 0
            END AS amount_change
        FROM customer_transactions
    )
    , BalanceCalc AS (
        SELECT
            customer_id
          , txn_date
          , SUM(amount_change) OVER (
                PARTITION BY customer_id 
                ORDER BY txn_date 
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
            ) AS running_balance
        FROM RunningBalance
    )
    , DailyInterest AS (
        SELECT
            customer_id
          , txn_date
          , running_balance
          , running_balance * 0.06 / 365.0 AS daily_interest ## simple interest
        FROM BalanceCalc
    )
    , MonthlyInterest AS (
        SELECT
            customer_id
          , DATENAME(MONTH, txn_date) AS month_name
          , SUM(daily_interest) AS total_interest
        FROM DailyInterest
        GROUP BY
            customer_id
          , DATENAME(MONTH, txn_date)
    )
    SELECT
        customer_id
      , month_name
      , total_interest
      , SUM(total_interest) OVER (PARTITION BY month_name) AS total_system_interest
    FROM MonthlyInterest
    ORDER BY customer_id
        , month_name
      


