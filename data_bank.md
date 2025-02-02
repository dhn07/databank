### Data Bank Case Study

In this case study, I will conduct some data analysis on Data Bank, a digital bank with customers, nodes and transactions. I will answer a set of important questions below using SQL.
This was originally sourced from the [8-week SQL challenge.](https://8weeksqlchallenge.com/case-study-4/) by Danny Ma.
Feel free to contact me on my [LinkedIn](https://www.linkedin.com/in/dhn07/) for any enquiries.

---

##### A. Customer Nodes Exploration
1. How many unique nodes are there on the Data Bank system?
2. What is the number of nodes per region?
3. How many customers are allocated to each region?
4. How many days on average are customers reallocated to a different node?
5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

##### B. Customer Transactions
6. What is the unique count and total amount for each transaction type?
7. What is the average total historical deposit counts and amounts for all customers?
8. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
9. What is the closing balance for each customer at the end of the month?

---
#### A. Customer Nodes Exploration
---

##### Q1. How many unique nodes are there on the Data Bank system?

Nodes are listed in the ```customer_nodes``` table under ```node_id``` so we will count the number of distinct ```node_id```'s to determine the number of unique nodes.

```sql

SELECT 
    COUNT(DISTINCT node_id) 
FROM data_bank.customer_nodes
```

##### Query Results

| count |
| ----- |
| 5     |


There are 5 unique nodes. 

---

##### Q2. What is the number of nodes per region?

We will need to join the ```regions``` table to determine the number of nodes per region. Similar to Q1 we then need to count the distinct ```node_ids```.

```sql
  SELECT 
    r.region_name,
    r.region_id,
    COUNT(DISTINCT c.node_id) AS unique_nodes
  FROM customer_nodes c
  LEFT JOIN regions r ON r.region_id = c.region_id
  GROUP BY r.region_name, r.region_id
  ORDER BY r.region_id ASC
```

##### Query Results

| region_name | region_id | unique_nodes |
|-------------|-----------|--------------|
| Australia   | 1         | 5            |
| America     | 2         | 5            |
| Africa      | 3         | 5            |
| Asia        | 4         | 5            |
| Europe      | 5         | 5            |

There are 5 nodes per region.

---

##### Q3. How many customers are allocated to each region?

Since we have already grouped the data by region, we can simply start counting the total number of rows to determine the number of customers allocated to each region.

```sql

  SELECT 
    r.region_name,
    COUNT(c.customer_id) AS total_customers
  FROM customer_nodes c
  LEFT JOIN regions r ON r.region_id = c.region_id
  GROUP BY r.region_name, r.region_id
  ORDER BY r.region_id ASC
```

##### Query Results

| region_name | total_customers |
|-------------|----------------|
| Australia   | 770            |
| America     | 735            |
| Africa      | 714            |
| Asia        | 665            |
| Europe      | 616            |

-There are 770 customers allocated to Australia
-There are 735 customers allocated to America
-There are 714 customers allocated to Africa
-There are 665 customers allocated to Asia
-There are 616 customers allocated to Europe

---

##### Q4. How many days on average are customers reallocated to a different node?

Using the ```customer_nodes``` table we can subtract the end_date from the start_date to get a number of days the customer was allocated to that node. Once we have all the days we can get the average with the ```AVG``` function.
There are some outlier dates found in the rows which we have excluded with the ```WHERE``` function.

```sql
SELECT 
    ROUND(AVG(end_date - start_date), 2) AS avg_days_in_node
FROM customer_nodes 
WHERE end_date != '9999-12-31'	        
```

##### Query Results

| avg_days_in_node |
|------------------|
| 14.63           |

The average number of days a customer is allocated to a node is 14.63 days.

---

##### Q5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

 For this question we are asked to get median, 80th and 95th percentile values for the number of days customers are in the node rather than the average.
 For this we are using the ```percentile_cont``` function which allows us to specify an exact percentage in the range of values.

 ```sql

 SELECT 
  r.region_name,
  percentile_cont(0.5) WITHIN GROUP (ORDER BY end_date - start_date) AS median_days,
  percentile_cont(0.8) WITHIN GROUP (ORDER BY end_date - start_date) AS "80th percentile",
  percentile_cont(0.95) WITHIN GROUP (ORDER BY end_date - start_date) AS "95th percentile"
FROM customer_nodes c
LEFT JOIN regions r ON r.region_id = c.region_id
WHERE end_date != '9999-12-31'
GROUP BY r.region_name
```

##### Query Results

| region_name | median_days | 80th percentile | 95th percentile |
|-------------|------------|-----------------|-----------------|
| Africa      | 15         | 24              | 28              |
| America     | 15         | 23              | 28              |
| Asia        | 15         | 23              | 28              |
| Australia   | 15         | 23              | 28              |
| Europe      | 15         | 24              | 28              |

The results range from 15 days in the median result to 28 days in the 95th percentile which is expected of the top end of the range.

---

#### B. Customer Transactions Exploration

---

##### Q6. What is the unique count and total amount for each transaction type?

We will now explore the ```customer_transactions``` table. To answer this question we will ```COUNT``` the total amount of transaction types and ```SUM``` the total amount of those transactions. Finally grouping them by transaction type.

```sql

SELECT 
	txn_type, 
	COUNT(txn_type) AS total_transactions,
    SUM(txn_amount) AS total_amount
FROM customer_transactions
GROUP BY txn_type
```

##### Query Results

| txn_type   | total_transactions | total_amount |
|------------|-------------------|--------------|
| purchase   | 1617              | 806537       |
| deposit    | 2671              | 1359168      |
| withdrawal | 1580              | 793003       |

---

##### Q7. What is the average total historical deposit counts and amounts for all customers?
To answer this question, we need to group the customer deposit transactions by their ```customer_id```. Then we ```COUNT``` the number of deposit transactions so that we can average it, and get the ```AVG``` of their transaction amounts. 

```sql
SELECT 
    ROUND(AVG(transaction_number), 2) as avg_deposits,
    ROUND(AVG(average_deposit_amount), 2) as avg_amount
FROM(
    SELECT 
      customer_id,
      COUNT(customer_id) as transaction_number,
      AVG(txn_amount) as average_deposit_amount
    FROM customer_transactions
    WHERE txn_type = 'deposit'
    GROUP BY customer_id
    ) AS deposits
```

##### Query Results

| avg_deposits | avg_amount |
|-------------|------------|
| 5.34        | 508.61     |

---

##### Q8. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

For this question, we first need to create a query that counts the number of each transaction type for every customer. Not only that, but it should be separated by month as the question asks. We ccan use the ```DATE_PART``` function to extract the month from the ```txn_date``` column. Then we can use the ```SUM(CASE WHEN)``` function to count the number of transactions of each type.
Once we have that result, we can then filter as needed using the ```WHERE``` function.

```sql
SELECT 
    month, 
    COUNT(DISTINCT customer_id)
FROM (
      SELECT 
          customer_id, 
          DATE_PART('month', txn_date) AS month,
          SUM(CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) AS deposits,
          SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) AS withdrawals,
          SUM(CASE WHEN txn_type = 'purchase' THEN 1 ELSE 0 END) AS purchases
      FROM customer_transactions
      GROUP BY customer_id, DATE_PART('month', txn_date)
      ORDER BY customer_id ASC
    ) as monthly_transactions
WHERE deposits > 1 AND withdrawals > 0 OR deposits > 1 AND purchases > 0 
GROUP BY month
```

##### Query Results

| month | count |
|-------|-------|
| 1     | 168   |
| 2     | 181   |
| 3     | 192   |
| 4     | 70    |

---

##### Q9. What is the closing balance for each customer at the end of the month?
This was a very tricky question to answer. The approach I took was to first create a table with a monthly net balance amount which adds deposit amounts and subtracts purchases and withdrawals.
This is achieved via the ```SUM(CASE WHEN)``` function that checks the transaction type then adds or subtracts depending on the type. The problem with this is that is results in only the monthly change, so it does not take into account the running cumulative balance. To get that figure, we select the table again but use a ```SUM OVER PARTITION BY``` function on the month_net_balance to add or subtract the previous months net balance in the final query. I have limited the query to the first 10 given the size being too large.

```sql
SELECT
  customer_id,
  month,
  SUM(month_net_balance) OVER (PARTITION BY customer_id
                                 ORDER BY month
                                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
                                 AS monthly_ending_balance
FROM(
    SELECT 
        customer_id,
        DATE_PART('month', txn_date) AS month,
        SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE 0 END) 
          - SUM(CASE WHEN txn_type IN ('purchase', 'withdrawal') THEN txn_amount ELSE 0 END) 
          AS month_net_balance
    FROM customer_transactions
    GROUP BY customer_id, DATE_PART('month', txn_date)
    ORDER BY customer_id, DATE_PART('month', txn_date)
) AS balances
LIMIT 10
```
##### Query Results

| customer_id | month | monthly_ending_balance |
|------------|-------|------------------------|
| 1          | 1     | 312                    |
| 1          | 3     | -640                   |
| 2          | 1     | 549                    |
| 2          | 3     | 610                    |
| 3          | 1     | 144                    |
| 3          | 2     | -821                   |
| 3          | 3     | -1222                  |
| 3          | 4     | -729                   |
| 4          | 1     | 848                    |
| 4          | 3     | 655                    |

Some balances are negative which is typically not what we would see in real transactions, but this is due to the dataset only providing transaction data and no balance data for example.

---


### Conclusion

This was a very interesting case study with banking as data type. Achieving certain results with SQL feel much different to a language like python or even using excel. I was familiar with many function types but learnt a lot more in this case study. I recommend doing this yourself if you are learning SQL!

--- 

#####  Links:
- [8-week SQL challenge.](https://8weeksqlchallenge.com/)
- [GitHub](https://github.com/dhn07)
- [LinkedIn](https://www.linkedin.com/in/dhn07/)