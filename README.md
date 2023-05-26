```sql
/* 
The goal of these codes are to demonstrate my data analysis skills with SQL
I run these queries in MS SQL server. 
These codes have four parts: 
    Data extraction and manipulation, 
    Amortization Schedule creation with Windows function, 
    Pivot table creation with subqueries, 
    and Appendix. 

Dataset:    I have 3 csv files called customers, lookup, and application: 
Table Name  Column name 
application Applicant_Beacon 
application Applicant_Province 
application ApplicationID 
application column6 
application FirstSubmissionDate 
application Last_Credit_Declinedcision_CoDeclined 
customers   Actual_Interest_Rate 
customers   Contract_Date 
customers   Customer_Name 
customers   CustomerID 
customers   Date_of_Birth 
customers   Desired_Interest_Rate 
customers   EP_Program 
customers   Financed_Amount 
customers   Province 
lookup  column1 
lookup  column2 
lookup  column3 
lookup  column4 
lookup  column5 
lookup  column6 
*/

-- Part 1
/* 
1. How many customers in Ontario born before 1/1/1975?
SQL Server uses a different date format (YYYY-MM-DD) than Excel. 
*/

SELECT COUNT(*)
FROM customers
WHERE Province = 'ON' AND Date_Of_Birth < '1975-01-01';

-- 2. How many customers with the word "Smith" in their name?

SELECT COUNT(*) 
FROM customers
WHERE Customer_Name LIKE '%Smith%';

-- 3. Weighted APR% for 4 Ride and Alberta Customers that born after 1/1/1980.

SELECT SUM(c.Actual_Interest_Rate * c.Financed_Amount) / SUM(c.Financed_Amount) as Weighted_APR
FROM customers c
WHERE c.EP_Program = '4 Ride' AND c.Province = 'AB' AND c.Date_of_Birth > '1980-01-01';

-- 4. Find the Financed Amount of CustomerID EP00009.

SELECT Financed_Amount
FROM customers
WHERE CustomerID = 'EP00009';

-- 5. Fill the Desired_Interest_Rate column based on the Province and EPProgram of the lookup table.

UPDATE customers
SET Desired_Interest_Rate = 
    (SELECT 
        CASE 
            WHEN EP_Program = '5 Ride' THEN [5_Ride]
            WHEN EP_Program = '4 Ride' THEN [4_Ride]
            WHEN EP_Program = '3 Ride' THEN [3_Ride]
            WHEN EP_Program = '2 Ride' THEN [2_Ride]
            WHEN EP_Program = 'EP Ride+' THEN [EP_Ride+]
        END
     FROM lookup
     WHERE Province = customers.Province);

SELECT CustomerID, Customer_Name, Province, EP_Program, Desired_Interest_Rate
FROM customers;

-- 6. List all 4 Ride customer names.

SELECT Customer_Name
FROM customers
WHERE EP_Program = '4 Ride';

-- 7. Hide the row(s) depends on the selection of the drop-down in cell C40.
-- In a SQL context, instead of hiding rows, I just select them in query based on some condition.

SELECT *
FROM customers
WHERE customers.EP_Program = '4 Ride';
