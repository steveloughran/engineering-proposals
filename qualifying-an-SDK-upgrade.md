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

# Qualifying an AWS SDK upgrade 


> The AWS SDKs and CLI are designed for usage with official AWS services.
> We may introduce and enable new features by default, such as these new default integrity protections,
> prior to them being supported or otherwise handled by third-party service implementations.

That is a quote from an [announcement of a somewhat incompatible change](https://github.com/aws/aws-sdk-java-v2/discussions/5802) which shipped in v 2.30.0 of the AWS SDK.

It highlights the SDK team's point of view: their job is work on the SDK to support AWS's own services.
Compatibility with third party services is not their problem, and they do not test against such stores.

This makes sense from their perspective: if someone implements their own S3 store, then it is
their task to make it compatible with AWS S3, even as that is a moving target with no public
formal API specification.

The S3A connector is one of the most popular of S3 connectors used to connect JVM-hosted big-data applications to AWS S3 *and to other S3-compatible stores*.
We do not have the luxury of saying "third party stores are not our problem", so have to make
sure that our release works with all stores.

And because of that broad adoption, we need to make sure that it works in different deployment scenarios, with different configurations even within AWS.

The task of qualifying an AWS SDK is a lot more than just incrementing a number in a maven POM file.


## Introduction

An AWS SDK update is a significant change to the codebase.

The S3A connector is utterly dependent upon it and even a minor change can have serious consequences.
That is: a single line change in a maven file can bring new features and needed bug fixes.
It can also cause a lot of damage, albeit unintentionally.

Some example regresssions encountered previously include:
* The SDK printing a warning message telling developers off every time a specific object in the SDK is instantiated
  This breaks all tests which look for specific output strings and runs a risk of generating support calls asking "why is my application telling me off?" 
* A change in the semantics of calling `abort()` on a stream.
  This was a valid design decision. However, it was unexpected. And again the warning message printed every time the stream was closed prematurely flooded application logs.
* Instabilities in the shading of third-party libraries (slf4j, etc)
* The shaded library unintentionally declaring dependencies which redundant due to the shading.

Third-party store support can also be trouble as it does not appear to be something tested by the AWS SDK team themselves (why would they?). This means our code may be one of the first contact points between an update of the SDK and third-party stores.

The core semantics of the S3A/SDK integration can be reasonably well tested simply by running the S3A integration test suite with all the optional features covered:
* KMS encryption
* Versioned bucket support
* AWS access points
* STS session tokens
* Third-party storage


The challenge when qualifying an SDK is to make sure that condition holds. 

From the outset, assume that there is a regression -and that your challenge is to find it.
That is rather than the qualification being a process  "run some automated and manual test to show that all is well",
the task has to be approached as one of "find out what has broken, where and why".
Then we can worry about how to fix. 

The test process then: run the usual integration test with as many of the optional features covered.
Do not simply verify that everything appears to have worked:
you must also look through all the log output to make sure there are no new warning messages being printed indicating a mismatch between how the S3A code is using the library and the library expects to be used. 


What happens if a regression does surface and the qualification process that did not find it
-and now the SDK upgrade has been applied?

We revert. Immediately. Then the process for identifying and trying to remedy the issue surfaces.
If the library has already shipped, then this is harder.
Here as well as identifying the root course we need to assess what is the impact of this in production deployment.
We may need to issue a new Hadoop release.
This is time-consuming and painful and for a simple needless hard work. This is why it is so important we have to get it right.

What happens if I absolutely need a new feature in the latest SDK?
Congratulations! You have just taken on the task of qualifying the SDK release!


## Stop! Is this a last minute action before a release?

If so: _it is too late_.

We need at least two weeks of stabilization to see if other developers
encounter problems related to their own set-ups: endpoints, networks,
credentials -as well as applications built on top of it.

If it is for any feature: postpone the release or don't do the update
Is it for a critical fix: postpone the release.

Either way: we need that time to find things before shipping. 

## Choosing an SDK

Use an SDK which has been out for two or more weeks.


* If we do need a specific release for a fix: go with that one or later.
* If it is a feature we need, do always try for a slightly later build.

Features always take time to stabilize, so let others find the problems and
AWS engineers the solutions.

1. Look at [announcements](https://github.com/aws/aws-sdk-java-v2/discussions/categories/announcements)
   to see if there is a recent announcement related to S3 or core authentication.
2. Look at [issues](https://github.com/aws/aws-sdk-java-v2/issues) to see what
   recently reported issues are which may cause problems.
   Look at the discussion, and if it is relevant, subscribe.
   Consider also examining our code to see if there is any actual exposure.
   Do not just look at the open issues: look at all recent issues as there may be recently closed bugs whose fixes must be picked up; this
   search identifies them.

## Test Setup
To be confident the upgraded SDK works in many deployment configurations, we
need to validate it with as many of the different configuration options and
store types we can.
### Test Buckets


Submitter must have the following buckets:
* B1: 
    - S3 standard
    - SSE-KMS at bucket level
    - Also has S3 server logging to B2. Not
* B2: 
    - S3 standard
    - versioned (with versions configured to delete after 7 days)
    - configured with path style access.
    - This should also have an access point defined -`B2AP` is a bucket configuration
      to access it via the AP.
* B3: S3 express
    - Using CSE-KMS
* B4: S3 standard  (i.e. if you test in usw-2, this is is us-east
    - S3 standard
    - us-central/us-east-1n
    - with long-long distance link to the test system
* B5: third-party store.
  - Google GCS is straightforward here, and documented in [third party stores](./third-party.html).

Testing with a least one third-party store is critical, as is an S3 Express store.
Ideally, test with multiple third-party stores.

with long link to the test system

| Id   | Class             | Config                                                    |
|------|-------------------|-----------------------------------------------------------|
| B1   | S3 standard       | SSE-KMS; versioned. Has S3 server logging to B2           |
| B2   | S3 standard       | Path style access, MUST BE same region as B1.             |
| B2AP | Access Point      | Access Point to B2 also with access point (TLS 1.3+ only) |
| B3   | S3 express        | Optionally: encrypted with CSE-KMS                        |
| B4   | S3 standard       | Long haul link in US and access point access              |
| B5   | Third-party store | Google GCS or other third-party store                     |

These are the core storage class/configurations which are used in production,
hence are part of the qualification process.

One of the buckets B1-B4 MUST be in a region for which there is a FIPS endpoint,
so that it can be configured to use it for access.
That bucket must therefore be within a US region.

A third party store *must* be tested.

*Note* in the XML below, replace B1, B2, etc with the names of your test buckets.

```xml
<property>
  <name>fs.s3a.bucket.B4.endpoint.fips</name>
  <value>true</value>
</property>
```


All buckets which support lifecycle policies SHOULD be set to abort all pending uploads
after 24h, delete all files after 7d.

### Test Host

We also need two test hosts to validate behaviour in the two key scenarios: in AWS and outside of it.

You can start with a single host, but you will need to validate the behaviour of the SDK in both scenarios.

Within AWS
1. EC2/kerberos deployment outside us-central and within a VPC whose network rules can be configured to not allow access to us-central/us-east.
   The build can done without that rule (needed for the artifact download), but a test run must be one locked down. This is to validate local region resolution.
3. On a remote host, with any config for the AWS CLI (temporarily) renamed from `~/.aws/config`. to something else.
  This is needed to make sure the SDK isn't reading region/endpoint info from that file, as
  it can do.


### Configuration and extra services

Submitter MUST have the extra AWS setup for: 
* KMS encryption
* Assumed role for session token tests.
* An access point.

This may seem a lot of preparation but it is needed for full test coverage. 

These configuration SHOULD go into an XIncludable configuration file which can be referenced absolutely,
for example in a directory `~/config/auth-keys.xml`, which can be
referenced both from hadoop-aws tests, and in full distributions you have
built up.

Ideally, both `hadoop-tools/hadoop-aws/src/test/resources/auth-keys.xml`
and `etc/hadoop/core-site.xml` will look identical

```xml
<configuration>
  <include xmlns="http://www.w3.org/2001/XInclude"
    href="///users/alice/config/auth-keys.xml">
  </include>
</configuration>
```

Tip: initialize that `~/config/` directory as a *local* github repo, it is easier to
see what you've broken. Obviously you MUST NOT push it to any remote repo if it contains
your AWS secrets.


```xml

<configuration>
  <property>
    <name>fs.s3a.assumed.role.arn</name>
    <value>$ROLE_ARN</value>
  </property>

  <property>
    <name>fs.s3a.assumed.role.external.id</name>
    <value>test-id</value>
  </property>

  <property>
    <name>fs.s3a.assumed.role.sts.endpoint</name>
    <value>$STSENDPOINT</value>
  </property>

  <property>
    <name>fs.s3a.assumed.role.sts.endpoint.region</name>
    <value>$REGION</value>
  </property>

  <property>
    <name>fs.s3a.assumed.role.external.id</name>
    <value>any arbitrary.value</value>
  </property>

  <property>
    <name>fs.s3a.bucket.B1.encryption.algorithm</name>
    <value>SSE-KMS</value>
  </property>

  <property>
    <name>fs.s3a.bucket.B1.encryption.key</name>
    <value>$KMSKEY</value>
  </property>

  <property>
    <name>fs.s3a.bucket.B1.encryption.cse.kms.region</name>
    <value>$REGION</value>
  </property>

  <property>
    <name>fs.s3a.bucket.B3.connection.expect.continue</name>
    <value>false</value>
  </property>

  <property>
    <name>fs.s3a.bucket.B1.input.stream.type</name>
    <value>analytics</value>
  </property>
  
  <property>
    <name>fs.s3a.bucket.B2.input.stream.type</name>
    <value>classic</value>
  </property>
  
</configuration>
```

#### Testing Open SSL

On any test system other than an ARM-based macbook, require openssl for one of the buckets other than B1

```xmln
<property>
  <name>fs.s3a.bucket.B2.ssl.channel.mode</name>
  <value>openssl</value>
</property>
```

As `wildfly.jar` doesn't include the ARM64 native libraries -just skip it there.
See [wildfly-openssl](https://github.com/wildfly-security/wildfly-openssl/tree/main) for details.


#### Set up the env vars B1 to B5 as the names of the bucket


```bash
export BUCKETNAME=example-bucket-name
export BUCKET=s3a://$BUCKETNAME

export B2=s3a://bucket-2

# needs an bucket config to match
export B2AP=s3a://bucket-2-access-point

```


## Preflight: Before You Upgrade

### Create a "notes" document to track your work, including timing information from test runs.

### Seek help from others

Getting others to help in the qualification process can help in multiple ways:
1. Splits the work testing the upgrade, which can be done with different buckets.
2. Different development setups helps identify different deployment issues, such
   as in-AWS versus out-AWS clients, where bandwidth and latency are very different.


### JIRA 

Create the JIRA.

Use the title "S3A: Upgrade AWS V2 SDK" ; once a specific version has
been selected it must be renamed to that version.

In the JIRA include references to any AWS issues you have identified which can
require code changes.

### Create the branch

1. Check out `trunk`
2. Create a new branch for the update.
3. Get ready to upgrade!

### Is everything currently OK?


Kick off the initial build with that chosen SDK release with the target bucket you use for normal building and testing.

```bash
# whole project
mvn -T 1C

# hadoop-aws
time mvn -T 1C integration-test -Dmaven.plugin.validation=none -Dparallel-tests -DtestsThreadCount=9 -Dscale
```

If it compiles and the tests work, this is the first good sign.

Save the command and the duration (as well as any test failures) to the notes document.
The duration of the upgrade branch's test run will be compared against this later.

```
________________________________________________________
Executed in   17.37 mins    fish           external
   usr time   26.35 mins   91.00 micros   26.35 mins
   sys time    3.23 mins  854.00 micros    3.23 mins


```


### Do a full hadoop release build

```sh
mvn package -Pdist -DskipTests -Dmaven.javadoc.skip=true -DskipShade
```

Move this to a path outside the hadoop source tree.

```sh
mkdir ../Releases
mv hadoop-dist/target/hadoop-3.5.0-SNAPSHOT/ ../Releases/before-update
```

Copy into its `etc/hadoop` dir the `core-site.xml` config referencing your
separate `auth-keys.xml` file.

This is now your reference "before the upgrade" release build.
If you see what may be a regression during manual qualification,
you can try with this release to see if the problem holds there
too.
If it does: file a bug report independent of the qualification
JIRA, and crosslink with a "testing discovered" relation


### Create a full release for manual testing

Create 
```sh
mvn -T 1C clean package -Pdist -DskipTests -Dmaven.javadoc.skip=true
```

move it somewhere.

```fish
mv hadoop-dist/target/hadoop-3.5.0-SNAPSHOT/ ../Releases/preflight
```

### Check out the AWS SDK 

Get the AWS SDK from  [github/aws/aws-sdk-java-v2](https://github.com/aws/aws-sdk-java-v2),
into a new directory.
For example, using the github `gh` cli:

```bash
mkdir ~/External
cd ~/External
gh repo clone aws/aws-sdk-java-v2
cd aws-sdk-java-v2
```

This will add a new directory `~/External/aws-sdk-java-v2` with the repository.
If you have already got this directory or an equivalent, update it.

Look in the [tag list](https://github.com/aws/aws-sdk-java-v2/tags) for the tag of the release and check this out.

```
git checkout tag/2.30.27
```

This is for identifying what has changed in this release, including what has changed near code which is now failing in tests,
as well as how major changes are affecting classes we use.
Creating a new project in your IDE can assist here.

## Qualification

Allocate a whole week for this, _including preparing your test buckets and other storage details_

This is not just for the overhead of the test setup, and execution, it assumes that there will be regressions and they will need fixing and retesting.


### Clean up all buckets.

From your preflight release, clean out the buckets.
```sh
bin/hadoop fs -rm $B1/\*
bin/hadoop fs -rm $B2/\*
bin/hadoop fs -rm $B3/\*
bin/hadoop fs -rm $B4/\*
bin/hadoop fs -rm $B5/\*
```
This helps verify that every test bucket is well-configured.

### Update the SDK in your local branch

In a new branch off trunk

Update the value of `aws-java-sdk-v2.version` in `hadoop-project/pom.xml` to the new SDK version. 
```xml
    <aws-java-sdk-v2.version>2.30.27</aws-java-sdk-v2.version>

```
In `LICENSE-binary` update the line declaring the version of the bundle.jar artifact included
in distributions.
For example:
```
software.amazon.awssdk:bundle:2.30.27
```

### Do a clean build and create the PR if all is good.

If it compiles
1. Commit the change, including the version number in the title
1. Push to github
1. Create a PR -don't include the version there yet.

After this, leave yetus to do its work. 

As you continue your work, place test results and stack traces into the PR, making it visible to all.
Anyone who is collaborating should do the same.

### Do a clean build and test with your normal bucket

In `hadoop-aws` directory
1. Run `mvn verify`
1. Run the `ILoadTest*` load tests from your IDE or via maven through
      `mvn verify -Dtest=skip -Dit.test=ILoadTest\* -Dscale`  ; look for regressions in performance
      as much as failures.
1. Create the site with `mvn site -DskipTests`; look in `target/site` for the report.
1. Review *every single `-output.txt` file in `hadoop-tools/hadoop-aws/target/failsafe-reports`,
  paying particular attention to
  `org.apache.hadoop.fs.s3a.scale.ITestS3AInputStreamPerformance-output.txt`,
  as that is where changes in stream close/abort logic will surface.



### Testing all the buckets.
 
Run the `ITests` against the other buckets.
This is the most time consuming parts of the process, ~20 minutes for each run and the setup time, assuming they actually work.

What's the best order?
* Start with your normal development bucket, as changes in behavior will be more obvious there.
* Proceed to the third-party store, as that is the most likely to have problems.
* Then the long-haul link
* After that: whatever is most convenient.



### Manual, Exploratory testing.


We need a run through of the CLI to see if there have been changes there
which cause problems, especially whether new log messages have surfaced,
or whether some packaging change breaks that CLI.

It would be straightforward to automate a sequence of commands,
but we do not want to because actually having you use the command line from a terminal window is part of the qualification process, as it can identify issues.

* Does it work?
* Does it suddenly pause for long periods of time? 
* Are AWS SDK libraries printing warning messages? hadoop-aws code?
* Has that some other change in the code base unrelated to the SDK which is now printing new warning messages?

These are things we need to know before end users find out.

The commands below list the minimum set of commands to run;  more you can think of will be wonderful.

In fact, an ideal outcome of qualifying a upgrade is that you have some new commands to add to this list.

In particular, we could benefit from a lot more fault injection to see how well the SDK recovers from problems.
This is often hard to test because S3 has such great reliability and because all of us developers working with cloud storage have fast and reliable networks.
In production enough requests are made to S3 through our code every day that many applications will actually encounter transient failures of the S3 end points, which need to be recovered from.
And people running this code are often doing it remotely, often through proxy, and sometimes to other S3 endpoints

It is always interesting when doing this to enable IOStatistics reporting:
```xml
<property>
  <name>fs.iostatistics.logging.level</name>
  <value>info</value>
</property>
```

From the root of the project, create a command line release `mvn package -Pdist -DskipTests -Dmaven.javadoc.skip=true  -DskipShade`;

1. Change into the `hadoop-dist/target/hadoop-x.y.z-SNAPSHOT` dir.
1. Copy a `core-site.xml` file into `etc/hadoop`.
1. Set the `HADOOP_OPTIONAL_TOOLS` env var on the command line or `~/.hadoop-env`.

```bash
export HADOOP_OPTIONAL_TOOLS="hadoop-aws"
```

Now run some basic hadoop CLI operations.

*note* these examples are from the previous qualifying section in testing.md; they have not been updated for the multi-bucket setup.
Changing the environment variables should suffice.

```bash

export BUCKETNAME=example-bucket-name
export BUCKET=s3a://$BUCKETNAME

bin/hadoop s3guard bucket-info $BUCKET

bin/hadoop s3guard uploads $BUCKET
# repeat twice, once with "no" and once with "yes" as responses
bin/hadoop s3guard uploads -abort $BUCKET

# ---------------------------------------------------
# root filesystem operatios
# ---------------------------------------------------

bin/hadoop fs -ls $BUCKET/
# assuming file is not yet created, expect error and status code of 1
bin/hadoop fs -ls $BUCKET/file

# exit code of 0 even when path doesn't exist
bin/hadoop fs -rm -R -f $BUCKET/dir-no-trailing
bin/hadoop fs -rm -R -f $BUCKET/dir-trailing/

# error because it is a directory
bin/hadoop fs -rm $BUCKET/

bin/hadoop fs -touchz $BUCKET/file
# expect I/O error as it is the root directory
bin/hadoop fs -rm -r $BUCKET/

# succeeds
bin/hadoop fs -rm -r $BUCKET/\*

# ---------------------------------------------------
# File operations
# ---------------------------------------------------

bin/hadoop fs -mkdir $BUCKET/dir-no-trailing
bin/hadoop fs -mkdir $BUCKET/dir-trailing/
bin/hadoop fs -touchz $BUCKET/file
bin/hadoop fs -ls $BUCKET/
bin/hadoop fs -mv $BUCKET/file $BUCKET/file2
# expect "No such file or directory"
bin/hadoop fs -stat $BUCKET/file

# expect success
bin/hadoop fs -stat $BUCKET/file2

# expect "file exists"
bin/hadoop fs -mkdir $BUCKET/dir-no-trailing
bin/hadoop fs -mv $BUCKET/file2 $BUCKET/dir-no-trailing
bin/hadoop fs -stat $BUCKET/dir-no-trailing/file2
# treated the same as the file stat
bin/hadoop fs -stat $BUCKET/dir-no-trailing/file2/
bin/hadoop fs -ls $BUCKET/dir-no-trailing/file2/
bin/hadoop fs -ls $BUCKET/dir-no-trailing
# expect a "0" here:
bin/hadoop fs -test -d  $BUCKET/dir-no-trailing ; echo $?
# expect a "1" here:
bin/hadoop fs -test -d  $BUCKET/dir-no-trailing/file2 ; echo $?
# will return NONE unless bucket has checksums enabled
bin/hadoop fs -checksum $BUCKET/dir-no-trailing/file2
# expect "etag" + a long string
bin/hadoop fs -D fs.s3a.etag.checksum.enabled=true -checksum $BUCKET/dir-no-trailing/file2
bin/hadoop fs -expunge -immediate -fs $BUCKET

# ---------------------------------------------------
# Delegation Token support
# ---------------------------------------------------

# failure unless delegation tokens are enabled
bin/hdfs fetchdt --webservice $BUCKET secrets.bin
# success
bin/hdfs fetchdt -D fs.s3a.delegation.token.binding=org.apache.hadoop.fs.s3a.auth.delegation.SessionTokenBinding --webservice $BUCKET secrets.bin
bin/hdfs fetchdt -print secrets.bin

# expect warning "No TokenRenewer defined for token kind S3ADelegationToken/Session"
bin/hdfs fetchdt -renew secrets.bin


# ---------------------------------------------------
# Copy to from local
# ---------------------------------------------------

time bin/hadoop fs -copyFromLocal -t 10  share/hadoop/tools/lib/*aws*jar $BUCKET/

# expect the iostatistics object_list_request value to be O(directories)
bin/hadoop fs -ls -R $BUCKET/

# expect the iostatistics object_list_request and op_get_content_summary values to be 1
bin/hadoop fs -du -h -s $BUCKET/

mkdir tmp
time bin/hadoop fs -copyToLocal -t 10  $BUCKET/\*aws\* tmp

# ---------------------------------------------------
# Cloudstore
# check out and build https://github.com/steveloughran/cloudstore
# then for these tests, set CLOUDSTORE env var to point to the JAR
# ---------------------------------------------------

bin/hadoop jar $CLOUDSTORE storediag $BUCKET

# stresses upload speed, and that the pool and timeout settings work
time bin/hadoop jar $CLOUDSTORE bandwidth 512M $BUCKET/testfile

```

+Any other commands you can think of!


## More Testing


* Whatever applications you have which use S3A: build and run them before the upgrade,
Then see if complete successfully in roughly the same time once the upgrade is applied.
* Test any third-party endpoints you have access to.
* Try different regions (especially a v4 only region), and encryption settings.
* Any performance tests you have can identify slowdowns, which can be a sign
  of changed behavior in the SDK (especially on stream reads and writes).
* If you can, try to test in an environment where a proxy is needed to talk to AWS services.
* Try and get other people, especially anyone with their own endpoints,
  apps or different deployment environments, to run their own tests.
* Run the load tests, especially `ILoadTestS3ABulkDeleteThrottling`.



## Committing the patch

When the patch is committed: update the JIRA to the version number actually
used; use that title in the commit message.

Be prepared to roll-back, re-iterate or code your way out of a regression.

There may be some problem which surfaces with wider use, which can get
fixed in a new AWS release, rolling back to an older one,
or just worked around [HADOOP-14596](https://issues.apache.org/jira/browse/HADOOP-14596).

Don't be surprised if this happens, don't worry too much, and,
while that rollback option is there to be used, ideally try to work forwards.

If the problem is with the SDK, file issues with the
 [AWS V2 SDK Bug tracker](https://github.com/aws/aws-sdk-java-v2/issues).
If the problem can be fixed or worked around in the Hadoop code, do it there too.


## Handling compile/test failures


### Dealing with Deprecated APIs and New Features

A Yetus run should tell you if there are new deprecations.
If so, you should think about how to deal with them.

Moving to methods and APIs which weren't in the previous SDK release makes it
harder to roll back if there is a problem; but there may be good reasons
for the deprecation.

At the same time, there may be good reasons for staying with the old code.

* AWS have embraced the builder pattern for new operations; note that objects
constructed this way often have their (existing) setter methods disabled; this
may break existing code.
* New versions of S3 calls (list v2, bucket existence checks, bulk operations)
may be better than the previous HTTP operations & APIs, but they may not work with
third-party endpoints, so can only be adopted if made optional, which then
adds a new configuration option (with docs, testing, ...). A change like that
must be done in its own patch, with its new tests which compare the old
vs new operations.


### What to do if there is a regression?

Obviously, the PR canot be merged until resolved.
The cause has to be identified, then fixed.

The default assumption should be "our assumptions about the behaviour of the SDK proved to be incorrect".
Identifying what has gone wrong and aware those assumptions were made means that we can fix it ourselves
Although this does require engineering effort, it doesn't guarantee that we can get a fix in without waiting for any changes from the AWS SDK developers.

### Tracking down a test failure

What to do if a test starts failing?

First, add the stack traces to the PRs as a comment.
As well as warning everyone of problems, it generates a searchable trace for the future.

Next, *do not asssume that this is a bug in the tests*.
Assume that the test has identified a regression in production code.
Hopefully it is just a test failure do to minor changes in SDK behavior -however this is the best case scenario.
Assuming it is a test failure and so disabling the test case/assertion is a mistake.
The root cause is still out there, waiting to surface again in production.

_only disable a test case/assert once you are confident it is not a production-time issue_.

### Process for fixing an emergency facility/bug in our code. 

Treat it as any other bug in the code. 
1. Create a new JIRA ticket.
2. Link to the SDK upgrade JIRA as a Blocker.

As usual, it needs a test to replicate it and a fix.
If the change can be done independently of the SDK update,
then it can go into the code base before the version is updated.

What about changes which require the SDK in first?
If it is a small change, especially a low risk test one, include it in the SDK update.
 
If it is a large change:
1. Create a Hadoop PR to update the SDK only.
1. Create a feature branch with the commit of #1 at the bottom.
2. Get all the code reviewed by the normal process *but do not merge it once approved*
3. The final merge should be done with a merge of the SDK in first, with that commit
   message declaring it must be followed by the big patch (state the JIRA and PR IDs))
4. Apply the big patch immediately after the SDK update PR is merged, with a mention
   of that JIRA/PR ID in the commit message body.


### What if it is a bug in the AWS SDK itself?

This is a problem, the seriousness depends on the nature of the issue.

#### Create a Hadoop JIRA

1. Create a JIRA with as much information as you can.
1. Describe the problem replicated, the consequences.
1. Check out the relevant SDK release and see if you can identify the root cause.
   It's complex enough that this is unlikely, but gaining familiarity with
   the SDK is a good investement.

#### Look for an existing SDK issue

Search [AWS v2 SDK isues](https://github.com/aws/aws-sdk-java-v2/issues) for
reports of the topic.
Include recently closed issues, as it may have been fixed.

If the issue has been reported and is still open:
* Add a comment stating you are also affected, and the impact on our code.
* Subscribe to the issue
* Link the HADOOP JIRA to it.

#### Create an AWS SDK github issue if needed

Create a matching issue for the AWS, providing the same information in as much detail as you can.
Cross-link with the hadoop JIRA and vice versa.

Then try and come up with a workaround. This may take a significant amount of effort.
Note that the class `org.apache.hadoop.fs.s3a.impl.AwsSdkWorkarounds` is a place for workarounds...
logging is already in there.

This class has its own ITests. Ideally these tests should fail when the underlying issue is fixed
-this hightlighs when a workaround can be removed.

If one cannot be found then we are essentially blocked from upgrading the AWS SDK at all.
We have had to do exactly this with problems related to library shading.

You cannot expect a bug report to the SDK team to result in any urgent fixes.  
This is "unfortunate" – especially given the petabytes of data which must be passed through the S3A connector to and from S3 every day.

[//]: # (we suspect that there is some Conway's law-structure at work here.)

[//]: # (Whoever maintains the SDK is not the same people who run the S3 service and that's priority does not appear to go to sort of library even though many of their premium customers are using our code. )

If this all seems a bit negative – do not panic.
Most of the upgrades are straightforward and do not appear to cause any problems.


## Declaring the PR ready to merge


Storediag output of all stores are attached. for B1, diagnostics through an access point are also attached.

Then we have a set of attestations
[ ] I have run the S3 ITests against B1;  no failures were observed.
[ ] I have used distcp to collect the audit logs from B2 -logs spanning the timespan of the tests. 
[ ] I have run the ITests against B3; no failures were observed.
[ ] I have run the ITests against B4; no failures were observed
[ ] If available, I have run the ITests against B3; no failures were observed
[ ] I have compared the logs of the before and after runs; no differences were observed. (maybe we should provide a log4j format which logs at info and doesn't include time and thread IDs?)
[ ] I have run the CLI tests against all buckets, no failures or changes in logs were observed
[ ] I have added one or more new CLI tests to run; they are included in this PR. (forces submitter to think of new tests rather than set as "complete")
[ ] The formatted test results are attached as a single .tar file containing a subdir for each test bucket.

Then

1. Measure the execution time of before/after runs should be listed to see if there is any slowdown vs the previous version.
A simple `time mvn -T 1C verify -Dscale -Dparallel` of the hadoop-aws dir after just having done a `mvn clean install` to take that out of the timing would be enough.
This is to identify major changes. Ideally this should be done on an EC2 VM, so there are no network-related issues.

2. Check out the relevant tag of the aws-sdk-v2 and use ripgrep to count the # of matches of the pattern `\.warn\(` in those modules we care about.
For files where there's a change, open them, get the history, see what has changed.
This not just to identify where are being told of in a way which makes for noisy clients,
it is to see if there are potentially things we are getting wrong and which should fix.

The PR submitter then has to make some commitments to followup on regressions.

If there is a regression identified by anyone
* I understand that the immediate action is a revert of the PR until addressed.
* I will collaborate with others to identify and replicate the problem.
* If a fix is needed in our code, I will collaborate on fixing the issue including writing automated/manual tests, and running them
* If the regression is in AWS code, I will file the AWS issue. Furthermore, if I'm an AWS engineer: file an internal one.
* If a workaround is needed to fix the SDK problem, I will collaborate with others to design and implement the workaround.

The key point here is to 
1. Make clear that and that whoever providing the update owns a lot of the upgrade problem, rather than expect others to handle it.
2. Highlight that regressions are blockers on upgrades.

#### SDK summary

Assuming that we are the first people to deploy applications with the V2 SDK the scale of terabytes to petabytes of data a day, I think we should be treated as a priority source of bug reports: _we are finding things before other people encounter them_.
Sometimes we do find them in production environments -but as they will be widely encountered in many other installations, and each of these may surface as an escalation through their customer account, early fixes matter.



# Appendices

## Basic set of manual commands



## Fish functions for a happier maven

Fish is [the command shell for the 1990s](https://fishshell.com/) and makes for a far better
testing environment than bash, zsh etc!

Here are some fish functions which provide useful shortcuts
to common operations.
All must go into `~/.config/fish/functions/`, in filenames matching
the function names.

```sh

# ~/.config/fish/functions/mci.fish
function mci --description 'Maven clean install; no tests or shading'
  mvn -T 1C clean install -DskipTests -DskipShade $argv
end

#  ~/.config/fish/functions/mi.fish
function mi --description 'Maven install -no tests or shading'
  mvn -T 1C  install -DskipTests -DskipShade -Dmaven.plugin.validation=none $argv
end

# ~/.config/fish/functions/mvndep.fish
function mvndep --description 'mvn dependency:tree -Dverbose'
  mvn -T 1C dependency:tree -Dverbose $argv
end

# ~/.config/fish/functions/mvit.fish
function mvit --description 'mvn integration test'
  mvn -T 1C integration-test $argv
end

# ~/.config/fish/functions/mvt.fish
function mvt --description 'mvn test'
  mvn -T 1C test $argv
end
```

Note: while the `-T 1C` improves parallelism, sometimes maven
has the odd concurrency-related failure, usually surfacing as lock acquisition
or the whole build actually hanging.

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-remote-resources-plugin:1.5:process (default) on project hadoop-aws: Execution default of goal org.apache.maven.plugins:maven-remote-resources-plugin:1.5:process failed: Could not acquire lock(s) -> [Help 1]
```

Do remember that these functions all set the `-T 1C` and, if builds
show problems -stop using them.

# Forcing TLS 1.3 on an access point

Mandating use of TLS 1.3 on an access point is a way to validate the s3a client access works with TLS1.3  without any behind-the-scenes downgrading.

Here is an example policy to restrict access, based on [an AWS blog post](https://aws.amazon.com/blogs/storage/enforcing-encryption-in-transit-with-tls1-2-or-higher-with-amazon-s3/).

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:us-east-2:123456789012:accesspoint/ap-awsexamplebucket/object/*",
            "Condition": {
                "NumericLessThan": {
                    "s3:TlsVersion": [
                        "1.3"
                    ]
                }
            }
        }
    ]
}
```