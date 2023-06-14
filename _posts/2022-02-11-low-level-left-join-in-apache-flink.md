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
image: /assets/images/flink-1.webp
redirect_from:
  - /low-level-left-join-in-apache-flink/
  - /low-level-streams-left-join-in-apache-flink/
last_modified_at: '2023-06-14 21:41 +0700'
---

## Background

To do low-level left joins between streams in Apache Flink, we need to utilize one of the process function classes, i.e., the `KeyedCoProcessFunction` class.<!--more--> The `KeyedCoProcessFunction` class gives us the ability to use [keyed state][flink-state]. Flink's keyed state will hold the state of the right side stream, which is the stream we use to enrich the left side stream. In this implementation below, every time we pass a record into the left side stream, the `KeyedCoProcessFunction` will look into the state of the right side stream and then call a method to transform the left side stream with data from the right side stream, if data with the same key exists. After that, the `KeyedCoProcessFunction` will output the transformed data.

## Implementation

### The `LeftStreamJoin` Class

A fairly basic example of an abstract class extending `KeyedCoProcessFunction` to do low-level left join will look like this:

```java
/**
 * The {@code LeftStreamJoin} defines common method to left join two streams by common key.
 *
 * @param <KEY>   join key
 * @param <LEFT>  left side stream
 * @param <RIGHT> right side stream
 * @param <OUT>   joined stream
 */
public abstract class LeftStreamJoin<KEY, LEFT, RIGHT, OUT> extends
    KeyedCoProcessFunction<KEY, LEFT, RIGHT, OUT> {
  private static final long serialVersionUID = 1L;

  protected final String stateName;
  protected final Class<RIGHT> rightSideClass;

  protected transient ValueState<RIGHT> rightSideState;

  /**
   * Constructs a new {@link LeftStreamJoin} with state name and class of the right side stream.
   *
   * @param stateName      name for Flink state for the right side stream
   * @param rightSideClass right side stream class
   */
  public LeftStreamJoin(final String stateName, final Class<RIGHT> rightSideClass) {
    this.rightSideClass = rightSideClass;
    this.stateName = stateName;
  }

  /**
   * Joins left side and right side streams with the same key.
   *
   * @param key   join key
   * @param left  left side stream
   * @param right right side stream
   * @return the joined stream
   */
  public abstract OUT join(KEY key, LEFT left, RIGHT right);
}
```

In this abstract class, we define `LeftStreamJoin` extending `KeyedCoProcessFunction`. We define `stateName` as Flink's state name (Flink uses state name as the identifier) and `rightSideClass` as the class of the right side stream. We also created the transient field `rightSideState`, which we will initialize in the `open()` method. This field will hold the state for the right side stream, employing the Flink's `ValueState`. We will update the `rightSideState` every time a new record with the same key arrives. The abstract method `join()` will define how we join the streams in the implementing class.

### The `open()` Method

The `open()` method, to initialize the `ValueState`:

```java
@Override
public void open(final Configuration parameters) throws Exception {
  super.open(parameters);

  // Creating the state descriptor
  final ValueStateDescriptor<RIGHT> rightSideStateDescriptor = new ValueStateDescriptor<>(
      stateName, rightSideClass);

  // Initializing the ValueState
  rightSideState = getRuntimeContext().getState(rightSideStateDescriptor);
}
```

### What to Do When the Right Side Stream Arrives

When we process the right side stream, we will update the `ValueState` every time a new record arrives. The implementation will look like this:

```java
@Override
public void processElement2(final RIGHT value, final Context ctx, final Collector<OUT> out)
    throws Exception {
  rightSideState.update(value); // Update the state with latest record for the key
}
```

### Transforming the Left Side Stream

Last but not least, we need to implement the method to process the left side stream. In this process, we will call the `join()` method to join the incoming record with the record from the right side stream that we currently have in the state store. It will look like this:

```java
@Override
public void processElement1(final LEFT value, final Context ctx, final Collector<OUT> out)
    throws Exception {
  final KEY key = ctx.getCurrentKey();
  final RIGHT right = rightSideState.value(); // Get the value of the right side for this key

  out.collect(join(key, value, right)); // Join streams and output the result
}
```

## Conclusion

As the name suggests, to do low-level join is not that straightforward in Apache Flink. However, with low-level join, we can control how exactly the logic of the stream join is, as demonstrated with the above abstract class where we can fine-tune and change the logic of the `join()` method in the implementing class. Optionally, since we can have side outputs in Flink, we can add some logic to output records to side outputs, e.g., records without corresponding right side stream.

[flink-state]: https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/datastream/fault-tolerance/state/