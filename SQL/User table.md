

```sql
--STEP1   CREATE DATABASE & SCHEMA  & TABLES 
show databases
use fetch_rewards;

show schemas;
use schema public;

CREATE OR REPLACE TEMPORARY TABLE users (
  json_data VARIANT);
  --keep your data as JSON then need to load it into a column with a datatype of VARIANT, not VARCHAR 
SELECT * FROM Users;


--STEP2  CREATE INTERNAL STAGE
CREATE OR REPLACE STAGE my_internal_stage;

--STEP3  CREATE FILE FORMAT 
CREATE OR REPLACE FILE FORMAT myjsonformat
TYPE = 'JSON'
STRIP_OUTER_ARRAY = TRUE;

list @my_internal_stage;

--STEP4  STAGE THE DATA FILES 
-- USE SNOWSQL COMMAND ON CLI
PUT file:///Users/sakichen/Desktop/Fetch/Data/users.json @my_internal_stage;

-- STPE5 COPY DATA INTO THE TARGET TABLE
  COPY INTO users
  FROM @my_internal_stage/users.json.gz
  FILE_FORMAT = (FORMAT_NAME = myjsonformat)
  ON_ERROR = 'skip_file';

--STEP6 CHECK IF LOAD ERRORS AND FIX 
SELECT * FROM USERS LIMIT 5;

-- STEP7 REMOVE THE DATA FILE FROM STAGE
REMOVE @my_internal_stage PATTERN='.*.json.gz';

-- FLATTEN

SELECT replace(JSON_DATA:_id:     "$oid",'"') as user_id,
       JSON_DATA:active::Boolean as active,
       JSON_DATA:createdate:     "$date"::NUMBER as createdate,
       JSON_DATA:laslogin:     "$date" as lastlogin,
       replace(JSON_DATA:role, '"')as role,
       replace(JSON_DATA:signUpSource, '"')as signUpSource,
       JSON_DATA:state::string as state 
from FETCH_REWARDS.PUBLIC.USERS;



--  with space in each column ) , speacil character '$', no unique identifier userID1,2,3......., 


select JSON_DATA:createdate as tmp
from users;

select json_data from users limit 1;

WITH users_temp AS (
  SELECT 
    json_data:_id.     "$oid"::VARCHAR AS oid,
    json_data:state::VARCHAR AS state,
    json_data:createdDate.     "$date"::NUMBER AS created_date_ms,
    json_data:lastLogin.     "$date"::NUMBER AS last_login_ms,
    json_data:role::VARCHAR AS role,
    json_data:signUpSource::VARCHAR AS signUpSource,
    json_data:active::VARCHAR AS active
  FROM users
)
 SELECT 
  oid,
  state,
  TO_TIMESTAMP(created_date_ms / 1000) AS createdDate,
  TO_TIMESTAMP(last_login_ms / 1000) AS lastLogin,
  role,
  signUpSource,
  active
FROM users_temp;


-- ==========================================================
CREATE OR REPLACE table users_flatten 
AS 
select 
-- row_number()over(order by oid) as rid_no,
  userid,
  state,
  TO_TIMESTAMP(created_date_ms / 1000) AS createdDate,
  TO_TIMESTAMP(last_login_ms / 1000) AS lastLogin,
  role,
  signUpSource,
  active
FROM 
(SELECT 
    json_data:_id.     "$oid"::VARCHAR AS userid,
    json_data:state::VARCHAR AS state,
    json_data:createdDate.     "$date"::NUMBER AS created_date_ms,
    json_data:lastLogin.     "$date"::NUMBER AS last_login_ms,
    json_data:role::VARCHAR AS role,
    json_data:signUpSource::VARCHAR AS signUpSource,
    json_data:active::VARCHAR AS active
  FROM users);

select * FROM users_flatten limit 3;




















```