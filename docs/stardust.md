# What is Stardust?

# Abstract
Stardust is a globally distributed database which is _resource-constrained_. Specifically, Stardust is built such that the resource consumption of any datacenter within its system can be optimized to remain within a bounded resource constraint (as best as it is able, at least).

# The Problem
Distributed databases use a lot of resources:
    * It requires lots of servers to handle storage and workloads
    * It requires lots of active intervention and care to keep resource utilization optimal
        * Constantly tuning stuff
    * It requires tons of _electricity_
        * And that electrical demand is highly inconsistent as well, rising and falling with demand. Makes it very hard to get favorable power rates, and could cause datacenters to be forced into non-firm power contracts in the future.

A lot of this resource consumption, especially in a multi-datacenter database system, are partly or wholly wasted.

For example:

A common multi-DC replication strategy is 3 datacenters each having either 2(replication factor 6) or 3 nodes (replication factor 9). If you have a bunch of small databases, each one replicating to 6 nodes in 3 datacenters but with barely any traffic, you end up with 5 idle machines, which is very wasteful. To get around this, you try to share machines
through virtualization/containerization/etc. but that has it's own share of problems:

* Unbalanced traffic between clients results in different users seeing vastly different latencies (and sometimes problems)
* Containerization is a very blunt instrument for resource management. It allows no fine-tuned resource integration
* A computer which is fully-utilitized 100% of the time wears out faster, so you have higher machine costs.

So one way or another, you either end up wasting resources, or spending lots of human-time trying to get the right balance of these systems.

Zooming out, one of the largest datacenter costs is electricity: 15-25% of datacenter's operating costs are on electricity, and those high loads prop up expensive and dirty power generation technology like natural gas. 

And fundamentally, the underlying software that you're trying to run just was never designed to be resource-constrained in _any way_. The assumption is _always_ that the only resources which matter are servers: CPU, memory, disk, and network. But cost, electricity, and time should _also_ be taken into account.

Stardust aims to resolve this problem

# The goal
Stardust has three main goals:
* Geographically distributed. You can run the same Stardust instance across multiple datacenters that are geographically dispersed.
* Transactionally correct. You can rely on its transaction system to perform correctly
* Resource-managed. The system will dynamically adjust itself in order to stay within externally set resource constraints.

## Example situation
( This isn't necessarily a highly _feasible_ example, but it's a good illustration of the goal).

Suppose that you want to run a globally distributed database _purely on solar power_. In order to do this, you either need to
1. Shut off the system when the sun goes down
2. Move workloads to different datacenters across the continent(world?) to follow the sun

A way to approach this, broadly, would be to design your physical infrastructure as follows:
1. Somewhat overbuild your solar arrays, and add in batteries.
    * The batteries provide power stability
    * They can also be used to provide a much _lower_ level of power over the dark hours.
2. Place datacenters at different longitudes, to take advantage of the east-west movement of the sun 
    * The system then must move its workload from east to west, as the sun moves in the sky
3. Design your system to move itself into a "low power mode", which operates with much worse performance, but can keep up its reliability requirements while still using less electricity than you have contained in the battery-banks.

You cannot make this work with existing database designs: 
1. There is no tooling to actually tell you about resource consumption outside of literally looking at the wattage being used on a given server rack
2. Reducing power consumption on a per-node basis is _at best_ a brute-force exercise of using containerization limits. This is an inherently guess-and-check approach. "If I reduce CPU on this set of nodes, will that reduce power? Nope. Should I turn them back up then?" And so on.
3. There are no easy, out-of-the-box mechanisms that route actual customer requests based on the resource consumption _of the destination_.
4. Moving actual workloads required potentially dangerous, random, _manual_ processes:
    * If the leader in a leader-follower design needs to move to another datacenter, it's a non-trivial task to ensure it ends up in the _correct_ datacenter

# The Major Features
Stardust aims to solve (or at least improve) these problems:

1. Allow the setting of global, datacenter-wide resource constraints, which Stardust will seek to manage.
2. Provide dynamic, resource-aware routing for customer workloads
    *  automatically move customer workloads to different datacenters in order to reduce resource consumption
3. Dynamic resource management
    * _Automatically_ increase and reduce thread counts, memory consumption,and other resources on a per-node basis in order to keep the datacenter within a constrained resource budget.
4. Provide increased tooling to view how each component contributes to resource utilization, allowing for more strategic decisions.

Of course, Stardust is still a database, so it still has to provide certain baseline features:
* SERIALIZABLE transactions
* Multi-datacenter reliability ( other features aside, if you're going to move workloads between datacenters, then you need to be able to operate across multiple datacenters)
* Effective query planning and execution

## A note about performance
It is commonly held that performance is tightly coupled to available resources in a distributed system. Or rather, that it is impossible to have both "scalability" _and_ resource constraints. As you increase the overall workload necessary, you will eventually hit resource limits which force you to either increase resources or limit scalability.

This is probably true. There are definitely tricks that Stardust will need to engage with in order to reduce this. Things like really efficient code, good query optimization, etc., but fundamentally you cannot sustain infinitely large workloads with finite resources. Stardust does, at the end of the day, favor staying within resource bounds _over_ performance. It probably makes the entire project a bit of an academic exercise, but that's what we're doing.





