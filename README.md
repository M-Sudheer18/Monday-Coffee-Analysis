# Monday Coffee Expansion Analysis using SQL

![Monday Coffee Logo](https://github.com/M-Sudheer18/Monday-Coffee-Analysis/blob/main/1.png)

## Overview
This project involves an analysis of Monday Coffee's sales, customers, products, and city data using SQL. The goal is to identify the best cities for opening new coffee shop locations based on sales, customer base, estimated coffee consumers, and rent. This README provides a detailed account of the project's objectives, business problems, solutions, findings, and recommendations.

## Objectives

- Estimate the coffee consumer market in each city.
- Analyze revenue, product sales, and customer counts by city.
- Compare average sales per customer against average rent per customer.
- Track monthly sales growth for each city.
- Recommend the top cities for expansion.

## Dataset

The data for this project covers four related tables:

- **city:** population, estimated rent, and rank for each city
- **products:** coffee products and their prices
- **customers:** customers and the city they belong to
- **sales:** individual sales with date, product, customer, total, and rating

- **Dataset Link:** [Add your dataset link]

## Schema

```sql
DROP TABLE IF EXISTS sales;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS city;

-- Import order: city -> products -> customers -> sales

CREATE TABLE city
(
    city_id         INT PRIMARY KEY,
    city_name       VARCHAR(15),
    population      BIGINT,
    estimated_rent  FLOAT,
    city_rank       INT
);

CREATE TABLE customers
(
    customer_id    INT PRIMARY KEY,
    customer_name  VARCHAR(25),
    city_id        INT,
    CONSTRAINT fk_city FOREIGN KEY (city_id) REFERENCES city(city_id)
);

CREATE TABLE products
(
    product_id    INT PRIMARY KEY,
    product_name  VARCHAR(35),
    price         FLOAT
);

CREATE TABLE sales
(
    sale_id      INT PRIMARY KEY,
    sale_date    DATE,
    product_id   INT,
    customer_id  INT,
    total        FLOAT,
    rating       INT,
    CONSTRAINT fk_products  FOREIGN KEY (product_id)  REFERENCES products(product_id),
    CONSTRAINT fk_customers FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Verify the data load
SELECT * FROM city;
SELECT * FROM products;
SELECT * FROM customers;
SELECT * FROM sales;
```

## Business Problems and Solutions

### 1. Coffee Consumers Count

How many people in each city are estimated to consume coffee, given that 25% of the population does?

```sql
SELECT 
    city_name,
    ROUND((population * 0.25) / 1000000, 2) AS coffee_consumers_in_millions,
    city_rank
FROM city
ORDER BY 2 DESC;
```

**Objective:** Estimate the size of the coffee-drinking market in each city.

### 2. Total Revenue from Coffee Sales

What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?

```sql
-- Overall
SELECT 
    SUM(total) AS total_revenue
FROM sales
WHERE 
    EXTRACT(YEAR FROM sale_date) = 2023
    AND EXTRACT(QUARTER FROM sale_date) = 4;

-- By city
SELECT 
    ci.city_name,
    SUM(s.total) AS total_revenue
FROM sales AS s
JOIN customers AS c
    ON s.customer_id = c.customer_id
JOIN city AS ci
    ON ci.city_id = c.city_id
WHERE 
    EXTRACT(YEAR FROM s.sale_date) = 2023
    AND EXTRACT(QUARTER FROM s.sale_date) = 4
GROUP BY 1
ORDER BY 2 DESC;
```

**Objective:** Measure Q4 2023 revenue overall and for each city.

### 3. Sales Count for Each Product

How many units of each coffee product have been sold?

```sql
SELECT 
    p.product_name,
    COUNT(s.sale_id) AS total_orders
FROM products AS p
LEFT JOIN sales AS s
    ON s.product_id = p.product_id
GROUP BY 1
ORDER BY 2 DESC;
```

**Objective:** Find the best-selling and slowest-selling products. A `LEFT JOIN` keeps products with no sales.

### 4. Average Sales Amount per City

What is the average sales amount per customer in each city?

```sql
SELECT 
    ci.city_name,
    SUM(s.total) AS total_revenue,
    COUNT(DISTINCT s.customer_id) AS total_cx,
    ROUND(
        SUM(s.total)::numeric / COUNT(DISTINCT s.customer_id)::numeric,
        2
    ) AS avg_sale_pr_cx
FROM sales AS s
JOIN customers AS c
    ON s.customer_id = c.customer_id
JOIN city AS ci
    ON ci.city_id = c.city_id
GROUP BY 1
ORDER BY 2 DESC;
```

**Objective:** Compare total revenue, customer count, and average spend per customer across cities.

### 5. City Population and Coffee Consumers (25%)

Provide a list of cities along with their populations and estimated coffee consumers. Return city name, total current customers, and estimated coffee consumers.

```sql
WITH city_table AS 
(
    SELECT 
        city_name,
        ROUND((population * 0.25) / 1000000, 2) AS coffee_consumers
    FROM city
),
customers_table AS
(
    SELECT 
        ci.city_name,
        COUNT(DISTINCT c.customer_id) AS unique_cx
    FROM sales AS s
    JOIN customers AS c
        ON c.customer_id = s.customer_id
    JOIN city AS ci
        ON ci.city_id = c.city_id
    GROUP BY 1
)
SELECT 
    customers_table.city_name,
    city_table.coffee_consumers AS coffee_consumer_in_millions,
    customers_table.unique_cx
FROM city_table
JOIN customers_table
    ON city_table.city_name = customers_table.city_name;
```

**Objective:** Compare the potential market (estimated consumers) with the current customer base in each city.

### 6. Top Selling Products by City

What are the top 3 selling products in each city based on sales volume?

```sql
SELECT * 
FROM
(
    SELECT 
        ci.city_name,
        p.product_name,
        COUNT(s.sale_id) AS total_orders,
        DENSE_RANK() OVER(
            PARTITION BY ci.city_name 
            ORDER BY COUNT(s.sale_id) DESC
        ) AS rank
    FROM sales AS s
    JOIN products AS p
        ON s.product_id = p.product_id
    JOIN customers AS c
        ON c.customer_id = s.customer_id
    JOIN city AS ci
        ON ci.city_id = c.city_id
    GROUP BY 1, 2
) AS t1
WHERE rank <= 3;
```

**Objective:** Use `DENSE_RANK()` to find each city's three most ordered products.

### 7. Customer Segmentation by City

How many unique customers are there in each city who have purchased coffee products?

```sql
SELECT 
    ci.city_name,
    COUNT(DISTINCT c.customer_id) AS unique_cx
FROM city AS ci
LEFT JOIN customers AS c
    ON c.city_id = ci.city_id
JOIN sales AS s
    ON s.customer_id = c.customer_id
WHERE 
    s.product_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14)
GROUP BY 1;
```

**Objective:** Count the unique customers in each city who bought coffee products (product IDs 1 to 14).

### 8. Average Sale vs Rent

Find each city and its average sale per customer and average rent per customer.

```sql
WITH city_table AS
(
    SELECT 
        ci.city_name,
        SUM(s.total) AS total_revenue,
        COUNT(DISTINCT s.customer_id) AS total_cx,
        ROUND(
            SUM(s.total)::numeric / COUNT(DISTINCT s.customer_id)::numeric,
            2
        ) AS avg_sale_pr_cx
    FROM sales AS s
    JOIN customers AS c
        ON s.customer_id = c.customer_id
    JOIN city AS ci
        ON ci.city_id = c.city_id
    GROUP BY 1
),
city_rent AS
(
    SELECT 
        city_name, 
        estimated_rent
    FROM city
)
SELECT 
    cr.city_name,
    cr.estimated_rent,
    ct.total_cx,
    ct.avg_sale_pr_cx,
    ROUND(
        cr.estimated_rent::numeric / ct.total_cx::numeric,
        2
    ) AS avg_rent_per_cx
FROM city_rent AS cr
JOIN city_table AS ct
    ON cr.city_name = ct.city_name
ORDER BY 4 DESC;
```

**Objective:** Weigh what each customer spends against the rent each customer has to cover.

### 9. Monthly Sales Growth

Calculate the percentage growth (or decline) in sales month over month for each city.

```sql
WITH monthly_sales AS
(
    SELECT 
        ci.city_name,
        EXTRACT(MONTH FROM sale_date) AS month,
        EXTRACT(YEAR FROM sale_date) AS year,
        SUM(s.total) AS total_sale
    FROM sales AS s
    JOIN customers AS c
        ON c.customer_id = s.customer_id
    JOIN city AS ci
        ON ci.city_id = c.city_id
    GROUP BY 1, 2, 3
),
growth_ratio AS
(
    SELECT
        city_name,
        month,
        year,
        total_sale AS cr_month_sale,
        LAG(total_sale, 1) OVER(
            PARTITION BY city_name 
            ORDER BY year, month
        ) AS last_month_sale
    FROM monthly_sales
)
SELECT
    city_name,
    month,
    year,
    cr_month_sale,
    last_month_sale,
    ROUND(
        (cr_month_sale - last_month_sale)::numeric / last_month_sale::numeric * 100,
        2
    ) AS growth_ratio
FROM growth_ratio
WHERE last_month_sale IS NOT NULL;
```

**Objective:** Use `LAG()` to compare each month with the previous one and spot growing or declining cities.

### 10. Market Potential Analysis

Identify the top 3 cities based on highest sales. Return city name, total sale, total rent, total customers, and estimated coffee consumers.

```sql
WITH city_table AS
(
    SELECT 
        ci.city_name,
        SUM(s.total) AS total_revenue,
        COUNT(DISTINCT s.customer_id) AS total_cx,
        ROUND(
            SUM(s.total)::numeric / COUNT(DISTINCT s.customer_id)::numeric,
            2
        ) AS avg_sale_pr_cx
    FROM sales AS s
    JOIN customers AS c
        ON s.customer_id = c.customer_id
    JOIN city AS ci
        ON ci.city_id = c.city_id
    GROUP BY 1
),
city_rent AS
(
    SELECT 
        city_name, 
        estimated_rent,
        ROUND((population * 0.25) / 1000000, 3) AS estimated_coffee_consumer_in_millions
    FROM city
)
SELECT 
    cr.city_name,
    total_revenue,
    cr.estimated_rent AS total_rent,
    ct.total_cx,
    estimated_coffee_consumer_in_millions,
    ct.avg_sale_pr_cx,
    ROUND(
        cr.estimated_rent::numeric / ct.total_cx::numeric,
        2
    ) AS avg_rent_per_cx
FROM city_rent AS cr
JOIN city_table AS ct
    ON cr.city_name = ct.city_name
ORDER BY 2 DESC;
```

**Objective:** Combine revenue, rent, customer count, and market size in one view to shortlist expansion cities.

## SQL Concepts Used

- Aggregation: `COUNT`, `SUM`, `ROUND`, `GROUP BY`, `ORDER BY`
- Joins: `JOIN`, `LEFT JOIN` across four related tables
- Window functions: `DENSE_RANK()`, `LAG()` with `PARTITION BY`
- Subqueries and CTEs (`WITH`)
- Date functions: `EXTRACT(YEAR / QUARTER / MONTH ...)`
- Filtering: `WHERE`, `IN`, `IS NOT NULL`
- Type casting: `::numeric`

## Findings and Conclusion

- **Market Size:** Estimating 25% of each city's population as coffee drinkers shows where the largest potential customer base is.
- **Revenue and Customers:** Revenue, customer counts, and average spend per customer differ clearly between cities.
- **Product Demand:** Top-selling products by city show which items to stock first in a new location.
- **Cost vs Revenue:** Comparing average sale per customer with average rent per customer highlights cities where rent is easy to absorb.
- **Growth Trend:** Month-over-month growth shows which cities are gaining momentum.

### Recommendations

**City 1: Pune**
1. Average rent per customer is very low.
2. Highest total revenue.
3. Average sales per customer is also high.

**City 2: Delhi**
1. Highest estimated coffee consumers at 7.7 million.
2. Highest total number of customers, which is 68.
3. Average rent per customer is 330 (still under 500).

**City 3: Jaipur**
1. Highest number of customers, which is 69.
2. Average rent per customer is very low at 156.
3. Average sales per customer is better at 11.6k.

This analysis provides a data-backed view of where Monday Coffee should open new locations and can help inform expansion strategy and decision-making.

## How to Run

1. Create a PostgreSQL database.
2. Run `Schemas.sql` to create the tables.
3. Import the CSV files in this order: `city`, `products`, `customers`, `sales`.
4. Run the queries in `Solutions.sql`.

## Author - [Your Name]

This project is part of my portfolio, showcasing the SQL skills essential for data analyst roles. If you have any questions, feedback, or would like to collaborate, feel free to get in touch!

### Stay Updated and Join the Community

For more content on SQL, data analysis, and other data-related topics, make sure to follow me on social media:

- **YouTube**: [Add your channel link]
- **Instagram**: [Add your profile link]
- **LinkedIn**: [Add your profile link]
- **GitHub**: [Add your profile link]

Thank you for your support, and I look forward to connecting with you!
