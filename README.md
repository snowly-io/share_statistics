![alt text](https://snowly.io/img/svg/snowly_logo_beta.svg "Snowly.io")

# Intro
To help you analyze Snowflake's usage patterns and monitor your cost, Snowly needs some information about the way you use your Snowflake account. For example: what type of queries you typically runs, at what times and by whom.
<div style="width=100%; text-align:center"><img src="https://snowly.io/img/database.png" align="center" alt="SnowShare" width="400"/></div><br>
No actual data (i.e. from your data tables) is being shared with Snowly. Only information about the use of Snowflake is being shared. For example: query execution times, utilization of warehouses etc.
 [READ MORE](https://snowly.io/how-to)

# Install 2.3
## What does the installation script do?
1. Create a new database for the shared information
2. Create a process (stored procedure) to keep the information updated
3. Make sure that the updating process is running regularly (scheduled)
4. Establish a Snowflake data-sharing


## Got it. Let's do this.

### Create a new database
This script requires ACCOUNTADMIN ROLE in order to create a Share and read from "SNOWFLAKE" internal database
```sql
USE role ACCOUNTADMIN;
```

Create a new database to be shared later using SnowShare
```sql
Create database SHARE_USAGE ;
Use database SHARE_USAGE;
Create Schema ACCOUNT_USAGE;
USE Schema ACCOUNT_USAGE;

```

### Create s stored procedure
The full script can be found [here](/SP_ACCOUNT_USAGE_DATA.md)
This is the object that will be created
```sql
CREATE OR REPLACE PROCEDURE SHARE_USAGE.ACCOUNT_USAGE.SP_ACCOUNT_USAGE_DATA(LOAD_METHOD STRING)
RETURNS STRING
LANGUAGE javascript
strict
EXECUTE AS owner
AS
$$ 
.... 
``` 
[Full Create Script](/SP_ACCOUNT_USAGE_DATA.md)


### Run Once
Execute the procedure above to create the tables and load the data
```sql
call SHARE_USAGE.ACCOUNT_USAGE.SP_ACCOUNT_USAGE_DATA('FULL');
```


### Schedule task to update the tables 

*NOTE: it is recommended to create a separate Warehouse to monitor the cost of the sync
```sql
CREATE WAREHOUSE WH_SNOWLY WITH WAREHOUSE_SIZE = 'XSMALL' WAREHOUSE_TYPE = 'STANDARD' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;
```
This task will be executed every 8 hours at 04:00, 12:00 and 20:00 UTC
```sql
  --CREATE TASK
  create or replace task SHARE_USAGE.ACCOUNT_USAGE.TASK_ACCOUNT_USAGE
  warehouse = WH_SNOWLY
  schedule = 'using cron 0 4,12,20 * * * UTC'
    as
  call SHARE_USAGE.ACCOUNT_USAGE.SP_ACCOUNT_USAGE_DATA('');
```

Activate the Task
```sql
--ACTIVATE TASK
  ALTER TASK SHARE_USAGE.ACCOUNT_USAGE.TASK_ACCOUNT_USAGE  RESUME;
```
<div data-v-241e3379="" class="msg-box-one two" style="padding-left: 180px;padding-right: 180px;"><img src="https://snowly.io/img/yellow-shape-2.svg">
</div>

## Share
Up until now, no data was shared

### Create a Snowflake data sharing

```sql
set add_comment= CONCAT(current_account(),' | ',  current_region());

CREATE SHARE SHARE_USAGE_SH COMMENT=$add_comment ;
grant usage on database  SHARE_USAGE  to share SHARE_USAGE_SH;
```

### Share with Snowly Snowflake account
Data sharing is availble only between same regions.
Supported regions are:

| Cloud | Region | Snowly Account | NAME |
|-|-|-|-|
| AWS | US-EAST-1 | CH52627 | US East (N. Virginia) |
| AWS | EU-CENTRAL-1 | YA21808 | EU (Frankfurt) |
| AWS | EU-WEST-1 | HB37582 | EU (Ireland) |
| AZURE | EAST-US-2 | XL70015 | East US 2 (Virginia) |
| AZURE | WEST-EUROPE | VISIONBI | West Europe (Netherlands) |
| GCP |  |  |Comming Soon... |

Share Data
```sql
ALTER SHARE SHARE_USAGE_SH ADD ACCOUNTS = <ACCOUNT_NAME>;
```
Example
```sql
-- FOR AWS US-EAST:
ALTER SHARE SHARE_USAGE_SH ADD ACCOUNTS = CH52627;
```

### Grant permissions
Last step - grant permissions to the Share
```sql
grant usage on schema SHARE_USAGE.ACCOUNT_USAGE to share SHARE_USAGE_SH;
grant select on all tables in schema SHARE_USAGE.ACCOUNT_USAGE to share SHARE_USAGE_SH;
GRANT USAGE ON SCHEMA SHARE_USAGE.ACCOUNT_USAGE TO SHARE SHARE_USAGE_SH;
GRANT SELECT ON VIEW SHARE_USAGE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE_HISTORY TO SHARE SHARE_USAGE_SH;
GRANT SELECT ON VIEW SHARE_USAGE.ACCOUNT_USAGE.QUERY_HISTORY TO SHARE SHARE_USAGE_SH;
GRANT SELECT ON VIEW SHARE_USAGE.ACCOUNT_USAGE.STORAGE_USAGE TO SHARE SHARE_USAGE_SH;
GRANT SELECT ON VIEW SHARE_USAGE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS TO SHARE SHARE_USAGE_SH;
GRANT SELECT ON VIEW SHARE_USAGE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY TO SHARE SHARE_USAGE_SH;
GRANT SELECT ON VIEW SHARE_USAGE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY TO SHARE SHARE_USAGE_SH;
GRANT SELECT ON VIEW SHARE_USAGE.ACCOUNT_USAGE.LOGIN_HISTORY TO SHARE SHARE_USAGE_SH;
GRANT SELECT ON VIEW SHARE_USAGE.ACCOUNT_USAGE.CLOUD_SERVICES_WO_WAREHOUSE_SIZE TO SHARE SHARE_USAGE_SH;
GRANT SELECT ON VIEW SHARE_USAGE.ACCOUNT_USAGE.META_DATA_CHANGES TO SHARE SHARE_USAGE_SH;
```





