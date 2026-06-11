# Ending the Myth of Semantic Versioning

[Semantic Versioning](https://semver.org/), SemVer, is a an artifact numbering scheme that a lot of people believe their releases follow, and which even more people believe their dependencies comply with

> Given a version number MAJOR.MINOR.PATCH, increment the:
>
> MAJOR version when you make incompatible API changes
> MINOR version when you add functionality in a backward compatible manner
> PATCH version when you make backward compatible bug fixes
> Additional labels for pre-release and build metadata are available as extensions to the MAJOR.MINOR.PATCH format.

This is all very well but Software Engineers who have studiously read the work of DL Parnas will be aware of the nuances of the epiphenomena of implementation details and how applications end up coding against them, rather than the interface themselves.
Those who haven't read his work: you are not Software Engineers. Sorry.

* Every change is potentially backwards incompatible, even if it is not directly observable in the public API, at least as observed as in the unit test suites. *

Therefore: ALL minor and ALL patch versions MAY be backwords incompatible, even if there is no change in the public API.
That Thriftlib update is an example of this; internal changes meant that while before ParquetProtocol could get away with supplying a null TTransport to its thrift TProtocol superclass constructor parameter, this parameter was now used, so triggering NPEs.
It's quite likely that the thrift API spec was explicitly requiring TTransport to not be null, but in thr absence of validation, Parquet unintentionally coded against the implementation.
When a nominally compatible patch release shipped, things broke.



* A major version self-identifies as incompatible, so scares people into not updating *

As I've just shown, point releases may break things.
Major releases are when teams go big, remove APIs, change types, function signatures and more.
That is: API level incompatiblity
Sticking to point releases of dependencies has a short term benefit: you only have deal with Semantic incompatibilities. Major versions add compile-time/link-time incompatibilities.

It's easy to save code reworking by staying on older releases.
But there are costs
- You are stuck on an older version
- If an OSS project has made any commitment to support that older version, it adds a cost to their work, which can slow down the release rate of the latesr version
- If they don't update that version, you get to keep it and its bugs.

In a world without CVEs, you'd be left with only those bugs, which you either didn't encounter in your use, or knew about and worked around.

But we live in a world of CVEs, and the number of CVEs in a project is going to be some function of time and code quality.
- When eventually there is perceived to be a critical need to upgrade a dependency, it'll now be an emergency CVE fix crisis where all incompatibilities surface at once.

*Freezing a dependency at an older version is creating a CVE debt that'll grow over time*

...and as I've shown based in my own PR, doesn't stop regressions at the semantic level anyway.


This then, is the harsh truth of Semanti Versioning

* Major: things may break at compile, link and during execution. Deal with it.
* Minor: things that work may break during execution.  Deal with it.
* Minor: things that work may break during execution. Complain, then deal with it.

What if move to a time based model, similar to ubuntu numbering?

- Makes the age of a dependency really obvious, so its likely CVE debt.
- Stops trying to pretend that changes don't break existing code.

That's very much "pretending that changes don't break existing code".

The trouble here though: guava's habit of cutting things.

## what if it's impossible to upgrade even a point release?

Tricky one this, shows how meaningless SemVer is and highlights how versioning is the big unsolved problem in computing

One example of a point release breaking things was AWS SDK 2.34, which broke compatibility with third party object stores implementing the S3 API due to changes in checksumming.
AWS's PoV was a "worksforme", which, from their perspective was good enough to ship:
Third party reimplementations of their S3 API should simply stay current.
Sadly, Apache Iceberg shipped with a this SDK and had to issue a new release rolling back to a working one, as .
Hadoop stayed with the old one until eventually the SDK team did a later update which made the new checksum optional.
We had to make a lot of noise for that, and suspect that without support of AWS engineers especially Ahmar Suhail, and the little detail that hundreds of petabytes of data runs between S3 and EC2 instances every day through our OSS code and that of others meant that we mattered.
This would count as an argument against SemVer except it shows the "things may break at runtime" warning may be a blocker to an update, even a point version.

### What should library developers do then?

Try not to ruin the lives of people who use their library.