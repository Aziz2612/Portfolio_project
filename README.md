-- Most active retailers by number and value of orders 
SELECT r.id,r.name,r.retailer_type,COUNT(o.id) AS orders_count,SUM(o.total_initial_gross) AS orders_value
FROM retailers r
JOIN orders o
ON r.id = o.retailer_id
WHERE r.retailer_type NOT IN (7,3,32)
GROUP BY r.id,r.name,r.retailer_type
ORDER BY orders_value DESC;

-- AVG order frequency per Active retailer per month
WITH monthly_orders AS (
SELECT 
r.id AS retailer_id,
r.name AS retailer_name,
r.retailer_type, 
date_trunc('month',o.created_at) AS order_month, 
COUNT(*) AS monthly_order_count
FROM retailers r 
JOIN orders o
ON r.id = o.retailer_id
WHERE o.status = 7
AND date_trunc('month',o.created_at) >= '2025-01-01'
GROUP BY r.id,r.name,r.retailer_type, date_trunc('month',o.created_at)
),
active_retailers AS (
SELECT retailer_id
FROM monthly_orders
GROUP BY retailer_id
HAVING COUNT(DISTINCT order_month) = 7 -- Must have orders in all 7 months
)
SELECT 
m.retailer_id,
m.retailer_name,
m.retailer_type, 
ROUND(AVG(monthly_order_count),0) AS avg_monthly_orders
FROM monthly_orders m
JOIN active_retailers a ON m.retailer_id = a.retailer_id
GROUP BY m.retailer_id,m.retailer_name,m.retailer_type
ORDER BY avg_monthly_orders DESC;

-- Retention rate of retailers
  WITH retailer_months AS (
  SELECT date_trunc('month',o.created_at) AS order_month , o.retailer_id 
  FROM orders o 
  WHERE o.status = 7 -- DELIVERED orders only
  AND date_trunc('month',o.created_at) > '2024-11-30T00:00:00' -- this year
  ), 
  retention_rate AS (
  SELECT prv.retailer_id, prv.order_month AS previous_month, crnt.order_month AS current_month
  FROM retailer_months prv
  LEFT JOIN retailer_months crnt
  ON prv.retailer_id = crnt.retailer_id
  AND crnt.order_month = prv.order_month + INTERVAL '1 month'
  )
  SELECT previous_month, 
    COUNT(DISTINCT retailer_id) AS total_active_retailers, 
    COUNT (DISTINCT CASE WHEN current_month IS NOT NULL THEN retailer_id END) AS retained_retailers,
    ROUND(COUNT(DISTINCT CASE WHEN current_month IS NOT NULL THEN retailer_id END)::decimal / NULLIF(COUNT(DISTINCT retailer_id),0) * 100,2) AS retention_rate_percent
  FROM retention_rate
  GROUP BY previous_month
  ORDER BY previous_month;
  
  -- Churn rate of retailer 
    WITH retailer_months AS (
  SELECT date_trunc('month',o.created_at) AS order_month , o.retailer_id 
  FROM orders o 
  WHERE o.status = 7 -- DELIVERED orders only
  AND date_trunc('month',o.created_at) > '2024-11-30T00:00:00' -- this year
  ), 
  churn_rate AS (
  SELECT prv.retailer_id, prv.order_month AS previous_month, crnt.order_month AS current_month
  FROM retailer_months prv
  LEFT JOIN retailer_months crnt
  ON prv.retailer_id = crnt.retailer_id
  AND crnt.order_month = prv.order_month + INTERVAL '1 month'
  )
  SELECT previous_month, 
    COUNT(DISTINCT retailer_id) AS total_active_retailers, 
    COUNT (DISTINCT CASE WHEN current_month IS NULL THEN retailer_id END) AS churned_retailers,
    ROUND(COUNT(DISTINCT CASE WHEN current_month IS NULL THEN retailer_id END)::decimal / NULLIF(COUNT(DISTINCT retailer_id),0) * 100,2) AS churned_rate_percent
  FROM churn_rate
  GROUP BY previous_month
  ORDER BY previous_month;
  
  -- How many retailers use BNPL service VS How many not?
SELECT 
  COUNT(*) FILTER (WHERE 'Instalments' = ANY(allowed_payment_methods)) AS BNPL_retailers,
  COUNT(*) FILTER (WHERE NOT 'Instalments' = ANY(allowed_payment_methods)) AS Non_BNPL_retailers
FROM retailers;
 
 -- Top suppliers by delivered GMV and number of orders this year
 SELECT o.supplier_id, s.name, SUM(o.total_initial_gross) AS total_GMV, COUNT(DISTINCT o.id) AS orders_count, ROUND(SUM(o.total_initial_gross)/COUNT(DISTINCT o.id),2) AS AVG_order_value
 FROM orders o
 JOIN suppliers s
 ON o.supplier_id = s.id 
 WHERE o.status = 7
 AND DATE_TRUNC('month',o.created_at) > '2024-12-31T00:00:00'
 GROUP BY o.supplier_id, s.name
 ORDER BY total_GMV DESC;
 
  -- Top areas by delivered GMV this year
WITH Top_cities AS (SELECT 
c.id, c.name, a.id,a.name, SUM(o.total_initial_gross) AS Total_sold_GMV,
RANK() OVER (partition by c.id order by SUM(o.total_initial_gross) DESC) AS rank
FROM orders o 
JOIN retailer_branches rb 
ON o.retailer_branch_id = rb.id 
AND o.created_at >= DATE_TRUNC ('year', o.created_at)
AND o.status = 7
and (o.flags->'enterprise_order') is null
JOIN areas a 
ON rb.area_id = a.id 
JOIN cities c
ON a.city_id = c.id
GROUP BY c.id, c.name, a.id,a.name
ORDER BY Total_sold_GMV DESC)
SELECT *
FROM top_cities 
WHERE rank = 1;

-- Top sold products 
SELECT 
p.id,bp.name, ROUND(sum(COALESCE((od.price_gross * od.amount),0)),2) AS total_sold_GMV
FROM order_details od 
JOIN products p 
            ON od.product_id = p.id
AND od.cancelled = 'false'
JOIN base_products bp 
ON p.base_product_id = bp.id
JOIN orders o 
ON od.order_id = o.id 
AND o.status = 7
and (o.flags->'enterprise_order') is null
AND o.created_at >= DATE_TRUNC ('year', current_date)
GROUP BY 1,2
ORDER BY total_sold_GMV DESC;

-- Top profitable products 
SELECT 
p.id,bp.name, ROUND(sum(COALESCE((od.extra_Details->>'back_margin_amount')::numeric, 0)),2) as back_margin,ROUND(sum(COALESCE((od.extra_Details->>'front_margin_amount')::numeric, 0)),2) as front_margin, 
(ROUND(sum(COALESCE((od.extra_Details->>'back_margin_amount')::numeric, 0)),2) + ROUND(sum(COALESCE((od.extra_Details->>'front_margin_amount')::numeric, 0)),2)) AS total_GMV
FROM order_details od 
JOIN products p 
ON od.product_id = p.id
AND od.cancelled = 'false'
JOIN base_products bp 
ON p.base_product_id = bp.id
JOIN orders o 
ON od.order_id = o.id 
AND o.status = 7
and (o.flags->'enterprise_order') is null
AND o.created_at >= DATE_TRUNC ('year', current_date)
GROUP BY 1,2
ORDER BY total_GMV DESC;
