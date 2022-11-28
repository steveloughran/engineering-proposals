
---
Key concepts

Object stores can be slow to interact with because of the often-limited bandwidth between Hadoop clusters and remote stores, the latency of HTTPS requests, and because IO request throttling can place an upper limit on how many reads, writes, renames and deletes can be performed against a store per second. 

Creating large distcp deployments with many workers can be I counterproductive. As well as competing for bandwidth, the many workers may overload the remote store. The cloud stores respond with 503 "Slow Down" responses, which recognized in the clients and trigger an exponential back off and re-try process. As a result, performance across the entire cluster may collapse.

It is also Worth being aware that the object store clients use multiple threads for parallelized uploads and downloads of blocks of data, for better performance. The default values here our optimized four Data analysis jobs querying ORC and the Parquet data. It is possible to chew need settings for better performance with distcp. To make the most of this parallelization, you need to request distcp workers with multiple cores allocated to each worker. With the right configurations, the S3A and ABFS clients can fully utilize the allocated cores. 

It is generally better to use fewer multi-core distcp workers than many single core workers.


The queuing of blocks from a single file to upload responds better to throttling and bandwith limitations.

It reduces/eliminates the risk of slow ABFS uploads from timing out. Every block uploaded to Azure storage through the ABFS connector must be uploaded within 90s. With heavy contention for CPU time or bandwidth, uploads may fail.

Clients which support parallel block download (currently the ABFS connector) benefit equally well when downloading from the cloud store. There are some settings which need to be changed for this -they will be covered in a later section.

Other details to be aware of

Checksums 

Object stores which do not have checksums compatible with HDFS do not use checksum comparisons during updates. Instead, file length alone is used to decide whether or not to upload a file. If a source file is longer or shorter than that's at the destination it will be uploaded. If the length is the same it will not.
In all the distcp examples in this document, -skipCrcCheck is set to make clear that there's no expectation of checksum validation. This is not currently needed for distcp operations to any of the object stores -however, were they ever to support checksums, then, unless those checksums were compatible with HDFS, the mismatch




------------

