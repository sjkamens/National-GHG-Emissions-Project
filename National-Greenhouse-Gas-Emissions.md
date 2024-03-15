# National Greenhouse Gas Emissions

This project was my first independent project utilising SQL, and was carried out upon completion of the Google Data Analytics Professional Certificate. Its purpose is two-fold:
  1. To share my capabilities in writing SQL queries for data processing, cleaning, transformation, and validation.
  2. To develop a database for which I will use to analyse national greenhouse gas emissions trends, with a specific focus on assessing the current status of green growth, the idea that economic growth can be decouple from greenhouse gas emissions.

Dataset ID: national-ghgs.national_ghg_timeseries

I developed this database through the aggregation of data from several sources. See citations below.

Ritchie, Hannah, Max Roser, Edouard Mathieu, Bobbie Macdonald, and Pablo Rosado. 2023. “CO₂ and Greenhouse Gas Emissions.” Our World in Data. https://github.com/owid/co2-data?tab=readme-ov-file.

The World Bank. 2024a. “The World by Income and Region.” The World Bank. https://datatopics.worldbank.org/world-development-indicators/the-world-by-income-and-region.html.

The World Bank. 2024b. “GDP (Constant 2015 US$).” The World Bank. https://data.worldbank.org/indicator/NY.GDP.MKTP.KD.

The World Bank. 2024c. “GDP (Constant LCU).” The World Bank. https://data.worldbank.org/indicator/NY.GDP.MKTP.KN.

The World Bank. 2024d. “GDP (Current US$).” The World Bank. https://data.worldbank.org/indicator/NY.GDP.MKTP.CD.

The World Bank. 2024e. “GDP, PPP (Constant 2017 International $).” The World Bank. https://data.worldbank.org/indicator/NY.GDP.MKTP.PP.KD.

------------------------------
## Query 1
```
-- Original table import of country classification data did not keep column names as intended.
-- This query creates a new table with new, descriptive column names.
CREATE TABLE national-ghgs.national_ghg_timeseries.country_classifications_2 AS
SELECT
  string_field_0 AS country_name,
  string_field_1 AS country_short,
  string_field_2 AS region,
  string_field_3 AS econ_class,
  string_field_4 AS lending_class
FROM
  national-ghgs.national_ghg_timeseries.country_classifications;

--This query is to verify that the new table appears as intended.
SELECT *
FROM national-ghgs.national_ghg_timeseries.country_classifications_2 LIMIT 10;
```
---------------------------------------
## Query 2
```
-- This query uses a join to add the country classification details to the national ghg time series data. It creates a new table for the results to maintain the original tables.

CREATE TABLE national-ghgs.national_ghg_timeseries.ghgs_time_series_2 AS
SELECT *
FROM
  national-ghgs.national_ghg_timeseries.ghgs_time_series_base
LEFT JOIN
  national-ghgs.national_ghg_timeseries.country_classifications_2 ON ghgs_time_series_base.country = country_classifications_2.country_name;
```
--------------------------------------------
## Query 3
```
-- As I am using the Sandbox version of BigQuery, I cannot use the DELETE FROM query
-- So, I must create a new table to delete rows.
CREATE TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_3` AS
SELECT *
FROM `national-ghgs.national_ghg_timeseries.ghgs_time_series_2` 
WHERE year >= 2000;
```
--------------------------------------------
## Query 4
```
-- This query identifies the non-country data and the country data for which the 
-- previous JOIN did not work correctly for.It will then be used to clean the dataset
-- to ensure it is solely a time series with countries as the main unit of observation
SELECT DISTINCT country
FROM `national-ghgs.national_ghg_timeseries.ghgs_time_series_3`
WHERE econ_class IS NULL;
```
---------------------------------------------
## Query 5
```--Query 5
-- The previous query returned a list of observations that the join did not work for.
-- Now, I start by removing the rows for the observations that are not countries.
-- The remaining observations will need to be manually made consistent across the ghg
-- and classifications tables. This uses CREATE TABLE as a workaround due to my use of
-- BigQuery Sandbox
CREATE TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_4` AS
SELECT *
FROM `national-ghgs.national_ghg_timeseries.ghgs_time_series_3`
WHERE country NOT IN (
  SELECT country
  FROM national-ghgs.national_ghg_timeseries.query_4_results
  WHERE is_a_country_ = FALSE
);
```
-----------------------------------------------------
## Query 6
```--Query 6
--After the previous queries, there were a set of country names that potentially needed to be updated. 
--Ultimately, I did a quick manual check across the output from Query 4 and identified which countries need to be updated and which will need to be removed.
--In this query, I update the country names, which will then be used to finish the LEFT JOIN performed in Query 2 to have a more complete dataset. 
--This method had to be done as I cannot use the UPDATE query with Sandbox.

CREATE TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_5` AS
SELECT
  CASE
    WHEN mapping.new_name IS NOT NULL THEN mapping.new_name
    ELSE original.country
  END AS country,
  original.* EXCEPT(country)
FROM 
  `national-ghgs.national_ghg_timeseries.ghgs_time_series_4` AS original
LEFT JOIN 
  `national-ghgs.national_ghg_timeseries.countries_to_update` AS mapping
ON
  original.country = mapping.country;
```
------------------------------------------------
## Query 7
```--Query 7
--Now, similar to the previous query, I amend the main table. 
--This time, I want to remove observations from countries that have not been classified economically/regionally by the World Bank. 
--This is done using a csv file created in Excel, similar to the one I made with the updated country names. 
--Again, as I have Sandbox I need to create a new table instead of using DELETE.

CREATE TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_6` AS
SELECT main.*
FROM `national-ghgs.national_ghg_timeseries.ghgs_time_series_5` AS main
LEFT JOIN national-ghgs.national_ghg_timeseries.countries_to_remove AS remove
ON main.country = remove.country
WHERE remove.country IS NULL;
```
----------------------------------------------------------------------
## Query 8
```--Query 8
--Originally, I wanted to just fill in the new values in the classification columns.
--However, I realize this may be a bit challenging. 
--For me, an easier way seemed to just be to delete the columns from the original LEFT JOIN from Query 2 and then perform a new LEFT JOIN.

CREATE TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_7` AS
SELECT * EXCEPT(country_name, country_short, region, econ_class, lending_class)
FROM `national-ghgs.national_ghg_timeseries.ghgs_time_series_6`;
```
----------------------------------------------------------------------------------
## Query 9
```--Query 9
--Finally, I complete the table using a LEFT JOIN.

CREATE TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_8` AS
SELECT *
FROM
  national-ghgs.national_ghg_timeseries.ghgs_time_series_7
LEFT JOIN 
  national-ghgs.national_ghg_timeseries.country_classifications_2 ON ghgs_time_series_7.country = country_classifications_2.country_name;
```
-------------------------------------------------------------------------------
## Query 10
```--Query 10
--These data points are the main data of interest for my analyses.
--Identifying where this data does not exist will help me to assess the completeness of the data.

SELECT * 
FROM `national-ghgs.national_ghg_timeseries.ghgs_time_series_8`
WHERE 
  gdp IS NULL OR 
  population IS NULL OR 
  total_ghg IS NULL;

--Overall, it seems recent GDP data is missing as well as GDP data for select economies is the main source of missing data that can impact my analyses.
--I could choose to search for recent GDP data from the same source as it was obtained from in this dataset or choose to neglect more recent years.
```
-------------------------------------------------
## Query 11
```--Query 11
-- Before addressing the GDP null value issue, I will remove the tables from my dataset that are no longer needed.

DROP TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_7`;
DROP TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_6`;
DROP TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_5`;
DROP TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_4`;
DROP TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_3`;
DROP TABLE `national-ghgs.national_ghg_timeseries.ghgs_time_series_2`;
```
-----------------------------------------------------
## Query 12
```--Query 12
--After investigating the null values for GDP, the data source used in the original GHG emissions time series table is the Maddison Project Database. This only has GDP data through 2018.
--Thus, an alternative source for all GDP data was needed. I chose to obtain four different measures of GDP, including two traditional GDP measures along with one in Purchasing Power Parity (PPP) and one in Local Currency Units (LCU). 
--To be able to join them with the table I will be working with, I need to convert these tables from wide to long.
--The following queries will do this, so that the country code can be used to join the GDP data with the main ghg time series table. 
--I also selected data from 2000 onwards as this is what is needed for the GHG emissions table. 
--Also tried to change the name of the country code column from "string_field_1" but was limited by the use of Sandbox.

CREATE TABLE national-ghgs.national_ghg_timeseries.gdp_2015_usd_long AS
SELECT string_field_1, "2000" AS year, double_field_44 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2001" AS year, double_field_45 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2002" AS year, double_field_46 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2003" AS year, double_field_47 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2004" AS year, double_field_48 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2005" AS year, double_field_49 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2006" AS year, double_field_50 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2007" AS year, double_field_51 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2008" AS year, double_field_52 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2009" AS year, double_field_53 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2010" AS year, double_field_54 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2011" AS year, double_field_55 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2012" AS year, double_field_56 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2013" AS year, double_field_57 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2014" AS year, double_field_58 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2015" AS year, double_field_59 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2016" AS year, double_field_60 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2017" AS year, double_field_61 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2018" AS year, double_field_62 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2019" AS year, double_field_63 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2020" AS year, double_field_64 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2021" AS year, double_field_65 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`
UNION ALL
SELECT string_field_1, "2022" AS year, double_field_66 AS value FROM `national-ghgs.national_ghg_timeseries.gdp_2015_usd`;
```
-------------------------------------------------------
## Queries 13, 14, and 15
```--Queries 13, 14, 15
--Here, I just repeated Query 12 above for each of the following: 
--GDP in current USD in the new table gdp_current_usd_long
--GDP in 2017 PPP in the new table gdp_2017_ppp_long
--GDP in constant LCU in the new table gdp_constant_lcu_long
```
--------------------------------------------------------
## Query 16
```--Query 16
--Now that I have formatted the four GDP tables in long format, I will left join them to the main table, using the country code and years as the common column and using this to include the corresponding GDP values in the updated table.
--Note: Initially ran without using the CAST function to change the year columns to integers and noticed an error. Realised in these tables, year was included as a string.

CREATE TABLE national-ghgs.national_ghg_timeseries.ghgs_time_series_9 AS
SELECT
  main.*,
  usd2015.value AS gdp_2015_usd,
  ppp.value AS gdp_2017_ppp,
  lcu.value AS gdp_constant_lcu,
  usd.value AS gdp_current_usd
FROM national-ghgs.national_ghg_timeseries.ghgs_time_series_8 AS main
LEFT JOIN national-ghgs.national_ghg_timeseries.gdp_2015_usd_long AS usd2015 ON
  main.iso_code = usd2015.string_field_1 AND main.year = CAST(usd2015.year AS INT64)
LEFT JOIN national-ghgs.national_ghg_timeseries.gdp_2017_ppp_long AS ppp ON
  main.iso_code = ppp.string_field_1 AND main.year = CAST(ppp.year AS INT64)
LEFT JOIN national-ghgs.national_ghg_timeseries.gdp_constant_lcu_long AS lcu ON
  main.iso_code = lcu.string_field_1 AND main.year = CAST(lcu.year AS INT64)
LEFT JOIN national-ghgs.national_ghg_timeseries.gdp_current_usd_long AS usd ON
  main.iso_code = usd.string_field_1 AND main.year = CAST(usd.year AS INT64);
```
------------------------------------------
## Query 17
```--Query 17
--Now, I want to ensure there were no major errors with the JOIN queries so I will check for null values. 
--While there was some unavailable data for a few observations, ideally there will be very few null values. 
--I create a table with the results to conduct further analyses, mainly to identify the affected countries.

CREATE TABLE `national-ghgs.national_ghg_timeseries.query_17_results` AS 
SELECT
  country,
  iso_code,
  year,
  gdp_2015_usd,
  gdp_2017_ppp,
  gdp_constant_lcu,
  gdp_current_usd
FROM `national-ghgs.national_ghg_timeseries.ghgs_time_series_9`
WHERE 
  gdp_2015_usd IS NULL OR
  gdp_2017_ppp IS NULL OR
  gdp_constant_lcu IS NULL OR
  gdp_current_usd IS NULL;
```
-------------------------------------------
## Query 18
```--Query 18
--To identify the countries with missing data, I execute the following query.

SELECT DISTINCT country
FROM `national-ghgs.national_ghg_timeseries.query_17_results`

--I end up with a list of 30 countries. I notice a trend across almost all of them - most are with:
--very small countries or even territories, 
--countries that have undergone substantial crises or territorial shifts during the time period of interest (2000-2022), 
--and/or have limited participation in international systems that have likely contributed to lesser availability of data.
```
----------------------------------------------------------------------------------------------------
