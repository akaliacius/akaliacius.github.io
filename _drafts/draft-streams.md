---
layout: post
title:  "Java Streams, RxJava and CompletableFutures"
description: Find out what are similarities and differences between these concepts
date:   2022-01-25 16:58:28 +0000
categories: jekyll update
permalink: /:year/:title.jGeek/
tags: Java RxJava Stream
---

## Data

![Data fiction](/assets/img/data.jpg){:class="img-responsive"}

The good thing is that all 3 concepts offer us a **lazy evaluation** in contrast 
to classical eager iterables. The difference between them is the channels they offer to us.
Let's dive into each of them and discuss these channels 

### Stream

Streams have only **data chanel** which can be represented in three formats
* Zero data - `Stream.empty()`
* Single data - `Stream.of("single data")`
* Multiple data - `Stream.generate(() -> UUID.randomUUID()).limit(10)`

### RxJava

RxJava, on the other hand, has three channels to offer
* data channel
* error channel
* complete chanel

We will discuss data and complete channels for now. Error channel will be discussed in 
further chapters.
Data channel is same as Stream has
* Zero data - `Maybe.empty()`
* Single data - `Single.just("single data")`
* Multiple data - `Observable.fromCallable(() -> UUID.randomUUID()).take(10)`

### CompletableFuture

And the last that not the least CompletableFuture (CF), same as RxJava, has three channels 
* data chanel
* error channel
* complete chanel

Unfortunately, in opposite to RxJava and Stream, CF data channel offers fewer options
* Zero data - `CompletableFuture.<Void>supplyAsync(() -> null)`
* Single data - `CompletableFuture.supplyAsync(() -> "single data")`
  
The main CF's drawback is that it has no option for multiple data. To demonstrate why 
it's so important let's look at the example below. 
There is a data store, list of integers in this example.

`List<Integer> datastore = List.of(1,2,3,4,5)`
We want to perform few steps with this data. Here is pseudocode
```
1. Get a square of the number
2. Print the result to the console
```
Very simple, right? Let's use Stream API first
```java
datastore.stream()
      .map(i -> i * i)
      .forEach(System.out::println);
```
Nice and tidy! Now give a go to RxJava, shall we?
```java
Observable.fromIterable(datastore)
      .map(i -> i * i)
      .subscribe(System.out::println);
```
Pretty same as Stream. Easy and self declarative with functional flavour.
Now, as I said, there is no way to handle multiple data with CompletableFuture, thus 
I had to think of some workaround. The provided solution is just my way to solve this
particular problem. It is not necessarily correct one and there might be much better
ways. If you know, please share it in comments.
```java
List<CompletableFuture<Void>> tasks = new LinkedList<>();
datastore.forEach(integer ->
  tasks.add(CompletableFuture.supplyAsync(() -> integer)
    .thenApply(i -> i * i)
    .thenAccept(System.out::println)
));
CompletableFuture.allOf(tasks.toArray(new CompletableFuture[0]));
```
That is hideous, isn't it? We need to introduce additional collection to keep references
to all CompletableFutures, so we can combine them together. Also, we had to iterate our data store
in an imperative fashion.

## Error handling

![Error](/assets/img/error.png){:class="img-responsive" width="100%" height="300px"}


## Sync and Async vs Sequential and Parallel

### Stream

Java streams can be either sequential or parallel. Let's look at the simple example bellow. 
I create a stream of random UUID strings and perform few transformations using `map()` function. 
Streams are sequential by default, therefore 
```java
Stream.generate(() -> UUID.randomUUID().toString())
      .limit(3)
      .map(String::toLowerCase)
      .peek(uuid -> System.out.println(Thread.currentThread().getName() + " uuid:" + uuid))
      .map(String::length)
      .count();
```
\
In the following output we can see that all transformations happened on **main** thread.
```
main uuid:f5ef4383-76c4-473e-8a44-c4c02fa2ca72
main uuid:43a4eb4f-f97d-4252-ae85-d0eafc666eb3
main uuid:dd6c26a4-6f5a-43df-be7e-2f8f26685c09
```
\
Java Stream provides us with `parallel()` method to make our pipeline parallel. 
Next code snippet is exactly the same as previous with one extra line where parallel method 
has been introduced.
```java 
Stream.generate(() -> UUID.randomUUID().toString())
      .parallel()
      .limit(3)
      .map(String::toLowerCase)
      .peek(uuid -> System.out.println(Thread.currentThread().getName() + " uuid:" + uuid))
      .map(String::length)
      .count();
```
This time we can see, that all three transformations were performed on different threads. The first one on **main** thread
and two others threads used from ForkJoinPool thread pool introduced since Java 7. Stream API uses this thread pool to execute its 
tasks in parallel.
```
main uuid:ba2084be-ffda-4a14-bd51-cbbba4665842
ForkJoinPool.commonPool-worker-23 uuid:c3377f72-c8dd-444b-8c5b-9b52bd0ece99
ForkJoinPool.commonPool-worker-13 uuid:d95ca3f9-610e-4534-aa87-803b57dd5349
```
### RxJava


CompletableFuture 

Stream