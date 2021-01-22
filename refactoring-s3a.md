# 2019-03-15 Refactoring S3A

### Doc History: 
 
| date | revision | contents |
|------|----------|----------|
| 2019-03-15 | 0.0 alpha | initial draft |
| 2019-04-05 | 0.1 beta | published |
| 2019-05-07 | 0.2 beta | updated from rename experience |
| 2019-07-23 | 0.2.1 beta | Operation Context |
| 2019-10-01 | 0.3.0 | Another revision |
| 2019-10-24 | 0.3.1 | Async initialize |
| 2020-02-19 | 0.3.2 | request factory (not yet merged in) |
| 2020-09-30 | 0.4.0 | directory markers |
| 2021-01-13 | 0.5.0 | Consistency; IOStatistics |

 
# Introduction

The `S3AFileSystem` is getting over complex and unmanageable. 

There: I said it. Bit by bit, patch by patch, it has grown to a single 4000 line file with more methods than I want to look at, and becoming trouble to work with.

This is slows down all development, because the "blast radius" of a change is
quite high - and it makes reviewing hard.

This document reviews the current codebase from the perspective of somebody who has written a lot of the code and the tests. My authorship of much of the code inevitably easy is my maintenance of the module -I know my way around, how things get invoked, and many of the design decisions. As a result, there may be overoptimism in my praise of the current system.

At the same time I get to the field support calls related to that code, and,
because of my ongoing work, suffer more than most people from the flaws in the current structure of the code.



## Bits that work well

* The invoker with the once/retry lambda expressions, albeit at the expense of some of the stack traces you see on failure.
 It's been applicable to the low level store operations, to dynamo db calls and to eventual-consistency events when we can anticipate a mismatch between S3Guard and s3.

* The `@Retry` annotations help show what's going on too,
  though you need to be careful to review them, and as we delegate work to
  the S3A transfer manager, there are unmonitored retries going on where åwe can't see them.
  (We really need to do more with fault injection in the S3 client to explore the transfer
   manager's failure modes).

* The `S3AReadOpContext` to cover the context of a read has been able to evolve well and adapt to supporting S3 Select input streams alongside the classic one.

* `WriteOperationsHelper`. It has evolved, but has grown as the internal API for operations which write data into S3: the `S3ABlockOutputStream`, the committers, now the S3A Implementation of the Multipart Upload API.
    This shows the benefit of having a private API which works at the store level, yet is still wrapped by S3Guard, knows about encryption, what to do after you materialize a file,...

* Collecting statistics for different streams, merging them into the central statistics;
making visible through `InputStream.toString()` and using in tests to ensure the cost of operations doesn't increase.

Those statistics are now being picked up by Impala to be used in their statistics and performance tuning.


* The `DurationInfo` logging of the duration of potentially slow operations.
  Currently mostly for the new commit calls, but extensible with ease. (This is now in hadoop-common via HADOOP-16093)

* Submitting work into the `ExecutorService` thread pools (bounded and unbounded), e.g async block writes, and with `openFile()`, async file opening, 

* The inner-operation calls which don't do the final lossy translation of exceptions
into error codes and the like, especially for those operations (`rename()`) which
downgrade failures to a "return 0".

## Bits that don't work so well

###  large monolithic classes.

The classes are too big and complicated for people to safely contribute to.
That is: you need a lot of understanding of what happens, which operations incur
a cost of a S3 operation, what forms the public API vs private operations.

Not as much parallelization on batch operations as is possible, especially for the
slow-but-low-IO calls (COPY, LIST), S3Guard calls which may be throtted.

### `initialize()`

Initialization is fairly fragile.

As the new extension classes (e.g. Delegation Tokens) are passed in instances of S3AFileSystem during construction, they can call things before they are ready.

It means reordering code there is dangerous.

It means it is slow and getting slower, because we are invoking  network operations. That is 

1. for  complicated delegation token and  authentication token implementations
-to connect to a remote credential service.
1. To initialise s3guard and hence a DynamoDB connection.
1. to probe for the store existing and failing "fast"
1. optionally to remove pending uploads
(side issue: should we cut that now that S3 lifecycle operations can do it?),


This means at least one HTTPS connection set up and used; often two to S3 and DDB, and possibly more. These are all set up and invoked in sequence.

This can actually cause problems in applications which are trying to instantiate the FS in multiple threads.

The `FileSystem.get()` operation looks for an FS instance serving that URI;
If there is none in the cache then it creates and initialises a new one.
Then a check for a cached entry is repeated -the new instance is added if there is still no entry; else it is closed and discarded.

`S3AFilesystem.initialize()` can take so long that many threads try to instantiate their own copy, all but one of which are then discarded.

We have tried in the past to do and asynchronous start-up where we do our checks for the bucket existing, etc in a separate thread.
This proved too complex to support because every method in the class which required a live connection would have to probe for and possibly block for the initialisation to have completed.

With a multilayer structure, we could push the asynchronous start-up
one layer down and gate those operations on the completed initialisation.

We should also be able to parallelise part of this process -the dynamoDB
and S3 bucket + purge operations in particular. As they may both depend
on delegation tokens, that must come first.

There is a price to this: failures are not reported until later, and we may have three separate failures to report. We should probably prioritise them in the order of: DT, s3guard, bucket.

Similarly, we can potentially parallelise shutdown, if there is cost there
(Side issue: `FileSystem.closeAllForUGI()`) closes all FS instances sequentially. That could be pushed out to a thread pool too.

### cross-invocation

Our re-referencing other top-level operations (`getFileStatus()` etc) was convenient, and nominally reduces cost, but it means the "it's a filesystem" metaphor is sustained deeper into the code than it needs to be.
And the cost of those little `getFileStatus()` calls (2x `HEAD` + 1x `LIST`) is steep.
Similarly, there are calls to `getObjectMetadata` &c which sneak in when you aren't paying attention. All too often the state of an object is lost and then
queried again immediately after; a side effect of us calling into our own exported methods.

Example: when we rename a directory we list the children, then `copy()`, which does its own HEAD on
each file. Better strategy: pass in the file status or core attributes (name, version, etag)
and skip the calls.

There's no explicit context to operations.
There's some per-thread fields to count bytes read and written, but as they don't span threads,
in a multi-threaded world, we can't collect the statistics on an operation, trace spans etc.
This makes it near-impossible to collect adequate statistics on the cost of a query in any long-lived process (Spark, Hive LLAP….)

Backporting is getting harder, with a lot of the merge problems often being from incidental changes in the tail of the `S3AFileSystem` file, `S3AUtils`, etc.
These are "False merge conflicts" in that there's no explicit conflict in the patches, just that because we usually append new methods/constants to the tail of files, they don't merge well.


Reviewing is getting harder.
This increases the cost of getting changes in, maintenance etc.
This is a consequence of the increasing functionality: S3Guard, delegation tokens, S3 Select, -and the sublties of S3 interaction,
which are incompletely understood by all, but slightly better understood by those how have fixed many bugs 

With reentrant calls, there's more duplication of parameter validation, path qualification etc.

### parental references in extension classes

We have added significant large modules in their own classes -S3 Select, S3Guard
and S3A Delegation Tokens being key examples. They do both get handed it down a reference to the owner, so that they used its methods, and can actually talk
to AWS services (S3, DDB and STS respectively).

This makes it harder for us to isolate these extensions for maintenance and testing. As we cannot control which methods they invoke, it is hard to differentiate external APIs from internal ones. 




### expensive checks to enforce fileystem path restrictions


Applications which know the state of the FS before they start don't need all the overhead of the checks for parent paths, deleting fake parent dirs, etc.
We skip all that in the commit process, for example.

We actually skip checking all the way up the parent path to make sure that no ancestor is file. Nobody has ever noticed this in production -presumably because no applications ever attempt to do this. We do checks during directory creation; for files the cost of such scans would be overwhelming -and we can get away without them.

Which raises a question: what else can we get away with? 

Maybe we should think about cacheing locally the path of the most recently created directory, so we can eliminate checks for that path when a programme is creating many files in the same directory.

###  limited code sharing across object store client implementations

The primary means of sharing code across the other stores has tended to be
copy and paste. While yes, they are pretty different, there are some things
which you could consider as something worth standarizing

A key example would be a cache mechanism for storing contents of input stream in 1+MB blocks,
as ABFS and google GS do. We could do something like pull ABFS code up into common; add unit
tests there, wire it across the stores consistently, also with common stats about
cache miss, evict etc.

### lambdas and futures crippled by checked exceptions

Java-8 lambda expressions and IOException-throwing operations. This is a fundamental flaw in the Java language -checked exceptions aren't sustainable. It makes lambda expressions significantly less usable than in Groovy and Scala, to name but two other JVM languages which don't have this issue.

We are going to have to live with this. What we can do is develop the helper classes we need to make this easier.

### testing

Testing: we're only testing at the FS API level, not so much lower down except indirectly.

The way things are structured this means that we're probably not exploring
all failure modes.

### @Retry tags

Reviewing maintaining `@Retry `attributes is all manual, and doesn't seem to be consistent.
For example the `o.a.h.fs.s3a.S3Guard.S3Guard` class isn't complete.
As its unique to this module, bringing in new developers adds homework to them.


### Utility classes generates needless cherry-picking conflicts

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

At the very least, new utils can be partitioned into their own class by purpose.


# Proposed: Three layers: Two Model, one View, "Operations"

Move to a model/view world, but with two models: the hierarchial FS view presented by S3Guard + S3, and the lower S3 layer. 

The FileSystem API is the view.

To avoid the under layers becoming single large classes on their own, break up them by feature `MultiObjectDeleteSupport`, `CopySupport` which can be maintained & cherry picked independently.

* Benefits: better isolation of functionality.
* Weakness: what about if feature (a) needs feature (b)?

You've just lost that isolation and added more complexity.
Maybe we can do more at the `S3AStore` level, where features are given a ref to the `StoreContext` so can interact directly with that, along with the metastore

### "Operations": short to medium life actions performed by the FileSystem

Some of the emulations of filesystem operations are very complex, especially those
working with directories including, inevitably, rename.

These operations can be be pulled out into their own "Operations" classes,
each of which is instantiated for the life of the filesystem-level operation.

* _Operations_ hide the lower level details of a sequence of actions within their
own class.
* _Operations_ must never be given a reference to the S3AFileSystem which owns them.
* _Operations_ are given a `StoreContext` in their constructor.

This is intended to pull out the complex workflows into their own isolated classes,
structured so that they can have isolated unit tests, where possible.

These Operations resemble the [_Command Pattern_](http://wiki.c2.com/?CommandPattern) of
Gamma et al; however there is no expected notion of an "Undo" operation.
Nor are they transactions. They are merely a way of isolating
filesystem-level pieces of work into their own classes and ensuring that their
interactions are solely with the layers below.

*Note*: without clean layering, the initial operations will end
up calling back into the S3A FS -but this can/should be done by providing
explicit callbacks, rather than a reference to the base class.


## Structure

### Package: `o.a.h.fs.s3a.impl`

Place new stuff in here so that there's no ambiguity about what's for external consumption vs private.
Leave existing stuff as is to avoid backport pain.

This was already been added in HADOOP-15625; new classes should go there.
If a java 9 module spec can be added we can completely isolate this from the outside. 

### `o.a.h.fs.s3a.impl.StoreContext`

Common variables which need be shared all down the stack.
These are mainly the services used by extension points today, when handed an
S3AFileSystem instance (as Delegation Token support does).

To access S3A FS operations, an interface `ContextAccessors` will provide
restricted access; this can be provided with fake implementations in tests.

The context will include

* The filesystem URI
* The instance of `S3AInstrumentation` for instrumentation.
* The executor pool.
* Bucket location (`S3AFileSystem.getBucketLocation()`)
* The FileSystem instance's `Configuration`
* User agent to use in requests.
* The User who created the filesystem instance.
* Metastore, if present.
* The functions to map from a key to a path and a path to a key for this filesystem.
* `Invoker` instances from the FS, which contain recovery policies
* `ChangeDetectionPolicy`
* Maybe: `Logger` of FS.
* Configuration flags extracted from the FS state: multiObjectDeleteEnabled, useListV2,...
* DirectoryAllocator and `createTmpFileForWrite()` method (Use: `S3ADataBlocks`)

Instrumentation, including:

* Reference to `S3AInstrumentation` (`S3AFileSystem.getInstrumentation()`)
* Reference to `S3AStorageStatistics`
* An `incrementStatistic(), incrementGauge(), decrementGauge()`

Operations which rely to the filesystem, or, during testing, other implementations
of the Functions. These will all be `Function` interfaces which can be configured
with -lambda expression or FS methods-. 
*update* No, via an interface `ContextAccessors` with an implementation in `S3AFileSystem`
plus others in test suites as appropriate.

* `metastoreOperationRetried()` (used in `DynamoDBMetadataStore`).
* Credential sharing `shareCredentials()` (used in `DynamoDBMetadataStore` + some tests).
* `operationRetried()`

To aid in execution of work, rather than just expose the thread pool,
an operation to create new `SemaphoredDelegatingExecutor` instances shall be
provided, one which throttles the amount of active work a single operation
can consume.

This is important for slow operations (bulk uploads), and for many small operations
(single dynamoDB uploads, POST requests, etc). Operations which use
completable futures to asynchronous work SHOULD be submitted through such an
executor, so that a single operation (bulk deletes of 1000+ files on DDB, etc)
do not starve all other threads trying to use the same S3A instance.

### `o.a.h.fs.s3a.impl.S3Access`

`S3Access` talks directly to S3 -and is the only place where we do this.

Some operations would be at the S3 model layer, e.g. `getObjectMetadata`: the filesystem
view should not be preserved.


* The `com.amazonaws.services.s3.AmazonS3` instance created during FS initialization
  will be a field here.
* And it is passed in a reference to the executor threadpool for async operations
* parameters are generally simple keys rather than Paths
* New operations MAY move to an async API, in anticipation of a move to async AWS SDK 2.0.
  But: need to look at that future SDK to make sure the work here is compatible.

Q. What about moving to a grand Java-8 `completableFuture` world everywhere?
A. Risk of over-ambition with a programming model we haven't used enough to really
know how best to use. Becaus this is intended to private, mmoving to an async
API can be done internally.


### interface `S3AStore` and  `S3AStoreImpl` implementation

This layer has a notion of a filesystem with directories and can assume: paths are qualified.

* Have an interface/impl split for ease of building a mock impl and so aid testing the view layer.
  (this is potentially really hard, and given Sean is doing a mock S3Client for HBoss, possibly 
  superflous. But if we start with a (private) interface, we can change that interface
  without worrying about breaking compatibility
* Contains references to `S3Access` and the `S3Guard` Metastore.
* Calls here have consistency. `WriteOperationsHelper` will generally invoke operations
at this level.
* If operations are invoked here which need access to specific files in `S3AFilesystem`,
rather than pass in a reference to the owner,
the specific fields are passed down as contructor parameters, `StoreContext` and, when appropriate,
method parameters, including an `OperationContext`. 


### `o.a.h.fs.s3a.S3AFileSystem`

The external view of the store: public APIs. The class which brings things together.

All the FS methods are here; as we move things into the layers below. Ideally it should
be fairly thin: it's the view, not not the model

## `S3AOpContext`: Context passed with operations (existing; to be extended).

`S3AOpContext` tuned to remove the destination field, and only have notion of: primary and S3Guard invoker; (future) trace context, statistics.
Maybe: factor out `TraceContext` which only includes trace info (not invokers), and is actually passed in to the Invoker operations for better tracing.

* add `User-Agent` field for per-request User-Agent settings, for tracing &c. (or even: a function to generate the UA for extra dynamicness)

* Include `FileStatus` of pre-operation source and dest as optional fields.
If present, operations can use these rather than issue new requests for the data -and update as they do so.

* Credential chains if per-request credentials are to be used.

* Optional Progressable for providing progress/liveness callbacks to during long operations.
This is important if a worker task is performing any O(data) operations yet still need to heartbeat
back to a manager process, especially during commit workflows. We can't retrofit this to
the current Hadooop API, but new builder-based operations should be given one.

* `Statistics`  of the thread initating the write; this will be shared
across threads and avoid the current problem wherein reads and writes performed on
other threads are not counted within the IO load of the base thread, so omitted
from the MapReduce, Spark and Hive reports

* `BulkOperationState`  - See below.


### `S3AWriteOperationContext`:  new end-to-end context

`S3AWriteOperationContext extends S3AOpContext`: equivalent of the `S3AReadOpContext`;
tracks a write across threads. 

To contain the state needed through a write all the way to the `finishedWrite()` method which updates S3Guard

This includes: mkdir, simple writes, multipart uploads and files instantiated when committing a job.

as/when a copy operation is added, it will also need to use it

Fields

```java
/** Is this write creating a directory marker. */
boolean isDirMarker;
```


* Add `Configuration` map of options set in the `createFile()` builder;
  allows us to add the option to suppress creating parent paths after PUT of a file
  which is a bottlneck for some uses (flink checkpointing). 
* Because writes take place across threads, this needs to be thread safe.
* Update it with statistics and results after the operation, e.g. metadata from the
PUT/POST response, including versionID & etag.
* Add a callback for invocation on finalized write + failure.

## `o.a.h.fs.s3a.WriteOperationHelper`

We create exactly one of these right now; `S3AFileSystem.getWriteOperationHelper()` serves it up.
If we make it per-instance and add a `S3AWriteOperationContext` as a constructor parameter, then 
the context can be used without having to retrofit it as a new parameter everywhere.

### Partitioned `S3AUtils` Utility methods

Splitting up `o.a.h.fs.s3a.S3AUtils` will reduce the merge problems,
and provide a better conceptual model to work with and maintain.

#### `o.a.h.fs.s3a.impl.AwsIntegration`

Static operations to help us integrate with the AWS SDK, e.g. type mapping,
taking a `MultiObjectDeleteException` and converting to a list of Paths.

Some of the error logic in `S3AUtils.translateException()` should really be moved into
an independent class, but as that is a common maintenance point, it needs to stay
where is —more specifically, the existing code does. New code does not.


#### `o.a.h.fs.s3a.impl.StoreSetup`

All the code from `S3AUtils` to deal with FS setup and configuration.
Initially: new methods.


#### `o.a.h.fs.s3a.impl.StoreOperations`

All the java-8 lambda level support which isn't added to hadoop-common for broader use.

#### `o.a.h.fs.s3a.auth.AuthenticationBinding`

Where we add (ultimately move) methods related to setting up AWS authentication.
Placed in the `o.a.h.fs.s3a.auth` package to be adjacent to the
classes it uses. 

## Testing 

### What works

* `AbstractS3ATestBase` as base class for most tests.
* Parallel test runs.
* `-Dscale` option.
* `InconsistentS3Client` allows for S3Guard inconsistencies to be generated in both ITests and in larger QE tests.
* Ability for contributors to test against private stores.
* "Declare your test endpoint" patch review policy; openness about tansient failures welcomed, but that test run is required. For every test patch.
* Options for testing encryption, roles, etc.
* IO tests on the landsat CSVs for fast setup and no local billing.
* Automated generation of unique test paths from method names.
* `ContractTestUtils`, mostly.
* Use of metrics to internally test behaviours of system, e.g. number of HEAD and DELETE calls made when creating files and directories.
* `LambdaTestUtils`, especially it's `intercept()` clone of ScalaTest.
* Our initial adoption of AspectJ

### What doesn't

* Option matrix too big, with local/DDB S3Guard, auth/non-auth, scale, plus bucket options (versioning, endpoint, encryption, role) All self-declared test runs are likely to only have of these options
* The test runs do take time when you are working locally, and you have to wait for them. Much of my life is spent waiting for runs to complete.
* Builds up very large sets of versioned tombstone markers unless your bucket rules aggressively delete them.
* No real tests of wide/deep stores as they take too long to set up.
* Getting `MiniMRCluster` to do real test run of delegation due to problems setting up Yarn in secure mode for DT collection; `AbstractDelegationIT` has to mix live code with mocking to validate behaviours.
* Dangerously easy to create DDB tables which continue to run up bills.
* Parameterized tests can create test paths with same name, so lead to some constistency issues.
* Mix of Test and ITest in same directory, mostly `o.a.h.fs.s3a` makes that a big and messy dir
* `S3ATestUtils` is another false-merge-confict file making backporting and patch merging hard.
* Lack of model/view separation makes it hard to test view without the real live model. Compare with ABFS whose store class is an interface implemented by the live and mock back ends.
* package-private use of S3AFileSystem methods makes test cases brittle to maintain -more of a surface area to change S3AFS

### Proposed

* We identify public containers with large/deep directory trees and use them for scale input tests.
Example: import the landsat tree.
* Expansion of `InconsistentS3AClient` to simulate more failures.
* Support for on-demand DDB table creation and use of these in test runs.
* Fix up test path generation to always include unique value, such as timestamp.
* Better split of Test/ITest for new tests.
* Stop adding new helper methods to `S3ATestUtils`; instead partition by function `PublicDatasetTestUtils`, etc, all in `o.a.h.fs.s3a.test` package.
* Ability to declare on maven command line where the `auth-keys.xml` file lives, so make it easier to run tests with a clean source tree.
* Expand use of AssertJ and extend `ContractTestUtils` with new `assertThat` options for filesystems and paths, e.g.

        assertThat(filesystem, path).pathExists().isFile().hasLength(3)
* See if we can use someone else's mock S3 library to simulate the back end. There are things like [S3 Mock](https://github.com/adobe/S3Mock), which simulate the entire REST API (good) but mean you rely on them implementing it consistently with your expectations, and you still have something you can't deploy in unit test runs.
* Otherwise: New `S3ClientImpl` which simulates base S3 API, so allow for may ITests to run locally as Tests with the right -D option on the CLI (or default if you don't have cluster bindings?). Becomes a bigger engineering/maintenance issue the more you do things like actually implement object storage or validate the requests.
* Restric ITest use of methods in S3AFileSystem to only those exported from some `S3ATestOperations` interface.
 This helps us manage the binding better.


And a process enchancement to consider:

* People who spend most of their day on this codebase set up a private jenkins/yetus build in an isolated docker container to test their own PRs, for a workflow of: local test run + fuller test of the whole option matrix.
* Provided the users set these containers up with transient restricted-role creds on a daily (or even per-run) basis,
  the ability actually run third party PRs without risk of key theft.
  (Note: if you run in EC2, hard/impossible to stop malicious access of EC2 creds, so must deploy with restricted rights there too or not do external PR validation)/


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

## Experience of HADOOP-15183 rename work

HADOOP-15183 started some of this work with a `StoreContext`, instantiable via
`S3AFileSystem.createStoreContext()`.

At the time of writing, with all this new code, the length of the `S3AFileSystem`
is now over 4200 lines, so it hasn't got any shorter. But: The new incremental
and parallelized rename code, along with handling of partial deletes was
all a fair amount of work. Apart from the context setup and operation
instantiation and invocation, this has all be done outside the core FS,
and is lined up for working with a relayered S3A filesystem.

### `StoreContext`

```java
public class StoreContext {

  URI fsURI;
  String bucket;
  Configuration configuration;
  String username;
  UserGroupInformation owner;
  ListeningExecutorService executor;
  int executorCapacity;
  Invoker invoker;
  S3AInstrumentation instrumentation;
  S3AStorageStatistics storageStatistics;
  S3AInputPolicy inputPolicy;
  ChangeDetectionPolicy changeDetectionPolicy;
  boolean multiObjectDeleteEnabled;
  boolean useListV1;
  MetadataStore metadataStore;
  ContextAccessors contextAccessors;
  ITtlTimeProvider timeProvider;
}
```

Initially some lamba expressions/Functions were used for operations (e.g. `getBucketLocation()`),
but this didn't scale. Instead a `ContextAccessors` interfacve was written to
offer those functions which operations may need. There's a non-static
implementation of this in `S3AFileSystem`, and another in the unit test
`TestPartialDeleteFailures`


```java
interface ContextAccessors {

  /**
   * Convert a key to a fully qualified path.
   * @param key input key
   * @return the fully qualified path including URI scheme and bucket name.
   */
  Path keyToPath(String key);

  /**
   * Turns a path (relative or otherwise) into an S3 key.
   *
   * @param path input path, may be relative to the working dir
   * @return a key excluding the leading "/", or, if it is the root path, ""
   */
  String pathToKey(Path path);

  /**
   * Create a temporary file.
   * @param prefix prefix for the temporary file
   * @param size the size of the file that is going to be written
   * @return a unique temporary file
   * @throws IOException IO problems
   */
  File createTempFile(String prefix, long size) throws IOException;

  /**
   * Get the region of a bucket. This may be via an S3 API call if not
   * already cached.
   * @return the region in which a bucket is located
   * @throws IOException on any failure.
   */
  @Retries.RetryTranslated
  String getBucketLocation() throws IOException;
}
  ```

*Note*: all new new stuff is going into `o.a.h.fs.s3a.impl` to make
clear it's not for public play. With a java9 module-info files we could make this
explicit. 


### `AbstractStoreOperation`

Base class for short/medium length operations:

```java
/**
 * Base class of operations in the store.
 * An operation is something which executes against the context to
 * perform a single function.
 * It is expected to have a limited lifespan.
 */
public abstract class AbstractStoreOperation {

  private final StoreContext storeContext;

  /**
   * constructor.
   * @param storeContext store context.
   */
  protected AbstractStoreOperation(final StoreContext storeContext) {
    this.storeContext = checkNotNull(storeContext);
  }

  /**
   * Get the store context.
   * @return the context.
   */
  public final StoreContext getStoreContext() {
    return storeContext;
  }

}

```

This was originally used for the rename trackers, but, at Gabor Bota's suggestion,
most of the `S3AFileSystem.rename()` operation was pulled out into one too.
Doing that added a need to call various operations in `S3AFileSystem`.
Rather than give up and say "here's your owner S3AFS", a new interface purely for
those rename callbacks was added, with an implementation class in S3AFS

```java

public interface RenameOperationCallbacks {

  S3ObjectAttributes createObjectAttributes(
      Path path,
      String eTag,
      String versionId,
      long len);

  S3ObjectAttributes createObjectAttributes(
      S3AFileStatus fileStatus);

  S3AReadOpContext createReadContext( FileStatus fileStatus);

  void finishRename(Path sourceRenamed, Path destCreated) throws IOException;

  @Retries.RetryMixed
  void deleteObjectAtPath(Path path, String key, boolean isFile)
      throws IOException;

  RemoteIterator<S3ALocatedFileStatus> listFilesAndEmptyDirectories(
      Path path) throws IOException;

  @Retries.RetryTranslated
  CopyResult copyFile(String srcKey,
      String destKey,
      S3ObjectAttributes srcAttributes,
      S3AReadOpContext readContext)
      throws IOException;

  @Retries.RetryMixed
  void removeKeys(
      List<DeleteObjectsRequest.KeyVersion> keysToDelete,
      boolean deleteFakeDir,
      List<Path> undeletedObjectsOnFailure)
      throws MultiObjectDeleteException, AmazonClientException,
      IOException;
}
```

It'd be an ugly mess if every factored-out operation had a similar
(interface, implementation) pair, but if you look at the operations,
none of these are directly at the Hadoop FS API level, more one level down
`copyFile()`, or at the S3 layer `removeKeys()`. Paths and keys get used
a bit interchangeably. 

As for `finishedRename()`, it triggers the work needed to maintain the 
filesystem metaphor

```java
public void finishRename(final Path sourceRenamed, final Path destCreated)
    throws IOException {
  Path destParent = destCreated.getParent();
  if (!sourceRenamed.getParent().equals(destParent)) {
    LOG.debug("source & dest parents are different; fix up dir markers");
    deleteUnnecessaryFakeDirectories(destParent);
    maybeCreateFakeParentDirectory(sourceRenamed);
  }
}
```

## `BulkOperationState` for the Metastores

To avoid massively amplifying the number of DDB PUT operations in the rename,
an abstract `BulkOperationState` class was implemented, which metastore implementations
can use to track their ongoing state. The default is `null`

```java
default BulkOperationState initiateBulkWrite(
    BulkOperationState.OperationType operation,
    Path dest) throws IOException {
  return null;
}
```

All mutating operations in the Metastore API were extended to take this
state. In the DynamoDB store, operations update this state as they write data,
recording which ancestor entries have been written. This avoids duplication
on single-file renames within a bulk rename, and single commits in a job commit.

For this to work, the state needs to be requested at the start of the bulk
operation and passed back at the end. 

For the rename process, this was manageable in the new code.

For the commit process, a lot of retrofitting was needed to pass this back
without tainting the S3A committers with knowledge of metastore state.
This was handled by making the existing `CommitOperations` class's commit/abort
operations private and providing a non-static inner class, `CommitContext`
which would be instantiaed with the `BulkOperationState` and then invoke
the `CommitOperation`'s commit/abort/revert methods, passing in that
`BulkOperationState` instance as appropriate;

`WriteOperationHelper` the low-level internal API to S3AFileSystem also needed
updating to take an optional `BulkOperationState` to wire this together.

As a result: there's a lot more state passing by way of new parameters.
 
The `CommitContext` class actually simplified a bit of the S3A committer code
though, so it is not all bad.

## Experience of HADOOP-16430 `DeleteOperation`

The work of the `RenameOperation` can be reused for delete.

And the `RenameOperation` callbacks into S3AFS expanded to become `OperationCallbacks`
to support both. This can grow in future. There's a risk of exposing too much,
but as it is structured as an interface we may be able to test better.

## Experience of HADOOP-13230 Directory Marker Retention

This is the first non- backwards compatible change we have ever deliberately
done to the code. It is not yet time to review the repercussions of that, other
than to note back porting is complicated.

_Good_

*   Having an isolated `RenameOperation` helped implementing one of the hardest bits
    of the change -not copying directory markers other than leaf markers.

*   Self contained tracker of those markers (`DirMarkerTracker`) for that change
    could be reused for the `MarkerTool` CLI entry point.

*   `s3guard bucket-info` command adding probes for marker support and state. By
    adding a new `-markers` option, all releases without marker awareness implicitly
    fail with a usage error.

*   As does a new path capabilities probe. (cloudstore's CLI support for this probe
    is handy ... should be made available in hadoop CLI)

*   Isolating the marker policy into a class
    `org.apache.hadoop.fs.s3a.impl.DirectoryPolicy` + lambda-expression callback
    allows for easy unit testing, and with a subset backported, helps with backport.
    This class also handles the `hasPathCapability(path, option)` probes, so keeps
    some complexity out of `S3AFileSystem`.

*   New declarative syntax for declaring the operation count of operations. Lambda
    expression to execute, metrics to always/conditionally evaluate, and an
    `OperationCost` class to contain cost of HEAD/LIST calls of an operation, with a
    `+` operation to easily combine mutiple operations to build the aggregate cost.
    Until now we'd been doing it with constants in the code, and that was
    unsustainable.

```java
verifyMetrics(() ->
        execRename(srcFilePath, destFilePath),
    whenRaw(RENAME_SINGLE_FILE_DIFFERENT_DIR),
    with(DIRECTORIES_CREATED, 0),
    with(DIRECTORIES_DELETED, 0),
    // keeping: only the core delete operation is issued.
    withWhenKeeping(OBJECT_DELETE_REQUESTS, DELETE_OBJECT_REQUEST),
    withWhenKeeping(FAKE_DIRECTORIES_DELETED, 0),
    // deleting: delete any fake marker above the destination.
    withWhenDeleting(OBJECT_DELETE_REQUESTS,
        DELETE_OBJECT_REQUEST + DELETE_MARKER_REQUEST),
    withWhenDeleting(FAKE_DIRECTORIES_DELETED,
        directoriesInPath(destDir)));
````

This is a nice design and fits in well with my notion of using instrumentation
as an observation point of distributed system testing.

_Bad_

*   Usual issues with long-lived branches: merge conflict, especially with ongoing
    list performance enhancements. The new declarative language for operation costs
    meant that the existing test suite (`ITestS3AFileOperationCost`) was almost
    completely rewritten, with inevitable conflict issues.

*   I inadventently added a requirement for the client to have
    `s3:deleteObjectVersion` permission in some cases. This only surfaced in QE
    testing of the next CDP release. Proposed: we explicitly define the minimum set
    of permissions we need, and use them in our assumed role tests. This will find
    changes immediately.

*   S3Guard compatibility issues have surfaced after the PR was merged -
    HADOOP-17244. I only discovered when doing other work on rename/2
    (HADOOP-11452)...a different test (inadvertently) caught the situation. I'm not
    going to backport the fixes there, so we cannot have older clients doing R/W IO
    on buckets with dir markers. Shows: better test coverage always helps.


_Ugly_

We now have an even more complicated test matrix with S3Guard and Marker
policies. There's nothing which can be done here except make sure we are doing
that broad test matrix before changes and on regular "what have we broken?"
runs. The more people running tests with different settings, the better.


## Troublespots

* We now have more classes to worry about. If every FS-level action is its own
`AbstractStoreOperation`, then there'll be one per API call.

* S3AFilesystem has added two new interfaces implementations as nested classes.
The first, the `ContextAccessors` for the store context is meant to be
general; it will evolve. As for any per-operation interfaces, a sign a
refactored design is correctly layered is if all operations only
call downwards to the levels below.

* Wiring up new ongoing state, such as `BulkOperationState`, went through a
lot of the code. This created merge conflict with onther ongoing work.

* None of this stuff is going to be easily backportable. From now on, the
first step to backporting subsequent changes in the HADOOP-15183
patch.

A more radical restructing of the codebase is going be a full "bridge-burning"
module rewrite. If we believe it is needed, then it has to be done.
We just all need to be aware of the consequences. 

We also need to avoid going "version 2.0" on the rework, taking this
as a chance to add new features because they've been on our wish list for
some time, or as placeholders for work which we don't immediately have
on our development plans. 

We should be able to take what we have and relayer it, while adding more notions
of context/state in the process.
If this is restricted to the existing `S3AFileSystem` and `WriteOperationHelper`,
without adding new features, we could have minimum-viable-refactoring lining
us up for future development.

## January 2021: Consistent S3

S3 in now Consistent. This means that all of S3Guard is superflous.

This is a significant change which simplifies the entire operation of the
S3A connector. 
As of now there is no need to turn S3Guard on.
It means we can also close S3Guard-related JIRAs as WONTFIX.
It means that at some point in the future we can even think about how
to remove it.

If we leave it there then it will slowly atrophy and stop working without
anybody noticing. Explicitly removing it is a one-way operation but
immediately simplifies the codebase and test policy.

Independent of that -what does it mean for the refactoring?

The proposed three layer model will need to be revisited. Could two-layergts work?

There's no need to pass down special S3Guard references into
  "lower-layer" code.
The handling of multipart delete failures is a key example. Most of
`org.apache.hadoop.fs.s3a.impl.MultiObjectDeleteSupport` is dedicated to
working out which paths to remove from the S3Guard table and which not.

The `BulkOperation` parameter is superflous wherever it is passed around.

Of the `hadoop s3guard` tool, only the non-s3guard related operations
(`markers`, `bucket-info` and `uploads`) are needed.

## January 2021: IOStatistics

The big IOStatistics Patch is in with:

* A public API for asking for statistics
* Support in S3A Filesystem, Streams, and the list RemoteIterators
* The final builder API makes construction of counters and gauges straightforward  
* The duration tracking class has been extended and integrated, making it
  simple to track the duration of network operations.

```java
  IOStatisticsStore st = iostatisticsStore()
      .withCounters(
          COMMITTER_BYTES_COMMITTED.getSymbol(),
          COMMITTER_BYTES_UPLOADED.getSymbol(),
          COMMITTER_COMMITS_CREATED.getSymbol(),
          COMMITTER_COMMITS_ABORTED.getSymbol(),
          COMMITTER_COMMITS_COMPLETED.getSymbol(),
          COMMITTER_COMMITS_FAILED.getSymbol(),
          COMMITTER_COMMITS_REVERTED.getSymbol(),
          COMMITTER_JOBS_FAILED.getSymbol(),
          COMMITTER_JOBS_SUCCEEDED.getSymbol(),
          COMMITTER_TASKS_FAILED.getSymbol(),
          COMMITTER_TASKS_SUCCEEDED.getSymbol())
      .withDurationTracking(
          COMMITTER_COMMIT_JOB.getSymbol(),
          COMMITTER_MATERIALIZE_FILE.getSymbol(),
          COMMITTER_STAGE_FILE_UPLOAD.getSymbol())
      .build();
```


The public API is split into the public interface and the private
implementation, except for `IOStatisticSnapshot` which has to be
part of the public API so it can be serialized.

The S3A side is very big as it moved to a full interface/impl split
for statistics collection. It collects a lot of information.

ABFS is now adding its support too, leaving only the issue of application
takeup.

This took almost a year of work on and off, with one person doing almost all the work.
Having a unified patch meant there was time to get that API coherent, but it
made for a bigger patch.

The lesson there is: a few people working together should be able to do this
fast, as they'd be reviewing/merging each others' work. 

*It would have been better off as medium-lived branch in
 the ASF hadoop repo, merged in as a series of changes with incremental reviews
 and which could be merged into hadoop trunk once happy.*

For application takeup we need a thread-local `IOStatisticsContext`. This would
exist for each hive/spark/whatever worker thread, but need passing in to
executor threads performing work on their behalf. 

### January 2021: Incremental Listing

All the list operations are optimised for incremental listing of directories.
They are now significantly worse when listing files or empty directories, but
for directories with one or more files, we have reduced the number of S3 API
calls by 3/4. With asynchronous fetching of the next page of results,
Applications which is the incremental APIs to list directories/directory trees
can compensate for list performance by doing anything they can per listing
entry.

Note: ABFS is starting to follow this same design.

We need to move applications onto these APIs, in hadoop's own codebase, but
especially in query planning in Hive and Spark.

This also means we need incremental versions of FileSystem.globStatus and and
LocatedStatusFetcher. The latter is now being used in Spark as it offers
parallelised tree scans. The globStatus call is one of those popular methods
where use of it is so widespread that there's a risk of any change proving
pathologically bad in some scenarios. This is why I abandoned (HADOOP-13371)[https://issues.apache.org/jira/browse/HADOOP-13371]
_S3A globber to use bulk listObject call over recursive directory scan_.


#### Proposed: Public APIs for Tree Scanning

* New glob/located status fetcher classes in hadoop-common (with backports
  somewhere) for executing incremental, parallelised scans.
* To take an interface for callbacks to the FS, for FileSystem, FileStatus and
  any tests we want.
* Use the incremental List calls to direct the worker threads and to feed data
  into the result iterator
* Incremental returning of results to caller through RemoteIterator
* Instrumented for IOStats collection
* multithreaded into a supplied or constructed executor
* builder API.
* Closeable; close() will stop the scan.

A key aspect of the design would be that there would be a RemoteIterator of
results which would be populated with new values as worker threads find them.
With the workers doing incremental listings, results could feed back as soon as
any thread had results. for any directory found, unless a deep treewalk was in
progress, the dir would be queued for its own scan.

If you look at the two scanners, they aren't significantly different. the
globber is doing pattern expansion; LocatedStatusFetcher is already
parallelised.

Troublespots:

* results will not arrive in filename order, but instead at random.
* time for next/hasNext to block is indeterminate. It may be good to provide a
  Progressable callback there which is regularly updated on 1+ worker thread.
  Maybe: whenever a new dir scan kicks off.
* Could we safely use deep tree walks? And if so: how best to do it?
## Next Steps


### Roll out `StoreContext`, especially in 3.3+ classes -*Done*

The new `StoreContext` class should be usable as the binding argument for
those subcomponents of the S3A connector which are currently passed
a direct `S3AFileSystem` reference.

* `WriteOperationHelper`. This will implicitly switch those classes which
use that as the low-level API for store operations to using the store context
* `S3ABlockOutputStream` (which also takes a `WriteOperationHelper`)
* `o.a.h.fs.s3a.Listing`
* `o.a.h.fs.s3a.commit.MagicCommitIntegration`
* `o.a.h.fs.s3a.S3ADataBlocks`
* `AbstractDTService` and `S3ADelegationTokens`. Issue: any implementation
of `AbstractDTService` is going to break here, which means any S3A
delegation token binding outside of the `hadoop-aws` module.
* the S3Guard Metastore implementations, especially `DynamoDBMetadataStore`.

These can be done one at a time; there's no obvious gain from doing it in bulk.
We could consider moving some of them (Listing; `S3ADataBlocks`) into the `.impl`
package at the same time, again, at a cost of backport pain.

Initial targed: new classes added in Hadoop 3.3. These have no backport cost within the ASF branches.


### Design an `OperationContext` for propagating through `innerXYZ` operations.

The S3Guard `BulkOperationState` class is becoming something more broadly passed around
(HADOOP-16430)[https://issues.apache.org/jira/browse/HADOOP-16430]. Rather than do this everywhere,
an `OperationContext` class should be defined which we can extend over time.

| Field | Type | Purpose |
|-------|------|---------|
| `bulkOperationState` | `BulkOperationState` | nullable ref to any ongoing S3Guard bulk operation |
| `userAgentSuffix` | `String` | optional suffix to user agent (request tracking) |
| `requestSigner` | callback | whatever signs requests |
| `trace` | openTrace ref | opentrace context |

The `bulkOperationState` field would not be final; it can be created on demand, after which it is used until closed.
_Issue_: need to make sure that once closed it is removed from the context; maybe make way to close one 
the `close()` operation in this context. Only a select few bulk operations (rename, delete, commit, purge, import, fsck) need
bulk operation state now; if their lifecycle is managed from within the `AbstractStoreOperation` implementing them, then
this becomes less complex. We just need a setter for the state which has the precondition: if the new state is non-null, the
current state MUST be null.

Per-request UA suffix, Auth and openTrace are all features some of us have been discussing informally.
There's no implementation of these features, which can all be added in the future. What is key is that
creating and passing round an `OperationContext` everywhere, rather than a `BulkOperationState`, isolates
code from the details of S3Guard, while giving us the ability to add these features without adding yet another parameter across
every single function, caller and test which needs it.

*Update*: This gets complicated fast.

## `RequestFactory` class to isolate request construction.

An initial PoC refactoring has shown that there is a modest amount of code in S3AFS class dedicated purely
to building request structures to pass down to the AWS client SDK, along with similar operations (e.g. adding
SSE-C encryption keys). These can all be pulled out to a self-containerd `RequestFactory` class.
This is a low-trauma housekeeping change which can applied ahead of any other work.

*there is no grand design which needs to be "got right" here, just a coalescing of related operations into their own
isolated class, one with no dependencies on any other part of an S3A FS instance*

## Issues

### Will layering work?

Can we really do a clean separation of Store-level operations from the FileSystem model?

The context which comes down is likely to take a `Path` reference at the very least;
it's things like rename, mkdirs etc which we could try to keep away.
(Update: the experience on `rename()` in HADOOP-15183 implies we very much need "something" beneath
the public FileSystem API yet which exports all the operations medium-life store
operations need, and taking that ongoing context.

### How do we do this in a backport-friendly way? 

How do you do this gracefully and incrementally, yet still be confident
the final architecture is going to work?

1. Use this doc as model for writing new code and tests.
1. A quick, aggressive refactoring as a PoC, without worrying about
backporting problems, etc. This would be to say "it can be done". Assume 1+ days
work in the IDE. For anything bigger, make a collaborative dev on a branch.
This would purely be a "this is what we can do" prototype, with no plan to
retain. However, future work can pick up the structure of it as appropriate.
1. Create and evolve the `StoreContext` class, use as constructor parameter for new modules in the .impl package
1. Test runner changes to go in in invidual patches (i.e. not with any other code changes)
1. As for "the final architecture" -we get to evolve it.

### How do you stop this becoming a vast over-the-top rework?

This is always the risk.

Splitting the `S3AFileSystem` up into two initial layers, with the underlying layer
providing the functions invoked by the `S3AFileSystem` class, by `WriteOperationsHelper`
and those needed by `RenameOperation` and any siblings added is the foundational step.

That would define the model/view split with effectively four views

* Hadoop `FileSystem` API. 
* and the private `WriteOperationsHelper`
*`RenameOperation.RenameOperationCallbacks` 
* `o.a.h.fs.s3a.impl.ContextAccessors`  

All of these MUST be able to interact with our store model.

Partitioning `S3AFileSystem` is the big first step. 
It's where an initial "this is what we should" do PoC could be written,
which we'd then review and see if it looked viable.

No other changes to functionality would be made other than passing an operation context
with every operation. For that we could add it in the API, even if they were not
initially being passed in. (or we just had some stub `createOperationContext()...`) call



* implement the lowest layer in front of the AWSClient and access over passing that client around
* add the WriteOpContext


# Layering Design

Split the S3A code into layers

* S3ADelegationTokens to remain as is; interface to be an option in future
* Add interfaces + impls for new classes
* S3AStore + Impl
* RawS3A + Impl
* Statistics will be moved to interfaces in HADOOP-16830, 
  _Add public IOStatistics API; S3A to collect and report across threads_

S3AFS will create the others and start in order: DelegationTokens, RawS3A, S3AStore, Metastore -Asynchronously.

This will involve wrapping all access of DTs, s3client, Metastore to block until that layer is complete, 
or raise an exception if instantiation of it/predecessor failed.

New layers will all be subclasses of Service, split into Interface and Impl, 
so we can manage the init/start/stop lifecycle with existing code or one or two shared utility operations.

New layers will be part of S3AContext, though the under layers will not get a context with the upper layers defined;
this avoids upcalls. 

* We may need extra callbacks, especially on RawS3A to call Metastore during failure handling*


## Partition operations to avoid re-replicating the same layers

Common Hadoop FS layer operations like open/openfiles and listfiles/liststatus/listLocatedStatus are very
much the FS API, but are complex in their own right. There's often > 1 Hadoop FS entry point with slightly different parameters.

Proposed: pull these out into their own classes, e.g

```
OpenFileApiImpl
ListApiImpl
StatusProbesImpl (getFileStatus, isDirectory, isFile, exists)
```

these are isolated in that
* every ApiImpl MUST NOT call peer-impl classes, but only down to the layers below.
* they MAY call their own internal methods however they feel like
* they MUST NOT persist internal state. These are just partitioning of operations, not stateful classes.
* they must be re-entrant across threads
* they MUST NOT need to be close()-d or have any managed lifecycle. They can be created on demand, reused, etc.


As usual, the backwards compatibility problem holds us up here. The key argument against splitting this up is "these are unstable areas where we need to backport fixes".

Maybe that is what we should use as our metrics for suitability of refactoring:

1. New features: factor out cleanly from the start, with StoreContext, interface for callbacks etc.
1. Enhancement of stable features: pull out if the effort is justified.
1. Important code paths where changes or fixes will need to be backported: leave
1. Fixes: leave.
1. Stable features where there are minor "irritants", e.g. variable names, import ordering: leave.
   +A bit more freedom here on test suites which are rarely changed by others.

This makes for a fairly straightforward checklist for changes.
