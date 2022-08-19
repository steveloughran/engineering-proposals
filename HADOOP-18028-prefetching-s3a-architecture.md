# HADOOP-18028 prefetching s3a stream: architecture

## org.apache.hadoop.fs.common;

Broadly useful code; stuff we can put into hadoop-common, either in fs.impl or o.h.util (more of a committment there)

pool management

```java
BoundedResourcePool<T>
ResourcePool<T> "
```

### `BlockOperations`: queue of active operations

Core representation of block and any fetch in progress

Includes stat collection on System.nanoTime() of operations, but in `java.util.DoubleSummaryStatistics` format, not iostats. we should be able to map them
and so collect in job reports.

`BlockOperations.Kind` lists all operations which can be queued; there is also an End of each of these, which is placed on the queue when done. 

##  CachingBlockManager 

This is where all the caching is done.
* creates its own executor pool.
* turns off caching if things take too long. complicates debugging

## BlockCache/SingleFilePerBlockCache

map of block number to cached data; implementation uses files in the local FS with Path class mapping to java.io, not the hadoop one.

Default path for cached files is local FS temp dir.

**critical**
need to make sure that on a cluster job, this is under ENV_LOCAL
so when a container is destroyed all cached data is discarded.

simplest: s3a.buffer.dir

(side issue: oozie's problems w/ local file io; not an issue if java
IO is used directly)

## Retryer

used to handle sleep waits; updates status.
Lots of duplication with o.a.h.io.retry, but this is simpler and
not intended for network failover.

## use

```java
/**
 * Controls whether the prefetching input stream is enabled.
 */
public static final String PREFETCH_ENABLED_KEY = "fs.s3a.prefetch.enabled";

```
not sure we'd have it as default, not until we trusted it.


```java
/**
 * The size of a single prefetched block in number of bytes.
 */
public static final String PREFETCH_BLOCK_SIZE_KEY = "fs.s3a.prefetch.block.size";
public static final int PREFETCH_BLOCK_DEFAULT_SIZE = 8 * 1024 * 1024;
```

If the default values are used, each file opened for reading will consume
64 MB of heap space (8 blocks x 8 MB each).
if the size of a file is less than this, it is loaded into memory
for the duration of the stream.

```java

/**
 * Maximum number of blocks prefetched at any given time.
 */


public static final String PREFETCH_BLOCK_COUNT_KEY = "fs.s3a.prefetch.block.count";
public static final int PREFETCH_BLOCK_DEFAULT_COUNT = 8
    2;

```

* used in S3A FS to add size of bounded thread pool
* used in stream for # of active threads.
number of active prefetches a singe fs instance can perform.

fs.s3a.prefetch.block.count


