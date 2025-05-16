---

# **Amazon USA Sales Analysis Project**

### **Difficulty Level: Advanced**

---

## **Project Overview**

This project analyzing a dataset of over 20,000 sales records from an Amazon-like e-commerce platform. This project involves extensive querying of customer behavior, product performance, and sales trends using PostgreSQL. Through this project, I have tackled various SQL problems, including revenue analysis, customer segmentation, and inventory management.

The project also focuses on data cleaning, handling null values, and solving real-world business problems using structured queries.

An ERD diagram is included to visually represent the database schema and relationships between tables.

---

![ERD Scratch](https://github.com/kblaryea/Amazon_project/blob/main/ERD.png)

## **Database Setup & Design**

### **Schema Structure**

```sql
--Creating category table

CREATE TABLE category
(	category_id INT PRIMARY KEY,
	category_name VARCHAR(20)
);


--Creating customers table

CREATE TABLE customers
(	Customer_id INT PRIMARY KEY,
	first_name VARCHAR(50),
	last_name VARCHAR(50),
	state VARCHAR(20),
	address VARCHAR(10) DEFAULT('xxx')
);

--Creating sellers table

CREATE TABLE sellers
(	seller_id INT PRIMARY KEY,
	seller_name VARCHAR(30),
	origin VARCHAR(10)
);

--Creating product table

CREATE TABLE products
(	product_id INT PRIMARY KEY,
	product_name VARCHAR(50),
	price FLOAT,
	cogs FLOAT,
	category_id INT, -- FK
	CONSTRAINT product_fk_category FOREIGN KEY(category_id) REFERENCES category(category_id)
);


--Creating orders table
CREATE TABLE orders
(	order_id INT PRIMARY KEY,
	order_date DATE, 
	customer_id INT, --fk
	seller_id INT, --fk
	order_status VARCHAR(20),
	CONSTRAINT orders_fk_customers FOREIGN KEY(customer_id) REFERENCES customers(customer_id),
	CONSTRAINT orders_fk_sellers FOREIGN KEY(seller_id) REFERENCES sellers(seller_id)
);

--Creating order_items table
CREATE TABLE order_items
(	order_item_id INT PRIMARY KEY,	
	order_id INT,--fk
	product_id INT, --fk
	quantity INT,
	price_per_unit FLOAT,
	CONSTRAINT order_items_fk_orders FOREIGN KEY(order_id) REFERENCES orders(order_id),
	CONSTRAINT order_items_fk_products FOREIGN KEY(product_id) REFERENCES products(product_id)
);

--Creating payments table

CREATE TABLE payments
(	payment_id INT PRIMARY KEY,
	order_id INT,--fk
	payment_date DATE,
	payment_status VARCHAR(20),
	CONSTRAINT payments_fk_orders FOREIGN KEY(order_id) REFERENCES orders(order_id)
);

--Creating shippings table

CREATE TABLE shippings
(	shipping_id INT PRIMARY KEY,
	order_id INT, --fk
	shipping_date DATE,
	return_date DATE,
	shipping_providers VARCHAR(20),
	delivery_status VARCHAR(20),
	CONSTRAINT shippings_fk_orders FOREIGN KEY(order_id) REFERENCES orders(order_id)
);

--Creating inventory table

CREATE TABLE inventory
(	inventory_id INT PRIMARY KEY,
	product_id INT, --fk
	stock INT,
	warehouse_id INT,
	last_stock_date DATE,
	CONSTRAINT inventory_fk_products FOREIGN KEY(product_id) REFERENCES products(product_id)
);

-- End of Schemas

```

---

## **Task: Data Cleaning**

I cleaned the dataset by:
- **Removing duplicates**: Duplicates in the customer and order tables were identified and removed.
- **Handling missing values**: Null values in critical fields (e.g., customer address, payment status) were either filled with default values or handled using appropriate methods.

---

## **Handling Null Values**

Null values were handled based on their context:
- **Customer addresses**: Missing addresses were assigned default placeholder values.
- **Payment statuses**: Orders with null payment statuses were categorized as “Pending.”
- **Shipping information**: Null return dates were left as is, as not all shipments are returned.

---

## **Objective**

The primary objective of this project is to showcase SQL proficiency through complex queries that address real-world e-commerce business challenges. The analysis covers various aspects of e-commerce operations, including:
- Customer behavior
- Sales trends
- Inventory management
- Payment and shipping analysis
- Forecasting and product performance
  

## **Identifying Business Problems**

Key business problems identified:
1. Low product availability due to inconsistent restocking.
2. High return rates for specific product categories.
3. Significant delays in shipments and inconsistencies in delivery times.
4. High customer acquisition costs with a low customer retention rate.

---



## **Solving Business Problems**

### Solutions Implemented:
1. Top Selling Products
Query the top 10 products by total sales value.
Challenge: Include product name, total quantity sold, and total sales value.

```sql
Select 
	pr.product_id, 
	pr.product_name, 
	sum(quantity)as total_quantity, 
	sum(quantity*price_per_unit) as total_sale_value
From products as pr
Join order_items as oi
on pr.product_id = oi.product_id
Group by pr.product_id, pr.product_name
Order by total_sale_value desc
Limit 10;

-- ALTERNATIVE SOLUTION

--Creating new column for total sales
Alter Table order_items
Add Column total_sale Float;

--Updating qty * price per unit
Update order_items
Set total_sale = quantity * price_per_unit;



Select 
	oi.product_id,
	p.product_name,
	count(o.order_id)as total_orders,
	sum(oi.total_sale) as total_sale
From orders as o
Join order_items as oi
On oi.order_id = o.order_id
Join products as p
On p.product_id = oi.product_id
Group by oi.product_id, p.product_name
Order by total_sale desc
Limit 10;
```

![](https://github.com/kblaryea/Amazon_project/blob/main/Top_10_products.png)

#### Remarks:
The Apple iMac Pro leads sales by a wide margin at $630K, followed by the iMac 27‑Inch Retina ($232K) and the Canon EOS R5 ($222K), while most other top products each generate between $145K and $190K.


2. Revenue by Category
Calculate total revenue generated by each product category.
Challenge: Include the percentage contribution of each category to total revenue.

```sql
Select 
	t1.category_id, 
	t1.category_name, 
	t1.total_revenue, 
	t1.total_revenue/(Select sum(total_sale)from order_items) * 100 as share_of_TR
From
(
Select 
	cat.category_id, 
	cat.category_name, 
	sum(oi.total_sale)::numeric as total_revenue
From products as p
Join category as cat
On p.category_id = cat.category_id
Join order_items as oi
On p.product_id = oi.product_id
Group by 
	cat.category_id, 
	cat.category_name
)t1
Group by 
	t1.category_id, 
	t1.category_name, 
	t1.total_revenue
Order by total_revenue desc
```
![](https://github.com/kblaryea/Amazon_project/blob/main/Item_categories.png)

#### Remarks
Electronics overwhelmingly leads with $11.34 M in revenue—about 89.7% of the total. 
The next highest categories are Sports & Outdoors ($457 K, 3.6%) and Toys & Games ($354 K, 2.8%). 
Pet Supplies contributes $262 K (2.1%), while Clothing and Home & Kitchen each account for just over 1% and under 1% of revenue, respectively.

3. Average Order Value (AOV)
Compute the average order value for each customer.
Challenge: Include only customers with more than 5 orders.

```sql
Select 
	cu.customer_id, 
	Concat(cu.first_name, ' ', cu.last_name) as full_name,  
	avg(oi.total_sale) as average_order_value,
	count(o.order_id) as total_orders
From orders as o
Join customers as cu
On o.customer_id = cu.customer_id
Join order_items as oi
On o.order_id = oi.order_item_id
Group by cu.customer_id
Having count(o.order_id) > 5
Order by average_order_value desc;
```

4. Monthly Sales Trend
Query monthly total sales over the past year.
Challenge: Display the sales trend, grouping by month, return current_month sale, last month sale!

```sql
---FOR ALL MONTHS AND ALL YEARS
Select 
	Extract(Month from order_date) as Month,
	Extract(Year from order_date) as Year,
	round(sum(oi.total_sale)::numeric, 2) as current_month_sales,
	round(lag (sum(oi.total_sale)::numeric, 1) over(Order by Extract(Year from order_date), Extract(Month from order_date)), 2) as previous_month_sales
From order_items as oi
Join orders as o
on oi.order_id = o.order_id
Where o.order_status != 'Returned' or o.order_status != 'Cancelled'
Group by Month, Year
Order by Year, Month;


--ALTERNATIVE SOLUTION FOR THE PAST YEAR
Select 
	year, 
	month, 
	total_sale as current_month_sale,
	Lag(total_sale, 1) Over(Order by year, month) as last_month_sale
From
(
Select 
	Extract(Month from order_date) as Month,
	Extract(Year from order_date) as Year,
	round(sum(oi.total_sale)::numeric, 2) as total_sale
From orders as o
Join order_items as oi
on oi.order_id = o.order_id
Where order_date >= Current_Date - Interval '1 year'
Group by Month, Year
Order by Year, Month
);
```


5. Customers with No Purchases
Find customers who have registered but never placed an order.
Challenge: List customer details and the time since their registration.

```sql
Select *
From customers as c
Left Join orders as o
On c.customer_id = o.customer_id
Where order_id is Null;

--ALTERNATIVE SOLUTION

Select * From customers
Where  customer_id not in (Select 
				Distinct customer_id
				From orders
				);
```

6. Least-Selling Categories by State
Identify the least-selling product category for each state.
Challenge: Include the total sales for that category within each state.

```sql
Select *
From 
(
Select 
	c.state, 
	cat.category_name, 
	sum(oi.total_sale) as sales_by_state,
	Rank () Over (Partition by c.state Order by sum(oi.total_sale) Asc) as Rank
From 
	orders as o
Join 
	order_items as oi
On 
	o.order_id = oi.order_id
Join 
	customers as c
On 
	o.customer_id = c.customer_id
Join 
	products as p
On 
	oi.product_id = p.product_id
Join 
	category as cat
On 
	p.category_id = cat.category_id
Group by 
	c.state, cat.category_name
Order by 1, 3 Asc
)
Where rank = 1;

```


7. Customer Lifetime Value (CLTV)
Calculate the total value of orders placed by each customer over their lifetime.
Challenge: Rank customers based on their CLTV.

```sql
Select 
	c.customer_id, 
	concat(c.first_name, ' ', c.last_name) as full_name,
	round(sum(oi.total_sale)::numeric, 2) as total_value,
	 dense_rank () over(Order by sum(oi.total_sale) Desc) as rank
From 
	orders as o
Join 
	order_items as oi
On 
	o.order_id = oi.order_id
Join 
	customers as c
On 
	o.customer_id = c.customer_id
Group by c.customer_id, full_name
Order by total_value Desc;
```


8. Inventory Stock Alerts
Query products with stock levels below a certain threshold (e.g., less than 10 units).
Challenge: Include last restock date and warehouse information.

```sql
Select 
	inv.inventory_id,
	p.product_id, 
	p.product_name, 
	inv.last_stock_date, 
	inv.stock, 
	inv.warehouse_id
From inventory as inv
Left Join products as p
On inv.product_id = p.product_id
Where inv.stock < 10
Order by inv.stock Desc;
```

9. Shipping Delays
Identify orders where the shipping date is later than 3 days after the order date.
Challenge: Include customer, order details, and delivery provider.

```sql
Select 
	s.shipping_id,
	o.order_id,
	c.customer_id,
	concat(c.first_name, ' ', c.last_name) as full_name,
	s.shipping_providers,
	o.order_date,
	s.shipping_date,
	s.shipping_date - o.order_date as so_diff
From shippings as s
Join orders as o
On s.order_id = o.order_id
Left Join customers as c
On o.customer_id = c.customer_id
Where s.shipping_date - o.order_date > 3
```

10. Payment Success Rate 
Calculate the percentage of successful payments across all orders.
Challenge: Include breakdowns by payment status (e.g., failed, pending).

```sql
Select 
	payment_status, 
	count(*)::numeric as payment_count,
	round(count(*)/(select count(*) from orders)::numeric * 100, 2) as share_of_payment
From payments
Group by payment_status;
```
![](https://github.com/kblaryea/Amazon_project/blob/main/payment_status.png)

#### Remarks
Most transactions are successful, indicating a stable payment system. 
However, the 13.13% refund rate is relatively high and may warrant further investigation into causes such as order issues, product returns, or user errors.


11. Top Performing Sellers
Find the top 5 sellers based on total sales value.
Challenge: Include both successful and failed orders, and display their percentage of successful orders.

```sql
With top_sellers
As
(
Select 
	s.seller_id, 
	s.seller_name, 
	sum(oi.total_sale) as total_sale_value
From orders as o
Join sellers as s
On o.seller_id = s.seller_id
Join order_items as oi
On o.order_id = oi.order_id
Group by s.seller_id, 2
Order by total_sale_value Desc
Limit 5
),
seller_report
As (
Select 
	o.seller_id,
	ts.seller_name,
	o.order_status, 
	count(*) as total_orders,
	ts.total_sale_value as total_sale_value
from orders as o
Join top_sellers as ts
On ts.seller_id = o.seller_id
Where o.order_status Not In ('Inprogress', 'Returned')
Group by o.order_status, o.seller_id, ts.seller_name, ts.total_sale_value)

Select 
	seller_id,
	seller_name,
	Sum(Case when order_status = 'Completed' then total_orders Else 0 End) as completed_orders,
	Sum(Case when order_status = 'Cancelled' then total_orders Else 0 End) as cancelled_orders,
	Sum(total_orders) as total_orders,
	round(Sum(Case when order_status = 'Completed' then total_orders Else 0 End)::numeric/
	Sum(total_orders)::numeric * 100, 2) as successful_order_ratio,
	total_sale_value
from seller_report
Group by seller_id, seller_name, total_sale_value;
```


12. Product Profit Margin
Calculate the profit margin for each product (difference between price and cost of goods sold).
Challenge: Rank products by their profit margin, showing highest to lowest.
*/


```sql
Select 
	p.product_id,
	p.product_name,
	sum(oi.quantity) as quantity_orders,
	p.cogs,
	sum(total_sale)::numeric as total_revenue,
	(sum(oi.quantity)*cogs):: numeric as total_cost,
	(sum(total_sale)::numeric - (sum(oi.quantity)*cogs):: numeric) as profit,
	round((sum(total_sale)::numeric - (sum(oi.quantity)*cogs):: numeric)/sum(total_sale)::numeric *100, 2) as profit_margin,
	dense_rank () over (order by ((sum(total_sale)::numeric - (sum(oi.quantity)*cogs):: numeric)/sum(total_sale)::numeric *100) Desc)
From order_items as oi
Join products as p
On oi.product_id = p.product_id
Group by p.product_id, p.product_name
Order by profit_margin Desc
Limit 10;
```

13. Most Returned Products
Query the top 10 products by the number of returns.
Challenge: Display the return rate as a percentage of total units sold for each product.

```sql
Select 
	p.product_id, 
	p.product_name, 
	sum(quantity)::numeric as total_unit_sold,
	sum(Case when o.order_status = 'Returned' then 1 Else 0 End)::numeric as total_return,
	round(sum(Case when o.order_status = 'Returned' then 1 Else 0 End)::numeric/sum(quantity)::numeric *100, 2) as percentage_returned
From orders as o
Join order_items as oi
On o.order_id = oi.order_id
Join products as p
On oi.product_id = p.product_id
Group by p.product_id, p.product_name 
Order by round(sum(Case when o.order_status = 'Returned' then 1 Else 0 End)::numeric/sum(quantity)::numeric *100, 2) Desc

```

14. Inactive Sellers
Identify sellers who haven’t made any sales in the last 6 months.
Challenge: Show the last sale date and total sales from those sellers.

```sql
Select 
	s.seller_id,
	s.seller_name,
	max(o.order_date) as last_order_date, 
	round(sum(oi.total_sale)::numeric, 2) as sellers_total_sale
From orders as o
Join order_items as oi
On o.order_id = oi.order_id
Right Join sellers as s
On s.seller_id = o.seller_id
Group by s.seller_id
Having max(o.order_date) > current_date - interval '10 month' or max(o.order_date) is Null

```


15. IDENTITY customers into returning or new
if the customer has done more than 5 return categorize them as returning otherwise new
Challenge: List customers id, name, total orders, total returns

```sql
Select 
	t1.full_name as customers,
	t1.total_orders,
	t1.total_return,
	Case
	When total_return > 5 then 'Returning Customer' 
	Else 'New Customer'
	End as cx_category
From
(
Select 
	concat(c.first_name, ' ', c.last_name) as full_name,
	count(o.order_id) as total_orders,
	sum(case when o.order_status = 'Returned' then 1 else 0 End) as total_return	
From orders as o
Join customers as c
on c.customer_id = o.customer_id
Join order_items as oi
on oi.order_id = o.order_id
Group by concat(c.first_name, ' ', c.last_name)
) as t1

```


16. Top 5 Customers by Orders in Each State
Identify the top 5 customers with the highest number of orders for each state.
Challenge: Include the number of orders and total sales for each customer.
```sql
-- Ranking by highest amount spend by each customer
Select *
From
(
Select
concat(c.first_name, ' ', c.last_name) as full_name,
c.customer_id,
count(o.order_id) as total_order,
sum(oi.total_sale)::numeric as total_spent,
c.state,
Rank () Over(Partition by state Order by sum(oi.total_sale)::numeric Desc) as Rank
From orders as o
Join order_items as oi
on o.order_id = oi.order_id
Join customers as c
On o.customer_id = c.customer_id
Group by c.customer_id, state
Order by State
) as t1
Where rank between 1 and 5



-- Ranking by highest number of orders made by each customer

Select *
From
(
Select
concat(c.first_name, ' ', c.last_name) as full_name,
c.customer_id,
count(o.order_id) as total_order,
sum(oi.total_sale)::numeric as total_spent,
c.state,
Dense_Rank () Over(Partition by state Order by count(o.order_id) Desc) as Rank
From orders as o
Join order_items as oi
on o.order_id = oi.order_id
Join customers as c
On o.customer_id = c.customer_id
Group by c.customer_id, state
Order by State
) as t1
Where rank between 1 and 5



```

17. Revenue by Shipping Provider
Calculate the total revenue handled by each shipping provider.
Challenge: Include the total number of orders handled and the average delivery time for each provider.

```sql

Select 
s.shipping_providers, 
count(o.order_id) as total_orders_handled, 
round(sum(oi.total_sale)::numeric, 2) as total_revenue_handled
From orders as o
Join order_items as oi
on o.order_id = oi.order_id
Join shippings as s
On s.order_id = o.order_id
Group by  s.shipping_providers;
```

18. Top 10 product with highest decreasing revenue ratio compare to last year(2022) and current_year(2023)
Challenge: Return product_id, product_name, category_name, 2022 revenue and 2023 revenue decrease ratio at end Round the result
Note: Decrease ratio = cr-ls/ls* 100 (cs = current_year ls=last_year)

```sql
Create Table table_2022
As
(
Select *
From
(
Select 
p.product_id,
p.product_name,
c.category_name,
sum(oi.total_sale)::numeric as total_revenue,
Extract(Year from o.order_date) as Year
From orders as o
Join order_items oi
On o.order_id = oi.order_id
Join products as p
On oi.product_id = p.product_id
Left Join category as c
On p.category_id = c.category_id
Where (o.order_status = 'Inprogress' or o.order_status = 'Completed') and
 (Extract(Year from o.order_date) = 2022 or Extract(Year from o.order_date) = 2023)
 Group by year, p.product_name, p.product_id, c.category_name
 ) as t1
 Where year = 2022
)

--Creating a table for 2023 data 

Create Table table_2023
As
(
Select *
From
(
Select 
p.product_id,
p.product_name,
c.category_name,
sum(oi.total_sale)::numeric as total_revenue,
Extract(Year from o.order_date) as Year
From orders as o
Join order_items oi
On o.order_id = oi.order_id
Join products as p
On oi.product_id = p.product_id
Left Join category as c
On p.category_id = c.category_id
Where (o.order_status = 'Inprogress' or o.order_status = 'Completed') and
 (Extract(Year from o.order_date) = 2022 or Extract(Year from o.order_date) = 2023)
 Group by year, p.product_name, p.product_id, c.category_name
 ) as t1
 Where year = 2023
)

--Join both tables and find the percentage change in revenue. 

Select 
t2.product_id, 
t2.product_name, 
t2.category_name, 
t2.total_revenue as rev_2022, 
t2.year, 
t3.total_revenue as rev_2023, 
t3.year,
round((t3.total_revenue-t2.total_revenue)/t2.total_revenue*100, 2) as revenue_ratio
From table_2022 as t2
Join table_2023 as t3
On t2.product_id = t3.product_id
Order by revenue_ratio 
Limit 10;
```

----------------------------------------------------------------------------------------------------

---

## **Learning Outcomes**

This project enabled me to:
- Design and implement a normalized database schema.
- Clean and preprocess real-world datasets for analysis.
- Use advanced SQL techniques, including window functions, subqueries, and joins.
- Conduct in-depth business analysis using SQL.
- Optimize query performance and handle large datasets efficiently.

---


## **Key Techniques Used**
- Joins: Combined data from multiple tables for comprehensive analysis.
- Window Functions: Used RANK(), DENSE_RANK(), and LAG() for ranking and trend analysis.
- Aggregations: Applied SUM(), COUNT(), and AVG() for calculations.
- Conditional Logic: Used CASE statements for categorization.
- Subqueries and CTEs: Modularized queries for better readability and reusability.


## Conclusion
This project demonstrates the power of SQL in solving complex business problems. By analyzing sales, customer behavior, product performance, and operational efficiency, the project provides valuable insights to improve revenue, customer satisfaction, and operational processes.
By completing this project, I have gained a deeper understanding of how SQL can be used to tackle complex data problems and drive business decision-making.
