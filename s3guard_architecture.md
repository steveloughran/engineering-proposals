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

# S3Guard Architecture


# Introduction

> "Even before I graduated, I spent most of my development time trying to work
> around the limitations of AWS services". 
*
_Anonymous software developer, 2019_

This document tries to describe the architecture of S3Guard, as opposed to
how to use it.


## S3 Eventual Consistency

### Further Reading

## Renaming

The rename process is complicated.

## Table Internals


### Version Marker

There's a special item with the key `"../VERSION`, which contains a version
field.

| name | type | value |
|------|------|-------|
| `table_version` | int | table version |
| `table_created` | long | table creation in epoch milliseconds |


The `table_version` value is currently set to `100`. It will be changed
if and only if an incompatible table format is used 

When S3Guard is initialized, 

1. If the table version marker is missing: failure. This is not a S3Guard table.
1. If the version marker does nto contain a `table_version` column *and no tag*: failure.
1. If the `table_version` field is a different value: failure.

If at some point a new table version is required, it is expected that 
some migration mechanism will be supported.

Alongside the marker in the table, we now add a tag attribute to the table itself.
This is done when new tables are created. It is also done when opening a table which does not have one of these tags (if the client has the permission to do so)

When a table is opened which lacks the version marker entry, but it does have a version tag, then the version marker is recreated.

This addresses some issues which have surfaced in the field:
* rebuilding a table after somebody deletes everything from it.
* handling race conditions during table creation, where at the process creating a table is in a sleep/pool cycle awaiting the table's existence, and another process get access to the instantiated table first. With a 30s sleep between polling built into the AWS SDK, this is not a rare event if processes are kicked off shortly after the `s3guard init` command was issued. 


### File Entry
### Directory Entry
### Tombstone


## How it is used


### `listFiles()`

### `listStatus()`

### `getFileStatus()`

### `open()`

### `delete()`


*Partial Delete Failures*. If an S3 bulk Delete request fails with only some
of the files being deleted, but others rejected (usual cause: permissions),
S3Guard needs to be updated with what happened.

This is done during failure handling of the request. The exception raised by
the AWS service lists all files not deleted; we use that to calculate which
files have been deleted, and so should be removed from the table.

### `rename()`


## DynamoDB Metastore

There's an (interface, implementation) split with multiple metastores:

* Null
* Local
* DynamoDB

The local one is only for testing; the DynamoDB one is the only one
for production use.

It uses an AWS DynamoDB table for storing directory entries.

 Each item in the table
 represents a single directory or file.  Its path is split into separate table
 attributes:

1. parent (absolute path of the parent, with bucket name inserted as
 first path component). 
1. child (path of that specific child, relative to parent). 
1. optional boolean attribute tracking whether the path is a directory.
      Absence or a false value indicates the path is a file. 
1. optional long attribute revealing modification time of file.
      This attribute is meaningful only to file items.
1. optional long attribute revealing file length.
      This attribute is meaningful only to file items.
1. optional long attribute revealing block size of the file.
      This attribute is meaningful only to file items.
1. optional string attribute tracking the s3 eTag of the file.
      May be absent if the metadata was entered with a version of S3Guard
      before this was tracked.
      This attribute is meaningful only to file items.
 1. optional string attribute tracking the s3 versionId of the file.
      May be absent if the metadata was entered with a version of S3Guard
      before this was tracked.
      This attribute is meaningful only to file items.

The DynamoDB partition key is the parent, and the range key is the child.

To allow multiple buckets to share the same DynamoDB table, the bucket
name is treated as the root directory.

For example, assume the consistent store contains metadata representing this
file system structure:
 
```
 s3a://bucket/dir1
 |-- dir2
 |   |-- file1
 |   `-- file2
 `-- dir3
     |-- dir4
     |   `-- file3
     |-- dir5
     |   `-- file4
     `-- dir6
```

This is persisted to a single DynamoDB table as:

| parent                 | child | is_dir | mod_time | len | etag | ver_id |  is_authoritative | is_deleted | last_updated |
-------------------------|-------|--------|----------|-----|------|--------|-------------------|------------|--------------|
| /bucket                | dir1  | true   |          |     |      |        |                   |            |              |
| /bucket/dir1           | dir2  | true   |          |     |      |        |                   |            |              |
| /bucket/dir1           | dir3  | true   |          |     |      |        |                   |            |              |
| /bucket/dir1/dir2      | file1 |        |   100    | 111 | abc  |  mno   |                   |            |              |
| /bucket/dir1/dir2      | file2 |        |   200    | 222 | def  |  pqr   |                   |            |              |
| /bucket/dir1/dir3      | dir4  | true   |          |     |      |        |                   |            |              |
| /bucket/dir1/dir3      | dir5  | true   |          |     |      |        |                   |            |              |
| /bucket/dir1/dir3/dir4 | file3 |        |   300    | 333 | ghi  |  stu   |                   |            |              |
| /bucket/dir1/dir3/dir5 | file4 |        |   400    | 444 | jkl  |  vwx   |                   |            |              |
| /bucket/dir1/dir3      | dir6  | true   |          |     |      |        |                   |            |              |

This choice of schema is efficient for read access patterns.
`get(Path)` can be served from a single item lookup.
`listChildren(Path)` can be served from a query against all rows
matching the parent (the partition key) and the returned list is guaranteed
to be sorted by child (the range key).  Tracking whether or not a path is a
directory helps prevent unnecessary queries during traversal of an entire
sub-tree.


Some mutating operations, notably
`MetadataStore.deleteSubtree(}` and `MetadataStore.move()}`
are less efficient with this schema.
They require mutating multiple items in the DynamoDB table.



### Authoritative

Authoritative Mode is both powerful and a potential source of trouble.

When a store or path is declared as authoritative then

* directory listings do not fall back to S3 when the directory entry in the DDB table is marked "authoritative".
* file status probes (getFileStatus, open,...) do not fall back to S3. There is no check for Updated values, including changed file lengths or even  existence.

What does it do? It delivers significant performance improvements because we are only talking to dynamoDB. The listing improvements speed up directory scans especially recursive ones during hive/spark/impala query planning.

However, it has some key requirements:

* we need to be able to build up the directory tree from an initial set of imported files.
* all subsequent clients must interact with the store using the same dynamo DB table, again in authoritative mode.
*  the SCA client code has to be completely rigourous about keeping DDB in sync with operations made on S3. A lot of work was done in rename and delete operations to make sure the updates were incremental. As soon as a file is copied or deleted, we update the table.
* we need to be able to recover from the failure of the nonatomic S3 + DDB sequence. Hence, the fsck command.

This makes it the most advanced use of the tables; it also highlights the fundamental limitations of the process -and indeed where the directory tree-as-list-of-table-entries model is reaching the end of its life.

Applications and benefit from authoritative mode because it compensates for the performance limitations of S3 listing as well as the consistency. However, while S3 has the worst consistency model of any of the object stores, all the objects stores accessed over REST APIs are potentially slow to query for both LIST and HEAD calls. What was convenient and efficient in HDFS is not ultimately sustainable.

### Pruning


## Areas of Trouble

Here are the areas which proven "troublesome" during development.
These can often hints that areas are complex, incompletely understood,
beyond the skills of us developers or just a very quirky bit of S3.

### Testing

#### Throttling

We'd assumed that the DDB library from Amazon would handle throttling. It
did "a bit", but not much, and not well. It was only when doing testing
in EC2 with a bucket with capacity of 2 that it became clear that throttle
events could still be sent up. Cue work wrapping all calls with retry logic
(HADOOP-15426). Of course, pay-per-request obsoletes most of this, but
it's still there in case even that overloads.

#### Ordering of changes

All operations MUST work top-down on changes which add entries, bottom-up
on deletion, pruning etc. 


## Cost, Performance and Scale

### Performance

DDB isn't that performant, at least when you ask for consistent GETs, which
we do. Some of our operations (e.g. the Purge entry point) count the number
of DDB operations and print out the results in a per-operation call.

Reasons for this

1. DDB isn't instantaneous.
1. There's the overhead of an HTTP request for each non-batch call.
1. Writes are slow.
1. The S3Guard metastore client blocks for each operation to complete on 
a single HTTPS connection.

The blocking problem is being slowly addressed by doing more in the thread pool of the
owning filesystem, which will then make use of the pool of HTTP connections
maintained within the AWS SDK. That is: if you invoke DDB from two threads,
they use different HTTP connections.

There are more opportunities to improve here

* Do more incremental operations, especially in the batch delete call
(which we should be doing anyway).
* Probes for parent directories existing can be parallellised as we want to
know the results all the way up the tree.
* 




## Related Work

### Netflix S3mper

S3mper (Pronounced, "s3mper") gave us two things.

* Evidence that you can can use DDB for consistency.
* A pattern for name. S3mper -> S3guard.
 

### AWS EMRFS


If the EMR team had contributed their code back we'd have been very happy.
A lot of time and effort was put into the S3Guard code, and we assume/hope that
they've done the same. 

Sadly, they appear to have opted out of collaborating with the open source
codebase here. This isn't good for us: a few of us have to duplicate coding
and testing, and it's not clear how well it helps them. Yes, Consistent EMR
shipped ahead of S3Guard, but then the S3A "zero rename" committers shipped
a year before the EMR team came up with a reimplementation which we still
consider broken [cite]

Note: we've never set out to break EMRFS; it'd be
interesting to try based on on our knowledge of troublespots.



## Further Reading

* How long is eventually


## Future Work

If you look at where we are going, it's away from "everything is a filesystem"
towards "post-consistency" object-storage interaction.

S3mper, Consistent EMRFS and S3Guard all do their best to make
an eventually consistent object store look like a filesystem, as indeed do the
`s3://` and `s3a://` connectors.

There are limits to how long this metaphor is sustainable. Places where it fails
are

* the expectation that directory renames are _O(1)_ atomic operations.
* the expectation that directory deletes are _O(1)_ atomic operations.
* the expectation that listing directories is fast
* the expectation that files exist once you create them,
  rather than stay invisible until the write is completed.
* The expectation that once you call `flush()` on a stream it is persisted, so the
  store is a safe place to stream logs and other such files.
* The notion of directories with Posix-style permissions.
* The notion of files having Posix-style permissions, owners, etc.
* The belief that recursive treewalking is an efficient way to scan directories.


Object stores bring other things to the table

1. A file comes into existence "atomically"; there is never a partially-written file
visible.
2. That file instantiation can be executed on a different machine from where it
was created.
3. The ability to work with a deep directory tree without performing any treewalk. 

Inside the S3A connector, we know those new features exist and work with
them, especially in the S3A committers. They are not widely used outside,
partly because we haven't exposed the APIs completely, but also because
everyone still "codes for the Posix filesystem metaphor".

We can and should expose more of a store's capabilities to the caller, and
provide APIs for these features. However, if you look to the future, it
is formats like Apache Iceberg which embrace the object store most completely,
and move from "committing work by rename", beyond "commit by completing uploads"
into commit by writing a new index file in a single atomic PUT.


## History

### S3Guard: Creation to Production

S3Guard surfaced as a proposal from Cloudera. There'd been some discussion
about doing "something like this" in Hortonworks, but nobody had designed
anything --let alone started implementing any of it.

S3mper had shown that DynamoDB could be used to detect inconsistencies;
Amazon's Consistent EMR had taken this and made it a product. S3Guard
was needed to meet those same requirements: "make S3 Behave the way
applications expect it to".


The branch HADOOP-13345 was created on the Apache repository, for joint
development. At the same time, CDH 5.x had the original design prototyped;
the changes were used as the basis for the branch. As the ASF released,
CDH tracked the changes. This helped qualify the code, as integration tests
could find problems which the `hadoop-aws` tests had not.
(for example, that DynamoDB deletion was eventually consistent)

Apache Hadoop 2.9 and 3.0 both shipped with S3Guard; the branch-2 line has
had very minimal maintenance and has diverged simply by not keeping up with
ongoing work from Hadoop 3.1+.

HDP didn't ship with S3Guard until HDP-3.0, at the same time as the S3A Committers.
As with CDH, "Authoritative mode" was not documented other than to say
it existed and that wasn't ready for use.

## New Features

### Tombstones

### "Authoritative Mode"

### Out-of-band operations

### Etag and Version ID tracking: Hadoop 3.2

This was a big bit of work by our Cerner customers -hence the backport to CDH 5.x.

These provide significant improvements in the ability to detect and cope with updated files, because
the S3Guard table knows which file was created/discovered by a client.

The table has been expanded to store the `etag` of all objects, and, when deployed on
an S3 Store with object versioning enabled, the version ID.

* Knowing the etag allows us to spin when opening a file until we can get back data from the object
with that specific etag; timing out if it does not eventually appear. We also track the
etag during file reading, and if the etag changes when we seek round a file during a read (either due to
an overwritten file or an older version being retrieved), then again: we can retry and possibly recover.

* Knowing the version ID allows us to explicitly request that version of an object.
Even if it has been overwritten, provided the old version is still stored in the bucket, it will
be retrieved. We also ensure that only this version is read, even while we close and reopen a file during
seek operations.

These changes move the S3Guard table to being significantly more normative with respect to data listings;
we move beyond the list of files we believe to exist to tracking the exact version expected.

There's some complexity in building this information: when we write a file we get back the etag and version ID (but
not the timestamp of the file); when we list a directory we get the timestamp and etag, but not the
version ID. It'd be nice if the S3A would let us get the versions during a list, as it stops the
`s3guard import` command building up that data unless we were to extend it with a HEAD call on every
single file in the store.

_Note_: versioned repositories run up charges for old data, and tombstones collecting there can create
scale/performance problems. The extra `hadoop-aws` tests for versioned artifacts also slow down our test
runs. However, overall the benefits appear justified.  

### FSCK

### Metadata Time to Live

### Rename Performance and resilience

### Delegation Tokens

## Credit

In alphabetical order

TODO: check the spelling

```
Gabor Bota
Aaron Fabbri
Mingliang Liu
Steve Loughran
Sean Mackrory
Chris Nauroth
Andrew Olson
Ben Roling
```
