# BODS-Data-Integrity-Checks
Functions for SAP Data Services to perform basic data integrity checks at the beginning and end of a data load. 

## Overview
Every batch data integration (ETL) job should employ measures to ensure data integrity from source to target. For example, when loading a data warehouse, we should be able to confirm that the sum of the daily sales has been loaded to the target weekly sales bucket. These checks are typically source and target record counts or record sums. Data Services has some built-in auditing functionality, but it is rather simplistic and limited.

## Requirements
- This method is tested with SAP Data Services 4.2. It should work with earlier versions, but has not been tested.
- A database where you can create a table that will hold a history of the data integrity checks.
  - I recommend having an ETL support database that resides on the same server as your repositories. There are a number of instances where it's handy to be able to persist data related to the operation of Data Services. In my example, the name of the DB is etl_support.
  - In the functions below we assume this Datastore is named ETL_Support_DS.
- A solid understanding of your source and target systems and how you will know that your source data has been successfully moved to the target.

## Setup
1. Create a new table in your ETL Support database. Below is the basic schema used in this example.

| Column | Characteristics | Data Type | Notes |
| --- | --- | --- | --- |
| job_data_integrity_id | PK, Identity (1,1) | Int | Update key |
| job_name | NK, Not Null | Varchar(64) |
| job_execution_date | NK, Not Null | Datetime |
| data_point_label | NK, Not Null | Varchar(100) |
| source_value | Not Null | Decimal(28,10) | 28 is the max precision Data Services can handle. |
| target_value | Null | Decimal(28,10) |
| error_tolerance_threshold | Null | Decimal(10,9) | Optional, -1 indicates disregard |
| warning_tolerance_threshold | Null | Decimal(10,9) | Optional, -1 indicates disregard |

2. Create custom functions in Data Services. 
- CF_Data_Integrity_Insert
- CF_Data_Integrity_Update

## Usage
The relative difference between source and target is calculated as follows: `abs(source_value - target_value) / source_value`

**The basic Data Services batch job flow would be:**
1. In the initialize script, capture current date/time in a global variable.
2. In the initialize script, capture required source counts and/or other values. For each: Execute a Data Services function that inserts a record into the above table. 
3. Do the movement/transformation of data from source to target.
4. In the finalize script, capture the required target counts and/or values. For each, Execute a Data Services function that updates the record from step 2 above with the target value and error/warning tolerance threshold values. The error threshold will be checked first; violation of error threshold will cause the script to raise an exception.

Violation of warning threshold will cause a warning message to be printed. A function return code of "2" will indicate a warning and the developer can take additional steps if they chose (ex: email someone).

**Example initialize script**

```
# $GV_LoadDateTime is the job execution date/time
# 'sales_budget_revenue' is the data point label specific to this job
# 2.1 is the source value - this would typically be in the form of a variable containing
# a record count or aggregate value. This cannot be null or zero.

$GV_LoadDateTime = sysdate( );
CF_Data_Integrity_Insert($GV_LoadDateTime, 'sales_budget_revenue', 2.1);
```

**Example finalize script**

```
# This function returns 1 if everything is fine and 2 if the warning threshold was breached.
# 2 is the target value - this would typically be in the form of a variable and cannot be null
# .05 is the error tolerance threshold. Enter a value between 0 and 1 or -1 to ignore
# -1 is the warning tolerance threshold. Enter a value between 0 and 1 or -2 to ignore

if (CF_Data_Integrity_Update($GV_LoadDateTime, 'sales_budget_revenue', 2, .05, -1) = 2)
begin
  print('Data integrity warning threshold breached - consider emailing someone.');	
end
```

**Example Data**

|Job data integrity id | Job name | Job execution date | Data point label | Source value | Target value | Error tolerance threshold | Warning tolerance thershold |
|---|---|---|---|---|---|---|---|
| 1 | JOB_Strategic_AP_Demand | 11/3/2014 10:57:01 AM | Demand_lbs_total | 1118123456.123 | 1118123990.123 | .01 | .005 | 
| 2 | JOB_Strategic_AP_Demand | 11/3/2014 10:57:01 AM | Budget_total | 618123456.123 | 618123456.123 | .01 | .005 |
| 3 | JOB_Budget_Load | 11/3/2014 11:03:00 AM | Record_count | 550 | 550 | 0 | -1 |
