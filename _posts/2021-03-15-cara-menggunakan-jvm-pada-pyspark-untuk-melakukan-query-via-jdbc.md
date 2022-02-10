---
layout: post
title: Menggunakan PySpark untuk Melakukan Hive CTAS via JDBC
date: '2021-03-15 20:58 +0700'
preview: /assets/images/hive-logo.png
meta: Apache Hive logo
description: 'Dalam sebuah use case, saya mendapati ada beberapa table di Hive yang harus diproses menggunakan PySpark ternyata merupakan view.'
excerpt: 'Dalam sebuah use case, saya mendapati ada beberapa table di Hive yang harus diproses menggunakan PySpark ternyata merupakan view.'
lang: id-ID
categories:
  - data engineering
  - big data
  - tutorial
tags:
  - hive
  - spark
  - python
lastmod: '2022-02-09 13:38 +0700'
---

## Background

Dalam sebuah _use case_, saya mendapati ada beberapa _table_ di **Hive** yang harus diproses menggunakan **PySpark**
(**Hive** dan **PySpark** di sini berada dalam satu _cluster_ yang sama, sehingga harusnya _performance_ bukan merupakan
_issue_) ternyata merupakan _view_. Penggunaan _view_ ini menjadi masalah karena saya tidak memiliki akses _read_ pada
_table_ aslinya. Masalah ini muncul dikarenakan **PySpark** ternyata tidak mengakses _view_ yang ada di
**Hive Metastore**, melainkan langsung mengakses _files_ pada _source table_, padahal saya tidak mempunyai akses ke
_table_ tersebut. Umumnya, untuk mengatasi masalah seperti ini, yang akan dilakukan adalah menggunakan koneksi JDBC ke
**HiveServer2** atau melalui **Impala** JDBC, sehingga **PySpark** akan mengakses _view_ melalui koneksi JDBC tersebut.

Masalah lain muncul, yaitu dikarenakan satu dan lain hal, performa JDBC melalui **HiveServer2** terasa kurang baik. Nah,
tulisan berikut adalah sebuah _workaround_ untuk mengatasi _drop performance_ tersebut.

## High-level Flow

_Let's revisit how we access Hive tables through JDBC_. Umumnya, ketika menggunakan koneksi **Hive** JDBC pada
**Spark/PySpark**, kita menggunakan _method_ berikut:
```python
df = (spark
    .read
    .format("jdbc")
    .option(...)
    ...
    .load())
```
Dengan menggunakan _method_ tersebut, **Spark** akan melempar _query_ ke **HiveServer2** dan kemudian _fetch_ data
melalui JDBC _connection_ tersebut. Metode ini sangat tidak efisien untuk digunakan pada _table_ berukuran jumbo.
Apalagi, menggunakan koneksi JDBC untuk mengakses _table_ **Hive** yang berada pada satu _cluster_ sebenarnya tidak
direkomendasikan (JDBC _connection_ harusnya dipakai untuk _querying_ data antar _cluster_).

Seorang rekan kemudian menyarankan untuk melakukan CTAS (_create table as select_) menggunakan koneksi JDBC untuk
_dumping_ data dari _view_ menjadi _materialized temporary table_. Permasalah lain muncul: **PySpark** tidak punya
_native method_ untuk membuat _cursor_ dan _execute query_ ke JDBC (tentu saja masalah ini tidak akan muncul jika kita
menggunakan **Java** atau **Scala**).

Nah, solusinya adalah dengan menggunakan JVM dari _driver_ **Spark** _job_ yang kita jalankan. Seperti yang kita tahu,
meskipun API-nya menggunakan **Python**, **PySpark** tetap berjalan di atas JVM. Maka, kita dapat _invoke_ **Java**
_methods_ melalui JVM dari _driver_ **Spark** _job_ yang berjalan tersebut.

## Implementation

**PySpark** menggunakan _module_ **Py4J** untuk berkomunikasi dengan JVM. Untuk mengakses JVM pada **PySpark**, kita
akan menggunakan _**gateway**_ yang ada pada **SparkContext** dengan cara berikut (asumsi variabel `spark` adalah
**SparkSession** yang sedang berjalan):
```python
jvm = spark.sparkContext._gateway.jvm
```
Kemudian, untuk melakukan _query_ pada JVM, kita harus menggunakan `DriverManager` untuk membuat koneksi ke JDBC server,
membuat `statement` atau `preparedStatement`, dan kemudian melakukan `execute`. Menggunakan **Py4J**, kita dapat
melakukan _import_ `DriverManager` menggunakan _function_ `java_import` seperti berikut:
```python
from py4j.java_gateway import java_import
java_import(jvm, "java.sql.Drivermanager")
```
Kemudian selanjutnya untuk membuat _connection_ dan _execute query_:
```python
JDBC_URL = "jdbc:hive2://<jdbc_server>:<port>/<schema>"
VIEW_NAME = "<view_name>"
TEMP_TABLE = "<temp_table_name>"

con = jvm.DriverManager.getConnection(JDBC_URL)
stmt = con.createStatement()

query = """
    CREATE TABLE {temp} AS
        SELECT *
        FROM {view}
""".format(
    temp=TEMP_TABLE,
    view=VIEW_NAME
)

stmt.executeUpdate(query)
```
_Done!_ _Materialized temporary table_ sudah dapat diakses secara normal menggunakan **PySpark**.
```python
df = spark.table(TEMP_TABLE)
```

## Caveats and Potential Issue

1. Jika _jar_ JDBC _driver_ berada di HDFS, maka `sparkContext` tidak akan secara otomatis menambah _jar_ tersebut pada
_classpath_. _Workaround_ untuk _issue_ ini adalah dengan melakukan _query_ ringan terlebih dahulu melalui:
```python
_df = (spark
    .read
    .format("jdbc")
    .option("dbtable", "(SELECT * FROM <small_table> LIMIT 1) example")
    ...
    .load())
```

2. Untuk _kerberised cluster_, `JDBC_URL` harus disesuaikan dengan `AuthMethod` yang tersedia pada **HiveServer2**.
Cara untuk membuat koneksi ke _kerberised_ **Hive** _cluster_ akan dijelaskan pada kesempatan lainnya.
