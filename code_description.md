# Table of Contents
1. [Introduction](#Introduction)
2. [Data extraction and manipulation](#data-extraction-and-manipulation)
3. [Amortization Schedule creation with Windows function](#amortization-schedule-creation-with-windows-function)
4. [Pivot table creation with subqueries](#pivot-table-creation-with-subqueries)
5. [Appendix](#appendix)

# Introduction

This repository demonstrates my data analysis skills with SQL, specifically with MS SQL server. 
The following tables and their corresponding columns were used in this demonstration:

| Table Name | Column Name |
| --- | --- |
| application | Applicant_Beacon |
| application | Applicant_Province |
| application | ApplicationID |
| application | FirstSubmissionDate |
| application | Decision |
| customers | Actual_Interest_Rate |
| customers | Contract_Date |
| customers | Customer_Name |
| customers | CustomerID |
| customers | Date_of_Birth |
| customers | Desired_Interest_Rate |
| customers | EP_Program |
| customers | Financed_Amount |
| customers | Province |
| lookup | Province |
| lookup | 5_Ride |
| lookup | 4_Ride |
| lookup | 3_Ride |
| lookup | 2_Ride |
| lookup | EP_Ride+ |

# Data Extraction and Manipulation
-- 1. How many customers in Ontario born before 1/1/1975?
SQL Server uses a different date format (YYYY-MM-DD) than Excel.


```sql
SELECT COUNT(*)
FROM customers
WHERE Province = 'ON' AND Date_Of_Birth < '1975-01-01';
```
-- 2. How many customers with the word "Smith" in their name?
```sql
SELECT COUNT(*) 
FROM customers
WHERE Customer_Name LIKE '%Smith%';
```
-- 3. Weighted APR% for 4 Ride and Alberta Customers that born after 1/1/1980.
```sql
SELECT SUM(c.Actual_Interest_Rate * c.Financed_Amount) / SUM(c.Financed_Amount) as Weighted_APR
FROM customers c
WHERE c.EP_Program = '4 Ride' AND c.Province = 'AB' AND c.Date_of_Birth > '1980-01-01'

-- 4. Find the Financed Amount of CustomerID EP00009.
```sql
SELECT Financed_Amount
FROM customers
WHERE CustomerID = 'EP00009';
```
-- 5. Fill the Desired_Interest_Rate column based on the Province and EPProgram of the lookup table.
```sql
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
```
```sql
SELECT CustomerID, Customer_Name, Province, EP_Program, Desired_Interest_Rate
FROM customers;

-- 6. List all 4 Ride customer names.
```sql
SELECT Customer_Name
FROM customers
WHERE EP_Program = '4 Ride';
```

-- 7. Hide the row(s) depends on the selection of the drop-down in cell C40.
-- In a SQL context, instead of hiding rows, I just select them in query based on some condition.
```sql
SELECT *
FROM customers
WHERE customers.EP_Program = '4 Ride'
```

# Amortization Schedule creation with Windows function
/* 
Create an Amortization Schedule
Loan Amount			 25,000 
Rate			13.99%
Term in Years			 6 
Frequency			 Bi-Weekly 
No of Payments			 156 
1st Payment Date:			1-May-2023

Columns are
PMT#	PMT Date	 Beginning Balance 	 PMT Amount 	 Interest Paid 	 Principal Paid 	 Ending Balance 
*/

```sql
DECLARE @loanAmount DECIMAL(18, 2) = 25000.0;
DECLARE @interestRate DECIMAL(18, 4) = 0.1399;
DECLARE @termInYears INT = 6;
DECLARE @frequency INT = 26; -- Bi-weekly
DECLARE @noOfPayments INT = 156;
DECLARE @firstPaymentDate DATE = '2023-05-01';

DECLARE @ratePerPeriod DECIMAL(18, 4) = @interestRate / @frequency;
DECLARE @pmt DECIMAL(18, 2) = @loanAmount * @ratePerPeriod / (1 - POWER(1 + @ratePerPeriod, -@noOfPayments));

WITH AmortizationSchedule (PmtNo, PmtDate, BeginningBalance, PmtAmount, InterestPaid, PrincipalPaid, EndingBalance) AS
(
    SELECT 
        1 AS PmtNo,
        @firstPaymentDate AS PmtDate,
        @loanAmount AS BeginningBalance,
        @pmt AS PmtAmount,
        CAST(@loanAmount * @ratePerPeriod AS DECIMAL(18,2)) AS InterestPaid,
        CAST(@pmt - @loanAmount * @ratePerPeriod AS DECIMAL(18,2)) AS PrincipalPaid,
        CAST(@loanAmount - (@pmt - @loanAmount * @ratePerPeriod) AS DECIMAL(18,2)) AS EndingBalance

    UNION ALL

    SELECT 
        PmtNo + 1,
        DATEADD(WEEK, 2, PmtDate),
        CAST(EndingBalance AS DECIMAL(18,2)),
        @pmt,
        CAST(EndingBalance * @ratePerPeriod AS DECIMAL(18,2)),
        CAST(@pmt - EndingBalance * @ratePerPeriod AS DECIMAL(18,2)),
        CAST(EndingBalance - (@pmt - EndingBalance * @ratePerPeriod) AS DECIMAL(18,2))
    FROM AmortizationSchedule
    WHERE PmtNo < @noOfPayments
)

SELECT * FROM AmortizationSchedule
OPTION (MAXRECURSION 156);
```
# Pivot table creation with subqueries
/* 
 Using the Data tab, create a Pivot table with the following requirements
- Province as columns, Date as rows
- A timeline which filters out Mar applications only
- Summary by running total of Number and % of applications by day


Columns in application are
ApplicationID	FirstSubmissionDate	Applicant_Province	Applicant_Beacon	Decision
*/

--First, I create the pivot table with Provinces as columns and Date as rows. I list only three provinces (ON, BC, AB) for illustation purposes.
```sql
SELECT 
    FirstSubmissionDate, 
    [ON] AS ON_Applications, 
    [BC] AS BC_Applications, 
    [AB] AS AB_Applications
FROM
    (SELECT 
        FirstSubmissionDate, 
        Applicant_Province 
    FROM 
        application 
    WHERE 
        MONTH(FirstSubmissionDate) = 3) AS SourceTable
PIVOT
(
    COUNT(Applicant_Province)
    FOR Applicant_Province IN ([ON], [BC], [AB])
) AS PivotTable;
```
--Then I manually calculate a running total for Ontario
```sql
SELECT 
    FirstSubmissionDate, 
    SUM([ON]) OVER (ORDER BY FirstSubmissionDate) AS RunningTotal_ON,
    SUM([ON]) OVER (ORDER BY FirstSubmissionDate) / SUM([ON]) OVER () AS RunningPercent_ON
FROM 
    (SELECT 
        FirstSubmissionDate, 
        Applicant_Province 
    FROM 
        application 
    WHERE 
        MONTH(FirstSubmissionDate) = 3) AS SourceTable
PIVOT
(
    COUNT(Applicant_Province)
    FOR Applicant_Province IN ([ON])
) AS PivotTable;
```


# Appendix
-- Importing files and data cleaning
```sql
SELECT 
    t.name AS 'Table Name',
    c.name AS 'Column Name'
FROM 
    sys.tables t
JOIN 
    sys.columns c
ON 
    t.object_id = c.object_id
ORDER BY 
    t.name, c.name;

EXEC sp_rename 'lookup.column1', 'Province', 'COLUMN';
EXEC sp_rename 'lookup.column2', '5_Ride', 'COLUMN';
EXEC sp_rename 'lookup.column3', '4_Ride', 'COLUMN';
EXEC sp_rename 'lookup.column4', '3_Ride', 'COLUMN';
EXEC sp_rename 'lookup.column5', '2_Ride', 'COLUMN';
EXEC sp_rename 'lookup.column6', 'EP_Ride+', 'COLUMN';

ALTER TABLE application DROP COLUMN column6;

SELECT name FROM sys.tables;

SELECT COLUMN_NAME 
FROM INFORMATION_SCHEMA.COLUMNS 
WHERE TABLE_NAME = 'lookup' ;

-- check non-numberic columns and make them numeric
DELETE FROM customers;

SELECT *
FROM customers
WHERE ISNUMERIC(Financed_Amount) = 0;

SELECT *
FROM customers
WHERE ISNUMERIC(Actual_Interest_Rate) = 0;
```

-- I found Actual_Interest_Rate is stored as "13.0%" and I'd like to have it as "0.13"
```sql
ALTER TABLE customers
ADD Actual_Interest_Rate_num float;

UPDATE customers
SET Actual_Interest_Rate = CAST(REPLACE(Actual_Interest_Rate, '%', '') AS float) / 100;

ALTER TABLE customers
DROP COLUMN Actual_Interest_Rate;

EXEC sp_rename 'customers.Actual_Interest_Rate_num', 'Actual_Interest_Rate', 'COLUMN';
```
