# Retail-Data-Analysis
## 1.1 EXECUTIVE SUMMARY 
This project analyzed retail transaction data containing over 6,000 records across products, sales channels, discounts, and geographic regions. Due to data quality challenges such as missing values, inconsistent formatting, and invalid pricing, the dataset was first cleaned and standardized using SQL to ensure analytical reliability.
The analysis reveals that revenue is highly concentrated in the Apparel and Footwear categories, which together contribute 66% of total revenue. Sales performance is balanced across web and in-store channels, while discounts increase order volume but do not materially improve revenue or margins. Performance is strongest in 2024 and 2025, with earlier and later years reflecting partial data coverage rather than true trends. Geographically, revenue is dominated by a small number of core markets, led by Kenya.
Overall, the findings highlight opportunities to optimize product focus, refine discount strategies, strengthen data capture processes, and prioritize high-performing markets. The cleaned dataset and insights provide a reliable foundation for ongoing reporting and decision-making.

## 1.2 Dataset Overview

The raw dataset for Trendify Retail Co consisted of **6,196 records (rows)** and **15 columns**.

The data contained multiple quality issues such as:

- Duplicate Order IDs
- Missing and inconsistent values
- Invalid identifiers
- Mixed date formats
- Negative numeric values
- Inconsistent text formatting
  
All cleaning activities were carried out on **14/12/2025** with the objective of producing a reliable, analysis-ready dataset.

<img width="760" height="297" alt="Raw data" src="https://github.com/user-attachments/assets/5f1a8e91-3bd3-4e8e-b1a2-7d81c56a8705" />

## 1.3	Methodology
Data cleaning and preparation for the Trendify Retail Co dataset were performed using SQL (MySQL). The objective was to transform the raw transactional data into a consistent, accurate, and analysis-ready dataset suitable for exploratory analysis and business reporting.
The cleaning process followed a structured and rule-based approach:
- **Data integrity first:** Records with invalid or missing primary identifiers (such as OrderID) were excluded to prevent duplicate counting and ensure transactional accuracy.
- **Selective imputation:** Inconsistent formats (dates, text fields, country names) were standardized, while irreparable records were removed rather than imputed.
- **Single-record enforcement:** Duplicate OrderIDs were resolved using SQL window functions to retain only one representative record per transaction.
- **Data type enforcement:** Key columns were converted to appropriate data types and constraints were applied to improve reliability.
- **Text normalization:** Customer names, product names, and categorical fields were cleaned to remove inconsistencies, extra spaces, and placeholder values.
- **Numerical corrections:** Negative values in quantity and unit price fields were corrected to maintain logical consistency.
  
Each cleaning step was validated using diagnostic SQL queries, including distinct counts, null checks, and data type verification. The final dataset reflects a balance between data quality and data retention, ensuring trustworthy insights while minimizing unnecessary data loss.
## 1.4	Data Cleaning Steps
### 1.4.1	Identifying and Removing Duplicate Orders
Although the dataset had **6,196 rows**, only **4,855 distinct OrderIDs** were present, indicating 8*duplicate records.*
To eliminate duplicate `OrderID` values, a window function was used to retain only the first occurrence of each order.

```sql
CREATE TABLE retail_data_clean AS
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY OrderID ORDER BY OrderID) AS rn
    FROM retail_data_raw
) t
WHERE rn = 1;

```
Validation was performed using:
```SQL
SELECT DISTINCT OrderID
FROM retail_data_clean;
```
<img width="624" height="237" alt="Distinct Orders" src="https://github.com/user-attachments/assets/8da27a5d-92a6-4d94-bd76-b7510052e5ce" />

### 1.4.2	Handling OrderID Inconsistencies
Several invalid OrderID values were identified, including blanks and placeholders such as ???, 99999, and ORDX.

```sql
SELECT DISTINCT OrderID
FROM retail_data_raw
WHERE
    OrderID IS NULL
    OR TRIM(OrderID) = ''
    OR OrderID IN ('0', '99999', 'ORDX', 'OrderId', '???', '');
```
These invalid records were removed:
```sql
DELETE FROM retail_data_clean
WHERE OrderID IS NULL
OR OrderID IN (' ','0','???','99999','OrderID','ORDX');
```
After cleanup, the OrderID column was enforced as a unique identifier.
```sql
ALTER TABLE retail_data_clean
MODIFY OrderID BIGINT NOT NULL,
ADD UNIQUE (OrderID);
```
<img width="178" height="187" alt="Inconsistent OrderID" src="https://github.com/user-attachments/assets/d09fc09a-46d9-4e36-b19e-34de2199438a" />

### 1.4.3	Cleaning and Standardizing Order Dates
Blank and missing OrderDate values were identified:
```sql
SELECT OrderID, OrderDate
FROM retail_data_raw
WHERE
    OrderDate IS NULL
    OR TRIM(OrderDate) = '';
```
A total of 213 records had missing dates and were deleted.

<img width="271" height="343" alt="Missing Order Dates" src="https://github.com/user-attachments/assets/6b1d3cae-9ac5-4e37-a184-5b0acb6271e9" />

Date formats varied significantly (DD/MM/YYYY, MM/DD/YYYY, DD MON YY, etc.). These were standardized to ISO format (YYYY MM DD):
```sql
UPDATE retail_data_clean
SET OrderDate =
    CASE
        WHEN OrderDate REGEXP '^[0-9]{2}/[0-9]{2}/[0-9]{4}$'
            THEN DATE_FORMAT (STR_TO_DATE(OrderDate, '%d/%m/%Y'), '%Y-%m-%d')
        WHEN OrderDate REGEXP '^[0-9]{2}/[0-9]{2}/[0-9]{4}$'
            AND STR_TO_DATE (OrderDate, '%m/%d/%Y') IS NOT NULL
            THEN DATE_FORMAT (STR_TO_DATE(OrderDate, '%m/%d/%Y'), '%Y-%m-%d')
        WHEN OrderDate REGEXP '^[0-9]{2}-[A-Za-z]{3}-[0-9]{2}$'
            THEN DATE_FORMAT (STR_TO_DATE(OrderDate, '%d-%b-%y'), '%Y-%m-%d')
        WHEN OrderDate REGEXP '^[0-9]{2}-[A-Za-z]{3}-[0-9]{4}$'
            THEN DATE_FORMAT (STR_TO_DATE(OrderDate, '%d-%b-%Y'), '%Y-%m-%d')
        ELSE OrderDate
    END;
```
Finally, the column was converted to DATE datatype:
```sql
ALTER TABLE retail_data_clean
MODIFY OrderDate DATE NOT NULL;
```
### 1.4.4	Customer Name Standardization
Customer names contained inconsistent characters and spacing.
Correction of embedded special characters:
```sql
UPDATE retail_data_clean
SET CustomerName = REGEXP_REPLACE(CustomerName,'''([A-Za-z])''','\\1')
WHERE CustomerName REGEXP '''[A-Za-z]''';
```
Null and blank names were identified and standardized:
```sql
UPDATE retail_data_clean
SET CustomerName = NULL
WHERE TRIM(CustomerName) = '';
```
Extra spaces were removed:
```sql
UPDATE retail_data_clean
SET CustomerName = TRIM(REGEXP_REPLACE(CustomerName, '\\s+', ' '))
WHERE CustomerName IS NOT NULL;
```
Name casing was standardized:
```sql
UPDATE retail_data_clean
SET CustomerName =
    CONCAT(
        UPPER(LEFT(CustomerName,1)),
        LOWER(SUBSTRING(CustomerName,2))
    )
WHERE CustomerName IS NOT NULL;;
```
### 1.4.5	Country Standardization
Distinct country values were reviewed:
```sql
SELECT DISTINCT Country
FROM retail_data_clean;
```
<img width="232" height="316" alt="Counties Inconsistencies" src="https://github.com/user-attachments/assets/e220d6f1-3847-411c-a015-e4bcf4c320e2" />

Country abbreviations were standardized:
```sql
UPDATE retail_data_clean
SET Country = 'Rwanda'
WHERE Country IN ('RW','Rw');
```
<img width="195" height="214" alt="Clean Countries" src="https://github.com/user-attachments/assets/c49671b2-b097-46e5-baab-becf56edb98c" />

### 1.4.6	Column Pruning
The email column was removed as it was irrelevant for analysis:
```sql
ALTER TABLE retail_data_clean
DROP COLUMN email;
```
### 1.4.7	Product Data Cleaning
Product consistency was validated:
```sql
SELECT DISTINCT ProductID, ProductName
FROM retail_data_clean
ORDER BY ProductID;
```
<img width="222" height="258" alt="ProductID and Product Name" src="https://github.com/user-attachments/assets/66a1e5d9-5575-4e52-bbe9-b960852d3a49" />

Invalid and placeholder product names were removed:
```sql
DELETE FROM retail_data_clean
WHERE ProductName IN ('misc item', 'Unknown', 'Unnamed product', '()');
```
### 1.4.8	Category & Numerical Corrections
Negative numeric values were corrected:
```sql
UPDATE retail_data_clean
SET UnitPrice = ABS(UnitPrice)
WHERE UnitPrice < 0;
```
<img width="441" height="337" alt="Category Distribution before cleaning" src="https://github.com/user-attachments/assets/34e8aed5-2d4e-4be9-8e4f-b235ae31c1a8" />

```sql
UPDATE retail_data_clean
SET Quantity = ABS(Quantity)
WHERE Quantity < 0;
```
<img width="497" height="285" alt="Clean Category Value" src="https://github.com/user-attachments/assets/2ef909fb-2761-464b-bf33-b70096a5f2a7" />

### 1.4.9	Null Quantity and Unit Prices
A total of 80 records were identified with zero or null quantity values.
These values were imputed using the global average of valid quantities, calculated via a scalar subquery, ensuring minimal distortion to overall sales volume while preserving record completeness.
```sql
UPDATE retail_data_clean
SET Quantity = (
    SELECT AVG(Quantity)
    FROM retail_data_clean
    WHERE Quantity > 0
)
WHERE Quantity = 0
   OR Quantity IS NULL;
```

Additionally, 80 records were found with unit prices recorded as 0.0.
Since valid prices for these products existed elsewhere in the dataset, unit prices were corrected using a correlated subquery that matched records by product name and product category. This ensured product-level price consistency while avoiding cross-category contamination.
```sql
UPDATE retail_data_clean t
 SET UnitPrice = (
    SELECT MAX(s.UnitPrice)
    FROM retail_data_clean s
    WHERE s.ProductName = t.ProductName
      AND s.ProductCategory = t.ProductCategory
      AND s.UnitPrice > 0
)
WHERE (t.UnitPrice = 0 OR t.UnitPrice IS NULL)
  AND EXISTS (
      SELECT 1
      FROM retail_data_clean s
      WHERE s.ProductName = t.ProductName
        AND s.ProductCategory = t.ProductCategory
        AND s.UnitPrice > 0
  );
```
Post-imputation validation confirmed no remaining zero or null quantities or unit prices.
### 1.4.10	 Final Dataset Summary
After completing all cleaning steps:
- Initial records: 6,196 rows and 15 columns
- Final clean records: 4,298 rows and 13 columns
  
  <img width="624" height="299" alt="Clean TrendyCo data" src="https://github.com/user-attachments/assets/eecdfd29-1384-44de-937e-a5ee8589a2cd" />

This dataset is now consistent, standardized, and ready for reliable exploratory analysis and business reporting.
## 1.5	Data Analysis
The objective of the analysis phase is to identify sales trends, revenue drivers, product and category performance, channel effectiveness, and discount impact to support business decision-making.
The analysis aims to answer the following questions:
- Which products and product categories are the primary drivers of revenue, and which are underperforming?
- Which sales channels contribute most to overall revenue and performance?
- Are discounts increasing sales volume, or are they eroding profit margins?
- How do sales trends vary over time and across different geographic regions?
- Which areas of operations require improvement to support accurate and reliable reporting?

### 1.5.1	Overall Business Snapshot
I start analysis by establishing baseline business metrics to validate the dataset and anchor all downstream insights.
```sql
SELECT 
COUNT(distinct OrderID) AS TotalOrders,
COUNT(distinct CustomerName) AS ToatalCustomers,
 COUNT(distinct Country) AS TotalCountries,
 COUNT(DISTINCT ProductName) AS TotalProducts,
 COUNT(DISTINCT Category) AS TotalCategories,
 COUNT(DISTINCT SalesRep) AS TotalSalesrep,
 COUNT(DISTINCT PaymentMethod) AS TotalPaymentmethod,
 COUNT(distinct OrderSource) AS TotalOrderSource,
 SUM(Quantity) AS TotalQuantities,
 SUM(UnitPrice*Quantity) AS Revenue
FROM retail_data_clean
```

<img width="701" height="62" alt="Overall Business Snapshot" src="https://github.com/user-attachments/assets/5dda76e9-f18a-482a-9935-11e092396e39" />

This query was executed to generate a high-level overview of the business after data cleaning.
The resulting metrics provide baseline indicators of operational scale, customer reach, product diversity, sales channels, and revenue performance. These measures serve as reference points for all subsequent analyses and ensure consistency across detailed product, channel, time-based, and geographic evaluations.
 
### 1.5.2	Objective Driven Exploratory Analysis 
Following the establishment of baseline business metrics, the analysis proceeded in alignment with the core business questions. Each analytical section was designed to directly address a specific performance concern related to products, channels, pricing strategies, time-based trends, and geographic performance.


#### 1.5.2.1	Product and Category Analysis
**Business problem:** Which products and categories drive revenue, and which are underperforming?
To find the Categories performance, this query was run.
```sql
SELECT Category,
SUM(Quantity*UnitPrice) AS Revenue,
SUM(Quantity) AS TotalQiantity
FROM retail_data_clean
GROUP BY Category
ORDER BY Revenue
```
<img width="434" height="193" alt="Performance by Category" src="https://github.com/user-attachments/assets/81f3eb23-08ab-4a63-9e58-d4dfd7c0cf3a" />

The analysis shows that Apparel is the top-performing category, contributing 35% of total revenue, followed by Footwear (31%), Accessories (20%), and Clothes (14%). 
Revenue is therefore concentrated in Apparel and Footwear, which together account for 66% of total sales value.
This indicates that the business is heavily reliant on these two categories for revenue generation, while the Clothes category may require further review to understand its lower contribution.

#### 1.5.2.2	Top product by revenue
**Business Problem:** Which product drives revenue?
```sql
SELECT Category, ProductID, ProductName, SUM(Quantity*UnitPrice) AS ProductRevenue
FROM retail_data_clean
GROUP BY Category,ProductID,ProductName
ORDER BY ProductRevenue desc
LIMIT 10;
```
<img width="525" height="260" alt="Performing Products" src="https://github.com/user-attachments/assets/9df90771-70e8-4f93-9370-0eda85daaad7" />

To identify the key revenue drivers at the product level, total revenue was aggregated per product and category. This analysis highlights the products that contribute the most to overall sales value.
The results show that revenue is driven by a relatively small number of products across multiple categories. 
Denim Jacket is the highest-revenue product, followed by Bracelet and Jeans. Apparel and Footwear feature prominently among the top products, reinforcing their importance as core revenue-generating categories.
```sql
SELECT Category, ProductID, ProductName, SUM(Quantity*UnitPrice) AS ProductRevenue
FROM retail_data_clean
GROUP BY Category,ProductID,ProductName
ORDER BY ProductRevenue asc
LIMIT 10;
```
<img width="456" height="273" alt="Bottom 10 Performing Products" src="https://github.com/user-attachments/assets/9c647ccc-7fa5-4dc8-a9fa-413a40f09ac1" />

To identify underperforming products, total revenue was aggregated at the product level and ranked in ascending order. 
The analysis shows that the lowest-performing products generate substantially less revenue compared to top performers. 
Products such as Necklace, Boots, and Beanie contribute minimal revenue, indicating limited demand, low pricing, or reduced sales frequency. 
These products are distributed across multiple categories, suggesting that underperformance is driven by individual product performance rather than category-wide weakness.
#### 1.5.2.3	Sales Channel & Order Source Analysis
Business Problem: Which sales channels contribute most to performance?
To evaluate the contribution of different sales channels, order volume and total revenue were aggregated by order source.
```sql
SELECT 
    OrderSource,
    COUNT(DISTINCT OrderID) AS Orders,
    SUM(UnitPrice * Quantity) AS Revenue
FROM retail_data_clean
GROUP BY OrderSource
ORDER BY Revenue DESC;
```
<img width="313" height="176" alt="Channel Performance" src="https://github.com/user-attachments/assets/e07e214a-b7b2-4b5d-b9c7-10e0eacf6cf9" />

The analysis of sales performance by order source reveals a balanced distribution between online and physical channels, with web and in-store purchases leading overall performance.
- Store is the top-performing channel by order volume with 1,214 orders, generating 112,132.11 in revenue.
- Web closely follows, contributing 1,191 orders and 112,574.89 in revenue — the highest revenue-generating channel.
- Mobile and Affiliate channels show moderate performance, indicating secondary but meaningful contribution to total sales.
- Orders with no recorded source (“none”) account for a noticeable portion of revenue, highlighting a data capture gap that may affect channel attribution accuracy.
#### 1.5.2.4	Discount Impact Analysis
Business Problem: Are discounts driving higher sales volume, or are they reducing overall revenue performance?
```sql
SELECT  
CASE
WHEN DiscountCode IS NULL
OR TRIM(DiscountCode)=''
OR LOWER(TRIM(DiscountCode))='null'
THEN 'NO Discount'
ELSE 'Discount Applied'
END AS DiscountFlag,
COUNT(DISTINCT OrderID) AS Orders,
COUNT(Quantity) AS TotalQuantity,
SUM(UnitPrice*Quantity) AS Revenue
FROM retail_data_clean
GROUP BY DiscountFlag 
```
<img width="419" height="95" alt="Discount Impact on Revenue" src="https://github.com/user-attachments/assets/f8efdd42-2994-4154-a742-b46405c31ac1" />

The analysis shows that discounted and non-discounted orders contribute almost equally to total revenue.
While slightly more orders were placed with discounts applied (2,193 vs 2,105), the total revenue generated by non-discounted sales is marginally higher.
This indicates that discounts are effective at increasing order volume, but do not significantly increase overall revenue. The near-equal revenue contribution suggests that discounted sales may be offset by lower unit prices, limiting their impact on revenue growth.
That said, an analysis on which discount code is performing better was run. 
```sql
SELECT 
    DiscountCode,
    COUNT(DISTINCT OrderID) AS Orders,
    SUM(UnitPrice * Quantity) AS Revenue
FROM retail_data_clean
WHERE DiscountCode IS NOT NULL
  AND LOWER(TRIM(DiscountCode)) <> 'null'
GROUP BY DiscountCode
ORDER BY Revenue DESC;
```
<img width="308" height="126" alt="Discount Codes Performance" src="https://github.com/user-attachments/assets/03174b5d-1bd2-4f21-a2a7-8191cf1e792b" />

The analysis shows SUMMER discount code generated the highest revenue, its margin contribution is only slightly higher than the other discount campaigns, with margins of approximately 71k for SUMMER, 68k for NEW10, and 62k for DISC-20. 
This indicates that while SUMMER performs marginally better, the difference in profitability across discount codes is not substantial. 
Overall, discounted sales deliver comparable margin outcomes, suggesting that discounts influence purchasing behavior without significantly differentiating profit performance. 
As a result, discount effectiveness appears to be driven more by transaction value mix than by a clear margin advantage
#### 1.5.2.5	Time-Based Sales Trends
**Business Problem:** How does sales performance vary over time, and are there identifiable trends or seasonality patterns?
This section leverages the cleaned and standardized OrderDate column to analyze temporal performance. Time-based analysis helps the business understand growth patterns, demand cycles, and timing of revenue concentration, which are critical for forecasting, inventory planning, and promotional timing.
```sql
SELECT 
    YEAR(OrderDate) AS Year,
    COUNT(DISTINCT OrderID) AS Orders,
    SUM(UnitPrice * Quantity) AS Revenue
FROM retail_data_clean
GROUP BY YEAR(OrderDate)
ORDER BY Year;
```

<img width="269" height="125" alt="Over time Performance" src="https://github.com/user-attachments/assets/9c767224-311a-47b0-aeca-9ed7efa0da59" />

The yearly sales analysis indicates that 2024 is the dominant performance year, contributing 51.1% of total revenue and 52% of total orders, making it the primary driver of overall business activity. This is followed by 2025, which accounts for 44.6% of total revenue and 43.6% of total orders, reflecting sustained but slightly lower performance compared to 2024.
In contrast, 2023 and 2026 contribute marginally to overall performance, with 2023 generating 2.7% of revenue and 2.55% of orders, while 2026 accounts for 1.6% of revenue and 1.7% of orders. The limited contribution of these years suggests partial-year coverage or lower transaction volume, indicating that performance insights are primarily driven by activity in 2024 and 2025.
As a result, subsequent trends and seasonal analyses are most meaningful when focused on 2024 and 2025, where sufficient transaction volume exists to support reliable conclusions.
##### 1.5.2.5.1	Monthly Trends for 2024 and 2025
Business Problem: What is the trend of performance through out 2024 and 2025 ?
```sql
SELECT 
    YEAR(OrderDate) AS Year,
    MONTHNAME(OrderDate) AS Month,
    COUNT(DISTINCT OrderID) AS Orders,
    SUM(UnitPrice * Quantity) AS Revenue
FROM retail_data_clean
WHERE YEAR(OrderDate) IN (2024, 2025)
GROUP BY YEAR(OrderDate), MONTH(OrderDate), MONTHNAME(OrderDate)
ORDER BY Year, MONTH(OrderDate);

```

<img width="399" height="520" alt="Performance Trend over time" src="https://github.com/user-attachments/assets/020f3cfb-dee9-4293-83fa-c070541cc604" />

The monthly sales trend analysis for 2024 and 2025 shows relatively stable performance across most months, with moderate fluctuations in both order volume and revenue. In 2024, revenue remains consistently distributed throughout the year, with small peaks observed in March, August, and October, indicating steady consumer demand rather than strong seasonality.
In 2025, performance follows a similar pattern during the first eight months, with revenue peaking notably in August, suggesting a strong mid-year sales period. However, a sharp decline is observed from September through December 2025, where both order counts and revenue drop significantly. This decline is likely driven by incomplete or partial data capture for the latter months of 2025 rather than an actual collapse in business performance.
Overall, the trend indicates that the business experiences its strongest and most consistent sales activity during the first three quarters of the year, while year-end results should be interpreted cautiously due to potential data availability limitations.

##### 1.5.2.5.2	Month-over-Month (MoM) Revenue Growth Analysis
**Business Problem:** How does revenue change from month to month, and what does this indicate about performance stability?
Month-over-month (MoM) revenue growth was analyzed to assess short-term performance momentum across the available period (2023–2026). Monthly revenue was compared to the previous month to identify growth patterns and volatility.
The results show relatively stable MoM changes during periods with full data coverage, particularly throughout 2024 and the first three quarters of 2025, indicating consistent sales performance. Larger positive and negative MoM fluctuations occur mainly around year transitions and in late 2025–2026, which aligns with reduced or partial data availability rather than a true decline in business activity.
Overall, the MoM analysis confirms that revenue volatility outside core periods is primarily driven by data completeness, while underlying sales performance remains stable during fully captured months.
```sql
WITH monthly_revenue AS (
    SELECT
        YEAR(OrderDate) AS Year,
        MONTH(OrderDate) AS Month,
        DATE_FORMAT(OrderDate, '%Y-%m') AS YearMonth,
        SUM(UnitPrice * Quantity) AS Revenue
    FROM retail_data_clean
    WHERE YEAR(OrderDate) BETWEEN 2023 AND 2026
    GROUP BY YEAR(OrderDate), MONTH(OrderDate), DATE_FORMAT(OrderDate, '%Y-%m')
)
SELECT
    Year,
    Month,
    YearMonth,
    Revenue,
    LAG(Revenue) OVER (ORDER BY Year, Month) AS PreviousMonthRevenue,
    ROUND(
        (Revenue - LAG(Revenue) OVER (ORDER BY Year, Month)) /
        NULLIF(LAG(Revenue) OVER (ORDER BY Year, Month), 0) * 100,
        2
    ) AS MoM_Growth_Percent
FROM monthly_revenue
ORDER BY Year, Month;
```
<img width="316" height="177" alt="Regional Trends" src="https://github.com/user-attachments/assets/f2e41784-498e-4255-9280-e8dae3e8dbc7" />

#### 1.5.2.6	Geographic / Regional Performance Analysis
Business Problem: How does sales performance vary by geography, and which regions drive or lag revenue?
```sql
SELECT
    Country,
    COUNT(DISTINCT OrderID) AS Orders,
    SUM(UnitPrice * Quantity) AS Revenue
FROM retail_data_clean
GROUP BY Country
ORDER BY Revenue DESC;
```

The geographic analysis shows that revenue is concentrated in a small number of core markets. Kenya is the strongest performing country, leading both in order volume (1,290 orders) and revenue generation (119,487.01), indicating a well-established and active customer base. 
Tanzania, Uganda, and Rwanda form a secondary performance tier with comparable revenue contributions, suggesting consistent but slightly lower demand levels across these markets.
South Sudan and Burundi contribute the lowest share of revenue and orders, reflecting limited market activity. 
This geographic distribution highlights Kenya as the primary revenue driver, while secondary markets present opportunities for targeted growth initiatives or deeper market penetration strategies.
## 1.6	Key findings
**1.	Revenue is highly concentrated in two core categories.**
Apparel and Footwear together account for 66% of total revenue, making them the primary drivers of business performance. Other categories, particularly Clothes, contribute significantly less and may require strategic review.

**2.	A small set of products drives a disproportionate share of revenue.**
Products such as Denim Jacket, Bracelet, and Jeans generate the highest revenue, while several products across multiple categories contribute minimal sales. Underperformance is product-specific rather than category-wide.

**3.	Sales performance is balanced across online and physical channels.**
Web and in-store channels contribute nearly equal revenue and order volumes, indicating a well-distributed omnichannel presence. Mobile and affiliate channels provide secondary support, while missing order-source values highlight a data capture gap.

**4.	Discounts increase order volume but do not significantly improve revenue or margins.**
Discounted orders slightly exceed non-discounted orders in volume, but total revenue remains nearly equal. Individual discount codes show only marginal differences in margin contribution, suggesting limited profitability differentiation.

**5.	Business performance is driven primarily by activity in 2024 and 2025.**
These two years account for over 95% of total revenue and orders. Performance in 2023 and 2026 is minimal and likely reflects partial data coverage rather than true business trends.

**6.	Sales trends show stability rather than strong seasonality.**
Monthly performance across 2024 and most of 2025 remains consistent, with moderate fluctuations. Sharp declines in late 2025 align with incomplete data rather than an operational downturn.

**7.	Revenue is geographically concentrated in a few core markets.**
Kenya is the dominant market by both orders and revenue, followed by Tanzania, Uganda, and Rwanda. South Sudan and Burundi show limited activity, indicating either early-stage markets or low demand.

## 1.7	Business Recommendations
- **Prioritize and scale high-performing categories (Apparel & Footwear)**
Given that Apparel and Footwear contribute 66% of total revenue, the business should prioritize inventory planning, marketing spend, and product expansion within these categories. Ensuring consistent stock availability and introducing complementary products in these segments can maximize revenue growth while leveraging proven demand.
- **Optimize the product portfolio by addressing underperforming products**
Products identified among the bottom performers contribute minimal revenue despite occupying inventory and operational resources. These products should be reviewed for pricing strategy, promotion effectiveness, or potential discontinuation. Redirecting focus toward high-performing products can improve overall revenue efficiency.
- **Strengthen omnichannel strategy while improving order source tracking**
Web and in-store channels generate comparable revenue, indicating a strong omnichannel presence. The business should continue investing in both digital and physical channels while addressing the “none” order source category to improve attribution accuracy and enable more precise channel-level decision-making.
- **Refine discount strategies to focus on profitability, not just volume**
Discounts increase order volume but do not significantly improve revenue or margins. Rather than broad discounting, the business should implement targeted promotions focused on specific products, categories, or customer segments to improve profitability without eroding margins.
- **Use 2024–2025 trends as the primary basis for forecasting and planning**
Since over 95% of revenue and orders are generated in 2024 and 2025, forecasting, budgeting, and performance benchmarking should be anchored on these years. Data from 2023 and 2026 should be treated as partial and excluded from long-term trend assumptions.
- **Capitalize on stable demand by planning around consistent monthly performance**
The absence of strong seasonality suggests predictable demand throughout most of the year. This enables more stable inventory planning, workforce allocation, and marketing scheduling, reducing the need for aggressive seasonal adjustments.
- **Deepen market penetration in core regions while selectively growing secondary markets**
Kenya is the strongest market and should remain the primary focus for revenue growth initiatives. Secondary markets such as Tanzania, Uganda, and Rwanda present opportunities for targeted expansion through localized promotions or partnerships. Lower-performing regions should be evaluated for growth potential before further investment.
## 1.8	Dashboard
 


## 1.9	Conclusion
- The objective of this project was to transform raw transactional data from Trendify Retail Co into a reliable, analysis-ready dataset and to extract actionable business insights to support data-driven decision-making.
- Through a structured SQL-based data cleaning process, the dataset was reduced from 6,196 raw records to 4,298 high-quality records, ensuring consistency in identifiers, dates, product attributes, categories, and numerical values. This established a solid foundation for trustworthy analysis.
- The exploratory analysis revealed that revenue is highly concentrated in Apparel and Footwear, which together account for over two-thirds of total sales value. Product-level analysis showed that a small number of products drive a disproportionate share of revenue, while several low-performing products contribute minimal value. Channel analysis highlighted a balanced omnichannel model, with web and in-store sales leading performance, while also exposing gaps in order source data capture.
- Discount analysis indicated that while promotions increase order volume, they do not significantly improve revenue or margins, suggesting that discounting strategies should be refined and targeted. Time-based analysis confirmed that business performance is driven primarily by 2024 and 2025, with relatively stable monthly trends and no strong seasonality. Geographic analysis identified Kenya as the core revenue market, supported by several secondary regional markets.
- Overall, the findings provide clear guidance for optimizing product strategy, refining discount policies, strengthening channel attribution, improving forecasting accuracy, and focusing growth efforts on high-impact categories and regions.
- This analysis positions Trendify Retail Co to make informed strategic decisions supported by clean, reliable data and objective-driven insights.


 <img width="624" height="343" alt="Dashboard" src="https://github.com/user-attachments/assets/a4cc9c99-d2d7-4522-8df5-f9053228454c" />







