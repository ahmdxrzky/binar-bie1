# Project Overview
This project is a Business Intelligence Project that creates dashboard to give monitorization to Marketing and Sales Department upon product sales, region performance, and target achievement periodically. By finishing this project, we hope we can replace project consolidation process that done manually along the time and provide a dashboard that accessible by all of the region manager to monitor and evaluate performance of their region.

# Project Objectives
Objective of creation of this dashboard is to give end-to-end visualization to Marketing and Sales Department upon product sales in all region and also to give monitorization privilege about sales in certain period.

# Technical Design Diagram
![tdd](https://user-images.githubusercontent.com/99194827/232273568-569e5e0b-fed9-4bcc-98e8-4e2c56f7e13e.png)

### Components
MongoDB and PostgreSQL are activated with this docker-compose [file](https://github.com/ahmdxrzky/binar-bie1/blob/main/docker-compose.yaml).
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
Data Transformation in this project done with **Pentaho Data Integration** software. We can define all pipeline from extract data from data lake until load data to data marts in a single execution. First, we need to define connection with MongoDB and PostgreSQL. Then, we create step for each process in a [Pentaho transformation diagram](https://github.com/ahmdxrzky/binar-bie1/blob/main/bie1_project.ktr). Last, we schedule a job to run this transformation pipeline every certain time. We can use **Pentaho Server** if we want to run Pentaho job from the app UI or we can utilized common scheduler, such as cron or PM2.

#### Diagram
![image](https://user-images.githubusercontent.com/99194827/232277932-0e84eab2-e904-45b5-a0bc-3186612f8bb5.png)

#### Steps
| Name | Step Title | Detail | Notes |
|------|------------|--------|-------|
| Extract_Orders | MongoDB input | Extract orders data from MongoDB | Connect to db_orders in superstore_db |
| Staging_Orders | Table output | Load orders data to PostgreSQL | Connect to staging.orders in superstore_dwh |
| Sort_cust_id_from_orders | Sort rows | Sort orders table based on customer_id | Preparation to join with customers table |
| Extract_Customers | MongoDB input | Extract customers data from MongoDB | Connect to db_customers in superstore_db |
| Staging_Customers | Table output | Load orders data to PostgreSQL | Connect to staging.customers in superstore_dwh |
| Sort_cust_id_from_customers | Sort rows | Sort customers table based on customer_id | Preparation to join with orders table |
| First_Join | Merge join | Inner join orders table with customers table based on customer_id | Redundant columns deleted |
| Sort_prod_id_from_first_join | Sort rows | Sort table from join process based on product_id | Preparation to join with products table |
| Extract_Products | MongoDB input | Extract products data from MongoDB | Connect to db_products in superstore_db |
| Staging_Products | Table output | Load products data to PostgreSQL | Connect to staging.products in superstore_dwh
| Sort_prod_id_from_products | Sort rows | Sort products table based on product_id | Preparation to join with table from previous join process |
| Last_Join | Merge join | Inner join the table from previous join process with products table | Redundant columns deleted |
| Sort_Rows | Sort rows | Sort result table based on order_date, row_id, order_id, customer_id, and product_id | So fact table looks good visually |
| Load_Fact | Table output | Load fact table to PostgreSQL | Connect to production.fact_table in superstore_dwh |
| Extract_for_daily_report | Table input | Extract data for daily report from fact_table with SQL query | SELECT <br>&emsp; order_date AS record_date, <br>&emsp; COUNT(sales) AS total_sales, <br>&emsp; SUM(sales - discount * sales) AS net_sales <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_date <br> ORDER BY record_date; |
| Load_daily_report | Table output | Load daily report data to PostgreSQL | Connect to mart.daily_report in superstore_dmart |
| Extract_for_monthly_report | Table input | Extract data for monthly report from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(MONTH FROM order_date) AS integer) AS record_month, <br>&emsp; COUNT(sales) AS total_sales, <br>&emsp; SUM(sales - discount * sales) AS net_sales <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_month <br> ORDER BY record_year, record_month; |
| Load_monthly_report | Table output | Load montly report data to PostgreSQL | Connect to mart.monthly_report in superstore_dmart |
| Extract_for_quarterly_report | Table input | Extract data for quarterly report from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(QUARTER FROM order_date) AS integer) AS record_quarter, <br>&emsp; COUNT(sales) AS total_sales, <br>&emsp; SUM(sales - discount * sales) AS net_sales <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_quarter <br> ORDER BY record_year, record_quarter; |
| Load_quarterly_report | Table output | Load quarterly report data to PostgreSQL | Connect to mart.quarterly_report in superstore_dmart |
| Extract_for_yearly_report | Table input | Extract data for yearly report from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; COUNT(sales) AS total_sales, <br>&emsp; SUM(sales - discount * sales) AS net_sales <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year <br> ORDER BY record_year; |
| Load_yearly_report | Table output | Load yearly report data to PostgreSQL | Connect to mart.yearly_report in superstore_dmart |
| Extract_for_actual_vs_budget | Table input | Extract data for actual and budget comparison | SELECT <br>&emsp; SUM(actual_value) AS total_actual_value, <br>&emsp; SUM(budget_value) AS total_budget_value, <br>&emsp; ((SUM(actual_value)/SUM(budget_value)) * 100) AS percentage <br> FROM (<br>&emsp; SELECT <br>&emsp;&emsp; record_year, <br>&emsp;&emsp; net_sales AS actual_value, <br>&emsp;&emsp; (LAG(net_sales) OVER (ORDER BY record_year) * 1.1) AS budget_value <br>&emsp; FROM (<br>&emsp;&emsp; SELECT <br>&emsp;&emsp;&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp;&emsp;&emsp; COUNT(sales) AS total_sales, <br>&emsp;&emsp;&emsp; SUM(sales - discount * sales) AS net_sales <br>&emsp;&emsp; FROM superstore_dwh.production.fact_table <br>&emsp;&emsp; GROUP BY record_year <br>&emsp;&emsp; ORDER BY record_year <br>&emsp; ) AS subsubquery <br> ) AS subquery;
| Load_actual_vs_budget | Table output | Load actual and budget comparison report to PostgreSQL | Connect to mart.actual_vs_budget in superstore_dmart |
| Extract_for_monthly_growth | Table input | Extract data for monthly growth from fact_table with SQL query | SELECT <br>&emsp; record_year, <br>&emsp; record_month, <br>&emsp; gross_sales, <br>&emsp; ((gross_sales - LAG(gross_sales) OVER (ORDER BY record_year, record_month))/LAG(gross_sales) OVER (ORDER BY record_year, record_month)) * 100 AS growth_percentage <br> FROM ( <br>&emsp; SELECT <br>&emsp;&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp;&emsp; CAST(EXTRACT(MONTH FROM order_date) AS integer) AS record_month, <br>&emsp;&emsp; SUM(sales) AS gross_sales <br>&emsp; FROM superstore_dwh.production.fact_table <br>&emsp; GROUP BY record_year, record_month <br> ) AS subquery <br> ORDER BY record_year, record_month; |
| Load_monthly_growth | Table output | Load monthly growth data to PostgreSQL | Connect to mart.monthly_growth in superstore_dmart |
| Extract_for_quarterly_growth | Table input | Extract data for quarterly growth from fact_table with SQL query | SELECT <br>&emsp; record_year, <br>&emsp; record_quarter, <br>&emsp; gross_sales, <br>&emsp; ((gross_sales - LAG(gross_sales) OVER (ORDER BY record_year, record_quarter))/LAG(gross_sales) OVER (ORDER BY record_year, record_quarter)) * 100 AS growth_percentage <br> FROM ( <br>&emsp; SELECT <br>&emsp;&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp;&emsp; CAST(EXTRACT(QUARTER FROM order_date) AS integer) AS record_quarter, <br>&emsp;&emsp; SUM(sales) AS gross_sales <br>&emsp; FROM superstore_dwh.production.fact_table <br>&emsp; GROUP BY record_year, record_quarter <br> ) AS subquery <br> ORDER BY record_year, record_quarter; |
| Load_quarterly_growth | Table output | Load quarterly growth data to PostgreSQL | Connect to mart.quarterly_growth in superstore_dmart |
| Extract_for_yearly_growth | Table input | Extract data for yearly growth from fact_table with SQL query | SELECT <br>&emsp; record_year, <br>&emsp; gross_sales, <br>&emsp; ((gross_sales - LAG(gross_sales) OVER (ORDER BY record_year))/LAG(gross_sales) OVER (ORDER BY record_year)) * 100 AS growth_percentage <br> FROM ( <br>&emsp; SELECT <br>&emsp;&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp;&emsp; SUM(sales) AS gross_sales <br>&emsp; FROM superstore_dwh.production.fact_table <br>&emsp; GROUP BY record_year <br> ) AS subquery <br> ORDER BY record_year; |
| Load_yearly_growth | Table output | Load yearly growth data to PostgreSQL | Connect to mart.yearly_growth in superstore_dmart |
| Extract_for_loss_report | Table input | Extract data for loss report from fact_table with SQL query | SELECT <br>&emsp; order_date AS record_date, <br>&emsp; city, <br>&emsp; region, <br>&emsp; product_id, <br>&emsp; product_category, <br>&emsp; product_subcategory, <br>&emsp; product_name, <br>&emsp; SUM(profit) AS loss <br> FROM superstore_dwh.production.fact_table <br> WHERE profit < 0 <br> GROUP BY record_date, city, region, product_id, product_category, product_subcategory, product_name <br> ORDER BY record_date; |
| Load_loss_report | Table output | Load loss report data to PostgreSQL | Connect to mart.loss_report in superstore_dmart |
| Extract_for_cust_segmentation | Table input | Extract data for customer segmentation from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(QUARTER FROM order_date) AS integer) AS record_quarter, <br>&emsp; customer_id, <br>&emsp; customer_name, <br>&emsp; customer_segment,  <br>&emsp; (CASE  <br>&emsp;&emsp; WHEN SUM(sales) < 200 THEN 'Bronze' <br>&emsp;&emsp; WHEN (SUM(sales) > 200) AND (SUM(sales) < 500) THEN 'Silver' <br>&emsp;&emsp; ELSE 'Gold' <br>&emsp; END) AS shopping_segment <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_quarter, customer_id, customer_name, customer_segment <br> ORDER BY record_year, record_quarter; |
| Load_cust_segmentation | Table output | Load customer segmentation data to PostgreSQL | Connect to mart.customer_segmentation in superstore_dmart |
| Extract_for_prod_segmentation | Table input | Extract data for product segmentation from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(MONTH FROM order_date) AS integer) AS record_month, <br>&emsp; city, <br>&emsp; region, <br>&emsp; product_id, <br>&emsp; product_category, <br>&emsp; product_subcategory, <br>&emsp; product_name, <br>&emsp; (CASE <br>&emsp;&emsp; WHEN SUM(quantity) < 5 THEN '3rd Product' <br>&emsp;&emsp; WHEN (SUM(quantity) > 5) AND (SUM(quantity) < 10) THEN '2nd Product' <br>&emsp;&emsp; ELSE '1st Product' <br>&emsp; END) AS product_segment <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_month, city, region, product_id, product_category, product_subcategory, product_name <br> ORDER BY record_year, record_month; |
| Load_prod_segmentation | Table output | Load product segmentation data to PostgreSQL | Connect to mart.product_segmentation in superstore_dmart |
| Extract_for_reg_segmentation | Table input | Extract data for region segmentation from fact_table with SQL query | SELECT <br>&emsp; CAST(EXTRACT(YEAR FROM order_date) AS integer) AS record_year, <br>&emsp; CAST(EXTRACT(MONTH FROM order_date) AS integer) AS record_month, <br>&emsp; city, <br>&emsp; region, <br>&emsp; (CASE <br>&emsp;&emsp; WHEN SUM(sales) < 1000 THEN 'Kategori I' <br>&emsp;&emsp; WHEN (SUM(sales) > 1000) AND (SUM(sales) < 2000) THEN 'Kategori II' <br>&emsp;&emsp; ELSE 'Kategori III' <br>&emsp; END) AS region_segment <br> FROM superstore_dwh.production.fact_table <br> GROUP BY record_year, record_month, city, region <br> ORDER BY record_year, record_month; |
| Load_reg_segmentation | Table output | Load region segmentation data to PostgreSQL | Connect to mart.region_segmentation in superstore_dmart |

#### Entitity Relationship Diagram
Data Warehouse in this project is modelled in Star Schema. Star Schema is a modelling for data warehouse which consists of a fact table built from two dimension tables or more. This schema often used for analysis that specific on a subject only.
![star_schema drawio](https://user-images.githubusercontent.com/99194827/232319171-e8e8ce03-8eef-4983-9c0f-03271f08629b.png)

### Access Request
There are few roles that should be given access to the visualization:
- Head of Marketing and Sales Department -> access to all city and region
- All of Branch Manager -> access to specific city only
- All of Region Manager -> access to specific region only

# Resources Requirement
### Department & Staff
People that involved in building this project:
- Head of Marketing and Sales Department
- Business Intelligence Engineer
- Business Intelligence Analyst

### Technical Dependencies
- Pentaho job to process data from Data Lake to Data Marts will start to be executed at 8 AM GMT+7 in each day.
- Customers' identity are all valid and complete.
- Products' detail are complete, but there are 32 invalid data. These data said to be invalid because they have same product_id with 32 other items (a product_id refers to two different products). After further confirmation to Head of Marketing and Sales Department, for each product_id, the valid one is the newer product. So, we can delete old product for each product_id. This elimination stage is [not included](https://github.com/ahmdxrzky/binar-bie1/blob/main/ingest_raw.ipynb) in Pentaho job.

### Non-Technical Dependencies
Anomaly on data has being communicated with user, which is Head of Marketing and Sales Department).

# Dashboard
All data marts are visualized in 4 different dashboards (based on timeframe) using **Tableau Desktop**. It can be shared to other stakeholders if we deployed the dashboard to **Tableau Server**. File for workbook and dashboard layout is [here](https://github.com/ahmdxrzky/binar-bie1/blob/main/bie1_project.twb).

### Daily Dashboard
#### Initial (no filter)
![Screenshot (70)](https://user-images.githubusercontent.com/99194827/232369887-6535c601-1208-49af-a0ea-4af4059923c3.png)

#### Apply filter "Date"
![Screenshot (71)](https://user-images.githubusercontent.com/99194827/232369938-759d4fa7-d0d0-4e79-b03d-9574da07feac.png)
On November 24th, 2014, there are **19** sales that produce net sales equal to **4206.97** US dollars.

#### Apply filter "City" and "Region"
![Screenshot (73)](https://user-images.githubusercontent.com/99194827/232369953-6b6a494c-4dda-4c53-93f3-ca351685de74.png)
On November 24th, 2014, there is a region in Columbus city which located in **East Columbus**. This region has **2** products that produce loss.

### Monthly Dashboard
#### Initial (no filter)
![Screenshot (75)](https://user-images.githubusercontent.com/99194827/232369961-d56f77e4-287d-412c-a645-102746ac22bf.png)

#### Apply filter "Year", "Month", and "Date Range"
![Screenshot (79)](https://user-images.githubusercontent.com/99194827/232369970-769adca0-51ef-4a01-90fc-53fe3f1f8ea6.png)
On September 2015, there are **293** sales that produce net sales equal to **56915.14** US dollars. Compared to August 2015 (the previous month), this month has growth percentage equal to **75.06%**.

#### Apply filter "City"
![Screenshot (80)](https://user-images.githubusercontent.com/99194827/232369974-4bf45558-f076-40c4-974c-db103855849b.png)
On September 2015, there are **2** region (67%) in Columbus city that belong to "Kategori I" and **1** region (33%) belongs to "Kategori II".

#### Apply filter "Region"
![Screenshot (81)](https://user-images.githubusercontent.com/99194827/232369984-f4b00897-c65c-493b-b4b3-424bab3c85f8.png)
On September 2015, East Columbus region which belongs to "Kategori I" has **5** different products with their categories. 1 product belongs to "1st Product" and "2nd Product", respectively and the other 3 products belong to "3rd Product" (20%:20%:60%).

### Quarterly Dashboard
#### Initial (no filter)
![Screenshot (85)](https://user-images.githubusercontent.com/99194827/232369991-d1757272-f184-4e5e-ad65-964efb496725.png)

#### Apply filter "Year", "Quarter", and "Month Range"
![Screenshot (86)](https://user-images.githubusercontent.com/99194827/232370004-00e3976d-c86e-4057-9106-04313bfa706e.png)
On 3rd Quarter of 2015, there are **592** sales that produce net sales equal to **113725.88** US dollars. Compared to 2nd Quarter of 2015 (the previous quarter), this quarter has growth percentage equal to **46.16%**.

#### Apply filter "Customer Segment"
![Screenshot (87)](https://user-images.githubusercontent.com/99194827/232370011-13522396-8622-4de4-866e-0b47db9aedca.png)
On 3rd Quarter of 2015, there are many customers that belong to "Gold" category. We should consider to give them bonus to maintain their loyalty to the company.

### Yearly Dashboard
#### Initial (no filter)
![Screenshot (90)](https://user-images.githubusercontent.com/99194827/232370018-8167927c-3a24-48fe-b4fb-578cd0e3b3f5.png)
In total (not based on the year), this company has growth percentage from 2014 to 2017 equal to **133.77%**. Its actual sales value equal to **1.972 million** US dollars with budget (target) sales value equal to **1.474 million** US dollars.

#### Apply filter "Year"
![Screenshot (91)](https://user-images.githubusercontent.com/99194827/232370024-b117435e-ca12-4a62-8832-162170f470c0.png)
On 2015, there are **2102** sales that produce net sales equal to **407671.32** US dollars. Compared to 2014 (the previous year), this year has growth percentage equal to **53.88%**.
