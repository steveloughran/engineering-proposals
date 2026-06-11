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






