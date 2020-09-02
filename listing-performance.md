<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  
   http://www.apache.org/licenses/LICENSE-2.0
  
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

# Maximing Hadoop API List performance (especially on object stores)


## Summary

* Embrace the incremental list operations for better scale and support of paged downloads (object storage), adopting 

* Try to do useful work in each iteration, to make the most of of clients which do asynchronous fetch
  of the next page of results.
* Be prepared for listing failures (AccessDenied, FileNotFound) to surface in that first `hasNext()`/`next()` call;
  as asynchronous IO may delay the problem being observed in the `list` operation.
* `RemoteIterator<T>` implementations which implement `Closeable` MUST be closed.
   That is as important for local `file://` filesystems as object stores.
* `RemoteIterator<T>` implementations which implement statistics collection can be queried for them
   on builds with the HADOOP-16830 API, *and* will print them in their `toString()` calls.
   Consider logging at debug to see those values.

## Avoid 
 * Discarding information and asking the FS for it later in `getFileStatus()` calls
 * `globStatus()`
 * Probing for directories existing (`getFileStatus()`, `exists()`, `isDirectory()`) 
  before executing the list call. Do the list, and expect failures to surface in the list call.
 
## Embrace

Use in order of preference

* `listFiles(path, recursive = true)`  -deep listing, avoids any simulation of directories.
   S3A: `O(files)` with async page fetch irrespective of directory structure;
   elsewhere incremental treewalk (which could do async work in future)

* `listFiles(path, recursive = false)` -shallow listing, potentially efficient.
* `listStatusIterator(path)` - a direct replacement for `FileSystem.listStatus(): FileStatus[]`

And for when you want store location: `listLocatedStatus()`.

Mapreduce `LocatedFileStatusFetcher` can retrieve files in parallel to build its list up (and will provide IOStatistics too). 
(`org.apache.hadoop.mapred.LocatedFileStatusFetcher`)
It is/will be tagged as @Public/@Unstable, because applications/librareies turn out to have been using it (hello Parquet!). 
It is/will be an `IOStatisticsSource` too, providing
stats on the list operations, though sadly not the `globStatus()` phase.

## How to Parallelise

How best to parallelise your own listing code, if you are doing it to treewalk

* Use an incremental list operation.
* Have a (limited) set of worker threads doing their own processing/scanning,
  reading from a blocking queue and feeding the results into another queue
* Feed the listing entries into the queue of child directories to scan. 
* Provide another `RemoteIterator<>` to access the results, reading from the result queue.
* Which should implement `Closeable.close()` if you want to allow faster abort of sequence from the reader.
* Otherwise: Add a way to stop the scan process.

Application code to :

* Go through the remote iterator, being prepared for failures to not surface until the first
* Do something with each entry, the better to make use of asynchronous fetching.
* Consider measuring list performance. Take a look at `org.apache.hadoop.util.DurationInfo` if
you want something minimal.


## Making the best use of `RemoteIterator<>` listings

1. Consider that the read may block on the initial `list` call, the `hasNext()` or the `next()` call.
1. There's some ambiguity as to when `FileNotFoundException` and similar events may surface(*). It may be
in the actual evaluation where `FileNotFoundException` surfaces, because it was only in an
asynchronous listing that the operation failed. 

Some iterators may be `Closeable` for a faster/cleaner shutdown of background IO.
Consider checking to see if they do, Closeable, cast and close, 
e,g through `IOUtils.cleanupWithLogger(LOGGER, (Closeable) it);`.

This isn't just for the object stores; once we hook the iterable
`java.nio.Files.newDirectoryStream(path)` listing to `listStatusIterator()`,
we will need to close `DirectoryStream` at the end to avoid leaking references.
The FS APIs can do all the passthrough, but the callers will need to wrap up. 


## RemoteIterators and IOStatistics


Once HADOOP-16380 is in, some `RemoteIterators` willimplement `IOStatisticSource`,
and, if invoked, may return either `null` or an `IOStatistics` implementation. 

Specifically: 

* The S3A listing operations will report on the number of http list/list-continuation requests made,
  and the mix/mean/max values.
* The mapping/transforming wrapper iterators in `org.apache.hadoop.fs.functional` all
  implement `IOStatisticsSource` and will pass through the `getIOStatistics()` call.
  They will also forward `Closeable.close()`.

You can use `IOStatisticsLogging` to log these, or collect results into
an `IOStatisticsSnapshot` for aggregation and reporting. 
his snapshot is Serializable as java or JSON, so total statisics can be reported back. 

Without using the new API, consider logging the iterator's `.toString()` value at debug-level, if you
want to get performance statistics from them.


## Implementors notes 

### General

* Focus on `listStatus()` and `listLocatedStatus()`.

### Asynchronous listing

* Use a filesystem thread pool Rather than spin up threads of your own.
* Try to prefetch as much as you can.
* But be aware some code probes for a dir having children with a `listStatus(path).hasNext()` operation,
  so that initial scan may be something to make fast.
* Do not rely on `close()` being invoked.

### `IOStatistics`

* Collect statistics, report them through `IOStatistics` and `toString()`
* Especially for blocking/async IO operations against remote stores.
  Identifying listing-related latencies is invaluable for performance tuning query planning
* Wrap with `o.a.h.fs.functional.RemoteStatistics` wrappers,
  which pass through `IOStatistics` and `Closeable` APIs.
  You should be able to use these for most of your simple 1:1 map transformations
  &mdash; unless you need to do things with exception handling/transformation.

 