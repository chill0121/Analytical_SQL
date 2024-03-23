# Analytical Problems in SQL

An exercise of PostgreSQL using the ClassicModels database.

---

## Table of Contents <a name="toc"></a>
- [Problem 1: Grace Period is Over](#prob1)
- [Problem 2: Overly-Friendly Salespeople](#prob2)
- [Problem 3: Shipping Department Woes](#prob3)

---

## Problem 1: <a name="prob1"></a>

The end of the fiscal year is arriving soon, accounting noticed a large amount of unpaid invoices that needs to be closed out before the year-end deadline. 

List the top 5 customers with their city and country concatenated together that have greatest remaining outstanding balance, meaning (sum total of all orders) - (total payments).

### Query:


```python
/* SELECT SubQuery:
 Outer query finds total payments grouped by customer.
 SELECT Inner query finds total order amount grouped by customer,
 output to a column. */
SELECT C.customernumber, customername, 
    city || ', ' || country AS city_country,
    -- Subquery in SELECT
    COALESCE( -- Coalesce to turn NULLs into 0s.
        (SELECT sum(priceeach * quantityordered) - sum(amount)
        FROM orders O,
            orderdetails D
        WHERE C.customernumber = O.customernumber
            AND O.ordernumber = D.ordernumber
        GROUP BY customernumber
    ), 0) AS balance_due
FROM customers C
    LEFT JOIN payments P
        ON C.customernumber = P.customernumber
GROUP BY C.customernumber
ORDER BY 4 DESC LIMIT 5;
```

### Output Answer Set:

<img src="https://github.com/chill0121/Analytical_SQL/blob/main/Images/Final_Prob_1_Output.png?raw=true" alt="1" width="700"/>

*Note: Full answer set shown here with LIMIT 5.*

---

## Problem 2: <a name="prob2"></a>

We've recently noticed a reduction in our profit margins when comparing our product's MSRP to our Total Sales numbers. We suspect our salespeople are giving out hefty discounts. 

Utilizing 5 tables, Employees > Customers > Order > OrderDetails > Products, generate a table of each employee's total sales, the total amount of discounts given ((QuantityOrdered*MSRP) - TotalSales), as well as the average value of their discount. Order the table by average discount in descending order, showing the employees who are giving the largest discounts at the top.

### Query:


```python
/* FROM subquery:
FROM inner query used to gather/calculate all aggregating info.*/
SELECT employee_name,
    to_char(sum(total_sold), 'FM9,999,999.99') as total_sales, -- sum all sales.
    to_char(sum(discount), 'FM9,999,999.99') AS total_discounts, -- all discounts given.
    round(avg(discount), 2) AS average_discount
FROM ( -- Start a long-chained inner query.
    SELECT -- All aggregation calculations here.
        (E.firstname || ' ' || E.lastname) AS employee_name,
        -- Calc price products sold at.
        COALESCE((D.priceeach * D.quantityordered), 0) AS total_sold,
        -- Calc MSRP - price sold = discount
        COALESCE(((P.msrp * D.quantityordered) - (D.priceeach * D.quantityordered)), 0) AS discount
    FROM employees E
        LEFT JOIN customers C -- Left join here will show those with 0 sales.
            ON E.employeenumber = C.salesrepemployeenumber
        LEFT JOIN orders O
            ON O.customernumber = C.customernumber
        LEFT JOIN orderdetails D
            ON O.ordernumber = D.ordernumber
        LEFT JOIN products P
            ON D.productcode = P.productcode
) AS TB -- Need an alias for the outer query
GROUP BY employee_name
ORDER BY avg(discount) DESC;
```

### Output Answer Set:

<img src="https://github.com/chill0121/Analytical_SQL/blob/main/Images/Final_Prob_2_Output.png?raw=true" alt="2" width="700"/>

*Note: Screenshot is not showing all rows in answer set table. Total 23 rows.*

---

## Problem 3: <a name="prob3"></a>

We're starting a pilot program to offer discounts to customers with previous shipping issues. These discounts will be offered to customers who have orders that have taken longer than 5 days from their order date to ship. 

We need to generate a table of customer's contact information, order and shipping date information, that meet the greater-than-5-days-to-ship criteria. Furthermore, to qualify, orders placed on or near the weekend need to be adjusted since it has been communicated that orders will not ship over the weekend. Orders placed on Thursday, Friday, Saturday, Sunday should have their dates adjusted the appropriate number of days to the following Monday. Also, the order comments need to be referenced to make sure we aren't discounting shipments that were delayed because a customer had exceeded their credit limit.

(e.g. on an order placed on Friday, 3 days will be added to the order date and the 5 day discount timer will start on Monday as long as the customer's credit is in good standing.) 

### Query:


```python
SELECT customer_name, 
    phone, 
    mod_orderdate, 
    shippeddate, 
    -- Display how many days it took to ship order.
    (shippeddate - mod_orderdate) AS days_to_ship
FROM (
    SELECT
        O.shippeddate,
        C.phone,
        O.comments,
        (C.contactfirstname || ' ' || C.contactlastname) AS customer_name,
        /*Check order day of the week, orders placed near or on the weekend will be padded
        since orders don't usually ship on the weekend.*/
        CASE
            WHEN to_char(orderdate, 'DY') IN ('THU') THEN orderdate + INT '4'
            WHEN to_char(orderdate, 'DY') IN ('FRI') THEN orderdate + INT '3'
            WHEN to_char(orderdate, 'DY') IN ('SAT') THEN orderdate + INT '2'
            WHEN to_char(orderdate, 'DY') IN ('SUN') THEN orderdate + INT '1'
        ELSE orderdate
    END AS mod_orderdate -- New column for adjusted orderdate
    FROM customers C
        LEFT JOIN orders O
            ON C.customernumber = O.customernumber
) AS TB
WHERE (shippeddate - mod_orderdate) > 5 -- check ship date against order date.
    -- Check comments to make sure order wasn't delayed for exceeded credit limit.
    AND (comments NOT like '%credit%' OR comments IS NULL)
ORDER BY days_to_ship DESC;
```

### Output Answer Set:

<img src="https://github.com/chill0121/Analytical_SQL/blob/main/Images/Final_Prob_3_Output.png?raw=true" alt="3" width="700"/>

*Note: Screenshot is not showing all rows in answer set table. Total 26 rows.*

###### [Back to Table of Contents](#toc)

---

---
