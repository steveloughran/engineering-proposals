# library updates

I never share any details of the security list, except here are my observations of it and those of what I know of other projects, as sent to someone whose team were asking for the timeline for an update of a dependency with a CVE on the release we ship with.
That's a CVE to which our code isn't exposed, BTW

Which would you prefer those of us on the Hadoop security list to be working on, given there is a limited amount of time which can be allocated to this amongst all our other deliverables, and knowing there is nobody is this or any other OSS project who works full time on security:
1. Dealing with what we believe to be real CVEs in our code
2. Upgrading transitive dependencies which we know are real vulnerabilities 
3. Upgrading transitive dependencies of things like that thrift 0.23 update where it's clearly not on any codepath we use, such as CVE-2026-43870 Apache Thrift vulnerable to Path Traversal, HTTP Request/Response Splitting, Uncontrolled Resource Consumption

Right now, OSS projects do not have the capacity to cope with the amount of CVE related workload being dumped on us. 

If you really want to get those changes in on a timetable which works, you are going to have to be proactive here and with all the other projects that you consider critical parts of your supply chain
- supplying PRs which pass tests, ideally work in your deployments
- help test the releases
- help fix those bugs reported as potential CVEs but triaged down to normal issues.

Hopefully we can get through this. But i think we are going to have adopt a radical release process of 90% automated monthly releases with the option of interim OOB security fixes. 

