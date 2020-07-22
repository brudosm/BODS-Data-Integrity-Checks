# BODS-Data-Integrity-Checks
Functions for SAP Data Services to perform basic data integrity checks at the beginning and end of a data load. 

## Overview
Every batch data integration (ETL) job should employ measures to ensure data integrity from source to target. For example, when loading a data warehouse, we should be able to confirm that the sum of the daily sales has been loaded to the target weekly sales bucket. These checks are typically source and target record counts or record sums. Data Services has some built-in auditing functionality, but it is rather simplistic and limited.

## Requirements
- This method is tested with SAP Data Services 4.2. It should work with earlier versions, but has not been tested.
- A database where you can create a table that will hold a history of the data integrity checks.
-- I recommend having an ETL support database that resides on the same server as your repositories. There are a number of instances where it's handy to be able to persist data related to the operation of Data Services. In my example, the name of the DB is etl_support.
- A solid understanding of your source and target systems and how you will know that 

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

2. Create custom function in Data Services. 
- CF_Data_Integrity_Insert
- CF_Data_Integrity_Update
