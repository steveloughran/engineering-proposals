# Open Source and CVEs: the Forever War

*CVE Debt*. noun.

The security equivalent of technical debt, accrued by not proactively updating dependencies.
Example usages
- "I know moving from AWS V1 to V2 SDK is enough of a nightmare that it makes me feel they are punishing me for using the old SDK, but it's unmaintained and the CVE debt must be growing"
- "wow, you're still on WinXP -your production line has enough CVE debt that I'm surprised you can get insurance"

Since about February all OSS projects are getting a flood of email reports of people who got an AI tool, usually claude, to discover a vulnerability and who want credit for the CVE.
We have to triage these into various categories

1. Utterly bogus. Hadoop YARN is a Job Submission Engine, so any Remote Code Execution as an authenticated user is correct behaviour. Similarly, if you deploy HDFS with Kerberos turned off, expect permission bypasses.

2. Bugs. Example: Incomplete validation of arguments could lead to allocation of INT_MAX memory. Submitters seeking CVEs to their name want CVEs here, but generally projects are being ruthless here, unless it is part of an DoS attack from an unprivileged source, it's just a "file a JIRA" problem. But they might get priority just to keep the AI tools quiet.

3. Vulnerabilities with no understanding of the security model. If "OS is subverted" is a pre-req then you don't need to attack the layers above. Similarly, in-cloud deployments without kerberos are fine provided you aren't stupid enough to open the network ports up; HDFS will be transient, it'll only be cloud storage that contains persistent data. The credentials to access the store are attached to the VMs by cloud-specific mechanisms, and all privilieges and protections come via those credentials.

4. Real vulnerabilities. Some of the AI reports are genuine; engineers have to fix them, and by their nature they aren't done in the usual open to review collaboration which makes for OSS dev efficient. Some of these reports are so vague it's hard to distinguish those from the other types. Best are real [DAST PoC attacks](https://github.com/resources/articles/what-is-dast) on systems deployed with production network rules and security credentials. [SAST](https://github.com/resources/articles/what-is-sast) is something we developers should be having our our CI tools doing...anyone who wants to get a CVE against an OSS project and get the credit _themselves_, rather than have us credit Claude, need to put in the effort here rather than some AI generated PoC code the submitter doesn't actually understand.

As well as these third party reports, OSS teams also have our own AI tooling; Anthropic's Glasswing project and similar, which try to assist in finding vulnerabilities.
While this gives us access to better models, allowing us to explore identified vulnerabilities, it also leads to a lot more reports which we again have to triage into bugs and CVEs

Because of this expect all OSS projects to be releasing regularly this year due to their own newly discovered issues.
Because of transitive dependencies, expect these same projects to also be trying to cope with updates to everything they depend on.

As an extra complication, whereas historically CVE fixes could be snuck out prior to announcment as part of a larger change "improve compression tests" or "better logging", it is now possible to point an LLM at the commit log of any OSS project and ask it to review every commit to identify internal vulnerabilities fixed, then ask the tool to craft an exploit.
As a result, we may lose that ability to get fixes into the public branches before the releases.
What then? Private security-fixed branches only made public with that first RC of a new release? 
That'd kill the community.
Nightly  releases? Maybe. 

## Issue: Prioritisation

Which would downstream organisations prefer those of us in open source projects to be working on,
given there is a limited amount of time which can be allocated to this amongst all our other deliverables,
and knowing there is nobody is this or any other OSS project who works full time on security?

1. Dealing with what we consider to be real CVEs in our code?
2. Upgrading transitive dependencies which we know are real vulnerabilities?
3. Upgrading transitive dependencies of things where it's clearly not on any codepath we use?

Right now, OSS projects do not have the capacity to cope with the amount of CVE related workload being dumped on us. 

## Issue: What Contributions are Appreciated?

OSS projects don't need to be given lists of "here are the CVEs our audit found", especially when those lists haven't even been categorised into what library it is in. I take the view "if the author can't be bothered to do that, why should I?". Both [dependabot](https://github.com/dependabot) and [OSV-Scanner](https://google.github.io/osv-scanner/) givr us better lists.

*The challenge with dependencies is not knowing that the previous releases have CVEs, everyone should assume that, it is in getting the update in such thar it doesn't break our project or those downstream. Even when projects strive for compatibility, the hardening they do as part of CVE mitigation can introduce (necessary) regressions.*


If anyone really wants to get dependency uodates, especially "checklist cve updates" changes in on a timetable which works, they are going to have to be proactive here and with all the other projects that they consider critical parts of the supply chain. 
- supplying PRs which pass tests, ideally tested in their own deployments.
- help test the release candidates.
- help fix those bugs reported as potential CVEs but triaged down to normal issues.
- dependency pruning PRs.
- fuzzing! 
- help us develop more automated processes where we can offload lot more of the work to the infrastructure.

As an example, I recently provided a [PR to upgrade libtrift in Apache Parquet](https://github.com/apache/parquet-java/pull/3589).
As I noted, this was just a checklist CVE, but it was one we needed in Parquet 1.18.0.
To help get this ready for a new release, rather than expect others to fix it on my timeline, I created the PR.
The first commit was just the dependency update --but the PR CI test run identified NPEs in test cases in the parquet-thrift module.
I had to identify the cause, work on a fix under the supervision of a project committer, and then once they were happy it was approved and merged. 
The committer, Fokko, had to put in effort too; reviewing is work of its own, but by taking on the debugging and doing the changes he wanted, he got a fix he was happy with in, I got a fix I was happy with in, and everyone downstream will get fewer audit complaints.
I also know a bit more about Thrift internals, which is a learning that I wouldn't have got if I'd delegated it to AI.
This lines me up for exploring issues in that codebase better, and understanding any stack traces from it slightly better.
That is: along with the PR, I am improved (though as I had a carrot cake at the Canteen Cafe while I did the fix there, my cycling hill climbing is sadly degraded).

## Looking to the future

Hopefully we can get through this storm of CVEs, as the backlog of extent vulnerabilities are identified and fixed.
Do that and I don't see new LLMs suddenly discovering new exploits, simply able to chain together more complex attacks, maybe do more DAST experiments themselves.
We're going be be able to use those AI tools to help deal with reports.

### Continuous Releases of OSS artifacts

I think we are going to have adopt a radical release process of 90-95% automated monthly releases with the option of interim OOB security fixes.
We moed from specific product versions to timestamps "something-2026.07.14" and stop pretending that Semantic Versioning offers any guarantees of compatibility. 

At the same time, the spread of the Shai Halud worm through the NPM package ecosystem shows us that a fully automated release process is deadly, especially when coupled with a build system where naively trusting the latest release is the usual dependency model.

### Ending the Myth of Semantic Versioning

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





