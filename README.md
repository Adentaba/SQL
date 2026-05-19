/*
Project: Customer Sales Analysis (Sample)
Author: Kexin Zhao
Purpose:
This SQL script analyzes customer sales performance using order, product,
and payment data. It demonstrates joins, CTEs, aggregations, window functions,
ranking, and customer segmentation.

Database style: SQL Server / T-SQL
*/

WITH order_details AS (
    SELECT
        o.order_id,
        o.customer_id,
        c.customer_name,
        c.state,
        c.city,
        o.order_date,
        p.product_id,
        p.product_name,
        p.category,
        oi.quantity,
        oi.unit_price,
        oi.discount_amount,
        pay.payment_status,
        (oi.quantity * oi.unit_price) - ISNULL(oi.discount_amount, 0) AS net_sales
    FROM orders o
    INNER JOIN customers c
        ON o.customer_id = c.customer_id
    INNER JOIN order_items oi
        ON o.order_id = oi.order_id
    INNER JOIN products p
        ON oi.product_id = p.product_id
    LEFT JOIN payments pay
        ON o.order_id = pay.order_id
    WHERE
        o.order_status = 'Completed'
        AND pay.payment_status = 'Paid'
),

customer_summary AS (
    SELECT
        customer_id,
        customer_name,
        state,
        city,
        COUNT(DISTINCT order_id) AS total_orders,
        SUM(quantity) AS total_units_sold,
        SUM(net_sales) AS total_sales,
        AVG(net_sales) AS avg_order_line_value,
        MIN(order_date) AS first_order_date,
        MAX(order_date) AS last_order_date
    FROM order_details
    GROUP BY
        customer_id,
        customer_name,
        state,
        city
),

customer_ranking AS (
    SELECT
        *,
        RANK() OVER (ORDER BY total_sales DESC) AS sales_rank,
        NTILE(4) OVER (ORDER BY total_sales DESC) AS customer_quartile
    FROM customer_summary
),

category_summary AS (
    SELECT
        customer_id,
        category,
        SUM(net_sales) AS category_sales,
        RANK() OVER (
            PARTITION BY customer_id
            ORDER BY SUM(net_sales) DESC
        ) AS category_rank
    FROM order_details
    GROUP BY
        customer_id,
        category
),

favorite_category AS (
    SELECT
        customer_id,
        category AS top_category,
        category_sales
    FROM category_summary
    WHERE category_rank = 1
),

monthly_sales AS (
    SELECT
        customer_id,
        YEAR(order_date) AS sales_year,
        MONTH(order_date) AS sales_month,
        SUM(net_sales) AS monthly_sales
    FROM order_details
    GROUP BY
        customer_id,
        YEAR(order_date),
        MONTH(order_date)
),

customer_trend AS (
    SELECT
        customer_id,
        sales_year,
        sales_month,
        monthly_sales,
        LAG(monthly_sales) OVER (
            PARTITION BY customer_id
            ORDER BY sales_year, sales_month
        ) AS previous_month_sales
    FROM monthly_sales
)

SELECT
    cr.customer_id,
    cr.customer_name,
    cr.state,
    cr.city,
    cr.total_orders,
    cr.total_units_sold,
    cr.total_sales,
    cr.avg_order_line_value,
    cr.first_order_date,
    cr.last_order_date,
    fc.top_category,
    fc.category_sales AS top_category_sales,
    cr.sales_rank,

    CASE
        WHEN cr.customer_quartile = 1 THEN 'Top Customer'
        WHEN cr.customer_quartile = 2 THEN 'High Value'
        WHEN cr.customer_quartile = 3 THEN 'Medium Value'
        ELSE 'Low Value'
    END AS customer_segment,

    CASE
        WHEN cr.total_orders >= 10 AND cr.total_sales >= 5000 THEN 'VIP'
        WHEN cr.total_orders >= 5 THEN 'Repeat Customer'
        ELSE 'New or Occasional Customer'
    END AS customer_type

FROM customer_ranking cr
LEFT JOIN favorite_category fc
    ON cr.customer_id = fc.customer_id
ORDER BY
    cr.total_sales DESC;
