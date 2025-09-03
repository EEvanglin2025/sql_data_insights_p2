# Sales, Customers & Products Analytics using SQL

## Project Overview
**Project Title**: Sales, Customers & Products Analytics using SQL

## Project Overview: 
This project focuses on analyzing sales and customer data using SQL. The aim is to derive actionable insights such as sales trends, customer segmentation, performance benchmarking, and category-level contributions. By leveraging SQL queries, the project demonstrates how raw data can be transformed into meaningful business intelligence.

## Objectives
**1**.To evaluate sales performance over time (yearly and monthly).
**2**.To identify seasonality and long-term business growth trends.
**3**.To measure cumulative performance and detect declining/growing patterns.
**4**.To benchmark product and category performance against targets and averages.
**5**.To analyze category-level contributions to overall sales.
**6**.To segment customers (VIP, Regular, New) and calculate key KPIs like average order value and monthly spend.

## Project Structure

## 1. Change Over Time: 
      Tracking how sales evolve over years provides a high-level view for decision-making. At a monthly level, it highlights seasonality patterns.

**Change Over Year**
```sql
SELECT 
year(order_date),
avg(price)
FROM gold.fact_sales
where order_date is not null
group by year(order_date)
order by year(order_date)
      );
```
```sql
SELECT 
YEAR(order_date)as order_year,
sum(sales_amount) as total_sales_amount,
count(distinct customer_key) as total_customers
FROM gold.fact_sales
where order_date is not null
group by YEAR(order_date)
order by YEAR(order_date);

```

**Change Over Year**
```sql
SELECT 
MONTH(order_date)as order_month,
sum(sales_amount) as total_sales_amount,
count(distinct customer_key) as total_customers
FROM gold.fact_sales
where order_date is not null
group by MONTH(order_date)
order by MONTH(order_date);

```
## 2. Cumulative Analysis
```sql
SELECT
order_date,
total_sales,
SUM(total_sales) OVER (partition by DATEPART(year,order_date) order by order_date) as running_total_sales
FROM
(
SELECT
DATETRUNC(month,order_date) as order_date,
SUM(sales_amount) as total_sales
FROM gold.fact_sales
WHERE order_date IS NOT NULL
GROUP BY DATETRUNC(month,order_date)
) t
```








