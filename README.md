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
YEAR(order_date)as order_year,
sum(sales_amount) as total_sales_amount,
count(distinct customer_key) as total_customers
FROM gold.fact_sales
where order_date is not null
group by YEAR(order_date)
order by YEAR(order_date);

```

**Change Over Month**
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
## 2. Cumulative Analysis:
                  Tracks both monthly and yearly running totals of sales. Helps visualize whether sales are steadily increasing or if there are dips over time.

**For Month**

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

**For Year**

```sql

SELECT
order_date,
total_sales,
SUM(total_sales) OVER (order by order_date) as running_total_sales,
AVG(avg_price) OVER (order by order_date) as running_avg_prices
FROM
(
SELECT
DATETRUNC(year,order_date) as order_date,
SUM(sales_amount) as total_sales,
AVG(price) as avg_price
FROM gold.fact_sales
WHERE order_date IS NOT NULL
GROUP BY DATETRUNC(year,order_date)
) t

```

## 3. Performance Analysis:
                  Compares current product sales against average and previous year’s performance, highlighting underperforming or outperforming products.
      
```sql

WITH yearly_product_sales as
(
SELECT 
year(s.order_date) as order_year,
p.product_name,
sum(s.sales_amount) as current_sales
FROM gold.fact_sales s
LEFT JOIN gold.dim_products p
on s.product_key = p. product_key
where s.order_date is not null
group by p.product_name,year(order_date)
)

SELECT 
order_year,
product_name,
current_sales,
AVG (current_sales) OVER (PARTITION BY product_name) AS avg_current_sales,
current_sales - AVG (current_sales) OVER (PARTITION BY product_name) AS diff_avg,
CASE WHEN current_sales - AVG (current_sales) OVER (PARTITION BY product_name)  > 0 THEN 'Above Avg'
     WHEN current_sales - AVG (current_sales) OVER (PARTITION BY product_name)  < 0 THEN 'Below Avg'
     ELSE 'Avg'
END change_avg,
LAG (current_sales) OVER (PARTITION BY product_name ORDER BY order_year) AS py_sales,
current_sales - LAG (current_sales) OVER (PARTITION BY product_name ORDER BY order_year) AS diff_py_sales,
CASE WHEN current_sales - LAG (current_sales) OVER (PARTITION BY product_name ORDER BY order_year)  > 0 THEN 'Increase'
     WHEN current_sales - LAG (current_sales) OVER (PARTITION BY product_name ORDER BY order_year)  < 0 THEN 'Decrease'
     ELSE 'No Change'
END sales_change
from yearly_product_sales
order by product_name, order_year

```

## 4.Part-to-Whole Analysis:
                  Analyzes contribution of each category to total sales. Useful for identifying the most impactful categories driving overall business.


```sql

WITH category_sales AS
(
SELECT 
p.category,
sum(s.sales_amount) as total_sales
FROM gold.fact_sales s
LEFT JOIN gold.dim_products p
ON s.product_key = p.product_key
GROUP BY p.category
)

SELECT 
category,
total_sales,
SUM(total_sales) OVER() AS overall_sales, 
CONCAT(ROUND((CAST (total_sales AS FLOAT) / SUM(total_sales) OVER())*100, 2), '%') AS percentage_of_total 
FROM category_sales 
ORDER BY total_sales desc

```

## 4.Customer Segmentation Cost Range :
                             Classifies products into cost ranges (Below 100, 100–500, 500–1000, Above 1000) to understand product distribution across price bands. Helps in identifying which price segment dominates the catalog.


```sql

WITH product_segment AS
(
SELECT 
product_key,
product_name,
cost,
CASE WHEN cost < 100 THEN 'Below 100'
     WHEN cost BETWEEN 100 AND 500 THEN '100-500'
     WHEN cost BETWEEN 500 AND 1000 THEN '500-1000'
     ELSE 'Above 1000'
END cost_range
FROM gold.dim_products
)

SELECT 
cost_range,
COUNT(product_key) AS total_products
FROM product_segment
GROUP BY cost_range
ORDER BY total_products DESC

```

## 5.Customer Segmentation (VIP, Regular, New):
                        Divides customers into groups based on lifespan and spending patterns, then aggregates KPIs like total orders, sales, average spend. This helps in targeting retention and marketing strategies.

```sql

WITH customer_spending AS(
SELECT
c.customer_key,
SUM(s.sales_amount) AS total_spending,
MIN(order_date) AS first_order,
MAX(order_date) AS last_order,
DATEDIFF(month, MIN(order_date), MAX(order_date)) AS lifespan
FROM gold.fact_sales s
LEFT JOIN gold.dim_customers c
ON s.customer_key  = c.customer_key
GROUP BY c.customer_key
)

SELECT
customer_segment,
COUNT(customer_key)AS total_customers
FROM
(
    SELECT
    customer_key,
    total_spending,
    lifespan,
    CASE WHEN lifespan >= 12 AND total_spending > 5000 THEN 'VIP'
         WHEN lifespan >= 12 AND total_spending <= 5000 THEN 'Regular'
         ELSE 'New'
    END customer_segment
    FROM customer_spending
) t
GROUP BY customer_segment
ORDER BY total_customers

```

## 6. KPI Analysis:
            Builds a customer-level view that segments users into groups (VIP, Regular, New) and age bands. It aggregates orders, sales, products purchased, and lifespan to calculate KPIs such as average order value and monthly spend. This helps identify high-value customers and tailor retention strategies.

```sql

/*1. Base query: Retreives core columns from tables
2. Segments customers into categories(VIP ,Regular, New) and age group
3. Aggregates customer-level metrics:
    -total orders
    -total sales
    -total quantity purchased
    -lifespan (in months)
4. Calculating valuable KPIs:
     -average order value
     -average monthly spend 
*/
--1. Base query: Retreives core columns from tables

CREATE VIEW gold.report_customers AS
WITH base_query AS
(
SELECT 
s.order_number,
s.product_key,
s.order_date,
s.sales_amount,
s.quantity,
c.customer_key,
c.customer_number,
CONCAT(c.first_name, ' ',c.last_name) AS customer_name,
DATEDIFF(year,c.birthdate, GETDATE()) AS age
FROM gold.fact_sales s
LEFT JOIN gold.dim_customers c
ON s.customer_key=c.customer_key
WHERE order_date IS NOT NULL
)
, customer_segment AS
(
SELECT 
    customer_key,
    customer_number,
    customer_name,
    age,
COUNT(DISTINCT order_number) AS total_orders,
SUM(sales_amount)AS total_sales,
COUNT(DISTINCT product_key)AS total_products,
DATEDIFF(month, MIN(order_date), MAX(order_date)) AS lifespan
FROM base_query
GROUP BY
    customer_key,
    customer_number,
    customer_name,
    age
)
SELECT
customer_key,
customer_number,
customer_name,
age,
CASE WHEN age < 20 THEN 'Under 20'
     WHEN age BETWEEN 20 AND 29 THEN '20-29'
     WHEN age BETWEEN 30 AND 39 THEN '30-39'
     WHEN age BETWEEN 40 AND 49 THEN '40-49'
     ELSE '50 and Above'
END AS age_group,
CASE 
     WHEN lifespan >= 12 AND total_sales > 5000 THEN 'VIP'
     WHEN lifespan >= 12 AND total_sales <= 5000 THEN 'Regular'
     ELSE 'New'
END AS customer_group,
total_orders,
total_sales,
total_products,
lifespan,
CASE WHEN total_sales = 0 THEN 0
     ELSE total_sales/total_orders
END AS avg_order_value,
CASE WHEN lifespan = 0 THEN total_sales
     ELSE total_sales/lifespan
END AS avg_monthly_spend
FROM customer_segment

SELECT * FROM gold.report_customers


```

## Findings:

**Yearly Sales and Price Trends**:
            *Average price and sales vary across years.
            *Provides a high-level view of long-term performance.

**Monthly Sales (Seasonality)**:
            *Monthly data shows seasonal changes in sales and customers.
            *Useful for identifying peak and low periods.

**Cumulative Analysis**:
            *Running totals confirm overall sales growth over months and years.
            *Helps track whether the business is progressing or slowing down.

**Performance Analysis**:
            *Product sales compared with averages and previous year highlight top and weak performers.
            *Supports better product decisions.

**Part-to-Whole (Category Contribution)**:
            *Certain categories contribute more to overall sales.
            *Shows which categories have the greatest business impact.

**Product Segmentation (Cost Range)**:
            *Products are concentrated in mid-price ranges (100–500).
            *Premium and low-cost segments are smaller but important for niche demand.

**Customer Segmentation and KPIs**:
            *VIP customers: long-term and high spenders, most valuable.
            *Regular customers: steady but lower contribution.
            *New customers: significant in number, need retention focus.
            *KPIs like average order value and monthly spend provide clear benchmarks.


## Conclusion:
      The analysis shows that the business is experiencing steady growth with clear seasonal sales patterns. Mid-range products and a few key categories contribute the largest share of revenue, while premium and low-cost products play a smaller but important role. Customer segmentation highlights that VIP customers, though fewer in number, generate the highest value, whereas regular customers provide consistent revenue. A large proportion of new customers indicates growth opportunities but also emphasizes the need for retention strategies. Overall, the findings support focused actions in pricing, product mix, and customer engagement to drive sustainable business performance.







