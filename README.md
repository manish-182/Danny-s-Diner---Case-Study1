### Dannys_Diner-Case Study
Danny’s Diner is a little restaurant that sells 3 favourite foods: sushi, curry, and ramen. In the analysis, the queries are developed to understand the customer preference and their buying patterns.

This challenge and data set is provided by <a href="https://www.linkedin.com/in/datawithdanny/?originalSubdomain=au/"> Danny Ma </a>.
<p>Want to know more about 8 Week SQL Challenge case study? <a href="https://8weeksqlchallenge.com/case-study-1/"> Click Here </a></p>

----------------------


#### 1. What is the total amount each customer spent at the restaurant?

```sql
	SELECT 
	  customer_id, 
	  SUM(price) AS tot_amt
	FROM 
	  sales AS s 
	  INNER JOIN menu AS m ON s.product_id = m.product_id
	GROUP BY 
	  customer_id;
```

#### Result:

```markdown
	| customer_id | tot_amt |
	|-------------|---------|
	| A           | 76      |
	| B           | 74      |
	| C           | 36      |
```

----------------------
#### 2. How many days has each customer visited the restaurant?

```sql
	SELECT 
	  customer_id, 
	  count (distinct order_date) as no_of_visit
	FROM 
	  sales as s 
	GROUP BY
	  customer_id;
```

#### Result:

```markdown
	| customer_id | no_of_visit |
	|-------------|-------------|
	| A           | 4           |
	| B           | 6           |
	| C           | 2           |
```

----------------------
#### 3. What was the first item from the menu purchased by each customer?
	
	-- Created new table using row_number function to serially list the items ordered by each customer group by date

```sql
	WITH items_purchase AS (
	  SELECT 
	    *, 
	    ROW_NUMBER() over(
	      PARTITION BY customer_id 
	      ORDER BY order_date) AS item_rank 
	  FROM 
	    sales AS s
	)
```
	-- using above table (CTE) to filter 1st order by each customer
	
```sql	
	SELECT 
	  customer_id, 
	  product_name 
	FROM items_purchase AS ip 
	  INNER JOIN menu AS m 
	    ON ip.product_id = m.product_id 
	WHERE 
	  item_rank = '1'
```

#### Result:

```markdown
	| customer_id | product_name |
	|-------------|--------------|
	| A           | sushi        |
	| B           | curry        |
```

----------------------
#### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
	
	--Created ta table to list the product name and  sum of no of orders. 

```sql	
	WITH items_purchased AS (
	  SELECT 
	    product_name, 
	    COUNT(*) AS no_of_orders 
	  FROM 
	    sales AS s 
	    INNER JOIN menu AS m ON s.product_id = m.product_id 
	  GROUP BY 
	    product_name
	)
```
	-- Filtering max order from the table created

```sql	
	SELECT product_name, no_of_orders 
	FROM
	  items_purchased 
	WHERE 
	  no_of_orders = (SELECT MAX(no_of_orders) FROM items_purchased)
```

#### Result:

```markdown
	| product_name | no_of_orders |
	|--------------|--------------|
	| ramen        | 8            |
```


----------------------
#### 5. Which item was the most popular for each customer?
	
	-- Count the items and group by customer and product
	

```sql
	WITH cpn AS (
		SELECT customer_id, product_name, COUNT(s.product_id) AS no_of_ord
		FROM sales AS s
		INNER JOIN menu AS m
		ON s.product_id = m.product_id
		GROUP BY customer_id, product_name
		)
```

	-- create subquery selecting max value and match cutomer to the main query to bring all customer results
```sql
	SELECT customer_id, product_name, no_of_ord
	FROM cpn c
	WHERE c.no_of_ord = (SELECT MAX(no_of_ord) FROM cpn WHERE customer_id = c.customer_id);
```

#### Result:

```markdown
	| customer_id | product_name | no_of_ord |
	|-------------|--------------|-----------|
	| C           | ramen        | 3         |
	| B           | sushi        | 2         |
	| B           | curry        | 2         |
	| B           | ramen        | 2         |
	| A           | ramen        | 3         |
```

----------------------
#### 6. Which item was purchased first by the customer after they became a member?
	
	--created table joining sales and members table and filtered for transactions on and after the date of membership, 
	--this is then ranked by using window function by customer serially.
	
	
```sql	
	WITH member_sales AS (
		SELECT s.customer_id, order_date, join_date, product_name,
		ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY order_date) AS row_num
		FROM sales AS s
		INNER JOIN members AS m1
		ON s.customer_id = m1.customer_id
		INNER JOIN menu AS m
		ON s.product_id = m.product_id
		WHERE order_date >= join_date
		)
```
	-- the 1st purchase will have the rank 1 and is filtered on second query.

```sql
	SELECT customer_id, product_name
	FROM member_sales
	WHERE row_num = '1';
```

#### Result:

```markdown
	| A | curry |
	|---|-------|
	| B | sushi |
```

----------------------
#### 7. Which item was purchased just before the customer became a member?
	
	--created table joining sales and members table and filtered for transactions before the date of membership, 
	--this is then ranked by using window function by customer and order date serially.

```sql
	WITH nmp AS (
		SELECT s.customer_id, order_date, join_date, product_name,
		ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY order_date) AS item_rank
		FROM sales AS s
		INNER JOIN members AS m1
		ON s.customer_id = m1.customer_id
		INNER JOIN menu AS m
		ON s.product_id = m.product_id
		WHERE order_date < join_date
			)
```
	-- sub query to create table showing max rank which is then joined to find the last item purchased. 
	

```sql
	SELECT nmp.customer_id, product_name
	FROM nmp
	INNER JOIN (
		SELECT customer_id, MAX(item_rank) AS max_rank
		FROM nmp
		GROUP BY customer_id
		) maxnmp
	ON nmp.customer_id = maxnmp.customer_id
	AND item_rank = max_rank;
```

#### Result:  

```markdown
	| A | curry |
	|---|-------|
	| B | sushi |
```

----------------------
#### 8. What is the total items and amount spent for each member before they became a member?

	--Table with items purchased and amount spent before they become member. 
	
```sql
	WITH nmp AS (
		SELECT s.customer_id, order_date, join_date, product_name, price
		FROM sales AS s
		INNER JOIN members AS m1
		ON s.customer_id = m1.customer_id
		INNER JOIN menu AS m
		ON s.product_id = m.product_id
		WHERE order_date < join_date
		)
```
	--Used aggregation to find total items purchased and total amount spent by each customer who became member. 

```sql
	SELECT customer_id, count(product_name) tot_items, sum(price) tot_spend
	FROM nmp
	GROUP BY customer_id;
```
#### Result:  

```markdown
	| customer_id | tot_items | tot_spend |
	|-------------|-----------|-----------|
	| A           | 2         | 25        |
	| B           | 3         | 40        |
```

----------------------
#### 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
	
	-- Created new points table using case logic to dictate points for each product type based on policy.

```sql
	WITH points AS (
		SELECT * , 
		CASE WHEN product_name = 'sushi' 
		THEN price * 20 
		ELSE price * 10
		END AS points
		FROM menu
		)
```
	
	--Points table is joined with sales table to find total points for each customer. 
```sql
	SELECT customer_id, SUM(points) AS tot_points
	FROM sales AS s
	INNER JOIN points AS p
	ON s.product_id = p.product_id
	GROUP BY customer_id;
```

#### Result:  

```markdown
	| customer_id | tot_points |
	|-------------|------------|
	| A           | 860        |
	| B           | 940        |
	| C           | 360        |
```

----------------------
#### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?


	-- Created new points table using case logic to dictate points for each product type based on policy.
	
* For items purchased within a week after being a member AND if purchased within January = **20 pts**
* For purchase of 'Sushi' after being member = **20 pts**
* For purchase of other items ~~'Sushi'~~  after being member = **10 pts** 
* For purchase made before being member = **0 pts**

```	
-- calculated reward points and aggregated customerwise.
```

```sql
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
	GROUP BY s.customer_id;
```

#### Result:  

```markdown
	| customer_id | points |
	|-------------|--------|
	| A           | 1020   |
	| B           | 440    |
```


----------------------
#### 11. Join all the details to present the comprehensive view to the management.

```sql
	SELECT s.customer_id, order_date, product_name, price,
		CASE WHEN order_date >= join_date AND s.customer_id = m2.customer_id 
			THEN 'Y'
			ELSE 'N'
		END AS member
	FROM sales AS s
	INNER JOIN menu AS m
	ON s.product_id = m.product_id
	FULL JOIN members AS m2
	ON s.customer_id = m2.customer_id;
```

#### Result:  

```markdown
	| customer_id | order_date | product_name | price | member |
	|-------------|------------|--------------|-------|--------|
	| A           | 2021-01-01 | sushi        | 10    | N      |
	| A           | 2021-01-01 | curry        | 15    | N      |
	| A           | 2021-01-07 | curry        | 15    | Y      |
	| A           | 2021-01-10 | ramen        | 12    | Y      |
	| A           | 2021-01-11 | ramen        | 12    | Y      |
	| A           | 2021-01-11 | ramen        | 12    | Y      |
	| B           | 2021-01-01 | curry        | 15    | N      |
	| B           | 2021-01-02 | curry        | 15    | N      |
	| B           | 2021-01-04 | sushi        | 10    | N      |
	| B           | 2021-01-11 | sushi        | 10    | Y      |
	| B           | 2021-01-16 | ramen        | 12    | Y      |
	| B           | 2021-02-01 | ramen        | 12    | Y      |
	| C           | 2021-01-01 | ramen        | 12    | N      |
	| C           | 2021-01-01 | ramen        | 12    | N      |
	| C           | 2021-01-07 | ramen        | 12    | N      |
```

----------------------
#### 12. Ranking of customer products for members only
	
	--Created Table showing all purchases seperating member and non member purchases.

```sql
	WITH overall_report AS (
		SELECT s.customer_id, order_date, product_name, price,
			CASE WHEN order_date >= join_date AND s.customer_id = m2.customer_id 
				THEN 'Y'
				ELSE 'N'
			END AS member
		FROM sales AS s
		INNER JOIN menu AS m
		ON s.product_id = m.product_id
		FULL JOIN members AS m2
		ON s.customer_id = m2.customer_id
		)
```
	--Ranking of purchases made by the members only. 

```sql
	SELECT *,
		CASE WHEN member = 'Y' 
		THEN 
			RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date)
		ELSE null
		END AS ranking
	FROM overall_report;
```

#### Result:  

```markdown
	| customer_id | order_date | product_name | price | member | ranking |
	|-------------|------------|--------------|-------|--------|---------|
	| A           | 2021-01-01 | sushi        | 10    | N      | NULL    |
	| A           | 2021-01-01 | curry        | 15    | N      | NULL    |
	| A           | 2021-01-07 | curry        | 15    | Y      | 1       |
	| A           | 2021-01-10 | ramen        | 12    | Y      | 2       |
	| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
	| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
	| B           | 2021-01-01 | curry        | 15    | N      | NULL    |
	| B           | 2021-01-02 | curry        | 15    | N      | NULL    |
	| B           | 2021-01-04 | sushi        | 10    | N      | NULL    |
	| B           | 2021-01-11 | sushi        | 10    | Y      | 1       |
	| B           | 2021-01-16 | ramen        | 12    | Y      | 2       |
	| B           | 2021-02-01 | ramen        | 12    | Y      | 3       |
	| C           | 2021-01-01 | ramen        | 12    | N      | NULL    |
	| C           | 2021-01-01 | ramen        | 12    | N      | NULL    |
	| C           | 2021-01-07 | ramen        | 12    | N      | NULL    |
```



