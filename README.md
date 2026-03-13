# Bicycle Manufacturer Business Performance Analysis

## Table of Contents

* [Business Problem](#business-problem)
* [Dataset](#dataset)
* [Analysis](#analysis)
* [Insights](#insights)
* [Recommendations](#recommendations)

## Business Problem
A bicycle manufacturing company wants to better understand its business performance in order to support data-driven decision making. The management team needs insights into product performance, sales trends, customer retention, and inventory efficiency.

This project analyzes historical sales data to answer several key business questions:

* Which product subcategories generate the highest sales?

* Which product categories are growing the fastest year over year?

* Which sales territories generate the most orders?

* How effective are seasonal discounts?

* How well does the company retain customers?

* Is inventory aligned with product demand?

The goal of this analysis is to uncover meaningful insights that can help improve sales performance, optimize inventory, and support strategic decision making.

## Dataset
This project uses the AdventureWorks2019 dataset, a widely used sample database representing a bicycle manufacturing company.

The dataset contains multiple related tables covering sales transactions, product information, customers, and purchasing activities.

Key tables used in this analysis include:

* Sales.SalesOrderDetail

* Sales.SalesOrderHeader

* Production.Product

* Production.ProductSubcategory

* Sales.SpecialOffer

* Production.WorkOrder

* Purchasing.PurchaseOrderHeader

## Analysis
The analysis focuses on answering key business questions using SQL queries in BigQuery.

The full SQL analysis can be viewed [here]().

**1. Calculate total quantity sold, total sales value, and order count by product subcategory over the last 12 months.**

```sql
with raw_data as(
  SELECT 
    FORMAT_DATE('%b %Y',a.ModifiedDate) period
    ,extract(year from a.ModifiedDate) year   -- lấy để sắp xếp 
    ,extract(month from a.ModifiedDate) month -- lấy để sắp xếp 
    ,c.name
    ,sum(a.OrderQty) qty_item
    ,sum(a.LineTotal) total_sales
    ,count(distinct a.SalesOrderID) order_cnt
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  left join `adventureworks2019.Production.Product` b
    on a.ProductID=b.ProductID
  left join `adventureworks2019.Production.ProductSubcategory` c
    on cast(b.ProductSubcategoryID as int)=c.ProductSubcategoryID --2 key khác kiểu dữ liệu
  where date(a.ModifiedDate) >= (select  -- date trước ngày lớn nhất 12 tháng
                                    date_sub(max(date(ModifiedDate)), interval 12 month) b12m -- 2013-06-30
                                from `adventureworks2019.Sales.SalesOrderDetail`)  
  group by 1,2,3,4
  order by 2,3,4     -- sắp xếp theo thời gian tăng tăng dần: tháng 6 năm 2013-> tháng 6 năm 2014
) 

select
  period
  ,name
  ,qty_item
  ,total_sales
  ,order_cnt
from raw_data;
```

| period | name | qty_item | total_sales | order_cnt | 
|------|------|------|------| ------| 
| Jun 2013	| Bib-Shorts	| 2	| 116.987 | 1 |
| Jun 2013	| Bike Racks	| 363	| 24684.000000000004 | 57 |
| Jun 2013	| Bike Stands	| 2	| 318.0	| 2 |
| Jun 2013	| Bottles and Cages	| 356	| 1071.1076000000005 | 64 |

**2. Calculate YoY growth rate in sales quantity for each product subcategory and identify the top 3 fastest-growing categories.**

```sql
with 
present_year as(
  SELECT 
    extract(year from a.ModifiedDate) year    
    ,c.name
    ,sum(a.OrderQty) qty_item
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  left join `adventureworks2019.Production.Product` b
    on a.ProductID=b.ProductID
  left join `adventureworks2019.Production.ProductSubcategory` c
    on cast(b.ProductSubcategoryID as int)=c.ProductSubcategoryID 
  group by 1,2
  order by 1,2
),

call_qty_diff as(
  select 
    year       -- trường year để biết năm hiện tại
    ,name
    ,qty_item
    ,lag(qty_item)over(partition by name order by year) prev_qty
    ,qty_item/lag(qty_item)over(partition by name order by year) -1 qty_diff
    -- nếu tính %yoy thì kết quả phải*100. Ví dụ Mountain Frames năm 2012 tăng trưởng 521.18% so với năm trước đó.
  from present_year
),

ranking_qty_diff as(
  select              
    name
    ,qty_item
    ,prev_qty
    ,round(qty_diff,2) qty_diff
    ,dense_rank()over(order by qty_diff desc) dk 
  from call_qty_diff
  where prev_qty is not null
  order by dk
)

select
  name
  ,qty_item
  ,prev_qty
  ,qty_diff 
from ranking_qty_diff 
where dk<=3  --top 3 cat with highest grow rate
order by dk;
```

| name | qty_item | prev_qty | qty_diff | 
|------|------|------|------| 
| Mountain Frames	| 3168	| 510	| 5.21 |
| Socks	| 2724	| 523	| 4.21 |
| Road Frames	| 5564	| 1137	| 3.89 |

**3. Rank the top 3 territories with the highest order quantity each year.**

```sql
with 
qty_territory as(
  SELECT 
    extract(year from a.ModifiedDate) year    
    ,b.TerritoryID
    ,sum(a.OrderQty) qty_item
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  left join `adventureworks2019.Sales.SalesOrderHeader` b
    on a.SalesOrderID=b.SalesOrderID
  group by 1,2
  order by 1,2
),

ranking_qty as(
  select 
    year    
    ,TerritoryID
    ,qty_item
    ,dense_rank()over(partition by year order by qty_item desc) dk 
  from qty_territory
  order by year, dk
)

select 
  year    
  ,TerritoryID
  ,qty_item
  ,dk
from ranking_qty
where dk<=3;
```

| year | TerritoryID | qty_item | dk | 
|------|------|------|------|
| 2011	| 4	| 3238	| 1 | 
| 2011	| 6	| 2705	| 2 | 
| 2011	| 1	| 1964	| 3 | 
| 2012	| 4	| 17553	| 1 |
| 2012	| 6	| 14412	| 2 |
| 2012	| 1	| 8537	| 3 |
| 2013	| 4	| 26682	| 1 |
| 2013	| 6	| 22553	| 2 |
| 2013	| 1	| 17452	| 3 |
| 2014	| 4	| 11632	| 1 |
| 2014	| 6	| 9711	| 2 |
| 2014	| 1	| 8823	| 3 |

**4. Calculate the total cost of seasonal discounts by product subcategory.**


**5. Analyze customer retention in 2014 by identifying each customer's first purchase month and measuring the percentage of customers who return in subsequent months.**


**6. Analyze stock levels by product and calculate Month-over-Month (MoM) growth in inventory levels.**

```sql

```

**7. Calculate the ratio of stock quantity to sales quantity by product and month.**

```sql
```

**8. Calculate the number and total value of purchase orders with pending status in 2014..**

```sql

```

## Insights

*

## Recommendations

*  
