# Objective
## The goal of this project is to analyze the sales data of Monday Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

## 📊 Business Questions & Queries

### Q1.  
What is the adjusted population (in millions) of each city (calculated as 25% of the population × 10), and what is their rank?
```sql
SELECT 
	city_name,
	ROUND((population * 0.25 * 10) / 1000000, 2) as population_in_million ,
	city_rank
FROM city
ORDER BY 2

```


### Q2.  
What is the total revenue generated from sales in Q4 of 2023, grouped by city?
```sql
SELECT 
	SUM(sl.total) as total_revenue,
	cy.city_name
FROM sales as sl
JOIN customers as cs ON sl.customer_id = cs.customer_id  
JOIN city as cy ON cs.city_id = cy.city_id
WHERE
	EXTRACT(YEAR FROM sale_date) = 2023 
	AND
	EXTRACT(QUARTER FROM sale_date) = 4
GROUP BY 2
ORDER BY 1 DESC

```


### Q3.  
What are the total number of orders placed for each product?
```sql
SELECT 
	p.product_name,
	COUNT(s.sale_id) as total_orders
FROM products p
JOIN sales s ON p.product_id = s.product_id
GROUP BY 1
ORDER BY 2 DESC


```


### Q4.  
What is the average sales amount per customer in each city?
```sql
WITH customer_totals AS (
  SELECT 
    c.customer_id,
    cy.city_name,
    SUM(s.total) AS total_sales_per_customer
  FROM sales s
  JOIN customers c ON s.customer_id = c.customer_id
  JOIN city cy ON c.city_id = cy.city_id
  GROUP BY c.customer_id, cy.city_name
)
SELECT 
  city_name,
  AVG(total_sales_per_customer) AS avg_sales_per_customer
FROM customer_totals
GROUP BY city_name
ORDER BY avg_sales_per_customer DESC;

```


### Q5.  
What is the estimated number of coffee consumers in each city (25% of the population), and how many unique customers made purchases?
```sql
WITH coffee_consumer_in_city_table
AS(
	SELECT 
		city_name,
		ROUND((population * 0.25 )	/ 1000000, 2) as estimated_coffee_consumers_in_million
	FROM city
),

total_customers_table
AS(
	SELECT 
		ci.city_name,
		COUNT(DISTINCT c.customer_id) as total_unique_customers
	FROM sales s
	JOIN customers c ON s.customer_id = c.customer_id
	JOIN city ci ON c.city_id = ci.city_id
	GROUP BY 1 
)
SELECT 
    t.city_name,
    c.estimated_coffee_consumers_in_million,
    t.total_unique_customers
FROM coffee_consumer_in_city_table c
JOIN total_customers_table t ON c.city_name = t.city_name
ORDER BY 2 DESC;

```


### Q6.  
What are the top 3 most sold products in each city?
```sql
SELECT *
FROM 
(
	SELECT 
		p.product_id,
		p.product_name,
		ci.city_name,
		COUNT( s.sale_id) as total_id,
		DENSE_RANK() OVER(PARTITION BY ci.city_name ORDER BY COUNT( s.sale_id) DESC) as rnk
	FROM products p
	JOIN sales s ON p.product_id = s.product_id
	JOIN customers c ON s.customer_id = c.customer_id
	JOIN city ci ON c.city_id = ci.city_id
	GROUP BY 1,2,3
) as t1
WHERE rnk <= 3

```


### Q7.  
How many unique customers bought coffee-related products, grouped by city?
```sql
SELECT 
    ci.city_name,
    COUNT(DISTINCT c.customer_id) AS unique_coffee_customers
FROM sales s
JOIN products p ON s.product_id = p.product_id
JOIN customers c ON s.customer_id = c.customer_id
JOIN city ci ON c.city_id = ci.city_id
WHERE LOWER(p.product_name) LIKE '%coffee%'
GROUP BY ci.city_name
ORDER BY unique_coffee_customers DESC;

```


### Q8.  
What is the average sales per customer and average rent per customer in each city?
```sql
WITH 
city_customer_totals 
AS (
  SELECT 
    cy.city_name,
	COUNT(DISTINCT s.customer_id) as total_customers,
    ROUND(SUM(s.total)::NUMERIC/ COUNT(DISTINCT s.customer_id)::NUMERIC,2) AS avg_sales_per_customer
  FROM sales s
  JOIN customers c ON s.customer_id = c.customer_id
  JOIN city cy ON c.city_id = cy.city_id
  GROUP BY 1
),
city_rent
AS(
	SELECT 
		city_name,
		estimated_rent
	FROM city
)
	SELECT 
		cct.total_customers,
		cr.city_name,
		cct.avg_sales_per_customer,
		cr.estimated_rent,
		ROUND(cr.estimated_rent::NUMERIC/cct.total_customers::NUMERIC,2) as avg_rent_per_customers
	FROM city_rent cr
	JOIN city_customer_totals cct ON cr.city_name = cct.city_name
	ORDER BY avg_sales_per_customer DESC

```


### Q9.  
What is the month-over-month growth or loss ratio in total sales for each city?
```sql
WITH 
		monthly_sales
	AS(
		SELECT 
			ci.city_name,
			EXTRACT(MONTH FROM s.sale_date) as month,
			EXTRACT(YEAR FROM s.sale_date) as year,
			SUM(s.total) as total_sale
		FROM sales s
		JOIN customers c ON s.customer_id = c.customer_id
		JOIN city ci ON c.city_id = ci.city_id
		JOIN products p ON s.product_id = p.product_id 
		GROUP BY 1,2,3
		ORDER BY 1,3,2
	),
	growth_ratio
	AS(
		SELECT 
			city_name,
			month,
			year,
		total_sale as current_month_sale,
		LAG(total_sale, 1) OVER(PARTITION BY city_name ORDER BY year, month) as last_month_sale
		FROM monthly_sales
	)
	
	SELECT 
		city_name,
		month,
		year,
		current_month_sale,
		last_month_sale,
		ROUND((current_month_sale - last_month_sale)::NUMERIC/last_month_sale::NUMERIC * 100, 2) as growth_or_loss_ratio
	FROM growth_ratio
	WHERE last_month_sale IS NOT NULL

```


### Q10.  
Which top 3 cities have the highest average sales per customer, and what are their rent, revenue, and estimated coffee consumer metrics?
```sql
WITH 
city_customer_totals 
AS (
  SELECT 
    cy.city_name,
	SUM(s.total) as total_revenue,
	COUNT(DISTINCT s.customer_id) as total_customers,
    ROUND(SUM(s.total)::NUMERIC/ COUNT(DISTINCT s.customer_id)::NUMERIC,2) AS avg_sales_per_customer
  FROM sales s
  JOIN customers c ON s.customer_id = c.customer_id
  JOIN city cy ON c.city_id = cy.city_id
  GROUP BY 1
  ORDER BY total_revenue DESC
),
city_rent
AS(
	SELECT 
		city_name,
		estimated_rent,
		ROUND(population * 0.25/1000000, 3) as estimated_coffee_consumer_in_million
	FROM city
)
	SELECT 
		cct.total_customers,
		cr.city_name,
		cct.total_revenue,
		cct.avg_sales_per_customer,
		cr.estimated_rent as total_rent,
		cr.estimated_coffee_consumer_in_million,
		ROUND(cr.estimated_rent::NUMERIC/cct.total_customers::NUMERIC,2) as avg_rent_per_customers
	FROM city_rent cr
	JOIN city_customer_totals cct ON cr.city_name = cct.city_name 
	ORDER BY avg_sales_per_customer DESC
	LIMIT 3

```


### Recommendations
After analyzing the data, the recommended top three cities for new store openings are:

City 1: Pune

- Average rent per customer is very low.
- Highest total revenue.
- Average sales per customer is also high.

City 2: Delhi

- Highest estimated coffee consumers at 7.7 million.
- Highest total number of customers, which is 68.
- Average rent per customer is 330 (still under 500).

City 3: Jaipur

- Highest number of customers, which is 69.
- Average rent per customer is very low at 156.
- Average sales per customer is better at 11.6k.
