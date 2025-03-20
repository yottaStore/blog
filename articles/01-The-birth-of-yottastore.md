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

# The second iteration: Dynamo in Golang

# The Rust Renaissance
