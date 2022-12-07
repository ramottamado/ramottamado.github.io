---
layout: post
title: Custom Logging in PySpark
date: 2022-12-04 21:21 +0700
categories:
  - data engineering
  - tutorial
  - big data
tags:
  - spark
  - python
  - pyspark
keywords:
  - apache spark
  - pyspark
  - custom logging
description: Custom logging in PySpark, and disabling excessive logging.
excerpt: Are you frustrated by excessive logging in PySpark?
preview: /assets/images/spark.png
meta: Apache Spark logo
lastmod: 2022-12-07 10:32 +0700
---

Are you frustrated by excessive logging in PySpark? Do you feel those logs are not
useful in any way? Do you wish to log things your way when you encounter errors
in your data pipelines? Then hopefully this article will help you write your own
logs in PySpark.

## Disabling Excessive Logs

As PySpark runs the process in JVM, the logs are Log4j logs. This means we can
programmatically set the log level in our app. To do this, we will need to access
the underlying JVM gateway and then looks for the Log4j class. The code will look
like this:

```python
from pyspark.sql import SparkSession

spark = (SparkSession.builder
    .appName("My Data Pipeline")
    .getOrCreate())

log4j = spark._jvm.org.apache.log4j # noqa
```

To disable Log4j logs for all classes with the prefix `org`
(like `org.apache.spark`)
we need to add this line of code:

```python
log4j.LogManager.getLogger("org").setLevel(log4j.Level.OFF)
```

This will tell the Log4j log manager to set the logging level of classes with the
`org` prefix to **OFF**. If you are not comfortable with turning off the logs completely,
you can use `Level.ERROR` or `Level.FATAL` instead.

## Creating Custom Log4j Logger

To create a custom Log4j logger, this code will do the trick:

```python
my_logger = log4j.LogManager.getLogger("my_logger")
my_logger.setLevel(...)
```

And to use it, every time you want to log something you just need to do this:

```python
# DEBUG logging
my_logger.debug("This type of logs should be enabled explicitly.")

# INFO logging
my_logger.info("Trying to write to database...")

# WARN logging
my_logger.info("Cannot find database 'boombox', will use 'default' instead.")

# ERROR logging
my_logger.error("This should not happen.")

# FATAL logging
my_logger.fatal("Your cluster has been hit by cosmic radiation.")
```

## Creating Custom Logger by Class Name

To create a custom logger for each class you have, you will need to make a custom
class like below[^1]:

```python
class LoggerProvider:
    """
    Custom Logger
    """

    def get_logger(self, log4j):
        return log4j.LogManager.getLogger(
            self.__full_name__()
        )

    def __full_name__(self):
        klass = self.__class__
        module = klass.__module__

        if module == "__builtin__":
            return klass.__name__ # avoid outputs like '__builtin__.str'

        return module + "." + klass.__name__
```

Then, your classes will need to extend the `CustomLogger` class like this:

```python
class MyPipeline(LoggerProvider):
    """
    Docstring
    """

    def __init__(self):
        log4j = spark._jvm.org.apache.log4j # noqa
        self.logger = self.get_logger(log4j)
        self.logger.setLevel(...)

    def run(self):
        self.logger.info("Running")
```

And that's how to create custom logging in PySpark.

[^1]: [Writing PySpark logs in Apache Spark and Databricks by Ivan Trusov](https://polarpersonal.medium.com/writing-pyspark-logs-in-apache-spark-and-databricks-8590c28d1d51)