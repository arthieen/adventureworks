# adventureworks
-- Query 01: Calc Quantity of items, Sales value & Order quantity by each Subcategory in L12M
SELECT 
      FORMAT_DATE('%b %Y', s.ModifiedDate) month
      ,ps.Name
      ,sum(OrderQty) item_qty 
      ,sum(LineTotal) sales
      , count(SalesOrderID) order_qty
  
FROM `adventureworks2019.Sales.SalesOrder` s
JOIN `adventureworks2019.Sales.Product` p
on s.ProductID = p.ProductID
JOIN `adventureworks2019.Production.ProductSubcategory` ps
ON p.ProductSubcategoryID = cast(ps.ProductSubcategoryID as STRING)
WHERE
  DATE(s.ModifiedDate) >= DATE_SUB((SELECT MAX(DATE(ModifiedDate)) FROM `adventureworks2019.Sales.SalesOrder`), INTERVAL 12 MONTH)
  AND DATE(s.ModifiedDate) <= (SELECT MAX(DATE(ModifiedDate)) FROM `adventureworks2019.Sales.SalesOrder`)
group by 2,1
order by 1,2

-- Query 02: Calc % YoY growth rate by SubCategory & release top 3 cat with highest grow rate. Can use metric: quantity_item. Round results to 2 decimal
with t1 as(
SELECT FORMAT_DATE('%Y', s.ModifiedDate) yr
      ,ps.Name
      ,sum (OrderQty ) as item_qty
      
FROM `adventureworks2019.Sales.SalesOrder`  s
JOIN `adventureworks2019.Sales.Product` p
on s.ProductID = p.ProductID
JOIN `adventureworks2019.Production.ProductSubcategory` ps
ON p.ProductSubcategoryID = cast(ps.ProductSubcategoryID as STRING)
group by 2,1
order by 2 asc,1 desc
), t2 as (
select *
      ,LEAD(item_qty) OVER (partition by t1.Name ORDER BY yr) AS next_value
from t1
)
select Name
      , item_qty
      , next_value
      ,round((next_value-item_qty)/item_qty,2) AS YoYrate
from t2
order by YoYrate desc
limit 3

-- Query 03: Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number
with t1 as(
  SELECT FORMAT_DATE('%Y', D.ModifiedDate) yr
        , TerritoryID
        , SUM(OrderQty) AS item_qty
  FROM `adventureworks2019.Sales.SalesOrderDetail` D
  JOIN `adventureworks2019.Sales.SalesOrderHeader` H
  ON D.SalesOrderID = H.SalesOrderID
  GROUP BY 1,2
  ORDER BY 2,1
), t2 as(
select *
      , rank() over (partition by yr order by item_qty desc) as rk
from t1
)
select *
from t2
where rk in (1,2,3)
order by 1 desc

-- Query 04: Calc Total Discount Cost belongs to Seasonal Discount for each SubCategory
SELECT FORMAT_DATE('%Y', s.ModifiedDate) yr
      , Subcategory
      , sum( OrderQty*UnitPrice*DiscountPct) as dis
FROM `adventureworks2019.Sales.SalesOrderDetail`s
join `adventureworks2019.Sales.Product`p
on s.ProductID = p.ProductID
join `adventureworks2019.Sales.SpecialOffer`o
on s.SpecialOfferID = o.SpecialOfferID
where lower(Type) like '%seasonal discount%' 
group by 1,2

-- Query 05: Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)
with t1 as  
  (SELECT  
          H.customerID
          ,rank () over (partition by H.customerID order by H.ModifiedDate) as rk
          ,EXTRACT(MONTH FROM H.ModifiedDate ) mth
          
  FROM `adventureworks2019.Sales.Customer` C
  JOIN `adventureworks2019.Sales.SalesOrderHeader` H
  ON C.CustomerID = H.CustomerID
  where FORMAT_DATE('%Y', H.ModifiedDate)= '2014'
    AND H.Status=5
  order by 1,2 
  )
, first_mth as(
  SELECT customerID
        , rk
        , mth as mth_join  
  FROM t1
  WHERE rk =1
  )
SELECT 
      first_mth.mth_join
      ,CONCAT('M-', t1.mth-first_mth.mth_join) mth_diff
      ,count(distinct t1.customerID) customer_qty
FROM t1
JOIN first_mth
ON first_mth.customerID = t1.customerID
group by 1,2
order by 1,2

-- Query 06: Trend of Stock level & MoM diff % by all product in 2011. If %gr rate is null then 0. Round to 1 decimal
with t1 as
  (SELECT Name
        , extract(month from W.ModifiedDate) as mth 
        , extract(year from W.ModifiedDate) as yr 
        , sum(StockedQty) st_qty
  FROM `adventureworks2019.Production.WorkOrder`W
  join `adventureworks2019.Production.Product` P
  on W.ProductID = P.ProductID
  where FORMAT_DATE('%Y', W.ModifiedDate) = '2011'
  group by 1,2,3
  order by 1,2
  )
  , t2 as (
  select *
        , LEAD(st_qty) OVER (PARTITION BY Name order by mth desc ) prv_qty
  from t1
  order by 1
  )
select *
      , case when st_qty/prv_qty is null then 0
              else round ((st_qty-prv_qty)/prv_qty*100,1) end as MoM_rate
from t2
