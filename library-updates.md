# library updates

I never share any details of the security list, except here are my observations of it and those of what I know of other projects, as sent to someone whose team were asking for the timeline for an update of a dependency with a CVE on the release we ship with.
That's a CVE to which our code isn't exposed, BTW


---


Since about February all OSS projects are getting a flood of email reports of people who got an AI tool, usually claude, to discover a vulnerability and who want credit for the CVE. We have to triage these

1. Utterly bogus. Hadoop YARN is a Job Submission Engine, so any Remote Code Execution as an authenticated user is correct behaviour. Similarly, if you deploy HDFS with Kerberos turned off, expect permission bypasses.
2. Bugs. Example: Incomplete validation of arguments could lead to allocation of INT_MAX memory. Submitters seeking CVEs to their name want CVEs here, but generally projects are being ruthless here, unless it is part of an DoS attack from an unprivileged source, it's just a "file a JIRA" problem. But they might get priority just to keep the AI tools quiet. 
3. vulnerabilities with no understanding of the security model. If "OS is subverted" is a pre-req then you don't need to attack the layers above.
4. Real vulnerabilities. Some of the AI reports are genuine; engineers have to fix them, and by their nature they aren't done in the usual open to review collaboration which makes for OSS dev efficient. 

As well as these third party reports, we also have our own AI tooling, specifically Anthropic's Glasswing project and similar, which tries to assist in finding vulnerabilities. While this gives us access to better models, allowing us to explore identified vulnerabilities, it also leads to a lot more reports which we have to triage into bugs and CVEs

Because of this expect all OSS projects to be releasing regularly this year due to their own newly discovered issues. Because of transitive dependencies, expect these same projects to also be trying to cope with updates to everything they depend on.

Which would you prefer those of us on the Hadoop security list to be working on, given there is a limited amount of time which can be allocated to this amongst all our other deliverables, and knowing there is nobody is this or any other OSS project who works full time on security:
1. Dealing with what we believe to be real CVEs in our code
2. Upgrading transitive dependencies which we know are real vulnerabilities 
3. Upgrading transitive dependencies of things where it's clearly not on any codepath we use
Right now, OSS projects do not have the capacity to cope with the amount of CVE related workload being dumped on us. 

If you really want to get dependency uodates, especially " hecklist cve updates" changes in on a timetable which works, you are going to have to be proactive here and with all the other projects that you consider critical parts of your supply chain
- supplying PRs which pass tests, ideally work in your deployments
- help test the releases
- help fix those bugs reported as potential CVEs but triaged down to normal issues.

Hopefully we can get through this. But i think we are going to have adopt a radical release process of 90% automated monthly releases with the option of interim OOB security fixes. 

