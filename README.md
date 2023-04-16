# Project Overview
This project is a Business Intelligence Project that creates dashboard to give monitorization to Marketing and Sales Department upon product sales, region performance, and target achievement periodically. By finishing this project, we hope we can replace project consolidation process that done manually along the time and provide a dashboard that accessible by all of the region manager to monitor and evaluate performance of their region.

# Project Objectives
Objective of creation of this dashboard is to give end-to-end visualization to Marketing and Sales Department upon product sales in all region and also to give monitorization privilege about sales in certain period.

# Technical Design Diagram
![tdd](https://user-images.githubusercontent.com/99194827/232273568-569e5e0b-fed9-4bcc-98e8-4e2c56f7e13e.png)

### Components
#### Connection & Location
Data Source for this project is located on company's data lake. Company utilize **MongoDB**, a NoSQL Database, as Data Lake. It can be accessed in company's server (a Virtual Machine), exposed at port **27017**. There is 3 databases from the data lake that used in this project, which are **db_customers** (contains identity of customers), **db_products** (contains detail of products), and **db_orders** (contains record of orders).
#### Server & Storage
To maintain data quality, data source will be copied to data staging. From this data staging, we will do transformation and stored it in Data Warehouse. After that, we will also do further transformation and split data into Data Marts. We will utilize **PostgreSQL**, a Relational Database Management System (RDBMS), as Data Staging, Data Warehouse, and Data Mart. Data Staging and Data Warehouse will be located in a same PostgreSQL database named as **superstore_dwh** (but they will be put on different schema), while Data Mart will be located in separate PostgreSQL database named as **superstore_dmart**. PostgreSQL can be accessed in the same server as MongoDB. Superstore_dwh is exposed at port **5432**, while superstore_dmart is exposed at port **5433**.
#### Logical Data Flow <br>
![low_level](https://user-images.githubusercontent.com/99194827/232196388-cf54c01e-380c-45c0-922d-526e43fa8935.png)

### Attachments and Links
No attachment or link need to be embedded.

# Technical Specification
### Application Server
Server used in this project is utilized Virtual Machine instance provided by Google Compute Engine, with specifications as follow:
- Image of OS: Debian GNU/Linux 11
- Machine Type: E2
- Memory: 4GB
- CPU: 2 Core
- Disk: 10 GB
- Software: MongoDB and PostgreSQL

### Database and Server Request
Data from database used as data source for this project. Grant access as viewer to Business Intelligence Engineers is enough to fulfill _principle of least privilege_.

### Data Pipeline
#### Tools
Data Transformation in this project done with Pentaho Data Integration software. We can define all pipeline from extract data from data lake until load data to data marts in a single execution. First, we need to define connection with MongoDB and PostgreSQL. Then, we create step for each process. Last, we schedule a job to run this transformation pipeline every certain time.

#### Diagram
![image](https://user-images.githubusercontent.com/99194827/232277932-0e84eab2-e904-45b5-a0bc-3186612f8bb5.png)

#### Steps
| Name | Detail | Notes |
|------|--------|-------|
| Extract_Orders | Extract orders data from MongoDB | Connect to db_orders in superstore_db |
| Staging_Orders | Load orders data to PostgreSQL | Connect to staging.orders in superstore_dwh |
| Sort_cust_id_from_orders | Sort orders table based on customer_id | Preparation to join with customers table |
| Extract_Customers | Extract customers data from MongoDB | Connect to db_customers in superstore_db |
| Staging_Customers | Load orders data to PostgreSQL | Connect to staging.customers in superstore_dwh |
| Sort_cust_id_from_customers | Sort customers table based on customer_id | Preparation to join with orders table |
| First_Join | Inner join orders table with customers table based on customer_id | Redundant columns deleted |
| Sort_prod_id_from_first_join | Sort table from join process based on product_id | Preparation to join with products table |
| Extract_Products | Extract products data from MongoDB | Connect to db_products in superstore_db |
| Staging_Products | Load products data to PostgreSQL | Connect to staging.products in superstore_dwh
| Sort_prod_id_from_products | Sort products table based on product_id | Preparation to join with table from previous join process |
| Last_Join | Inner join the table from previous join process with products table | Redundant columns deleted |
| Sort_Rows | Sort result table based on order_date, row_id, order_id, customer_id, and product_id | So fact table looks good visually |
| Load_Fact | Load fact table to PostgreSQL | Connect to production.fact_table in superstore_dwh |
| Extract_for_daily_report | Extract data for daily report from fact_table with SQL query | SELECT <br>&emsp; order_date AS record_date, <br>&emsp; COUNT(sales) AS total_sales, <br>&emsp; SUM(sales - discount * sales) AS net_sales <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_date <br> ORDER BY record_date; |
| Load_daily_report | Load daily report data to PostgreSQL | Connect to mart.daily_report in superstore_dmart |
| Extract_for_monthly_report | Extract data for monthly report from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(MONTH FROM order_date) AS integer) AS record_month, <br>&emsp; COUNT(sales) AS total_sales, <br>&emsp; SUM(sales - discount * sales) AS net_sales <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_month <br> ORDER BY record_year, record_month; |
| Load_monthly_report | Load montly report data to PostgreSQL | Connect to mart.monthly_report in superstore_dmart |
| Extract_for_quarterly_report | Extract data for quarterly report from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(QUARTER FROM order_date) AS integer) AS record_quarter, <br>&emsp; COUNT(sales) AS total_sales, <br>&emsp; SUM(sales - discount * sales) AS net_sales <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_quarter <br> ORDER BY record_year, record_quarter; |
| Load_quarterly_report | Load quarterly report data to PostgreSQL | Connect to mart.quarterly_report in superstore_dmart |
| Extract_for_yearly_report | Extract data for yearly report from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; COUNT(sales) AS total_sales, <br>&emsp; SUM(sales - discount * sales) AS net_sales <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year <br> ORDER BY record_year; |
| Load_yearly_report | Load yearly report data to PostgreSQL | Connect to mart.yearly_report in superstore_dmart |
| Extract_for_actual_vs_budget | Extract data for actual and budget comparison | SELECT <br>&emsp; SUM(actual_value) AS total_actual_value, <br>&emsp; SUM(budget_value) AS total_budget_value, <br>&emsp; ((SUM(actual_value)/SUM(budget_value)) * 100) AS percentage <br> FROM (<br>&emsp; SELECT <br>&emsp;&emsp; record_year, <br>&emsp;&emsp; net_sales AS actual_value, <br>&emsp;&emsp; (LAG(net_sales) OVER (ORDER BY record_year) * 1.1) AS budget_value <br>&emsp; FROM (<br>&emsp;&emsp; SELECT <br>&emsp;&emsp;&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp;&emsp;&emsp; COUNT(sales) AS total_sales, <br>&emsp;&emsp;&emsp; SUM(sales - discount * sales) AS net_sales <br>&emsp;&emsp; FROM superstore_dwh.production.fact_table <br>&emsp;&emsp; GROUP BY record_year <br>&emsp;&emsp; ORDER BY record_year <br>&emsp; ) AS subsubquery <br> ) AS subquery;
| Load_actual_vs_budget | Load actual and budget comparison report to PostgreSQL | Connect to mart.actual_vs_budget in superstore_dmart |
| Extract_for_monthly_growth | Extract data for monthly growth from fact_table with SQL query | SELECT <br>&emsp; record_year, <br>&emsp; record_month, <br>&emsp; gross_sales, <br>&emsp; ((gross_sales - LAG(gross_sales) OVER (ORDER BY record_year, record_month))/LAG(gross_sales) OVER (ORDER BY record_year, record_month)) * 100 AS growth_percentage <br> FROM ( <br>&emsp; SELECT <br>&emsp;&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp;&emsp; CAST(EXTRACT(MONTH FROM order_date) AS integer) AS record_month, <br>&emsp;&emsp; SUM(sales) AS gross_sales <br>&emsp; FROM superstore_dwh.production.fact_table <br>&emsp; GROUP BY record_year, record_month <br> ) AS subquery <br> ORDER BY record_year, record_month; |
| Load_monthly_growth | Load monthly growth data to PostgreSQL | Connect to mart.monthly_growth in superstore_dmart |
| Extract_for_quarterly_growth | Extract data for quarterly growth from fact_table with SQL query | SELECT <br>&emsp; record_year, <br>&emsp; record_quarter, <br>&emsp; gross_sales, <br>&emsp; ((gross_sales - LAG(gross_sales) OVER (ORDER BY record_year, record_quarter))/LAG(gross_sales) OVER (ORDER BY record_year, record_quarter)) * 100 AS growth_percentage <br> FROM ( <br>&emsp; SELECT <br>&emsp;&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp;&emsp; CAST(EXTRACT(QUARTER FROM order_date) AS integer) AS record_quarter, <br>&emsp;&emsp; SUM(sales) AS gross_sales <br>&emsp; FROM superstore_dwh.production.fact_table <br>&emsp; GROUP BY record_year, record_quarter <br> ) AS subquery <br> ORDER BY record_year, record_quarter; |
| Load_quarterly_growth | Load quarterly growth data to PostgreSQL | Connect to mart.quarterly_growth in superstore_dmart |
| Extract_for_yearly_growth | Extract data for yearly growth from fact_table with SQL query | SELECT <br>&emsp; record_year, <br>&emsp; gross_sales, <br>&emsp; ((gross_sales - LAG(gross_sales) OVER (ORDER BY record_year))/LAG(gross_sales) OVER (ORDER BY record_year)) * 100 AS growth_percentage <br> FROM ( <br>&emsp; SELECT <br>&emsp;&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp;&emsp; SUM(sales) AS gross_sales <br>&emsp; FROM superstore_dwh.production.fact_table <br>&emsp; GROUP BY record_year <br> ) AS subquery <br> ORDER BY record_year; |
| Load_yearly_growth | Load yearly growth data to PostgreSQL | Connect to mart.yearly_growth in superstore_dmart |
| Extract_for_loss_report | Extract data for loss report from fact_table with SQL query | SELECT <br>&emsp; order_date AS record_date, <br>&emsp; city, <br>&emsp; region, <br>&emsp; product_id, <br>&emsp; product_category, <br>&emsp; product_subcategory, <br>&emsp; product_name, <br>&emsp; SUM(profit) AS loss <br> FROM superstore_dwh.production.fact_table <br> WHERE profit < 0 <br> GROUP BY record_date, city, region, product_id, product_category, product_subcategory, product_name <br> ORDER BY record_date; |
| Load_loss_report | Load loss report data to PostgreSQL | Connect to mart.loss_report in superstore_dmart |
| Extract_for_cust_segmentation | Extract data for customer segmentation from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(QUARTER FROM order_date) AS integer) AS record_quarter, <br>&emsp; customer_id, <br>&emsp; customer_name, <br>&emsp; customer_segment,  <br>&emsp; (CASE  <br>&emsp;&emsp; WHEN SUM(sales) < 200 THEN 'Bronze' <br>&emsp;&emsp; WHEN (SUM(sales) > 200) AND (SUM(sales) < 500) THEN 'Silver' <br>&emsp;&emsp; ELSE 'Gold' <br>&emsp; END) AS shopping_segment <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_quarter, customer_id, customer_name, customer_segment <br> ORDER BY record_year, record_quarter; |
| Load_cust_segmentation | Load customer segmentation data to PostgreSQL | Connect to mart.customer_segmentation in superstore_dmart |
| Extract_for_prod_segmentation | Extract data for product segmentation from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(MONTH FROM order_date) AS integer) AS record_month, <br>&emsp; city, <br>&emsp; region, <br>&emsp; product_id, <br>&emsp; product_category, <br>&emsp; product_subcategory, <br>&emsp; product_name, <br>&emsp; (CASE <br>&emsp;&emsp; WHEN SUM(quantity) < 5 THEN '3rd Product' <br>&emsp;&emsp; WHEN (SUM(quantity) > 5) AND (SUM(quantity) < 10) THEN '2nd Product' <br>&emsp;&emsp; ELSE '1st Product' <br>&emsp; END) AS product_segment <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_month, city, region, product_id, product_category, product_subcategory, product_name <br> ORDER BY record_year, record_month; |
| Load_prod_segmentation | Load product segmentation data to PostgreSQL | Connect to mart.product_segmentation in superstore_dmart |
| Extract_for_reg_segmentation | Extract data for region segmentation from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(MONTH FROM order_date) AS integer) AS record_month, <br>&emsp; city, <br>&emsp; region, <br>&emsp; (CASE <br>&emsp;&emsp; WHEN SUM(sales) < 1000 THEN 'Kategori I' <br>&emsp;&emsp; WHEN (SUM(sales) > 1000) AND (SUM(sales) < 2000) THEN 'Kategori II' <br>&emsp;&emsp; ELSE 'Kategori III' <br>&emsp; END) AS region_segment <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_month, city, region <br> ORDER BY record_year, record_month; |
| Load_reg_segmentation | Load region segmentation data to PostgreSQL | Connect to mart.region_segmentation in superstore_dmart |

#### Entitity Relationship Diagram
Data Warehouse in this project is modelled in Star Schema. Star Schema is a modelling for data warehouse which consists of a fact table built from two dimension tables or more. This schema often used for analysis that specific on a subject only.

### Access Request
There are few roles that should be given access to the visualization:
- Head of Marketing and Sales Department -> access to all city and region
- All of Branch Manager -> access to specific city only
- All of Region Manager -> access to specific region only
