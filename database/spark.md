# Azure Spark Connectors

## Read & write from SQL DB
```python
jdbcHostname = "servername.database.windows.net" 
jdbcPort     = "1433"
jdbcDatabase = "dbname" 
properties   = {
  "user" : "dbuser", 
  "password" : "dbpassword" 
}

url = "jdbc:sqlserver://{0}:{1};database={2}".format(jdbcHostname, jdbcPort, jdbcDatabase)

# read
sdf = sqlContext.read.jdbc(url=url, table="tablename", properties=properties)

# or
sdf = (
  spark.read 
       .format("jdbc") 
       .option("url", url) 
       .option("dbtable", "tablename") 
       .option("user", properties.get("user")) 
       .option("password", properties.get("password"))
       .load()
)

# write 
sdf.write.jdbc(url, "table_modified", properties=properties)
```

## Read & write from Azure Blob Storage

Blob structure
```
blob_account
└── blob_container
    └── path_to_file
        └── file.csv [data/file.csv is the blob_path] 
```

```python
blob_account   = "blobname"
blob_container = "containername"
blob_sas_token = "Password123!"

spark.conf.set('fs.azure.sas.%s.%s.blob.core.windows.net' % (blob_container, blob_account), blob_sas_token)

blob_path  = "<path>/<to>/<file>/0001.csv" 

wasbs_path = 'wasbs://%s@%s.blob.core.windows.net/%s' % (blob_container, blob_account, blob_path)

# read
sdf = (
  spark.read
       .option("delimiter", ",")
       .option("inferSchema", "false")
       .option("header", "true")
       .csv(wasbs_path)
)
```

## Read & write from Azure Data Lake

ADLS structure
```
storage_account
└── file_system [container]
    └── folder_path 
        └── file.csv [filename] 
```

```python
storage_account = "adlsname"
file_system     = "sysname" 
folder_path     = "<path>/<to>/<folder>"
filename        = "0001.xlsx"

spark.conf.set(
  "fs.azure.sas.%s.%s.blob.core.windows.net" % (file_system, storage_account),
  "Password123!"
)

path  = "abfss://%s@%s.dfs.core.windows.net/%s/%s" % (file_system, storage_account, folder_path, filename)

# read
sdf = (
    spark.read
        .format("com.crealytics.spark.excel")
        .option("header", "true")
        .load(path)
)

# write
filename_out = "0001.parquet"
path_out     = "abfss://%s@%s.dfs.core.windows.net/%s/%s" % (file_system, storage_account, folder_path, filename_out)
sdf.write.parquet(path_out)
```