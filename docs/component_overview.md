# Stardust Architecture

# Main components
* Consensus/Transaction
* Resource Manager
* Routing
* Schema management
* Query optimization
* Client Interaction

# Transaction structure
The premise of Stardust is to reduce the physical footprint of data management within datacenters as much as possible without affecting reliability (although obviously performance is fair game). The biggest resource management technique is to move workloads from serving location to serving location, which means multiple datacenters. To that end,
we need a database which is multi-datacenter aware, and can maintain correctness even within multiple datacenters.

At the same time, a database is not a generic datastore: There are required features which can't really be sacrificed. In particular, transactions are a necessity. 

There are basically three (modern) designs for this:
* Google Spanner
* Calvin/FaunaDB
* Amazon Aurora

(there are other designs, but the above two are the ones that are the most coherently structured)

Here's a brief overview of the architectures. I'm being super brief here, and so broadly oversimplifying the architecture.

## Amazon Aurora
Aurora adopts the _disaggregation_ design. Essentially, it takes the integrated MySQL stack and splits into three: Query answering, log replication, and storage. When data is written, it is logged to a distributed log, and separate machines read from that log to build tables and indexes locally. When a query is issued, a local replica of built tables is read from. To make all this work, Aurora adopts a primary-secondary design model where only one primary exists to take writes, and "up to 15" read-only replicas can be used to scale query performance.

There are some distinct advantages to this design:
* They didn't have to write a new query planner (arguably the hardest part of a database)
* Log replication is a relatively easy problem with lots of solutions available (Raft is the most popular and it works very well, especially with primary-secondary designs that don't care about Raft's own primary-secondary design)
* In _principle_, you can scale storage separately from query performance
    *  I've always been skeptical that this is a truly meaningful advantage. The question that comes to mind for me is: does storage _actually need_ to scale independent of execution? 
        * The only way to get to storage is through the execution engine, which means that there will always be at least as much traffic heading to the execution engine as to storage.
        * In general, query execution is more expensive than data storage, because joins, sorting, etc. are all expensive operators relative to a table scan (this is why it's so useful to use secondary indexing as sort-order projections, because avoiding a sort is a Big Deal(TM)). So are you more likely to need more storage nodes, or more execution? This is an argument for leaving storage alone and scaling the execution engine, I suppose. But Aurora limits read-only replicas for practical reasons, so there is an inherent limit on this approach.
        * Still, the disaggregation model has some valuable insights to take away. Is there a way to reduce the amount of duplicated effort in the system? This is a good performance advantage, but also would reduce resource cost across the board and would make resource management easier.
* Multi-tenancy is fairly easy:
    * If you want a new database, turn on another of these


There are also some distinct _disadvantages_ to the design:
* Using an existing query planner means that the planner isn't aware of the physical layout of data on the network. 
* It leaves some data-specific optimizations on the table (for example, doing in-situ joins for co-located data)
* It doesn't perform any explicit sharding, so while the _storage_ size is theoretically unlimited (via the replicated log structure and scaling storage nodes), the execution engine likely imposes limits on how much data any single instance can actually manage.
* It's operationally more complicated:
    * How to configure? Optimize?
    * How do you scale different nodes?
* There are more sources of error:
    * The interface between execution and log replication
    * The interface between log replication and storage
* Multi-tenancy is expensive:
    * Spawning a new database system involves bringing an entire new system online.


## Calvin / FaunaDB
FaunaDB is the production version built on the Calvin design model (sadly, the company supporting FaunaDB closed its doors in March 2025). Calvin adopts a single-pass transaction model: first you perform a read(which gives you a timestamp), then you attempt a write through log replication (which gives you a different timestamp). Once the write is replicated,
another local read is performend--if the version of the local read is higher than the first read version, then there is a conflict and the transaction is aborted. 

This is an extremely clever design which leverages log replication to create a serializable ordering of writes, at the expense of additional _local_ reads. The result is a high-throughput write structure without requiring a lot of funky operational compromises.

The advantages:
* It is a high-throughput design
* Multi-datacenter support is built in through log replication (using Raft, which is pretty well supported and quite fast)
* Transactions don't have to interact with each other to detect conflicts

The disadvantages:
* There is no inherent mechanism for sharding--if you want to shard data, you just have to run multiple instances and use the application level to ensure that no transaction crosses the boundary between the two
* There is no associated query planner -- gotta build your own
* There are some odd optimizations (like a 10ms batch wait time) which improve overall throughput, but create latency barriers
* Multi-write transactions are non-trivial to implement
* Raft is a primary-secondary model, which can cause problems at scale. This is not as big of a problem as you might think, because pretty much all the other approaches will have a similar architecture.
 
All in all, the Calvin model could be treated as an effective _building block_ for a larger-scale distributed system, but it lacks most of the other pieces.

## Spanner
Spanner, like Calvin, is also (at root) a transaction model, rather than a full database construction. It uses a very traditional transaction structure of timestamped transactions:
when you create a transaction, you get a timestamp which is an _actual machine clock timestamp_. Reads and writes are always performed as of that timestamp (with some allowances for read-your-writes). At commit time, the system does a conflict check (similar to how FoundationDB works) to ensure no conflicts, then obtains a commit timestamp. This timestamp has some allowed "smear" to understand clock uncertainty, which the commit _waits_ for. Once it is not possible for _any_ machine in the cluster to get a commit timestamp that is in the
window of uncertainty, it performs the actual commit. 

Spanner explicitly allows for sharding, and uses two-phase-commit to perform transactions which cross shard boundaries.

This design has a lot of inherent advantages:
* The transaction model is relatively understandable: If you can work with Postgres's transaction model, you can work with Spanner
* The built-in multi-tenancy allows for global distribution (in the paper, they mention how they really only wanted a _single_ Spanner instance worldwide)
* Cross-datacenter support is built off the building-block transactions: just put replicas in multiple datacenters
* Spinning new databases up is mostly a matter of finding an open cluster
* Scaling system size is straightforward: just add more nodes.
* No disaggregation means that you can optimize locally _and_ globally.

There are disadvantages too:
* The clock dependency causes _very long waits_ if clocks do not have tight synchronization boundaries. 
    * In 2012 (when the paper was written), this was a big deal because most datacenters didn't have access to that kind of high-quality time syncronization.
    * In 2025, this is significantly less of a big deal: Amazon offers clock-syncronization through PTP with 100μs syncronization. It's some work to measure clock uncertainty and thus create true-time bounds, but it's very reasonable to assume tight clock bounds in the modern datacenter.
* There are no additional components: 
    * No query planner
    * No execution engine


# Resource Manager
Recall that the entire point of stardust is to dynamically manage existing resources to balance cost(i.e. power consumption) against performance through active
management of resources. This means that the resource manager will take in information from individual servers, and feed back instructions on those servers.

In theory, the resource that we are attempting to balance could be anything:
* average energy consumption (i.e. Joules, MWh, etc.)
* peak power demand (i.e. Watts at peak)
* Actual cost (i.e. some aggregated dollar cost function).

We are attempting to maximize database performance _subject to_ the stated cost restrictions. As part of this design, we do not think of costs as being fixed with time. In fact,
they can vary significantly over time, as the real and actual costs of operating a datacenter do. We think of these cost functions as dynamic rate limiters.

For convenience, let's describe a few terms here:
* "Capacity" is the amount of computing resources available _at a given moment in time_ for _a given computing entity_. A single server can have capacity, a cluster of nodes has capacity, and the entire datacenter has capacity. The entire _system_ has capacity. 
* "Power" is the amount of computing resources _currently being used_ by the entity. 

So, to take an example, suppose that we use peak electrical demand as our cost function. Break time into windows that are T seconds wide (for example, 10 seconds, 100 seconds, etc.). Within each window, our cost function `C(T)` is the peak electrical demand that the system is allowed to use during that time. If we are imagining that we are running a datacenter
entirely off of a solar array that generates 1MW at noon, and no power at all from 8PM - 8AM, then we could break our time window down and say that `C(T)` is our solar-array's peak output during that window (obviously, this is simplified--you'd need to use batteries to actually align peak demand and peak production within `T`, but ignore the electrical engineering for now)

So given that, the _capacity_ of the datacenter running on that solar array is the total amount of computing _of any kind_ that the system can use in order while staying under the
cost function `C(T)`. And then from there, we can say that the _power_ is the amount of computing that is _actually_ being used.

Obviously, power `P` is always \leq capacity `Ca`, because you can only use the computing power that is within the capacity of the datacenter. 

Stardust's resource manager aims to maximize `Ca` while keeping it below the total cost function `C(T)`. In a very practical sense, this is not possible at _every_ moment: There is a time lag between the resource changing and the capacity updating itself that cannot be avoided. This is most obvious if you refer back to the solar-array example. If the solar
array is obscured by a cloud for 30 seconds, then it will see a significant drop in electrical generation. If we were to keep the datacenter always running under solar generation,
then it would need to _instantaneously_ react to adjust resources, which is not possible within the constraints of network traffic. 

Instead, stardust aims to make the best effort it _can_, and attempts to adjust after the fact to account for overshoots. Using the solar array again, if the solar generation
drops briefly, Stardust will rely on some other source of electricity, then attempt at some point in the _near_ future to adjust downward for stability. When it adjusts downward,
it should attempt to be conservative, in order to "net out" the cost function when that makes sense (i.e. drop electrical consumption below necessary for a minute or two to allow
the batteries to recharge from the excess solar, in our example). The design cannot be perfect, but it should make its best possible effort.

Of course, there is a minimum _required_ capacity that the system requires: Machines must be available to service requests, which involves a certain fixed level of consumption that cannot be avoided. The resource manager will not shut down computers, only reduce what the database will attempt down to its minimum.

However, there are a _lot_ of different strategies that the resource manager might attempt:
1. Move client workloads from a node with low capacity to a node with higher capacity (to make better use of a constrained resource)
2. Move "idle traffic" (that is, data regions with very very low traffic/ low resource requirements) to regions with lower capacity, so that higher-resource-requirement
systems may use the available resource. 
    * A good example are idle databases. If the database is sitting idle, then it has no traffic. Since it has no traffic, we move the routing responsibility so that
        it routes to a datacenter with low capacity. This allows us to free up high-capacity resources for other traffic without actually harming a customer. Then, when the customer
        load picks up on that database, we can move it back (or hopefully it just correlates with a higher capacity where it is)
3. Spin up/down the number of client requests that a server can do simultaneously (i.e. reduce/increase network threads).
    * Reducing threads means the server will do less work overall
    * Increasing threads allows the server to do more

## Techniques that may be useful
* `docker update`. When using containers, `docker update` can be used to update CPU and memory resource availability in real-time. This is a bit of a big hammer approach
to resource management, in that it doesn't really address the actual systemic components. It can also have bad consequences if the underlying application isn't prepared for it (sudden deadlocks can sometimes result if you steal a core out from under an app, and if the data held in RAM hasn't been logged properly data could be lost if you take away memory).


# Query Optimization

# Client
