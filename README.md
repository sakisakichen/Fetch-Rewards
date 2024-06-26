# Fetch-Rewards

The assignment included reviewing unstructured JSON data and creating a structured relational data model, generating queries to answer specific business questions, identifying data quality issues, and communicating findings to stakeholders. 
by utilizing data for querying and data analysis, providing detailed documentation and notes to support my approaches. 

Tasks Completed:  
(1) Reviewed unstructured JSON data and developed a structured relational data model.    
(2) Wrote SQL queries to address predetermined business questions.    
(3) Identified and addressed data quality issues.  
(4) Constructed an email/slack message to explain data-related findings to stakeholders, showcasing effective communication skills in a non-technical context.  


<details>

<summary>First: Relational Data Model</summary>

#### Review Existing Unstructured Data and Diagram a New Structured Relational Data Model

[ER Diagram](https://github.com/sakisakichen/Fetch-Rewards/blob/main/Fetch%20Rewards_ER.pdf)

</details>

<details>

<summary>Second: SQL Queries </summary>

#### Write queries that directly answer predetermined questions from a business stakeholder

##### Q1 What are the top 5 brands by receipts scanned for most recent month?

```sql
'''
CLARIFICATION: How to define top brand? 
               the most frequently purchased? or the most total spent or item qty?
                I assume the most frequently purchased item as top 
Note:
1. there are several null values in the receipt item table
1. based on the purchased qty to filter top 5 brands, only 2 record with item barcode in the recent month (2021-03) , and no brandname information 
'''    
    -- Q1-1 In the set query , I get the data in most recent month:2021-03 , only get 2 item_barcode B076FJ92M4 & B07BRRLSVC
        with cte as (
       SELECT
        b.item_barcode, to_char(DATESCANNED,'yyyy-mm') as DATESCANNED,
        sum(b.item_quantitypurchased) as purchase_qty
        from receipt a JOIN receipt_item b 
        ON a.receiptid = b.receiptid 
        group by 1,2
        order by DATESCANNED DESC
        )
            SELECT  cte.item_barcode,brand.brandname,brand.brandcode,DATESCANNED,purchase_qty
            FROM CTE
            left JOIN brand
            ON cte.item_barcode = brand.barcode 
         where DATESCANNED like '2021-03%'
        order by datescanned DESC;
```
![image](https://github.com/sakisakichen/Fetch-Rewards/assets/72574733/709f6152-3519-4912-9fb2-971f8ce048db)

##### Q2 How does the ranking of the top 5 brands by receipts scanned for the recent month compare to the ranking for the previous month?
```sql
 -- Q2-1 from the previous query, the most recent month is 2021/03, which I need to find the data in 2021/02
  with cte as (
       SELECT b.item_barcode, b.item_description,to_char(DATESCANNED,'yyyy-mm') as DATESCANNED,  sum(b.item_quantitypurchased) as purchase_qty
        from receipt a JOIN receipt_item b 
        ON a.receiptid = b.receiptid 
        group by 1,2,3
        order by DATESCANNED DESC
        )
            SELECT   cte.item_barcode,cte.item_description, brand.brandname,brand.brandcode,DATESCANNED,purchase_qty
            FROM CTE
            left JOIN brand
            ON cte.item_barcode = brand.barcode 
         where DATESCANNED like '2021-02%' 
        order by datescanned DESC,purchase_qty DESC;
        
select item_barcode,item_description 
FROM receipt_item
 ;
 -- Q2-2 From the query result, I fount out these 2 item_barcode B076FJ92M4 & B07BRRLSVC also on the list, but no further information about brandname or brandcode,
 -- there also item code listed in 2021/02, purchased qty between 5 and 4, if steakholder needs to find the name of the brand, I suggest to find the data in 2021/1
```
![image](https://github.com/sakisakichen/Fetch-Rewards/assets/72574733/d7ec493d-6125-4a5c-962e-b7d207fa0908)
![image](https://github.com/sakisakichen/Fetch-Rewards/assets/72574733/28d707f1-3034-431e-9935-69e80ed6fae9)


##### Q3 When considering average spend from receipts with 'rewardsReceiptStatus’ of ‘Accepted’ or ‘Rejected’, which is greater?
```sql
--  Status with Finshed is greater in Average spent 
-- Note: status with FINISHED AVG spent is 80.854305019
SELECT rewardsReceiptStatus, AVG(totalspent) as avg_spent
from receipt
where rewardsreceiptstatus= 'FINISHED'
group by 1;

-- Note: status with REJECTED AVG spent is 23.326056338
SELECT rewardsReceiptStatus, AVG(totalspent) as avg_spent
from receipt
where rewardsreceiptstatus= 'REJECTED'
group by 1;

```
##### Q4 When considering total number of items purchased from receipts with 'rewardsReceiptStatus’ of ‘Accepted’ or ‘Rejected’, which is greater?

```sql
- Since question asks the number of item purchased, in my assumption, some items with null barcode will be counted 
--  REJECTED number is 164
SELECT  count(receipt_item_pk) as cnt
FROM receipt a JOIN receipt_item b ON a.receiptID = b.receiptID 
where rewardsreceiptstatus= 'REJECTED'
;
-- What is accepted status?  including finished, pending, submitted? only finished status is 5918 ,  if these status included, the number cnt is 5967
SELECT  count(receipt_item_pk) as cnt
FROM receipt a JOIN receipt_item b ON a.receiptID = b.receiptID 
where rewardsreceiptstatus= 'FINISHED' 
-- OR rewardsreceiptstatus= 'SUBMITTED' 
-- OR rewardsreceiptstatus= 'PENDING' 
;

```
##### Q5 Which brand has the most spend among users who were created within the past 6 months?

```sql
-- Stpe 1: find the most recent user account create date 
-- Note: The most recent user created account is in 2021/2, the past 6 month is 2020/9- 2021/2
SELECT oid, CREATEDDATE
from users_flatten
order by CREATEDDATE DESC
;
-- Step 2: Find the user account created in the period and what receipt item they purchase  
with cte as (
SELECT b.userid, b.RECEIPTID,totalspent,RECEIPT_ITEM_PK,item_barcode, item_finalprice
from users_flatten a 
JOIN receipt b 
JOIN receipt_item c 
ON b.receiptid = c.receiptID
ON a.oid = b.userid
WHERE LEFT(CREATEDDATE, 7) BETWEEN '2020-09' AND '2021-02'
AND item_finalprice is not null
order by item_finalprice DESC 
) 
-- Step3:find the item barcode and corresponding brandcode& brand name
SELECT item_barcode,item_finalprice,
brandID, brandCode,
FROM cte
LEFT JOIN BRAND 
ON brand.barcode = cte.item_barcode
order by item_finalprice DESC


```

##### Q6 Which brand has the most transactions among users who were created within the past 6 months?

```sql
-- Q6:Which brand has the most transactions among users who were created within the past 6 months?
-- what is transaction?  points awarded by brand to user?

-- Step 1: Find the user account created in the period and what receipt item they purchase  
with cte as (
SELECT b.userid, b.RECEIPTID,totalspent,RECEIPT_ITEM_PK,item_barcode, REWARDSPRODUCTPARTNERID
from users_flatten a 
JOIN receipt b 
JOIN receipt_item c 
ON b.receiptid = c.receiptID
ON a.oid = b.userid
WHERE LEFT(CREATEDDATE, 7) BETWEEN '2020-09' AND '2021-02'
AND REWARDSPRODUCTPARTNERID is not null
) 
-- Step2: find the brand name in brand table 
SELECT b.brandID, brandname, count(*) as cnt 
from brand b
JOIN CTE 
ON b.cpg_ID = cte.REWARDSPRODUCTPARTNERID
GROUP BY 1,2
order by cnt DESC 
```
</details>


<details>

<summary>Third: Identify Data Quality Issues </summary>

#### Evaluate Data Quality Issues in the Data Provided
```sql
(1) Data quality issue when importing data 
    (1)-a. inconsistant data type
        -- bonus points earned is INT but point is FLOAT => BONUS POINT ALWAYS INT ?
        -- ITEM PRICE & ITEM final price should be float same as total spent 
        -- related date attributes should be timestamp 

    (1)-b. Attributes not standardized 
         when importing data from JSON file, there are some empty spaces or speacil character '$' in the attributes, and not consisitant with 
        lower/capital cases, it may causing data importing null value.

sample SQL
SELECT 
    JSON_DATA:_id.     "$oid"::VARCHAR AS receiptID,
    JSON_DATA:bonusPointsEarned::INT AS bonusPointsEarned,
    JSON_DATA:bonusPointsEarnedReason::STRING AS bonusPointsEarnedReason,
    JSON_DATA:createDate. "$date" ::NUMBER AS createDate,
    JSON_DATA:dateScanned. "$date" ::NUMBER AS dateScanned,
    JSON_DATA:finishedDate. "$date" ::NUMBER AS finishedDate,
    JSON_DATA:modifyDate. "$date" ::NUMBER AS modifyDate,
    JSON_DATA:pointsAwardedDate. "$date" ::NUMBER AS pointsAwardedDate,
    JSON_DATA:pointsEarned::FLOAT AS pointsEarned,
    JSON_DATA:purchaseDate. "$date" ::NUMBER AS purchaseDate,
    JSON_DATA:purchasedItemCount::INT AS purchasedItemCount,
    JSON_DATA:rewardsReceiptStatus::STRING AS rewardsReceiptStatus,
    JSON_DATA:totalSpent::FLOAT AS totalSpent,
    JSON_DATA:userId::VARCHAR AS userID


from FETCH_REWARDS.PUBLIC.RECEIPTS

```

```sql
(2) Missing Value Handling 
    (2)-a. item_barcode with null value in RECEIPT_ITEM table, need to verify if null value needs to fill with other value or ok with it.
            -- fill with other value
            SELECT IFNULL(item_barcode,0) as item_barcode

```

```sql
(3) Duplicated attributes
    (3)-a. item_finalprice & itemprice in RECEIPT_ITEM table got the identical data, need to verify if the finalprice should included tax 
```
```sql
(4) Recalculate Quantities and Total
    (4)-a. Verify purchasedItemCount and totalSpent based on corrected item list.

    SELECT a.receiptID , sum(b.item_finalprice)over(partition by a.receiptID) as sumprice, totalspent
    FROM receipt a 
    JOIN receipt_item b 
    ON a.receiptID = b.receiptID;

```
</details>
<details>

<summary>Forth: Communicate with Stakeholders </summary>

### Communicate with Stakeholders about Data concern
Dear Stakeholder,

I hope this message finds you well.  There are some requests about the current dataset and we identified several data quality issues that need to be clarified.

1. Questions About the Data: <be> 
  * Bonus Points Data Type: Should bonus points always be an integer, or is there a scenario where they could be a float?   <br>
  * Missing Item Barcodes: How should we handle items with null barcodes? Should we fill these with a placeholder value, or is it acceptable to leave them as null?<br>
  * Item Prices: Should the item_finalprice include tax, and should it always be consistent with itemprice?<br>
  * Attribute Standardization: Are there specific guidelines for attribute naming conventions to avoid inconsistencies in cases and special characters?<br>

2. Discovery of Data Quality Issues:<br>
  We discovered these data quality issues during the import process and subsequent data analysis. Here are the specific concerns identified:<br>
  * Inconsistent Data Types: bonusPointsEarned is an integer, whereas pointsEarned is a float. Similarly, itemPrice, item_finalprice, and totalSpent should consistently be floats. Date attributes should be timestamps.<br>
  * Non-standardized Attributes: Some attributes contain empty spaces or special characters like '$' and are inconsistent in their casing (lower/capital), leading to null values during data import.<br>
  * Missing Values: Instances of null values in item_barcode in the RECEIPT_ITEM table, and the related attributes in other tables.<br>

3. Information Needed to Resolve Issues:<br>
  To effectively resolve these data quality issues, we need clarification on the following:<br>
 * The intended data type for bonus points and if any adjustments are necessary for item prices and date attributes.<br>
 * Guidelines for handling null values in item_barcode.<br>
 * Clarification on whether item_finalprice should include tax.<br>
 * Standardization rules for attribute naming conventions to avoid inconsistencies.<br>

4. Additional Information Needed:<br>
  To optimize the data assets we are creating, it would be helpful to have:<br>
  * Detailed documentation on the expected data schema, including data types and naming conventions.<br>
  * A comprehensive list of required fields and acceptable default values for missing data.<br>

5. Performance and Scaling Concerns:<br>
  In production, we anticipate the following performance and scaling concerns:<br>
  * Data Consistency: Ensuring data type consistency and standardized attributes across large datasets can be challenging.<br>
  * Scalability: Efficient handling of increased data volume without compromising performance.<br>
  * Data Validation: Implementing robust data validation mechanisms to catch inconsistencies and missing values during the import process.<br>
To address these concerns, we plan to:<br>
  * Optimize database indexing and query performance to handle large datasets efficiently.<br>
  * Regularly audit and clean the data to maintain high quality.<br>

Your input and guidance on these points would be invaluable as we work to resolve the identified issues and enhance the quality of our data. Please let us know if there are any additional considerations or specific requirements we should take into account.

Thank you for your attention to these matters. We look forward to your feedback and suggestions.

Best regards,


</details>
