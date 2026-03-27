# 🍽️ Danny’s Diner SQL Case Study

Danny wants to use data to better understand his customers, including their visiting patterns, spending behavior, and favorite menu items.

## Datasets
- `sales`
- `menu`
- `members`

## ERD
*Insert ERD Diagram Here*

---

## 1) What is the total amount each customer spent at the restaurant?

```sql
SELECT 
  sales.customer_id, 
  SUM(menu.price) AS total_amount
FROM sales 
JOIN menu 
  ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```

| customer_id | total_amount |
|-------------|--------------|
| A           | 76           |
| B           | 74           |
| C           | 36           |

## 2) How many days has each customer visited the restaurant?
```sql
SELECT 
  customer_id, 
  COUNT(DISTINCT order_date) AS total_visits
FROM sales
GROUP BY customer_id;
```

| customer_id |total_visits|
|------------|-------------|
| A          | 4           |
| B          | 6           |
| C          | 2           |

## 3) What was the first item from the menu purchased by each customer?

```sql
SELECT DISTINCT
  customer_id, 
  product_name AS first_order
FROM (
  SELECT 
    sales.customer_id,
    menu.product_name,
    RANK() OVER (
      PARTITION BY sales.customer_id 
      ORDER BY sales.order_date
    ) AS rn
  FROM sales 
  JOIN menu  
    ON sales.product_id = menu.product_id
) x
WHERE rn = 1;
```
| customer_id | first_order |
|-------------|--------------|
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

## 4) What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT 
  menu.product_name, 
  COUNT(sales.product_id) AS most_purchased
FROM sales
JOIN menu 
  ON sales.product_id = menu.product_id
GROUP BY menu.product_name
ORDER BY most_purchased DESC
LIMIT 1;
```
product_name most_purchased
ramen 8
| product_name |most_purchased|
|------------|-------------|
| ramen          | 8           |

## 5) Which item was the most popular for each customer?
```sql
WITH popular AS (
  SELECT
    sales.customer_id,
    menu.product_name,
    COUNT(*) AS order_count,
    RANK() OVER (
      PARTITION BY sales.customer_id
      ORDER BY COUNT(*) DESC
    ) AS rank
  FROM sales
  JOIN menu 
    ON sales.product_id = menu.product_id
  GROUP BY 
    sales.customer_id, 
    menu.product_name
)

SELECT 
  customer_id, 
  product_name, 
  order_count
FROM popular
WHERE rank = 1;
```

| customer_id | product_name |order_count|
|-------------|--------------|--------------|
| A           | curry        |       3      |
| B           | sushi        |       2      |
| B           | curry        |       2      |
| B           | ramen        |       2      |
| C           | ramen        |       3      |

## 6) Which item was purchased first by the customer after they became a member?
```sql
WITH member_first_order AS (
  SELECT
    members.customer_id, 
    sales.product_id,
    sales.order_date,
    members.join_date,
    ROW_NUMBER() OVER (
      PARTITION BY members.customer_id
      ORDER BY sales.order_date
    ) AS row_num
  FROM members
  JOIN sales
    ON members.customer_id = sales.customer_id
   AND sales.order_date > members.join_date
)

SELECT 
  customer_id, 
  product_name
FROM member_first_order
JOIN menu 
  ON member_first_order.product_id = menu.product_id
WHERE row_num = 1
ORDER BY customer_id;
```

| customer_id | product_name |
|-------------|--------------|
| A           | ramen        |
| B           | sushi        |

## 7) Which item was purchased just before the customer became a member?
```sql
WITH member_first_order AS (
  SELECT
    members.customer_id, 
    sales.product_id,
    sales.order_date,
    members.join_date,
    ROW_NUMBER() OVER (
      PARTITION BY members.customer_id
      ORDER BY sales.order_date DESC
    ) AS row_num
  FROM members
  JOIN sales
    ON members.customer_id = sales.customer_id
   AND sales.order_date < members.join_date
)

SELECT 
  customer_id, 
  product_name
FROM member_first_order
JOIN menu 
  ON member_first_order.product_id = menu.product_id
WHERE row_num = 1
ORDER BY customer_id;
```

| customer_id | product_name |
|-------------|--------------|
| A           | sushi        |
| B           | sushi        |

## 8) What is the total items and amount spent for each member before they became a member?
```sql
SELECT 
  sales.customer_id, 
  COUNT(sales.product_id) AS total_items, 
  SUM(menu.price) AS total_sales
FROM sales
JOIN members
  ON sales.customer_id = members.customer_id 
 AND sales.order_date < members.join_date
JOIN menu
  ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```
customer_id, total_items, total_sales
A 2 25
B 3 40
| customer_id | total_items  | total_sales|
|-------------|--------------|------------|
| A           | 2            | 25         |    
| B           | 3            | 40         |   

## 9) If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
WITH points AS (
  SELECT 
    menu.product_id,
    CASE
      WHEN product_id = 1 THEN price * 20
      ELSE price * 10
    END AS points
  FROM menu
)

SELECT 
  sales.customer_id, 
  SUM(points.points) AS total_points
FROM points
JOIN sales 
  ON sales.product_id = points.product_id
GROUP BY sales.customer_id
ORDER BY total_points DESC;
```

| customer_id | total_points |
|-------------|--------------|
| B           | 940          |
| A           | 860          |
| C           | 360          |

## 10) In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH dates AS (
  SELECT 
    customer_id,
    join_date, 
    join_date + 6 AS valid_date, 
    '2021-01-31'::date AS last_date
  FROM members
)

SELECT 
  sales.customer_id, 
  SUM(
    CASE
      WHEN menu.product_name = 'sushi' THEN 2 * 10 * menu.price
      WHEN sales.order_date BETWEEN dates.join_date AND dates.valid_date THEN 2 * 10 * menu.price
      ELSE 10 * menu.price 
    END
  ) AS points
FROM sales
JOIN dates 
  ON sales.customer_id = dates.customer_id
 AND sales.order_date >= dates.join_date
 AND sales.order_date <= dates.last_date
JOIN menu 
  ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY points DESC;
```

| customer_id | points |
|-------------|--------------|
| A           | 1020        |
| B           | 320        |





