---
layout: post
title: Low-level Left Join in Apache Flink
date: '2022-02-11 14:13 +0700'
categories:
  - data engineering
tags:
  - flink
  - stream processing
  - java
keywords:
  - apache flink
  - left join
description: Using Apache Flink stateful stream processing to do left join between streams.
excerpt: 'To do low-level left join between streams in Apache Flink, we need to utilize one of the Flink''s process function classes, i.e., KeyedCoProcessFunction class.'
preview: /assets/images/flink-1.webp
redirect_from:
  - /low-level-left-join-in-apache-flink/
  - /low-level-streams-left-join-in-apache-flink/
lastmod: '2022-02-20 22:58 +0700'
---

## Background

To do low-level left joins between streams in Apache Flink, we need to utilize one of the process function classes, i.e., the `KeyedCoProcessFunction` class.<!--more--> The `KeyedCoProcessFunction` class gives us the ability to use [keyed state][flink-state]. Flink's keyed state will hold the state of the dimension stream, which is the stream we use to enrich the primary stream. In this implementation below, every time we pass a record into the primary stream, the `KeyedCoProcessFunction` will look into the state of the dimension stream, and then call a method to transform the primary stream with data from the dimension stream, if data with the same key exists. After that, the `KeyedCoProcessFunction` will output the transformed data.

## Implementation

### The `FactDimStreamJoin` class

A fairly basic example of an abstract class extending `KeyedCoProcessFunction` to do low-level left join will look like this:

```java
/**
 * The {@code FactDimStreamJoin} defines common method
 * to join fact and dimension streams.
 *
 * @param <KEY>  join key
 * @param <FACT> fact stream
 * @param <DIM>  dimension stream
 * @param <OUT>  joined stream
 */
public abstract class FactDimStreamJoin<KEY, FACT, DIM, OUT> extends
    KeyedCoProcessFunction<KEY, FACT, DIM, OUT> {
  private static final long serialVersionUID = 111L;

  protected final String stateName;

  protected final Class<DIM> dimClass;

  protected transient ValueState<DIM> dimState;

  /**
   * Constructs a new {@link FactDimStreamJoin} with
   * state name and class of the dimension stream.
   *
   * @param stateName name for Flink state for dimension stream
   * @param dimClass  class of the dimension stream
   */
  public FactDimStreamJoin(
      final String stateName,
      final Class<DIM> dimClass) {
    this.dimClass = dimClass;
    this.stateName = stateName;
  }

  /**
   * Joins fact and dimension streams with the same key.
   *
   * @param key  join key
   * @param fact fact stream
   * @param dim  dimension stream
   * @return joined the stream
   */
  public abstract OUT join(KEY key, FACT fact, DIM dim);
}
```

In this abstract class, we define `FactDimStreamJoin` extending `KeyedCoProcessFunction`. We define `stateName` and `dimClass` as Flink's state name (Flink uses state name as the identifier) and the class of the dimension stream. We also created the field `dimState`, which we will initialize in the `open()` method. This field will hold the state for the dimension stream, employing the `ValueState` type. We will update the `dimState` every time a new record with the same key arrives. The abstract method `join()` will define how we join the streams in the implementing class.

### The `open()` method

The `open()` method, to initialize the `ValueState`:

```java
@Override
public void open(final Configuration parameters) throws Exception {
  super.open(parameters);

  // Creating the state descriptor
  final ValueStateDescriptor<DIM> dimensionStateDescriptor = new ValueStateDescriptor<>(
      stateName, dimClass);

  // Initializing the ValueState
  dimState = getRuntimeContext().getState(dimensionStateDescriptor);
}
```

### What to do when the dimension stream arrived

When we process the dimension stream, we will update the `ValueState` every time a new record arrives. The implementation will look like this:

```java
@Override
public void processElement2(final DIM value, final Context ctx, final Collector<OUT> out)
    throws Exception {
  dimState.update(value); // Update the state with latest record for the key
}
```

### Transforming the Primary Stream

Last but not least, we need to implement the method to process the primary stream. In this process, we will call the `join()` method to join the incoming record with the record from the dimension stream that we currently have in the state store. It will look like this:

```java
@Override
public void processElement1(final FACT value, final Context ctx, final Collector<OUT> out)
    throws Exception {
  final KEY key = ctx.getCurrentKey();
  final DIM dim = dimState.value(); // Get the value of the dimension state for this key

  out.collect(join(key, value, dim)); // Join streams and output the result
}
```

## Conclusion

As the name suggests, to do low-level join is not that straightforward in Apache Flink. However, with low-level join, we can control how exactly the logic of the stream join is, as demonstrated with the above abstract class where we can fine-tune and change the logic of the `join()` method in the implementing class. Optionally, since we can have side outputs in Flink, we can add some logic to output records to side outputs, e.g., records without corresponding dimension states.

[flink-state]: https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/datastream/fault-tolerance/state/