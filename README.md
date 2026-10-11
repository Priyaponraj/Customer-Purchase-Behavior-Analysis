
# Customer Analytics with SQL: RFM-Based Customer Segmentation

This project performs RFM (Recency, Frequency, Monetary) Analysis on customer transaction data to segment customers into meaningful business groups. RFM analysis is a powerful marketing technique used to identify high-value customers and optimize customer retention strategies.


🎯 What is RFM?
RFM evaluates customers using three key metrics:





| Metric          | Meaning                                      | Example    |
| :-------------- | :------------------------------------------- | :--------- |
| `Recency (R)`   | How recently the customer purchased          | 5 days ago |
| `Frequency (F)` | How often the customer purchased             | 20 orders  |
| `Monetary (M)`  | How much the customer spent                  | ₹250,000   |



## Core Concept

```
Recent Purchase + Frequent Purchases + High Spending  
                    ↓  
            High-Value Customer
```
## 🗄️ Database Setup

```
CREATE DATABASE Customer_Analytics_with_SQL;
USE Customer_Analytics_with_SQL;
```

## 📋 Tables Used

orders - Contains transaction data (customer_id, order_id, order_date, total_amount, order_status)

customer - Contains customer information (customer_id, and other demographics)

## 🔄 Analysis Workflow

Step 1: Calculate RFM Metrics
Calculate the raw RFM values for each customer:

Recency: Days since last purchase (relative to the latest completed order date)

Frequency: Count of distinct orders

Monetary: Total amount spent

```
with rfm as (
select customer_id,
	datediff((select max(order_date) from orders where order_status='Completed') , max(order_date))  as 'recency',
	count(distinct order_id) as 'frequency',
    sum(total_amount) as 'monetary'
from orders
group by customer_id
)
select * from rfm;
```
Sample Output

![RFM Output](https://github.com/user-attachments/assets/1b0e9ac1-f833-4f3b-95d6-d3f91916996c)

## Step 2: Assign RFM Scores

Use NTILE(5) to divide customers into 5 equal groups, assigning scores from 1-5:

```
with rfm as (
select customer_id,
		datediff((select max(order_date) from orders where order_status='Completed') , max(order_date)) as 'recency',
        count(distinct order_id) as 'frequency',
        sum(total_amount) as 'monetary'
from orders
group by customer_id
), rfm_score as (
 select *,ntile(5) over(order by recency) as 'recency_score',
		   ntile(5) over(order by frequency ) as 'frequency_score',
           ntile(5) over(order by monetary) as 'monetary_score'
 from rfm
)
select * from rfm_score;
```

Sample Output

![RFM Output](https://github.com/user-attachments/assets/4a122764-2a9f-419f-aa7d-51ba97a41bc7)

Note: For Recency, lower values are better (more recent purchases). The scoring can be inverted depending on business logic.

## Step 3: Customer Segmentation

Customers are segmented into 6 meaningful groups based on their RFM scores:

| Parameter            | Type     | Description                          |
| :------------------- | :------- | :----------------------------------- |
| `Champions`          | `string` | **Required**. R≥4, F≥4, M≥4         |
| `Loyal Customers`    | `string` | **Required**. R≥3, F≥4, M≥3         |
| `New Customers`      | `string` | **Required**. R≥4, F≥2              |
| `At-Risk Customers`  | `string` | **Required**. R≥2, F≥3, M≥3         |
| `Lost Customers`     | `string` | **Required**. R≥2, F≥2, M≥2         |
| `Potential Loyalists`| `string` | **Optional**. All other combinations |

```
with rfm as (
select customer_id,
		datediff((select max(order_date) from orders where order_status='Completed') , max(order_date)) as 'recency',
        count(distinct order_id) as 'frequency',
        sum(total_amount) as 'monetary'
from orders
group by customer_id
), rfm_score as (
 select *,ntile(5) over(order by recency) as 'recency_score',
		   ntile(5) over(order by frequency ) as 'frequency_score',
           ntile(5) over(order by monetary) as 'monetary_score'
 from rfm
), customer_segmention as (
select *, case
			  when recency_score >= 4 and frequency_score >= 4 and monetary_score >= 4 then 'Champions'
			  when recency_score >= 3 and frequency_score >= 4 and monetary_score >= 3 then 'Loyal Customers'
			  when recency_score >= 4 and frequency_score >= 2 then 'New Customers'
			  when recency_score >= 2 and frequency_score >= 3 and monetary_score >= 3 then 'At-Risk Customers'
			  when recency_score >= 2 and frequency_score >= 2 and monetary_score >= 2 then 'Lost Customers'
			  else 'Potential Loyalists'
		 end as customer_segment
from rfm_score          
)
select c.customer_id,
cs.recency,cs.frequency,cs.monetary,
cs.recency_score,cs.frequency_score,cs.monetary_score,
cs.customer_segment from customer_segmention cs
join customer c
on cs.customer_id = c.customer_id
order by cs.monetary desc;
```

Sample Output

![RFM Output](https://github.com/user-attachments/assets/faa0845b-0f47-44f8-b71f-1b321a28afc4)

## Step 4: Segment Summary

Analyze each segment with aggregate statistics:

```
with rfm as (
select customer_id,
		datediff((select max(order_date) from orders where order_status='Completed') , max(order_date)) as 'recency',
        count(distinct order_id) as 'frequency',
        sum(total_amount) as 'monetary'
from orders
group by customer_id
), rfm_score as (
 select *,ntile(5) over(order by recency) as 'recency_score',
		   ntile(5) over(order by frequency ) as 'frequency_score',
           ntile(5) over(order by monetary) as 'monetary_score'
 from rfm
), customer_segment as (
select *, case
			  when recency_score >= 4 and frequency_score >= 4 and monetary_score >= 4 then 'Champions'
			  when recency_score >= 3 and frequency_score >= 4 and monetary_score >= 3 then 'Loyal Customers'
			  when recency_score >= 4 and frequency_score >= 2 then 'New Customers'
			  when recency_score >= 2 and frequency_score >= 3 and monetary_score >= 3 then 'At-Risk Customers'
			  when recency_score >= 2 and frequency_score >= 2 and monetary_score >= 2 then 'Lost Customers'
			  else 'Potential Loyalists'
		 end as customer_segment
from rfm_score
          
)
SELECT
    customer_segment,
    COUNT(*) AS customer_count,
    ROUND(AVG(monetary), 2) AS avg_customer_value,
    ROUND(AVG(frequency), 2) AS avg_frequency,
    ROUND(AVG(recency), 2) AS avg_recency
FROM customer_segment
GROUP BY customer_segment
ORDER BY customer_count DESC;

```

Sample Output

![RFM Output](https://github.com/user-attachments/assets/f39c372b-19b0-4997-b6a9-f03054c3200f)

## 📈 RFM View

A reusable view rfm_view is created for easy access to segmented customer data:

```
create view rfm_view as
with rfm as (
select customer_id,
		datediff((select max(order_date) from orders where order_status='Completed') , max(order_date)) as 'recency',
        count(distinct order_id) as 'frequency',
        sum(total_amount) as 'monetary'
from orders
group by customer_id
), rfm_score as (
 select *,ntile(5) over(order by recency) as 'recency_score',
		   ntile(5) over(order by frequency ) as 'frequency_score',
           ntile(5) over(order by monetary) as 'monetary_score'
 from rfm
), customer_segment as (
select *, case
			  when recency_score >= 4 and frequency_score >= 4 and monetary_score >= 4 then 'Champions'
			  when recency_score >= 3 and frequency_score >= 4 and monetary_score >= 3 then 'Loyal Customers'
			  when recency_score >= 4 and frequency_score >= 2 then 'New Customers'
			  when recency_score >= 2 and frequency_score >= 3 and monetary_score >= 3 then 'At-Risk Customers'
			  when recency_score >= 2 and frequency_score >= 2 and monetary_score >= 2 then 'Lost Customers'
			  else 'Potential Loyalists'
		 end as customer_segment
from rfm_score
)
SELECT *
FROM customer_segment


select * from rfm_view;
```


## 📌 Notes

* The analysis date is assumed to be the latest completed order date

* Only orders with status 'Completed' are considered

* RFM scores range from 1 (lowest) to 5 (highest)

* Adjust segmentation thresholds based on business needs
