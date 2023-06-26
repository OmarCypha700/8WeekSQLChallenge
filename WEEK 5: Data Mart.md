# WEEK 5: Data Mart 🏪

## Table of Contents
- [Data Cleaning Steps](#a-data-cleaning-steps)
- [Data Extrapolation](#b-data-extrapolation)
- [Before & After Analysis](#c-before-&-after-analysis)
- [Bonus Question](#d-bonus-questions)

## A. Data Cleaning Steps

#### In a single query, perform the following operations and generate a new table in the data_mart schema named clean_weekly_sales:
- Convert the week_date to a DATE format

- Add a week_number as the second column for each week_date value, for example any value from the 1st of January to 7th of January will be 1, 8th to 14th will be 2 etc

- Add a month_number with the calendar month for each week_date value as the 3rd column

- Add a calendar_year column as the 4th column containing either 2018, 2019 or 2020 values

- Add a new column called age_band after the original segment column using the following mapping on the number inside the segment value

- Add a new demographic column using the following mapping for the first letter in the segment values

- Ensure all null string values with an "unknown" string value in the original segment column as well as the new age_band and demographic columns

- Generate a new avg_transaction column as the sales value divided by transactions rounded to 2 decimal places for each record

```sql
DROP TABLE IF EXISTS clean_weekly_sales;

CREATE TABLE clean_weekly_sales AS
SELECT  
		STR_TO_DATE(week_date, '%d/%m/%y') AS week_date,
		WEEK(STR_TO_DATE(week_date, '%d/%m/%y')) AS week_number,
		MONTH(STR_TO_DATE(week_date, '%d/%m/%y')) AS month_number,
		YEAR(STR_TO_DATE(week_date, '%d/%m/%y')) AS calendar_year,
		region,
		platform,
        CASE WHEN segment LIKE 'C%' OR segment LIKE 'F%' THEN segment
             ELSE 'unkown' END AS segment,
		CASE 
			WHEN segment LIKE '%1' THEN 'Young Adults'
			WHEN segment LIKE '%2' THEN 'Middle Aged'
			WHEN segment LIKE '%3' OR segment LIKE '%4' THEN 'Retirees'
			ELSE 'unknown' END AS age_band,
		CASE 
			WHEN segment LIKE 'C%' THEN 'Couples'
			WHEN segment LIKE 'F%' THEN 'Families'
			ELSE 'unknown' END AS demographic,
		customer_type,
		transactions,
		sales,
		ROUND(sales/transactions, 2) AS avg_transaction
FROM (
SELECT  week_date,
		region,
        platform,
        customer_type,
        transactions,
        sales,
        segment
FROM data_mart.weekly_sales
) AS clean_data;

SELECT *
FROM data_mart.clean_weekly_sales;
```

![SA_Q1](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/a7c4736e-0033-4f4e-9cda-6cc57d3a25ee)

## B. Data Extrapolation

#### What day of the week is used for each week_date value?

```sql
SELECT DAYNAME(week_date) AS day_of_week,
	   COUNT(week_date)AS week_date_count
FROM data_mart.clean_weekly_sales
GROUP BY day_of_week;
```

![SB_Q1](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/21b4180d-7a5b-4cff-b0d1-52a73ffac37b)

#### What range of week numbers are missing from the dataset?

```sql
WITH RECURSIVE numbers AS (
               SELECT 0 AS weeks
               UNION
               SELECT weeks + 1
               FROM numbers
               WHERE weeks < 52
               )
SELECT weeks FROM numbers 
WHERE weeks NOT IN (
		  SELECT DISTINCT(week_number) AS week_number 
		  FROM data_mart.clean_weekly_sales
                );
```

![SB_Q2](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/7e1088f4-5efc-4e6e-b863-47284571a7bc)

#### How many total transactions were there for each year in the dataset?

```sql
SELECT calendar_year,
	   COUNT(transactions) AS total_transactions_count
FROM data_mart.clean_weekly_sales
GROUP BY calendar_year;
```

![SB_Q3](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/fe64e5f8-ddea-4c1f-ba3e-cb8e3748c48f)

#### What is the total sales for each region for each month?

```sql
SELECT region, MONTHNAME(week_date) AS month,
	   SUM(sales) AS total_sales
FROM data_mart.clean_weekly_sales
GROUP BY region, month
ORDER BY total_sales DESC;
```

![SB_Q4](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/e4dd8fa7-dbcb-4d91-ab8d-1713aa8213d5)

#### What is the total count of transactions for each platform

```sql
SELECT platform,
	   COUNT(transactions) AS transactions_count
FROM data_mart.clean_weekly_sales
GROUP BY platform;
```

![SB_Q5](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/b1356004-ebc7-4a04-879d-7e43afbed32c)

#### What is the percentage of sales for Retail vs Shopify for each month?

```sql
SELECT
    MONTHNAME(week_date) AS month,
    SUM(CASE WHEN platform = 'Retail' THEN sales ELSE 0 END) AS retail_sales,
    SUM(CASE WHEN platform = 'Shopify' THEN sales ELSE 0 END) AS shopify_sales,
    ROUND((SUM(CASE WHEN platform = 'Retail' THEN sales ELSE 0 END) / SUM(sales)) * 100, 1) AS retail_percentage,
    ROUND((SUM(CASE WHEN platform = 'Shopify' THEN sales ELSE 0 END) / SUM(sales)) * 100, 1) AS shopify_percentage
FROM data_mart.clean_weekly_sales
GROUP BY month;
```

![SB_Q6](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/c27a2c92-d920-4f5d-8c17-98e2aed1740f)

#### What is the percentage of sales by demographic for each year in the dataset?

```sql
SELECT
    YEAR(week_date) AS year,
    SUM(CASE WHEN demographic = 'Couples' THEN sales ELSE 0 END) AS couples_sales,
	  ROUND((SUM(CASE WHEN demographic = 'Couples' THEN sales ELSE 0 END) / SUM(sales)) * 100, 1) AS couples_percentage,
    SUM(CASE WHEN demographic = 'Families' THEN sales ELSE 0 END) AS families_sales,
    ROUND((SUM(CASE WHEN demographic = 'Families' THEN sales ELSE 0 END) / SUM(sales)) * 100, 1) AS families_percentage,
    SUM(CASE WHEN demographic = 'unknown' THEN sales ELSE 0 END) AS unknown_sales,
    ROUND((SUM(CASE WHEN demographic = 'unknown' THEN sales ELSE 0 END) / SUM(sales)) * 100, 1) AS unknown_percentage
FROM data_mart.clean_weekly_sales
GROUP BY year;

```

![SB_Q7](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/574c1908-3d0e-480e-bcb8-35a40850e3aa)

#### Which age_band and demographic values contribute the most to Retail sales?

```sql
SELECT age_band,
	   demographic,
       SUM(sales) AS total_sales
FROM data_mart.clean_weekly_sales
WHERE platform = 'Retail' AND demographic != 'unknown'
GROUP BY age_band, demographic
ORDER BY total_sales DESC
LIMIT 1;
```

![SB_Q8](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/9ae54207-5b74-46fd-9335-3aebac888978)

#### Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?

```sql
SELECT
    YEAR(week_date) AS year,
    ROUND(AVG(CASE WHEN platform = 'Retail' THEN avg_transaction END), 2) AS retail_avg_transaction,
    ROUND(AVG(CASE WHEN platform = 'Shopify' THEN avg_transaction END), 2) AS shopify_avg_transaction
FROM clean_weekly_sales
GROUP BY year;
```

![SB_Q9](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/04f0abfa-9b0d-43e6-80db-715ac8fd24b2)

## C. Before & After Analysis

This technique is usually used when we inspect an important event and want to inspect the impact before and after a certain point in time.

Taking the week_date value of 2020-06-15 as the baseline week where the Data Mart sustainable packaging changes came into effect.

We would include all week_date values for 2020-06-15 as the start of the period after the change and the previous week_date values would be before

Using this analysis approach - answer the following questions:

#### What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?

```sql
SELECT total_sales_before, 
       total_sales_after,
       (total_sales_before - total_sales_after) AS sales_difference,
       ROUND(((total_sales_after - total_sales_before) / total_sales_before) * 100, 1) AS growth_rate
FROM (SELECT
    SUM(CASE WHEN week_date BETWEEN DATE_SUB('2020-06-15', INTERVAL 4 WEEK) AND '2020-06-15' THEN sales ELSE 0 END) AS total_sales_before,
    SUM(CASE WHEN week_date BETWEEN '2020-06-15' AND DATE_ADD('2020-06-15', INTERVAL 4 WEEK) THEN sales ELSE 0 END) AS total_sales_after
FROM clean_weekly_sales
) AS sales_before_and_after;
```

![SC_Q1](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/b071b406-29d8-42eb-a930-873ffdf65565)

#### What about the entire 12 weeks before and after?

```sql
SELECT total_sales_before, 
       total_sales_after,
       (total_sales_before - total_sales_after) AS sales_difference,
       ROUND(((total_sales_after - total_sales_before) / total_sales_before) * 100, 1) AS growth_rate
FROM (SELECT
    SUM(CASE WHEN week_date BETWEEN DATE_SUB('2020-06-15', INTERVAL 12 WEEK) AND '2020-06-15' THEN sales ELSE 0 END) AS total_sales_before,
    SUM(CASE WHEN week_date BETWEEN '2020-06-15' AND DATE_ADD('2020-06-15', INTERVAL 12 WEEK) THEN sales ELSE 0 END) AS total_sales_after
FROM clean_weekly_sales
) AS sales_before_and_after;
```

![SC_Q2](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/94fa1568-2be0-4732-9e69-0eb72da5dba0)

#### How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?

```sql
WITH sales_2020 AS (
SELECT year,
       (total_sales_after_12weeks - total_sales_before_12weeks) AS actual_value_12weeks, 
       ROUND(((total_sales_after_12weeks - total_sales_before_12weeks) / total_sales_before_12weeks) * 100, 1) AS growth_rate_12weeks,
       (total_sales_after_4weeks - total_sales_before_4weeks) AS actual_value_4weeks, 
       ROUND(((total_sales_after_4weeks - total_sales_before_4weeks) / total_sales_before_4weeks) * 100, 1) AS growth_rate_4weeks
FROM ( SELECT calendar_year AS year,
       SUM(CASE WHEN week_date BETWEEN DATE_SUB('2020-06-15', INTERVAL 12 WEEK) AND '2020-06-15' THEN sales ELSE 0 END) AS total_sales_before_12weeks,
       SUM(CASE WHEN week_date BETWEEN DATE_SUB('2020-06-15', INTERVAL 4 WEEK) AND '2020-06-15' THEN sales ELSE 0 END) AS total_sales_before_4weeks,
       SUM(CASE WHEN week_date BETWEEN '2020-06-15' AND DATE_ADD('2020-06-15', INTERVAL 12 WEEK) THEN sales ELSE 0 END) AS total_sales_after_12weeks,
       SUM(CASE WHEN week_date BETWEEN '2020-06-15' AND DATE_ADD('2020-06-15', INTERVAL 4 WEEK) THEN sales ELSE 0 END) AS total_sales_after_4weeks
FROM data_mart.clean_weekly_sales
GROUP BY year) AS sales_2020
),
sales_2019 AS (
SELECT year,
       (total_sales_after_12weeks - total_sales_before_12weeks) AS actual_value_12weeks, 
       ROUND(((total_sales_after_12weeks - total_sales_before_12weeks) / total_sales_before_12weeks) * 100, 1) AS growth_rate_12weeks,
       (total_sales_after_4weeks - total_sales_before_4weeks) AS actual_value_4weeks, 
       ROUND(((total_sales_after_4weeks - total_sales_before_4weeks) / total_sales_before_4weeks) * 100, 1) AS growth_rate_4weeks
FROM ( SELECT calendar_year AS year,
       SUM(CASE WHEN week_date BETWEEN DATE_SUB('2019-06-15', INTERVAL 12 WEEK) AND '2019-06-15' THEN sales ELSE 0 END) AS total_sales_before_12weeks,
       SUM(CASE WHEN week_date BETWEEN DATE_SUB('2019-06-15', INTERVAL 4 WEEK) AND '2019-06-15' THEN sales ELSE 0 END) AS total_sales_before_4weeks,
       SUM(CASE WHEN week_date BETWEEN '2019-06-15' AND DATE_ADD('2019-06-15', INTERVAL 12 WEEK) THEN sales ELSE 0 END) AS total_sales_after_12weeks,
       SUM(CASE WHEN week_date BETWEEN '2019-06-15' AND DATE_ADD('2019-06-15', INTERVAL 4 WEEK) THEN sales ELSE 0 END) AS total_sales_after_4weeks
FROM data_mart.clean_weekly_sales
GROUP BY year) AS sales_2019
),
sales_2018 AS (
SELECT year,
      (total_sales_after_12weeks - total_sales_before_12weeks) AS actual_value_12weeks, 
      ROUND(((total_sales_after_12weeks - total_sales_before_12weeks) / total_sales_before_12weeks) * 100, 1) AS growth_rate_12weeks,
      (total_sales_after_4weeks - total_sales_before_4weeks) AS actual_value_4weeks, 
      ROUND(((total_sales_after_4weeks - total_sales_before_4weeks) / total_sales_before_4weeks) * 100, 1) AS growth_rate_4weeks
FROM ( SELECT calendar_year AS year,
       SUM(CASE WHEN week_date BETWEEN DATE_SUB('2018-06-15', INTERVAL 12 WEEK) AND '2018-06-15' THEN sales ELSE 0 END) AS total_sales_before_12weeks,
       SUM(CASE WHEN week_date BETWEEN DATE_SUB('2018-06-15', INTERVAL 4 WEEK) AND '2018-06-15' THEN sales ELSE 0 END) AS total_sales_before_4weeks,
       SUM(CASE WHEN week_date BETWEEN '2018-06-15' AND DATE_ADD('2018-06-15', INTERVAL 12 WEEK) THEN sales ELSE 0 END) AS total_sales_after_12weeks,
       SUM(CASE WHEN week_date BETWEEN '2018-06-15' AND DATE_ADD('2018-06-15', INTERVAL 4 WEEK) THEN sales ELSE 0 END) AS total_sales_after_4weeks
FROM data_mart.clean_weekly_sales
GROUP BY year) AS sales_2018
)
SELECT *
FROM sales_2020
WHERE growth_rate_4weeks IS NOT NULL AND growth_rate_12weeks IS NOT NULL
UNION
SELECT *
FROM sales_2019
WHERE growth_rate_4weeks IS NOT NULL AND growth_rate_12weeks IS NOT NULL
UNION
SELECT *
FROM sales_2018
WHERE growth_rate_4weeks IS NOT NULL AND growth_rate_12weeks IS NOT NULL;
```

![SC_Q3](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/aaaeb3e5-39dc-4e83-9d25-f79ef5006900)

## D. Bonus Question

#### Which areas of the business have the highest negative impact in sales metrics performance in 2020 for the 12 week before and after period?

- region
- platform
- age_band
- demographic
- customer_type
  
Do you have any further recommendations for Danny’s team at Data Mart or any interesting insights based off this analysis?

```sql
WITH region AS (
SELECT year, region,
       total_sales_before, 
       total_sales_after,
       (total_sales_before - total_sales_after) AS sales_difference,
       ROUND(((total_sales_after - total_sales_before) / total_sales_before) * 100, 1) AS growth_rate
FROM (
      SELECT 
      calendar_year AS year, 
      region,
      SUM(CASE WHEN week_date BETWEEN DATE_SUB('2020-06-15', INTERVAL 12 WEEK) AND '2020-06-15' THEN sales ELSE 0 END) AS total_sales_before,
      SUM(CASE WHEN week_date BETWEEN '2020-06-15' AND DATE_ADD('2020-06-15', INTERVAL 12 WEEK) THEN sales ELSE 0 END) AS total_sales_after
FROM clean_weekly_sales
WHERE calendar_year = '2020'
GROUP BY region, year
) AS region
),
platform AS (
SELECT year, platform,
       total_sales_before, 
       total_sales_after,
       (total_sales_before - total_sales_after) AS sales_difference,
       ROUND(((total_sales_after - total_sales_before) / total_sales_before) * 100, 1) AS growth_rate
FROM (
      SELECT 
      calendar_year AS year, 
      platform,
      SUM(CASE WHEN week_date BETWEEN DATE_SUB('2020-06-15', INTERVAL 12 WEEK) AND '2020-06-15' THEN sales ELSE 0 END) AS total_sales_before,
      SUM(CASE WHEN week_date BETWEEN '2020-06-15' AND DATE_ADD('2020-06-15', INTERVAL 12 WEEK) THEN sales ELSE 0 END) AS total_sales_after
FROM clean_weekly_sales
WHERE calendar_year = '2020'
GROUP BY platform, year
) AS platform
),
age_band AS (
SELECT year, age_band,
       total_sales_before, 
       total_sales_after,
       (total_sales_before - total_sales_after) AS sales_difference,
       ROUND(((total_sales_after - total_sales_before) / total_sales_before) * 100, 1) AS growth_rate
FROM (
      SELECT 
      calendar_year AS year, 
      age_band,
      SUM(CASE WHEN week_date BETWEEN DATE_SUB('2020-06-15', INTERVAL 12 WEEK) AND '2020-06-15' THEN sales ELSE 0 END) AS total_sales_before,
      SUM(CASE WHEN week_date BETWEEN '2020-06-15' AND DATE_ADD('2020-06-15', INTERVAL 12 WEEK) THEN sales ELSE 0 END) AS total_sales_after
FROM clean_weekly_sales
WHERE calendar_year = '2020'
GROUP BY age_band, year
) AS age_band
),
demographic AS (
       SELECT year, 
       demographic,
       total_sales_before, 
       total_sales_after,
       (total_sales_before - total_sales_after) AS sales_difference,
       ROUND(((total_sales_after - total_sales_before) / total_sales_before) * 100, 1) AS growth_rate
FROM (
SELECT 
      calendar_year AS year, 
      demographic,
      SUM(CASE WHEN week_date BETWEEN DATE_SUB('2020-06-15', INTERVAL 12 WEEK) AND '2020-06-15' THEN sales ELSE 0 END) AS total_sales_before,
      SUM(CASE WHEN week_date BETWEEN '2020-06-15' AND DATE_ADD('2020-06-15', INTERVAL 12 WEEK) THEN sales ELSE 0 END) AS total_sales_after
FROM clean_weekly_sales
WHERE calendar_year = '2020'
GROUP BY demographic, year
) AS demographic
),
customer_type AS (
SELECT year, customer_type,
       total_sales_before, 
       total_sales_after,
       (total_sales_before - total_sales_after) AS sales_difference,
       ROUND(((total_sales_after - total_sales_before) / total_sales_before) * 100, 1) AS growth_rate
FROM (
SELECT 
      calendar_year AS year, 
      customer_type,
      SUM(CASE WHEN week_date BETWEEN DATE_SUB('2020-06-15', INTERVAL 12 WEEK) AND '2020-06-15' THEN sales ELSE 0 END) AS total_sales_before,
      SUM(CASE WHEN week_date BETWEEN '2020-06-15' AND DATE_ADD('2020-06-15', INTERVAL 12 WEEK) THEN sales ELSE 0 END) AS total_sales_after
FROM clean_weekly_sales
WHERE calendar_year = '2020'
GROUP BY customer_type, year
) AS customer_type
)
SELECT year,
	ROUND(AVG(r.growth_rate),1) AS region_impact,
       ROUND(AVG(p.growth_rate),1) AS platform_impact,
       ROUND(AVG(ab.growth_rate),1) AS age_band_impact,
       ROUND(AVG(d.growth_rate),1) AS demographic_impact,
       ROUND(AVG(ct.growth_rate),1) AS customer_type_impact
FROM region AS r
JOIN platform AS p USING (year)
JOIN age_band AS ab USING (year)
JOIN demographic AS d USING (year)
JOIN customer_type AS ct USING (year)
GROUP BY year;
```

![SD_Q1](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/9c140217-9480-43c2-bf37-a54f2ceea92d)
