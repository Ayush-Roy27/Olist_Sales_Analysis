# Olist_Sales_Analysis

### Project Overview

This data analysis project aims to provide insights into the sales performance of a Brazilian ecommerce company named, Olist over the past ## months. By analyzing various aspects of the sales data, we seek to identify trends and gain a deeper understanding of the company's performance.

### Data Sources

The datasets used for this analysis are as follows:

- olist_customers_dataset.csv
- olist_geolocation_dataset.csv
- olist_order_items_dataset.csv
- olist_order_payments_dataset.csv
- olist_order_reviews_data.csvset
- olist_orders_dataset.csv
- olist_products_dataset.csv
- olist_sellers_dataset.csv
- product_category_name_translation.csv

### Tools

MySQL - Data Cleaning and Analysis
PowerBi - Creating Reports

### Data Cleaning/Preparation

In the initial data preparation phase, we performed the following tasks:

1. Data loading and inspection.
2. Standardizing names for consistency.

- Changed the product_category_name of the products from Brazilian Portuguese to English
```
update products p
inner join pc_name_translation pc
on p.product_category_name = pc.ï»¿product_category_name
set p.product_category_name = pc.product_category_name_english;
```
   
3. Setting the correct data type for each column.

- Changed the datatype of order_purchase_timstamp column in the orders table from 'text' to 'datetime'.
```
alter table orders
modify order_purchase_timestamp datetime;
```

- Changed blank values to null of order_approved_at, order_delivered_carrier_date, order_delivered_customer_date columns in the orders table to simplify the process of changing the data types of the aformentioned
  columns from 'text' to 'datetime'.

```
update orders
set order_approved_at = null
where length(order_approved_at) = 0;

update orders
set order_delivered_carrier_date = null
where length(order_delivered_carrier_date) = 0;

update orders
set order_delivered_customer_date = null
where  length(order_delivered_customer_date) = 0;

alter table orders
modify order_approved_at datetime;

alter table orders
modify order_delivered_carrier_date datetime;

alter table orders
modify order_delivered_customer_date datetime;
```
- Changed the data type of 'order_estimated_delivery_date' from 'text' to 'datetime'
```
alter table orders
modify order_estimated_delivery_date datetime;
```

### Questions

1. What is the total revenue?
2. What is the 12 months historical revenue trend?
3. What is the average order value?
4. What is the 12 months historcial average order value trend?
5. How much percentage of orders are delivered on time?
6. What is the 12 months historical orders delivered on time trend?
7. What is the average customer satisfaction rating out of 5?
8. What is the 12 months historical average customer satisfaction rating out of 5?
9. What are the top 5 states in Brazil by revenue?
10. What are the top 5 product categories by revenue?
11. Who are the top 10 customers by revenue in the last 12 months?

### Data Analysis

1. What is the total revenue?
   
```
select sum(price) as total_sales from order_items oi
inner join  orders o
on oi.order_id = o.order_id
where o.order_status = 'delivered';
```

2. What is the 12 months historical revenue trend?

```
with cte as
(select month(o.order_purchase_timestamp) as month, year(o.order_purchase_timestamp) as year,
sum(price) as total_sales from order_items oi
inner join  orders o
on oi.order_id = o.order_id
where o.order_purchase_timestamp > '2017-09-01' and o.order_status = 'delivered'
group by year(o.order_purchase_timestamp), month(o.order_purchase_timestamp)
order by year(o.order_purchase_timestamp), month(o.order_purchase_timestamp))
select concat(month,'-',year) as date, total_sales
from cte;
```

3. What is the average order value?

```
with cte as
(select oi.order_id, sum(price) as sales_per_order from order_items oi
inner join orders o
on oi.order_id = o.order_id
where o.order_status = 'delivered'
group by oi.order_id)
select avg(sales_per_order) as avg_order_size from cte;
```

4. What is the 12 months historcial average order value trend?

```
with cte as
(select month, year, avg(sales_per_order) as avg_order_value
from (select month(o.order_purchase_timestamp) as month, year(o.order_purchase_timestamp) as year,
	 oi.order_id, sum(price) as sales_per_order from order_items oi
	 inner join orders o
	 on oi.order_id = o.order_id
	 where o.order_status = 'delivered' and o.order_purchase_timestamp > '2017-09-01'
	 group by oi.order_id, year(o.order_purchase_timestamp), month(o.order_purchase_timestamp)
	 order by year(o.order_purchase_timestamp), month(o.order_purchase_timestamp)) as new_table
group by year, month)
select concat(month,'-',year) as date, avg_order_value
from cte;
```

5. How much percentage of orders are delivered on time?

```
with new_table as
(select count(*) as delivered_on_time, (select count(*) 
										from orders
                                        where order_status = 'delivered') 
                                        as total_no_of_orders
from orders
where order_status = 'delivered' and order_delivered_customer_date < order_estimated_delivery_date)
select round((delivered_on_time/total_no_of_orders),2) as percentage_of_order_delivered_on_time 
from new_table;
```

6. What is the 12 months historical orders delivered on time trend?

```
with cte1 as
(select month(order_purchase_timestamp) as month, year(order_purchase_timestamp) as year,
count(*) as delivered_on_time 
from orders
where order_status = 'delivered' and order_delivered_customer_date < order_estimated_delivery_date and
order_purchase_timestamp > '2017-09-01'
group by year(order_purchase_timestamp), month(order_purchase_timestamp)
order by year, month),

cte2 as
(select month(order_purchase_timestamp) as month, year(order_purchase_timestamp) as year,
 count(*) as total_no_of_orders
from orders
where order_status = 'delivered' and order_purchase_timestamp > '2017-09-01'
group by year(order_purchase_timestamp), month(order_purchase_timestamp)
order by year, month)

select concat(c1.month,'-', c1.year) as date, 
(delivered_on_time/total_no_of_orders) as delivered_on_time from cte1 c1
inner join cte2 c2
on c1.month = c2.month;
```

7. What is the average customer satisfaction rating out of 5?

```
with new_table as
(select review_score, count(*) as no_of_reviews
from order_reviews
group by review_score)
select concat(round(sum(review_score*no_of_reviews)/sum(no_of_reviews),0),'/5') as avg_ratings
from new_table;
```
8. What is the 12 months historical average customer satisfaction rating out of 5?

```
with cte as
(select month, year, sum(review_score*no_of_reviews)/sum(no_of_reviews) as avg_ratings
from (select month(o.order_purchase_timestamp) as month, year(o.order_purchase_timestamp) as year,
	 o_r.review_score, count(*) as no_of_reviews from order_reviews o_r
	 inner join orders o
	 on o_r.order_id = o.order_id
	 where o.order_purchase_timestamp > '2017-09-01'
	 group by o_r.review_score, year(o.order_purchase_timestamp), month(o.order_purchase_timestamp)
	 order by year, month, o_r.review_score) as new_table
group by year, month
order by year, month)
select concat(month,'-',year) as date, avg_ratings
from cte;
```
9. What are the top 5 states in Brazil by revenue?

```
select c.customer_state, sum(oi.price) as total_sales from customers c
inner join orders o
on c.customer_id = o.customer_id
inner join order_items oi
on o.order_id = oi.order_id
where o.order_status = 'delivered'
group by c.customer_state
order by total_sales desc
limit 5;
```
10. What are the top 5 product categories by revenue?

```
select p.product_category_name, sum(oi.price) as total_sales 
from products p inner join order_items oi
on p.product_id = oi.product_id
inner join orders o
on oi.order_id = o.order_id
where o.order_status = 'delivered'
group by p.product_category_name
order by total_sales desc
limit 5;
```
11. Who are the unique ids of the top 10 customers by revenue in the last 12 months?

```
select c.customer_unique_id, sum(oi.price) as total_sales 
from customers c inner join orders o
on c.customer_id = o.customer_id
inner join order_items oi
on o.order_id = oi.order_id
where o.order_purchase_timestamp > '2017-09-01' and o.order_status = 'delivered'
group by c.customer_unique_id
order by total_sales desc
limit 10;
```

### Findings

The analysis results are summarized as follows:

 1. Total revenue is R$13.22M
 2. .
 3. Average order value is R$137.04.
 4. .
 5. 92% of orders are delivered on time.
 6. .
 7. The average rating for any given product is 4 stars out of 5.
 8. .
 9. Sao Paolo, Rio de Janeiro, Minas Gerais, Parana, Rio Grande do Sul, are the top 5 states in terms of revenue.
10. Health & Beauty, Watches & Gifts, Bed & Bath & Table, Sports & Leisure, Computer Accessories are the top 5 product categories in terms of revenue.
11. 0a0a92112bd4c708ca5fde585afaa872, 763c8b1c9c68a0229c42c9fc6f662b93, 459bef486812aa25204be022145caa62, 4007669dec559734d6f53e029e360987, 48e1ac109decbb87765a3eade6854098, a229eba70ec1c2abef51f04987deb7a5,
    edde2314c6c30e864a128ac95d6b2112, fa562ef24d41361e476e748681810e1e, c8460e4251689ba205045f3ea17884a1 and ca27f3dac28fb1063faddd424c9d95fa are the unique ids of the top 10 customers in terms of revenue.

### Recommendations

