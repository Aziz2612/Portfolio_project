-- 1. What is the total amount each customer spent at the restaurant?
SELECT customer_id,SUM(price) total_spent
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
GROUP BY customer_id
ORDER BY customer_id;

-- 2. How many days has each customer visited the restaurant?
SELECT customer_id, COUNT(DISTINCT order_date)
FROM sales
GROUP BY customer_id
ORDER BY customer_id;

-- 3. What was the first item from the menu purchased by each customer?

WITH First_product AS(
SELECT 
 s.customer_id, m.product_name,order_date,
 RANK () OVER (PARTITION BY customer_id ORDER BY order_date) AS rank
FROM sales s
JOIN menu m 
ON s.product_id = m.product_id
)
SELECT  
customer_id, product_name,order_date
FROM First_product 
WHERE rank = 1

-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
WITH total_sales AS (
SELECT 
s.product_id, m.product_name, COUNT(s.product_id) AS total_sales
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
GROUP BY s.product_id, m.product_name
  ),
 top_sold AS (
    SELECT
    product_id, product_name, total_sales,
    RANK () OVER (ORDER BY total_sales DESC) AS rank
    FROM total_sales
    )
    
    SELECT product_id, product_name, total_sales 
    FROM top_sold
    WHERE rank = 1

-- 5. Which item was the most popular for each customer?
WITH total_purchases AS (
  SELECT
s.customer_id, m.product_name, COUNT(s.product_id) AS count
FROM sales s
JOIN menu m 
ON s.product_id = m.product_id
GROUP BY customer_id, product_name
  ),
  top_product AS (
    SELECT 
    customer_id, product_name, count,
    RANK () OVER (PARTITION BY customer_id ORDER BY count DESC) AS rank
    FROM total_purchases
    )
    SELECT 
     customer_id, product_name
     FROM top_product
     WHERE rank = 1

-- 6. Which item was purchased first by the customer after they became a member?
WITH orders_after_joining_date AS (
  SELECT
s.customer_id, m.product_name, s.order_date, me.join_date
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
JOIN members me
ON s.customer_id = me.customer_id
  WHERE s.order_date >= me.join_date
  ),
 ranked_orders AS (
   SELECT *,
   RANK () OVER (PARTITION BY customer_id ORDER BY order_date ASC) AS rank
   FROM orders_after_joining_date
   )
  SELECT 
  customer_id, product_name, order_date
FROM ranked_orders
WHERE rank = 1

-- 7. Which item was purchased just before the customer became a member?
WITH orders_before_joining_date AS (
  SELECT
s.customer_id, m.product_name, s.order_date, me.join_date
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
JOIN members me
ON s.customer_id = me.customer_id
  WHERE s.order_date < me.join_date
  ),
 ranked_orders AS (
   SELECT *,
   RANK () OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rank
   FROM orders_before_joining_date
   )
  SELECT 
  customer_id, product_name, order_date
FROM ranked_orders
WHERE rank = 1

-- 8. What is the total items and amount spent for each member before they became a member?
  SELECT
s.customer_id, COUNT(s.product_id) AS tota_items_before_membership, SUM(m.price) AS amount_spent_before_membership
FROM sales s
JOIN menu m
ON s.product_id = m.product_id
JOIN members me
ON s.customer_id = me.customer_id
  WHERE s.order_date < me.join_date
  GROUP BY s.customer_id
  ORDER BY s.customer_id

-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
WITH product_points AS (
SELECT
product_id , product_name, price, 
CASE 
WHEN product_name = 'sushi' THEN (price * 20)
ELSE (price * 10) END AS points
FROM menu
  )
  SELECT s.customer_id, SUM (points) AS total_points
  FROM sales s
  JOIN product_points p
  ON s.product_id = p.product_id
  GROUP BY s.customer_id
  ORDER BY s.customer_id

-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

WITH all_sales AS (
  SELECT
  s.customer_id,
  s.order_date,
  m.product_name,
  m.price,
  me.join_date,
  CASE 
  		WHEN s.order_date < me.join_date THEN 
  			CASE WHEN m.product_name = 'sushi' THEN m.price*20 ELSE m.price*10 END
  		WHEN s.order_date BETWEEN me.join_date AND me.join_date + INTERVAL '6 days' THEN m.price*20
  		WHEN s.order_date < '2021-02-01' THEN 
  			CASE WHEN m.product_name = 'sushi' THEN m.price*20 ELSE m.price*10 END
  		ELSE 0 
  	END AS points
  FROM sales s
  JOIN menu m ON s.product_id = m.product_id
  JOIN members me ON s.customer_id = me.customer_id
  WHERE s.order_date < '2021-02-01'
  )
  SELECT 
  customer_id , SUM(points) AS total_points 
  FROM all_sales
  GROUP BY customer_id
  ORDER BY customer_id
  
