1. OOM exceptions commonly happen when an Apache Spark job reads a large number of small files from Amazon Simple Storage Service (Amazon S3). Resolve driver OOM exceptions with DynamicFrames using one or more of the following methods. use useS3ListImplementation - It is explained here. [1]

I would like to recommend is enabling useS3ListImplementation. With useS3ListImplementation enabled  "AWS Glue doesn't cache the list of files in memory all at once. Instead, AWS Glue caches the list in batches.This will overall improve the execution time of the glue job that you are running. 

     The advantage of using this is to have an optimized mechanism to list files on S3 while reading data into a DynamicFrame. The Glue S3 Lister can be enabled by setting the DynamicFrame’s additional_options parameter useS3ListImplementation to True. The Glue S3 Lister offers advantage over the default S3 list implementation by strictly iterating over the final list of filtered files to be read.

           Here's an example of how to enable useS3ListImplementation with from_catalog:
           datasource0 = glueContext.create_dynamic_frame.from_catalog(database = "database", table_name = "table", additional_options = {'useS3ListImplementation': True}, transformation_ctx = "datasource0")

           Here's an example of how to enable useS3ListImplementation with from_options:
           datasource0 = glueContext.create_dynamic_frame.from_options(connection_type="s3", connection_options = {"paths": ["s3://input_path"], "useS3ListImplementation":True,"recurse":True}, format="json")

     The useS3ListImplementation feature is an implementation of the Amazon S3 ListKeys operation, which splits large results sets into multiple responses. It's one of the best practice as well to use useS3ListImplementation in case you want to use Glue Job bookmarks down the line. USE PARQUET FILES instead of csv files while writing the data as an output in the Glue ETL Job. ( attached the example on how to use parquet while writing a dataframe in Spark ) 

        Also please keep in mind The 'useS3ListImplementation' option is intended for Driver (OOM issues  if any ) and thus will not result in any improvements for the eexecutors (OOM issue if any ). 
        
        Executor (OOM issues ) are simply caused by Spark parallelism: Spark will distribute your dataset across a cluster of nodes, and if one of these nodes receives more data than it can handle, the executor running within it will crash. This data excess can be caused by improper partitioning (data skew, as imbalance in the data distribution resulting in one node receiving more data than the rest) or by non-splittable compression formats being used (as uncompressed data in memory can consume much larger space and Spark has no way to predict the uncompressed size of the partition).


2. Add more DPU's - Please take a deeper look at the job execution metric which shows the required vs  allocated max dpu(s) and accordingly increase the number of DPU's. This is explained well here[4] how to interpret the metrics.

3. If you are using Parquet format for the output datasets while writing , you can definitely use  --enable-s3-parquet-optimized-committer  —this   Enables the EMRFS S3-optimized committer for writing Parquet data into Amazon S3. You can supply the parameter/value pair via the AWS Glue console when creating or updating an AWS Glue job. Setting the value to true enables the committer. By default the flag is turned off. The details are provided here [5]

4. Input Datasets Compression type - In case you are using parquet format for the input datasets , please ensure those are compressed . for example with parquet - snappy compression type goes really well.

5. Too many small files or some large files in input ? - The input datasets as a general practice should not contain too many small file like in Kb's or MB's or few large files like in GB's. The number of input files for the datasets must be good in size and fairly distributed across the partitions.

6. if the dataset (SOURCE) is partitioned - You can try using pushdown predicate as well within glue. this restricts the amount of data being read by spark in the first place. Explained here [6].

7. Another thumb rule of spark while reading the input datasets please read the data only which is required. If there are certain attributes(columns) in the sourced which are not required in the output datasets - please drop/filter them in the first place. or there might be certain attributes which are used as a look-up/Reference - once these are used you can drop them early rather than taking them until the end write them in the output. This will save overall execution time and the memory on the driver and the executors.

8. In Spark, there are 2 versions of the output committer v1 and v2. When File Output Committer Algorithm version is 1 is used, both commitTask and commitJob operations are performed. When File Output Committer Algorithm version is 2[1], commitTask moves data generated by a task directly to the final destination and commitJob is basically a no-op. Glue Service sets a default value for "spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version" parameter as 2 in the background - this is part of the service design.

Suggestions:
You can consider the following options to mitigate this issue:

1. Use EMRFS S3-optimized Committer: 
Glue support EMRFS S3-optimized Committer for Glue ETL jobs. The EMRFS S3-optimized committer is an alternative OutputCommitter implementation that is optimized for writing Parquet files to Amazon S3 which was only available in EMR earlier[2][3]. It helps to avoid issue that can occur with Amazon S3 eventual consistency during job and task commit phases, and helps improve job correctness under task failure conditions. To use the optimized S3 committer, you can supply Key/Value pair, “--enable-s3-parquet-optimized-committer, true” as special Job parameters[4].

Through Glue console while creating or editing the job:

    Go to 'Security configuration, script libraries, and job parameters (optional)' section
    Under 'Job parameters', enter key-value pair as below:


Through Glue APIs via CLI or SDK:

    If you are using Glue API to create Jobs, you can pass the Key/Value pair“--enable-s3-parquet-optimized-committer", "true” in the DefaultArguments[5] to enable this feature while creating Jobs.
    This parameter can also be passed with StartJobRun API in Arguments[6].


    2. Use coalesce() or repartition() reduce the number of output partitions/files.
    Another approach to reduce the chances of such issues due to S3 eventual consistency is to reduce the number of output partitions/files using coalesce() or repartition() functions of Glue[7] or Spark before writing the data.

    Additionally, you can also configure the 'Number of retries' parameter. This parameter specifies the number of times that Glue should automatically restart the job if it fails. It can be set from 0 to 10.

    ============
    References:
    [1]Recommended settings for writing to object stores :- https://spark.apache.org/docs/2.3.0/cloud-integration.html#recommended-settings-for-writing-to-object-stores
    [2]Using the EMRFS S3-optimized Committer :- https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-spark-s3-optimized-committer.html
    [3]Improve Apache Spark write performance on Apache Parquet formats with the EMRFS S3-optimized committer :- https://aws.amazon.com/blogs/big-data/improve-apache-spark-write-performance-on-apache-parquet-formats-with-the-emrfs-s3-optimized-committer/
    [4]Special Parameters Used by AWS Glue - https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-glue-arguments.html
    [5]CreateJob - DefaultArguments :- https://docs.aws.amazon.com/glue/latest/webapi/API_CreateJob.html#Glue-CreateJob-request-DefaultArguments
    [6]StartJobRun - Arguments :- https://docs.aws.amazon.com/glue/latest/webapi/API_StartJobRun.html#Glue-StartJobRun-request-Arguments
    [7]DynamicFrame coalesce : https://docs.aws.amazon.com/glue/latest/dg/aws-glue-api-crawler-pyspark-extensions-dynamic-frame.html#aws-glue-api-crawler-pyspark-extensions-dynamic-frame-coalesce


9.  USE PARQUET FILES instead of csv files while writing the data as an output in the Glue ETL Job. ( attached the example on how to use parquet while writing a dataframe in Spark ) 

        After I analyzed the code, I did recommend you to use Parquet file format for writing the dataset as it will speed up the write process for glue etl job.

    ==============================================================
    Advantages/Why to Use Parquet file Format with Glue ETL Job ?
    ==============================================================

          Parquet files are splittable, since the blocks can be located after reading the footer and can then be processed in parallel.Also Parquet files don’t need sync markers since the block boundaries are stored in the footer metadata.This is possible because the metadata is written after all the blocks have been written, so the writer can retain the block boundary positions in memory until the file is closed.Some of the other Advantages : 

            1. Reduces IO operations.
            2. Fetches specific columns that you need to access.
            3. It consumes less space.
            4. Support type-specific encoding.
10. Using "Overwrite" option while writing the output of the Glue ETL job

      Based on our discussion w.r.t the use case that you had - you wanted to overwrite the data within Glue ETL job. I confirmed "Currently Glue DynamicFrame does not support "overwrite" mode to overwrite the existing target files. " . Glue DynamicFrame add new files to target s3 path with every job run and but you would like to overwrite the files/objects in s3 path.

      There are 2 workarounds which can be considered for this approach based on your use case :

              Option 1: boto3API 

                  You can consider to use boto3 APIs to delete the target files before starting Glue ETL job which writes to s3 location [1].

              Option 2:  (RECOMMENDED) :

                  Use native spark dataframe [2]. - I had attached the python script with the syntax for this . actual_code.py which demonstrates how you can use the overwrite mode.

11.   if using s3 optimized committer is not an option -> Writing your Parquet files using the 'glueparquet' format option instead of 'parquet'. This option will make use of the 'DirectParquetOutputCommitter' class which directly writes the output to the designated output path instead of writing to a temporary path, then moving. Please keep in mind that the 'glueparquet' option has some limitations[2], so I would recommend reading the provided documentation links [2] and doing thorough testing before implementing it. 

12. If the reading from s3 is slow => Check how many files do we have in the source i.e. in AWS s3 bucket .

For example - prod/image-tags/plugin/entity/v1/year=2020/month=04/day=19/ here , how many part files approx. ? 

If these are less than 50K please do not use s3 "recurse": True and "groupFiles": "inPartition"  as AWS glue uses these two param's internally and come up with the optimum number based on the number of partitions it determines while reading the data from AWS s3. Apache Spark v2.2 can manage approximately 650,000 files on the standard AWS Glue worker type. So as per my knowledge we do not require these params.


Check the number of Tasks which gets created because more the # of tasks created in spark stages , more the time it will take overall.

The number of tasks is determined by the number of partitions.For a source that is read from AWS s3 ( glueContext.create_dynamic_frame_from_options( ... ) for example ) the number of partitions is the number of splits generated by the input format. 

This can be determined by the below : 

container_1587694819939_0001_01_000001/stdout:2020-04-24 02:34:40,319 INFO  [dag-scheduler-event-loop] cluster.YarnClusterScheduler (Logging.scala:logInfo(54)) - Adding task set 1.0 with 48626 tasks

If we see the above log snippet we can determine this spark job created 48K tasks which are too many.We need to set the partition number to a smaller number if you want to reduce the number of tasks.And I also see Glue internally did set the spark.executor.cores=4 which means that each executor can run a maximum of four tasks at the same time.So if we think we have 48K + tasks for the entire job how much time it would take and swamp the executors.


container_1587694819939_0001_01_000001/stdout:2020-04-24 02:34:44,287 INFO  [dispatcher-event-loop-1] yarn.YarnAllocator (Logging.scala:logInfo(54)) - Driver requested a total number of 9 executor(s).

This impacts the amount of data Spark can cache, as well as the maximum sizes of the shuffle data structures used for grouping, aggregations, and joins.Hence if we have optimum number of partitions , it's better. 


=====================

Recommendation 

====================

1. Now the trick generally used is to dynamically set the number of partition based on the datasize. If you know the size of each row of your data you can estimate how many rows you want to keep per partition. Lets say its value is X.

    Then you can set the num_of_partitions to be

        dataframe.count / x 

        Another example: You can configure the # of partitions (splits) for the entire process as the second parameter to a job, e.g. for parallelize if we want 3 partitions:

        a = sc.parallelize(myCollection, 3)
        
        Spark will divide the work into relatively even sizes (*) . Large files will be broken down accordingly - you can see the actual size by:

          rdd.partitions.size
          
          So no you will not end up with single Worker chugging away for a long time on a single file.

          (*) If you have very small files then that may change this processing. But in any case large files will follow this pattern.

2. Apache Spark UI for AWS Glue jobs - You can also use AWS Glue’s support for Spark UI to inpect and scale your AWS Glue ETL job by visualizing the Directed Acyclic Graph (DAG) of Spark’s execution, and also monitor demanding stages, large shuffles, and inspect Spark SQL query plans. 

3. Also please enable glue job metrics and take a look at it while the job is running.As in we want to see for example - How many number of maximum needed executors vs the number of active executors over the course of job execution which will help us to determine if we need to bump up the number of DPU's as well.

Reference documentation:

[1] https://aws.amazon.com/premiumsupport/knowledge-center/glue-oom-java-heap-space-error/
[2] https://docs.aws.amazon.com/glue/latest/dg/monitor-profile-debug-oom-abnormalities.html
[3] https://aws.amazon.com/blogs/big-data/best-practices-to-scale-apache-spark-jobs-and-partition-data-with-aws-glue/
[4] https://docs.aws.amazon.com/glue/latest/dg/monitor-debug-capacity.html
[5] https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-glue-arguments.html
[6] https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-partitions.html#aws-glue-programming-etl-partitions-pushdowns
