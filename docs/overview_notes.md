
# The problem:
* Datacenters have very high power demand.
    * Power demand is asymmetric, and unpredictable
        * Traffic can raise and lower unpredictably and very quickly
        * The presence of backup generation(particularly batteries) may mean that datacenters drop from the grid and rely on batteries based on internal power quality metrics. This results in load "dissappearing", which is hard for grids to deal with.
    * Power is an expensive part of datacenter operations
        * at 50$/MWh, a 1.5 GW datacenter can be paying up to $75,000/ hour to run at full load. Over 24 hours, that is $1.8 million _per day_.
        * This does not take into account backup generation costs
        * Some of this is alleviated with on-site solar generation and storage, and hedged against with low-cost renewable PPAs (where applicable), but the cost is still incredibly significant.
    * Power demand translates to cooling demand
        * large datacenters require large amounts of cooling to deal with energy loss. This cooling is sometimes (although not always) water-based, which means datacenters use a lot of water.
* Power demand is a function of hardware x usage
    * Different pieces of hardware have different power consumption profiles, so changing the configuration of the datacenter can result in wild swings in power demand over the longer term.
        * Network routers have high power demand, but it is relatively inflexible. Routers do not use much more power when running than when idle[Tech Carbon Standard]
        * CPUs radically scale their power consumption in proportion to their utilization[Multicore vs Manycore]
        * GPUs are _extremely_ power-hungry, because they use hundreds of processing cores when in operation, making them have multiples of CPU power consumption.
    * Hardware uses power in proportion to it's demand--that is, to how much and how often it is called upon to perform. This is a software problem. 
        * The quality and nature of software ultimately determines power demand, when scaled to the datacenter level
* Software is not engineered for power efficiency
    * It is almost universally ignored when considering power tradeoffs.
    * Very few tools exist to measure it, and even fewer techniques exist for optimizing it.
        * It is telling that the primary advice for developing "green software" is a variation on the same themes as everything else: Make it faster, less resource intensive, and smaller. It is essentially the \*nix philosophy laundered into "green" form. 
    * Considerations for the power consumption of algorithmic and software design choices are purely academic, with almost no regard made in the real world.
* Software design choices have immediate power impacts
    * Distributed systems have a higher power footprint than non-distributed ones
        * This is pretty obvious, but partly it is due to just having more hardware running. More servers, more routers, etc.
        * However, distribution is widely considered a "safe" form of computing, in that it can lead to more reliable software.
    * Cloud-based systems often share physical hardware (virtualization) which makes it harder to isolate power consumption to a specific software project
* Not all software has the same power impact:
    * Networking software may not have a huge impact on overall power demand, because routers have relatively constant demand
    * Storage devices also tend to not have a large spread in demand[Backblaze]. Additionally, they do not draw much power overall, regardless of utilization rates.
    * RAM is generally quite low in power draw, only drawing up to 1.2 W under load[SoftHandTech]
    * CPU energy costs scales dramatically with utilization[Multicore vs Manycore]. The power consumption can double or triple when in use (depending on load), and the number of CPUs scales linearly with the number of computers. 
    * Therefore, software operations that reduce CPU usage are the most likely to be effective at reducing power consumption.
        * This is an explanation for why C/C++ consistently shows itself near the top of the leaderboard for most "energy-efficient" code, while python shows at the least. All else being equal, a heavily optimized, CPU-friendly algorithm will use less energy than a python one, 
        just because of the design of the language
    * Concurrency also plays a major role in power consumption, because each additional hardware core that is engaged scales up the power consumption of the CPU. 
        * This is doubly true for a GPU, which makes much heavier use of parallelism, allowing far greater power consumption. 
            * For example, the GeForce RTX 5090 consumes up to 575 W under load[EatYourBytes GPU], while an Intel Core i9-14900KS consumes up to 253 W[EatYourBytes CPU].

* Thus, in order to lower the power consumption of a piece of software, we can devise a few solid rules:
    * Reduce concurrency
        * Use the GPU _much_ less
        * Use the CPU less
    * Favor RAM over disk whenever possible (energy reduction factor of ~5)
    * Favor disk over networking (because networking _typically_ requires more CPU, but also because less networking means fewer routers)
    * Improve CPU utilization:
        * Algorithms which are better designed for your load
        * Reducing steps and unnecessary activity
    * Try to use fewer computers

* When working in a datacenter, however, there are new complexities which work against the above efforts:
    * Replication:
        * Distributed systems tend to replicate data, and (sometimes) replicate computation as well. 
            * For example, a database may make multiple copies of the data on different machines, and each machine will perform pre-processing steps which duplicate CPU efforts.
    * Sharding:
        * Storing large data sets across multiple servers involves _sharding_: splitting that data into multiple groups and distributing those groups around to multiple different machines. 
            * When the dataset is _truly_ large, this is necessary for the functioning of the system as a whole.
    * Machine Utilization
        * For reliability reasons, server administrators rarely want computers to run at 100% load, because of a number of problems
            * Poor performance: An overloaded server will result in slower responses
            * Instability: Under heavy load, many software systems will fail (due to design or poor development, either one). 
            * Cascading overload: A server under heavy load may cause a different system to enter heavy load also (such as a router backing up because a destination server is not receiving messages quickly enough). When this happens, connected systems may also demonstrate instability, leading to the dreaded "cascading failure"
    * The net result of these three factors is that datacenters generally have _a lot_ of servers, but with each server operating within a range of power draw scenarios, from a minimum (idle) up to the administrator's determined "max utilization". 
        * 2 machines running at 50% load will draw ~ the same amount of power as a single machine running at 100%. 
        * 2 machines idling will run at more than a single idle machine.
        * This creates an engineered incentive to run each machine at as close to it's "optimal" power draw as possible
            * However, the uncertainty of actual operation counts makes this very difficult (nearly impossible) to determine.
    * There are basically two solutions to datacenter power draw reduction:
        * Use fewer machines 
            * This requires a different design of the actual software system being made, and may involve tradeoffs either in replication or sharding. 
            * However, this is where software design strategies can come into play. If a distributed system is twice as efficient, it may require fewer machines to run.
        * Ensure that the machines you do have are running as close as possible to their optimal utilization rate _at all times_. 
            * This is the strategy behind _virtualization_ -- i.e. "the cloud". If you rent server hardware, then you can share hardware with other customers. If the administrators balance the load correctly, then more machines will run close to optimal power draw, resulting in less overall power usage for the same workloads.

* Amazon AWS, Digital Ocean, Microsoft Azure, Google Cloud, etc. all have an engineering and financial incentive to keep their "cloud" datacenters running at optimal capacity, because 
    * It means more customers using their products, creating more revenue
    * It amortizes the power cost across more customers, lowering the marginal cost of operations.
* However, no cloud provider has an incentive to reduce total machine utilization, as it lowers revenue
* Software engineers seeking to use datacenters have an incentive to lower their machine utilization, because it lowers their operations bill. 
    * Conversely, however, software engineers are expensive, and time spent improving the efficiency of code is money spent on salary, HR, etc. For a smaller software shop, it can be more expensive to optimize their product than it would be to just pay the cloud provider bill.
    * There is also little incentive to improve efficiency overall, because efficiency is not a saleable feature for most software products. More revenue can be made by paying the cloud bill and focusing on saleable features.
    * These above facts are especially true in highly competitive product environments, where distinctions are largely made on provided features rather than quality of delivery.
* Cloud products often over-offer their features. If you want a hosted database for your small data set, you mmay only _need_ a single machine, but you'll often _get_ a replicated solution which magnifies the energy cost. These over-offers are more convenient for the cloud-provider, who can sell just a single product, but result in higher overall energy
consumption by individual software shops.

* "AI" datacenters, by contrast, operate in a very different software design parameters. "AI" as used by OpenAI et al uses machine learning algorithms to create their end product. Machine learning algorithms are often (but not always) _trained_ on their data. That is to say, there are two stages to creating an LLM model like OpenAI or Google Gemini
    * Step 1 is to _train_ the system. This is a fancy description for allowing the system to make an extremely large number of guesses about what the correct answer to the problem is. It is then told, after each guess, whether it is correct or not. The more guesses it can make, the more "correct" it can become (although there are mathematical limits). After the system has been run on enough data that the administrators decide it's good enough (or that it's not going to get any better), the results are shipped out to answer queries
    * Step 2 is identical to any other data-backed request system. A user sends in requests, and the trained model is employed to provide answers. 
* Step 1 is actually _significantly_ more expensive than Step 2 in terms of energy (and computing resources more generally). The algorithm might well end up making billions of training guesses before passing, or it might fail and need to be adjusted and tried again. This means _enormous_ amounts of computational work.
    * If it were done on a normal CPU, it would probably never finish. 
    * However, the training problem is _embarassingly parallel_, meaning that you can basically just throw more computers at it to make it faster. 
        * Even better, you can throw GPUs at the problem, which is even _more_ efficient at solving the problem.
    * But the volume of data used during training is so large (basically the entire internet) that even with datacenters full of GPU-centric machines, the training process is quite slow. Therefore, the _bigger_ the datacenter, the _faster_ you'll train results.
* The net effect is that you need a huge datacenter full of specialized computing equipment designed to make a _single problem_ go faster.
    * The power profile for these datacenters is significantly different from that of a cloud-computing datacenter. Instead of having a large number of small customers sharing resources and (ideally) bringing machine utilization to it's optimal power draw for the maximum amount of time, you have datacenters which spin up, run extremely power-hungry algorithms
    for a while, then spin down when the training process is done/cancelled/whatever. Instead of a near-flat power draw profile, you have a large bump profile, where power rises much higher, then falls much lower when machines are taken offline. 
* These datacenters are also constructed differently internally, under different pressures:
    * In a "cloud" datacenter, the number of servers which are placed has countervailing pressures:
        * The more servers you run per datacenter, the fewer datacenters you have to manage, which lowers costs. This pushes datacenters larger
        * The fewer datacenters you have, the larger the impact if one of those datacenters goes offline for any reason. This pushes for _more_ datacenters
        * The larger the datacenter, the greater the risk that you have unused machines, which counters the entire purpose of cloud computing. This may result in wasted resources(and power) per datacenter. This pushes datacenters to be smaller
        * Thus, a "cloud" datacenter's multiple countervailing forces tend to push them into trading off size of individual datacenters versus the number of datacenters, which results in a kind of equilibrium size.
    * Training datacenters have no such countervailing forces:
        * The more machines you can throw at training, the faster training becomes. Thus, the incentive is _always_ to push datacenters larger and larger, up to the limits of your capital expenditure budget.
    * Similarly, a "cloud" datacenter is often made of commodity hardware, both for cost reasons and because "commodity" is generally what most of their customer need. 
        * Because administrators (try to) run their machines at below-100% utilization, there is also no need for water-cooling systems. Instead, they can maintain temperature by carefully laying out server racks and managing their HVAC systems appropriately.
        * Many general purpose servers do not have GPUs, because a server rarely has to do graphics processing, and because GPUs are very expensive to run in a sever environment.
    * AI servers are special-built machines, packing late-model GPUs.
        * They are also run full-tilt during a training run. There is no need to avoid utilization, instead you want as close to 100% as possible, which means higher power draw and more heat. 
        * That heat forces water-cooling, which cannot be converted to air-cooling.
* The end result is that AI datacenters have wildly different design parameters than your typical cloud datacenter:
    * They are larger
    * They use water-cooling systems which cannot be swapped to air-cooling without changing the built design
    * The workloads are larger, draw more power, and run less predictably
* A cloud provider does not want a 4.5 GW datacenter, because it would have too high a risk profile, both financially and from an engineering risk perspective.




# Notes and References
[Tech Carbon Standard]: https://www.techcarbonstandard.org/technology-categories/networks
[Multicore vs Manycore]: https://link.springer.com/chapter/10.1007/978-3-319-43659-3_40
[Backblaze]: https://www.backblaze.com/blog/storage-pod-5-0-hack/
[EatYourBytes GPU]: https://www.eatyourbytes.com/list-of-gpus-by-power-consumption/
[EatYourBytes CPU]: https://www.eatyourbytes.com/desktop-cpus-ranked-by-their-max-power/
[SoftHandTech]: https://softhandtech.com/how-much-power-does-ram-use/
