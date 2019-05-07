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

# A Common Input Stream Cache for Object stores

Proposed: 

1. We have one.
1. It implements an aligned page block design, as done in Azure WASB (and apparently in google GS).
1. We choose a page block size which works well with ORC and Parquet read patterns.
1. We could consider an async prefetch of the next page, assuming a forward read pattern.

The strengths of this design are

* Reduced cost/time of reading data already in page.
* A shared design across stores means one implementation to get correct.
* And one implementation to tune for the IO Patterns which can be collected
from traces of queries.

If/when a vectored IO input stream API is added, we need to plan for it to
work with this. 

Obvious first step: if in cache, return immediately.

But what if it is not in that cache, or only partially in cache?


1. read new pages in and then fill in response. Best if subsequent reads
will be using the same pages.
1. Or: assume app really knows what it is doing and bypass the cache to retrieve
all missed data. That is: only ask for the ranges and don't cache.

Strategy #1 would seem best if the subsequent reads were likely to read
from the adjacent bits of the file. We are assuming locality of reads.

For an seek+read input source this probably holds, at least with non-columnar
data sources.

Strategy #2 says: the read patten is explictly declared in the API sequence.

That is: if there is locality, it will be included from the list of read
operations, *and if not so declared, there is no locality*.





