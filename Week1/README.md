Danny wants to use data to answer questions about his customers, especially about their visiting patterns, how much money they've spent and also which menu items are their favourite. 
3 key datasets:
- sales
- menu
- members

ERD
* Insert ERD Diagram
1) What is the total amount each customer spent at the restaurant?
SELECT sales.customer_id, SUM(menu.price) AS total_amount
FROM sales 
JOIN menu ON sales.product_id = menu.product_id
GROUP BY customer_id
ORDER BY customer_id;
Resulting table:
customer_id, total_amount
A 76
B 74
C 36

2) How many days has each customer visited the restaurant?
SELECT customer_id, COUNT(DISTINCT order_date) AS total_visits
FROM sales
GROUP BY customer_id;
customer_id, total_visits
A 4
B 6
C 2

3) What was the first item from the menu purchased by each customer?
SELECT DISTINCT customer_id, product_name AS first_order
FROM (SELECT 
    sales.customer_id,
    menu.product_name,
    RANK() OVER (
        PARTITION BY sales.customer_id 
        ORDER BY sales.order_date
    ) AS rn
FROM sales 
JOIN menu  
ON sales.product_id = menu.product_id)x
WHERE rn = 1;
customer_id first_order
A curry
A sushi
B curry
C ramen

4) What is the most purchased item on the menu and how many times was it purchased by all customers?
SELECT menu.product_name, count(sales.product_id) AS most_purchased
FROM sales
JOIN menu ON sales.product_id = menu.product_id
GROUP BY menu.product_name
ORDER BY most_purchased DESC
LIMIT 1;
product_name most_purchased
ramen 8

5) Which item was the most popular for each customer?
WITH popular AS(
  SELECT
    sales.customer_id,
    menu.product_name,
    COUNT(*) AS order_count,
    RANK () OVER (
      PARTITION BY sales.customer_id
      ORDER BY COUNT(*) DESC
    ) AS rank
  FROM sales
  JOIN menu ON sales.product_id = menu.product_id
  GROUP BY sales.customer_id, menu.product_name
)

SELECT customer_id, product_name, order_count
FROM popular
WHERE rank = 1;
customer_id, product_name, order_count
A ramen 3
B ramen 2
B curry 2
B sushi 2
C ramen 3

6) Which item was purchased first by the customer after they became a member?
WITH member_first_order AS (
SELECT
members.customer_id, 
sales.product_id,
sales.order_date,
members.join_date,
ROW_NUMBER() OVER (
PARTITION BY members.customer_id
ORDER BY sales.order_date) AS row_num
FROM members
JOIN sales
ON members.customer_id = sales.customer_id
AND sales.order_date > members.join_date
)

SELECT customer_id, product_name
FROM member_first_order
JOIN menu ON member_first_order.product_id = menu.product_id
WHERE row_num =1
ORDER BY customer_id;

customer_id, product_name
A ramen
B sushi

7) Which item was purchased just before the customer became a member?
WITH member_first_order AS (
SELECT
members.customer_id, 
sales.product_id,
sales.order_date,
members.join_date,
ROW_NUMBER() OVER (
PARTITION BY members.customer_id
ORDER BY sales.order_date DESC) AS row_num
FROM members
JOIN sales
ON members.customer_id = sales.customer_id
AND sales.order_date < members.join_date
)

SELECT customer_id, product_name
FROM member_first_order
JOIN menu ON member_first_order.product_id = menu.product_id
WHERE row_num =1
ORDER BY customer_id;
customer_id, product_name
A sushi
B sushi

8) What is the total items and amount spent for each member before they became a member?
SELECT sales.customer_id, COUNT(sales.product_id) AS total_items, SUM(menu.price) AS total_sales
FROM sales
JOIN members
ON sales.customer_id = members.customer_id AND sales.order_date < members.join_date
JOIN menu
ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
customer_id, total_items, total_sales
A 2 25
B 3 40

9) If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have? 



