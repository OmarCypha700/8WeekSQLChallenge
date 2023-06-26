# WEEK 4: DATA BANK 🏦

## Table of Contents
- [Customer Nodes Exploration](#a-customer-nodes-exploration)
- [Customer Transactions](#b-customer-transactions)
- [Data Allocation Challenge](#c-data-allocation-challenge)
- [Extra Challenge](#d-extra-challenge)

## A. Customer Nodes Exploration

#### How many unique nodes are there on the Data Bank system?

```sql
SELECT COUNT(DISTINCT node_id) AS unique_nodes
FROM data_bank.customer_nodes;
```
![SA_Q1](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/fe3d1f8b-4a77-42b9-a33f-acc66a952ebc)

#### What is the number of nodes per region?

```sql
SELECT region_name, COUNT(node_id) AS node_count
FROM data_bank.customer_nodes AS cn
JOIN data_bank.regions AS r
USING (region_id)
GROUP BY region_name
ORDER BY node_count DESC;
```
![SA_Q2](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/d53945d3-7124-4fcc-907a-fe3862c03eb5)

#### How many customers are allocated to each region?

```sql
SELECT region_name, COUNT(DISTINCT customer_id) AS customer_count
FROM data_bank.customer_nodes AS cn
JOIN data_bank.regions AS r
USING (region_id)
GROUP BY region_name
ORDER BY customer_count DESC;
```
![SA_Q3](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/3e0aa93b-e2ac-4043-8c4d-be45b8891b01)

#### How many days on average are customers reallocated to a different node?

```sql
WITH reallocation AS (
SELECT customer_id,
	     node_id,
	     start_date,
	     CASE WHEN LAG(node_id) OVER(PARTITION BY customer_id ORDER BY start_date) != node_id
			 THEN DATEDIFF(start_date, LAG(start_date) OVER (PARTITION BY customer_id ORDER BY start_date))
			 END AS reallocation_days
 FROM customer_nodes
)
SELECT ROUND(AVG(reallocation_days)) as avg_reallocation_days
  FROM reallocation;
```
![SA_Q4](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/e109e73e-215b-4761-87d5-cd4e8c584717)


#### What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

```sql
WITH reallocation_cte AS (
    SELECT
        r.region_name AS region,
        cn.customer_id,
        cn.node_id,
        cn.start_date,
        CASE WHEN LAG(cn.node_id) OVER (PARTITION BY cn.customer_id ORDER BY cn.start_date) != cn.node_id 
            THEN DATEDIFF(cn.start_date, LAG(cn.start_date) OVER (PARTITION BY cn.customer_id ORDER BY cn.start_date))
            ELSE NULL
        END AS reallocation_days
    FROM
        customer_nodes cn
    JOIN
        regions r ON cn.region_id = r.region_id
)
SELECT
    region,
    MAX(CASE WHEN percentile = 50 THEN reallocation_days END) AS median,
    MAX(CASE WHEN percentile = 80 THEN reallocation_days END) AS eightieth_percentile,
    MAX(CASE WHEN percentile = 95 THEN reallocation_days END) AS ninetyfifth_percentile
FROM
    (
        SELECT
            region,
            reallocation_days,
            NTILE(100) OVER (PARTITION BY region ORDER BY reallocation_days) AS percentile
        FROM
            reallocation_cte
    ) AS subquery
GROUP BY
    region;
```
![SA_Q5](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/dd9e98bb-8901-41cd-b4bb-cf987244e22e)

## B. Customer Transactions

#### What is the unique count and total amount for each transaction type?

```sql
SELECT txn_type, 
       COUNT(txn_type) AS txn_type_count,
	     SUM(txn_amount) AS total_txn_amount
FROM data_bank.customer_transactions
GROUP BY txn_type;
```
![SB_Q1](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/798269f1-cbd2-4f4a-ab05-b522e0fbe649)

#### What is the average total historical deposit counts and amounts for all customers?

```sql
SELECT ROUND(COUNT(txn_type) / COUNT(DISTINCT customer_id)) AS avg_deposit_count,
	     ROUND(SUM(txn_amount) / COUNT(DISTINCT customer_id)) AS avg_deposit_amt
FROM data_bank.customer_transactions
WHERE txn_type = 'deposit';
```
![SB_Q2](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/be1266e2-1bc4-498e-8a08-e249c0037dc5)

#### For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

```sql
WITH txn AS (
SELECT customer_id, 
	     txn_type, 
       COUNT(txn_type) AS txn_type_count,
       MONTHNAME(txn_date) AS month
FROM data_bank.customer_transactions
GROUP BY customer_id,txn_type, month
), 
deposit AS (
SELECT customer_id, 
	     txn_type, 
       COUNT(txn_type) AS txn_type_count
FROM data_bank.customer_transactions
WHERE txn_type = 'deposit'
GROUP BY customer_id,txn_type
),
purchase AS (
SELECT customer_id, 
	     txn_type, 
       COUNT(txn_type) AS txn_type_count
FROM data_bank.customer_transactions
WHERE txn_type = 'purchase'
GROUP BY customer_id,txn_type
),
withdrawal AS (
SELECT customer_id, 
	     txn_type, 
       COUNT(txn_type) AS txn_type_count
FROM data_bank.customer_transactions
WHERE txn_type = 'withdrawal'
GROUP BY customer_id,txn_type
)
SELECT t.month, COUNT(customer_id) AS customer_count
FROM txn AS t
JOIN deposit AS d USING (customer_id)
JOIN purchase AS p USING (customer_id)
JOIN withdrawal AS w USING (customer_id)
WHERE d.txn_type_count > 1 
AND (p.txn_type_count = 1 OR w.txn_type_count = 1)
GROUP BY t.month;
```
![SB_Q3](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/ceee79e3-fd3e-4f45-b628-5ba7ce50e5f7)

#### What is the closing balance for each customer at the end of the month?

```sql
SELECT
    DISTINCT customer_id,
    MONTHNAME(txn_date) AS month,
    SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount
		 ELSE -txn_amount END) AS closing_balance
FROM
    data_bank.customer_transactions
WHERE
    MONTH(txn_date) = MONTH(LAST_DAY(txn_date)) -- Filter transactions for the end of the month
GROUP BY
    customer_id, month
ORDER BY customer_id;
```
![SB_Q4](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/ef446bdb-8bb1-4bf3-9978-bbce6e6bc580)

#### What is the percentage of customers who increase their closing balance by more than 5%?

```sql
SELECT
    ROUND((COUNT(*) / (SELECT COUNT(DISTINCT customer_id) FROM customer_transactions) * 100), 1) AS percentage
FROM (
  SELECT
        customer_id,
        MAX(closing_balance) AS max_balance,
        MIN(closing_balance) AS min_balance
FROM (
	SELECT
        DISTINCT customer_id,
        MONTHNAME(txn_date) AS month,
        SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount
           ELSE -txn_amount END) AS closing_balance
	FROM
		data_bank.customer_transactions
	WHERE
		MONTH(txn_date) = MONTH(LAST_DAY(txn_date)) -- Filter transactions for the end of the month
	GROUP BY
		customer_id, month
	ORDER BY customer_id
) AS t
GROUP BY
	customer_id
HAVING
	(((max_balance - min_balance) / min_balance) * 100) > 5
) AS subquery;
```
![SB_Q5](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/1abf9cdf-9f68-4830-8cb6-f9ee1a39e72c)

## C. Data Allocation Challenge

#### To test out a few different hypotheses - the Data Bank team wants to run an experiment where different groups of customers would be allocated data using 3 different options:

- Option 1: data is allocated based off the amount of money at the end of the previous month
- Option 2: data is allocated on the average amount of money kept in the account in the previous 30 days
- Option 3: data is updated real-time
- 
#### For this multi-part challenge question - you have been requested to generate the following data elements to help the Data Bank team estimate how much data will need to be provisioned for each option:

- running customer balance column that includes the impact each transaction
- customer balance at the end of each month
- minimum, average and maximum values of the running balance for each customer

```sql
SELECT
    customer_id,
    txn_date,
    txn_type,
    txn_amount,
    SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) OVER (PARTITION BY customer_id ORDER BY txn_date) AS running_balance,
    LAST_DAY(txn_date) AS end_of_month,
    MONTHNAME(LAST_DAY(txn_date)) AS month,
    MIN(SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END)) OVER (PARTITION BY customer_id ORDER BY LAST_DAY(txn_date)) AS min_running_balance,
    AVG(SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END)) OVER (PARTITION BY customer_id ORDER BY LAST_DAY(txn_date)) AS avg_running_balance,
    MAX(SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END)) OVER (PARTITION BY customer_id ORDER BY LAST_DAY(txn_date)) AS max_running_balance
FROM
    customer_transactions
GROUP BY
    customer_id, txn_date, txn_type, txn_amount
ORDER BY
    customer_id, txn_date;
```
![SC_Q1](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/61fa5ed4-58bb-4ae8-85b8-d9ed5c511e76)

#### Using all of the data available - how much data would have been required for each option on a monthly basis?

```sql
SELECT
      month,
      MAX(running_balance) AS data_option_1,
      ROUND(AVG(avg_running_balance), 2) AS data_option_2,
      MAX(max_running_balance) AS data_option_3
FROM (
    SELECT
          customer_id,
          txn_date,
          txn_type,
          txn_amount,
          SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) 
           OVER (PARTITION BY customer_id ORDER BY txn_date) AS running_balance,
          LAST_DAY(txn_date) AS end_of_month,
          MONTHNAME(LAST_DAY(txn_date)) AS month,
          MIN(SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END)) OVER (PARTITION BY customer_id ORDER BY LAST_DAY(txn_date)) AS min_running_balance,
          AVG(SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END)) OVER (PARTITION BY customer_id ORDER BY LAST_DAY(txn_date)) AS avg_running_balance,          
          MAX(SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END)) OVER (PARTITION BY customer_id ORDER BY LAST_DAY(txn_date)) AS max_running_balance
    FROM
        customer_transactions
    GROUP BY
        customer_id, txn_date, txn_type, txn_amount
) AS subquery
GROUP BY
    month;
```
![SC_Q2](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/6c89c455-7c08-4905-865b-695ab4ff459c)

## D. Extra Challenge

#### Data Bank wants to try another option which is a bit more difficult to implement. 
They want to calculate data growth using an interest calculation, just like in a traditional savings account you might have with a bank. 
If the annual interest rate is set at 6% and the Data Bank team wants to reward its customers by increasing their data allocation based off the interest calculated on a daily basis at the end of each day, how much data would be required for this option on a monthly basis?
#### Special notes:
Data Bank wants an initial calculation which does not allow for compounding interest, however they may also be interested in a daily compounding interest calculation so you can try to perform this calculation if you have the stamina!

```sql
SELECT
    month,
    ROUND(MAX(running_balance) * (1 + 0.06 / 12), 2)AS data_option_1_interest, -- Option 1: Data allocated based on the amount at the end of the previous month
    ROUND(AVG(avg_running_balance) * (1 + 0.06 / 12), 2) AS data_option_2, -- Option 2: Data allocated based on the average amount in the previous 30 days
    ROUND(MAX(max_running_balance) * POW((1 + 0.06 / 365), 30), 2) AS data_option_3 -- Option 3: Data allocated with daily compounding interest
FROM (
    SELECT
        customer_id,
        txn_date,
        txn_type,
        txn_amount,
        SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) OVER (PARTITION BY customer_id ORDER BY txn_date) AS running_balance,
        LAST_DAY(txn_date) AS end_of_month,
        MONTHNAME(LAST_DAY(txn_date)) AS month,
        MIN(SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END)) OVER (PARTITION BY customer_id ORDER BY LAST_DAY(txn_date)) AS min_running_balance,
        AVG(SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END)) OVER (PARTITION BY customer_id ORDER BY LAST_DAY(txn_date)) AS avg_running_balance,
        MAX(SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END)) OVER (PARTITION BY customer_id ORDER BY LAST_DAY(txn_date)) AS max_running_balance
    FROM
        customer_transactions
    GROUP BY
        customer_id, txn_date, txn_type, txn_amount
) AS subquery
GROUP BY
    month;
```
![SD_Q1](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/43d4a625-5c1f-46dc-8ef5-e091bbce56c4)

In this query, the data allocation for each option is calculated by applying the interest rate to the respective balance values. 
Option 1 and Option 2 use a monthly interest rate, while Option 3 applies daily compounding interest over 30 days. 
The results are grouped by month to provide the data required for each option on a monthly basis.

## CHECK OUT [WEEK 5](https://github.com/OmarCypha700/8WeekSQLChallenge/blob/main/WEEK%205:%20Data%20Mart.md) SOLUTION.
