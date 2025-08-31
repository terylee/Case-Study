# Case-Study
# A. Customer Nodes Exploration
### 1. How many unique nodes are there on the Data Bank system?
```sql
    SELECT COUNT(DISTINCT node_id) unique_nodes 
    FROM customer_nodes	
```
- Result:

  <img width="357" height="127" alt="image" src="https://github.com/user-attachments/assets/daaf9738-3d05-4f2a-9428-ae56ccb699fe" />

### 2. What is the number of nodes per region? & 3. How many customers are allocated to each region?
```sql    
    SELECT r.region_name
        , COUNT(DISTiNCT c.node_id) num_of_nodes
        , COUNT(DISTINCT c.customer_id) total_customers
    FROM customer_nodes c
    LEFT JOIN regions r
    ON c.region_id = r.region_id
    GROUP BY r.region_name 
    ORDER BY r.region_name ASC
```
- Result:

  <img width="868" height="316" alt="image" src="https://github.com/user-attachments/assets/46eb3bff-002c-4978-bf92-873cf22cafdf" />

### 4. How many days on average are customers reallocated to a different node?
#### 4.1 check outlier is 9999-12-31
```sql
        SELECT Distinct end_date FROM customer_nodes
        ORDER BY end_date DESC
```
- Result:

    <img width="287" height="242" alt="image" src="https://github.com/user-attachments/assets/1a9a5f00-df2d-4841-81de-bc05c4e29a92" />

#### 4.2 average number of day
```sql
        SELECT ROUND(AVG(DATEDIFF(day, start_date, end_date)), 0) AS avg_num_of_day
        FROM customer_nodes
        WHERE end_date <> '9999-12-31'
```
- Result:

    <img width="376" height="124" alt="image" src="https://github.com/user-attachments/assets/bfe88f0e-5f45-4d18-a5bf-27f59a294876" />

### 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region? 
```sql
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
```
- Result:

  <img width="1023" height="307" alt="image" src="https://github.com/user-attachments/assets/8ca2fed1-eddd-417c-b592-ab723d6e91b3" />

# B. Customer Transactions
### 1. What is the unique count and total amount for each transaction type?
```sql
    SELECT txn_type
        , COUNT(txn_type) AS unique_count
        , SUM(txn_amount) AS total_amount
    FROM customer_transactions 
    GROUP BY txn_type
```
- Result:

  <img width="779" height="213" alt="image" src="https://github.com/user-attachments/assets/499b3662-f0f9-493c-814e-1c10bb845b8c" />

### 2. What is the average total historical deposit counts and amounts for all customers?
```sql
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
```
- Result:

  <img width="612" height="116" alt="image" src="https://github.com/user-attachments/assets/866e39ef-3dc2-4fcd-9faa-f52089bc9c9c" />

### 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
```sql
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
```
- Result:

  <img width="881" height="253" alt="image" src="https://github.com/user-attachments/assets/9f131530-a483-4251-8930-0a86c9affc35" />

### 4. What is the closing balance for each customer at the end of the month?
```sql
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
```
- Result:

  <img width="1185" height="425" alt="image" src="https://github.com/user-attachments/assets/1ad04109-4103-43ee-8dbc-2af95d812882" />

### 5. What is the percentage of customers who increase their closing balance by more than 5%?
```sql
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
```
- Result:

    <img width="323" height="121" alt="image" src="https://github.com/user-attachments/assets/9b36e54a-ab2d-4f55-80ef-c99f3f4ed2b5" />

# C. Data Allocation Challenge
### Opt 1. running customer balance column that includes the impact each transaction
```sql
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
```
- Result:

  <img width="1227" height="544" alt="image" src="https://github.com/user-attachments/assets/3451e888-2031-4f84-bda3-b5d5c0b5da09" />

- Summary: phù hợp xem chi tiết sao kê và nắm bắt nhanh được số dư cuối tháng.
    Tuy nhiên có thể không phản ánh chính xác nhu cầu thực tế: 
    + nếu khách hàng có số dư cao giữa tháng nhưng rút gần hết trước cuối tháng → ngân hàng sẽ dự trù thiếu
    + Rủi ro không đủ dung lượng để đáp ứng khi khách tăng sử dụng trong tháng 


### Opt 2. customer balance at the end of each month
```sql
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
```
- Result:

  <img width="1206" height="543" alt="image" src="https://github.com/user-attachments/assets/6e6284cf-aa11-405a-9c08-0c1dca763ca2" />

- Summary: có thế xem được xu hướng tiêu dùng của khách hàng trong ngắn hạn, giảm rủi ro thiếu dự trù hơn Option 1 và là option tối ưu nhất


### Opt 3. minimum, average and maximum values of the running balance for each customer
```sql
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
```
- Result:

  <img width="1340" height="547" alt="image" src="https://github.com/user-attachments/assets/d91d19f1-a495-4956-97cb-320be1e819c0" />

- Summary: Cung cấp cái nhìn tổng quan nhất: biết mức thấp nhất, trung bình và cao nhất của số dư.
Giúp Data Bank chuẩn bị dung lượng đủ cho tình huống xấu nhất (max balance) → an toàn cho hệ thống.
Tuy nhiên có thể dẫn tới dự trù dư thừa (overallocation), gây lãng phí tài nguyên nếu lấy max làm chuẩn.


# D. Extra Challenge
Data Bank wants to try another option which is a bit more difficult to implement - they want to calculate data growth using an interest calculation, just like in a traditional savings account you might have with a bank.
If the annual interest rate is set at 6% and the Data Bank team wants to reward its customers by increasing their data allocation based off the interest calculated on a daily basis at the end of each day, how much data would be required for this option on a monthly basis?

### không tính lãi kép
```sql
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
```

