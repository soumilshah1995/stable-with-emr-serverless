# stable-with-emr-serverless
stable-with-emr-serverless

![Screenshot 2024-03-15 at 11 43 22 AM](https://github.com/soumilshah1995/stable-with-emr-serverless/assets/39345855/00341603-e6d5-4c98-a7b9-d61d9e0e4d02)

# step 1: Download test data and jar files and upload then into folder /jar and test/ into S3
Link: https://drive.google.com/drive/folders/1mQzZSVgxQGksoXkYGb8DSn2jeR48VehS?usp=share_link

# Step 2: Create EMR Spark Cluster 

![image](https://github.com/soumilshah1995/stable-with-emr-serverless/assets/39345855/8b042c5a-2e15-434e-865b-47ad57f13615)


# Step 2: Edit ENV file
```
DEV_ACCESS_KEY="XX"
DEV_SECRET_KEY="XXX"
DEV_REGION='us-east-1'
ApplicationId="XX"
ExecutionArn="XX"
```

# Step 3: Install Boto3 and dotend librray 
```
pip install python-dotenv
pip install boto3
```

# Step 4: Submit Delta Streamer Job
```
try:
    import json
    import uuid
    import os
    import boto3
    from dotenv import load_dotenv

    load_dotenv(".env")
except Exception as e:
    pass

global AWS_ACCESS_KEY
global AWS_SECRET_KEY
global AWS_REGION_NAME

AWS_ACCESS_KEY = os.getenv("DEV_ACCESS_KEY")
AWS_SECRET_KEY = os.getenv("DEV_SECRET_KEY")
AWS_REGION_NAME = os.getenv("DEV_REGION")

client = boto3.client("emr-serverless",
                      aws_access_key_id=AWS_ACCESS_KEY,
                      aws_secret_access_key=AWS_SECRET_KEY,
                      region_name=AWS_REGION_NAME)


def lambda_handler_test_emr(event, context):
    # ------------------Hudi settings ---------------------------------------------
    glue_db = "hudi_db"
    table_name = "invoice"
    op = "UPSERT"
    table_type = "COPY_ON_WRITE"

    record_key = 'invoiceid'
    precombine = "replicadmstimestamp"
    partition_feild = 'destinationstate'
    source_ordering_field = 'replicadmstimestamp'

    delta_streamer_source = 's3://XXX/test/'
    hudi_target_path = 's3://XX/testcases/'

    # ---------------------------------------------------------------------------------
    #                                       EMR
    # --------------------------------------------------------------------------------
    ApplicationId = os.getenv("ApplicationId")
    ExecutionTime = 600
    ExecutionArn = os.getenv("ExecutionArn")
    JobName = 'delta_streamer_{}'.format(table_name)

    # --------------------------------------------------------------------------------

    spark_submit_parameters = ' --conf spark.jars=/usr/lib/hudi/hudi-utilities-bundle.jar,s3://soumil-dev-bucket-1995/jar/hudi-extensions-0.1.0-SNAPSHOT-bundled.jar,s3://soumil-dev-bucket-1995/jar/hudi-java-client-0.14.0.jar'
    spark_submit_parameters += ' --conf spark.serializer=org.apache.spark.serializer.KryoSerializer'
    spark_submit_parameters += ' --conf spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension'
    spark_submit_parameters += ' --conf spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog'
    spark_submit_parameters += ' --conf spark.sql.hive.convertMetastoreParquet=false'
    spark_submit_parameters += ' --conf mapreduce.fileoutputcommitter.marksuccessfuljobs=false'
    spark_submit_parameters += ' --conf spark.hadoop.hive.metastore.client.factory.class=com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory'
    spark_submit_parameters += ' --class org.apache.hudi.utilities.deltastreamer.HoodieDeltaStreamer'

    arguments = [
        "--table-type", table_type,
        "--op", op,
        "--enable-sync",
        "--sync-tool-classes", "io.onetable.hudi.sync.OneTableSyncTool",
        "--source-ordering-field", source_ordering_field,
        "--source-class", "org.apache.hudi.utilities.sources.ParquetDFSSource",
        "--target-table", table_name,
        "--target-base-path", hudi_target_path,
        "--payload-class", "org.apache.hudi.common.model.AWSDmsAvroPayload",
        "--hoodie-conf", "hoodie.datasource.write.keygenerator.class=org.apache.hudi.keygen.SimpleKeyGenerator",
        "--hoodie-conf", "hoodie.datasource.write.recordkey.field={}".format(record_key),
        "--hoodie-conf", "hoodie.datasource.write.partitionpath.field={}".format(partition_feild),
        "--hoodie-conf", "hoodie.deltastreamer.source.dfs.root={}".format(delta_streamer_source),
        "--hoodie-conf", "hoodie.datasource.write.precombine.field={}".format(precombine),
        "--hoodie-conf", "hoodie.database.name={}".format(glue_db),
        "--hoodie-conf", "hoodie.datasource.hive_sync.enable=true",
        "--hoodie-conf", "hoodie.datasource.hive_sync.table={}".format(table_name),
        "--hoodie-conf", "hoodie.datasource.hive_sync.partition_fields={}".format(partition_feild),
        "--hoodie-conf", "hoodie.onetable.formats.to.sync=ICEBERG",
        "--hoodie-conf", "hoodie.onetable.target.metadata.retention.hr=168"

    ]

    response = client.start_job_run(
        applicationId=ApplicationId,
        clientToken=uuid.uuid4().__str__(),
        executionRoleArn=ExecutionArn,
        jobDriver={
            'sparkSubmit': {
                'entryPoint': "command-runner.jar",
                'entryPointArguments': arguments,
                'sparkSubmitParameters': spark_submit_parameters
            },
        },
        executionTimeoutMinutes=ExecutionTime,
        name=JobName,
    )
    print("response", end="\n")
    print(response)


lambda_handler_test_emr(context=None, event=None)

```
