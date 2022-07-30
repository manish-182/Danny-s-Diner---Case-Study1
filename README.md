# Dannys_Diner-Case Study


-- This is a solution to the Case Study #1 - Danny's Diner (8 Week SQL Challenge)

-- 1. What is the total amount each customer spent at the restaurant?


SELECT 
  customer_id, 
  SUM(price) as tot_amt
FROM 
  sales AS s 
  INNER JOIN menu AS m on s.product_id = m.product_id
GROUP BY 
  customer_id;


-- 2. How many days has each customer visited the restaurant?

SELECT 
  customer_id, 
  count (distinct order_date) as no_of_visit
FROM 
  sales as s 
GROUP BY
  customer_id;


-- 3. What was the first item from the menu purchased by each customer?
	
	-- using row_number function to serially list the items ordered by each customer group by date
WITH cp AS (
  SELECT 
    *, 
    ROW_NUMBER() over(
      PARTITION BY customer_id 
      ORDER BY order_date) AS item_no 
  FROM 
    sales AS s
)
	-- using above table to filter 1st order by each customer
SELECT 
  customer_id, 
  product_name 
FROM cp 
  INNER JOIN menu AS m 
    ON cp.product_id = m.product_id 
WHERE 
  item_no = '1'


-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
	
	--Created ta table to list the product name and  sum of no of orders. 
WITH product_orders AS (
  SELECT 
    product_name, 
    count(*) AS no_of_orders 
  FROM 
    sales AS s 
    INNER JOIN menu AS m ON s.product_id = m.product_id 
  GROUP BY 
    product_name
)

			-- Filtering max order from the table created
SELECT product_name, no_of_orders 
FROM
  product_orders 
WHERE 
  no_of_orders = (SELECT MAX(no_of_orders) FROM product_orders)



-- 5. Which item was the most popular for each customer?
	
	-- Count the items and group by customer and product
	-- create subquery selecting max value and match cutomer to the main query to bring all customer results

WITH cpn AS (
	SELECT customer_id, product_name, COUNT(s.product_id) AS no_of_ord
	FROM sales as s
	INNER JOIN menu as m
	ON s.product_id = m.product_id
	GROUP BY customer_id, product_name
	)
SELECT customer_id, product_name, no_of_ord
FROM cpn c
WHERE c.no_of_ord = (SELECT MAX(no_of_ord) FROM cpn WHERE customer_id = c.customer_id);


-- 6. Which item was purchased first by the customer after they became a member?
	
	--created table joining sales and members table and filtered for transactions on and after the date of membership, 
	--this is then ranked by using window function by customer serially
	-- the 1st purchase will have the rank 1 and is filtered on second query
WITH mp AS (
	SELECT s.customer_id, order_date, join_date, product_name,
	ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY order_date) AS row_num
	FROM sales AS s
	INNER JOIN members AS m1
	ON s.customer_id = m1.customer_id
	INNER JOIN menu AS m
	ON s.product_id = m.product_id
	WHERE order_date >= join_date
	)

SELECT customer_id, product_name
FROM mp
WHERE row_num = '1'


-- 7. Which item was purchased just before the customer became a member?
	
	--created table joining sales and members table and filtered for transactions before the date of membership, 
	--this is then ranked by using window function by customer serially


WITH nmp AS (
	SELECT s.customer_id, order_date, join_date, product_name,
	ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY order_date) AS row_num
	FROM sales AS s
	INNER JOIN members AS m1
	ON s.customer_id = m1.customer_id
	INNER JOIN menu AS m
	ON s.product_id = m.product_id
	WHERE order_date < join_date
	)

	-- sub query to create table showing max rank which is the last purchase by the customer

SELECT nmp.customer_id, product_name
FROM nmp
INNER JOIN (
	SELECT customer_id, MAX(row_num) AS max_num
	FROM nmp
	GROUP BY customer_id
	) maxnmp
ON nmp.customer_id = maxnmp.customer_id
AND row_num = max_num

-- 8. What is the total items and amount spent for each member before they became a member?

WITH nmp AS (
	SELECT s.customer_id, order_date, join_date, product_name, price
	FROM sales AS s
	INNER JOIN members AS m1
	ON s.customer_id = m1.customer_id
	INNER JOIN menu AS m
	ON s.product_id = m.product_id
	WHERE order_date < join_date
	)

SELECT customer_id, count(product_name) tot_items, sum(price) tot_spend
FROM nmp
GROUP BY customer_id


-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
	-- used case logic to dictate points for each product type based on policy

WITH points AS (
	SELECT * , 
	CASE WHEN product_name = 'sushi' 
	THEN price * 20 
	ELSE price * 10
	END AS points
	FROM menu
	)

SELECT customer_id, SUM(points) AS tot_points
FROM sales AS s
INNER JOIN points AS p
ON s.product_id = p.product_id
GROUP BY customer_id



-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
	--Used case logic to categorize points into 4 categories
	-- calculated reward points and aggregated customerwise

SELECT 
	s.customer_id, 
	SUM (
		CASE WHEN order_date BETWEEN join_date AND DATEADD(DAY, 6 , join_date) AND order_date <= '2021-01-31' THEN  price * 10 *2 
		 WHEN order_date >= join_date AND product_name = 'sushi' THEN price * 10 * 2
		 WHEN order_date >= join_date AND product_name <> 'sushi' THEN price * 10
		 ELSE price * 0
		END ) AS points
FROM sales AS s
INNER JOIN members AS m2
ON s.customer_id = m2.customer_id
INNER JOIN menu AS m
ON s.product_id = m.product_id
GROUP BY s.customer_id
