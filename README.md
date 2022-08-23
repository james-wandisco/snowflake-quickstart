# snowflake-quickstart

Replicating data from HDFS to Azure for Snowflake consumption.

### Environment
1. Build LDM on hadoop with Azure as a target. Azure target should have a adlsg2 storage account and container.

### Example source data
2. Pull data 

This sample data is a large csv file of dublin bike share data taken from here..

   https://data.gov.ie/dataset/dublinbikes-api
```
   curl -o data.csv https://data.smartdublin.ie/dataset/33ec9fe2-4957-4e9a-ab55-c5e917c7a9ab/resource/8ddaeac6-4caf-4289-9835-cf588d0b69e5/download/dublinbikes_20200401_20200701.csv
```

3. Put data into HDFS
```
kinit -kt /etc/security/keytabs/hdfs.headless.keytab hdfs-jhugh-02@TARGETREALM.HADOOP
```
```
hdfs dfs -mkdir -p /data/bikes/
```
```
hdfs dfs -put data.csv /data/bikes/
```
### Replicate ###
4. Add a LDM rule to replicate data to Azure. 

Get the source fs id.
```
curl -s -X 'GET'   'http://10.6.123.132:18080/fs/sources?verbose=true' | grep "fileSystemId" | awk '{print $3}' | sed 's/^"\([^"]*\).*/\1/'
```
Get the target fs id
```
curl -s -X 'GET'   'http://10.6.123.132:18080/fs/targets?verbose=true' | grep "fileSystemId" | awk '{print $3}' | sed 's/^"\([^"]*\).*/\1/'
```
Add the rule.
```
curl -X 'PUT' 'http://10.6.123.132:18080/migrations/bikes?path=/data/bikes/&source=source_jhugh02&target=AzureTarget&actionPolicy=com.wandisco.livemigrator2.migration.OverwriteActionPolicy&autoStart=true'
```
It was set to **autostart**, so check its status. If you **did not** start it then do so.
```
curl -s -X 'GET' 'http://10.6.123.132:18080/migrations/bikes' | grep "state" | awk '{print $3}' | sed 's/^"\([^"]*\).*/\1/'
curl -s -X 'POST' 'http://10.6.123.132:18080/migrations/bikes/start'
```
### Snowflake it up ###
5. Get a Snowflake instance. 

Sign up from here... (Click 'Start for free')
   https://www.snowflake.com/creating-azure-data-warehouse-snowflake/

6. Create your Snowflake dbs and tables.

Log into Snowflake, go to the 'Worksheets'. 
Execute these commands from here to create a database, table and import data. 

*You can use the UI also. But using the comparable SQL commands here.*
```
create DATABASE IDENTIFIER('"AzureDataDB"') COMMENT = '';


create or replace TABLE bikeDataTable (
"STATION_ID" INT,
"TIME" DATE,
"LAST_UPDATED" DATE,
"NAME" VARCHAR(50),
"BIKE_STANDS" INT, 
"AVAILABLE_BIKE_STANDS" INT, 
"AVAILABLE_BIKES" INT, 
"STATUS" VARCHAR(10), 
"ADDRESS" VARCHAR(50), 
"LATITUDE" DECIMAL(10,8), 
"LONGITUDE" DECIMAL(10,8)
);
```

7. Pull in data from Azure. 

You will a shared access signature (SAS) key.
First get the Azure SAS key(You can do this as usual with the UI, using the CLI here.)
```
az storage account keys list --resource-group SUPPORT-james.hughes4 --account-name jhadlsg2 | grep "value" | awk '{print $2}' | head -1 | sed 's/^"\([^"]*\).*/\1/'
az storage account generate-sas --account-key <KEY HERE> --account-name jhadlsg2 --expiry 2022-08-24T17:26Z --https-only --permissions rwdlacyx --resource-types sco --services b --start 2022-08-22T10:26Z
```

Add the SAS returned value to the following SQL for snowflake.
```
copy into AzureDataDB.bikeDataTable 
from 'azure://jhadlsg2.blob.core.windows.net/snowflake/data/bikes/data.csv' 
credentials=(AZURE_SAS_TOKEN='☺☺☺-your-token-here☺☺☺')
FILE_FORMAT = (TYPE = CSV FIELD_DELIMITER = ',' SKIP_HEADER = 1);
```

** Thats it, you will have data in Snowflake **

**8. Extra Notes**

Retrieve your storage account key
```
az storage account keys list --resource-group SUPPORT-james.hughes4 --account-name jhadlsg2 | grep "value" | awk '{print $2}' | head -1 | sed 's/^"\([^"]*\).*/\1/'
```
Use the key to get a SAS token
```
az storage account generate-sas --account-key <KEY HERE> --account-name jhadlsg2 --expiry 2022-08-24T17:26Z --https-only --permissions rwdlacyx --resource-types sco --services b --start 2022-08-22T10:26Z
```

List files in adls storage
```
curl -X GET -H "x-ms-version: 2019-12-12" "https://jhadlsg2.dfs.core.windows.net/snowflake?recursive=true&resource=filesystem&directory=data<INSERT TOKEN HERE>" | jq
```
### References ###

az command reference.

https://docs.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest

Azure API reference

https://docs.microsoft.com/en-us/rest/api/storageservices/datalakestoragegen2/filesystem/list


