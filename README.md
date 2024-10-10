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

### C.



Thank you for stopping by, and I'm pleased to connect with you, my new friend!

**Please do not forget to FOLLOW and star ⭐ the repository if you find it valuable.**

Wish you a day filled with happiness and energy!

Warm regards,

Hien Moon
