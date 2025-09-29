# Notes on database architecture revue


# The goal
Design a distributed database which dynamically scales itself in order to remain within resource budgets.

## Resource budgets
* Energy consumption (i.e. kwh during the day)
* Peak demand (i.e. max W of demand from a server)
* Monetary Cost (i.e. scale performance up and down to meet a set budget constraint)

# The pitch
It is generally assumed that 15-25% of the annual cost of operating a datacenter is power. Additionally, large-scale datacenters often require power guarantees that are not always available within specific regions, limiting the siting capability of these datacenters. At the same time, the large number of customers which use cloud computing pushes for ever more complicated structures for multitenancy, allowing many customers to create and destroy databases according to a wide variety of different rules. When using a traditional database, these two constraints work in opposition to one another: the need for multitenancy results in many individual database systems being constructed, while the need for managed power limits the capabilities that each database can offer.

We propose a new database design which seeks to harmonize multi-tenancy with resource management. It provides global, cross-datacenter transactions and extreme multitenancy, allowing for a wide range of workload scalability. At the same time, the database scales its performance structure dynamically, so that the system as a whole can operate within set cost parameters.

# Desired features
* SQL-like interface
    * Optional document-esque approaches? Nested lists, arrays? Could be problematic, and leads to optimizer challenges
* Query planner
    * Must take resource budget into consideration
    * Should also take distribution into consideration
* Multi-datacenter reliability
    * (Optional by configuration)
* Massive multi-tenancy
    * Use blueprinting to allow dynamic schema evolution
* SERIALIZABLE transactions
    * They are stupid, but its a nice selling point
    * Should allow lower forms of isolation (like SI or RC)
* Transparent data movement and rebalancing
    * it should be easy, and (relatively) painless to move data from one location to another
    * Include easy data export? 
    * Partial data backup?
* Transparency in cost management
    * Should be able to tell exactly where the budget is being used

# Possible additional features
* Pluggable storage engines
    * Parquet
    * RocksDB
    * BTree
* Multiple Index forms
    * Materialized views?
* Columnar storage
    * would need to be optional, to allow flexible data models
* Strong security

# Anti-features
Stuff that this system is just not going to be good at, and we aren't going to try
* Data Analytics
    * The system will be oriented towards OLTP workloads. Large scale data analytics is probably not going to work that well
* Graph processing
    * Use a graph database, or export data and do in-memory processing.


## Databases to think about
* Google Spanner
    * https://storage.googleapis.com/gweb-research2023-media/pubtools/1974.pdf
* Amazon Aurora
    * https://assets.amazon.science/dc/2b/4ef2b89649f9a393d37d3e042f4e/amazon-aurora-design-considerations-for-high-throughput-cloud-native-relational-databases.pdf
* Alibaba PolarDB
    * https://www.vldb.org/pvldb/vol18/p5059-chen.pdf
* Microsoft Socrates
    * https://www.microsoft.com/en-us/research/wp-content/uploads/2019/05/socrates.pdf
* Calvin/ FaunaDB
    * https://www.cs.albany.edu/~jhh/courses/readings/thomson.sigmod12.calvin.pdf

## Consensus protocols
* Pineapple: 
    * https://www.usenix.org/system/files/nsdi25-bantikyan.pdf
* EPaxos: 
    * https://www.usenix.org/system/files/nsdi21-tollman.pdf
* Raft
* FLEET
    * https://www.vldb.org/pvldb/vol18/p1522-fan.pdf
* Alvin
    * https://www.ssrg.ece.vt.edu/papers/opodis14-alvin.pdf

## Papers about Resource efficiency
* Proactive Energy Management in Database Systems:
    * https://hotcarbon.org/assets/2024/pdf/hotcarbon24-final111.pdf
