# Assumptions:
* Restrict analysis of August sales to 2004 only, as August 2005 data is mis-configured and not reflecting the actual value


**Connect to Teradata Dillards DB**
use ua_dillards

---

**Exercise 1**
How many distinct dates are there in the saledate column of the transaction table for each month/year combinationin the database?

```sql
SELECT
  EXTRACT (month FROM saledate) AS month_num,
  EXTRACT (year FROM saledate) AS year_num,
  COUNT (DISTINCT EXTRACT (day FROM saledate)) AS days_in_month,
  COUNT (EXTRACT (day FROM saledate)) AS num_transactions
GROUP BY month_num, year_num
ORDER BY year_num, month_num
```


|"month_num"|"year_num"|"days_in_month"|"num_transactions"|
|-----------|----------|---------------|------------------|
|8|2004|31|8292953
|9|2004|30|8967415
|10|2004|31|8412131
|11|2004|29|7047319
|12|2004|30|13383892
|1|2005|31|8952311
|2|2005|28|11352221
|3|2005|30|8940444
|4|2005|30|9082523
|5|2005|31|7715779
|6|2005|30|7922997
|7|2005|31|11122770
|8|2005|27|9724141

*From above results, Aug 2004 shows 31 days while Aug 2005 shows only 27. So I will use data between Aug 2004 - Jul 2005, excluding Aug 2005.*

---
**Exercise 2**
Use a CASE statement within an aggregate function to determine which sku had the greatest total sales during the combined summer months of June, July, and August.

```sql
SELECT
  sku,
  SUM( CASE WHEN EXTRACT(MONTH FROM saledate) = 6 AND stype = 'p' THEN amt END) as Jun_sale,
  SUM( CASE WHEN EXTRACT(MONTH FROM saledate) = 7 AND stype = 'p' THEN amt END) as Jul_sale,
  SUM( CASE WHEN EXTRACT(MONTH FROM saledate) = 8 AND stype = 'p' THEN amt END) as Aug_sale,
  SUM( Jun_sale + Jul_sale + Aug_sale ) as com_sale
FROM trnsact
GROUP BY 1;
```

*I was using above query, but got error msg:*
*Error Code - 3568*
*Error Message - [Teradata Database] [TeraJDBC 16.20.00.12] [Error 3568] [SQLState 42000] Cannot nest aggregate operations.*

*the error is caused by putting both SUM and + together in syntax., and I re-write below query to populate the correct results*

```sql
SELECT
  sku,
  SUM( CASE WHEN EXTRACT(MONTH FROM saledate) = 6 AND stype = 'p' THEN amt END) as Jun_sale,
  SUM( CASE WHEN EXTRACT(MONTH FROM saledate) = 7 AND stype = 'p' THEN amt END) as Jul_sale,
  SUM( CASE WHEN EXTRACT(MONTH FROM saledate) = 8 AND stype = 'p' THEN amt END) as Aug_sale,
  ( Jun_sale + Jul_sale + Aug_sale ) as com_sale
FROM trnsact
WHERE NOT ( EXTRACT(MONTH FROM saledate) = 8 AND EXTRACT(YEAR FROM saledate) = 2005 )
GROUP BY 1
ORDER BY com_sale DESC;
```
*I add conditions to filter out Aug 2005 data, but instructor did not do so in the key doc*

* Embed CASE with aggregate function allows me to generate summary results based on subset without change original data format.
* Teradata allows me to directly use derived field (i.e. Jun_sale) in SELECT syntax, which is quite handy.
* The processing order of logic operator NOT, AND, and OR are: NOT, AND, OR. So I need to add bracket to apply restrict conditions for month and year, so both two conditions will be applied together by NOT
---

**Exercise 3**
How many distinct dates are there in the saledate column of the transaction table for each month/year/store combinationin the database?  Sort your results by the number of days per combination in ascending order.

```sql
SELECT
  EXTRACT (month FROM saledate) AS month_num,
  EXTRACT (year FROM saledate) AS year_num,
  store,
  COUNT (DISTINCT saledate) AS days_in_month
FROM trnsact
WHERE NOT ( month_num = 8 AND year_num = 2005 )
GROUP BY 1,2,3
ORDER BY days_in_month ASC;
```

|"month_num"|"year_num"|"STORE"|"days_in_month"|
|-----------|----------|--|--|
|3|2005|8304|1
|9|2004|4402|1
|8|2004|9906|1
|7|2005|7604|1
|8|2004|8304|1
|8|2004|7203|3
|3|2005|6402|11
|4|2005|5703|16
|3|2005|3002|16
|10|2004|4903|17


* it is a good practice to check dates values and trends before prediction analysis
* if possible, we should exclude these records which has abnormal values in our analysis


---

**Exercise 4.a**  
What is the average daily revenue for each store/month/year combination in the database?  Calculate this by dividing thetotal revenue for a group by the number of sales days available in the transaction table for that group.

```sql
SELECT
  EXTRACT (month FROM saledate) AS month_num,
  EXTRACT (year FROM saledate) AS year_num,
  store,
  SUM(amt) / COUNT(DISTINCT saledate) AS daily_sale
FROM trnsact
WHERE stype = 'P'
GROUP BY 1,2,3
ORDER BY daily_sale DESC
;
```

* Now we finally use normalized sales (daily sale) to evaluate performance


**Exercise 4.b**
This following exercise will continue remove all data in Aug, 2005. Then, give the data we have availabble in our data set,
I will only look at areas where having at least 20 days of data within each month. This will helps exclude outlier data points and give us more robust analysis.
Also, I will concatenate YEAR and MONTH to be one conslidated column, instead of two seperate ones

```sql
SELECT
  EXTRACT (year FROM saledate) || ' -' || EXTRACT (month FROM saledate) AS year_month,  
  store,
  SUM(amt) / COUNT(DISTINCT saledate) AS avg_sale,
  COUNT(DISTINCT saledate) as days_in_month -- might be good to add column so we can test the condition
FROM trnsact
WHERE stype = 'P' AND ( NOT ( EXTRACT(MONTH FROM saledate) = 8 AND EXTRACT(YEAR FROM saledate) = 2005 ) )
GROUP BY 1,2
HAVING COUNT(DISTINCT saledate) > 20
ORDER BY avg_sale DESC;
;
;
```


**follow up question**
How do the population statistics of the geographical location surrounding a store relate tosales performance

---


**Exercise 5**
What is the average daily revenue brought in by Dillard’s stores in areas of high, medium, or low levels of high school education?

Define high school education levels:
* “low”: high school graduation rates between 50-60%,
* “medium”: high school graduation rates between 60.01-70%,
* “high”: high school graduation rates of above 70%.


Per suggestion I learned from valerielim, I checked the store distribution by education level from above definition first.
However, the results show skewed distribution.

```sql
SELECT
(CASE
WHEN msa_high>50 AND msa_high<=60 THEN 'LOW (>50%)'
WHEN msa_high>60 AND msa_high<=70 THEN 'MED (>60%)'
WHEN msa_high>70 THEN 'HIGH (>70%)'
END) AS education_levels,
COUNT (DISTINCT store) AS num_stores
FROM store_msa
GROUP BY education_levels
```
| EDUCATION_LEVEL | NUM_STORES |
| --------------- | ---------- |
| LOW (>50%) | 324
| MED (>60%) | 5
| HIGH (>70%) | 4


```sql
SELECT rate.MSA_HIGH_RATE, (SUM(revenue.total_revenue) / SUM(revenue.days_in_month)) AS daily_renuve
FROM
   (
     SELECT
       EXTRACT (year FROM saledate) || '-' || EXTRACT (month FROM saledate) AS year_month,
       store,
       SUM(amt) / COUNT(DISTINCT saledate) AS avg_sale,
       SUM(amt) as total_revenue,
       COUNT(DISTINCT saledate) as days_in_month
     FROM trnsact
        WHERE  NOT ( EXTRACT(MONTH FROM saledate) = 8 AND EXTRACT(YEAR FROM saledate) = 2005 ) AND stype = 'P'
        GROUP BY 1,2
        HAVING days_in_month > 20
   ) AS
revenue JOIN
     (
         SELECT store,
         CASE WHEN MSA_HIGH > 50 AND MSA_HIGH <= 60 THEN 'LOW (>50%)'
              WHEN MSA_HIGH > 60 AND MSA_HIGH <= 70 THEN 'MED (>60%)'
              WHEN MSA_HIGH > 70 THEN 'HIGH (>70%)'
              ELSE 'other'
         END AS MSA_HIGH_RATE
         FROM store_msa
      ) AS
rate
   ON revenue.store = rate.store
GROUP BY 1
;
```

* I specifically keep the same aggregate levels in sub-query 'revenue', which are year_month, and store. This is to ensure filtering condition still holds true at these levels.


---


**Exercise 6**
Compare the average daily revenues of the stores with the highest median msa_income and the lowest median msa_income.  In what city and state were these stores, and which store had a higher average daily revenue?  


```sql
SELECT msa.store, msa.state, msa.city, msa.msa_income,
      (SUM(revenue.total_revenue) / SUM(revenue.days_in_month)) AS daily_renuve
FROM
   (
     SELECT
       EXTRACT (year FROM saledate) || '-' || EXTRACT (month FROM saledate) AS year_month,
       store,
       SUM(amt) / COUNT(DISTINCT saledate) AS avg_sale,
       SUM(amt) as total_revenue,
       COUNT(DISTINCT saledate) as days_in_month
     FROM trnsact
        WHERE  NOT ( EXTRACT(MONTH FROM saledate) = 8 AND EXTRACT(YEAR FROM saledate) = 2005 ) AND stype = 'P'
        GROUP BY 1,2
        HAVING days_in_month > 20
   ) AS
revenue JOIN
     (
       SELECT store, state, city, msa_income
       FROM store_msa
       AS s JOIN
       (SELECT MAX(msa_income) as max_income, MIN(msa_income) as min_income from store_msa)
       AS m
       ON m.max_income  = s.msa_income or m.min_income = s.msa_income  
      ) AS
msa
   ON revenue.store = msa.store
GROUP BY 1,2,3,4
;
```

* Another simpler solution to isolate store with MAX/MIN income is to put both MAX income and MIN income values in one list, and use it in the filtering condition

```sql
SELECT msa.store, msa.state, msa.city,
      (SUM(revenue.total_revenue) / SUM(revenue.days_in_month)) AS daily_renuve
FROM
   (
     SELECT
       EXTRACT (year FROM saledate) || '-' || EXTRACT (month FROM saledate) AS year_month,
       store,
       SUM(amt) / COUNT(DISTINCT saledate) AS avg_sale,
       SUM(amt) as total_revenue,
       COUNT(DISTINCT saledate) as days_in_month
     FROM trnsact
        WHERE  NOT ( EXTRACT(MONTH FROM saledate) = 8 AND EXTRACT(YEAR FROM saledate) = 2005 ) AND stype = 'P'
        GROUP BY 1,2
        HAVING days_in_month > 20
   ) AS
revenue JOIN
     (
       SELECT store, state, city, msa_income
       FROM store_msa
       WHERE msa_income in
       (
         ( SELECT MAX(msa_income) FROM store_msa),
         ( SELECT MIN(msa_income) FROM store_msa))
       )
      AS
msa
   ON revenue.store = msa.store
GROUP BY 1,2,3
;
```


* A very interesting aspect of sub-query is that we cannot put LIMIT/TOP and ORDER BY in subqueries positioned within SELECT, WHERE, or HAVING clauses,
the only place that can hold these operations is FROM


---


**Exercise 7**
What is the brand of the sku with the greatest standard deviation in sprice?  Only examine skus that have been part of over 100 transactions.

*Measure of dispersion provides different insights about data. In stock analysis, high variability in stock prices indicates risky invest in assets; In Dillards data, a high standard deviation could indicate that the item often needs to be put on sale for customer to buy it, or the original price set by manufacturer or retailer is too high. Or it could simply indicates data error.

```sql
SELECT
  t.sku,
  s.brand,
  COUNT(DISTINCT ( t.trannum || t.register || t.saledate || t.seq)) as transact,
  AVG(t.sprice) as avg_price, -- I am curious to check mean as well
  STDDEV_SAMP(t.sprice) as dev_price
FROM trnsact t
   JOIN skuinfo s ON t.sku = s.sku
WHERE  NOT ( EXTRACT(MONTH FROM t.saledate) = 8 AND EXTRACT(YEAR FROM t.saledate) = 2005 ) AND t.stype = 'P'
GROUP BY 1,2
HAVING transact> 100
ORDER BY dev_price DESC;
```

* from ER diagram, unique transaction is defined by combo key of register, trannum, seq, and saledate


---


**Exercise 8**
Examine all the transactions for the sku with the greatest standard deviation in sprice, but only consider skus that are part of more than 100 transactions.


```sql
SELECT
  t.sku,
  s.brand,
  COUNT(DISTINCT ( t.trannum || t.register || t.saledate || t.seq)) as transact,
  AVG(t.orgprice) as org_price,
  AVG(t.orgprice) - avg(t.sprice) as price_diff,
  STDDEV_SAMP(t.sprice) as dev_price
FROM trnsact t
   JOIN skuinfo s ON t.sku = s.sku
WHERE  NOT ( EXTRACT(MONTH FROM t.saledate) = 8 AND EXTRACT(YEAR FROM t.saledate) = 2005 ) AND t.stype = 'P'
GROUP BY 1,2
HAVING transact> 100
ORDER BY dev_price DESC;
```

* Is it the org price too high? It is still hard to say so based on just the deviation, but z-score might be more reliable matrix to detect the normalized deviation of sales prices.
* in sales trend analysis, it is common to see montly variations, so it may make more sense to compare the sales during a given month to the sales in the same month of a previous year.

---


**Exercise 9**
What was the average daily revenue Dillard’s brought in during each month of the year?


```sql
SELECT
  EXTRACT (month FROM saledate) AS month_num,
  (CASE
    WHEN month_num=1 THEN 'Jan'
    WHEN month_num=2 THEN 'Feb'
    WHEN month_num=3 THEN 'Mar'
    WHEN month_num=4 THEN 'Apr'
    WHEN month_num=5 THEN 'May'
    WHEN month_num=6 THEN 'Jun'
    WHEN month_num=7 THEN 'Jul'
    WHEN month_num=8 THEN 'Aug'
    WHEN month_num=9 THEN 'Sep'
    WHEN month_num=10 THEN 'Oct'
    WHEN month_num=11 THEN 'Nov'
    WHEN month_num=12 THEN 'Dec'
    END) as month_name,  
  SUM(amt) / (1000000 * COUNT(DISTINCT saledate) ) AS daily_revenue_M
FROM trnsact
WHERE NOT ( EXTRACT(MONTH FROM saledate) = 8 AND EXTRACT(YEAR FROM saledate) = 2005 ) AND stype = 'P'
GROUP BY 1,2
ORDER BY daily_revenue_M DESC
```

---


**Exercise 10**
Which department, in which city and state of what store, had the greatest % increase in average daily sales revenue from November to December?    

```sql
SELECT
    t.store, d.deptdesc, st.state, st.city,
    SUM(CASE WHEN EXTRACT(month FROM t.saledate) = 11 THEN amt END) as Nov_rev,
    COUNT( DISTINCT (CASE WHEN EXTRACT(month FROM t.saledate) = 11 THEN saledate END) ) as Nov_days,
    SUM(CASE WHEN EXTRACT(month FROM t.saledate) = 12 THEN amt END) as Dec_rev,
    COUNT( DISTINCT (CASE WHEN EXTRACT(month FROM t.saledate) = 12 THEN saledate END) ) as Dec_days,  
    100* ( ( Dec_rev / Dec_days ) / (Nov_rev / Nov_days ) - 1 ) as Dec_div_Nov
  FROM trnsact t
  JOIN skuinfo s ON t.sku = s.sku
  JOIN deptinfo d ON s.dept = d.dept
  JOIN strinfo st ON t.store = st.store
  WHERE ( EXTRACT(MONTH FROM saledate) in (11,12) AND stype = 'P' )
  GROUP BY 1,2,3,4
  HAVING Nov_days >= 20 AND Dec_days >=20
ORDER BY Dec_div_Nov desc;
```

* transaction table has the lowest granularity lvl of both STORE and SKU/DEPARTMENT, and so I simply join other tables to transaction and append their attributes for further aggregation


---

**Exercise 11**
What is the city and state of the store that had the greatest decrease in average daily revenue fromAugust to September?



```sql
SELECT
    st.state, st.city, t.store,
    SUM(CASE WHEN EXTRACT(month FROM t.saledate) = 8 THEN amt END) as Aug_rev,
    COUNT( DISTINCT (CASE WHEN EXTRACT(month FROM t.saledate) = 8 THEN saledate END) ) as Aug_days,
    SUM(CASE WHEN EXTRACT(month FROM t.saledate) = 9 THEN amt END) as Sep_rev,
    COUNT( DISTINCT (CASE WHEN EXTRACT(month FROM t.saledate) = 9 THEN saledate END) ) as Sep_days,  
    100* ( ( Sep_rev/ Sep_days) / (Aug_rev/ Aug_days) - 1 ) as Sep_div_Aug
  FROM trnsact t
  JOIN strinfo st ON t.store = st.store
  WHERE t.stype = 'P' AND (NOT ( EXTRACT(MONTH FROM t.saledate) = 8 AND EXTRACT(YEAR FROM t.saledate) = 2005 ) )
  GROUP BY 1,2,3
  HAVING Aug_days >= 20 AND Sep_days >=20
ORDER BY Sep_div_Aug ASC;
```


---

**Exercise 12**
Determine the month of maximum total revenuefor each store.  Count the number of stores whose month of maximum total revenuewas in each of the twelve months.  Then determine the month of maximum average daily revenue.  Count the number of stores whose month of maximum average daily revenuewas in each of the twelve months.  How do they compare?

```sql
SELECT revenue.mon_num, count(*)
FROM
  (
  SELECT
       t.store, EXTRACT(month FROM t.saledate) as mon_num,
       SUM(amt) as revenue
    FROM trnsact t
    WHERE t.stype = 'P' AND (NOT ( EXTRACT(MONTH FROM t.saledate) = 8 AND EXTRACT(YEAR FROM t.saledate) = 2005 ) )
    GROUP BY 1,2
    HAVING COUNT( DISTINCT t.saledate) >= 20
  ) AS
revenue JOIN
  (
    SELECT rev.store, MAX(revenue) AS max_revenue
    FROM
    (
      SELECT
           t.store, EXTRACT(month FROM t.saledate) as mon_num,
           SUM(amt) as revenue
        FROM trnsact t
        WHERE t.stype = 'P' AND (NOT ( EXTRACT(MONTH FROM t.saledate) = 8 AND EXTRACT(YEAR FROM t.saledate) = 2005 ) )
        GROUP BY 1,2
        HAVING COUNT( DISTINCT t.saledate) >= 20
    ) AS
    rev
    GROUP BY 1
  ) as
max_mo ON revenue.store = max_mo.store AND revenue.revenue = max_mo.max_revenue
GROUP BY 1;

```

* Another simpler solution is to use RANK() and ROW_NUMBER() function from Teradata
```sql

SELECT rank.mon_num, COUNT(rank.mon_num) AS top_mon_ct  
FROM
(
    SELECT rev.store, rev.mon_num, rev.revenue,
      RANK() OVER(PARTITION BY store ORDER BY revenue DESC) AS "rank_revenue",
      ROW_NUMBER() OVER(PARTITION BY store ORDER BY revenue DESC) AS "row_number"
    FROM
    (
        SELECT
             t.store, EXTRACT(month FROM t.saledate) as mon_num,
             SUM(amt) as revenue
          FROM trnsact t
          WHERE t.stype = 'P' AND (NOT ( EXTRACT(MONTH FROM t.saledate) = 8 AND EXTRACT(YEAR FROM t.saledate) = 2005 ) )
          GROUP BY 1,2
          HAVING COUNT( DISTINCT t.saledate) >= 20
    ) AS
    rev
) AS
rank_rev
WHERE rank_revenue = 1
GROUP BY 1;
;
```
* PARTITION BY comes handy when I tried to rank revenue by each months
* rank and row_number variable is quoted
* initially the derived table was named as 'rank', but Teradata rejects the query as 'rank' overlaps with its official reserved key words, and so it is abandoned. To look for more reserved key words, check in this link: [SQL reserved words checker] (https://www.petefreitag.com/tools/sql_reserved_words_checker)



# exam cases

```sql
SELECT
    t.store, d.deptdesc, st.state, st.city,
    SUM(CASE WHEN EXTRACT(month FROM t.saledate) = 11 THEN amt END) as Nov_rev,
    COUNT( DISTINCT (CASE WHEN EXTRACT(month FROM t.saledate) = 11 THEN saledate END) ) as Nov_days,
    SUM(CASE WHEN EXTRACT(month FROM t.saledate) = 12 THEN amt END) as Dec_rev,
    COUNT( DISTINCT (CASE WHEN EXTRACT(month FROM t.saledate) = 12 THEN saledate END) ) as Dec_days,  
    100* ( ( Dec_rev / Dec_days ) / (Nov_rev / Nov_days ) - 1 ) as Dec_div_Nov
  FROM trnsact t
  JOIN skuinfo s ON t.sku = s.sku
  JOIN deptinfo d ON s.dept = d.dept
  JOIN strinfo st ON t.store = st.store

  WHERE ( EXTRACT(MONTH FROM t.saledate) in (11,12) AND t.stype = 'P' )
  AND s.dept in
  (from skuinfo s
  JOIN
    (
      SELECT S.dept,
      SUM(CASE WHEN EXTRACT(month FROM t.saledate) = 11 THEN amt END) as Nov_rev,
      SUM(CASE WHEN EXTRACT(month FROM t.saledate) = 12 THEN amt END) as Dec_rev
      FROM trnsact t
      JOIN skuinfo s ON t.sku = s.sku
      GROUP BY 1
      HAVING Nov_rev > 1000 AND Dec_rev > 1000 ) AS  dept_sub ON s.dept = dept_sub.dept )
  as dept_large
  GROUP BY 1,2,3,4
  HAVING Nov_days >= 20 AND Dec_days >=20
ORDER BY Dec_div_Nov desc;
;


SELECT rank.mon_num, COUNT(rank.mon_num) AS top_mon_ct  
FROM
(
    SELECT rev.store, rev.mon_num, rev.revenue,
      RANK() OVER(PARTITION BY store ORDER BY revenue ASC) AS "rank_revenue",
      ROW_NUMBER() OVER(PARTITION BY store ORDER BY revenue ASC) AS "row_number"
    FROM
    (
        SELECT
             t.store, EXTRACT(month FROM t.saledate) as mon_num,
             SUM(amt) as revenue
          FROM trnsact t
          WHERE t.stype = 'P' AND (NOT ( EXTRACT(MONTH FROM t.saledate) = 8 AND EXTRACT(YEAR FROM t.saledate) = 2005 ) )
          GROUP BY 1,2
          HAVING COUNT( DISTINCT t.saledate) >= 20
    ) AS
    rev
) AS
rank_rev
WHERE rank_revenue = 1
GROUP BY 1;
;


```
