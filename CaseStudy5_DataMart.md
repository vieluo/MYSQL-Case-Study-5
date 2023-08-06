# 8 Weeks SQL Challenge - 5th Week

### This [case study](https://8weeksqlchallenge.com/case-study-5/)  is provided by Danny Ma

## Introduction
Data Mart is Danny’s latest venture and after running international operations for his online supermarket that specialises in fresh produce - Danny is asking for your support to analyse his sales performance.

In June 2020 - large scale supply changes were made at Data Mart. All Data Mart products now use sustainable packaging methods in every single step from the farm all the way to the customer.

Danny needs your help to quantify the impact of this change on the sales performance for Data Mart and it’s separate business areas.

The key business question he wants you to help him answer are the following:

* What was the quantifiable impact of the changes introduced in June 2020?
* Which platform, region, segment and customer types were the most impacted by this change?
* What can we do about future introduction of similar sustainability updates to the business to minimise impact on sales?

The database can be found [HERE](https://8weeksqlchallenge.com/case-study-5/)

## Data Cleansing Steps
1. Convert the week_date to a DATE format

We can check the data type of the columns by

```sql
describe data_mart
```

| Field         | Type        | Null| 
|---------------|-------------|-----|
| week_date     | varchar(7)  | YES |  
| region        | varchar(13) | YES | 
| platform      | varchar(7)  | YES | 
| segment       | varchar(4)  | YES | 
| customer_type | varchar(8)  | YES |  
| transactions  | int         | YES |   
| sales         | int         | YES |  

Now we modity the week_date to date type.

```sql
ALTER TABLE data_mart
MODIFY week_date date

describe data_mart
```
| Field         | Type        | Null| 
|---------------|-------------|-----|
| week_date     | date        | YES |  
| region        | varchar(13) | YES | 
| platform      | varchar(7)  | YES | 
| segment       | varchar(4)  | YES | 
| customer_type | varchar(8)  | YES |  
| transactions  | int         | YES |   
| sales         | int         | YES |  

But in this case, the date type is automatically as yy/mm/dd which is not correct, we need it as dd/mm/yy

```sql
UPDATE data_mart
SET week_date = DATE_FORMAT(week_date, '%d/%m/%y')
```


2. Add a week_number as the second column for each week_date value, for example any value from the 1st of January to 7th of January will be 1, 8th to 14th will be 2 etc

```sql
ALTER TABLE data_mart
ADD number_week INTEGER
AFTER date_date
```
```sql
UPDATE data_mart
SET week_number = WEEK(week_date)
```

3. Add a month_number with the calendar month for each week_date value as the 3rd column

```sql
ALTER TABLE data_mart
ADD month INTEGER
AFTER week_number

UPDATE data_mart
SET month = month(week_date)
```

4. Add a calendar_year column as the 4th column containing either 2018, 2019 or 2020 values

```sql
ALTER TABLE data_mart
ADD year INTEGER
AFTER month

UPDATE data_mart
SET year = year(week_date)
```

5. Add a new column called age_band after the original segment column using the following mapping on the number inside the segment value

| segment | age_band      |
|---------|---------------|
| 1       | Young Adults  |
| 2       | Middle Aged   |
| 3 or 4  | Retirees      |

```sql
ALTER TABLE data_mart
ADD age_band VARCHAR(20)
AFTER segment

UPDATE data_mart
SET age_band = CASE 
WHEN segment LIKE '%1' THEN 'Young Adults'
WHEN segment LIKE '%2' THEN 'Middle Aged'
WHEN segment LIKE '%3' OR segment LIKE '%4' THEN 'Retirees'
ELSE NULL
END
```

6. Add a new demographic column using the following mapping for the first letter in the segment values:
| segment | demographic  |
|---------|--------------|
| C       | Couples      |
| F       | Families     |

```sql
ALTER TABLE data_mart
ADD demographic VARCHAR(10)
AFTER age_band

UPDATE data_mart
SET demographic = CASE 
WHEN segment LIKE 'C%' THEN 'Couples'
WHEN segment LIKE 'F%' THEN 'Families'
ELSE NULL
END
```

7. Enure all null string values with an "unknown" string value in the original segment column as well as the new age_band and demographic columns

```sql
UPDATE data_mart
SET demographic = 'unknown'
WHERE demographic IS NULL

UPDATE data_mart
SET age_band = 'unknown'
WHERE age_band IS NULL
```

How ever, then I try to update the column 'segment' is shows erro: Data too long for column 'segment' at row 3
Because the data type for this column is VARCHAR(4) we need to increase the number so that we can input a longer string. 

```sql
ALTER TABLE data_mart
MODIFY segment VARCHAR(10)	

UPDATE data_mart
SET segment = 'unknown'
WHERE segment = 'null'
```

8. Generate a new avg_transaction column as the sales value divided by transactions rounded to 2 decimal places for each record

```sql
ALTER TABLE data_mart
ADD COLUMN avg_transaction DECIMAL(10,2)

UPDATE data_mart
SET avg_transaction = sales/transactions
```

The finally resualt of these cleaning processes is: 

|week_date|week_number|month|year| region|platform|segment|age_band|demographic|customer_type|transactions|sales| avg_transaction |
|------------|---|---|------|--------|---------|---------|--------------|----------|----------|--------|----------|---------|
| 2020-08-31 | 35 | 8 | 2020 | ASIA   | Retail  | C3      | Retirees     | Couples  | New      | 120631 | 3656163  | 30.31   |
| 2020-08-31 | 35 | 8 | 2020 | ASIA   | Retail  | F1      | Young Adults | Families | New      | 31574  | 996575   | 31.56   |
| 2020-08-31 | 35 | 8 | 2020 | USA    | Retail  | unknown | unknown      | unknown  | Guest    | 529151 | 16509610 | 31.20   |
| 2020-08-31 | 35 | 8 | 2020 | EUROPE | Retail  | C1      | Young Adults | Couples  | New      | 4517   | 141942   | 31.42   |
| 2020-08-31 | 35 | 8 | 2020 | AFRICA | Retail  | C2      | Middle Aged  | Couples  | New      | 58046  | 1758388  | 30.29   |
| 2020-08-31 | 35 | 8 | 2020 | CANADA | Shopify | F2      | Middle Aged  | Families | Existing | 1336   | 243878   | 182.54  |
| 2020-08-31 | 35 | 8 | 2020 | AFRICA | Shopify | F3      | Retirees     | Families | Existing | 2514   | 519502   | 206.64  |


## Data Exploration

1. What day of the week is used for each week_date value?

```sql
SELECT dayofweek(week_date), count(week_date)
FROM data_mart
GROUP BY dayofweek(week_date)
```


|week_date|week_number|
|---------|-----------|
|    2    |   34234   |

We can know that it use only Monday for week_date value.

2. What range of week numbers are missing from the dataset?

```sql
SELECT count(distinct(week_number))
FROM data_mart
```

|count(distinct(week_number))|
|---------|
|    24   |

There were 53 weeks in 2020, so 29 rang of week numnbers are missing.

3. How many total transactions were there for each year in the dataset?

```sql
SELECT year, sum(transactions) AS total_transactions
FROM data_mart
GROUP BY year
```

| year |total_transactions|
|------|------------|
| 2020 | 751627302  | 
| 2019 | 731278570  | 
| 2018 | 692812920  | 

4. What is the total sales for each region for each month?

```sql
SELECT region, month, COUNT(sales) AS num_sales, SUM(sales) AS total_sales
FROM data_mart
GROUP BY region,month
ORDER BY month
```
|    region     | month | num_sales | total_sales |
|---------------|---|-----|-------------|
| ASIA          | 3 | 272 | 1059541586  |
| AFRICA        | 3 | 272 | 1135534960  |
| OCEANIA       | 3 | 272 | 1566565776  |
| EUROPE        | 3 | 270 | 70674186    |
| SOUTH AMERICA | 3 | 272 | 142046218   |
| USA           | 3 | 272 | 450706086   |
| CANADA        | 3 | 272 | 289268658   |
| EUROPE        | 4 | 948 | 254668510   |
| ASIA          | 4 | 952 | 3609257414  |


5. What is the total count of transactions for each platform

```sql
SELECT platform, SUM(transactions) AS total_transactions
FROM data_mart
GROUP BY platform
```

| platform|total_transactions|  
|---------|-------------|
| Retail  | 2163868454  | 
| Shopify | 11850338    | 

6. What is the percentage of sales for Retail vs Shopify for each month?

```sql
WITH sum_total AS (
SELECT month, SUM(sales) AS total_sales
FROM data_mart
GROUP BY month
)


SELECT platform, ROUND(SUM(sales)/total_sales*100,2) AS percentage, data_mart.month
FROM data_mart, sum_total
WHERE sum_total.month = data_mart.month
GROUP BY platform,month
ORDER BY month
```

| platform|percentage|month|
|---------|-------|----|
| Shopify | 2.46  | 3  |
| Retail  | 97.54 | 3  |
| Retail  | 97.59 | 4  |
| Shopify | 2.41  | 4  |
| Retail  | 97.30 | 5  |
| Shopify | 2.70  | 5  |
| Shopify | 2.73  | 6  |
| Retail  | 97.27 | 6  |
| Shopify | 2.73  | 6  |
| Retail  | 97.27 | 6  |


7. What is the percentage of sales by demographic for each year in the dataset?

```sql
WITH total_sales AS ( 
SELECT year, SUM(sales) AS total_sales
FROM data_mart
GROUP BY year
)

SELECT data_mart.year, demographic, ROUND(SUM(sales)/total_sales*100,2) AS percentage_sales_by_demographic
FROM data_mart, total_sales
WHERE total_sales.year = data_mart.year
GROUP BY data_mart.year, demographic
ORDER BY data_mart.year
```

| year | demographic | percentage_sales_by_demographic |
| ---- | -------- | ----- |
| 2018 | Couples  | 26.38 |
| 2018 | Families | 31.99 |
| 2018 | unknown  | 41.63 |
| 2019 | Couples  | 27.28 |
| 2019 | Families | 32.47 |
| 2019 | unknown  | 40.25 |
| 2020 | Couples  | 28.72 |
| 2020 | Families | 32.73 |
| 2020 | unknown  | 38.55 |


8. Which age_band and demographic values contribute the most to Retail sales?

```sql 
SELECT platform, age_band, demographic, SUM(sales) AS sales_by_age_demographic
FROM data_mart
WHERE platform = 'Retail' AND age_band != 'unknown' AND demographic != 'unknown'
GROUP BY age_band, demographic
ORDER BY SUM(sales) DESC
LIMIT 1
```

| platform | age_band | demographic | sales_by_age_demographic | 
| ------ | -------- | -------- |------------ |
| Retail | Retirees	| Families | 13269373832 |


10. Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?

```sql 
SELECT year, platform, AVG (avg_transaction) AS avg_transaction
FROM data_mart
GROUP BY year, platform
ORDER BY year
```

| year | platform | avg_transaction |
| ---- | ------- | ---------- |
| 2018 | Retail  | 42.906369  |
| 2018 | Shopify | 188.279272 |
| 2019 | Retail  | 41.968071  |
| 2019 | Shopify | 177.559562 |
| 2020 | Retail  | 40.640231  |
| 2020 | Shopify | 174.873569 |


## Before & After Analysis

This technique is usually used when we inspect an important event and want to inspect the impact before and after a certain point in time.

Taking the week_date value of 2020-06-15 as the baseline week where the Data Mart sustainable packaging changes came into effect.

We would include all week_date values for 2020-06-15 as the start of the period after the change and the previous week_date values would be before

Using this analysis approach - answer the following questions:

1. What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?

Fitst I checked which week number and weekday is for date 2020-06-15.

```sql 
SELECT week_date, week_number, weekday(week_date)
FROM data_mart
WHERE week_date = '2020-06-15'
```

| week_date | week_number | weekday(week_date) |
| ---------- | -- | - |
| 2020-06-15 | 24 | 0 |

We can see that 2020-06-15 is a Monday and we can use weeK_number = 24 as a seperation value, dont neet to take weekday into consideration.
I created 2 tables, one is 4 weeks after change (including the week 24), another is 4 week before change. I compared total sales and transactions values.

```sql
WITH after_change AS (
SELECT SUM(sales) AS total_sales, SUM(transactions) AS total_transactions
FROM data_mart
WHERE week_number >=24 AND week_number < 24+4 AND YEAR(week_date) = 2020
),

before_change AS (
SELECT SUM(sales) AS total_sales, SUM(transactions) AS total_transactions
FROM data_mart
WHERE week_number <24 AND week_number >= 24-4 AND YEAR(week_date) = 2020
)

SELECT after_change.total_sales-before_change.total_sales AS changed_sales, ROUND((after_change.total_sales-before_change.total_sales)/before_change.total_sales* 100,2) AS changed_percentage_sales,
after_change.total_transactions - before_change.total_transactions AS changed_transactions, ROUND (after_change.total_transactions/(after_change.total_transactions - before_change.total_transactions) *100, 2) AS changed_percentage_transactions
FROM after_change, before_change
```

| changed_sales | changed_percentage_sales | changed_transactions | changed_percentage_transactions | 
| ----------| --------- | ------ | --------- |
| -53768376	| -1.15	| -249332| -0.20 |

We can say that in the first 4 weeks after change, there is not positive resulte from the implementation.


2. What about the entire 12 weeks before and after?

```sql
WITH after_change AS (
SELECT SUM(sales) AS total_sales, SUM(transactions) AS total_transactions
FROM data_mart
WHERE week_number >=24 AND week_number < 24+12 AND YEAR(week_date) = 2020
),

before_change AS (
SELECT SUM(sales) AS total_sales, SUM(transactions) AS total_transactions
FROM data_mart
WHERE week_number <24 AND week_number >= 24-12 AND YEAR(week_date) = 2020
)

SELECT after_change.total_sales-before_change.total_sales AS changed_sales, ROUND((after_change.total_sales-before_change.total_sales)/before_change.total_sales* 100,2) AS changed_percentage_sales,
after_change.total_transactions - before_change.total_transactions AS changed_transactions, ROUND ((after_change.total_transactions - before_change.total_transactions)/before_change.total_transactions*100, 2) AS changed_percentage_transactions
FROM after_change, before_change
```

| changed_sales | changed_percentage_sales | changed_transactions | changed_percentage_transactions | 
| ---------- | --------- | ------ | --------- |
| -304650788 | -2.14	| -2501826| -0.66 |

We can say that in 12 weeks the decrease of sales is less significant compared to the first 4 weeks, however, the sales and trnasactions still droped greatly after the implementation. 

3. How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?

```sql
WITH temp1 AS (
WITH after_change_12weeks AS (
SELECT year, SUM(sales) AS total_sales
FROM data_mart
WHERE week_number >=24 AND week_number < 24+12
GROUP BY year
),

before_change_12weeks AS (
SELECT year, SUM(sales) AS total_sales
FROM data_mart
WHERE week_number <24 AND week_number >= 24-12
GROUP BY year
)

SELECT after_change_12weeks.year, after_change_12weeks.total_sales-before_change_12weeks.total_sales AS changed_sales_12weeks, ROUND((after_change_12weeks.total_sales-before_change_12weeks.total_sales)/before_change_12weeks.total_sales* 100,2) AS changed_percentage_sales_12weeks
FROM after_change_12weeks, before_change_12weeks
WHERE after_change_12weeks.year = before_change_12weeks.year
),

temp2 AS (
WITH after_change_4weeks AS (
SELECT year, SUM(sales) AS total_sales
FROM data_mart
WHERE week_number >=24 AND week_number < 24+4
GROUP BY year
),

before_change_4weeks AS (
SELECT year, SUM(sales) AS total_sales
FROM data_mart
WHERE week_number <24 AND week_number >= 24-4
GROUP BY year
)

SELECT after_change_4weeks.year, after_change_4weeks.total_sales-before_change_4weeks.total_sales AS changed_sales_4weeks, ROUND((after_change_4weeks.total_sales-before_change_4weeks.total_sales)/after_change_4weeks.total_sales * 100,2) AS changed_percentage_sales_4weeks
FROM after_change_4weeks, before_change_4weeks
WHERE after_change_4weeks.year = before_change_4weeks.year
)

SELECT temp1.year, temp1.changed_sales_12weeks, temp1.changed_percentage_sales_12weeks, temp2.changed_sales_4weeks, temp2.changed_percentage_sales_4weeks
FROM temp1 
JOIN temp2 ON
temp1.year = temp2.year
```

| year | changed_sales_12weeks | changed_percentage_sales_12weeks | changed_sales_4weeks | changed_percentage_sales_4weeks |
| ---- | ----------- | ---------- | ---------- | --------- |
| 2020 | \-304650788 | \-2.14  | \-53768376 | \-1.15 |
| 2019 | \-41480588  | \-0.30 | 4673188    | 0.10  |
| 2018 | 208512386   | 1.63   | 8204210    | 0.19  |

## Bonus Question

Which areas of the business have the highest negative impact in sales metrics performance in 2020 for the 12 week before and after period?

* region
* platform
* age_band
* demographic
* customer_type
Do you have any further recommendations for Danny’s team at Data Mart or any interesting insights based off this analysis?

```sql
-- top 1 changed sales by retail 

WITH retail AS (
WITH after_change AS (
SELECT year, platform, SUM(sales) AS total_sales
FROM data_mart
WHERE week_number >=24 AND week_number < 24+12 AND YEAR(week_date) = 2020
GROUP BY platform, year
),

before_change AS (
SELECT platform,  SUM(sales) AS total_sales
FROM data_mart
WHERE week_number <24 AND week_number >= 24-12 AND YEAR(week_date) = 2020
GROUP BY platform
)

SELECT year, after_change.platform, after_change.total_sales-before_change.total_sales AS changed_sales, ROUND((after_change.total_sales-before_change.total_sales)/after_change.total_sales * 100,2) AS changed_percentage_sales
FROM after_change, before_change
WHERE after_change.platform = before_change.platform
ORDER BY changed_sales
LIMIT 1
),

-- top 1 changed sales by region

region AS (
WITH after_change AS (
SELECT year, region, SUM(sales) AS total_sales
FROM data_mart
WHERE week_number >=24 AND week_number < 24+12 AND YEAR(week_date) = 2020
GROUP BY region,year
),

before_change AS (
SELECT region,  SUM(sales) AS total_sales
FROM data_mart
WHERE week_number <24 AND week_number >= 24-12 AND YEAR(week_date) = 2020
GROUP BY region
)

SELECT year, after_change.region, after_change.total_sales-before_change.total_sales AS changed_sales, ROUND((after_change.total_sales-before_change.total_sales)/after_change.total_sales * 100,2) AS changed_percentage_sales
FROM after_change, before_change
WHERE after_change.region = before_change.region
ORDER BY changed_sales
LIMIT 1
), 

-- top 1 changed sales by age_band 

age_band AS (
WITH after_change AS (
SELECT year, age_band, SUM(sales) AS total_sales
FROM data_mart
WHERE week_number >=24 AND week_number < 24+12 AND YEAR(week_date) = 2020 AND age_band != 'unknown'
GROUP BY age_band,year
),

before_change AS (
SELECT age_band,  SUM(sales) AS total_sales
FROM data_mart
WHERE week_number <24 AND week_number >= 24-12 AND YEAR(week_date) = 2020 AND age_band != 'unknown'
GROUP BY age_band
)

SELECT year, after_change.age_band, after_change.total_sales-before_change.total_sales AS changed_sales, ROUND((after_change.total_sales-before_change.total_sales)/after_change.total_sales * 100,2) AS changed_percentage_sales
FROM after_change, before_change
WHERE after_change.age_band = before_change.age_band 
ORDER BY changed_sales
LIMIT 1
),

-- top 1 changed sales by demographic 

demographic AS (
WITH after_change AS (
SELECT year, demographic, SUM(sales) AS total_sales
FROM data_mart
WHERE week_number >=24 AND week_number < 24+12 AND YEAR(week_date) = 2020 AND demographic != 'unknown'
GROUP BY demographic, year
),

before_change AS (
SELECT demographic,  SUM(sales) AS total_sales
FROM data_mart
WHERE week_number <24 AND week_number >= 24-12 AND YEAR(week_date) = 2020 AND demographic != 'unknown'
GROUP BY demographic
)

SELECT year, after_change.demographic, after_change.total_sales-before_change.total_sales AS changed_sales, ROUND((after_change.total_sales-before_change.total_sales)/after_change.total_sales * 100,2) AS changed_percentage_sales
FROM after_change, before_change
WHERE after_change.demographic = before_change.demographic 
ORDER BY changed_sales
LIMIT 1
),

-- top 1 changed sales by customer_type 

customer_type AS (
WITH after_change AS (
SELECT year, customer_type, SUM(sales) AS total_sales
FROM data_mart
WHERE week_number >=24 AND week_number < 24+12 AND YEAR(week_date) = 2020
GROUP BY customer_type, year
),

before_change AS (
SELECT customer_type,  SUM(sales) AS total_sales
FROM data_mart
WHERE week_number <24 AND week_number >= 24-12 AND YEAR(week_date) = 2020
GROUP BY customer_type
)

SELECT year, after_change.customer_type, after_change.total_sales-before_change.total_sales AS changed_sales, ROUND((after_change.total_sales-before_change.total_sales)/after_change.total_sales * 100,2) AS changed_percentage_sales
FROM after_change, before_change
WHERE after_change.customer_type = before_change.customer_type 
ORDER BY changed_sales
LIMIT 1
)

SELECT *
FROM retail
JOIN region ON
retail.year = region.year
JOIN age_band ON
retail.year = age_band.year
JOIN demographic ON
retail.year = demographic.year
JOIN customer_type ON
retail.year = customer_type.year
```



|year| platform| changed_sales| changed_percentage_sales| year| region| changed_sales| changed_percentage_sales| year| age_band| changed_sales| changed_percentage_sales| year| demographic| changed_sales| changed_percentage_sales| year| customer_type| changed_sales| changed_percentage_sales
| ---- | ----------- | ---------- | ---------- | --------- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- |
|2020 | Retail| -336167668| -2.49| 2020| OCEANIA| -142642200| -3.12| 2020| Retirees| -59099042| -1.| 2020| Families| -84640030| -1.85	2020| Existing| -167745946| -2.33

We can see that the change have most impact on Region - Oceania

