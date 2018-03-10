## A NOTE ON USING COOKIES
For `POST` requests, make sure that your endpoint requests use cookies, explicitly. CC requests have been redesigned as async calls to avoid blocking. Hence, if you don't use cookies, you won't be able to see the server response if it takes longer than a predefined time (default: `10 seconds`) even the session has not expired, yet (default: `5 minutes`). Also, such requests will each create a new session and excessive number of ongoing requests will make CC unable to create a new session due to hitting the maximum number of sessions limit. `GET` requests that are sent via a web browser typically use cookies by default; hence, you will preserve the session upon multiple calls to the same endpoint via a web browser.

* Here is a quick recap of how to use cookies with `POST` requests using `cURL`:
1. Create a cookie associated with a new request: `curl -X POST -c /tmp/mycookie-jar.txt "http://CRUISE_CONTROL_HOST:9090/kafkacruisecontrol/remove_broker?brokerid=1234&dryrun=false&kafka_assigner=true"`
2. Use an existing cookie from the created file for a request that has not completed: `curl -X POST -b /tmp/mycookie-jar.txt "http://CRUISE_CONTROL_HOST:9090/kafkacruisecontrol/remove_broker?brokerid=1234&dryrun=false&kafka_assigner=true"`

## GET REQUESTS
The GET requests in Kafka Cruise Control REST API are for read only operations, i.e. the operations that do not have any external impacts. The GET requests include the following operations:
* Query the state of Cruise Control
* Query the current cluster load
* Query partition resource utilization
* Get optimization proposals
* Bootstrap the load monitor
* Train the Linear Regression Model

### Get the state of Kafka Cruise Control
User can query the state of Kafka Cruise Control at any time by issuing an HTTP GET request.

    GET /kafkacruisecontrol/state?verbose=[true/false]&json=[true/false]

The returned state contains the following information:
* Monitor State:
  * State: **NOT_STARTED** / **RUNNING** / **SAMPLING** / **PAUSED** / **BOOTSTRAPPING** / **TRAINING** / **LOADING**,
  * Bootstrapping progress (If state is BOOTSTRAPPING)
  * Number of valid monitored windows / Number of total monitored windows
  * Number of valid partitions out of the total number of partitions
  * Percentage of the partitions that are valid
* Executor State:
  * State: **NO_TASK_IN_PROGRESS** /
	   **EXECUTION_STARTED** /
	   **REPLICA_MOVEMENT_IN_PROGRESS** /
	   **LEADER_MOVEMENT_IN_PROGRESS**
  * Total number of replicas to move (if state is REPLICA_MOVEMENT_IN_PROGRESS)
  * Number of replicas finished movement (if state is REPLICA_MOVEMENT_IN_PROGRESS)
* Analyzer State:
  * isProposalReady: Is there a proposal cached
  * ReadyGoals: A list of goals that are ready for running

If verbose is set to true. The details about monitored windows and goals will be displayed.

### Get the cluster load
Once Cruise Control Load Monitor shows it is in the RUNNING state, Users can use the following HTTP GET to get the cluster load:

    GET /kafkacruisecontrol/load?time=[TIMESTAMP]&granularity=[GRANULARITY]

When the time field is not provided, it is default to the wall clock time. If the number of workload snapshots before the given timestamp is not sufficient to generate a good load model, an exception will be returned.

The valid granularity settings are: `broker` and `replica`. The broker level load gives a summary of the workload on the brokers in the Kafka cluster. The replica granularity returns the detail workload of each replicas in the cluster in an XML format. The replica level information could be prohibitively big if the cluster hosts a lot of replicas.

NOTE: The load shown is only for the load from the valid partitions. i.e the partitions with enough metric samples. Even if the monitored valid partition percentage is lower than the configured percentage (e.g. 98%), the load will still be returned. So please always verify the Load Monitor state to decide whether the workload is representative enough.

### Query the partition resource utilization
The following GET request gives the partition load sorted by the utilization of a given resource:

    GET /kafkacruisecontrol/partition_load?resource=[RESOURCE]&start=[START_TIMESTAMP]&end=[END_TIMESTAMP]&json=[true/false]&entries=[MAX_NUMBER_OF_PARTITION_LOAD_ENTRIES_TO_RETURN]

The returned result would be a partition list sorted by the utilization of the specified resource in the given time window. By default the `start` is the earliest monitored time, the `end` is current wall clock time, `resource` is `DISK`, and `entries` is the all partitions in the cluster.

### Get optimization proposals
The following GET request returns the optimization proposals generated based on the workload model of the given timestamp. The workload summary before and after the optimization will also be returned.

    GET /kafkacruisecontrol/proposals?goals=[goal1,goal2...]&verbose=[true/false]&ignore_proposal_cache=[true/false]&data_from=[valid_windows/valid_partitions]&kafka_assigner=[true/false]&json=[true/false]

Kafka cruise control tries to precompute the optimization proposal in the background and caches the best proposal to serve when user queries. If users want to have a fresh proposal without reading it from the proposal cache, set the `ignore_proposal_cache` flag to true. The precomputing always uses available valid partitions to generate the proposals.

By default the proposal will be returned from the cache where all the pre-defined goals are used. Detailed information about the reliability of the proposals will also be returned. If users want to run a different set of goals, they can specify the `goals` argument with the goal names (simple class name).

If `verbose` is turned on, Cruise Control will return all the generated proposals. Otherwise a summary of the proposals will be returned.

Users can specify `data_from` to indicate if they want to run the goals with available `valid_windows` or available `valid_partitions` (default: `valid_windows`).

If `kafka_assigner` is turned on, the proposals will be generated in Kafka Assigner mode. This mode performs optimizations using the goals specific to Kafka Assigner -- i.e. goals with name `KafkaAssigner*`.

### Bootstrap the load monitor (NOT RECOMMENDED)
**(This is not recommended because it may cause the inaccurate partition traffic profiling due to missing metadata. Using the SampleStore is always the preferred way.)**
In order for Kafka Cruise Control to work, the first step is to get the metric samples of partitions on a Kafka cluster. Although users can always wait until all the workload snapshot windows are filled up (e.g. 96 one-hour snapshot window will take 4 days to fill in), Kafka Cruise Control supports a bootstrap function to load old metric examples into the load monitor, so Kafka Cruise Control can begin to work sooner. 

There are three bootstrap modes in Kafka Cruise Control:

* **RANGE** mode: Bootstrap the load monitor by giving a start timestamp and end timestamp.

        GET /kafkacruisecontrol/bootstrap?start=[START_TIMESTAMP]&end=[END_TIMESTAMP]&clearmetrics=[true/false]&json=[true/false]

* **SINCE** mode: bootstrap the load monitor by giving a starting timestamp until it catches up with the wall clock time.

        GET /kafkacruisecontrol/bootstrap?start=[START_TIMESTAMP]&clearmetrics=[true/false]&json=[true/false]

* **RECENT** mode: bootstrap the load monitor with the most recent metric samples until all the load snapshot windows needed by the load monitor are filled in. This is the simplest way to bootstrap the load monitor unless some of the load needs to be excluded. 

        GET /kafkacruisecontrol/bootstrap&clearmetrics=[true/false]

All the bootstrap modes has an option of whether to clear all the existing metric samples in Kafka Cruise Control or not. By default, all the bootstrap operations will clear all the existing metrics. Users can set the parameter clearmetrics=false if they want to preserve the existing metrics.

### Train the linear regression model (Testing in progress)
If use.linear.regression.model is set to true, user have to train the linear regression model before bootstrapping or sampling. The following GET request will start training the linear regression model:

    GET /kafkacruisecontrol/train?start=[START_TIMESTAMP]&end=[END_TIMESTAMP]&json=[true/false]

After the linear regression model training is done (users can check the state of Kafka Cruise Control).

## POST Requests
The post requests of Kafka Cruise Control REST API are operations that will have impact on the Kafka cluster. The post operations include:
* Add a list of new brokers to Kafka
* Decommission a broker from the Kafka cluster
* Trigger a workload balance
* Stop the current proposal execution task

The partition movement will be divided into small batches. At any given time, there will only be at most N replicas (configured by num.concurrent.partition.movements.per.broker) moving into / out of each broker.

All the POST actions except stopping current execution task has a dry-run mode, which only generate the rebalance proposals and estimated result but not really execute the proposals. To avoid accidentally triggering of data movement, by default all the POST actions are in dry-run mode. To let Kafka Cruise Control actually move data, users need to explicitly set dryrun=false.

### Add brokers to the Kafka cluster
The following POST request adds the given brokers to the Kafka cluster

    POST /kafkacruisecontrol/add_broker?brokerid=[id1,id2...]&dryrun=[true/false]&throttle_added_broker=[true/false]&kafka_assigner=[true/false]&json=[true/false]

When adding new brokers to a Kafka cluster, Cruise Control makes sure that the replicas will only be moved from the existing brokers to the given broker, but not moved among existing brokers. 

Users can choose whether to throttle the newly added broker during the partition movement. If the new brokers are unthrottled, there might be more than `num.concurrent.partition.movements.per.broker` moving into each new brokers concurrently.

### Remove a broker from the Kafka cluster
The following POST request removes a broker from the Kafka cluster:

    POST /kafkacruisecontrol/remove_broker?brokerid=[id1, id2...]&dryrun=[true/false]&throttle_removed_broker=[true/false]&kafka_assigner=[true/false]&json=[true/false]

Similar to adding broker to a cluster, removing a broker from a cluster will only move partitions from the broker to be removed to the other existing brokers. There won't be partition movements among remaining brokers.

User can choose whether to throttle the removed broker during the partition movement. If the removed broker is unthrottled, there might be more than `num.concurrent.partition.movements.per.broker` moving into that broker concurrently.

### Rebalance a cluster
The following POST request will let Kafka Cruise Control rebalance a Kafka cluster

    POST /kafkacruisecontrol/rebalance?goals=[goal1,goal2...]&dryrun=[true/false]&data_from=[valid_windows/valid_partitions]&kafka_assigner=[true/false]&json=[true/false]

**goals:** a list of goals to use for rebalance. When goals is provided, the cached proposals will be ignored.

**valid_windows:** rebalance the cluster based on the information in the available valid snapshot windows. A valid snapshot window is a windows whose valid monitored partitions coverage meets the requirements of all the goals. (This is the default behavior)

**valid_partitions:** rebalance the cluster based on all the available valid partitions. All the snapshot windows will be included in this case.

Users can only specify either `valid_windows` or `valid_partitions`, but not both.

When rebalancing a cluster, all the brokers in the cluster are eligible to give / receive replicas. All the brokers will be throttled during the partition movement.

By default the rebalance will be in DryRun mode. Please explicitly set dryrun to false to execute the proposals. Similar to the GET interface for getting proposals, the rebalance can also be based on available valid windows or available valid partitions.

### Stop an ongoing rebalance
The following POST request will let Kafka Cruise Control stop an ongoing `rebalance`, `add_broker`, or `remove_broker` operation:

    POST /kafkacruisecontrol/stop_proposal_execution

Note that Cruise Control does not wait for the ongoing batch to finish when it stops execution, i.e. the in-progress batch may still be running after Cruise Control stops the execution.