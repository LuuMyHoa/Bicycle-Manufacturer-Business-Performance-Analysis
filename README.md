# Bicycle Manufacturer Business Performance Analysis

![image](Gemini_Generated_Image_02.png).

## Table of Contents

* [Business Problem](#business-problem)
* [Dataset](#dataset)
* [Analysis](#analysis)
* [Conclusion](#conclusion)

## Business Problem
A bicycle manufacturing company wants clearer visibility into its business performance to support better operational and strategic decisions. The management team needs insights into product performance, sales trends, customer retention, and inventory efficiency.

Using historical data from the AdventureWorks dataset, this project uses SQL to analyze key aspects of the business, including sales performance, product growth, market demand, customer behavior, inventory management, and purchasing activity.

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

```sql
SELECT 
  extract(year from a.ModifiedDate) year 
  ,c.name
  ,sum(d.DiscountPct * a.UnitPrice * a.OrderQty) total_cost
  --Discount Cost = Disct Pct * Unit Price * Item Qty
FROM `adventureworks2019.Sales.SalesOrderDetail` a
left join `adventureworks2019.Production.Product` b
  on a.ProductID=b.ProductID
left join `adventureworks2019.Production.ProductSubcategory` c
  on cast(b.ProductSubcategoryID as int)=c.ProductSubcategoryID
left join `adventureworks2019.Sales.SpecialOffer` d
  on a.SpecialOfferID=d.SpecialOfferID
where lower(d.Type) like '%seasonal discount%' 
group by 1,2
order by 1,2;
```

| year | name | total_cost |
|------|------|------|
| 2012	| Helmets	| 827.64732 | 
| 2013	| Helmets	| 1606.041 | 

**5. Analyze customer retention in 2014 by identifying each customer's first purchase month and measuring the percentage of customers who return in subsequent months.**

```sql
/* 
Step 1: Tìm danh sách khách hàng và tháng đầu tiên (month_join) khách đó mua hàng (cte first_order)
Step 2: Đếm số khách group by month_join, month_diff = month_curr-month_join (mapping cte step 1 và bảng gốc)
Step 3: Dùng window function first_value() để lấy ra total_cus từng month_join -> tính % retention_rate
*/
with
first_order as(
  SELECT
    CustomerID
    ,extract(month from min(ModifiedDate)) month_join
  FROM `adventureworks2019.Sales.SalesOrderHeader`
  where CustomerID in (SELECT -- danh sách khách hàng mua lần đầu tiên
                        distinct CustomerID 
                      FROM `adventureworks2019.Sales.SalesOrderHeader`)
    and extract(year from ModifiedDate) =2014
    and Status =5 --shipped
  group by 1
),

months_data AS ( -- các tháng sau tháng đầu
  SELECT
    month_join
    ,extract(month from ModifiedDate) curr_month
    ,(extract(month from ModifiedDate)- month_join) month_diff
    ,count(distinct f.CustomerID) cnt_cus
  FROM first_order f
  left join `adventureworks2019.Sales.SalesOrderHeader` a
    on f.CustomerID = a.CustomerID
  where extract(year from ModifiedDate) =2014
  group by 1,2,3
)

select 
  month_join
  ,format('M-%d',month_diff) month_diff
  ,cnt_cus
  ,first_value(cnt_cus)over(partition by month_join order by month_diff) total_cus
  ,round(cnt_cus*100/first_value(cnt_cus)over(partition by month_join order by month_diff),2) retention_rate
from months_data
order by 1,2;
```

| month_join | month_diff | cnt_cus | total_cus | retention_rate |
|-------------|------------|--------|-----------|----------------|
| 1 | M-0 | 2076 | 2076 | 100 |
| 1 | M-1 | 78 | 2076 | 3.76 |
| 1 | M-2 | 89 | 2076 | 4.29 |
| 1 | M-3 | 252 | 2076 | 12.14 |
| 1 | M-4 | 96 | 2076 | 4.62 |
| 1 | M-5 | 61 | 2076 | 2.94 |
| 1 | M-6 | 18 | 2076 | 0.87 |
| 2 | M-0 | 1805 | 1805 | 100 |
| 2 | M-1 | 51 | 1805 | 2.83 |
| 2 | M-2 | 61 | 1805 | 3.38 |

**6. Analyze stock levels by product and calculate Month-over-Month (MoM) growth in inventory levels.**

```sql
with
call_stock_current as(
  SELECT 
    a.Name
    ,extract(year from w.ModifiedDate) year
    ,extract(month from w.ModifiedDate) month
    ,sum(w.StockedQty) stock_current
  FROM `adventureworks2019.Production.Product` a
  left join `adventureworks2019.Production.WorkOrder` w
    on a.ProductID=w.ProductID
  where extract(year from w.ModifiedDate) =2011
  group by 1,2,3
),

call_MoM as(
  select
    Name
    ,year
    ,month
    ,stock_current
    ,lead(stock_current)over(partition by Name, year order by month desc) stock_prv
    ,(stock_current/lag(stock_current)over(partition by Name, year order by month)-1)*100 MoM_rate
  from call_stock_current
)

select
  Name
  ,year
  ,month
  ,stock_current
  ,stock_prv
  ,round(coalesce(MoM_rate ,0) ,1) as MoM_rate
from call_MoM
order by 1,2,3 desc;
```

| Name | year | month | stock_current | stock_prv | MoM_rate |
|------|------|------|---------------|-----------|----------|
| BB Ball Bearing | 2011 | 12 | 8475 | 14544 | -41.7 |
| BB Ball Bearing | 2011 | 11 | 14544 | 19175 | -24.2 |
| BB Ball Bearing | 2011 | 10 | 19175 | 8845 | 116.8 |
| BB Ball Bearing | 2011 | 9 | 8845 | 9666 | -8.5 |
| BB Ball Bearing | 2011 | 8 | 9666 | 12837 | -24.7 |
| BB Ball Bearing | 2011 | 7 | 12837 | 5259 | 144.1 |
| BB Ball Bearing | 2011 | 6 | 5259 | null | 0.0 |
| Blade | 2011 | 12 | 1842 | 3598 | -48.8 |

**7. Calculate the ratio of stock quantity to sales quantity by product and month.**

```sql
with 
sale_info as(
  SELECT 
    extract(month from a.ModifiedDate) month
    ,extract(year from a.ModifiedDate) year
    ,a.ProductID
    ,b.Name
    ,sum(a.OrderQty) sales
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  left join `adventureworks2019.Production.Product` b
    on a.ProductID=b.ProductID
  where extract(year from a.ModifiedDate) =2011
  group by 1,2,3,4
),

stock_info as(
  SELECT 
    extract(month from ModifiedDate) month
    ,extract(year from ModifiedDate) year
    ,ProductID
    ,sum(StockedQty) stock
  FROM `adventureworks2019.Production.WorkOrder` 
  where extract(year from ModifiedDate) =2011
  group by 1,2,3
)

select 
  a.*
  ,stock
  ,round(coalesce(stock, 0)/sales,2) ratio
from sale_info a
full join stock_info b
  on a.ProductID=b.ProductID
  and a.month=b.month
  and a.year=b.year
order by month desc,ratio desc;
```

| month | year | ProductID | Name | sales | stock | ratio |
|------|------|-----------|------------ |-----------|-----------|------------|
| 12 | 2011 | 745 | HL Mountain Frame - Black, 48 | 1 | 27 | 27.0 |
| 12 | 2011 | 743 | HL Mountain Frame - Black, 42 | 1 | 26 | 26.0 |
| 12 | 2011 | 748 | HL Mountain Frame - Silver, 38 | 2 | 32 | 16.0 |
| 12 | 2011 | 722 | LL Road Frame - Black, 58 | 4 | 47 | 11.75 |
| 12 | 2011 | 747 | HL Mountain Frame - Black, 38 | 3 | 31 | 10.33 |
| 12 | 2011 | 726 | LL Road Frame - Red, 48 | 5 | 36 | 7.2 |

**8. Calculate the number and total value of purchase orders with pending status in 2014..**

```sql
SELECT 
  extract(year from ModifiedDate) year
  ,Status
  ,count(distinct PurchaseOrderID) order_cnt 
  ,sum(TotalDue) value 
FROM `adventureworks2019.Purchasing.PurchaseOrderHeader` 
where Status =1 --Pending status
  and extract(year from ModifiedDate) = 2014
group by 1,2;
```
| year | status | order_cnt | value |
|------|--------|-----------|-------|
| 2014 | 1 | 224 | 3873579.0123000029 |

## Conclusion
-


