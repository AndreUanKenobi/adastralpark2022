# Demo at Openreach 2022

Project: openreachday2022

## Bigtable Federation
1. on Cloud shell, check that data is there

```
cbt read bigtable-timeseries
```

2. verify the table defininition JSON file `tabledef.json` being defined as follows - and upload it to Cloud Shell:

```
{
    "sourceFormat": "BIGTABLE",
    "sourceUris": [
     https://googleapis.com/bigtable/projects/openreachday2022/instances/bigtable-timeseries/tables/bigtable-timeseries
    ],
    "bigtableOptions": {
        "readRowkeyAsString": "true",
        "columnFamilies" : [
            {
                "familyId": "cell_data",
                "onlyReadLatest": "true",
                "columns": [
              {
                  "qualifierString": "attr1",
                  "type": "STRING"
              },
              {
                  "qualifierString": "attr2",
                  "type": "STRING"
              }
              ,
                  {
                  "qualifierString": "attr3",
                  "type": "STRING"
              }
              ,
                  {
                  "qualifierString": "attr4",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr5",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr6",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr7",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr8",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr9",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr10",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "tstamp",
                  "type": "STRING"
              }
          ]

            }
        ]
    }
    
}
```
3. In Cloud Shell, create the external table using the following command
```
bq mk \
--external_table_definition=tabledef.json \
openreach.bigtable-timeseries-federated 
```


4. Query the whole table 
```
SELECT * FROM `openreachday2022.openreach.bigtable-timeseries-federated` LIMIT 1000
```

5. Create a "clean" view of this table
```
SELECT rowkey, 
cast(cell_data.attr1.cell.value as NUMERIC) attr1,
cast(cell_data.attr2.cell.value as NUMERIC) attr2,
cast(cell_data.attr3.cell.value as NUMERIC) attr3,
cast(cell_data.attr4.cell.value as NUMERIC) attr4,
cast(cell_data.attr5.cell.value as NUMERIC) attr5,
cast(cell_data.attr6.cell.value as NUMERIC) attr6,
cast(cell_data.attr7.cell.value as NUMERIC) attr7,
cast(cell_data.attr8.cell.value as NUMERIC) attr8,
cast(cell_data.attr9.cell.value as NUMERIC) attr9,
cast(cell_data.attr10.cell.value as NUMERIC) attr10,
timestamp(cell_data.tstamp.cell.value) ts
FROM `openreachday2022.openreach.bigtable-timeseries-federated` 
```

making sure to persist it to a Native Table `bigtable-timeseries-native`

6. Create a new dataset `reporting`
7. Create an authorized view 'timeseries' in the dataset `reporting` as 

```
SELECT ts,attr1,attr2,attr3  FROM `openreachday2022.openreach.bigtable-timeseries-native`
```
8. Touch of BigQuery ML to create a anomaly detection model:

```

CREATE MODEL `openreach.my_kmeans_model`
OPTIONS(
  MODEL_TYPE = 'kmeans',
  NUM_CLUSTERS = 8,
  KMEANS_INIT_METHOD = 'kmeans++'
) AS
SELECT 
  * EXCEPT(rowkey, ts) 
FROM 
  `openreachday2022.openreach.bigtable-timeseries-native`;

```

9. Detect anomalies:
```
SELECT
  *
FROM
  ML.DETECT_ANOMALIES(MODEL `openreach.my_kmeans_model`,
                      STRUCT(0.02 AS contamination),
                      TABLE `openreachday2022.openreach.bigtable-timeseries-native`)
                
                where is_anomaly is true;
```


## Cloud Storage Federation
1. Move the files to a local bucket (should already be done)
```
gsutil cp -r gs://andreuankenobi-demo-datasets/* gs://openreach-demo-2022
```
2. Create a native table from the federated view `client`

3. First inspect all columns
```
SELECT *
 FROM `openreachday2022.openreach.client` LIMIT 1000
```
4. Create an authorized view `client-curated` from the federated view, using the only non-PII data
```
SELECT gender, client_id, district_id, disp_id, traffice_source, dob, age, job
 FROM `openreachday2022.openreach.client`
 ```

5. Scan the federated table with DLP - to identify PIIs (SSN)
Remember to
- Detection: set Confidence Thresold to "Unspecified"
- Actions: publish to Data Catalog
The Job will run for 2/3 mins, in case show the already-run example

6. Create and assign policy tags:
- In Data Catalog, scearh for tag template "Data Loss Prevention Tags"
- The "Client" table will pop up. Preview it in the preview panel to confirm SSN is there 
- Click on the table, and confirm the SSN tag has been fired
- Confirm how there's not policy tag attached to the column 
- Scroll up, and select "Open in BigQuery"
- Under Schema, click "Edit Schema" - on the bottom
- Select SSN, select "Add Policy Tag"
- The table is now secured! 

## Dataplex Data Mesh
Referece user `tadej@2023528041521.altostrat.com` / `TadejRule5!@`
- Create "My first lake"  (will find it already created)
- Create the "Raw" Data zone 
- Add Assets:
  - BigQuery tables
  - Storage Buckets (mind the region!)
  - Grant Tadej view role on Dataplex resources. 
