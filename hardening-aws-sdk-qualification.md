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

# Proposed changes to the qualification process

## Background

Background: the purpose of the qualification process is not just to identify regressions in the AWS SDK: to identify where developers working on the S3A have made bad assumptions about behaviors of the library, or simply misunderstood bits of it.
That is: it is not simply validating the SDK –it is validating how it is used in our code.
Filing Hadoop JIRA is as legitimate an outcome as anything else.

## changes
Increase the number of S3 stores qualify is required to test.

MUST: S3 standard, S3 express,
MUST: google GCS
(GCS is low cost to get at and will help test new stuff like conditional PUT)
MAY: anything else you can get

SHOULD:
S3 Standard bucket to be versioned bucket (and set as change detection mode),
with version expiry time kept low (24h) to keep costs down.

MUST: KMS key be created so S3-KMS encryption can be tested.
MUST: new IAM role defined to test role acquisition. Grant this role no more rights than the primary role.


MUST: encryption, STS, 

SHOULD: cse

## command line tests




Emphasise that this is exploratory testing: the goal is not simply "does this work" but "does this work as I would expect?". That is: does it take longer, does it print odd things? This is why these tests are manual, rather than simply automated. We could do that –but we want the person doing the contribution to be an active participant in the process the way a script is not.

It is also intended to test scales we don't ever run integration tests at: things you would get bored waiting for.

Nor are the tests complete. Feel free to come up with more complicated operations that you can think of. The initial list was just some basic ones put together. 

Ideas
- distcp
- cp that bundle.jar between buckets, maybe even stores.
- download and verify checksums of uploaded file (cloudstore bandwidth 1GB) will do all this. get the results as text and CSV. attach to the JIRA.

MUST
cloudstore bandwidth for 1G

Other things to consider
* USE EC2 instance metadata as credential source
* 