# Introduction

Ever felt trapped by your database?  This is the story of how one engineer's struggle with scaling, cost, and 
inflexible schemas led to the creation of YottaStore, a new approach to databases.  This series will document the 
philosophy, architectural choices, and (hopefully!) the growth of YottaStore, as we seek contributors and users.

# How YottaStore was born

It was 2020, and I was a backend engineer at a fast-growing unicorn startup. I *thought* I knew databases – after 10 
years, I hadn't faced a problem I couldn't solve with my trusty toolkit (MongoDB, Postgres, Kafka, and Redis). 
I was, of course, arrogantly wrong.

My company had just experienced massive growth, and then – the pandemic hit. Engineering leadership, facing market 
uncertainty, made cost-cutting a top priority. The biggest target? Our database infrastructure.

We were running a microservices architecture, each service with its own Postgres database. These databases had been 
scaled up *dramatically* to handle traffic spikes, and the costs were becoming unsustainable. Engineering leadership's 
solution: migrate to DynamoDB. New services would use DynamoDB from the start, and existing services would be migrated 
over time.

But the business still demanded new features and the agility that a startup needs. This clashed directly with DynamoDB's 
strict schema requirements. I was tasked with making *every* field in our non-relational data indexed and searchable, 
across massive datasets.

I felt stuck. DynamoDB scans were slow and expensive, and the limits on indexes (even tighter back then) were a major 
roadblock. I believed the high Postgres costs stemmed from technical debt accumulated during our hyper-growth phase. 
Forcing non-relational data into Postgres had led to rushed schema designs and poorly optimized queries, all in an 
attempt to keep business leaders happy.

In hindsight, I'm incredibly grateful that my engineering director resisted my push for MongoDB and forced me out of 
my comfort zone. I'm also glad I stumbled upon the fantastic book, 
[Designing Data-Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/), 
which greatly helped me in finding a solution.

# The first iteration: DynamoDB with Postgres

Reading the book helped me to get familiar with the Dynamo paper, its strength and weaknesses, and so I started to 
imagine how the schema could have worked. To help you understand what we're talking about we have several different type
of vehicles, with different models for each type, each of those combinations representing a different object shape that 
needed to be stored.

We needed to support both OLTP and OLAP like workflows on the same data: The people in charge of operations 
needed to be able to plan and optimize their work by prioritizing which cluster of vehicles most needed their services
and which route they should follow to reach their cluster (OLAP like), after which they would need to update the 
data about each vehicle involved in maintenance (OLTP like).

On top of all these problems we also have issues with the quality of the data, to make a simple example there was not
a single format to store dates, a similar problems appeared in most fields. Let me be clear: I'm not criticizing the 
engineers or the leadership: in a cash strapped and fast-growing startup you need to make though choices, and I'm proud
of having fought in the trenches with my peers.

While trying to juggle all these problems I had *two eurekas* moments: For the *indexes* I could have used postgres, 
where I would store only a normalized and sanitized subset of the fields, let's call it metadata. 
These tables, organized by geographic area rather than model, would be lazily updated from a stream of events, 
while Dynamo would be the source of truth. This way I could have fast indexed searches at a fraction of the cost and 
latency that dynamo would have offered, at the cost of having to deal with possibly slightly stale data.

To handle *scans* instead I took inspiration from the excellent paper by [Harris](https://www.cl.cam.ac.uk/research/srg/netos/papers/2001-caslists.pdf):
using the compare and swap semantics of DynamoDB I could implement wait free linked lists, which I could use to scan
a small subset of the data instead of the full table. Especially this idea sent me on a long technical trip
of exploring the potential of the Dynamo paper, and imagining what I could have achieved.

Mind that all of this was done in Node.js, and although I would have reused the same design at different companies
for different problems, I strongly felt the limitations of the environment, especially when dealing with in memory 
operations. Also, the fact that DynamoDB is not a faithful implementation of the [dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf), 
which by now I was in love with, bothered me.

# The second iteration: Dynamo and Golang

To handle the inefficiencies of Node.js I decided to try Golang, and to deal with a true implementation of the dynamo
paper I decided to try S3 as the underlying storage. This gave me much more room to breath and experiment, but S3 is not 
cheap, especially if you try to perform exotic operations, like storing partial indexes as records to speedup subsequent 
searches. 

I took a look at using NVMe attached storage as a cache, to save costs compared to always hitting S3, and this opened
me the world of linux syscalls like `ftruncate`, `writev`, `O_DIRECT` flags and many other nightmarish tools I needed
to use if I wanted to store efficiently the cached data. At this point I was building partial indexes in the cache, using
LSM trees to efficiently handle updates.

This is where I came up with the idea of making another large architectural jump: What if we dropped S3, and instead
used NVMe without a filesystem, treating it as the RAM in a very large distributed machine. After all NVMe disks are 
0-indexed arrays of 4 kb cells of memory, which can be randomly accessed, exactly like RAM albeit with slower performance.
In this model I could reuse the large academic literature about in memory wait free algorithms, imagining a collection

What I needed was a highly performant interface to handle highly concurrent operations on NVMe and network. In my help
came the [NVMe ZNS specification](https://nvmexpress.org/specifications/), which provides a rich API to use NVMe storage
to the best of its performance, and the [io_uring](https://github.com/axboe/liburing) linux kernel interface

The problem was that making `io_uring`, the NVMe API and golang work together nicely proved to be extremely
difficult. I eventually made it several months later, but Golang forced me to write a lot of unidiomatic code, 
for example to handle memory aligned operations.

And so it seems I was stuck again...
