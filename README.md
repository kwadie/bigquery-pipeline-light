# bigquery-pipeline-light
A Light-weight pipeline to regularly load incremental data from GCS to BigQuery
using BigQuery Transfer Jobs, Scheduled Queries and MERGE sql-statement 

## Demo

#### Create a BigQuery Dataset
```
bq mk elt_demo
```

#### Create BigQuery Tables
```
bq query --use_legacy_sql=false \
"CREATE TABLE elt_demo.main
(
 key INT64,
 value STRING
);

CREATE TABLE elt_demo.staging
(
 key INT64,
 value STRING
);"
```

#### Load initial data to BigQuery
```
bq query --use_legacy_sql=false \
"INSERT INTO elt_demo.main 
SELECT 1 AS key, 'A' AS value UNION ALL 
SELECT 2 AS key, 'B' AS value"
```

#### Create new data file
```
echo "key;value" >> new_data.txt 
echo "1;A" >> new_data.txt  
echo "2;Changed" >> new_data.txt  
echo "3;New" >> new_data.txt
``` 

#### Copy new data file to GCS
```
gsutil cp new_data.txt gs://<bucket>/dir/
```

#### Create BigQuery Scheduled Transfer

Go to BigQuery >> Transfers >> Create New Transfer
and set up a GCS transfer to BigQuery 

![](images/transfer_job_1.png?raw=true)
![](images/transfer_job_2.png?raw=true)
![](images/transfer_job_3.png?raw=true)

Notes:
* Since we're not recreating the staging table at every load we need to
truncate the existing table and load new data as-is. This is the "Write Preference=Mirror" config.
* In this demo we choose to delete the source files after loading to BigQuery for simplicity to avoid 
  reloading the same data in next runs. [Read more](https://cloud.google.com/bigquery-transfer/docs/gcs-transfer-parameters#loading_a_snapshot_of_all_data_into_an_ingestion-time_partitioned_table) about loading data from time partitioned buckets.



#### Schedule a MERGE query

In a BigQuery tab, copy the below query
```
MERGE elt_demo.main M
USING elt_demo.staging S
ON M.key = S.key
WHEN MATCHED THEN
  UPDATE SET value = s.value
WHEN NOT MATCHED THEN
  INSERT (key, value) VALUES(key, value)
``` 
Then, select Schedule Query > Create new scheduled query

![](images/scheduled_query_button.png?raw=true)

Finally, set up the a schedule some time after the GCS transfer job start time 

![](images/scheduled_query_config.png?raw=true)

Notes:
 * one will need am estimate on how long the transfer job takes and add a reasonable buffer time
## Limitations
* [Cloud Storage Transfer limits](https://cloud.google.com/bigquery-transfer/docs/cloud-storage-transfer#limitations)
* For this demo, GCS-BigQuery Data Transfer jobs has a [minimum interval limitation](https://cloud.google.com/bigquery-transfer/docs/cloud-storage-transfer#minimum_intervals) which means
  that the job won't be able to pick up the newly created file on GCS in the first 60 minutes of creating it. One would need to wait 60 minutes after creating the new_data.txt file on GCS before running the pipeline end-to-end.  
## Further Reading
* [Loading a snapshot of all data into an ingestion-time partitioned table](https://cloud.google.com/bigquery-transfer/docs/gcs-transfer-parameters#loading_a_snapshot_of_all_data_into_an_ingestion-time_partitioned_table)
* [Setting up a cloud storage transfer](https://cloud.google.com/bigquery-transfer/docs/cloud-storage-transfer?hl=en_US#setting_up_a_cloud_storage_transfer)
* [CRON ref](https://cloud.google.com/appengine/docs/standard/python/config/cronref)