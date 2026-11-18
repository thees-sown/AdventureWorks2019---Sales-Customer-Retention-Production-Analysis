# 💻 AdventureWorks2019 — Sales, Customer Retention & Production Analysis

**Tools Used:** SQL · Google BigQuery

## 📋 Table of Contents

1. Background & Overview
2. Dataset Description & Data Structure
3. Main Process
4. Conclusion & Recommendations

## 📌 1 - Background & Overview

This project uses eight SQL queries to examine sales performance, customer repeat activity, production output, and pending purchase orders in AdventureWorks2019. It is intended for analysts and stakeholders in sales, production planning, and procurement.

The analysis asks which product subcategories and territories drive order volume, how seasonal discounts affect sales lines, when customers return after a shipped order, and how quantities stocked through work orders compare with quantities sold.

## 📂 2 - Dataset Description & Data Structure

**Data source:** The queries reference AdventureWorks2019 tables in a BigQuery project named `adventureworks2019`. The dataset size and import method were not supplied.

| Dataset | Table | Role in the analysis |
|---|---|---|
| `Sales` | `SalesOrderDetail` | Sales order lines, quantities, prices, and line totals |
| `Sales` | `SalesOrderHeader` | Order dates, customers, territories, and status |
| `Sales` | `SpecialOffer` | Offer types and discount percentages |
| `Production` | `Product` | Product names and subcategory IDs |
| `Production` | `ProductSubcategory` | Product subcategory names |
| `Production` | `WorkOrder` | Stocked quantities and work-order end dates |
| `Purchasing` | `PurchaseOrderHeader` | Purchase order dates, statuses, and totals due |

To reproduce the analysis, open BigQuery, verify access to these tables, replace the project identifier if needed, and run the queries separately. The SQL below preserves the supplied logic. No query outputs were provided, so each result section describes the expected output rather than reporting findings.

## ⚒️ 3 - Main Process

### 1 - Sales by subcategory in the latest 12 months

Track units ordered, sales value, and distinct sales orders by subcategory and month. The cutoff is 12 months before the latest line modification date, not 12 complete calendar months or a window based on order dates.

#### 🖊️ Query

```sql
-- Calc Quantity of items, Sales value & Order quantity by each Subcategory in L12M
select format_datetime('%b %Y', od.ModifiedDate) month
      ,ps.Name
      ,sum(od.OrderQty) qty_item
      ,sum(od.LineTotal) total_sales
      ,count(distinct od.SalesOrderID) order_cnt
FROM `adventureworks2019.Sales.SalesOrderDetail` od
left join `adventureworks2019.Production.Product` p
  on od.ProductID = p.ProductID
left join `adventureworks2019.Production.ProductSubcategory` ps
  on p.ProductSubcategoryID = cast(ps.ProductSubcategoryID as string)

where date(od.ModifiedDate) >=  (select date_sub(date(max(od.ModifiedDate)), INTERVAL 12 month)
                                from `adventureworks2019.Sales.SalesOrderDetail` )
group by 1,2
order by 2,1
```

#### 💡 Query Results

<img width="503" height="285" alt="1" src="https://github.com/user-attachments/assets/4219f36a-5568-4e1c-bac1-4f90b383c23f" />

### 2 - Top three subcategories by intended year-over-year unit growth

Identify the subcategories with the largest year-over-year changes in units ordered.

#### 🖊️ Query

```sql
-- Calc % YoY growth rate by SubCategory & release top 3 cat with highest grow rate. Can use metric: quantity_item. Round results to 2 decimal
-- qty_diff = qty_item / prv_qty - 1
with subcat_growth as (
select
  sc.name as subcategory_name
  ,sum(d.OrderQty) as qty_item
  ,lag(sum(d.OrderQty)) over (
    order by min(extract(year from h.OrderDate))
  ) as prev_qty
  ,round(sum(d.OrderQty) / lag(sum(d.OrderQty)) over (
    order by min(extract(year from h.OrderDate))
  ), 2) - 1 as qty_diff
FROM `adventureworks2019.Sales.SalesOrderDetail` d
join `adventureworks2019.Sales.SalesOrderHeader` h
  on d.SalesOrderID = h.salesOrderID
join `adventureworks2019.Production.Product` p
  on d.ProductID = p.ProductID
join `adventureworks2019.Production.ProductSubcategory` sc
  on safe_cast(p.ProductSubcategoryID as string) = safe_cast(sc.ProductSubcategoryID as string)
group by sc.name
)
select
  *
from subcat_growth
qualify
  dense_rank() over (
    order by qty_diff desc
  ) <= 3
```

#### 💡 Query Results

<img width="413" height="125" alt="2" src="https://github.com/user-attachments/assets/e0237501-f1d4-4a9d-8f8f-3d93fe31707d" />

### 3 - Top three territories by annual order volume

Rank territories by distinct sales order count in each year, without skipping ranks after ties.

#### 🖊️ Query

```sql
-- Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number
select
  extract(year from h.OrderDate) as yr
  ,h.TerritoryID
  ,count(distinct d.SalesOrderID) as order_qty
FROM `adventureworks2019.Sales.SalesOrderDetail` d
left join `adventureworks2019.Sales.SalesOrderHeader` h
  on d.SalesOrderID = h.salesOrderID
group by extract(year from h.OrderDate), h.TerritoryID
qualify 
    dense_rank() over (
    partition by min(extract(year from h.OrderDate))
    order by count(distinct d.SalesOrderID) desc
  ) <= 3
order by yr desc, order_qty desc
```

#### 💡 Query Results

<img width="475" height="239" alt="3" src="https://github.com/user-attachments/assets/400aacf2-95b3-4c53-907a-52bcc2ec4e9a" />

### 4 - Seasonal discount cost by subcategory

Estimate the value of seasonal discounts by order year and subcategory using the offer discount percentage multiplied by the line unit price and quantity.

#### 🖊️ Query

```sql
-- Calc Total Discount Cost belongs to Seasonal Discount for each SubCategory
select
  extract(year from h.OrderDate) as yr
  ,sc.name
  ,sum(o.DiscountPct * d.UnitPrice * d.OrderQty) as total_discount_cost
FROM `adventureworks2019.Sales.SalesOrderDetail` d
left join `adventureworks2019.Sales.SalesOrderHeader` h
  on d.SalesOrderID = h.salesOrderID
left join `adventureworks2019.Sales.SpecialOffer` o
  on d.SpecialOfferID = o.SpecialOfferID
left join `adventureworks2019.Production.Product` p
  on d.ProductID = p.ProductID
left join `adventureworks2019.Production.ProductSubcategory` sc
  on safe_cast(p.ProductSubcategoryID as string) = safe_cast(sc.ProductSubcategoryID as string)
where lower(o.Type) like '%seasonal discount%'
group by extract(year from h.OrderDate), sc.name
order by yr
```

#### 💡 Query Results

<img width="454" height="119" alt="4" src="https://github.com/user-attachments/assets/912a69ca-e6cd-42e3-baaf-46f9dcd2d845" />

### 5 - Shipped-order customer cohorts in 2014

Group customers by their first observed month with a shipped order in 2014 (`Status = 5`) and track subsequent months in which they placed another qualifying order.

#### 🖊️ Query

```sql
-- Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)
with retention as (
select
  extract(month from OrderDate) as mth
  ,customerID
from `adventureworks2019.Sales.SalesOrderHeader`
where
  extract(year from OrderDate) = 2014
  and status = 5
group by extract(month from OrderDate),customerID
), first_order as (
  select
    distinct mth as month_join, customerID
  from retention
  qualify row_number() over (
      partition by customerID
      order by mth
    ) = 1
)
select distinct
  f.month_join
  ,concat('M - ', mth - month_join) as month_diff
  ,count(distinct r.customerID) as customer_cnt
from retention r left join first_order f
  on r.customerID = f.customerID
group by 1, 2
order by 1, 2
```

#### 💡 Query Results

<img width="475" height="258" alt="5" src="https://github.com/user-attachments/assets/ff68f52f-4773-4e17-a786-92067175ba32" />

### 6 - Monthly stocked quantities and change in 2011

Aggregate work-order `StockedQty` by product name and work-order end month. Compute the percentage change from the preceding observed month, round to one decimal, and replace null changes with zero.

#### 🖊️ Query

```sql
-- Trend of Stock level & MoM diff % by all product in 2011. If %gr rate is null then 0. Round to 1 decimal
select
  p.name
  , extract(month from o.EndDate) as mth
  , extract(year from o.EndDate) as yr
  , sum(StockedQty) as stock_qty
  , lead(sum(StockedQty)) over (
    partition by p.name
    order by min(extract(month from o.EndDate)) desc
  ) as stock_prv
  , coalesce(round(100.0 * (sum(StockedQty) / lead(sum(StockedQty)) over (
    partition by p.name
    order by min(extract(month from o.EndDate)) desc
  ) - 1), 1), 0.0) as diff
from `adventureworks2019.Production.Product` p
join  `adventureworks2019.Production.WorkOrder` o
  on p.ProductID = o.ProductID
where extract(year from o.EndDate) = 2011
group by p.name,extract(month from o.EndDate),extract(year from o.EndDate)
```

#### 💡 Query Results

<img width="514" height="283" alt="6" src="https://github.com/user-attachments/assets/6fa8115d-909e-4781-8f64-92fd711215fb" />

### 7 - Monthly stock-to-sales ratio by product in 2011

Compare units stocked through work orders with units sold for each product and month. The ratio is stocked units divided by units sold, rounded to one decimal.

#### 🖊️ Query

```sql
-- Calc Ratio of Stock / Sales in 2011 by product name, by month. Order results by month desc, ratio desc. Round Ratio to 1 decimal mom yoy
with sales as (
  select
    extract(month from h.OrderDate) as mth
    ,extract(year from h.OrderDate) as yr
    ,p.ProductID
    ,p.name
    ,sum(d.OrderQty) as sales_cnt
  from `adventureworks2019.Sales.SalesOrderDetail` d
  join `adventureworks2019.Sales.SalesOrderHeader` h
    on d.SalesOrderID = h.salesOrderID
  join `adventureworks2019.Production.Product` p
    on d.ProductID = p.ProductID
  where extract(year from h.OrderDate) = 2011
  group by extract(month from h.OrderDate),extract(year from h.OrderDate), p.ProductID, p.name
), stock as (
  select
    extract(month from EndDate) as mth
    ,extract(year from EndDate) as yr
    ,ProductID
    ,sum(StockedQty) as stock_qty
  from `adventureworks2019.Production.WorkOrder`
  where extract(year from EndDate) = 2011
  group by extract(month from EndDate),extract(year from EndDate),ProductID
)

select
  s1.*
  ,s2.stock_qty
  ,round(1.0 * s2.stock_qty / s1.sales_cnt, 1) as ratio
from sales s1
  full join stock s2 
    on s1.mth = s2.mth
    and s1.yr = s2.yr
    and s1.productID = s2.ProductID
order by s1.mth desc, ratio desc
```

#### 💡 Query Results

<img width="537" height="246" alt="7" src="https://github.com/user-attachments/assets/0a7f94c6-1975-461e-855b-6e26d77ebacf" />

### 8 - Pending purchase orders in 2014

Count distinct purchase orders dated in 2014 with pending status (`Status = 1`) and sum their `TotalDue`.

#### 🖊️ Query

```sql
-- No of order and value at Pending status in 2014
select
  extract(year from OrderDate) as yr
  ,status
  ,count(distinct PurchaseOrderID) as order_cnt
  ,sum(TotalDue) as value
from `adventureworks2019.Purchasing.PurchaseOrderHeader`
where 
  extract(year from OrderDate) = 2014
  and status = 1
group by extract(year from OrderDate), status
```

#### 💡 Query Results

<img width="466" height="128" alt="8" src="https://github.com/user-attachments/assets/57f8ec88-ebd5-4231-8a48-a956b53ada24" />

## 🔎 4 - Conclusion & Recommendations

### What the analysis can support

The queries provide a starting point for comparing sales volumes across subcategories and territories, examining seasonal discount costs, measuring repeat shipped-order activity, and monitoring production and procurement. No rankings, trends, or numerical findings are claimed without query outputs.

### Recommended next steps

1. Fix the YoY grouping and validate that every comparison uses the same subcategory and consecutive calendar years before publishing a top-three ranking.
2. Add cohort sizes and retention percentages. Use zero for observable months with no returning customers and leave months beyond the data window blank.
3. Build a calendar-month grid for production metrics, retain keys from both sides of the stock-to-sales join, and distinguish work-order output from on-hand inventory.
4. Run all eight queries and add result tables or screenshots. Summarize verified findings only after checking the date basis, filters, and metric definitions.
