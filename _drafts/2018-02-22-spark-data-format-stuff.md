---
layout: post
title: Notes about Spark data format
date: 2018-02-22
tag:
  - spark
  - big data
  - data engineering
---

Three types of data abstractions are provided in Spark API:
- RDDs
- Dataframes
- Datasets


The major questions that should be answered before choosing which data structure to use:
- Whether your dataset is (well-)structured, semi-structured, or unstructured?
- What kind of operations you need to perform?
- Which style of code you want to write?
- What format of data input and output?


Define Spark Context and Spark SQL Context.

```scala
val sparkConext = new SparkContext(conf)
val sqlConext = new SQLContext(sparkConext)
```

### Input
Using Spark Context to load data into RDDs:
```scala
val rddFromTextFile = sparkConext.textFile("")
val rddFromSeqFile =

```

Using Spark SQL Context to load data into Dataframes:
```scala
val dfFromTextFile = sqlConext.textFile("")
val dfFromSeqFile =

```

### Output




Conversion between these types


Since our data is structured, we



Reference
