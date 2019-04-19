# 2019-03-15 Refactoring S3A

### Doc History: 
 
| date | revision | contents |
|------|----------|----------|
| 2019-03-15 | 0.0 alpha | initial draft |
| 2019-04-05 | 0.1 beta | published |
 
# Introduction

The `S3AFileSystem` is getting over complex and unmanageable. There: I said it. Bit by bit, patch by patch, its grown to a single 4000 line file with more methods than I want to look at, and becoming trouble to work with.

## Bits that work well

* The invoker with the once/retry lambda expressions, albeit at the expense of some of the stack traces you see on failure.
 It's been applicable to the low level store operations, to dynamo db calls and to eventual-consistency events when we can anticipate a mismatch between S3Guard and s3.

* The `@Retry` annotations help show what's going on too,
  though you need to be careful to review them, and as we delegate work to
  the S3A transfer manager, there are unmonitored retries going on where åwe can't see them.
  (We really need to do more with fault injection in the S3 client to explore the transfer
   manager's failure modes).

* The `S3AReadOpContext` to cover the context of a read has been able to evolve well and adapt to supporting S3 Select input streams alongside the classic one.

* `WriteOperationsHelper`. It's evolved, but has grown as the internal API for operations which write data into S3: the `S3ABlockOutputStream`, the committers, now the S3A Implementation of the Multipart Upload API.
    This shows the benefit of having a private API which works at the store level, yet is still wrapped by S3Guard, knows about encryption, what to do after you materialize a file,...

* Collecting statistics for different streams, merging them into the central statistics;
making visible through `InputStream.toString()` and using in tests to ensure the cost of operations
doesn't increase.

* The `DurationInfo` logging of the duration of potentially slow operations.
  Currently mostly for the new commit calls, but extensible with ease. (This is now in hadoop-common via HADOOP-16093)

* Submitting work into the `ExecutorService` thread pools (bounded and unbounded), e.g async block writes, and with `openFile()`, async file opening, 

* the inner-operation calls which don't do the final lossy translation of exceptions
into error codes and the like, especially for those operations (`rename()`) which
downgrade failures to a "return 0".

## Bits that don't work so well

Our re-referencing other top-level operations (`getFileStatus()` &c) was convenient, and nominally reduces cost, but it means the "it's a filesystem" metaphor is sustained deeper into the code than it needs to be.
And the cost of those little `getFileStatus()` calls (2x HEAD + 1x LIST) is steep.
Similarly, there are calls to `getObjectMetadata` &c which sneak in when you aren't paying attention. All too often the state of an object is lost and then
queried again immediately after; a side effect of us calling into our own exported methods.

Example: when we rename a directory we list the children, then copy(), which does its own HEAD on
each file. Better strategy: pass in the file status or core attributes (name, version, etag)
and skip the calls.

There's no explicit context to operations. There's some per-thread fields to count bytes read and written, but as they don't span threads, in a multi-threaded world, we can't collect the statistics on an operation, trace spans etc. This makes it near-impossible to collect adequate statistics on the cost of a query in any long-lived process (Spark, Hive LLAP….)

Backporting is getting harder, with a lot of the merge problems often being from incidental changes in the tail of the `S3AFileSystem` file, `S3AUtils`, etc.

Reviewing is getting harder. This increases the cost of getting changes in, maintenance etc. This is more a consequence of the increasing functionality: S3Guard, delegation tokens, S3 Select...

With reentrant calls, there's more duplication of parameter validation, path qualification etc.

Initialization is fairly fragile.
As the new extension classes (e.g. DelegationTokens) are passed in instances of S3AFileSystem during construction, they can call things before they are ready.
It means reordering code there is dangerous.

Applications which know the state of the FS before they start don't need all the overhead of the checks for parent paths, deleting fake parent dirs, etc.
We skip all that in the commit process, for example.

The classes are too big and complicated for people to safely contribute to.
That is: you need a lot of understanding of what happens, which operations incur
a cost of a S3 operation, what forms the public API vs private operations.

Not as much parallelization on batch operations as is possible, especially for the
slow-but-low-IO calls (COPY, LIST), S3Guard calls which may be throtted.

The primary means of sharing code across the other stores has tended to be
copy and paste. While yes, they are pretty different, there are some things
which you could consider as something worth standarizing

A key example would be a cache mechanism for storing contents of input stream in 1+MB blocks,
as ABFS and google GS do. We could do something like pull ABFS code up into common; add unit
tests there, wire it across the stores consistently, also with common stats about
cache miss, evict etc.

Java-8 lambda expressions and IOException-throwing operations. This is a fundamental
flaw in the Java language -checked exceptions aren't sustainable. It makes
lambda expressions significantly less usable than in Groovy and Scala, to name but two
other JVM languages which don't have this issue.

Testing: we're only testing at the FS level, not so much lower down except indirectly.
In particular, the way things are structured we're probably not exploring
all failure modes.

Reviewing maintaining `@Retry `attributes is all manual, and doesn't seem to be consistent.
For example the `org.apache.hadoop.fs.s3a.S3Guard.S3Guard` class isn't complete.
As its unique to this module, bringing in new developers adds homework to them.

Too much stuff accruing in `S3AUtils` and `S3ATestUtils`. As this is where
we place most of the static operations, they're both unstructured collections
of unreleated operations. While this isn't iself an issue, the fact that
most new features add new operations to the bottom of these files, 
it is a permanent source of merge problems. 

Specific issues with `S3AUtils` seem to be:

1. Two independent patches often conflict with each other at the base of this files,
leading to another (bad) habit: we add methods into the middle in the hope this
is less traumatic.

1. Cherrypicking patches back to previous versions also encounters merge problems,
because the merge is unable to find the right place to insert files.

1. We don't have much in the way of testing of this; certainly no real structure.

When you look at how those methods get used, they come into some explicit self contained groups

* credential setup
* configuration reading/writing/clearing (with per-bucket options)
* error translation
* Some java-8 lambda support
* low level S3 stuff

Most of these are different enough they could be split up into separate groups.
However, it is probably too late to do that without making backporting a nightmare.
*This does not mean we should make it worse*.
At the very least, new utils can be partitioned into their own class by function.


## Three layers: Two Model, one View

Move to a model/view world, but with two models: the hierarchial FS view presented by S3Guard + S3, and the lower S3 layer. 

The FileSystem API is the view.

To avoid the under layers becoming single large classes on their own, break up them by feature `MultiObjectDeleteSupport`, `CopySupport` which can be maintained & cherry picked independently.
Benefits: better isolation of functionality.
Weakness: what about if feature (a) needs feature (b)?
You've just lost that isolation and added more complexity.
Maybe we can do more at the `S3AStore` level, where features are given a ref to the `S3ABase` so can interact directly with that, along with the metastore



## Structure

### Package: `org.apache.hadoop.fs.s3a.impl`

Place new stuff in here so that there's no ambiguity about what's for external consumption vs private.
Leave existing stuff as is to avoid backport pain.

This was already been added in HADOOP-15625; new classes should go there.
If a java 9 module spec can be added we can completely isolate this from the outside. 

### `org.apache.hadoop.fs.s3a.impl.StoreContext`

Common variables which need be shared all down the stack.
These are mainly the services used by extension points today, when handed a
FileSystem instance (as Delegation Token support does, ).
Where the FS does need to be invoked, we use a `Function` type so that
any implementation of the function can be invoked

* the filesystem URI
* the instance of `S3AInstrumentation` for instrumentation.
* the executor pool.
* bucket location (`S3AFileSystem.getBucketLocation()`)
* the FileSystem instance's `Configuration`
* User agent to use in requests.
* User creating FS
* Metastore, if present.
* The functions to map from a key to a path and a path to a key for this filesystem.
* `Invoker` instances from the FS, which contain recovery policies
* `ChangeDetectionPolicy`
* Maybe: `Logger` of FS.
* Configuration flags extracted from the FS state: multiObjectDeleteEnabled, useListV2,///
* DirectoryAllocator and `createTmpFileForWrite()` method (Use: `S3ADataBlocks`)

Instrumentation, including:

* Reference to `S3AInstrumentation` (`S3AFileSystem.getInstrumentation()`)
* Reference to `S3AStorageStatistics`
* An `incrementStatistic(), incrementGauge(), decrementGauge()`

Operations which rely to the filesystem, or, during testing, other implementations
of the Functions. These will all be `Function` interfaces which can be configured
with lambda expression or FS methods.

* `metastoreOperationRetried()` (used in `DynamoDBMetadataStore`).
* Credential sharing `shareCredentials()` (used in `DynamoDBMetadataStore` + some tests).
* `operationRetried()`


### `org.apache.hadoop.fs.s3a.impl.S3ABase`

`S3ABase` talks directly to S3 -and is the only place where we do this.

Some operations would be at the S3 model layer, e.g. `getObjectMetadata`: the filesystem
view should not be preserved.


* The `com.amazonaws.services.s3.AmazonS3` instance created during FS initialization
  will be a field here.
* And it is passed in a reference to the executor threadpool for async operations
* parameters are generally simple keys rather than Paths
* new operations move to an async API, in anticipation of a move to async AWS SDK 2.0.
  But: need to look at that future SDK to make sure the work here is compatible.

What about moving to a grand Java-8 `completableFuture` world everywhere?
Risk of over-ambition with a programming model we haven't fully understood in practise.


### `org.apache.hadoop.fs.s3a.impl.S3AStore` + `StoreImpl`

This layer has a notion of a filesystem with directories & can assume: paths are qualified.

* Have an interface/impl split for ease of building a mock impl & so aid testing the view layer.
  (this is potentially really hard, and given Sean is doing a mock S3Client for HBoss, possibly 
  superflous. But if we start with a (private) interface, we can preserver it.
* Contains references to `S3ABase` and the `S3Guard` Metastore.
* Calls here have consistency. `WriteOperationsHelper` will generally invoke operations
at this level.
* If operations are invoked here which need access to specific files in `S3AFilesystem`,
rather than pass in a reference to the owner,
the specific fields are passed down as contructor parameters, `StoreContext` and, when appropriate,
method parameters, including an OperationContext. 


### `org.apache.hadoop.fs.s3a.S3AFileSystem`

The external view of the store: public APIs. The class which brings things together.

All the FS methods are here; as we move things into the layers below. Ideally it should
be fairly thin: it's the view, not not the model

## `S3AOpContext`: Context passed with operations.

`S3AOpContext` tuned to remove the destination field, and only have notion of: primary and S3Guard invoker; (future) trace context, statistics.
Maybe: factor out `TraceContext` which only includes trace info (not invokers), and is actually passed in to the Invoker operations for better tracing.

* add User-Agent field for per-request User-Agent settings, for tracing &c.

* Include FileStatus of pre-operation source and dest as optional fields.
If present, operations can use these rather than issue new requests for the data -and update as they do so.

* Credentials if per-request credentails are to be used

### `S3AWriteOpContext`: 

`S3AWriteOperationContext extends S3AOpContext`: equivalent of the `S3AReadOpContext`; tracks a write across threads. 


* Add `Configuration` map of options set in the `createFile()` builder;
  allows us to add the option to suppress creating parent paths after PUT of a file
  which is a bottlneck for some uses (flink checkpointing). 
* Because writes take place across threads, this needs to be thread safe.
* Update it with statistics and results after the operation, e.g. metadata from the
PUT/POST response, including versionID & etag.
* Add a callback for invocation on finalized write + failure.
* `StatisticsData` instance of the thread initating the write; this will be shared
across threads and avoid the current problem wherein reads and writes performed on
other threads are not counted within the IO load of the base thread, so omitted
from the MapReduce, Spark and Hive reports

## `org.apache.hadoop.fs.s3a.WriteOperationHelper`

We create exactly one of these right now; `S3AFileSystem.getWriteOperationHelper()` serves it up.
If we make it per-instance and add a `S3AWriteOperationContext` as a constructor parameter, then 
the context can be used without having to retrofit it as a new parameter everywhere.

### Partitioned Utility Classes

Splitting up S3A Utils should reduce the false positive merge problems, and provide a better conceptual model to work with and maintain.

#### `org.apache.hadoop.fs.s3a.impl.StoreFailures`

Static operations to help us integrate with the AWS SDK, e.g. type mapping,
taking a `MultiObjectDeleteException` and converting to a list of Paths.

Some of the error logic in `S3AUtils.translateException()` should really be moved into
an independent class, but as that is a common maintenance point, it needs to stay
where is —more specifically, the existing code does. New code does not.

#### `org.apache.hadoop.fs.s3a.impl.StoreConfiguration`

All the code from `S3AUtils` to deal with configuration.

#### `org.apache.hadoop.fs.s3a.impl.StoreOperations`

All the java-8 lambda level support which isn't added to hadoop-common for broader use.

## Testing 

## What works

* `AbstractS3ATestBase` as base class for most tests.
* Parallel test runs.
* `-Dscale` option.
* `InconsistentS3Client` allows for S3Guard inconsistencies to be generated in both ITests and in larger QE tests.
* Ability for contributors to test against private stores.
* "Declare your test endpoint" patch review policy; openness about tansient failures welcomed, but that test run is required. For every test patch.
* Options for testing encryption, roles, etc.
* IO tests on the landsat CSVs for fast setup and no local billing.
* Automated generation of unique test paths from method names.
* `ContractTestUtils` 
* Use of metrics to internally test behaviours of system, e.g. number of HEAD and DELETE calls made when creating files and directories.
* `LambdaTestUtils`, especially it's `intercept()` clone of ScalaTest

### What doesn't

* Option matrix too big, with local/DDB S3Guard, auth/non-auth, scale, plus bucket options (versioning, endpoint, encryption, role) All self-declared test runs are likely to only have of these options
* The test runs do take time when you are working locally, and you have to wait for them. Much of my life is spent waiting for runs to complete.
* Builds up very large sets of versioned tombstone markers unless your bucket rules aggressively delete them.
* No real tests of wide/deep stores as they take too long to set up.
* Getting `MiniMRCluster` to do real test run of delegation due to problems setting up Yarn in secure mode for DT collection; `AbstractDelegationIT` has to mix live code with mocking to validate behaviours.
* Dangerously easy to create DDB tables which continue to run up bills.
* Parameterized tests can create test paths with same name, so lead to some constistency issues.
* Mix of Test and ITest in same directory, mostly `org.apache.hadoop.fs.s3a` makes that a big and messy dir
* `S3ATestUtils` is another false-merge-confict file making backporting and patch merging hard.
* Lack of model/view separation makes it hard to test view without the real live model. Compare with ABFS whose store class is an interface implemented by the live and mock back ends.


### Proposed

* We identify public containers with large/deep directory trees and use them for scale input tests.
* Expansion of `InconsistentS3AClient` to simulate more failures.
* Support for on-demand DDB table creation and use of these in test runs.
* Fix up test path generation to always include unique value, such as timestamp.
* Better split of Test/ITest for new tests.
* Stop adding new helper methods to `S3ATestUtils`; instead partition by function `PublicDatasetTestUtils`, etc, all in `org.apache.hadoop.fs.s3a.test` package.
* Ability to declare on maven command line where the `auth-keys.xml` file lives, so make it easier to run tests with a clean source tree.
* Explore use of AssertJ and extend `ContractTestUtils` with new `assertThat` options for filesystems and paths, e.g.

        assertThat(filesystem, path).pathExists().isFile().hasLength(3)
* See if we can use someone else's mock S3 library to simulate the back end. There are things like [S3 Mock](https://github.com/adobe/S3Mock), which simulate the entire REST API (good) but mean you rely on them implementing it consistently with your expectations, and you still have something you can't deploy in unit test runs.
* Otherwise: New `S3ClientImpl` which simulates base S3 API, so allow for may ITests to run locally as Tests with the right -D option on the CLI (or default if you don't have cluster bindings?). Becomes a bigger engineering/maintenance issue the more you do things like actually implement object storage or validate the requests.
* Get rid of the local `LocalMetadataStore`. Some people use it for testing,
but with on-demand DDB there's no real justification for this. Removal simplifies
the test matrix and guarantess that S3Guard test runs always test against the real metastore.

And a process enchancement to consider:

* People who spend most of their day on this codebase set up a private jenkins/yetus build in an isolated docker container to test their own PRs, for a workflow of: local test run + fuller test of the whole option matrix.
* Provided the users set these containers up with transient restricted-role creds on a daily (or even per-run) basis, the ability actually run third party PRs without risk of key theft. (Note: if you run in EC2, hard/impossiblet to stop malicioius access of EC2 creds, so must deploy with restricted rights there too or not do external PR validation)/


## How to move to this?

-1 to any big refactoring festival. It'll only break things, take time, etc.
-1 to retaining code in S3AFS class which is duplicated in the new layers. After moving an operation: forward
-1 to a long-lived branch to do this with a big-bang switchover. 


+1 to baby steps and a one-by-one migration of operations.
+1 an initial sequence of patches to set things up, then evolution in place.

If things need to be backported, those initial steps can be cherry picked in too.


1. Create empty classes with constructors, instantiate in `S3AFileSystem` in order, but do nothing.
1. Add/extend existing `S3AOpContext` classes, and always pass them into the S3-layer calls (if need be, existing s3a ops which now forward to the new layers can just create a temp context)
1. `S3AFileSystem` to implement the core operations, passed in through `S3AStore`.
    Tests at this layer.
1. A new features are added to `S3AFileSystem` they move to the refactored classes. This drives implementation and use.
1. Some `S3AFileSystem` layer operations are pushed down, retaining the API entry points at the top for consistency. doing this one patch at a time enables the work to 
1. `WriteOperationsHelper` moves over to invoke `S3AStore`.
    If it needs to use `S3AFileSystem` calls, the layering isn't complete.
1. Recent features (S3A Delegation tokens) which take a back reference to `S3AFileSystem` are moved
to using the `BaseState` class for the core bits of service state it needs. (i.e these uses drive what
goes in). We can be backwards-incompatible for the post 3.2 features

Bug fixes of methods not moved into Core and Store shouldn't rush to move to the new structure,
as those can run all the way back to branch-2; it'd be too traumatic to do that here.

## Issues

Can we really do a clean separation of `S3ABase` operations from the FileSystem model?
The context which comes down is likely to take a `Path` reference at the very least; it's things like rename, mkdirs &c which we could try to keep away

How do we do this in a backport-friendly way? 

How do you do this gracefully and incrementally, yet still be confident
the final architecture is going to work?

1. Use this doc as model for writing new code and tests.
1. A quick, aggressive refactoring as a PoC, without worrying about
backporting problems, etc. This would be to say "it can be done". Assume 1+ days
work in the IDE. For anything bigger, make a collaborative dev on a branch.
This would purely be a "this is what we can do" prototype, with no plan to
retain. However, future work can pick up the structure of it as appropriate.
1. Create and evolve the `StoreContext` class, use as constructor PM for new modules in the .impl package
1. Test runner changes to go in in invidual patches (i.e. not with any other code changes)



