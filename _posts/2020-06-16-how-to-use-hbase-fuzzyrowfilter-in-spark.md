---
layout: post
title: How to Use HBase FuzzyRowFilter in Spark
date: '2020-06-16 21:57 +0700'
keywords:
  - hbase
  - spark
  - fuzzyrowfilter
  - big data
  - data engineering
preview: /assets/images/hbase-logo.png
meta: Apache HBase logo
categories:
  - tutorial
  - data engineering
  - big data
tags:
  - hbase
  - spark
  - scala
redirect_from:
  - /how-to-use-hbase-fuzzyrowfilter-in-spark/
lastmod: '2022-02-13 00:11 +0700'
---
I've been picking up some skills regarding big data engineering, and in this post, I want to share how to use Apache HBase **FuzzyRowFilter** in Apache Spark with Scala.<!--more--> Apache HBase is the Apache Hadoop database, a distributed, scalable, big data store.[^1] Apache Spark is a unified analytics engine for large-scale data processing.[^2]

## Background

Usually, the best use case of Apache HBase is when you need random, realtime read/write access to your Big Data. Since Apache HBase is modeled after Google's BigTable, HBase provides BigTable-like capabilities on top of Hadoop and HDFS.[^1] Apache HBase is a type of NoSQL database,[^3] and also is a type of OLTP database. Apache HBase saves the data in a key-value pair and saves them based on the key (in HBase term, _row key_) lexicographically. This means the row key design plays a super important part in everything, since in Apache HBase if you want to access some rows, you need the full row keys, or at least the prefix of the start and stop row keys for fast fetching the rows you need. If you don't know the row key design, you'll need to do a full table scan which is not efficient at all![^4]

This then brings us to a problem, what if we only need rows where we only know the design of the middle of the row keys. For example, how do we filter for rows with the timestamp of May 20th, 2020 and June 10th, 2020 if the row key design is `USERID_TIMESTAMP_HASH` and process them in Apache Spark? The one possible solution is **FuzzyRowFilter**! **FuzzyRowFilter** is a built-in filter in Apache HBase that will perform _fast-forwarding_ on the table scan based on the fuzzy row key mask provided by the user.[^4]

FuzzyRowFilter takes one parameter: a list of pairs of the **fuzzy row key patterns** and the corresponding mask info. The type of the pairs is `org.apache.hadoop.hbase.util.Pair` while the patterns and the mask info are in the type of `Array[Byte]`.

## Implementation

Let's try how to implement this in Apache Spark with Scala. First, we will create a **case class** containing the fuzzy row key pattern and the corresponding mask info.

```scala
case class FuzzyData(rowKeyPattern: String, maskInfo: String)
```

Suppose the row key design is `USERID_TIMESTAMP_HASH` with 4 characters of `USERID`, `TIMESTAMP` with the format of `YYYYMMDD`, and `HASH` containing the hash of the data. We can create the fuzzy row key pattern for rows with the timestamp of May 20th, 2020 as follows:

```scala
val fuzzyData1: FuzzyData = FuzzyData(
  rowKeyPattern = "????_20200520_",
  maskInfo =
    "\\x01\\x01\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00"
)
```

As we can see, we can omit the hash part of the row key pattern, this is because FuzzyRowFilter takes partial row key pattern as well as full row key pattern. For the mask info, we set bytes of _ones_ (`x01`) on the fuzzy positions and bytes of _zeroes_ (`x00`) on the fixed position (in this case the timestamp). Using the pattern above, we can create another fuzzy row key pattern for rows with the timestamp of June 10th, 2020 as follows:

```scala
val fuzzyData2: FuzzyData = FuzzyData(
  rowKeyPattern = "????_20200610_",
  maskInfo =
    "\\x01\\x01\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00"
)
```

After that, we create an array of the `FuzzyData` case class for use in the filter:

```scala
val fuzzyRows: Seq[FuzzyData] = Seq(fuzzyData1, fuzzyData2)
```

Now we create the function to create the filter for `HBaseConfiguration`.

```scala
import java.util.Base64

import org.apache.hadoop.hbase.client.Scan
import org.apache.hadoop.hbase.filter.FuzzyRowFilter
import org.apache.hadoop.hbase.protobuf.ProtobufUtil
import org.apache.hadoop.hbase.util.{ Bytes, Pair }

import scala.collection.JavaConverters._

def convertScanToString(scan: Scan): String = {
  val proto = ProtobufUtil.toScan(scan)

  Base64.getEncoder.encodeToString(proto.toByteArray)
}

def filterByFuzzy(fuzzyRows: Seq[FuzzyData]) = {
  val fuzzyRowsPair = {
    fuzzyRows map { fuzzyData =>
      new Pair(
        Bytes.toBytesBinary(fuzzyData.rowKeyPattern),
        Bytes.toBytesBinary(fuzzyData.maskInfo)
      )
    }
  }.asJava

  val fuzzyFilter = new FuzzyRowFilter(fuzzyRowsPair)
  val scan        = new Scan()

  scan.setFilter(fuzzyFilter)
  convertScanToString(scan)
}
```

And then we can use it as below:

```scala
import org.apache.hadoop.hbase.client.Result
import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.io.ImmutableBytesWritable
import org.apache.hadoop.hbase.mapreduce.TableInputFormat
import org.apache.spark.sql.SparkSession
import org.apache.spark.rdd.RDD

val spark = SparkSession.builder.getOrCreate

@transient val exampleHConf = HBaseConfiguration.create()

exampleHConf.set(TableInputFormat.INPUT_TABLE, "table_name")
exampleHConf.set(TableInputFormat.SCAN, filterByFuzzy(fuzzyRows))

val hbaseRdd: RDD[(ImmutableBytesWritable, Result)] = spark
  .sparkContext
  .newAPIHadoopRDD(
    exampleHConf,
    classOf[TableInputFormat],
    classOf[ImmutableBytesWritable],
    classOf[Result]
  )
```

Then, to convert the resulting RDD to DataFrame (assuming the HBase table has only one column family):

```scala
import org.apache.hadoop.hbase.util.Bytes
import org.apache.spark.sql.Row
import org.apache.spark.sql.types.{ StringType, StructField, StructType }

val columnFamily: String = ...
val columns: Seq[String] = ...

val schema = StructType(
  columns map (column =>
    StructField(column, StringType, nullable = true)
  )
)

val newRdd = hbaseRdd map { case (rowKey, result) =>
  Row.fromSeq {
    columns map { column =>
      Option(
        Bytes.toString(result.getValue(columnFamily.getBytes, column.getBytes))
      ).getOrElse("")
    }
  }
}

val hbaseDf = spark.createDataFrame(newRdd, schema)
```

## Conclusion

Note that this method can only be applied if the `USERID` part has a fixed length. The efficiency and performance of the FuzzyRowFilter usually depend on the cardinality of the fuzzy parts. For example, if the cardinality of the `USERID` part is very high, the performance of FuzzyRowFilter will be almost if not the same as a full table scan, since Apache HBase cannot skip any rows while scanning the tables. In the end, it's a matter of how do you implement the row key design since this will affect how much performance you will get by using Apache HBase. Hope this article clears any of your confusion with HBase FuzzyRowFilter. Until next time!

---
References:

[^1]: [Apache HBase official site.](https://hbase.apache.org/)

[^2]: [Apache Spark official site.](https://spark.apache.org/)

[^3]: [The Apache HBase book.](https://hbase.apache.org/book.html#arch.overview)

[^4]: [Sematext article.](https://sematext.com/blog/consider-using-fuzzyrowfilter-when-in-need-for-secondary-indexes-in-hbase/)
