# IOStatistics

## Questions developers ask

### Hadoop connector dev

hadoop-aws, hadoop-azure: 
  how many GET requests does reading a file do?
  how long does HEAD take?
  what API calls/operations do applications do?
  
  
###  Downstream Dev

* How many GET requests were aborted when seeking round a parquet file?
* When writing data in a spark task, how much time is spent waiting for a thread to upload a block?

### Production:

Why was this Hive/Spark/Impala query slow?

* was it throttled?
* how many bytes were copied as part of renames?
* How much data was thrown away in seek calls?



## Today

### Hadoop connector dev

Some of the S3A and ABFS input streams have their own private API to get stats on that stream
FileSystem.Statistics for whole FS collection...single threaded tests can just query those

```java
    verifyMetrics(() ->
            execRename(sourceFile, destFile),
        whenRaw(RENAME_SINGLE_FILE_SAME_DIR),
        with(OBJECT_COPY_REQUESTS, 1),
        with(DIRECTORIES_CREATED, 0),
        with(OBJECT_DELETE_REQUESTS, DELETE_OBJECT_REQUEST),
        with(FAKE_DIRECTORIES_DELETED, 0));
```

### Downstream

```
LOG.debug(s"Stream stats $fsDataInputStream")
```

### Production

`FileSysystem. ThreadLocal<StatisticsData>`
S3A committers collect FileSystem.Statistics, massively over-counts in spark



# HADOOP-16830. Add public IOStatistics API


Interface for listing: counters, gauges, min/mean/max values of a single instance of an IO object

Interface for anything to act as a source of statistics, `IOStatisticsSource`
hadoop-common wrapper classes all implement `IOStatisticsSource`



### Hadoop connector dev

* Standard API for: logging, aggregation
* integration with AssertJ.assertThat
* All standard FS input/output streams propagate
* RemoteIterator wrap/chain with propagation of interface
* Lots of code to help in implementation

```java
S3ListResult result = trackDurationOfOperation(trackerFactory,
      "object_list_request", () -> 
          S3ListResult.v1(s3.listObjects(request.getV1())));
```

CS/functional programming tricks:

```java
  public static <A, B> FunctionRaisingIOE<A, B> trackFunctionDuration(
      @Nullable DurationTrackerFactory factory,
      String statistic,
      FunctionRaisingIOE<A, B> inputFn);
```

In tests: assertJ assertions. 

```java
private void assertListCount(
  final LocatedFileStatusFetcher fetcher,
  final int expectedListCount) {
    IOStatistics iostats = extractStatistics(fetcher);
    LOG.info("Statistics of fetcher: {}", iostats);
    assertThatStatisticCounter(iostats,
        "object_list_request")
        .describedAs("stats of %s", iostats)
        .isEqualTo(expectedListCount);
  }
```

### Downstream

```
LOG.debug(s"Stream stats $fsDataInputStream")
```

Collect, aggregate, log

IOStatisticsSnapshot is JSON- and Java- serializable.


### Production

Next stage: thread context collection/aggregation

For now: log

```
LOG.debug(s"Stream stats $fsDataInputStream")
val remote = fs.listStatusIterator(dir)
while(remote.hasNext) {
  process(remote.next)
}
LOG.debug(s"List stats $remote")
```

If you can build against API: Explore collection from any class implementing IOStatisticsSource

## TODO `IOStatisticContext` 

```java
class IOStatisticContext {

  // return thread local
  public IOStatisticsAggregator getContext();
    
}
```
+ somehow propagate into executors, e.g. 

```java
  public static <B> CallableRaisingIOE<B> callInCurrentContext(
      FunctionRaisingIOE<A, B> inputFn) {
      IOStatisticsAggregator ctx = IOStatisticContext.getContext();
      return () -> {
        IOStatisticContext.push(ctx)
        try {
            return ctx.apply();
        } finally {
        IOStatisticContext.pop(ctx)
     };
  }

submitter(callInCurrentContext(() -> deleteFiles(path)))
```

# Call to Action

1. Review my PRs!
2. Instrument your own IO classes -using common statistics names.
3. Especially collecting durations of expensive or unreliable ops. 
4. Log streams and iterators @ debug
5. Try to use the API
6. Expecially aggregation, save/report
6. Help define what an `IOStatisticContext` would be.
   

  