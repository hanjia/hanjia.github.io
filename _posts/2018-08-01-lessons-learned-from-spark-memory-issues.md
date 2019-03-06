---
layout: post
title: Lessons Learned From Spark Memory Issues
date: 2018-08-01
tag:
  - spark
  - big data
  - data engineering
  - ETL
  - data processing
---

When we run a Spark application, it's quite common to see an out of memory error. Sometimes they are easy to resolve, while in others they can really be a headache. In this blog, I'm trying to put down some lessons learned from my daily development works. Those could be the things we should consider for fixing an OOM issue in the future.


Let's talk about fixes without code changes first. There are a couple of configs in Spark we can tweak. Note that it's not recommended to blindly increase these numbers without checking the implementation first. Also, you need to be clear about the capacity of your cluster and be cautious if the application is running in a shared cluster.



1. Memory-related Configuration

- spark.executor.memory
- spark.driver.memory
The maximum heap size to allocate to each executor/driver


- spark.yarn.executor.memoryOverhead
- spark.yarn.driver.memoryOverhead
The extra off-heap memory for each executor/driver

spark.executor.memory + spark.yarn.executor.memoryOverhead = Total memory that YARN can use to create a JVM process for a Spark executor


The above value is controlled by a YARN config:
yarn.nodemanager.resource.memory-mb



2. Physical Memory Limit


On some occasions, we will get an error from YARN suggesting that the application is running beyond the physical memory limits.

`Container [pid=47384,containerID=container_1447669815913_0002_02_000001] is running beyond physical memory limits. Current usage: 17.9 GB of 17.5 GB physical memory used; 18.7 GB of 36.8 GB virtual memory used. Killing container.`

This means probably we are processing multiple partitions on one single executor (VM)/ one host, and the total memory consumption exceeds the amount that server can afford. So the solution will be reducing the number of partitions
 for one executor by decreasing the following configuration.

spark.executor.cores

This config actually determines how many cpu cores that one executor can have. Since one cpu core will usually be responsible for one partition, so the number of cores will be equivalent to the number of partitions on the executor. For example, there are 4 cores per executor, we can try to set it to 2.



3. Parallelism


spark.default.parallelism VS. repartition()

spark.sql.shuffle.partitions



4. Cache


Cache() VS. Persist()



5. Join

spark.sql.autoBroadcastJoinThreshold




References
[1]: https://www.cloudera.com/documentation/enterprise/5-7-x/topics/cdh_ig_yarn_tuning.html
[2]: https://databricks.com/blog/2015/05/28/tuning-java-garbage-collection-for-spark-applications.html
