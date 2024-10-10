# 8 Week SQL Challenges by Danny | Case Study #4 - Data Bank
### ERD for Database

![image](https://github.com/user-attachments/assets/bf935242-6ace-40a0-b05b-febc4857c44f)

## Result of Questions
### Data Preparation
```sql
-- a. Crosscheck duplication for table 'customer_nodes'
SELECT count(*)
FROM
(
	SELECT customer_id, region_id, node_id, start_date, end_date
	,count(*) as records
	FROM data_bank.customer_nodes
	GROUP BY 1,2,3,4,5
)
WHERE records > 1;
-- Output: no duplicated rows in table 'customer_nodes'
-- b. Crosscheck duplication for table 'customer_nodes'
SELECT count(*)
FROM
(
	SELECT customer_id, txn_date, txn_type, txn_amount
	,count(*) as records
	FROM data_bank.customer_transactions
	GROUP BY 1,2,3,4
)
WHERE records > 1;
-- Output: no duplicated rows in table 'customer_transactions'
```

```sql
-- Define a list of wrong years
WITH wrong_years AS (
    SELECT
        extract(year FROM end_date) AS end_year,
        COUNT(*) AS count
    FROM
        data_bank.customer_nodes
    WHERE
        extract(year FROM end_date) IN (9999, 2100, 2200, 2300)  -- Add other wrong years here
    GROUP BY
        end_year
    ORDER BY
        end_year
)

-- Update the end_year column for the specified wrong years
UPDATE data_bank.customer_nodes
SET end_date = '2020-12-31' 
WHERE end_date IN ('9999-12-31', '2100-12-31', '2200-12-31', '2300-12-31');  -- Update all wrong years

--  check the results of the CTE if needed
SELECT * FROM wrong_years;
```
### A. Customer Nodes Exploration
### A1. How many unique nodes are there on the Data Bank system?
```sql
SELECT count(distinct node_id) as total_unique_nodes
FROM data_bank.customer_nodes;
```

_Result:_

![image](https://github.com/user-attachments/assets/48cd0730-ce40-40a6-b3b6-56be9133078e)

### A2. What is the number of nodes per region?
```sql
SELECT cn.region_id
,r.region_name
,count(node_id) as nodes_count
FROM data_bank.customer_nodes cn
JOIN data_bank.regions r on r.region_id = cn.region_id
GROUP BY 1,2
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/43797809-f3f6-448d-9c32-50e4d9e8288c)

### A3. How many customers are allocated to each region?
```sql
SELECT cn.region_id
,r.region_name
,count(customer_id) as customers_count
FROM data_bank.customer_nodes cn
JOIN data_bank.regions r on r.region_id = cn.region_id
GROUP BY 1,2
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/6efca23f-d8dc-4005-9852-d16afbc9387a)

### A4. How many days on average are customers reallocated to a different node?
```sql
SELECT 
round(avg(cn2.start_date - cn1.end_date),2) as avg_days_reallocated_different_node
FROM data_bank.customer_nodes cn1
JOIN data_bank.customer_nodes cn2 on cn2.customer_id = cn1.customer_id
WHERE cn2.node_id != cn1.node_id and cn2.start_date > cn1.end_date 
								and cn2.start_date is not null
								and cn1.end_date is not null
;
```

_Result:_

![image](https://github.com/user-attachments/assets/6e4c0a5f-da26-4b52-b53c-9ae82b07ff49)

### A5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
```sql
WITH reallocation_days as
(
	SELECT cn1.region_id	
	,age(cn2.start_date, cn1.end_date) as days_reallocated
	FROM data_bank.customer_nodes cn1
	JOIN data_bank.customer_nodes cn2 on cn2.customer_id = cn1.customer_id
	WHERE cn2.node_id != cn1.node_id and cn2.start_date > cn1.end_date
									 and cn2.start_date is not null
									 and cn1.end_date is not null						
)
SELECT region_id
,percentile_cont(0.5) within group (order by days_reallocated) as median_days_reallocated
,percentile_cont(0.8) within group (order by days_reallocated) as per80th_days_reallocated
,percentile_cont(0.95) within group (order by days_reallocated) as per95th_days_reallocated
FROM reallocation_days
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/88a1340d-4799-4a76-bcac-fdbff212277b)

Based on median, 80th and 95th percentile, we see Region 4 is the best performance with the lowest reallocation days and Region 1 needs to be improved.
- Process Review for Region 1: Identify workflow inefficiencies and optimize resource allocation to reduce reallocation times.
- Standardization of Best Practices: Share Region 4's practices with slower regions, like Region 1, to improve performance.


### B. Customer Transactions

### B1. What is the unique count and total amount for each transaction type?
```sql
SELECT txn_type
,count(customer_id) as customers_count
,sum(txn_amount) as total_amount
FROM data_bank.customer_transactions
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/477490cd-99e5-43f6-ab57-bd64b111139a)


### B2. What is the average total historical deposit counts and amounts for all customers?

```sql
SELECT count(*) as total_count
,sum(txn_amount) as total_amount
,round(avg(txn_amount),2) as avg_amount
FROM data_bank.customer_transactions
WHERE txn_type = 'deposit';
```

_Result:_

![image](https://github.com/user-attachments/assets/6991f815-6295-466e-b774-da87d3f84039)


### B3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
```sql
WITH txntype_by_customer as
(
	SELECT customer_id
	,to_char(txn_date,'MM-YYYY') as txn_month
	,count (case when txn_type = 'deposit' then customer_id end) as deposit_customers
	,count (case when txn_type = 'purchase' then customer_id end) as purchase_customers
	,count (case when txn_type = 'withdrawal' then customer_id end) as withdrawal_customers
	FROM data_bank.customer_transactions
	GROUP BY 1,2
)
SELECT txn_month as month
,count(*) as customer_count
FROM txntype_by_customer
WHERE deposit_customers > 1 and (purchase_customers >= 1 or withdrawal_customers >= 1)
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/a7949dda-faf8-44dd-b121-be107c49834b)

### B4. What is the closing balance for each customer at the end of the month?
```sql
WITH metrics as
(
	SELECT customer_id
	,to_char(txn_date,'MM-YYYY') as txn_month
	,sum(case when txn_type = 'deposit' then txn_amount else 0 end) as sum_deposit
	,sum(case when txn_type = 'purchase' then txn_amount else 0 end) as sum_purchase
	,sum(case when txn_type = 'withdrawal' then txn_amount else 0 end) as sum_withdrawal
	FROM data_bank.customer_transactions
	GROUP BY 1,2
)
, running_balance as
(
	SELECT customer_id
	,txn_month
	,sum(sum_deposit - sum_purchase - sum_withdrawal) over (partition by customer_id order by to_date(txn_month, 'MM-YYYY')) as closing_balance
	FROM metrics	
)
SELECT customer_id
,txn_month as month
,closing_balance
FROM running_balance
ORDER BY 1,2,3 DESC;
```

_Result:_

![image](https://github.com/user-attachments/assets/13ec935f-0be2-4f5d-91e9-34fcac612569)

### B5. What is the percentage of customers who increase their closing balance by more than 5%?
```sql
WITH metrics as
(
	SELECT customer_id
	,to_char(txn_date,'MM-YYYY') as txn_month
	,sum(case when txn_type = 'deposit' then txn_amount else 0 end) as deposit_sum
	,sum(case when txn_type = 'purchase' then txn_amount else 0 end) as purchase_sum
	,sum(case when txn_type = 'withdrawal' then txn_amount else 0 end) as withdrawal_sum
	FROM data_bank.customer_transactions
	GROUP BY 1,2
)
, running_balance as
(
	SELECT customer_id
	,txn_month
	,sum(deposit_sum - purchase_sum - withdrawal_sum) over (partition by customer_id order by to_date(txn_month,'MM-YYYY')) as closing_balance
	FROM metrics
)
, growth as
(
	SELECT rb1.customer_id
	,((rb2.closing_balance / rb1.closing_balance - 1) *100) as growth_percentage
	FROM running_balance rb1
	JOIN running_balance rb2 on rb1.customer_id = rb2.customer_id 
		and to_date(rb2.txn_month,'MM-YYYY') = to_date(rb1.txn_month,'MM-YYYY') + interval '1 month'
	WHERE rb1.closing_balance > 0
)
SELECT
count(distinct case when g.growth_percentage > 5 then g.customer_id end) as customers_growth
,count(distinct ct.customer_id) as all_customers
,round((count(distinct case when g.growth_percentage > 5 then g.customer_id end) *100/ count(distinct ct.customer_id)),2) || '%' as "the percentage of customers who increase their closing balance"
FROM growth g
JOIN data_bank.customer_transactions ct on ct.customer_id = g.customer_id
;
```

_Result:_

![image](https://github.com/user-attachments/assets/565153ec-ea8a-4269-a1d4-d7bc39e2d7cd)


### C. Data Allocation Challenge
#### Option 1: Customer Balance at the End of Each Month
```sql
WITH monthly_closing_balance as
(
	SELECT customer_id
	,to_char(txn_date,'MM-YYYY') as txn_month
	,sum(case when txn_type = 'deposit' then txn_amount
			  when txn_type = 'purchase' then -txn_amount
			  when txn_type = 'withdrawal' then -txn_amount
			  end) over (partition by customer_id, to_char(txn_date,'MM-YYYY') order by txn_date rows between unbounded preceding and unbounded following) as monthly_closing_balance	  		
	FROM data_bank.customer_transactions
)
SELECT *
FROM monthly_closing_balance
ORDER BY 1,2;
```

_Result:_

![image](https://github.com/user-attachments/assets/bb1e4d12-236c-447a-8e67-075afe5a786d)

#### Option 2: Average Customer Balance Over Last 30 Days
```sql
WITH last_30days_balance as
(
	SELECT customer_id
	,txn_date
	,round(avg(case when txn_type = 'deposit' then txn_amount
				when txn_type = 'purchase' then -txn_amount
				when txn_type = 'withdrawal' then -txn_amount
				end) over (partition by customer_id order by txn_date rows between 30 preceding and current row),2) as last_30days_balance
	FROM data_bank.customer_transactions
)
SELECT *
FROM last_30days_balance
ORDER BY 1,2;
```

_Result:_
![image](https://github.com/user-attachments/assets/c1cc30e9-c21c-4363-a086-3c29dc3f3bf6)

#### Option 3: Running Customer Balance on Realtime
```sql
WITH running_balance as
(
	SELECT customer_id
	,txn_date
	,txn_type
	,sum(case when txn_type = 'deposit' then txn_amount
		      when txn_type = 'purchase' then -txn_amount
		      when txn_type = 'withdrawal' then -txn_amount
		  end) over (partition by customer_id order by txn_date) as running_balance
	FROM data_bank.customer_transactions
)
SELECT *
FROM running_balance
ORDER BY 1,2;
```

_Result:_

![image](https://github.com/user-attachments/assets/aae6b790-a7d6-4c1a-a805-533ae4cd8f8c)

#### Multi-part challenge 
```sql
WITH metrics as
(
	SELECT customer_id
	,txn_date
	,sum(case when txn_type = 'deposit' then txn_amount
				when txn_type = 'purchase' then -txn_amount
				when txn_type = 'withdrawal' then -txn_amount
				end) over (partition by customer_id order by txn_date) as running_balance					
	FROM data_bank.customer_transactions	
)
SELECT customer_id
,round(max(running_balance),2) as max_balance
,round(min(running_balance),2) as min_balance
,round(avg(running_balance),2) as avg_balance
FROM metrics
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/84efbf99-f3dc-4d67-8f32-95b5f8583f91)

#### Example for Aggregating Monthly Data
```sql
WITH option1 AS (
    -- Calculate monthly closing balance (from Option 1 query)
),
option2 AS (
    -- Calculate average 30-day balance (from Option 2 query)
),
option3 AS (
    -- Calculate real-time running balance (from Option 3 query)
)
SELECT
    'Option 1' AS option,
    SUM(closing_balance) AS total_data
FROM option1
UNION
SELECT
    'Option 2' AS option,
    SUM(avg_30day_balance) AS total_data
FROM option2
UNION
SELECT
    'Option 3' AS option,
    SUM(running_balance) AS total_data
FROM option3;
```

Thank you for stopping by, and I'm pleased to connect with you, my new friend!

**Please do not forget to FOLLOW and star ⭐ the repository if you find it valuable.**

Wish you a day filled with happiness and energy!

Warm regards,

Hien Moon
