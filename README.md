# Amazon-sales-project-sql-
---
## **Project Overview**
I have worked on analyzing a dataset of over 20,000 sales records from an Amazon-like e-commerce platform. This project involves extensive querying of customer behavior, product performance and sales trends using MySQL. Through this project, I have tackled various SQL problems, including revenue analysis, customer segmentation, and inventory management.

The project also focuses on data cleaning, handling null values, and solving real-world business problems using structured queries.

An ERD diagram is included to visually represent the database schema and relationships between tables.
---
```sql
-- AMAZON_DB
-- CREATE TABLES
-- CATEGORY TABLE
create table category
(
category_id int primary key,
category_name varchar(25)
);

-- CUSTOMER TABLE
create table customers
(
customer_id int primary key,
f_name varchar(25),
l_name varchar(25),
state varchar(20),
address varchar(10) default('xyz')
);

-- SELLERS TABLE
create table sellers
(
seller_id int primary key,
seller_name varchar(25),
origin varchar(10)
);

-- PRODUCT TABLE
create table products
(
product_id int primary key,
product_name varchar(50),
price float,
cogs float,
category_id int, -- fk
constraint products_fk_category foreign key(category_id)references category(category_id)
);

-- ORDERS TABLE
create table orders
(
order_id int primary key,
order_date date,
customer_id int, -- fk
seller_id int, -- fk
order_status varchar(25),
constraint orders_fk_customers foreign key(customer_id)references customers(customer_id),
constraint orders_fk_sellers foreign key(seller_id)references sellers(seller_id)
);

-- ORDER ITEMS
create table order_items
(
order_item_id int primary key,
order_id int, -- fk
product_id int, -- fk
quantity int,
price_per_unit float,
constraint order_items_fk_orders foreign key(order_id)references orders(order_id),
constraint order_items_fk_product_id foreign key(product_id)references products(product_id)
);

-- PAYMENTS TABLE
create table payments
(
payment_id int primary key,
order_id int, -- fk
payment_date date,
payment_status varchar(50),
constraint payments_fk_orders foreign key(order_id)references orders(order_id)
);

-- SHIPPING TABLE
create table shipping
(
shipping_id int primary key,
order_id int, -- fk
shipping_date date,
return_date date,
shipping_providers varchar(40),
delivery_status varchar(30),
constraint shipping_fk_orders foreign key(order_id)references orders(order_id)
);

-- INVENTORY TABLE
create table inventory
(
inventory_id int primary key,
product_id int, -- fk
stock int,
warehouse_id int,
last_stock_date date,
constraint inventory_fk_products foreign key(product_id)references products(product_id)
);
```
## **Data Cleaning**

I cleaned the dataset by:
- Removing duplicates: Duplicates in the customer and order tables were identified and removed.
- Handling missing values: Null values in critical fields (e.g., customer address, payment status) were either filled with default values or handled using appropriate methods.
---
## **Handling Null Values**

Null values were handled based on their context:
- Customer addresses: Missing addresses were assigned default placeholder values.
- Payment statuses: Orders with null payment statuses were categorized as “Pending.”-
- Shipping information: Null return dates were left as is, as not all shipments are returned.
---
## **Objective**

The primary objective of this project is to showcase SQL proficiency through complex queries that address real-world e-commerce business challenges. The analysis covers various aspects of e-commerce operations, including:
- Customer behavior
- Sales trends
- Inventory management
- Payment and shipping analysis
- Forecasting and product performance
---
## **Identifying Business Problems**

Key business problems identified:
1. Low product availability due to inconsistent restocking.
2. High return rates for specific product categories.
3. Significant delays in shipments and inconsistencies in delivery times.
4. High customer acquisition costs with a low customer retention rate.
---
## **Solving Business Problems**
```sql
1-- TOP 10 SELLING PRODUCT
-- Include product name, total quantity sold, and total sales value.
select*from order_items;

 -- CREATING NEW COLUMN
 alter table order_items
 add column total_sale float;
 
 -- UPDATING QUANTITY * PRICE_PER_UNIT
 update order_items
 set total_sale=quantity*price_per_unit;
 
 select*from order_items
 order by quantity desc;
 
 -- JOIN ORDERS TABLE AND PRODUCTS TABLE WITH ORDER_ITEMS TABLE
 
 select 
 oi.product_id,
 p.product_name,
 round(sum(oi.total_sale),2) as total_sale,
 count(o.order_id) as total_order
 from orders as o
 join order_items as oi
 on o.order_id=oi.order_id
 join products as p
 on oi.product_id=p.product_id
 group by 1,2
 order by 3 desc
 limit 10;
 
 2-- REVENUE BY CATEGORY
 -- Include the percentage contribution of each category to total revenue.
 select c.category_name,sum(oi.total_sale),round(sum(oi.total_sale)/(select sum(total_sale) from order_items)*100) as contribution
from products as p
join category as c
on p.category_id=c.category_id
join order_items as oi
on oi.product_id=p.product_id
group by 1
order by 3 desc;

3-- AVERAGE ORDER VALUE (AOV)
-- Include only customers with more than 5 orders.
select c.customer_id,
concat(c.f_name,' ',c.l_name)as full_name,
sum(total_sale)/count(o.order_id)as avg_orders_value,
count(o.order_id)as total_orders
from orders as o
join customers as c
on c.customer_id=o.customer_id
join order_items as oi
on oi.order_id=o.order_id
group by 2
having count(o.order_id)>5;

4-- MONTHLY SALES TREND
-- Display the sales trend, grouping by month, return current_month sale, last month sale!
select year,month,total_sale,
lag(total_sale,1)over(order by year,month)as last_month
from
(
select month(o.order_date) as month,year(o.order_date)as year,round(sum(oi.total_sale))as total_sale
from orders as o
join order_items as oi
on oi.order_id=o.order_id
where order_date>=current_date - interval 1 year
group by 1,2
order by 2,1
)as t;

5-- CUSTOMERS WITH NO PURCHASE
-- Find customers who have registered but never placed an order.
select distinct c.customer_id,concat(f_name,' ',l_name) as full_name from customers as c
left join orders as o
on c.customer_id=o.customer_id
where o.customer_id is null;

6-- BEST SELLING CATEGORIES BY STATE
-- Identify best-selling category for each state
with best_selling as
(
select cu.state,c.category_name,sum(oi.total_sale) as total_sale,
rank() over(partition by cu.state order by sum(oi.total_sale)desc)as rnk
from category as c
join products as p
on c.category_id=p.category_id
join order_items as oi
on p.product_id=oi.product_id
join orders as o
on oi.order_id=o.order_id
join customers as cu
on o.customer_id=cu.customer_id
group by 1,2
)
select*from best_selling
where rnk = 1;

7-- CUSTOMER LIFETIME VALUE(CLTV)
-- Calculate the total value of orders placed by each customer over their lifetime.
select c.customer_id,concat(f_name,' ',l_name) as full_name,round(sum(total_sale),1) as cltv,
dense_rank() over(order by round(sum(total_sale),1)desc) as `rank`
from customers as c
join orders as o
on c.customer_id=o.customer_id
join order_items as oi
on o.order_id=oi.order_id
group by 1;

8--  INVENTORY STOCK ALERTS
-- Query products with stock levels below a certain threshold (e.g., less than 10 units)
select p.product_name,i.stock,i.last_stock_date,i.warehouse_id from inventory as i
join products as p
on i.product_id=p.product_id
where i.stock<10
order by i.last_stock_date desc;

9-- SHIPPING DELAYS
-- Identify orders where the shipping date is later than 3 days after the order date.
select o.order_id,s.shipping_id,o.order_date,s.shipping_date,s.shipping_providers,concat(f_name,' ',l_name) as full_name,
datediff(s.shipping_date,o.order_date) as days_took_to_ship
from orders as o
join shipping as s
on o.order_id=s.order_id
join customers as c
on o.customer_id=c.customer_id
where datediff(s.shipping_date,o.order_date)>3;

10--  PAYMENT SUCCESS RATE 
-- Calculate the percentage of successful payments across all orders.
select p.payment_status,count(*) as total_count,
count(*)/(select count(*) from payments) * 100 as percent from orders as o
join payments as p
on o.order_id=p.order_id
group by 1;

11-- TOP PERFORMING SELLERS
-- Find the top 5 sellers based on total sales value.
with top_sellers as 
(
select s.seller_id,s.seller_name,round(sum(oi.total_sale),2) as total_sale from sellers as s
join orders as o
on s.seller_id=o.seller_id
join order_items as oi
on o.order_id=oi.order_id
group by 1,2
order by total_sale desc
limit 5
),
seller_reports as
(
select o.seller_id,ts.seller_name,o.order_status,count(*) as total_orders 
from orders as o
join top_sellers as ts
on o.seller_id=ts.seller_id
where o.order_status not in('Returned','Inprogress')
group by 1,2,3
)
select seller_id,seller_name,
sum(case when order_status='Completed' then total_orders else 0 end) as completed_orders,
sum(case when order_status='Cancelled' then total_orders else 0 end) as cancelled_orders,
sum(total_orders) as total_orders,
sum(case when order_status='Completed' then total_orders else 0 end)/sum(total_orders)*100 as `% of successful orders`
from seller_reports
group by 1,2
order by 6 desc;
 
12-- PRODUCT PROFIT MARGIN
-- Calculate the profit margin for each product (difference between price and cost of goods sold). 
select p.product_id,p.product_name,
round(sum(oi.total_sale-(p.cogs*oi.quantity)),2) as profit,
sum(oi.total_sale-(p.cogs*oi.quantity))/sum(total_sale)*100 as profit_margin, 
dense_rank() over(order by(sum(oi.total_sale-(p.cogs*oi.quantity))/sum(total_sale)*100) desc)
from products as p
join order_items as oi
on p.product_id=oi.product_id
group by 1,2;

13-- MOST RETURNED PRODUCTS
-- Query the top 10 products by the number of returns.
select p.product_id,p.product_name,count(*),
sum(case when o.order_status='Returned' then 1 else 0 end) as total_order_returned,
sum(case when o.order_status='Returned' then 1 else 0 end)/count(*)*100 as `percent of total returned`
from products as p
join order_items as oi
on p.product_id=oi.product_id
join orders as o
on oi.order_id=o.order_id
group by 1
order by  `percent of total returned` desc ;

14-- IDENTITY CUSTOMERS INTO RETURNING OR NEW
-- if the customer has done more than 5 return categorize them as returning otherwise new.
select t.customer_id,full_name,total_orders,total_returns,
case when total_returns>5 then 'Returning customer' else 'New' end as cx
from
(
select c.customer_id,concat(f_name,' ',l_name) as full_name,count(o.order_id) as total_orders,
sum(case when o.order_status='Returned' then 1 else 0 end)as total_returns
from customers as c
join orders as o
on c.customer_id=o.customer_id
group by 1
)as t;

15-- Top 5 CUSTOMERS BY ORDERS IN EACH STATE
-- Identify the top 5 customers with the highest number of orders for each state.

select*from(
select c.state,c.customer_id,concat(f_name,' ',l_name) as full_name,
count(oi.order_id) as total_orders,
sum(oi.total_sale) as total_sale,
dense_rank() over(partition by c.state order by count(o.order_id) desc)as ranking
from orders as o
join customers as c
on o.customer_id=c.customer_id
join order_items as oi
on o.order_id=oi.order_id
group by 1,2
) as t
where ranking <=5;

16-- REVENUE BY SHIPPING PROVIDER
-- Calculate the total revenue handled by each shipping provider.
select s.shipping_providers,count(oi.order_id),round(sum(oi.total_sale),2),
coalesce(avg(s.return_date-s.shipping_date),0) as avg_days
from shipping as s
join order_items as oi
on s.order_id=oi.order_id
group by 1;

17-- INACTIVE SELLERS
-- Identify sellers who haven’t made any sales in the last 6 months
with cte1 as
(
select*from sellers 
where seller_id not in(select seller_id from orders where order_date>=current_date()-interval 1 year)
)
select o.seller_id,max(o.order_date)as last_sale_date,max(oi.total_sale)as last_sale_amount
from orders as o
join cte1
on cte1.seller_id=o.seller_id
join order_items as oi
on o.order_id=oi.order_id
group by 1;


18-- Top 10 product with highest decreasing revenue ratio compare to last year(2022) and current_year(2023)
-- Challenge: Return product_id, product_name, category_name, 2022 revenue and 2023 revenue decrease ratio at end Round the result
with 2022_sale as
(
select c.category_name,p.product_id,p.product_name,sum(oi.total_sale) as last_revenue,o.order_date from category as c
join products as p
on c.category_id=p.category_id
join order_items as oi
on p.product_id=oi.product_id
join orders as o
on oi.order_id=o.order_id
where year(o.order_date)=2022
group by 2,3
),
2023_sale as
(
select c.category_name,p.product_id,p.product_name,sum(oi.total_sale)as current_revenue,o.order_date from category as c
join products as p
on c.category_id=p.category_id
join order_items as oi
on p.product_id=oi.product_id
join orders as o
on oi.order_id=o.order_id
where year(o.order_date)=2023
group by 2,3
)
select cs.category_name,cs.product_id,cs.product_name,ls.last_revenue,cs.current_revenue,
ls.last_revenue-cs.current_revenue as revenue_diff,
round((ls.last_revenue-cs.current_revenue)/ls.last_revenue*100,2) as revenue_ratio
from 2022_sale as ls
join 2023_sale as cs
on ls.product_id=cs.product_id
where last_revenue>current_revenue
group by 1,2
order by revenue_ratio desc
limit 10;

-- Create a stored procedure that, when a product is sold, performs the following actions:
-- Inserts a new sales record into the orders and order_items tables.
-- Updates the inventory table to reduce the stock based on the product and quantity purchased.
-- The procedure should ensure that the stock is adjusted immediately after recording the sale.

delimiter $$

create procedure add_sales
(
p_order_id int,
p_customer_id int,
p_seller_id int,
p_order_item_id int,
p_product_id int,
p_quantity int
)
begin
declare v_count int;
declare v_price float;
declare v_product varchar(50);

select price,product_name into v_price,v_product
from products
where product_id=p_product_id;

select count(*) into v_count
from inventory
where product_id=p_product_id
and stock>=p_quantity;

if v_count>0 then

insert into orders(order_id, order_date, customer_id, seller_id)
values(p_order_id, CURRENT_DATE, p_customer_id, p_seller_id);

insert into order_items (order_item_id, order_id, product_id, quantity, price_per_unit, total_sale)
values(p_order_item_id, p_order_id, p_product_id, p_quantity, v_price, v_price * p_quantity);

update inventory
set stock=stock-p_quantity
where product_id=p_product_id;

select concat('Thank you, product: ', v_product, ' sale has been added and inventory stock updated.') as message;
    ELSE
        
select concat('Thank you for your info, product: ', v_product, ' is not available.') as message;
end if;
end$$

DELIMITER ;
```
## **Learning Outcomes**

This project enabled me to:
- Design and implement a normalized database schema.
- Clean and preprocess real-world datasets for analysis.
- Use advanced SQL techniques, including window functions, subqueries, and joins.
- Conduct in-depth business analysis using SQL.
- Optimize query performance and handle large datasets efficiently.

---

## **Conclusion**

This advanced SQL project successfully demonstrates my ability to solve real-world e-commerce problems using structured queries. From improving customer retention to optimizing inventory and logistics, the project provides valuable insights into operational challenges and solutions.

By completing this project, I have gained a deeper understanding of how SQL can be used to tackle complex data problems and drive business decision-making.

---

### **Entity Relationship Diagram (ERD)**
![ERD](https://github.com/najirh/amazon_usa_project5/blob/main/erd.png)

---
   



