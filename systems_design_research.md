# Distributed Locking Design Deep Dive

The provided sources explain a distributed locking system design utilizing **fencing tokens** and the **Raft consensus algorithm** to ensure **exclusivity**, **fault tolerance**, and **scalability**. Fencing tokens, monotonically increasing sequence numbers, prevent data corruption by validating operations, while Raft provides a consistent and fault-tolerant mechanism for managing lock state across multiple nodes. The design addresses the **thundering herd problem** by employing a queue-based approach for lock acquisition, notifying only the next node in line when the lock is released. This combination creates a robust system for coordinating access to shared resources in a distributed environment. A Google SWE's video transcript and supporting documentation present the design and the rationale behind using these techniques.

Briefing Document: 
------------------

**Subject:** Analysis of Google SWE's Distributed Locking Design and Supporting Concepts

**Sources:**

-   "Distributed Locking Design Deep Dive with Google SWE! | Systems Design Interview Question 24" (Video Transcript)
-   "Distributed Locking: Fencing Tokens, Raft Consensus, and Exclusivity" (Summary of Video)

**Main Themes & Key Ideas:**

This document analyzes a proposed design for a distributed locking system, emphasizing the importance of exclusivity, fault tolerance, and scalability. The design leverages fencing tokens and consensus algorithms like Raft to achieve these goals.

**1\. Functional Requirements & Use Cases:**

The primary goal is to build a distributed lock that functions similarly to a traditional lock but operates across multiple nodes in a cluster. This is crucial for scenarios such as:

-   **File Modification:** Preventing data corruption when multiple nodes attempt to edit the same file in a system like S3. "If two of them are editing one file at the same time that's going to be a problem that file is going to get corrupted so what they'll do instead is actually go ahead and grab a distributed lock."
-   **Resource Processing:** Avoiding duplicate work when multiple nodes pull and process elements from a queue (e.g., video onboarding). "Every single time that you want to basically process a chunk of video footage you grab a lock corresponding to that chunk and then that way every other node in the cluster will know that they're not able to go ahead and grab that distributed lock and as a result no double work will be done."

**2\. API Design:**

The system requires at least three basic operations:

-   GrabLock(): Attempts to acquire the lock.
-   ReleaseLock(): Releases the lock. This may require a fencing token to ensure the correct machine is releasing it.
-   Operation on a third party system, like S3, that takes in a fencing token.

**3\. Fault Tolerance & Fencing Tokens:**

Fault tolerance is paramount. The design must handle scenarios where the lock holder or the locking server fails. The document stresses the importance of fencing tokens to prevent data corruption in the event of failures, network delays, or process pauses.

-   **The Problem:** If a lock holder dies or experiences a delay, it might still attempt to perform operations (e.g., writing to S3) after another node has acquired the lock.
-   **The Solution (Fencing Tokens):** Fencing tokens are monotonically increasing sequence numbers assigned to each lock acquisition. The target service (e.g., S3) tracks these tokens and ignores requests with older tokens. "S3 is going to keep track of all the fencing tokens it's seen when machines edit files on it...s3 can go ahead and disregard those saying oh wait that's an old fencing token i don't want to worry about those this is invalid and it's going to mess things up."
-   **Why Timestamps Fail:** Timestamps are not reliable for generating fencing tokens in a distributed system. Relying on a single server for timestamps creates a single point of failure. If that server goes down, a new server may issue earlier timestamps, breaking the monotonicity requirement of fencing tokens.
-   **Why Quorums Fail:** Quorums do not contain the commit phase that consensus algorithms have, therefore resulting in more errors.

**4\. Consensus with Raft:**

The document advocates using a consensus algorithm like Raft to provide a fault-tolerant and consistent locking service.

-   **Raft Basics:** Raft maintains a distributed log of writes (events) replicated across multiple nodes. A leader is elected, and all write proposals are sent to the leader. The leader proposes the write to the followers. If a majority of followers agree, the leader commits the write locally and instructs the followers to commit as well. "This way it's never possible that you can have any sort of contradiction within the raft system because we know that for every single write in that distributed log a majority of nodes has agreed upon it."
-   **Leader Election:** Raft includes automatic leader election if the current leader fails, maintaining availability.
-   **Benefits for Locking:** Raft ensures that if one node successfully acquires the lock (convinces the Raft cluster), all other nodes will consistently see that the first node is holding the lock.

**5\. Scalability & Thundering Herd Problem:**

The design addresses the scalability challenge of the "thundering herd" problem.

-   **The Problem:** When a lock is released, all waiting nodes simultaneously attempt to acquire it, overwhelming the locking service. "Every single time one resource is released you have a ton of other machines trying to basically put a ton of strain on your single raft instance and that's really bad for it it's going to slow things down a lot."
-   **The Solution (Queue Lock):** Maintain a linked list (queue) of lock requests within the Raft cluster. When the lock is released, the next node in the queue is notified via a real-time event mechanism (e.g., Server-Sent Events or long polling) and assigned a new fencing token. This avoids a large number of requests hitting the Raft cluster simultaneously. "As opposed to just telling a machine that wants to grab the lock when the lock is not available...what you can actually do on raft is effectively keep a linked list of all the requests for the lock."
-   **Handling Node Failures in Queue:** If a node in the queue fails (heartbeat times out), it can be removed from the linked list without affecting the system's integrity.

**6\. Visualization & Example:**

The video provides a visual representation of the system:

-   Two clients attempting to acquire the lock.
-   Client 1 wins and is added to the head of the queue.
-   Client 2 is appended to the queue.
-   Client 1 holds the lock and uses a fencing token to write to S3.
-   If Client 1 times out, Client 2, with a higher fencing token, can proceed, and S3 will reject Client 1's delayed request.

**Conclusion:**

The discussed design provides a robust and scalable solution for distributed locking by combining fencing tokens and consensus algorithms. Using Raft for managing the lock state and a queue-based approach for lock acquisition mitigate common issues in distributed systems, resulting in a fault-tolerant and efficient system.

A Deep Dive Study Guide
=======================

Quiz
----

1.  What is a distributed lock and why is it useful in a clustered system?
2.  Describe two scenarios where distributed locks are beneficial.
3.  What is a fencing token and how does it prevent data corruption in distributed systems?
4.  Explain why using timestamps for generating fencing tokens is unreliable in a distributed environment.
5.  Briefly describe how the Raft consensus algorithm works and how it ensures consistency.
6.  Why does the video say quorums fail in distributed locking?
7.  Explain the "thundering herd" problem in the context of distributed locking.
8.  How does the proposed design use a linked list to mitigate the thundering herd problem?
9.  What real-time event mechanisms can be used to notify the next node in the linked list that it can acquire the lock?
10. Briefly explain the video's example visualization of the distributed locking process with two clients and S3.

Quiz Answer Key
---------------

1.  A distributed lock is a mechanism that provides exclusive access to a resource across multiple nodes in a cluster, preventing concurrent modifications. It's useful to prevent data corruption or ensure exclusive operations in distributed environments.
2.  Distributed locks are beneficial in scenarios like file modification in S3 (preventing simultaneous edits) and resource processing (avoiding duplicate work by ensuring only one node processes a specific chunk of data).
3.  A fencing token is a monotonically increasing sequence number assigned to each lock acquisition, ensuring that operations from a previous lock holder (that might be delayed or failed) are ignored. This prevents data corruption by ensuring only the most recent operation is valid.
4.  Timestamps are unreliable because clock synchronization across different nodes in a distributed system is challenging. If the timestamp server fails and another takes over, it might issue older timestamps, breaking the monotonicity requirement of fencing tokens.
5.  Raft maintains a distributed log replicated across multiple nodes. A leader is elected, and all write proposals are sent to the leader, who proposes it to the followers. If a majority agrees, the leader commits the write and instructs the followers to commit, ensuring consistency through majority agreement.
6.  Quorums do not contain the commit phase that consensus algorithms have, therefore resulting in more errors and making quorums an unreliable option for distributed locking systems.
7.  The "thundering herd" problem occurs when a lock is released, and all waiting nodes simultaneously try to acquire it, overwhelming the locking service with requests and causing performance degradation.
8.  The design maintains a linked list of lock requests within the Raft cluster. When the lock is released, only the next node in the queue is notified, preventing all waiting nodes from bombarding the locking service with requests.
9.  Server-Sent Events (SSE) or long polling can be used to notify the next node in the linked list that it can acquire the lock, providing a real-time notification mechanism.
10. The example shows two clients, with Client 1 winning the initial lock and being placed at the head of a queue, and Client 2 being appended to the queue. Client 1 uses its fencing token when writing to S3. If Client 1 times out, S3 will reject Client 1's request if Client 2 writes with a higher token.

Essay Questions
---------------

1.  Discuss the importance of fault tolerance in distributed locking systems and analyze how fencing tokens and the Raft consensus algorithm contribute to achieving this goal.
2.  Compare and contrast the use of timestamps, quorums, and consensus algorithms like Raft for assigning fencing tokens in a distributed locking system. Explain the advantages and disadvantages of each approach.
3.  Explain the "thundering herd" problem in the context of distributed locking and analyze how the proposed solution of using a linked list (queue) and real-time event notifications mitigates this issue.
4.  Imagine you are designing a distributed locking system for a specific use case (e.g., managing access to user profiles in a social media application). Describe the key requirements of your system and explain how you would adapt the concepts of fencing tokens and Raft consensus to meet those requirements.
5.  Evaluate the trade-offs involved in using memory versus disk for running Raft nodes in a distributed locking system. Consider factors such as performance, fault tolerance, and data durability.

Glossary of Key Terms
---------------------

-   **Distributed Lock:** A locking mechanism used to provide exclusive access to a shared resource across multiple nodes in a distributed system.
-   **Fencing Token:** A monotonically increasing sequence number used to prevent outdated or delayed operations from corrupting data in a distributed system.
-   **Fault Tolerance:** The ability of a system to continue operating correctly despite the failure of one or more components.
-   **Consensus Algorithm:** A distributed algorithm that allows a group of nodes to agree on a single value or state, even in the presence of failures.
-   **Raft:** A consensus algorithm that provides a fault-tolerant, distributed log.
-   **Leader Election:** The process by which a new leader is chosen in a distributed system when the current leader fails.
-   **Thundering Herd Problem:** A situation where a large number of processes or threads simultaneously contend for a single resource, leading to performance degradation.
-   **Server-Sent Events (SSE):** A server push technology enabling a server to send real-time updates to a client over a single HTTP connection.
-   **Long Polling:** A technique where a client requests information from a server, and the server holds the request open until new information is available, then sends a response.
-   **Quorum:** The minimum number of members of an assembly or society that must be present at any of its meetings to make the proceedings of that meeting valid.
-   **TTL (Time to Live):** In the context of distributed locking, TTL refers to the duration for which a lock is considered valid. If the lock holder fails to release the lock within the TTL, the lock is automatically released by the locking service.
-   **Heartbeat:** A periodic signal sent by a node to indicate that it is still alive and functioning correctly. In a distributed locking system, lock holders may send heartbeats to the locking service to prevent their locks from expiring prematurely.


# Distributed Job Scheduler Design

Briefing Document: 
------------------

**Source:** Excerpts from "Distributed Job Scheduler Design Deep Dive with Google SWE! | Systems Design Interview Question 25"

**Overview:** This document outlines the design of a distributed job scheduler system, covering functional requirements, capacity estimates, API design, database schema, and a proposed architecture. The design prioritizes ensuring every job is run at least once, even in the face of failures. The system must be scalable to handle tens of thousands of jobs per day.

**Key Themes and Ideas:**

1.  **Functional Requirements:**

-   **Scheduling Jobs:** The system must schedule jobs to run on a dedicated cluster of compute units (servers).
-   **Multiple Scheduling Types:** Support for instant scheduling (run now) and calendar-based scheduling (cron jobs). Examples include: "run every two weeks, run every month, run every day at five o'clock p.m. EST."
-   **At-Least-Once Execution:** Ensuring every job is executed at least once is critical. The design focuses on mechanisms to guarantee this.

1.  **Capacity Estimates:**

-   **Job Size:** Jobs are assumed to be compiled binary code, up to a few megabytes in size.
-   **Job Volume:** The system needs to handle tens of thousands of jobs running per day. This necessitates sharding.

1.  **API Design:**

-   **Schedule Job Endpoint:** Takes the binary code and scheduling information (timestamp or cron syntax) as input.
-   **Check Job Status Endpoint:** Allows users to query the status of a job (completed, failed, being retried, in queue).

1.  **Database Schema (Job Status Table):**

-   **Job ID:** Unique identifier for each job.
-   **Binary URL:** URL to the binary stored in S3. The rationale for using S3 is: "since it's a static file it should be going in something like S3 which is just an elastic object store".
-   **Status:** Enum representing the job's state (started, not started, finished, failed but retriable, failed but not retriable).
-   **Timestamp/Expiration Timestamp:** Indicates when the job should be run or when it's eligible for retry. If the current time is after the timestamp and status is "not started" then the job *could* be re-run.

1.  **System Architecture:**

-   **Client Interaction:** Clients use HTTP or RPC calls to a backend scheduling service. A reverse proxy or front-end server handles incoming requests.
-   **Binary Storage:** Binaries are stored in S3.
-   **Job Metadata Storage:** A database (potentially MySQL) stores metadata about each job (ID, S3 URL, timestamps, status). "We're going to need some sort of database and what the database is going to do is store metadata about each job".
-   **Database Indexing:** An index on the timestamp column is crucial for efficient querying of jobs that need to be run.
-   **Queueing Service:** Jobs are placed into a message queue (RabbitMQ or SQS are suggested over Kafka due to in-memory nature and the need for automatic retries). The rationale being "...we're going to be retrying jobs automatically so they don't have to be durable within the queue and additionally an in-memory broker should generally be a little bit faster and we don't really care about the order in which the jobs are run we just need them run so in memory message brokers here something like basically sqs or rabbitmq are probably a little bit more suitable than something like kafka which is going to be a log based message broker". Queues should be sharded.
-   **Consumer Nodes (Workers):** Consumer nodes pull jobs from the queue, load the binaries from S3, and execute them.
-   **Load Balancer:** Distributes jobs from the queues to the consumer nodes.
-   **Claim Service (Distributed Lock):** Uses a distributed lock (e.g., with ZooKeeper) to ensure only one worker processes a job at a time. This prevents duplicate execution. "...we don't want multiple nodes running that same job at the same time effectively what they're doing is they are hitting a distributed lock which we can manage behind the scenes with something like zookeeper and saying I am currently the only one running this job and I should be the only one running this job".
-   **Heartbeats:** Consumer nodes periodically send heartbeats to ZooKeeper to indicate they are still alive. If a heartbeat is missed, the job is considered failed and retried.

1.  **Failure Handling and Retries:**

-   **Database Queries for Retries:** The system queries the database to find jobs that haven't started or have started but timed out (indicating a failure). The query looks for jobs "that haven't started yet with a timestamp that is less than the current time and also go ahead and find all the jobs that have started however their timestamp that kind of you know they were last checked into the database so for example you know they're in queueing timestamp plus the actual you know hard set in queueing timeout which we've set in our system is going to be less than the current time".
-   **Timeouts:** Timeouts are used throughout the system to detect failures (e.g., queueing timeout, execution timeout).
-   **Zookeeper for Monitoring:** "...zookeeper needs to know if the consumer running the job is down or not".
-   **Metadata Updates on Failure:** The metadata database is updated to reflect failures, allowing for retries.

1.  **Job Prioritization (Optional):**

-   Multiple queues with different priority levels can be used.
-   Starvation can be mitigated by moving jobs from lower priority queues to higher priority queues after a certain amount of time.

1.  **Scheduled Jobs (Cron-like):**

-   After a scheduled job completes (successfully or unsuccessfully), the service updates the metadata and adds a new row to the database representing the next scheduled execution of the job. The author describes this as "... amortize this computation where let's say I have a job that's supposed to be run every two weeks when one version of that job completes you know like the the first one and it's successful or fails then basically what's going to happen is the service that goes ahead and updates the metadata base to say that this job succeeded or failed is also going to add an additional row in the database saying here's the next one that has to be done two weeks from now so basically the previous job is going to put the next job that has to be run in the database".
-   Transactions are used to ensure that both the completion update and the next job creation succeed or fail together.

1.  **Idempotency:** The author notes that the system may retry a job that *was* run, but for which there is no record of completion. He states that "...the only way I can think of to prevent this is to take some extra measures to ensure item potency of your code where basically perhaps you're consulting like a different database table or something to see if something with that exact request id has been run before".

**Diagram Components and Flow:**

1.  **Client:** Sends binary and schedule to the front-end scheduling service.
2.  **Front-end Scheduling Service:** Uploads binary to S3 and puts job info in MySQL.
3.  **Queueing Service:** Polls MySQL for jobs to queue (new or retries)
4.  **RabbitMQ/SQS:** In-memory queue for jobs
5.  **Load Balancer:** Distributes jobs to workers.
6.  **Workers:** Grab distributed lock, execute job, send heartbeats to Zookeeper.
7.  **Zookeeper:** Monitors workers, updates metadata if a worker fails.
8.  **MySQL (Job Status Database):** Stores metadata about each job.
9.  **S3:** Stores the binary files.

**Potential Bottlenecks and Considerations:**

-   **Database Query Performance:** Efficient indexing is critical for the database queries that retrieve jobs to be run.
-   **Distributed Lock Contention:** High contention on the distributed lock could impact performance.
-   **S3 Bandwidth:** Loading binaries from S3 could be a bottleneck if the jobs are large or the system is under heavy load.

This design provides a solid foundation for a distributed job scheduler, addressing key concerns like scalability, reliability, and ensuring at-least-once execution.


Deep Dive Study Guide
=====================

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  What are the two main endpoints that the API should implement for a distributed job scheduler?
2.  Why is it important to store the job binary in a service like S3?
3.  What are the key components of the job status table in the database?
4.  Why is a simple queue not sufficient for implementing a robust job scheduler?
5.  How does the system determine which jobs need to be run?
6.  Why is it important to have an index on the timestamp in the job status database?
7.  Why is an in-memory message broker more suitable for this system than a log-based message broker?
8.  What role does the "claim service," managed behind the scenes with Zookeeper, play in the architecture?
9.  How does the system ensure that jobs are run at least once?
10. How does the system handle jobs scheduled on a recurring basis (e.g., every two weeks)?

Quiz Answer Key
---------------

1.  The two main endpoints are for scheduling a job (taking the binary and scheduling information as input) and checking the job status (allowing users to see the current state of their submitted job).
2.  Storing the binary in a service like S3 is important because the binary files can be a few megabytes in size, and S3 is designed for efficiently storing and retrieving static files, thus freeing up the service and database from storing large files.
3.  The key components are a job ID (unique identifier), a binary URL (pointing to the location of the binary in S3), a status (indicating the current state of the job), and a timestamp (used for scheduling and retries).
4.  A simple queue lacks retriability features and the ability to store the status of each job. A job scheduler requires more complex tracking and management of job states.
5.  The system queries the database for all jobs where the status is not "complete" (either "not started" or "started") and the timestamp is less than the current time, indicating they are ready to be run or retried.
6.  An index on the timestamp allows for efficient querying of the database to find all jobs that are ready to be run, significantly speeding up the process of identifying and scheduling jobs.
7.  An in-memory message broker is more suitable because jobs are retried automatically, making durability less important. In-memory brokers are also faster, and the order of job execution doesn't matter.
8.  The claim service ensures that only one worker node is running a specific job at any given time, preventing duplicate executions. Zookeeper manages distributed locks behind the scenes to coordinate claims.
9.  The system uses timeouts and heartbeats to ensure that jobs are run at least once. If a worker fails or a job gets stuck in a queue, the system detects this and re-enqueues the job for retry.
10. When a scheduled job completes (either successfully or with an error), the service updates the database and adds a new row for the next instance of the job, scheduled according to the specified frequency.

Essay Questions
---------------

1.  Discuss the trade-offs between using MySQL versus a time-series database for the job status table.
2.  Elaborate on the role of distributed locking in ensuring that jobs are not executed more than once, and the challenges associated with implementing a robust distributed lock.
3.  Explain the importance of idempotency in the design of the job scheduler, and describe strategies for ensuring that jobs are idempotent.
4.  Analyze the different points of failure in the system and how the design addresses each of them.
5.  Describe a strategy for implementing job priorities in the scheduler, including potential issues and mitigations.

Glossary of Key Terms
---------------------

-   **Job Scheduler:** A system designed to execute tasks automatically based on predefined schedules or triggers.
-   **Binary:** A compiled executable file containing machine code that can be run by a computer.
-   **Cron Job:** A time-based job scheduling utility found in Unix-like operating systems.
-   **S3 (Simple Storage Service):** A scalable object storage service offered by Amazon Web Services (AWS) used for storing static files.
-   **API (Application Programming Interface):** A set of definitions and protocols that allows different software systems to communicate with each other.
-   **Endpoint:** A specific URL or entry point in an API where a service can be accessed.
-   **Enum (Enumeration):** A data type consisting of a set of named constants.
-   **Reverse Proxy:** A server that sits in front of one or more backend servers and forwards client requests to those servers.
-   **Metadata:** Data that provides information about other data. In this case, it refers to information about the jobs being scheduled.
-   **Index:** A data structure that improves the speed of data retrieval operations on a database table.
-   **Time Series Database:** A database optimized for handling time-series data, which is data indexed by time.
-   **ACID Compliance:** A set of properties (Atomicity, Consistency, Isolation, Durability) that guarantee database transactions are processed reliably.
-   **Message Broker:** Software that enables applications, systems, and services to communicate with each other and exchange information.
-   **In-Memory Message Broker:** A message broker that stores messages in RAM, providing faster performance but less durability than disk-based brokers.
-   **Log-Based Message Broker:** A message broker that stores messages in a durable, ordered log, providing high reliability and fault tolerance.
-   **SQS (Simple Queue Service):** A fully managed message queuing service offered by Amazon Web Services (AWS).
-   **RabbitMQ:** A widely deployed open-source message broker.
-   **Kafka:** A distributed, fault-tolerant streaming platform that is often used as a message broker.
-   **Load Balancer:** A device that distributes network traffic across multiple servers to ensure no single server is overwhelmed.
-   **Consistent Hashing:** A technique used to distribute data across a cluster of servers in a way that minimizes the amount of data that needs to be moved when servers are added or removed.
-   **Data Locality:** The principle of storing data close to where it is most frequently accessed to reduce latency.
-   **Consumer Node:** A server or process that consumes messages from a message queue and processes the associated job.
-   **Claim Service:** A service responsible for managing distributed locks and ensuring that only one consumer node is working on a particular job at any given time.
-   **Distributed Lock:** A lock that is shared across multiple processes or servers in a distributed system, used to ensure that only one process can access a shared resource at a time.
-   **Zookeeper:** A centralized service for maintaining configuration information, naming, providing distributed synchronization, and group services.
-   **Fencing Token:** A unique identifier that is incremented each time a resource is accessed, used to prevent race conditions and ensure that operations are performed in the correct order.
-   **Idempotency:** The property of an operation that allows it to be executed multiple times without changing the result beyond the initial application.
-   **Transaction:** A sequence of operations treated as a single logical unit of work, either all operations succeed, or none do.
-   **Amortize Computation:** Distributing the cost of a computation over a longer period of time, typically by performing the computation incrementally.


# Metrics/Logging Design Deep Dive

**This YouTube transcript outlines a system design for handling distributed metrics and logging.** The speaker, a Google SWE, details functional requirements, capacity estimates, and API endpoints for such a system. **The discussion covers the architecture, emphasizing message brokers like Kafka for message durability and replayability.** Various stream processing techniques using tools like Flink and Spark are examined for data enrichment and time-based aggregations. **The destination of processed data is explored, including time-series databases for efficient reads/writes and data lakes like S3 for unstructured data.** Batch processing with Hadoop and Spark is mentioned for further analysis and data warehousing. **The explanation closes with a diagram illustrating the data flow from clients to various storage solutions, incorporating load balancing, consumer hardware, and potential caching strategies.**


Briefing Document:
-----------------

**Source:** Excerpts from "Distributed Metrics/Logging Design Deep Dive with Google SWE! | Systems Design Interview Question 14"

**Overview:**

This document summarizes a system design approach for handling distributed metrics and logging, presented in a video format. While the presentation contains considerable amounts of irrelevant and unprofessional commentary, the core content outlines a practical architecture suitable for processing and analyzing large volumes of log and metric data. The design emphasizes scalability, durability, and the ability to perform both real-time (stream) and batch processing on the data.

**Key Themes and Ideas:**

1.  **Functional Requirements:**

-   Clients (including devices) should be able to generate both logs and metrics.
-   The generated data must be accessible to internal users for analysis and insights.
-   Data should be persisted for long-term analytical exploration (big data storage).

1.  *"Clients or even possibly devices if we're talking about metrics are able to generate both logs and metrics that other you know internal people working in the company will eventually be able to check out and hopefully gain some insights from...they should also be available in a way that is persistent for further analytical exploration down the line."*
2.  **Capacity Estimates:**

-   The design should accommodate a large volume of messages (logs/metrics) -- estimated at 100 million messages per day.
-   Assuming an average message size of 100 bytes, this translates to approximately 100 GB of data per day.

1.  *"Let's assume about 100 million messages whether that's logs metrics user generated anything like that or generated per day...let's say on average each message is 100 bytes...if that's the case then that means we have about 100 gigabytes of data per day to process"*
2.  **High-Level Architecture:**

-   The system uses a message broker (Kafka) to centralize messages from various clients and servers.
-   Consumer nodes (Flink or Spark Streaming) process these messages, performing tasks like stream enrichment and time-based aggregations.
-   Processed data is then sent to either a time-series database (for real-time analytics and visualization) or a data lake (S3) for batch processing and long-term storage.
-   Data in the data lake can be further processed using Hadoop/Spark and stored in a data warehouse for SQL-based insights.

1.  **Message Broker (Kafka):**

-   A log-based message broker (like Kafka) is preferred over an in-memory broker (like RabbitMQ) due to the need for message durability and replayability.
-   Durability ensures that no logs or metrics are lost, maintaining data accuracy.
-   Replayability allows for adding new consumer types and recovering consumer state in case of crashes.

1.  *"...for our case it seems like a log based message broker may be a little bit better...one of which is the durability of messages because everything is persisted to disk we can be sure that none of our metrics or number logs that we have are getting lost...having a message broker that allows us to replay messages that were in the past will allow us to a add new types of consumers that can do new types of processing on the data and b basically um you know get their state back if they were to crash."*
2.  **Stream Processing (Flink/Spark Streaming):**

-   Consumer nodes perform stream enrichment (e.g., joining user data to logs) and stream-stream joins to create more insightful logs.
-   Time-based aggregations (tumbling, hopping, and sliding windows) are used to create metrics over different time intervals.
-   Flink is preferred for real-time processing, while Spark Streaming is suitable for mini-batch processing.

1.  *"a lot of these messages are going to have things like a user id but they're probably not going to include a bunch of relevant data with that user which we may actually want in our logs so we don't have to query a sql table every time we see a given log...Flink is handling messages in real time whereas spark is handling messages in mini batches"*
2.  **Data Storage:**

-   **Time-Series Database:** Ideal for handling timestamped data (metrics and logs) due to optimized write and read performance. Uses mini-indexes (hyper tables) for each log source and time range to improve caching and deletion efficiency.
-   *"the really useful thing about a time series database is that is specifically created in order to handle this timestamp data like metrics and like logs so that it can be both written and read really quickly...all of these writes for you know a given day are probably only going to be going to a couple of these mini indexes and as a result we can cache the entire index in memory and get really really good write performance and read performance by doing that"*
-   **Data Lake (S3):** Stores unstructured or less formatted data for batch processing.
-   *"the better place to put it would be something like s3 and we would use s3 as something known as a data lake where we're basically just throwing in a bunch of jumbled unformatted data and then we know that we're going to be doing some processing on it later"*
-   **Data Warehouse:** Stores formatted data in a column-oriented format (e.g., using Parquet) for efficient SQL queries and business intelligence. Column-oriented storage and compression techniques (dictionary, bitmap encoding) improve data locality and reduce storage costs.
-   *"oftentimes it's actually stored in a column-oriented format and what that means is if you're trying to basically rip all the results from one single column say there's one metric that you're really interested in then being able to store all of your data in a column-oriented format means that all of the basically entries in a single column are going to be in the same file and you're going to get much better data locality"*

1.  **Data Encoding and Archiving:**

-   Data encoding frameworks (Avro, Protocol Buffers, Thrift) can be used to reduce data size during transmission and storage.
-   Data can be moved to slower, cheaper storage solutions (e.g., AWS Glacier) as it ages and becomes less relevant. Subsampling can also be used.
-   *"you can encode a lot of these messages using data encoding frameworks like avro protocol buffers or thrift and that's going to be super useful because not only does it reduce the amount of data that has to be sent over the network but more importantly it is going to reduce the amount of data that's actually being held on disk"*

**Diagram:**

The presenter provides a diagram (described verbally) that illustrates the data flow:

-   Clients/Servers -> Load Balancer -> Kafka -> Flink Consumers -> Time-Series DB and/or S3 -> Hadoop/Spark (for S3 data) -> Data Warehouse.

**Conclusion:**

The presented architecture provides a reasonable framework for building a scalable and robust distributed metrics and logging system. The use of Kafka, stream processing engines like Flink/Spark, time-series databases, and data lakes allows for both real-time and batch analysis of large data volumes. The emphasis on data encoding, compression, and archiving strategies is crucial for managing storage costs and ensuring long-term viability. The presenter emphasizes that this is only a starting point. Metrics and Logs are only useful with adequate downstream analysis and processing.


A Deep Dive Study Guide
=======================

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  What are the two primary functional requirements for a distributed metrics and logging system, as outlined in the source?
2.  Why is a log-based message broker, like Kafka, preferred over an in-memory message broker for this system?
3.  Explain the concept of stream enrichment and provide an example of how it enhances log data.
4.  What is the difference between tumbling, hopping, and sliding windows in the context of time-based aggregations?
5.  What are the advantages of using a time series database specifically for metrics and logging data compared to a general-purpose database?
6.  Explain how time series databases optimize write performance and read performance by splitting out the table into mini indexes?
7.  What is a data lake, and why is S3 often used as a data lake in this architecture?
8.  How does storing data in a column-oriented format improve data locality and read performance, especially for analytics?
9.  Describe data encoding frameworks and why they are important for the logging pipeline.
10. What are some of the considerations for data archival in a logging system?

Quiz Answer Key
---------------

1.  The two primary functional requirements are the ability for clients/devices to generate logs and metrics for internal analysis, and for these logs and metrics to be persistently stored for further analytical exploration (e.g., batch processing). This allows internal users to gain insights from the data and perform more in-depth analysis over time.
2.  A log-based message broker offers durability because messages are persisted to disk, ensuring no data loss. This is critical for accurate metrics and logs and allows consumer nodes to replay past messages, enabling new types of processing and recovery from crashes.
3.  Stream enrichment involves adding relevant contextual data to log messages. For example, enriching a log entry containing a user ID with additional user information (e.g., location, demographics) from a separate database to avoid repetitive SQL queries.
4.  Tumbling windows are fixed-length, fixed-start-time, non-overlapping windows, such as one-minute windows starting at the beginning of each minute. Hopping windows are similar but can overlap, created by aggregating tumbling windows. Sliding windows have a fixed length, but with no specific constraints on start or end time, allowing for more granular analysis.
5.  Time series databases are specifically designed to handle timestamped data like metrics and logs, optimizing both read and write performance. Their architecture allows for faster data visualization and analysis of time-dependent trends compared to general-purpose databases.
6.  Time series databases optimize performance by splitting the main table into mini indexes, with each index representing a log source and specific time range. This caching all the writes into that table into memory.
7.  A data lake is a storage repository that holds vast amounts of raw data in its native format, often including unstructured or semi-structured data. S3 is used as a data lake because it offers cheap, scalable storage for these diverse log formats without imposing strict formatting requirements upfront.
8.  Column-oriented storage organizes data by columns rather than rows, which improves data locality when querying a specific metric. This arrangement allows for faster reads because all entries for a single column are stored contiguously, reducing I/O operations.
9.  Data encoding frameworks compress data, improving both network transmission speed and required data storage. These frameworks can often be optimized for the format in which the data is written, allowing for further optimization.
10. Data archival involves strategies like sub-sampling (keeping only a portion of the data) or moving older, less-relevant data to slower, cheaper storage solutions like AWS Glacier. These techniques help reduce storage costs while preserving potentially valuable historical data.

Essay Questions
---------------

1.  Discuss the trade-offs between real-time and mini-batch processing for stream consumers in a distributed metrics and logging system. Which approach is more suitable for different use cases, and why?
2.  Explain the role of each component (client, load balancer, Kafka, Flink, time series database, S3, Hadoop, data warehouse) in the distributed metrics and logging architecture presented in the source. How do these components interact to achieve the overall goals of the system?
3.  Describe the importance of data formatting and encoding in a distributed logging and metrics system. Discuss the trade-offs between storing data in a raw, unformatted state versus enforcing a strict schema, and the implications for processing and analysis.
4.  Analyze the scalability and fault tolerance considerations in the distributed metrics and logging design. How does the architecture address potential bottlenecks and ensure data reliability in the face of component failures?
5.  Evaluate the potential benefits of machine learning to analyze logging and metrics data. How would the logging and metrics data, gathered in the described system, be leveraged to improve machine learning model performance.

Glossary of Key Terms
---------------------

-   **Metrics:** Numerical measurements that track the performance and behavior of a system over time.
-   **Logging:** Recording events or activities that occur within a system, providing detailed information for debugging, auditing, and analysis.
-   **Message Broker:** A software application or system that facilitates communication between different parts of a distributed system by managing and routing messages. Examples include Kafka and RabbitMQ.
-   **In-Memory Message Broker:** A message broker that primarily stores messages in memory, offering high throughput but with potential data loss in case of failures.
-   **Log-Based Message Broker:** A message broker that persists messages to disk, providing durability and enabling replayability, crucial for data accuracy and fault tolerance.
-   **Kafka:** A distributed, fault-tolerant, high-throughput log-based message broker widely used for building real-time data pipelines and streaming applications.
-   **Stream Enrichment:** The process of adding contextual data to log messages or metrics in real time, enhancing their analytical value.
-   **Flink:** A distributed stream processing framework that provides real-time processing and state management capabilities for complex event processing and analytics.
-   **Spark Streaming:** An extension of the Apache Spark framework that enables scalable and fault-tolerant stream processing using mini-batches of data.
-   **Time Series Database (TSDB):** A specialized database optimized for storing and retrieving time-stamped data, like metrics and logs, enabling efficient analysis of trends and patterns over time.
-   **Hyper Table:** A table split into mini indexes where each index represents one source of logs so that all writes for a given day will only go to a couple of mini indexes.
-   **Data Lake:** A storage repository that holds a vast amount of raw data in its native format, often including unstructured or semi-structured data.
-   **S3 (Simple Storage Service):** A scalable, low-cost object storage service offered by Amazon Web Services (AWS), commonly used as a data lake.
-   **Hadoop:** An open-source framework for distributed storage and processing of large datasets, often used for batch processing and analytics.
-   **Spark:** A fast and general-purpose distributed processing engine for big data, often used for data transformation, analytics, and machine learning.
-   **Data Warehouse:** A central repository for structured data that has already been processed and filtered for specific analytical purposes.
-   **Column-Oriented Storage:** A data storage format that organizes data by columns rather than rows, optimizing read performance for analytical queries that access specific columns.
-   **Parquet:** An open-source, column-oriented data storage format designed for efficient data compression and retrieval in big data environments.
-   **Data Encoding Frameworks:** Tools and protocols like Avro, Protocol Buffers, and Thrift used to serialize and compress data, reducing storage space and network bandwidth requirements.
-   **Sub Sampling:** Reducing the volume of data by only storing or analyzing a subset of the available data points.
-   **Archiving:** Moving older, less frequently accessed data to cheaper storage solutions for long-term preservation.

# Metrics and Logging System Design Deep Dive

**This YouTube transcript outlines a system design for handling distributed metrics and logging.** The speaker, a Google SWE, details functional requirements, capacity estimates, and API endpoints for such a system. **The discussion covers the architecture, emphasizing message brokers like Kafka for message durability and replayability.** Various stream processing techniques using tools like Flink and Spark are examined for data enrichment and time-based aggregations. **The destination of processed data is explored, including time-series databases for efficient reads/writes and data lakes like S3 for unstructured data.** Batch processing with Hadoop and Spark is mentioned for further analysis and data warehousing. **The explanation closes with a diagram illustrating the data flow from clients to various storage solutions, incorporating load balancing, consumer hardware, and potential caching strategies.**

Briefing Document:
------------------

**Source:** Excerpts from "Distributed Metrics/Logging Design Deep Dive with Google SWE! | Systems Design Interview Question 14"

**Overview:**

This document summarizes a system design approach for handling distributed metrics and logging, presented in a video format. While the presentation contains considerable amounts of irrelevant and unprofessional commentary, the core content outlines a practical architecture suitable for processing and analyzing large volumes of log and metric data. The design emphasizes scalability, durability, and the ability to perform both real-time (stream) and batch processing on the data.

**Key Themes and Ideas:**

1.  **Functional Requirements:**

-   Clients (including devices) should be able to generate both logs and metrics.
-   The generated data must be accessible to internal users for analysis and insights.
-   Data should be persisted for long-term analytical exploration (big data storage).

1.  *"Clients or even possibly devices if we're talking about metrics are able to generate both logs and metrics that other you know internal people working in the company will eventually be able to check out and hopefully gain some insights from...they should also be available in a way that is persistent for further analytical exploration down the line."*
2.  **Capacity Estimates:**

-   The design should accommodate a large volume of messages (logs/metrics) -- estimated at 100 million messages per day.
-   Assuming an average message size of 100 bytes, this translates to approximately 100 GB of data per day.

1.  *"Let's assume about 100 million messages whether that's logs metrics user generated anything like that or generated per day...let's say on average each message is 100 bytes...if that's the case then that means we have about 100 gigabytes of data per day to process"*
2.  **High-Level Architecture:**

-   The system uses a message broker (Kafka) to centralize messages from various clients and servers.
-   Consumer nodes (Flink or Spark Streaming) process these messages, performing tasks like stream enrichment and time-based aggregations.
-   Processed data is then sent to either a time-series database (for real-time analytics and visualization) or a data lake (S3) for batch processing and long-term storage.
-   Data in the data lake can be further processed using Hadoop/Spark and stored in a data warehouse for SQL-based insights.

1.  **Message Broker (Kafka):**

-   A log-based message broker (like Kafka) is preferred over an in-memory broker (like RabbitMQ) due to the need for message durability and replayability.
-   Durability ensures that no logs or metrics are lost, maintaining data accuracy.
-   Replayability allows for adding new consumer types and recovering consumer state in case of crashes.

1.  *"...for our case it seems like a log based message broker may be a little bit better...one of which is the durability of messages because everything is persisted to disk we can be sure that none of our metrics or number logs that we have are getting lost...having a message broker that allows us to replay messages that were in the past will allow us to a add new types of consumers that can do new types of processing on the data and b basically um you know get their state back if they were to crash."*
2.  **Stream Processing (Flink/Spark Streaming):**

-   Consumer nodes perform stream enrichment (e.g., joining user data to logs) and stream-stream joins to create more insightful logs.
-   Time-based aggregations (tumbling, hopping, and sliding windows) are used to create metrics over different time intervals.
-   Flink is preferred for real-time processing, while Spark Streaming is suitable for mini-batch processing.

1.  *"a lot of these messages are going to have things like a user id but they're probably not going to include a bunch of relevant data with that user which we may actually want in our logs so we don't have to query a sql table every time we see a given log...Flink is handling messages in real time whereas spark is handling messages in mini batches"*
2.  **Data Storage:**

-   **Time-Series Database:** Ideal for handling timestamped data (metrics and logs) due to optimized write and read performance. Uses mini-indexes (hyper tables) for each log source and time range to improve caching and deletion efficiency.
-   *"the really useful thing about a time series database is that is specifically created in order to handle this timestamp data like metrics and like logs so that it can be both written and read really quickly...all of these writes for you know a given day are probably only going to be going to a couple of these mini indexes and as a result we can cache the entire index in memory and get really really good write performance and read performance by doing that"*
-   **Data Lake (S3):** Stores unstructured or less formatted data for batch processing.
-   *"the better place to put it would be something like s3 and we would use s3 as something known as a data lake where we're basically just throwing in a bunch of jumbled unformatted data and then we know that we're going to be doing some processing on it later"*
-   **Data Warehouse:** Stores formatted data in a column-oriented format (e.g., using Parquet) for efficient SQL queries and business intelligence. Column-oriented storage and compression techniques (dictionary, bitmap encoding) improve data locality and reduce storage costs.
-   *"oftentimes it's actually stored in a column-oriented format and what that means is if you're trying to basically rip all the results from one single column say there's one metric that you're really interested in then being able to store all of your data in a column-oriented format means that all of the basically entries in a single column are going to be in the same file and you're going to get much better data locality"*

1.  **Data Encoding and Archiving:**

-   Data encoding frameworks (Avro, Protocol Buffers, Thrift) can be used to reduce data size during transmission and storage.
-   Data can be moved to slower, cheaper storage solutions (e.g., AWS Glacier) as it ages and becomes less relevant. Subsampling can also be used.
-   *"you can encode a lot of these messages using data encoding frameworks like avro protocol buffers or thrift and that's going to be super useful because not only does it reduce the amount of data that has to be sent over the network but more importantly it is going to reduce the amount of data that's actually being held on disk"*

**Diagram:**

The presenter provides a diagram (described verbally) that illustrates the data flow:

-   Clients/Servers -> Load Balancer -> Kafka -> Flink Consumers -> Time-Series DB and/or S3 -> Hadoop/Spark (for S3 data) -> Data Warehouse.

**Conclusion:**

The presented architecture provides a reasonable framework for building a scalable and robust distributed metrics and logging system. The use of Kafka, stream processing engines like Flink/Spark, time-series databases, and data lakes allows for both real-time and batch analysis of large data volumes. The emphasis on data encoding, compression, and archiving strategies is crucial for managing storage costs and ensuring long-term viability. The presenter emphasizes that this is only a starting point. Metrics and Logs are only useful with adequate downstream analysis and processing.

A Deep Dive Study Guide
=======================

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  What are the two primary functional requirements for a distributed metrics and logging system, as outlined in the source?
2.  Why is a log-based message broker, like Kafka, preferred over an in-memory message broker for this system?
3.  Explain the concept of stream enrichment and provide an example of how it enhances log data.
4.  What is the difference between tumbling, hopping, and sliding windows in the context of time-based aggregations?
5.  What are the advantages of using a time series database specifically for metrics and logging data compared to a general-purpose database?
6.  Explain how time series databases optimize write performance and read performance by splitting out the table into mini indexes?
7.  What is a data lake, and why is S3 often used as a data lake in this architecture?
8.  How does storing data in a column-oriented format improve data locality and read performance, especially for analytics?
9.  Describe data encoding frameworks and why they are important for the logging pipeline.
10. What are some of the considerations for data archival in a logging system?

Quiz Answer Key
---------------

1.  The two primary functional requirements are the ability for clients/devices to generate logs and metrics for internal analysis, and for these logs and metrics to be persistently stored for further analytical exploration (e.g., batch processing). This allows internal users to gain insights from the data and perform more in-depth analysis over time.
2.  A log-based message broker offers durability because messages are persisted to disk, ensuring no data loss. This is critical for accurate metrics and logs and allows consumer nodes to replay past messages, enabling new types of processing and recovery from crashes.
3.  Stream enrichment involves adding relevant contextual data to log messages. For example, enriching a log entry containing a user ID with additional user information (e.g., location, demographics) from a separate database to avoid repetitive SQL queries.
4.  Tumbling windows are fixed-length, fixed-start-time, non-overlapping windows, such as one-minute windows starting at the beginning of each minute. Hopping windows are similar but can overlap, created by aggregating tumbling windows. Sliding windows have a fixed length, but with no specific constraints on start or end time, allowing for more granular analysis.
5.  Time series databases are specifically designed to handle timestamped data like metrics and logs, optimizing both read and write performance. Their architecture allows for faster data visualization and analysis of time-dependent trends compared to general-purpose databases.
6.  Time series databases optimize performance by splitting the main table into mini indexes, with each index representing a log source and specific time range. This caching all the writes into that table into memory.
7.  A data lake is a storage repository that holds vast amounts of raw data in its native format, often including unstructured or semi-structured data. S3 is used as a data lake because it offers cheap, scalable storage for these diverse log formats without imposing strict formatting requirements upfront.
8.  Column-oriented storage organizes data by columns rather than rows, which improves data locality when querying a specific metric. This arrangement allows for faster reads because all entries for a single column are stored contiguously, reducing I/O operations.
9.  Data encoding frameworks compress data, improving both network transmission speed and required data storage. These frameworks can often be optimized for the format in which the data is written, allowing for further optimization.
10. Data archival involves strategies like sub-sampling (keeping only a portion of the data) or moving older, less-relevant data to slower, cheaper storage solutions like AWS Glacier. These techniques help reduce storage costs while preserving potentially valuable historical data.

Essay Questions
---------------

1.  Discuss the trade-offs between real-time and mini-batch processing for stream consumers in a distributed metrics and logging system. Which approach is more suitable for different use cases, and why?
2.  Explain the role of each component (client, load balancer, Kafka, Flink, time series database, S3, Hadoop, data warehouse) in the distributed metrics and logging architecture presented in the source. How do these components interact to achieve the overall goals of the system?
3.  Describe the importance of data formatting and encoding in a distributed logging and metrics system. Discuss the trade-offs between storing data in a raw, unformatted state versus enforcing a strict schema, and the implications for processing and analysis.
4.  Analyze the scalability and fault tolerance considerations in the distributed metrics and logging design. How does the architecture address potential bottlenecks and ensure data reliability in the face of component failures?
5.  Evaluate the potential benefits of machine learning to analyze logging and metrics data. How would the logging and metrics data, gathered in the described system, be leveraged to improve machine learning model performance.

Glossary of Key Terms
---------------------

-   **Metrics:** Numerical measurements that track the performance and behavior of a system over time.
-   **Logging:** Recording events or activities that occur within a system, providing detailed information for debugging, auditing, and analysis.
-   **Message Broker:** A software application or system that facilitates communication between different parts of a distributed system by managing and routing messages. Examples include Kafka and RabbitMQ.
-   **In-Memory Message Broker:** A message broker that primarily stores messages in memory, offering high throughput but with potential data loss in case of failures.
-   **Log-Based Message Broker:** A message broker that persists messages to disk, providing durability and enabling replayability, crucial for data accuracy and fault tolerance.
-   **Kafka:** A distributed, fault-tolerant, high-throughput log-based message broker widely used for building real-time data pipelines and streaming applications.
-   **Stream Enrichment:** The process of adding contextual data to log messages or metrics in real time, enhancing their analytical value.
-   **Flink:** A distributed stream processing framework that provides real-time processing and state management capabilities for complex event processing and analytics.
-   **Spark Streaming:** An extension of the Apache Spark framework that enables scalable and fault-tolerant stream processing using mini-batches of data.
-   **Time Series Database (TSDB):** A specialized database optimized for storing and retrieving time-stamped data, like metrics and logs, enabling efficient analysis of trends and patterns over time.
-   **Hyper Table:** A table split into mini indexes where each index represents one source of logs so that all writes for a given day will only go to a couple of mini indexes.
-   **Data Lake:** A storage repository that holds a vast amount of raw data in its native format, often including unstructured or semi-structured data.
-   **S3 (Simple Storage Service):** A scalable, low-cost object storage service offered by Amazon Web Services (AWS), commonly used as a data lake.
-   **Hadoop:** An open-source framework for distributed storage and processing of large datasets, often used for batch processing and analytics.
-   **Spark:** A fast and general-purpose distributed processing engine for big data, often used for data transformation, analytics, and machine learning.
-   **Data Warehouse:** A central repository for structured data that has already been processed and filtered for specific analytical purposes.
-   **Column-Oriented Storage:** A data storage format that organizes data by columns rather than rows, optimizing read performance for analytical queries that access specific columns.
-   **Parquet:** An open-source, column-oriented data storage format designed for efficient data compression and retrieval in big data environments.
-   **Data Encoding Frameworks:** Tools and protocols like Avro, Protocol Buffers, and Thrift used to serialize and compress data, reducing storage space and network bandwidth requirements.
-   **Sub Sampling:** Reducing the volume of data by only storing or analyzing a subset of the available data points.
-   **Archiving:** Moving older, less frequently accessed data to cheaper storage solutions for long-term preservation.

# Low Latency Stock Exchange Design Deep Dive

**The YouTube transcript presents a systems design interview question focused on building a low-latency stock exchange.** **The content creator, a Google SWE, emphasizes performing all operations in memory to achieve nanosecond-level execution.** **The discussion covers functional requirements like buying/selling stocks, canceling orders, and displaying market data.** **The video advocates using UDP multicast for broadcasting order information fairly and efficiently.** **It introduces retransmitters to handle dropped UDP messages and a secondary matching engine for fault tolerance through state machine replication.** **Sharding is considered for performance, and a diagram illustrates the architecture, highlighting the matching engine's role in processing orders and the importance of minimizing its workload.**

Briefing Document: 
------------------

**Source:** Excerpts from "Low Latency Stock Exchange Design Deep Dive with Google SWE! | Systems Design Interview Question 15"

**Main Theme:** Designing a high-performance, low-latency stock exchange system with a focus on in-memory processing and real-time data dissemination. The source contrasts this approach with implementations that rely on disk-based databases and message queues, arguing that true stock exchanges operate almost entirely in memory.

**Key Ideas and Facts:**

1.  **Functional Requirements:** The core functionalities of a stock exchange include:

-   Buying and selling stocks based on the spread (crossing of buy/sell prices).
-   Canceling existing orders.
-   Quickly accessing market data (volume at given price points).
-   API for buying, selling, canceling, and price/volume queries.

1.  **Capacity Estimates:** Modern stock exchanges handle high volumes of messages:

-   "Often 100,000 messages per second."
-   "If they're about 20 bytes per message that means we're dealing with you know say 20 gigabytes per day or something on that scale."

1.  **Rejection of Disk-Based Approaches:** A central argument is that real-time stock exchanges cannot rely on disk-based technologies due to latency constraints.

-   "The reason again why i'm skipping our database schema here is because the database schema itself is not going to make sense when we have no database in actual stock exchange we need to be executing orders on the order of magnitude of nanoseconds basically and that's obviously not going to be suitable if we're reading from disk so everything is going to be done in memory here and ideally a lot of it in cache."

1.  **Matching Engine and Limit Order Book:** The heart of the system is the matching engine, which implements the limit order book algorithm.

-   "The matching engine is basically going to encapsulate all of the functionality of the limit order book."
-   The limit order book "keeps track of all open orders for buys and sells of a given stock and once that spread is crossed basically meaning that someone is willing to buy at a price greater than or equal to someone else willing to sell then you go ahead and fulfill that order."

1.  **In-Memory Data Structures:** The matching engine requires efficient in-memory data structures:

-   A tree-based structure (potentially a binary search tree or self-balancing tree) for open buy and sell orders. Each node in the tree represents a limit price.
-   Linked lists at each node to store all orders at a given limit price.
-   Hash maps for fast lookups: one mapping limit price to the head of the linked list, and another mapping order ID to the node within the linked list for quick order cancellation.
-   Heaps can also be used to determine the minimum and maximum price
-   "At each node of a tree we have a linked list containing all the potential orders at a given limit price this way every single time that an order comes in you basically put it in the proper node of this probably binary search tree or potentially even a self-balanced tree at the proper limit price node"
-   "Two sets of hash maps one hash map that basically points to every single start of the linked list at a given limit and then a second hash map for within each limit pointing to each node"

1.  **Data Dissemination with UDP Multicast:** Order execution and market data updates are broadcast using UDP multicast.

-   "The technology that we would actually use is something called udp multicast."
-   UDP multicast is chosen over TCP for speed and fairness (simultaneous delivery to all subscribers). TCP handshakes introduce delays and unfairness.
-   "Multicast makes sure that all that information that's being sent over the network is being sent out at the same time so you're achieving maximum fairness whereas tcp you can only get basically two nodes to connect at the same time so you have to do all these handshakes in a sequential order and then it's completely unfair because one client is going to have its handshake done first"

1.  **Handling UDP Packet Loss: Retransmitters:** Because UDP is unreliable, retransmitter nodes are introduced.

-   Retransmitters log all outgoing messages from the matching engine.
-   Clients can request missed messages from the retransmitters.
-   New clients can use the retransmitter to catch up on the current state.
-   "The entire job of the retransmitter node is to basically keep a log of all those outgoing messages so that if any of the clients or interesting parties or anything like that realize that they missed a message they can just ask the retransmitter node for it"

1.  **Fault Tolerance: Secondary Matching Engine:** A secondary matching engine provides fault tolerance.

-   The secondary engine listens to all the messages broadcast by the primary and updates its own in-memory state (limit order book) through "state machine replication."
-   "When the primary matching engine is broadcasting all of its messages back to all the clients or to any of the interested parties the secondary matching engine is also listening to these and not only is it listening to these but it's updating its local state where its local state is going to be that limit order book."
-   In case of primary failure, a consensus mechanism (e.g., Zookeeper) can promote the secondary engine to primary.

1.  **Sharding Considerations:** Sharding by ticker symbol is a potential optimization, but it complicates multi-ticker (conditional) orders.

-   "The obvious way to do it would be on tickers right there are a bunch of different stocks out there you know there's apple there's google there's facebook there's microsoft all these things and we can make it such that you know we take our diagram our entire system of you know clients connecting with our couple of matching engines and a couple of pre-transmitters and all that and we just duplicate it you know 100 times over for all the tickers or something like that but the truth of the matter is even though this will make things more efficient we would also lose some abilities"

1.  **Importance of Latency:** Latency is the primary performance bottleneck. Minimizing latency maximizes throughput.

-   "In the stock exchange latency is throughput."
-   "Speed is kind of everything here but the only other thing we can really do is kind of shard it"

1.  **System Diagram Overview:**

-   Clients connect to Order Services (web servers).
-   Order Services send messages to the Matching Engine.
-   The Matching Engine processes orders and updates the limit order book.
-   Messages are broadcast to Order Services, Retransmitters, and the Backup Matching Engine.
-   A separate Cancel process handles order cancellations.
-   Stream processing microservices and databases (e.g., MySQL) handle transaction processing, fraud detection, and time-series data.

**Overall Impression:** The source provides a high-level overview of the architecture of a low-latency stock exchange, emphasizing the importance of in-memory data structures, real-time communication via UDP multicast, and fault tolerance. It's geared towards a systems design interview scenario, focusing on the core components and trade-offs involved in building such a system.

Design Study Guide
==================

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  Why are traditional databases generally unsuitable for low-latency stock exchange operations?
2.  Describe the primary function of the matching engine in a stock exchange.
3.  What is a limit order book and how does it determine when an order should be executed?
4.  Explain why it's important to avoid locking within the memory used by the matching engine.
5.  How do the two hash maps (for linked list starts and for nodes within each limit) contribute to efficient order management?
6.  What is UDP multicast and why is it preferred over TCP for broadcasting order information?
7.  What are retransmitters and what purpose do they serve in the stock exchange architecture?
8.  Why can't a consensus algorithm (like Zookeeper) keep primary and backup matching engines in sync?
9.  How does the backup matching engine maintain an up-to-date state despite relying on the primary matching engine for message ordering?
10. What is a drawback to sharding the stock exchange by ticker?

Quiz Answer Key
---------------

1.  Traditional databases rely on disk reads, which are too slow for the nanosecond-level execution speeds required in stock exchanges. Operations need to occur entirely in memory to minimize latency.
2.  The matching engine encapsulates the functionality of the limit order book, determining when buy and sell orders can be matched and executed based on price. It also broadcasts order information to interested parties.
3.  A limit order book is a record of all open buy and sell orders for a given stock at specific prices. An order is executed when the highest potential buying price crosses the lowest potential selling price.
4.  Locking can introduce delays and reduce the speed of order execution, which is critical for low-latency exchanges. Avoiding locking allows for faster processing of incoming orders and quicker determination of transactions.
5.  The first hash map allows for quick retrieval of the linked list of orders at a specific price, enabling efficient retrieval of volume at that price. The second hash map allows for constant-time removal of elements from a linked list during order cancellations.
6.  UDP multicast allows for simultaneous broadcasting of information to multiple recipients, ensuring fairness and speed. Unlike TCP, it avoids handshakes and sequential connections.
7.  Retransmitters are nodes in the system that keep a log of all outgoing messages from the matching engine. If clients miss a message, they can request it from the retransmitter, ensuring no information is lost.
8.  Consensus algorithms take too long because they require multiple network round trips between multiple nodes and can sometimes store data on disk. The system needs a solution that operates entirely in memory.
9.  The backup matching engine listens to all messages broadcast by the primary matching engine and updates its local state (limit order book) accordingly. This replication ensures that the backup engine has an up-to-date state and can take over if the primary engine fails.
10. Sharding by ticker prevents users from making conditional trades based on multiple tickers and prevents trades that span multiple tickers from being executed. This diminishes user abilities and can make the exchange less user-friendly.

Essay Questions
---------------

1.  Discuss the trade-offs between speed and reliability in the design of a low-latency stock exchange. How does the architecture described in the source material address these trade-offs?
2.  Explain the role of UDP multicast in the described stock exchange system. What are the benefits and drawbacks of using UDP multicast, and how are the drawbacks mitigated in this design?
3.  Describe the function of the retransmitter nodes, including how this component addresses a limitation in the matching engine.
4.  The source mentions state machine replication and how it is used in the secondary matching engine. In what ways can this replication speed up the functionality of the system?
5.  Explain the advantages and disadvantages of sharding the stock exchange system. Include in your answer ways that sharding would affect the exchange.

Glossary of Key Terms
---------------------

-   **Latency:** The delay before a transfer of data begins following an instruction for its transfer. In this context, it refers to the time it takes for an order to be processed and executed.
-   **Matching Engine:** The core component of a stock exchange responsible for matching buy and sell orders based on a specific algorithm (limit order book).
-   **Limit Order Book:** A record of all outstanding buy and sell orders for a specific security, organized by price.
-   **Spread:** The difference between the highest price a buyer is willing to pay for a security (bid) and the lowest price a seller is willing to accept (ask).
-   **UDP Multicast:** A network protocol that allows for the simultaneous transmission of data to multiple recipients without establishing individual connections.
-   **TCP:** Transmission Control Protocol is a connection-oriented communications protocol that provides reliable, ordered, and error-checked delivery of a stream of bytes between applications running on computers communicating via an IP network.
-   **Retransmitter:** A node in the system responsible for storing and re-broadcasting messages from the matching engine to clients who may have missed them.
-   **State Machine Replication:** A technique used to maintain a consistent state across multiple nodes by replicating all operations that cause state changes.
-   **Sharding:** Dividing a database or system into smaller, more manageable parts, often based on a specific attribute (e.g., ticker symbol).
-   **Ticker:** A unique abbreviation used to identify publicly traded shares of a particular stock on a particular stock market
-   **High-Frequency Trading (HFT):** A type of algorithmic trading characterized by high speeds, high turnover rates, and high order-to-trade ratios that uses sophisticated computer programs to analyze market data and execute trades.
-   **Consensus Algorithm:** A process in computer science used to achieve agreement on a single data value among distributed processes or systems. Zookeeper is an example of such an algorithm.
-   **Only Fans:** An Internet content subscription service.
-   **Karina Kopp:** An Internet content creator on Only Fans.


# Counting Unique Active Users: A System Design Deep Dive

**The YouTube video transcript presents a systems design problem: counting unique active users on a website at scale.** It considers challenges like multiple devices per user and the need for historical data. **The video explores various solutions, starting with naive database scans and progressing to more sophisticated approaches like using a hash map service.** It emphasizes the need to reduce space usage, ultimately advocating for the HyperLogLog algorithm for approximation. **This algorithm estimates unique counts by analyzing the rightmost zeros in user IDs, distributing the workload across multiple servers for efficiency.** The video also suggests a batch process for accurate counts and mentions the use of a user status table, Kafka, and Hadoop. **Finally, the video concludes by sharing plans for channel changes, expressing gratitude to viewers for their support and interaction.**

Briefing Document
-----------------

This document summarizes the key themes and ideas presented in the YouTube video transcript "Count Unique Active Users Design Deep Dive with Google SWE! | Systems Design Interview Question 26." The video focuses on designing a system to efficiently count the number of unique active users on a website, considering challenges related to scale, real-time requirements, and limited resources.

**Main Themes:**

-   **The Challenge of Counting Unique Users at Scale:** The core problem is efficiently counting unique users in a system with potentially billions of users and multiple devices per user. Standard approaches like full table scans and simple in-memory hashmaps become inadequate due to performance bottlenecks and memory limitations.
-   **Balancing Accuracy, Speed, and Resource Usage:** The design process involves trade-offs between the accuracy of the count, the speed of the query response, and the resources consumed by the system (e.g., servers, memory).
-   **Leveraging Approximation Algorithms:** The video explores the use of the HyperLogLog algorithm as a space-efficient approximation technique to address the memory constraints.
-   **Combining Real-time and Batch Processing:** The proposed architecture blends real-time approximate counting with background batch processing for accurate historical data and correction of approximations.
-   **Distributed Systems Design Principles:** The video applies core distributed systems concepts such as sharding, consistent hashing, load balancing, and data streaming to solve the problem.

**Key Ideas and Facts:**

-   **Functional Requirements:**Count unique active users on a website.
-   Handle a billion users with multiple devices each.
-   Provide a real-time endpoint for current unique user count.
-   Enable historical queries based on timestamps.
-   **Capacity Estimation:**Up to 1 billion users.
-   5-10 devices per user.
-   **API Design:**Single API endpoint to get the number of unique users.
-   Accepts an optional timestamp for historical data.
-   **Data Storage:**users table (SQL, e.g., MySQL) stores user ID, device ID, and online/offline status.
-   A Time-Series Database stores historical unique user counts.
-   **Bad Implementation:**"a bad implementation might be to actually do an entire scan on the database and then go ahead and return basically the number of unique users"
-   Full table scan on the users table is inefficient and resource-intensive.
-   **Improved (but Resource-Intensive) Implementation:**Using a hashmap or set to track unique user IDs in memory offers constant-time access.
-   Sharding the hashmap across multiple nodes with consistent hashing prevents duplicates.
-   **HyperLogLog Algorithm:**Approximates cardinality (number of unique elements) by analyzing the longest sequence of trailing zeros in hashed user IDs.
-   "it looks at the right most bits of those numbers and it's going to find the number in the set with the basically the most number of zeros as the rise right most bit"
-   Reduces space complexity to constant by only storing the user ID with the most trailing zeros seen so far.
-   **Distributed HyperLogLog:**Running multiple HyperLogLog instances on different servers and averaging the results improves accuracy.
-   Load balancer randomly assigns user IDs to servers to decrease variance.
-   **Batch Processing for Accuracy:**A background batch process (e.g., using Hadoop and Spark) performs an exact count from the database.
-   "parallel um batch process that runs in the background where the batch process since we don't really care about the amount of time that it takes to run can do an actual exact algorithm"
-   Populates the time-series database with accurate historical data, correcting HyperLogLog approximations.
-   **Proposed Architecture:Client:** Sends online/offline status updates or queries the count service.
-   **Status Service:** Updates the users table (MySQL).
-   **Kafka Queue:** Streams change data from the users table to Hadoop for batch processing.
-   **Load Balancer:** Distributes user IDs to HyperLogLog instances.
-   **HyperLogLog Instances:** Approximate the number of unique users in real-time.
-   **Hadoop Cluster:** Performs batch processing to calculate exact unique user counts.
-   **Time-Series Database:** Stores accurate historical unique user counts from Hadoop.
-   **Count Service:** Returns approximate counts from HyperLogLog or accurate counts from the time-series database.

**Key Quotes:**

-   "if you guys have seen from the thumbnail today we are going to be counting the unique number of users on a given website this problem could also be expanded to something like counting the unique number of search terms on google basically just counting the number of unique items where there are a ton of items such that this problem becomes tough at scale"
-   "because of the fact that there are up to a billion users that hash map is going to be pretty huge and as a result there's no guarantee we could even fit it on one single system"
-   "we're going to be using something called hyperloglog so hyperloglog basically does the following it says we have basically a bunch of user ids or a bunch of numbers and we're going to calculate how many of those numbers are unique"
-   "...we can ultimately perform an approximation that is going to allow us to in real time take in all of these user ids and spit back at least a semi-accurate approximation"

**Conclusion:**

The video provides a comprehensive overview of designing a system for counting unique active users at scale. It highlights the challenges of balancing accuracy, speed, and resource usage and demonstrates how approximation algorithms, distributed systems principles, and a combination of real-time and batch processing can be used to create a robust and efficient solution. The proposed architecture offers a practical approach to addressing this common systems design problem.

Study Guide Outline
-------------------

**I. Functional Requirements**

-   Counting unique users on a website or platform.
-   Extending the problem to counting unique search terms.
-   Handling a very large number of items (scalability).
-   Tracking unique users historically using timestamps.
-   Accounting for multiple devices per user.

**II. Capacity Estimation**

-   Up to one billion users.
-   Each user potentially having 5-10 devices.
-   Dealing with duplicate device entries.

**III. API Design**

-   Single API endpoint for getting the number of unique users.
-   Accepting a timestamp as a parameter for historical data.
-   Providing a real-time endpoint for current unique user count.

**IV. Database Tables**

-   Users table (user ID, online/offline status).
-   Table for historical unique user counts (timestamp, unique user count).

**V. Architectural Overview**

-   **Bad Implementation (Full Table Scan):**
-   Scanning the entire user table to count unique users.
-   Drawbacks: Database resource intensive, time-consuming, requires large memory, and slows down the system.
-   **Hash Map/Set Implementation:**
-   Storing unique user IDs in a hash map or set.
-   Benefits: Constant-time access for checking uniqueness.
-   Drawbacks: Potential memory issues with a billion users, possible overload for a single system.
-   **Sharded Hash Map with Consistent Hashing:**
-   Distributing the hash map across multiple nodes.
-   Using consistent hashing to ensure each user ID consistently goes to the same node.
-   Fixes duplicate counting across nodes.
-   **HyperLogLog Approximation Algorithm:**
-   Addressing the issue of high server costs and space requirements.
-   Overview of HyperLogLog:
-   Analyzing the rightmost bits of user IDs.
-   Finding the maximum number of consecutive rightmost zeros in the binary representation.
-   Estimating the number of unique elements as 2n, where n is the maximum number of rightmost zeros.
-   Distribution of the HyperLogLog Instances:
-   Using multiple instances of HyperLogLog on different servers.
-   Using a load balancer for random assignment of user IDs to servers.
-   Taking the average to decrease variance.
-   Mathematical bounds to establish the accuracy of HyperLogLog
-   **Batch Processing for Accuracy:**
-   Running a batch process in the background to calculate the exact unique user count.
-   Populating a time series database with accurate historical data.

**VI. System Diagram**

-   **Client:**Status Updates (Online/Offline).
-   Queries for the number of active users.
-   **Status Service:**User status table (MySQL).
-   Device ID, user ID, and online/offline status.
-   Streaming changes via Kafka to Hadoop for batch processing.
-   **HyperLogLog Instances (Buckets):**Load balancer distributes user ID changes.
-   Averages are aggregated back on the account service.
-   **Count Service:**Pulls numbers from HyperLogLog for approximate results.
-   Pulls numbers from a time series database for accurate results.
-   **Hadoop Cluster:**Batch processing for exact counts using Spark.
-   Populating the time series database.

**VII. Evaluation of the Diagram**

-   Parallel processing for approximate and accurate results.
-   Stream processing (HyperLogLog) for approximate results.
-   Batch processing (Hadoop) for accurate, fine-grained results.

Quiz
----

1.  What are the main functional requirements for the unique active user counting system?
2.  Why is performing a full table scan on a database to count unique users considered a bad implementation?
3.  Explain how sharding and consistent hashing can be used to implement a distributed hash map for counting unique users.
4.  Describe the basic principle behind the HyperLogLog algorithm for approximating unique counts.
5.  Why is randomization important when distributing HyperLogLog instances across multiple servers?
6.  How does HyperLogLog estimate the number of unique users based on the rightmost zeros of user IDs?
7.  How can the HyperLogLog approximation algorithm decrease space used?
8.  How does the system architecture use both stream processing and batch processing to provide both approximate and accurate results?
9.  What components are contained in the user status table?
10. What advantage does using hadoop with spark for batch processing have?

Quiz Answer Key
---------------

1.  The functional requirements include counting unique users and search terms at scale, tracking user counts historically via timestamp, and accounting for users with multiple devices. The system needs to provide both real-time and historical data.
2.  A full table scan consumes significant database resources, takes a long time to complete (especially with large tables), and requires substantial memory for storing user IDs, making it inefficient.
3.  Sharding distributes the hash map across multiple nodes to handle large datasets. Consistent hashing ensures that each user ID is consistently assigned to the same node, avoiding duplicate counting when aggregating results from different nodes.
4.  HyperLogLog estimates unique counts by analyzing the rightmost zeros in the binary representation of data elements. It finds the maximum number of consecutive rightmost zeros and estimates the cardinality of the set based on that value.
5.  Randomization (distributing HyperLogLog instances) is important to reduce the variance in the approximation. By randomly assigning user IDs to different instances, the overall estimate becomes more accurate.
6.  HyperLogLog finds the maximum number of consecutive rightmost zeros (n) in the binary representation of user IDs. It then estimates the number of unique users as 2n.
7.  The approximation algorithm only has to keep track of the user ID that they've seen with the most number of rightmost zeros and as a result of that all of the servers are only having to store one user ID at a time and it basically uses constant space complexity.
8.  Stream processing uses HyperLogLog for real-time, approximate results, while batch processing, using Hadoop/Spark, performs accurate calculations on historical data. The system combines these results to offer both speed and accuracy.
9.  The user status table contains device ID, user ID, and the user's online/offline status. It holds each user's current status.
10. Hadoop and Spark are used for efficient batch processing because they can handle large datasets effectively, perform complex queries, and provide data locality, ensuring accurate results for historical analysis.

Essay Questions
---------------

1.  Discuss the trade-offs between accuracy, latency, and resource consumption when choosing between a full table scan, a hash map implementation, and the HyperLogLog algorithm for counting unique users.
2.  Explain how consistent hashing addresses the challenges of distributing a hash map across multiple nodes in a system designed to count unique users. Provide a step-by-step example.
3.  Analyze the system architecture described in the source, focusing on the data flow from client requests to the eventual retrieval of unique user counts. Include the role of each component.
4.  Compare and contrast the advantages and disadvantages of using approximate algorithms like HyperLogLog versus exact counting methods in the context of a large-scale system. Consider various performance metrics.
5.  Design a similar system for counting unique search queries on a search engine. Adapt the concepts and technologies discussed in the source, justifying your design choices based on the specific challenges of this new use case.

Glossary of Key Terms
---------------------

-   **Cardinality:** The number of elements in a set.
-   **Consistent Hashing:** A technique that distributes data across a cluster in a way that minimizes the impact of adding or removing nodes.
-   **Full Table Scan:** Reading every row in a database table, which is inefficient for large tables.
-   **Hash Map (or Set):** A data structure that stores unique key-value pairs, providing fast lookups.
-   **Hadoop:** An open-source framework for distributed storage and processing of large datasets.
-   **HyperLogLog:** A probabilistic algorithm for approximating the number of distinct elements in a set (cardinality estimation).
-   **Load Balancer:** A device or software that distributes network traffic across multiple servers to prevent overload.
-   **MySQL:** An open-source relational database management system.
-   **Sharding:** Dividing a database or other data storage system into smaller, more manageable parts.
-   **Spark:** An open-source, distributed computing system used for big data processing and analytics.
-   **Timestamp:** A sequence of characters or encoded information identifying when a certain event occurred, usually giving date and time of day.
-   **Kafka:** A distributed, fault-tolerant, high-throughput streaming platform.
-   **API Endpoint:** A specific URL that a service exposes for clients to interact with its functionalities.
-   **Data Locality:** The closeness of data to the computation node that needs it.
-   **SLA:** Service Level Agreement.

# Distributed Message Broker Design

**The YouTube video presents a design deep dive into distributed message brokers, explaining their purpose in decoupling message publishing and consumption.** It outlines functional requirements, capacity estimations, and API design considerations for building such a system. **The video distinguishes between in-memory and log-based message brokers, detailing their respective advantages and disadvantages related to speed, durability, and consumer behavior.** Key architectural components like front-end services, load balancers, and coordination services (e.g., Zookeeper) are described. **The video also touches upon real-time communication protocols (suggesting long polling), replication strategies, and considerations for ensuring message processing.** The video concludes with a visual representation of a message queue architecture.

Briefing Document: 
------------------

**I. Overview**

This video explains the design of a distributed message broker, covering functional requirements, capacity estimates, API design, database schema considerations, and architectural choices. The speaker details two main types of message brokers: in-memory and log-based, outlining their trade-offs and common components. The video aims to provide an overview suitable for a systems design interview scenario.

**II. Key Concepts & Themes**

-   **Decoupling:** The primary purpose of a message broker is to "decouple the publishing and consumption of messages" allowing producers and consumers to interact asynchronously without direct connections. This enables independent scaling and fault tolerance.
-   **Scalability:** The design emphasizes scalability. The speaker notes that the goal is to build a broker that "can actually scale out to pretty much anything... you can just go ahead and add more of them."
-   **Functional Requirements:** The basic functional requirement is to "build me a message broker." This translates to enabling users to publish messages and consumers to retrieve them. The API includes:
-   PublishMessage(userID, messageContent) - Includes User ID for authentication and rate limiting.
-   ConsumeMessage(acknowledgement) - Optional acknowledgement parameter.
-   **Database Schema Considerations:** While a message broker is a kind of database, the speaker argues against using a traditional database engine. "Truthfully a database engine is going to allow you to do a lot more things than we're really going to need in a message broker." The focus is on efficient writing and deleting of messages, rather than the mutability offered by typical databases.

**III. Message Broker Types: In-Memory vs. Log-Based**

The video focuses on two primary types of message brokers, contrasting their storage mechanisms and use cases:

-   **In-Memory Message Broker:**
-   Stores messages in RAM.
-   Prioritizes speed and performance. "By keeping all of these messages in memory we can read them and write them much faster and as a result we should expect to have better performance."
-   Suitable for use cases where message order and specific consumer assignment are less critical, and speed is paramount.
-   Less durable; messages can be lost if the server crashes. "If that server were to crash we could potentially lose those messages and that would be very bad."
-   **Log-Based Message Broker:**
-   Leverages sequential writes to disk for durability. "A log-based message broker takes advantage of the fact that sequential writes to a disk are very fast."
-   Uses an append-only log. Messages are generally not deleted immediately.
-   Maintains consumer offsets to track read progress, ensuring at-least-once delivery.
-   Enables message replay. "Since we still have access to all those messages we can go ahead and replay them for our consumer."
-   Supports stateful consumers, as messages from a partition are delivered to the same consumer.
-   Slower than in-memory brokers due to disk I/O and acknowledgement waiting. "By using disk we are you know giving up a lot of latency."

**IV. Common Architectural Components**

Regardless of the chosen message broker type, the following components are generally required:

-   **Front-End Service:** Handles user authentication and rate limiting to prevent abuse.
-   **Load Balancer:** Distributes messages across partitions (queues). "We have to be able to kind of partition our cues in a way that certain messages are going to certain queues and other messages are going to other queues." Consistent hashing is used to ensure messages related to a specific topic or user ID go to the same partition.
-   **Coordination Service (e.g., ZooKeeper):** Manages the load balancer configuration, tracks the health of queue nodes using heartbeats, and ensures consistent hashing. ZooKeeper handles leader election and fault tolerance. "Zookeeper is basically going to tell the load balancer how to do consistent hashing based on you know which of the actual nodes holding cues are alive or not."
-   **Real-Time Communication Protocol:** Delivers messages to consumers. Long polling is suggested as a good option because the connection isn't maintained during message processing.

**V. Important Considerations**

-   **Exactly-Once Delivery:** Achieving exactly-once delivery is difficult and often involves significant overhead (e.g., two-phase commit). The speaker suggests using message IDs to allow consumers to detect and discard duplicate messages. "Perhaps what would be better is you just attach some sort of message id or you know request id to every single message and that way the consumers can actually go ahead and check whether they've seen them before when they're actually going ahead and processing those messages."
-   **Concurrency Control:** Thread-safe queues and mutex locks are needed to prevent race conditions when handling multiple concurrent messages, especially in in-memory brokers.
-   **Replication:** Single-leader replication is favored for its simplicity and fast write performance, though other replication strategies are possible.

**VI. Diagram Overview**

The speaker describes a visual architecture (though not explicitly provided here), which includes:

-   Clients hitting a Load Balancer.
-   An Authorization Service (horizontally scaled) behind the first Load Balancer.
-   A second Load Balancer distributing requests to the actual Queue Servers.
-   ZooKeeper managing the consistent hashing for the queue partitions.
-   The queues themselves (either in-memory or log-based).
-   Consumers using long polling to retrieve messages.

A Study Guide
-------------

### I. Key Concepts

**Message Broker:** An intermediary that translates messages between different protocols.

**Decoupling:** Separating the publishing and consumption of messages to avoid direct connections between producers and consumers.

**Functional Requirements:** What the system should *do*. In this case, to build a message broker.

**Capacity Estimates:** An estimation of system load (e.g., messages per minute, message size).

**API Design:** The structure and function of the interfaces (endpoints) for interacting with the system.

**Database Schema:** The structure of data stored within the system, including the need for internal databases.

**In-Memory Message Broker:** A message broker that stores messages in RAM for faster processing.

**Log-Based Message Broker:** A message broker that stores messages on disk in an append-only log.

**Durability:** The ability of a system to persist data and recover from failures.

**Partitioning:** Dividing the message stream into multiple queues for scalability and parallel processing.

**Stateful Consumption:** Consumers maintaining state based on the sequence of messages they receive from a specific partition.

**Front-End Service:** Handles user authentication and rate limiting.

**Load Balancer:** Distributes incoming requests across multiple queue servers.

**Coordination Service:** Manages the load balancer and ensures consistency across the distributed system (e.g., Zookeeper).

**Real-Time Communication Protocol:** The method used to deliver messages from the broker to the consumers (e.g., long polling, websockets).

**Two-Phase Commit (2PC):** A transaction protocol that ensures atomicity across multiple systems.

**Replication:** Creating multiple copies of data for fault tolerance and high availability.

**Single Leader Replication:** A replication strategy where one node is designated as the primary and all writes are directed to it.

**Consistent Hashing:** A technique for distributing data across a cluster of servers in a way that minimizes the impact of adding or removing servers.

**ZAB (Zookeeper Atomic Broadcast):** A consensus protocol used by Zookeeper to maintain consistency.

**Long Polling:** A technique where the consumer makes a request and the server holds the connection open until a message is available.

### II. Short Answer Quiz

1.  **What is the primary function of a message broker?** A message broker decouples the publishing and consumption of messages, acting as an intermediary to allow producers and consumers to communicate without direct connections. This enables asynchronous processing and scalability.
2.  **Explain the difference between an in-memory message broker and a log-based message broker.** An in-memory message broker stores messages in RAM for speed, but lacks durability. A log-based message broker writes messages to disk in an append-only log, providing durability but with potentially higher latency.
3.  **Why is partitioning important in a distributed message broker?** Partitioning allows the message stream to be divided into multiple queues, enabling horizontal scalability by distributing the load across multiple servers. This also supports parallel processing by multiple consumers.
4.  **What are the key responsibilities of the front-end service in the message broker architecture?** The front-end service is responsible for user authentication to ensure only authorized users can publish messages, and rate limiting to prevent abuse and maintain system stability.
5.  **What role does Zookeeper play in the message broker architecture?** Zookeeper acts as a coordination service, managing the load balancer through algorithms such as consistent hashing and ensuring consistency across the distributed system. It also monitors the health of the queue servers.
6.  **Why does the presenter suggest long polling as the preferred real-time communication protocol?** Long polling is preferred because it terminates the connection while the consumer is processing the message, reducing load on the queue server compared to persistent connections like websockets.
7.  **What problem is two-phase commit trying to solve in the context of message brokers, and why is it potentially not worth it?** Two-phase commit aims to ensure a message is delivered and processed exactly once, but it introduces significant latency and complexity with the transaction coordinator, making it less desirable.
8.  **Explain how an in-memory message broker can address the risk of message loss due to server failure.** While in-memory message brokers are inherently less durable, measures like replication or using a write-ahead log can be implemented to help guard against message loss from server failures.
9.  **Why is locking important within the in-memory message broker?** Locking is important because the broker handles multiple connections and attempts to push multiple messages onto the queue at the same time, so locking prevents race conditions.
10. **Describe the basic message queue architecture.** The client sends a message to a load balancer, which sends to an authorization service, which passes the message to partitions of queues upheld by Zookeeper coordination service. Then the consumer receives the message via long polling.

### III. Answer Key

1.  A message broker decouples the publishing and consumption of messages, acting as an intermediary to allow producers and consumers to communicate without direct connections. This enables asynchronous processing and scalability.
2.  An in-memory message broker stores messages in RAM for speed, but lacks durability. A log-based message broker writes messages to disk in an append-only log, providing durability but with potentially higher latency.
3.  Partitioning allows the message stream to be divided into multiple queues, enabling horizontal scalability by distributing the load across multiple servers. This also supports parallel processing by multiple consumers.
4.  The front-end service is responsible for user authentication to ensure only authorized users can publish messages, and rate limiting to prevent abuse and maintain system stability.
5.  Zookeeper acts as a coordination service, managing the load balancer through algorithms such as consistent hashing and ensuring consistency across the distributed system. It also monitors the health of the queue servers.
6.  Long polling is preferred because it terminates the connection while the consumer is processing the message, reducing load on the queue server compared to persistent connections like websockets.
7.  Two-phase commit aims to ensure a message is delivered and processed exactly once, but it introduces significant latency and complexity with the transaction coordinator, making it less desirable.
8.  While in-memory message brokers are inherently less durable, measures like replication or using a write-ahead log can be implemented to help guard against message loss from server failures.
9.  Locking is important because the broker handles multiple connections and attempts to push multiple messages onto the queue at the same time, so locking prevents race conditions.
10. The client sends a message to a load balancer, which sends to an authorization service, which passes the message to partitions of queues upheld by Zookeeper coordination service. Then the consumer receives the message via long polling.

### IV. Essay Questions

1.  Compare and contrast the trade-offs between in-memory and log-based message brokers, considering factors such as performance, durability, and use cases.
2.  Explain the role of each component in the message queue architecture (client, load balancer, authorization service, queue servers, Zookeeper, consumers) and how they interact to ensure a functional and scalable system.
3.  Discuss the challenges and solutions related to ensuring message delivery guarantees (at least once, exactly once) in a distributed message broker system.
4.  Analyze the factors that influence the choice of a real-time communication protocol (e.g., long polling, websockets, server-sent events) for delivering messages from the broker to the consumers, considering the specific requirements of the message broker system.
5.  Evaluate the importance of replication in a distributed message broker system and compare different replication strategies (single-leader, multi-leader, leaderless) in terms of their advantages and disadvantages.

# Twitch Systems Design: Live Streaming Architecture

**This YouTube transcript provides an overview of how to design a live streaming service like Twitch.** It begins by outlining requirements such as streaming, adaptive quality, recording, and chat. **The transcript explores various system design choices, focusing on networking considerations, data chunking, and aligning chunks for varying client network speeds.** It discusses delivering video to viewers using streaming servers and CDNs, addressing fault tolerance and scalability through consistent hashing and passive backups. **The latter half of the transcript covers recording streams with S3 and HBase, and then elaborates on different approaches for implementing chat functionality, particularly focusing on scalability using Cassandra.** The video concludes with a formal diagram showing how the various components interact.

Briefing Document: 
------------------

**Source:** Excerpts from "25: Live Streaming (Twitch) | Systems Design Interview Questions With Ex-Google SWE"

**Main Themes:**

-   **Core Functionality:** The primary goal is to enable streamers to broadcast live video and viewers to watch those streams with adaptive quality. Additionally, the platform needs to support recording streams for later playback and real-time chat functionality.
-   **Scalability and Performance:** The system must handle streams with potentially hundreds of thousands (or even millions) of concurrent viewers. Geographic distribution of viewers is a major consideration.
-   **Fault Tolerance:** The architecture needs to be resilient to failures, ensuring streams remain available even if individual servers or components fail.
-   **Network Considerations:** Balancing speed and reliability, the system needs to decide between UDP and TCP, and manage data transfer efficiently.
-   **Caching:** Implementing aggressive caching strategies is crucial for delivering content quickly to a large, distributed audience.
-   **Chat System Design:** The chat system needs to scale to handle high volumes of messages while maintaining reasonable consistency.
-   **Data Persistence:** The architecture needs to store video chunks and metadata for playback later using scalable storage.

**Key Ideas and Facts:**

1.  **Problem Requirements:**

-   Streamers must be able to stream.
-   Viewers need to be able to watch streams with adaptive quality (adjusting resolution based on network bandwidth).
-   Streams should be recordable and playable later.
-   Real-time chat functionality.
-   Streams can have hundreds of thousands of viewers, even millions.

1.  **Networking (TCP vs. UDP):**

-   The platform chooses TCP over UDP for video streaming, trading off some latency for reliability. The delay added to streams allows resending dropped packets, ensuring higher quality video.
-   "*Because of this delay now actually if I understand that I'm the streamer and I've dropped a few packets trying to send them to you I actually have time to go ahead and resend those so that you can get them back in time and basically recreate uh my full video stream without having to drop any frames.*"
-   The protocol used is rtmp or realtime message protocol.

1.  **Data Chunking:**

-   Video is broken into chunks to reduce overhead and simplify persistence.
-   Smaller chunks create more messages.
-   Larger chunks creates longer resends.
-   "*If we had to send a single message per frame to all of our clients uh that would be a lot of messages both for us to publish and for them to consume and there would be quite a bit of overhead there.*"
-   Chunk size needs to be carefully balanced. One second of video footage might be a good starting point.

1.  **Adaptive Streaming and Chunk Alignment:**

-   The system supports multiple resolutions (e.g., 480p, 720p, 1080p) and clients adaptively switch between them based on network conditions.
-   A metadata table aligns chunks across different resolutions, ensuring seamless transitions.
-   "*Let's imagine that we want to be able to support three different resolutions for our stream 480 720 and 1080p additionally we have all of these time stamps right this is chunk one this is chunk two chunk three and chunk four...*"
-   This prevents dropped or repeated frames when resolution changes.

1.  **Streaming Server and Encoding:**

-   Streamer PCs don't directly serve video to viewers. Instead, an intermediary streaming server handles this.
-   The streaming server encodes the video into multiple resolutions to support adaptive streaming. This offloads work from the streamer's PC.
-   "*Our streaming server now is actually going to basically take in the footage from the streaming PC it's going to encode it to a couple of different resolutions and then it's going to push it out to all of the interesting clients.*"

1.  **Fault Tolerance and Scalability of Streaming Servers:**

-   Consistent hashing (based on stream ID) is used to partition streams across multiple servers.
-   Two options for handling server failures:
-   Re-distribute streams from a failed server across the remaining servers.
-   Use passive backup servers that take over when a primary fails. The passive backup approach is preferred.
-   "*...if a stream server goes down basically offload all of the streams that were connected to it and you know evenly distribute them across all the existing stream servers...another possible option is that we actually run a bunch of passive stream servers which basically sit there doing nothing listening to zookeeper...*"

1.  **Content Delivery Network (CDN) and Caching:**

-   CDNs are essential for delivering video chunks quickly to geographically distributed viewers.
-   A separate cache (e.g., Redis) is used for chunk metadata.
-   "*...we can actually put them all over the globe additionally keep in mind that besides just having to know about the static content all of these viewers actually have to be able to read the metadata as well for each chunk and its offset and so we can just use a normal cash or something like Reddit for all of those chunk metadata rows.*"
-   Data can be pushed to the CDN or pulled in after an initial cache miss.

1.  **Handling "Thundering Herd" Problems:**

-   Stream raids can cause a sudden surge of viewers to an unpopular stream, leading to a "thundering herd" where many clients simultaneously request data from the origin server (encoding server).
-   A lock mechanism on the cache prevents the Thundering Herd problem.
-   "*...what could happen is it can lead to a Thundering Herd so what would this Thundering Herd look like like I mentioned imagine we have some unpopular streamer and now they have a lot of viewers within this uh the span of a couple of seconds...all of them at the same exact time are going to hit the cach and the cach is going to say I don't have data all at once...*"

1.  **Recording Streams:**

-   Video chunks are stored in S3 for later playback.
-   Metadata is stored in HBase.
-   HBase is partitioned by stream ID and sorted by timestamp to optimize reads.
-   "*... we just need to basically be able to access all of that metadata down the line and then from there uh pull the actual chunks out of a data store so what do we want to store our chunks in well probably S3 it's just a given...*"

1.  **Chat System Design:**

-   For smaller streams, a websocket-based chat server is used for real-time communication. Consistent hashing is used to assign users to chat servers.
-   For larger streams, a polling-based approach with Cassandra is used to handle high message volumes.
-   Websockets used for smaller chats so send and receive messages in real time.
-   "*... if this stream gets too popular what they're ultimately going to start doing is both writing to and reading from this guy over here which is our Cassandra database which as we know is a leader list database that we're going to Partition by stream ID and sort all of our messages by their timestamp.*"
-   Cassandra allows for high write throughput and scalability, but might not guarantee strict causal consistency. The lack of causal consistency is considered acceptable for Twitch chat.
-   Clients poll database every few seconds for new chat messages.
-   The system uses Zookeeper to switch between the websocket-based and polling-based chat systems based on stream popularity.

**Diagram Overview (Formal Diagram):**

1.  **Streamer:** Sends 1-second video chunks.
2.  **Load Balancer & Consistent Hashing:** Routes the stream to an appropriate encoding server.

-   **Encoding Server:**Encodes the video into multiple resolutions.
-   Puts video chunks in S3 for recording.
-   Puts metadata in HBase.
-   Populates caches (Redis metadata cache and CDN for video content).
-   **Viewers:**Access video from CDN.
-   Access metadata from Redis cache.
-   **Chat (Unpopular Streams):**Viewer connects to a chat server via a load balancer and websocket.
-   **Chat (Popular Streams):**Viewer reads from and writes to a Cassandra database.
-   Zookeeper manages switching between websocket and Cassandra based on stream popularity.

**In summary,** the document outlines a comprehensive systems design for a live streaming platform, addressing the challenges of scalability, fault tolerance, and efficient content delivery. The key takeaways are the use of TCP for reliable video transfer, adaptive streaming with chunk alignment, a CDN for caching, and a scalable chat system that adapts to stream popularity.

Design Study Guide
==================

Review of Key Concepts
----------------------

-   **Core Streaming Functionality:** Streamers broadcasting live, viewers watching with adaptive quality, recording, and real-time chat.
-   **Scalability is Critical:** The system must handle hundreds of thousands (or millions) of concurrent viewers with minimal latency.
-   **Fault Tolerance:** The system must be resilient to component failures.
-   **Network Protocol Choice (TCP vs. UDP):** TCP was chosen to favor reliability over speed.
-   **Data Chunking:** Breaking video into chunks for efficient transfer and storage.
-   **Adaptive Streaming:** Providing different resolutions to viewers based on their network conditions.
-   **Chunk Alignment:** Maintaining seamless transitions between resolutions.
-   **Stream Encoding:** The streaming server handles encoding the video into multiple resolutions.
-   **Content Delivery Networks (CDNs):** Caching video chunks closer to viewers for reduced latency.
-   **Caching:** Delivering content quickly to a large, distributed audience.
-   **Database selection:** Choosing database technology for different requirements.
-   **Chat System Scalability:** Handling high message volumes while maintaining reasonable consistency.
-   **Data Persistence:** Storing video chunks and metadata for later playback.

Short Answer Quiz
-----------------

1.  Why is TCP chosen over UDP for video streaming in this Twitch design, and what tradeoff does this decision involve?
2.  Explain the purpose of data chunking in a live streaming system and what factors influence the choice of chunk size?
3.  What is adaptive streaming, and how does chunk alignment contribute to a better viewing experience when switching resolutions?
4.  Why is it preferable to use a streaming server instead of having streamers directly serve video to viewers?
5.  Describe two strategies for ensuring fault tolerance in the streaming server infrastructure.
6.  What is the purpose of a CDN in the context of a live streaming platform, and how does it improve performance for geographically distributed viewers?
7.  Explain the "Thundering Herd" problem in caching and how it can be mitigated in a live streaming system.
8.  What storage solutions are used for storing video chunks and metadata, and why were these solutions chosen?
9.  Describe how the chat system adapts to the size of a stream's audience, and what technologies are used in each approach?
10. Describe the role of Zookeeper in the Twitch architecture and what component(s) is/are using it.

Answer Key to Quiz
------------------

1.  TCP is chosen over UDP for video streaming to prioritize reliability over speed. The tradeoff is increased latency due to the retransmission of lost packets, but this ensures higher video quality because missing packets can be resent.
2.  Data chunking reduces overhead, simplifies persistence, and enables adaptive streaming. Chunk size is influenced by balancing the number of messages and resending larger files.
3.  Adaptive streaming provides different resolutions to viewers based on network conditions, giving them a seamless transition and good experience. Chunk alignment ensures seamless transitions between resolutions without dropped or repeated frames.
4.  Using a streaming server offloads the encoding process from the streamer's PC, allowing them to focus on streaming a better broadcast. The load is minimized on their PC and distributing the stream to many viewers.
5.  One strategy is consistent hashing across multiple servers, and another is using passive backup servers that take over when a primary server fails. Passive backup servers are preferred.
6.  A CDN caches video chunks closer to viewers, reducing latency and improving performance for geographically distributed viewers.
7.  The "Thundering Herd" problem occurs when a sudden surge of viewers causes many clients to simultaneously request data from the origin server. A lock mechanism on the cache stops the encoding server from being overloaded.
8.  Video chunks are stored in S3 because it's very scalable, and metadata is stored in HBase because it has fast writes.
9.  For smaller streams, a websocket-based chat server is used. For larger streams, a polling-based approach with Cassandra is used to handle high message volumes and uses Zookeeper to switch between Websocket-based and polling-based systems.
10. Zookeeper manages switching between the websocket-based and polling-based chat systems based on stream popularity.

Essay Questions
---------------

1.  Discuss the trade-offs involved in choosing between TCP and UDP for a live streaming platform like Twitch. Justify the selection of TCP and describe alternative scenarios where UDP might be more appropriate.
2.  Explain the architecture of the Twitch chat system, detailing how it adapts to different stream sizes. Analyze the benefits and drawbacks of each approach (WebSockets vs. Cassandra polling) and propose a hybrid solution.
3.  Describe the process of adaptive streaming in a live video platform. How does the system determine when to switch resolutions for a viewer, and what mechanisms ensure a seamless transition?
4.  Evaluate the use of CDNs in a live streaming architecture. Discuss the strategies for populating CDNs (push vs. pull) and how they impact the viewing experience, particularly during stream raids or events that cause sudden surges in viewership.
5.  Analyze the data persistence strategy for video chunks and metadata in the Twitch system design. Explain the rationale behind choosing S3 for video chunks and HBase for metadata, and discuss alternative storage solutions that could be used.

Glossary of Key Terms
---------------------

-   **Adaptive Streaming:** The method of delivering video content in multiple resolutions, allowing clients to switch between them based on their network conditions.
-   **CDN (Content Delivery Network):** A geographically distributed network of servers that caches content closer to users, reducing latency.
-   **Chunk Alignment:** The process of synchronizing video chunks across different resolutions, ensuring seamless transitions when switching between them.
-   **Consistent Hashing:** A technique used to distribute data or requests across a set of servers in a way that minimizes disruption when servers are added or removed.
-   **Data Chunking:** Breaking down video content into smaller segments for efficient transfer and storage.
-   **Encoding Server:** A server that converts video into multiple resolutions and formats for adaptive streaming.
-   **Fault Tolerance:** The ability of a system to continue operating correctly even if one or more of its components fail.
-   **HBase:** A NoSQL, column-oriented database designed for fast read/write access to large datasets.
-   **Latency:** The delay between a request and a response in a network or system.
-   **Load Balancer:** A device or software that distributes network traffic across multiple servers to prevent overloading any single server.
-   **Metadata:** Data about data, such as timestamps, resolutions, and storage locations of video chunks.
-   **RTMP (Real-Time Messaging Protocol):** A protocol designed for streaming audio, video, and data over the Internet.
-   **S3 (Simple Storage Service):** A scalable object storage service offered by Amazon Web Services (AWS).
-   **Scalability:** The ability of a system to handle increasing amounts of traffic or data.
-   **Stream ID:** A unique identifier for a live stream.
-   **TCP (Transmission Control Protocol):** A reliable, connection-oriented protocol that guarantees delivery of data in the correct order.
-   **Thundering Herd:** A situation where a sudden surge of requests overloads a system, often due to cache misses.
-   **UDP (User Datagram Protocol):** A connectionless protocol that is faster than TCP but does not guarantee delivery or order of data.
-   **WebSockets:** A communication protocol that provides full-duplex communication channels over a single TCP connection.
-   **Zookeeper:** A centralized service for maintaining configuration information, naming, providing distributed synchronization, and group services.
-   **LSM Tree (Log-Structured Merge Tree):** A data structure used in databases, optimized for write-heavy workloads by buffering writes in memory and then periodically merging them into larger, sorted files on disk.
-   **Leaderless Replication:** A database replication strategy, where writes can be sent to any replica node and they coordinate the distribution of the data, without needing a central leader.
-   **Quorum Consistency:** A consistency model in distributed systems that requires a majority of nodes to acknowledge a write before it is considered successful, ensuring a certain level of data consistency.
-   **Anti-entropy:** A process in distributed databases that compares data across replicas and reconciles any differences, ensuring consistency over time.

# Designing a Scalable Web Crawler System

**The YouTube transcript presents a system design interview question focused on designing a web crawler.** The video explores the challenges involved in scraping, storing, and respecting website policies while efficiently processing a large amount of online data within a week. **It outlines the core components of a web crawler, including a "to crawl" list, robots.txt compliance checks, DNS resolution, content fetching, and duplicate content detection.** The speaker considers methods for optimizing the crawling process, such as distributed frontiers, content hash checking and DNS caching. **The discussion leads to a proposed architecture utilizing Kafka and Flink for a fault-tolerant and scalable web crawling system with S3 for storage.**

Briefing Document
-----------------

This document summarizes the key ideas and themes from the provided source, a discussion on designing a web crawler, similar to those used by Google or OpenAI. The discussion focuses on system design interview questions and provides a practical approach to building a scalable and efficient web crawler.

**I. Core Problem & Requirements:**

The primary goal is to design a system that can "crawl the web," meaning gathering a large amount of publicly available data from the internet. This data can be used for various purposes, including:

-   Building a search index (like Google).
-   Training large language models (like OpenAI).

The discussed requirements include:

1.  **Scrape all accessible online content.**
2.  **Store all the content.** While optional, the discussion considers the storage of the crawled data important for training and indexing purposes.
3.  **Respect website crawling policies (robots.txt).** Crawlers must adhere to the rules specified in robots.txt files to avoid violating website terms of service. "If you're a bot crawling my site here's what you can do here's what you can't do you don't want to violate those it's not right."
4.  **Complete the process within a week.** Given the vast amount of data on the web, this necessitates a distributed and parallel approach. "The web has a ton of data it's not that easy to just crawl it we're going to need a lot of computers probably to get this done."

**II. Capacity Estimation:**

A rough estimation of the required resources is performed:

-   **Number of web pages:** Assumed to be around 1 billion.
-   **Average web page size:** Assumed to be 1 MB.
-   **Total data to store:** Estimated to be 1 petabyte.
-   **Requests per second:** To crawl the web in a week, approximately 1,500 requests per second are needed.
-   **Number of servers:** Based on loading time per page and CPU core usage, approximately 100 servers might be needed. "Maybe we need what around 100 servers if we want to devote a decent amount of CPU usage to this."

**III. Web Crawling Process:**

The fundamental steps involved in web crawling are:

1.  **Fetch URL from the Frontier (to-crawl list):** A list of URLs to be crawled is maintained. The discussion refers to this list as the "frontier."
2.  **Check if already crawled:** Avoid duplicate work by checking if the URL has already been processed.
3.  **Check robots.txt compliance:** Ensure crawling the URL is allowed by the website's crawling policy.
4.  **Resolve IP Address (DNS):** Convert the hostname (e.g., twitter.com) to an IP address using a DNS server.
5.  **Fetch content (HTTP Request):** Send an HTTP request to the resolved IP address and retrieve the content.
6.  **Check for duplicate content:** Compare the content's hash with a database of already processed content to avoid storing duplicates, even if they originate from different URLs. "We have to check if we've already processed identical content from another URL so this is actually something that's apparently pretty common on the web."
7.  **Parse Content:** Analyze the retrieved content, typically HTML, and extract relevant information.
8.  **Store Results:** Store the parsed content in an object store (e.g., S3) or a distributed file system.
9.  **Extract URLs and add to Frontier:** Identify all URLs referenced in the HTML content and add them to the frontier for future crawling.

**IV. Optimization Techniques:**

The discussion explores several optimization strategies to improve the crawler's performance:

-   **Distributed Frontier:** Distributing the "to-crawl" list (frontier) across multiple nodes. Having a distributed frontier avoids a single point of failure and allows parallel processing.
-   **Local Frontier (per node):** Faster retrieval but suffers from load imbalance and potential duplicate processing.
-   **Centralized Database for URL tracking:** Avoids duplicates, but introduces network latency for each check.
-   **Partitioned Frontier (by Hash Range of URL):** Distributes URLs to specific nodes based on a hash of the URL. Allows for local caching of processed URLs.
-   **Duplicate Content Detection:Centralized Database for Content Hashes (Redis):** Stores hashes of already processed content. Allows for quick lookups to avoid storing duplicate content. "Considering that 8 GB is actually pretty small in theory we can store this entire data set on one node in memory and so that is actually a pretty practical solution right if we just had some reddis instance we could store all of our hashes there."
-   **Decentralized Set CRDT:** Using a Conflict-Free Replicated Data Type (CRDT) on each node for storing content hashes. Offers eventual consistency and avoids centralized bottleneck but may lead to occasional duplicate processing.
-   **DNS Optimization:Caching DNS Results per Host:** Since DNS entries are per host (e.g., wikipedia.com), caching these results on processing nodes can significantly reduce DNS lookups, especially when a website links to other pages within the same host. Partitioning by host helps enable this optimization.
-   **robots.txt Optimization:Caching robots.txt Files per Host:** Similar to DNS, caching robots.txt files per host enables faster policy checks. Partitioning nodes by host allows each node to maintain a local cache of the crawling policy for each host it is responsible for.
-   **Re-queuing URLs Based on Crawl Delay:** If robots.txt specifies a crawl delay, re-queue the URL back onto the frontier instead of blocking.
-   **Data Locality for Storage:**Storing results in a location physically close to the processing servers (e.g., using S3 in the same region as the EC2 instances) reduces latency for writing the crawled data.
-   **Frontier Modeling:Breadth-First Search (BFS) using a Queue:** Websites tend to link to other pages within the same host; caches are expected to get more hits as opposed to misses.
-   **Depth-First Search (DFS) using a Stack:** Recursion is difficult in a distributed setting.
-   **Priority Queue:** Includes a quality factor for every URL to crawl in preferential order; the discussion notes that this implementation may be out of the scope for the interview.
-   **Technology Stack:Kafka:** For reliable message queuing and persistence.
-   **Flink:** For stream processing, state management, and fault tolerance. "Flink of course is going to allow us to keep local state on that consumer that is Fault tolerant we can keep that state in memory it's okay if Flink goes down because occasionally it's going to be checkpointing."
-   **Redis:** As a fast, in-memory data store for caching content hashes.
-   **S3:** For storing the crawled content.

**V. Final Design:**

The final design involves:

-   A cluster of processing nodes, ideally collocated with Kafka queues and Flink instances.
-   A load balancer distributing URLs to processing nodes based on the host name.
-   Each node caching DNS results and robots.txt files per host.
-   A Redis cache (or a decentralized CRDT) for detecting duplicate content.
-   S3 for storing the crawled data.

**VI. Key Takeaways for System Design Interviews:**

-   Clearly define the problem and requirements.
-   Estimate capacity and resource needs.
-   Outline the core crawling process.
-   Identify and discuss various optimization techniques.
-   Justify technology choices based on scalability, reliability, and performance.
-   Consider trade-offs between different approaches.
-   Focus on data locality and caching to minimize network latency.

Study Guide
===========

Quiz
----

Answer each question in 2-3 sentences.

1.  What are the four key problem requirements for designing a web crawler, as discussed in the source?
2.  Why is estimating the number of computers needed to crawl the web important, and what factors influence this estimation?
3.  Describe the role of the "frontier" in the web crawling process, and explain why it's also called a "to crawl list."
4.  Explain the purpose of checking the robots.txt file and how it affects the web crawling process.
5.  Why is DNS resolution a necessary step in web crawling, and what does it accomplish?
6.  Describe why it is important to check for identical content from another URL, and why is this step performed after fetching the content?
7.  What are the potential downsides of storing the frontier locally on each node during web crawling?
8.  Explain how hash range partitioning can help avoid duplicate fetches of URLs in a distributed web crawler.
9.  What are the challenges associated with detecting duplicate content on different websites, and why can't nodes be partitioned based on content before fetching it?
10. Describe the centralized solution using Redis for content hash checking.

Quiz Answer Key
---------------

1.  The four key problem requirements are to scrape all accessible online content, store all of the scraped content, respect website crawling policies (robots.txt), and complete the process within a week. These requirements ensure comprehensive data collection while adhering to ethical and legal constraints.
2.  Estimating the number of computers is crucial for resource allocation and meeting the time constraint. Factors influencing this estimation include the number of web pages, the average size of each page, the speed of requests, and the CPU power of each server.
3.  The frontier is the list of URLs that need to be crawled, and it's critical for managing the crawling process. It is called the "to crawl list" because it represents the next set of URLs to be traversed, similar to the frontier in graph traversal algorithms.
4.  Checking the robots.txt file ensures that the crawler respects the website's policies regarding which pages can be accessed and how frequently. This step prevents the crawler from overloading the server and avoids violating ethical crawling practices.
5.  DNS resolution translates a human-readable domain name (e.g., wikipedia.com) into an IP address, which is needed to establish a connection with the server. Without this step, the crawler cannot locate and fetch content from the web server.
6.  Checking for identical content saves storage space and processing time by avoiding redundant indexing or training of the same data. This step is performed after fetching the content because the crawler needs to analyze the HTML to determine if it matches previously processed content.
7.  Storing the frontier locally can lead to uneven load balancing, where some nodes are overloaded while others are idle. It can also result in duplicate processing of links if multiple nodes have the same URL in their local frontiers.
8.  Hash range partitioning ensures that the same URL is always processed by the same node, enabling each node to maintain a local cache of already processed URLs. This avoids redundant database queries and reduces network calls.
9.  The main challenge is that the content hash is unknown before fetching the content, so you cannot partition the nodes based on the content hash in advance. As such, partitioning nodes based on content cannot be done smartly, as the content isn't known at the point when it is sent to the nodes.
10. The centralized solution uses a Redis instance to store content hashes, allowing each processing node to quickly check if it has already processed a particular piece of content. Although it is a quick approach, it could be slow for subsets of the processing nodes in geographically different locations.

Essay Questions
---------------

1.  Discuss the trade-offs between using a centralized database and a distributed system for managing the frontier in a web crawler. Consider factors such as consistency, scalability, and network latency.
2.  Explain how partitioning strategies can optimize a web crawler's performance. Compare and contrast partitioning by URL, host, and content hash, highlighting the advantages and disadvantages of each approach.
3.  Analyze the importance of respecting crawling policies and the challenges involved in implementing robots.txt compliance in a distributed web crawler.
4.  Evaluate the effectiveness of different caching mechanisms (DNS, robots.txt, content hashes) in reducing network load and improving the efficiency of a web crawler.
5.  Describe the final web crawler design, focusing on the roles of Kafka, Flink, Redis (or CRDT), and S3. Explain how these components work together to ensure a reliable and scalable crawling process.

Glossary of Key Terms
---------------------

-   **Web Crawler:** An automated program that systematically browses the World Wide Web to collect and index information.
-   **Robots.txt:** A text file on a website that instructs web robots (crawlers) which parts of the site should not be crawled.
-   **DNS Resolution:** The process of translating a domain name (e.g., wikipedia.com) into an IP address.
-   **Frontier (To Crawl List):** A queue or list of URLs that the web crawler intends to visit and process.
-   **Hash Range Partitioning:** A technique for distributing data or tasks across multiple nodes based on the hash of a key (e.g., URL), ensuring even distribution.
-   **Content Hash Checking:** The process of generating a unique fingerprint (hash) of the content of a web page and comparing it against previously seen hashes to avoid duplicate processing.
-   **CRDT (Conflict-Free Replicated Data Type):** A data structure that can be replicated across multiple nodes and updated independently without requiring coordination, guaranteeing eventual consistency.
-   **Kafka:** A distributed, fault-tolerant streaming platform often used as a message broker.
-   **Flink:** A distributed stream processing framework that allows for stateful computations and fault tolerance.
-   **Item Potency:** The property of an operation such that it can be performed multiple times without changing the result beyond the initial application.

# Amazon Systems Design: Retail Cart and Order Service

**The YouTube transcript presents a systems design interview question focused on building a limited feature set of Amazon or Flipkart's retail site.** **The discussion centers on designing a cart service, inventory management, and ensuring speed and scalability, excluding features like reviews and payments.** **Capacity estimates are provided, along with considerations for handling a petabyte of product data and optimizing read speeds to improve conversion rates.** **Key topics discussed encompass the products database, cart service implementation using Conflict-Free Replicated Data Types (CRDTs), order processing via Kafka and Flink, and strategies for optimizing reads by caching popular items and implementing a search index using Elasticsearch.** **The transcript also details the flow of data through the system, from product searches to order submissions, highlighting various database and caching technologies.** **Finally, the explanation covers processes for computing the popularity of a given product, batch processing considerations, and the use of caching and Elasticsearch.**


Briefing Document
-----------------

This document summarizes the key themes and ideas presented in the provided transcript of a systems design interview question focused on building a simplified Amazon/Flipkart retail site. The emphasis is on scalability, speed, and handling potential bottlenecks.

**I. Overview & Requirements:**

-   The problem involves designing a system to handle product browsing, adding items to a cart, and placing orders, focusing on the cart service and inventory management.
-   **Scope Limitations:** The design explicitly excludes features like reviews (covered in a previous Yelp design), payments (deserves its own video), Amazon Web Services integration, and customer service.
-   **Key Goal:** To create a fast and scalable system, prioritizing both read and write performance, especially for product loading and checkout. "Apparently and I've heard this in reference to Amazon and other e-commerce sites even so much as 100 milliseconds of a delay on our load times can really hurt our conversion rates in our site and so as a result we want to make reads as fast as possible wherever we can."
-   The design assumes a very high level of concurrency and potential contention, even if realistically less likely, to thoroughly explore system design fundamentals.

**II. Capacity Estimates (Handwavy):**

-   **Products:** 350 million unique products (rounded to 1 billion for scale).
-   **Orders:** 10 million orders per day (approximately 100 orders per second).
-   **Data per Product:** 1 MB of data per product (including images).
-   **Total Data:** 1 Petabyte of product data, necessitating partitioning.
-   **Images:** Hosted on S3 with CDN for faster load times.

**III. High-Level Design & Key Components:**

1.  **Products Database:**

-   **Challenge:** Storing and retrieving product information (1 PB).
-   **Solution:** Partitioned MongoDB.
-   **Rationale:**Single leader replication is sufficient because writes (product updates) are infrequent. "I think single leiter replication should be fine I think that um using a b Tree in theory is going to be optimal..."
-   Flexible schema (NoSQL) is beneficial due to diverse product types and data. "This is one of the very few cases where I'm interested in no SQL solely for the data model or the flexibility there and uh I think that mongod DB is going to be the choice here..."
-   B-tree index in MongoDB is preferred for faster reads.
-   Data locality (loading the full JSON document at once) improves performance.
-   Minimal denormalization is required in this simplified version.

1.  **Cart Service:**

-   **Challenge:** Handling concurrent cart modifications from multiple users.
-   **Bad Implementation 1:** Overwriting the entire cart with each update. Problematic because concurrent writes will overwrite each other.
-   **Bad Implementation 2:** Appending to a list. Problematic because appending requires a lock on the list, leading to contention.
-   **Better Implementation:** Using a set (table with card ID and product ID). Avoids list-based locking. However deletion of items would still require locking.
-   **Best Implementation:** Using a Conflict-Free Replicated Data Type (CRDT) with multiple database leaders. "In this case we want to use such a crdt to build a set amongst our leaders so the idea is we're going to have a crdt on every single one of these leaders each of them is going to basically have its own copy of what it thinks is in the cart and then eventually those guys will sync up with one another and then they should all converge to the same value"
-   An Observe-Remove set CRDT is used to handle add/delete/re-add scenarios.
-   Each add/remove operation is tagged with a unique UUID. This tag is used to determine the eventual state of the set when the leaders sync.
-   Clients should read from the same leader they write to, ensuring they see their own changes.

1.  **Order Service:**

-   **Challenge:** Ensuring sufficient stock when placing an order, avoiding overselling.
-   **Problem:** Concurrent decrement operations on stock counters require locks, potentially leading to contention.
-   **Sub-optimal Solution:** Counter CRDT. The true value is only known once the leaders sync.
-   **Solution:** Kafka queue for buffering orders and Flink (stream processing) for processing them one at a time.
-   Orders are placed in a Kafka queue partitioned by product ID.
-   Flink splits the order into individual product orders and routes them to the appropriate Kafka queue.
-   Flink node checks current stock and decrements it. Sends email to customer and updates product database if out of stock.
-   Inventory changes (from sellers) also flow through Kafka to update Flink's cached stock levels.
-   Guarantees atomicity of order processing.

1.  **Optimizing Reads:**

-   **Goal:** Reduce latency for product loading, improving conversion rates.
-   **Strategy:** Caching popular items.
-   **Determining Popularity:** Aggregate metrics (orders, clicks, reviews) over a time window.
-   **Implementation:** Spark streaming to window data and export metrics to HDFS.
-   Spark job ranks items by popularity, creates popularity scores, and populates Redis cache (for product data) and CDN (for images).
-   **Text Search Optimization:** Implementing a search index is completely necessary. An inverted index maps search terms to product IDs.
-   **Local indexing** is chosen, where each node contains a subset of the products. "so let's talk about local indexing because that is really our only alternative to Global indexing the idea here is that basically we put a set of products on every single uh node of our search index and then we index them locally so we just build that inverted index locally right off of that set of products"
-   Products are partitioned by product category and within each category by popularity score range, ensuring relevant products are on the same partition for efficient pagination.
-   **Updating Caches and Indexes:** Create a standby cluster and populate it from spark. Then, update the coordination service (zookeeper) and broadcast the changes to all the existing servers.

**IV. Overall Flow:**

1.  **Search:** Hit ElasticSearch (sharded by product type/tag and popularity score range).
2.  **Product Service:** If product is in product cache (Redis), return it. Otherwise, retrieve from MongoDB (sharded by product ID).
3.  **Cart Service:** Update cart DB (Riak, supporting CRDTs) with cart ID, product ID, deletion status, and unique tag. Partition by cart ID.

-   **Order Service:**Send full order to full orders Kafka queue.
-   Flink splits the order and routes products to product-specific Kafka queues.
-   Flink node performs inventory checks (using inventory changes from Kafka) and sends confirmation email.
-   **Popularity Update:**Flink node aggregates order data (sharded by product ID).
-   Spark batch job updates product cache, ElasticSearch (standby cluster), and Zookeeper to switch over to the new cluster.

**V. Key Takeaways:**

-   The importance of prioritizing read performance for e-commerce sites.
-   Strategies for handling concurrent writes and potential lock contention (CRDTs, Kafka/Flink).
-   The use of different database technologies based on specific requirements (MongoDB for flexible product data, Riak for CRDT cart management).
-   The value of stream processing (Flink) for inventory management and batch processing (Spark) for popularity calculations.
-   Importance of a properly sharded and indexed data model.

Study Guide
===========

I. Quiz: Short Answer Questions
-------------------------------

1.  **Why is a flexible schema beneficial for an Amazon products database?** Different product categories require different attributes and pieces of information; thus, a flexible schema allows for accommodating the diversity of products sold on Amazon.
2.  **What are the drawbacks of storing a shopping cart as a simple list and appending to it?** Appending to a list requires grabbing locks to maintain order, leading to potential contention if multiple users try to edit the same cart simultaneously.
3.  **What is a CRDT, and how can it be useful for a cart service?** A Conflict-Free Replicated Data Type (CRDT) is a data structure that ensures eventual consistency across multiple replicas, allowing multiple database leaders to sync up without conflicting writes, beneficial when multiple users edit the same cart.
4.  **Explain the purpose of the tags used in the "Observe and Remove Set" CRDT.** The tags, typically UUIDs, uniquely identify each addition or removal operation, ensuring the system can accurately resolve conflicts and converge to a consistent state even with concurrent add/remove operations.
5.  **Why is locking required when decrementing the stock count during order processing?** Locking prevents race conditions when multiple processes try to decrement the stock count simultaneously, ensuring accurate inventory management.
6.  **Why isn't Kafka, on its own, appropriate for ensuring successful checkout operations?** Because Kafka allows asynchronous processing, if an item is added to a cart and then the user attempts to check out, the item might not be present when the check out operation begins.
7.  **Explain the role of Flink in the order processing pipeline described.** Flink is used for stream processing, ensuring every message from Kafka is processed, guaranteeing every order is checked for stock and emails are sent.
8.  **What data is sent into the "full orders" Kafka queue?** The complete order details, which guarantees atomicity and ensures the entire order is eventually processed by the system.
9.  **Why are popular items cached in Redis and their images in a CDN?** Caching in Redis and using a CDN speeds up read times for popular items, improving site performance and conversion rates.
10. **How does local indexing speed up searches in a system like Amazon?** Local indexing means each node in the search index contains a subset of products, indexed locally, and partitioning by category and popularity score allows each node to return relevant results quickly, facilitating pagination.

### Quiz Answer Key

1.  Different product categories require different attributes and pieces of information; thus, a flexible schema allows for accommodating the diversity of products sold on Amazon.
2.  Appending to a list requires grabbing locks to maintain order, leading to potential contention if multiple users try to edit the same cart simultaneously.
3.  A Conflict-Free Replicated Data Type (CRDT) is a data structure that ensures eventual consistency across multiple replicas, allowing multiple database leaders to sync up without conflicting writes, beneficial when multiple users edit the same cart.
4.  The tags, typically UUIDs, uniquely identify each addition or removal operation, ensuring the system can accurately resolve conflicts and converge to a consistent state even with concurrent add/remove operations.
5.  Locking prevents race conditions when multiple processes try to decrement the stock count simultaneously, ensuring accurate inventory management.
6.  Because Kafka allows asynchronous processing, if an item is added to a cart and then the user attempts to check out, the item might not be present when the check out operation begins.
7.  Flink is used for stream processing, ensuring every message from Kafka is processed, guaranteeing every order is checked for stock and emails are sent.
8.  The complete order details, which guarantees atomicity and ensures the entire order is eventually processed by the system.
9.  Caching in Redis and using a CDN speeds up read times for popular items, improving site performance and conversion rates.
10. Local indexing means each node in the search index contains a subset of products, indexed locally, and partitioning by category and popularity score allows each node to return relevant results quickly, facilitating pagination.

II. Essay Questions
-------------------

1.  Discuss the trade-offs between using a single-leader replication versus multi-leader replication for the products database. Consider factors like read throughput, write throughput, data consistency, and complexity of implementation.
2.  Explain how the "Observe and Remove Set" CRDT addresses the challenges of concurrent cart modifications. Analyze the roles of tags, deletes, and eventual consistency in maintaining data integrity.
3.  Compare and contrast the two approaches for handling stock decrement with contention: using counter CRDTs and buffering events with Kafka. Discuss the advantages and disadvantages of each approach, considering scenarios with different levels of contention and consistency requirements.
4.  Describe the steps involved in calculating and utilizing popularity scores to optimize read performance. Include the roles of stream processing, batch processing, caching, and CDNs.
5.  Discuss the considerations involved in partitioning a search index for a large e-commerce platform like Amazon or Flipkart. Compare and contrast global term-based partitioning with local indexing.

III. Glossary of Key Terms
--------------------------

-   **B-Tree:** A self-balancing tree data structure commonly used for indexing in databases, optimized for disk-based storage and efficient range queries.
-   **CDN (Content Delivery Network):** A geographically distributed network of proxy servers and data centers that caches static content (e.g., images, videos) to reduce latency and improve performance for users around the world.
-   **CRDT (Conflict-Free Replicated Data Type):** A data structure designed to ensure eventual consistency across multiple replicas without requiring coordination or conflict resolution, allowing concurrent updates to converge to a consistent state.
-   **Data Denormalization:** Adding redundant data to a database table to improve read performance by reducing the need for joins.
-   **Eventual Consistency:** A consistency model where updates to data may not be immediately visible to all users, but will eventually propagate throughout the system.
-   **Flink:** A distributed stream processing framework that enables real-time data processing and analysis.
-   **HDFS (Hadoop Distributed File System):** A distributed file system designed for storing and processing large datasets, often used in conjunction with Hadoop and Spark.
-   **Inverted Index:** An index structure that maps keywords or terms to the documents or data items that contain them, used for fast text search.
-   **Kafka:** A distributed streaming platform that enables building real-time data pipelines and streaming applications.
-   **Lock Contention:** A situation where multiple processes or threads are competing for the same lock, leading to delays and performance degradation.
-   **MongoDB:** A NoSQL document database that stores data in flexible, JSON-like documents.
-   **NoSQL:** A type of database that doesn't adhere to the traditional relational database model, often offering greater flexibility and scalability for handling unstructured or semi-structured data.
-   **Observe and Remove Set:** A type of CRDT designed to handle concurrent adds and removes in a set, ensuring eventual consistency despite conflicting operations.
-   **Pagination:** Dividing a large dataset into smaller, discrete pages, displayed sequentially to a user.
-   **Petabyte:** A unit of data storage equal to 10^15 bytes (1,000 terabytes).
-   **Race Condition:** A situation where the outcome of a program depends on the unpredictable order in which multiple processes or threads access shared resources.
-   **Redis:** An in-memory data structure store, often used as a cache or message broker, known for its high performance and low latency.
-   **Replication:** The process of creating and maintaining multiple copies of data across different nodes or servers to improve availability, fault tolerance, and read performance.
-   **Riak:** A distributed NoSQL database known for its fault tolerance, scalability, and support for CRDTs.
-   **Single-Leader Replication:** A database replication strategy where one node is designated as the primary (leader) and all write operations are directed to it, while other nodes (followers) replicate the data from the leader.
-   **Spark Streaming:** An extension of the Apache Spark framework that enables real-time stream processing.
-   **SS Table:** Sorted String Table, a data structure used in some databases to store sorted key-value pairs on disk, optimized for read performance.
-   **Stream Processing:** Processing data in real-time as it arrives, rather than waiting for it to be stored in a batch.

# Google Maps System Design: Route Optimization and ETA Updates

**The YouTube transcript details a systems design approach to Google Maps, focusing on efficient route calculation.** It highlights the challenge of finding the quickest route between two locations, providing estimated times of arrival (ETAs), and updating them dynamically based on real-time traffic and weather conditions. **The video explores the use of contraction hierarchies to reduce the search space for Dijkstra's algorithm, employing multiple levels of sparse graphs for different distance ranges.** It advocates for a graph database like Neo4j for storing the base graph and considers in-memory storage for smaller, sparse graphs. **Partitioning by geohash is recommended for efficient data management, along with strategies for updating ETAs using Kafka and Flink for stream processing of real-time traffic data.** Finally, the video outlines a system for "bubbling up" traffic updates from individual roads to shortcut edges, ensuring accurate and efficient route calculations.


Briefing Document
-----------------

**Source:** Excerpts from "13: Google Maps | Systems Design Interview Questions With Ex-Google SWE"

**Main Themes:**

-   **Core Functionality & Requirements:** The primary goal of Google Maps is to provide the quickest route between two locations, provide an Estimated Time of Arrival (ETA), and update the ETA in real-time based on changing conditions like traffic and weather.
-   **Data Scale and Graph Representation:** Google Maps deals with a massive amount of data, modeled as a graph. Roads are edges, and intersections, highway exits, and addresses are nodes. The sheer size of the graph (potentially 50 billion nodes in the US alone) necessitates distributed storage and processing. "The point is we have a crap ton of data and for all of these nodes in our graph and for all of these edges we're going to need a decent amount of metadata."
-   **Dijkstra's Algorithm Limitations:** While Dijkstra's algorithm is a common pathfinding algorithm, its performance degrades significantly over long distances with many nodes and edges. "As the number of vertices and edges change in our graph as they increase Dyer's algorithm is going to become a lot slower."
-   **Contraction Hierarchies for Optimization:** Google Maps uses "contraction hierarchies" to reduce the search space. This involves removing unimportant edges and nodes and creating "shortcut edges" (cached edges between two points) to bypass less important routes. "The idea here is we can basically do two things to reduce the search space for our Dyers algorithm the first thing is that we can basically just straight up remove certain edges right...another thing that we can actually do is get rid of certain nodes and edges entirely by creating something known as shortcut edges."
-   **Multi-Level Sparse Graphs:** To handle varying distances, Google Maps uses multiple levels of increasingly sparse graphs. The most sparse graph might only contain major highways for long-distance routing, while less sparse graphs are used for medium distances and the uncontracted full graph is used for short distances. "If I want to run dyras on that it's just not possible right and on the other hand if I wanted to just go and stay within New York that might be a little bit easier and if I wanted to drive 10 minutes from my house that might be even easier."
-   **Database Choices:Base Graph (Uncontracted):** A native graph database like Neo4j is preferred for the full graph due to its ability to handle traversals efficiently. "The reason I say use a graph database is because probably storing this memory is going to be out of question...the reason I say a graph database is because we have to run traversals." The advantage of native graph databases is the use of pointers that allow constant lookups.
-   **Sparse Graphs:** Ideally, sparse graphs are stored in memory for faster access, with Neo4j as a backup. "However these things are technically smaller and if they're small enough Perhaps it is actually feasible to store them in memory."
-   **Partitioning:** Partitioning by geohash is essential for efficient searching. Geohashes group nearby points, allowing Dijkstra searches to primarily operate on the relevant partition. Dynamically sizing geohash partitions or using more replicas for high-traffic areas (like cities) can help balance load.
-   **Real-Time ETA Updates:Speed Calculation:** Driver speed is calculated using GPS signals and displacement over time.
-   **Road Identification:** A hidden Markov model (HMM) is used to determine the most likely road a driver is on, even with imperfect GPS data. This section refers to the Uber video where the speaker explains this approach in detail.
-   **Average Road Speed:** Kafka and Flink are used for stream processing to calculate the average speed on each road over a short time interval (e.g., 5 minutes). Flink nodes maintain a linked list of speeds for efficient average calculation.
-   **"Bubbling Up" Changes:** When the ETA of a road changes, the change must be propagated to any "shortcut edges" that depend on that road. A database of shortcut edge dependencies helps identify which edges need updating. "if we have a shortcut Road and the shortcut road is made up of a variety of individual real roads if one of those real roads all of a sudden gets a ton of traffic on it we may want to update that shortcut Edge because all of a sudden we don't want all of our cars being routed through that traffic we'd to wrap them elsewhere."
-   **Traffic Update Service:** The traffic update service uses Kafka and Flink to process GPS data, calculate road speeds, and update the graph data stored in Neo4j and in-memory sparse graphs. Change data capture (CDC) is used to propagate updates from the shortcut edge dependency database to Flink. "over here we've got our driver where the first thing that they want to do is actually go ahead and calculate their root so like we just showed above there are three different tables that they can go to there's the full map for close distances there's the sparse map for medium distances there's the super sparse map for large distances."

**Important Ideas and Facts:**

-   Google Maps makes a "trade-off" between finding the absolute best route and providing low latency results.
-   There are roughly 16 million intersections and 200 million places in the USA alone.
-   Dijkstra's algorithm involves classifying nodes into three sets: settled, frontier, and unvisited.
-   Contraction hierarchies involve removing unimportant edges and nodes and creating "shortcut edges."
-   The Uber video contains a detailed explanation of how a sharded hidden Markov model can be efficiently built in Flink to help map GPS signals to particular road segments
-   Partitioning by geohash helps keep the Dyjkstra search happening primarily on a single partition of data.
-   Kafka and Flink are extensively used for stream processing to calculate average road speeds and propagate ETA updates.
-   Change data capture is used to pipe data from the MySQL database of shortcut edge dependencies to Flink for fast evaluation of which edges depend on recently changed data.
-   Flink is partitioned by road ID to ensure that all speed updates for a given road are processed by the same Flink node.

**Quotes for Emphasis:**

-   "The point is we have a crap ton of data and for all of these nodes in our graph and for all of these edges we're going to need a decent amount of metadata."
-   "As the number of vertices and edges change in our graph as they increase Dyer's algorithm is going to become a lot slower."
-   "The idea here is we can basically do two things to reduce the search space for our Dyers algorithm the first thing is that we can basically just straight up remove certain edges right...another thing that we can actually do is get rid of certain nodes and edges entirely by creating something known as shortcut edges."
-   "The reason I say use a graph database is because probably storing this memory is going to be out of question...the reason I say a graph database is because we have to run traversals."
-   "if we have a shortcut Road and the shortcut road is made up of a variety of individual real roads if one of those real roads all of a sudden gets a ton of traffic on it we may want to update that shortcut Edge because all of a sudden we don't want all of our cars being routed through that traffic we'd to wrap them elsewhere."
-   "over here we've got our driver where the first thing that they want to do is actually go ahead and calculate their root so like we just showed above there are three different tables that they can go to there's the full map for close distances there's the sparse map for medium distances there's the super sparse map for large distances."

This briefing document provides a comprehensive overview of the key concepts and implementation details discussed in the source material, useful for understanding Google Maps system design and preparing for related interview questions.

A Study Guide
=============

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  What are the three primary requirements for Google Maps as identified in the video?
2.  Explain the difference between nodes and edges in the context of a Google Maps system.
3.  What are the three different types of areas that nodes are split into, according to Dijkstra's algorithm?
4.  Why is Dijkstra's algorithm not always feasible for long-distance route calculations?
5.  Explain the concept of "contraction hierarchies" and how they are used to optimize route calculations.
6.  What is a "shortcut edge," and how does it contribute to reducing search space?
7.  Why does Google Maps use multiple levels of contraction graphs?
8.  Explain why a graph database is preferred for storing the base graph in a Google Maps system.
9.  Why is partitioning by geohash useful for Google Maps data, and what potential problem does it introduce?
10. Explain the Hidden Markov Model and what part of Google Maps functionality it supports.

Quiz Answer Key
---------------

1.  The three requirements are to find the quickest possible route between two locations, provide an estimated time of arrival (ETA) for the route, and incorporate real-time updates (traffic, weather) into the ETA.
2.  Nodes represent locations such as intersections, addresses, or points of interest, while edges represent the roads connecting these locations, and have associated data like distance and travel time.
3.  The three areas are the settled set, which contains nodes with known shortest paths; the frontier set, which contains nodes that have been seen but the shortest path isn't confirmed; and the unvisited set, which contains nodes that haven't been encountered yet.
4.  Dijkstra's algorithm can become too slow for long-distance route calculations because it has to traverse a large number of nodes and edges; therefore the search space is too large, resulting in unacceptable latency.
5.  Contraction hierarchies involve simplifying the map by removing unimportant edges and nodes and creating shortcut edges to reduce the search space for Dijkstra's algorithm. This helps to focus the search on major roads.
6.  A shortcut edge is a cached edge between two points that bypasses intermediary nodes or roads; they represent pre-calculated routes that can significantly speed up the route calculation process.
7.  Multiple levels of contraction graphs, from sparse to super-sparse, allow for efficient route calculation at different scales. Higher-level graphs focus on major routes for long distances, while lower-level graphs provide detailed local routes.
8.  A graph database is preferred for storing the base graph because it is optimized for graph traversals, using pointers to nodes on disk. The native graph database architecture ensures fast retrieval of connected nodes.
9.  Partitioning by geohash groups nearby points together, making range queries efficient for local searches; however, it can lead to uneven load distribution as some geohash partitions (e.g., New York City) may receive significantly more traffic than others (e.g., Oklahoma).
10. The Hidden Markov Model allows Google Maps to determine the most likely road a user is traveling on based on a series of imperfect GPS signals. This is used to accurately attribute speed data to specific road segments, even when GPS data is noisy.

Essay Questions
---------------

1.  Discuss the trade-offs between accuracy and latency in the context of Google Maps route calculations. How does the system balance these competing priorities?
2.  Explain the architecture for updating ETAs in real-time, including the technologies used (Kafka, Flink) and the flow of data from driver to map.
3.  How do shortcut edges and dependency databases work together to ensure real-time updates to route calculations.
4.  Describe how load balancing and data partitioning techniques are employed to handle the high volume of requests and data in a Google Maps system. What are the challenges associated with each technique?
5.  Evaluate the design choices for different database technologies (Neo4j, MySQL) in the Google Maps architecture. What are the strengths and weaknesses of each choice, and how do they contribute to the overall system?

Glossary of Key Terms
---------------------

-   **Node:** A point on the graph representing a location (intersection, address, point of interest).
-   **Edge:** A connection between two nodes, representing a road, with attributes like distance and travel time.
-   **Dijkstra's Algorithm:** A graph search algorithm used to find the shortest path between two nodes in a graph.
-   **Settled Set:** The set of nodes for which the shortest path from the starting point is known.
-   **Frontier Set:** The set of nodes that have been seen but the shortest path has not yet been determined.
-   **Unvisited Set:** The set of nodes that have not yet been encountered in the graph traversal.
-   **Contraction Hierarchies:** A technique for simplifying a map by removing unimportant edges and nodes, and creating shortcut edges.
-   **Shortcut Edge:** A cached edge representing a pre-calculated route between two points, bypassing intermediary nodes.
-   **Geohash:** A hierarchical spatial index that divides the world into a grid of cells, assigning a unique ID to each cell.
-   **Graph Database:** A database optimized for storing and querying graph-structured data, using nodes and edges.
-   **Native Graph Database:** A graph database that uses pointers to nodes on disk for representing edges, enabling constant-time traversals.
-   **Non-Native Graph Database:** Using a relational database to implement a graph database (e.g., using many-to-many relationships to represent edges).
-   **Partitioning:** Dividing a dataset into smaller, more manageable pieces that can be stored and processed independently.
-   **Replication:** Creating multiple copies of data to improve availability and fault tolerance.
-   **ETA:** Estimated Time of Arrival.
-   **Hidden Markov Model (HMM):** A statistical model used to infer the most likely sequence of hidden states (roads) based on a series of observations (GPS signals).
-   **Viterbi Algorithm:** A dynamic programming algorithm used to find the most likely sequence of hidden states in a Hidden Markov Model.
-   **Kafka:** A distributed streaming platform used for building real-time data pipelines and streaming applications.
-   **Flink:** A distributed stream processing framework used for real-time data analysis and transformation.
-   **Change Data Capture (CDC):** A technique for capturing and propagating changes made to a database in real-time.
-   **Bubbling Up:** The process of propagating ETA updates from individual roads to shortcut edges and higher-level contraction graphs.

# Notification Service Design: Fan-Out Pattern and Delivery Guarantees

**The YouTube transcript presents a systems design interview question about building a notification service.** **It focuses on the "fan-out" pattern, where messages are delivered to many subscribed users.** **The design incorporates Kafka, Flink, and considerations for handling popular topics differently.** **The discussion examines ensuring message delivery occurs only once, addressing potential issues of memory footprint and partial failures.** **Solutions like item potency keys, bloom filters, and caching mechanisms are explored to optimize the service.** **Finally, client connection management and polling strategies are discussed, along with a system diagram illustrating the overall architecture.**

Briefing Document: 
------------------
**Overview:**

This document summarizes a systems design discussion on building a notification service, focusing on the "fan-out pattern" and ensuring message delivery with item potency. The core challenge is efficiently delivering notifications to millions of users based on topic subscriptions, while addressing scalability and data consistency. The discussion relies heavily on previous videos about problems like Dropbox and Facebook Messenger, making it less of a deep dive than a formalization.

**Key Themes and Concepts:**

1.  **The Fan-Out Pattern:** The central architectural theme is the "fan-out pattern," where a single message needs to be delivered to multiple subscribed users. The speaker emphasizes: "building a notification service is a pattern that we've now seen three different times on this channel...the fan out pattern AKA when you deliver a message to a variety of users that are all subscribed or interested to some sort of topic."

-   **Kafka:** Used as a stream for handling notifications, sharded by topic ID: "if we have all of these notifications to be delivered we would just be throwing them into a Kafka stream and we can Shard that up by the actual topic ID."
-   **Flink:** A consumer of the Kafka stream, responsible for routing messages. It is pre-populated with topic subscription information to avoid expensive database lookups. "we'll take our topic subscription table we can Shard It Out by user ID...but then when we actually push this change data to Kafka we can basically rehart it by our topic ID and so this goes in sharted by topic ID so that Flink can easily consume to just one kofka Q". Flink is sharded on topic ID.

1.  **Scalability and Capacity Planning:**

-   The document highlights the massive scale involved, with considerations for storage and partitioning. "if we have a billion topics...each topic is going to receive a th000 notifications per day of around 100 bytes on average...we're storing 100 terabytes of data per day 30 pedabytes a year."
-   Partitioning is necessary due to the data volume, enabling caching for faster reads.
-   **Popular vs. Unpopular Topics:** A key optimization is differentiating between popular topics (subscribed to by many users) and unpopular topics. Popular topics will be handled differently to avoid individually pushing notifications to every subscriber. "for certain topics uh we're going to have way too many users subscribing to it right such that it doesn't actually make sense to deliver that message to each user individually the cost is just going to be too expensive."
-   Popular topics are polled by clients instead of pushed in real-time

1.  **Item potency (Ensuring Exactly-Once Delivery):**

-   Preventing duplicate notifications is critical. "we do want to make sure that all of our messages are delivered only once".
-   **Client-Side Storage:** The initial suggestion is to store notification IDs on the client to prevent redelivery, using an "item potency key." "one thing we could do is that we can actually store the notifications that we've seen or at least the IDS of them on the client to make sure that we don't redeliver them so the concept of this is called something like an item potency key".
-   The practicality of this is questioned due to memory constraints on certain devices.
-   **Server-Side Storage:** Storing item potency keys on the notification server is considered, but this raises memory concerns and potential partial failure scenarios. "if we wanted to store this let's say on our actual notification server itself...that means we would have to store all of the item potent keys for all of the users that that notification server is uh cares about".
-   Partial failure considerations: if the client doesn't acknowledge the message, or if the server fails after writing the item potency key but before sending the message.
-   Two phase commit is mentioned as a theoretically perfect solution, but considered overkill

1.  **Bloom Filters for Database Optimization:**

-   To optimize database reads for item potency checks, Bloom filters are proposed. "we could use something like a bloom filter so a bloom filter is going to allow us to ideally get rid of this step one where we have to read the database to check if uh that item potency key has been seen already".
-   Bloom filters are probabilistic data structures that can quickly determine if an element *is not* in a set, avoiding unnecessary database lookups. They can have false positives, so they only avoid reads, they don't guarantee correctness.
-   When the bloom filter returns a negative, it means that the key *definitely* hasn't been seen, and we can send the notification to the client immediately without needing to query the database
-   Using a bloom filter still necessitates writing to the database.

1.  **Client-Side Architecture:**

-   Clients connect to notification servers via a routing server, which uses consistent hashing. "we've got some sort of notification server router which is going to listen to zookeeper get the current consistent hashing policy connect us to a notification server".
-   Heartbeats are used to detect broken connections and trigger reconnection attempts. "sending one another heartbeats via websockets this is going to allow me as the client to realize if I'm still connected to the notification server".
-   Random Jitter on reconnects to avoid the Thundering Herd problem

1.  **Data Storage Choices:**

-   **Topic Subscriptions:** Stored in a MySQL database, sharded by user ID, to facilitate lookups of topics for a given user.
-   **Notifications:** Stored in Cassandra, partitioned by topic ID and sorted by timestamp, for fast ingestion and querying. "this guy is going to be Cassandra because we are going to be publishing a ton of messages there from flank and uh it would be good to be able to injust them pretty quickly".
-   **Popular Notifications Cache:** Redis is used as a cache in front of the Cassandra database for popular notifications, using an LRU eviction policy.

**Diagram Overview**

-   The client connects to a notification server via a router which consults Zookeeper for consistent hashing policies.
-   Clients receive unpopular notifications via websockets.
-   Clients poll for popular notifications via a load balancer and notification polling service.
-   A Redis cache sits in front of a Cassandra database to speed up reads for popular notifications.
-   Flink fans out notifications to proper servers
-   Kafka is the intermediate message queue for delivering notifications.

**Conclusion:**

This design addresses the core challenges of building a scalable notification service by leveraging the fan-out pattern, differentiating handling of popular and unpopular topics, and carefully considering item potency. The use of Kafka, Flink, Cassandra, Redis, and Bloom filters demonstrates a layered approach to optimizing performance and ensuring reliable message delivery.

Study Guide
===========

I. Key Concepts
---------------

-   **Fan-Out Pattern:** A design pattern where a single message is delivered to multiple subscribers or interested parties.
-   **Topics:** Categories or subjects to which users can subscribe to receive notifications.
-   **Flank (Flink):** A stream processing framework used for distributing notifications to users.
-   **Kafka:** A distributed streaming platform used as a message queue for handling notifications.
-   **Topic Subscription Table:** A database table storing information about which users are subscribed to which topics.
-   **Popular Topics:** Topics with a large number of subscribers, requiring different handling to avoid overwhelming the system.
-   **Item Potency:** Ensuring that each notification is processed and delivered only once, even in the face of failures or retries.
-   **Item Potency Key:** A unique identifier for each notification used to prevent duplicate processing and delivery.
-   **Bloom Filter:** A probabilistic data structure used to quickly check if an item potency key has been seen before, reducing the need to query the database.
-   **Notification Server Router:** A component responsible for directing clients to the appropriate notification server based on load and availability.
-   **Consistent Hashing:** A technique used to distribute users across notification servers in a way that minimizes disruption when servers are added or removed.
-   **Thundering Herd:** A situation where a large number of clients simultaneously try to reconnect to a service, potentially overwhelming it.
-   **Notification Polling Service:** A service that allows clients to retrieve popular notifications on a periodic basis.
-   **Popular Notifications Cache:** A cache (e.g., Redis) used to store frequently accessed popular notifications for faster retrieval.
-   **Notifications Table:** A database table (e.g., Cassandra) used to store all notifications, both popular and unpopular.
-   **LSM Tree (Log-Structured Merge Tree):** A data structure used by Cassandra for fast ingestion of data.
-   **Two-Phase Commit:** A distributed transaction protocol used to ensure atomicity across multiple systems.

II. Quiz (Short Answer)
-----------------------

1.  **What is the fan-out pattern, and why is it relevant to building a notification service?** The fan-out pattern involves delivering a single message to multiple subscribers. It's relevant because notification services often need to send updates to many users interested in a specific topic or event.
2.  **Explain the role of Kafka and Flink in the proposed notification service architecture.** Kafka acts as a message queue, receiving notifications and distributing them to Flink. Flink then processes these messages, determines the appropriate recipients based on topic subscriptions, and routes the notifications accordingly.
3.  **What are "popular topics," and how should they be handled differently from other topics?** Popular topics have a vast number of subscribers. They should be handled differently because individually pushing notifications to each user becomes too expensive; instead, users poll for these notifications.
4.  **Define item potency and explain why it is important in a notification service.** Item potency ensures that a notification is processed and delivered only once. It's crucial to prevent users from receiving duplicate notifications, which can be annoying and lead to a poor user experience.
5.  **What is an item potency key, and how can it be used to ensure item potency?** An item potency key is a unique identifier for each notification. By tracking these keys, the service can determine whether a notification has already been processed and delivered, preventing duplicates.
6.  **Explain how a Bloom filter can be used to optimize item potency checks in a notification service.** A Bloom filter is a probabilistic data structure that quickly checks if an item potency key has been seen before. It reduces the need to query the database for every notification, improving performance.
7.  **Describe the function of the notification server router and how it uses consistent hashing.** The notification server router directs clients to the appropriate notification server based on load and availability. Consistent hashing ensures even distribution and minimizes disruption when servers are added or removed.
8.  **What is the Thundering Herd problem, and how can it be mitigated in the context of notification servers?** The Thundering Herd problem occurs when many clients simultaneously try to reconnect to a service after a failure. It can be mitigated by introducing a random jitter before clients attempt to reconnect.
9.  **Explain how a client receives both "unpopular" and "popular" notifications in this system.** Unpopular notifications are pushed to the client in real-time via WebSockets. Popular notifications are retrieved by the client on a polling interval from a notification polling service.
10. **Why is Cassandra a good choice for the Notifications Table, and how is it organized?** Cassandra is well-suited for the Notifications Table due to its fast ingestion capabilities via the LSM tree. It's sharded by topic ID and sorted by timestamp, enabling efficient retrieval of notifications for specific topics and time ranges.

III. Quiz Answer Key
--------------------

1.  The fan-out pattern involves delivering a single message to multiple subscribers. It's relevant because notification services often need to send updates to many users interested in a specific topic or event.
2.  Kafka acts as a message queue, receiving notifications and distributing them to Flink. Flink then processes these messages, determines the appropriate recipients based on topic subscriptions, and routes the notifications accordingly.
3.  Popular topics have a vast number of subscribers. They should be handled differently because individually pushing notifications to each user becomes too expensive; instead, users poll for these notifications.
4.  Item potency ensures that a notification is processed and delivered only once. It's crucial to prevent users from receiving duplicate notifications, which can be annoying and lead to a poor user experience.
5.  An item potency key is a unique identifier for each notification. By tracking these keys, the service can determine whether a notification has already been processed and delivered, preventing duplicates.
6.  A Bloom filter is a probabilistic data structure that quickly checks if an item potency key has been seen before. It reduces the need to query the database for every notification, improving performance.
7.  The notification server router directs clients to the appropriate notification server based on load and availability. Consistent hashing ensures even distribution and minimizes disruption when servers are added or removed.
8.  The Thundering Herd problem occurs when many clients simultaneously try to reconnect to a service after a failure. It can be mitigated by introducing a random jitter before clients attempt to reconnect.
9.  Unpopular notifications are pushed to the client in real-time via WebSockets. Popular notifications are retrieved by the client on a polling interval from a notification polling service.
10. Cassandra is well-suited for the Notifications Table due to its fast ingestion capabilities via the LSM tree. It's sharded by topic ID and sorted by timestamp, enabling efficient retrieval of notifications for specific topics and time ranges.

IV. Essay Questions
-------------------

1.  Discuss the trade-offs between storing item potency keys on the client versus on the server. Consider memory usage, network latency, and potential failure scenarios.
2.  Explain the benefits and drawbacks of using a Bloom filter for item potency checks. In what scenarios would a Bloom filter be most effective, and when might it be less suitable?
3.  Describe the end-to-end flow of a notification from the moment it is generated to the moment it is displayed on a user's device. Be sure to address both popular and unpopular topics.
4.  Evaluate the scalability and fault tolerance of the proposed notification service architecture. Identify potential bottlenecks and suggest ways to improve its resilience.
5.  Compare and contrast the use of Kafka and Cassandra in this system. How do their respective strengths contribute to the overall functionality of the notification service?

# Twitter Search: ElasticSearch Design Deep Dive

**The YouTube video transcript details the design of a distributed search engine, using Twitter search as a model.** It outlines functional requirements like high reliability and low latency, and then estimates capacity needs, discussing storage for tweets and index size. **The video explains API design, focusing on the search endpoint and parameters.** The core concept of an inverted index is introduced for single-node search, and the challenges of distributing this across multiple nodes, particularly regarding local versus global indexes, are addressed. **The discussion covers sharding strategies, the role of an aggregator server, and the importance of using a queue to keep the search index synchronized with the main database.** Finally, the transcript emphasizes the significance of caching for optimizing read speeds and presents a system diagram illustrating the architecture.

Briefing Document: 
------------------
**Source:** Excerpts from YouTube video "Twitter Search/ElasticSearch Design Deep Dive with Google SWE! | Systems Design Interview Question 8"

**Executive Summary:** This video delves into the design considerations for building a distributed search engine, similar to what powers Twitter search or Elasticsearch. It covers functional requirements, capacity estimation, API design, search index architecture (inverted index), sharding strategies, and the importance of using a queue to keep the search index synchronized with the primary data store. The speaker emphasizes that a search index is derived data and should not be used as the primary data store. The speaker also criticizes "Grocking the System Design Interview" for its inadequate coverage of the subject.

**Key Themes and Ideas:**

1.  **Functional Requirements:**

-   Search across plain text documents using search terms. (e.g., searching for "small hands" in tweets during a presidential election)
-   High reliability and low latency due to a large number of users.

1.  *"When you're building a search index the reason you're doing so generally speaking is because you have a bunch of plain text documents and you want to be able to query across all of these documents based on certain search terms"*
2.  **Capacity Estimation:**

-   The speaker bases his estimates on the "Grocking the System Design Interview" book.
-   Assumes 400 million tweets per day.
-   Estimates 300 bytes per tweet (140 characters).
-   Results in 120 GB of tweets per day.
-   Over five years: 730 billion tweets, 200 TB of tweet data.
-   Assuming a 5-byte (40-bit) tweet ID for uniqueness (allowing for more than 730 billion unique IDs).
-   Estimates 500,000 English terms to index.
-   Assuming 15 search terms per tweet results in 55 TB worth of storage in our index.
-   **Conclusion:** Sharding is necessary.

1.  *"We have 730 billion tweets and also 200 terabytes worth of tweets wowie then finally if we're going to basically say that we need five bytes to identify a tweet... that leads us to have about 55 terabytes worth of storage in our index what that means is we're definitely going to need sharding"*
2.  **API Design:**

-   A simple search API with a GET endpoint is proposed.
-   Parameters: Search term(s), sort order, page number, page size.
-   Sorting possibilities: scoring, document ID, terms.

1.  *"Our api design is going to be pretty tiny ... and we basically are just going to have one get endpoint which is going to be we can just call it search where we have some term or i don't know some combination of terms a sort order a page number and a page size"*
2.  **Search Index Architecture: Inverted Index**

-   Instead of a traditional database index, an inverted index is used.
-   The inverted index stores the mapping between a term and the list of IDs of documents containing that term.
-   The key of the hash map is the term, and the value is a list of tweet IDs.
-   Documents are parsed into relevant terms. Stop words and pre-processing of terms are important
-   Terms are sorted for prefix search optimization.

1.  *"Instead of a traditional index where documents are indexed by their primary key or some sort of id is we're using something called an inverted index now the inverted index is basically the central part of why search indexes actually work so well"*
2.  **Lucene and Search Index Features:**

-   Lucian handles prefix searching, suffix searching and fuzzy searching (Levenshtein distance).
-   Ranking documents based on relevance using NLP algorithms (Term Frequency, Inverse Document Frequency).

1.  *"A lot of the kind of logic for building a search index is how you can turn all of these basically string manipulation problems into prefix searches so you can binary search through the index this is um all kind of handled by this one really popular open source search indexing engine called lucian"*
2.  **Distributed Indexing: Local vs. Global**

-   Key question: Do you want a local index or a global index?
-   **Local Index:** Each node handles a range of documents and contains every possible search term only listing the documents on that node. This optimizes for writes.
-   **Global Index:** Each node handles a range of search terms and lists every single document containing them. This optimizes for reads, but may not be feasible.
-   A global index is likely not feasible at Twitter scale because some terms (e.g., "Donald Trump") would have too many associated document IDs, creating hotspots.
-   **Conclusion:** A local index (sharding documents across nodes) is the best approach. Elasticsearch uses this approach.

1.  *"Whenever we're going to be building out some sort of distributed index that's like a secondary index we always have to ask ourselves the following question do you want a local index or a global index... with a local index it optimizes for writes because you can basically just quickly write the document to one node and then know that it's going to go into that local secondary index without having to do any extra network coordination"*
2.  **Sharding Strategies:**

-   **Random Sharding:** Consistent hashing.
-   **Temporal Sharding:** Shard based on time (e.g., all tweets from a given day on the same node). Useful for searching recent posts.
-   **Related Document Sharding:** Group related documents (with common search terms) together. Most ideal, but difficult to implement.
-   A mapping of documents to their shard location is needed.
-   An aggregator server must collect results in parallel from each shard.
-   The aggregator needs to sort data based on score, date, likes, etc.

1.  *"In terms of the way that we may actually end up sharding those documents because like i mentioned we're going to be doing this locally we can either shard them completely randomly where we might do some sort of consistent hashing we can shard them in a way that's based on time temporally"*
2.  **Search Index as Derived Data and Data Synchronization:**

-   The search index is derived data and not the primary data store.
-   The primary data store is the database where tweets are held.
-   Avoid writing directly to both the database and the search index in parallel, which creates an atomic commit problem.
-   Instead, use a queue (e.g., Kafka) to propagate change data from the database to the search index.
-   The database publishes change data to a log-based message queue which is then consumed by the search index.
-   Ensure that the search index operations are idempotent.
-   Twitter's architecture already uses a message broker, so the same broker can be used for search indexing.

1.  *"Whenever you're using something like a search index what we're putting in there is derived data now derived data means that the search index itself is not our main data store and we shouldn't be using a search index as our main data store the reasoning for this is that generally speaking the documents are not going to be indexed by some sort of primary id"*
2.  **Elasticsearch and Caching:**

-   Elasticsearch heavily relies on caching to achieve fast read performance.
-   LRU cache for popular search terms.
-   Elasticsearch caches the entire results of a query, not just individual pages of the index, leading to significant performance gains.

1.  *"What I've just described is basically what elasticsearch does and elasticsearch these days is effectively the leading distributed search index and kind of the one thing that i haven't really mentioned that makes elasticsearch able to run as fast as it is is basically the very very heavy use of a ton of caching"*
2.  **Architecture Diagram**

-   Client -> Load Balancer -> Upload Service -> Log-based Message Broker
-   From Broker:
-   Feed Caches (Redis)
-   Tweet Database
-   Search Index (Partitioned and Replicated)
-   Search Cache
-   Search Index -> Search Service -> Client

**Criticisms:**

-   The speaker expresses dissatisfaction with "Grocking the System Design Interview" for its inadequate coverage of search index design.

This briefing document provides a comprehensive overview of the key design considerations for building a distributed search engine, as presented in the provided transcript. It highlights the importance of choosing the right architecture, sharding strategy, and data synchronization mechanisms to achieve high performance and reliability.

Study Guide
===========

I. Quiz
-------

Answer the following questions in 2-3 sentences each.

1.  What is the primary purpose of a search index in the context of Twitter?
2.  What are the two different types of indices?
3.  Why is high reliability and low latency crucial for a Twitter search service?
4.  What is an inverted index, and why is it essential for search functionality?
5.  What is the advantage of sorting terms in a search index?
6.  Explain how prefix searching works in an inverted index.
7.  What is derived data, and why shouldn't a search index be used as a primary data store?
8.  How does using a queue and change data capture (CDC) from the database help maintain consistency between the primary data store and the search index?
9.  How does Elasticsearch enhance search speed through caching?
10. How does a load balancer, upload service, message broker, feed caches, tweet database, and search index function together?

II. Quiz Answer Key
-------------------

1.  The primary purpose of a search index on Twitter is to allow users to quickly and efficiently search through a large volume of tweets based on specific keywords or terms. This enables users to find relevant information within the vast amount of real-time data generated on the platform.
2.  The two different types of indices are local and global. In a local index, each node handles a range of documents with every possible search term and listings of the documents held on that node. In a global index, every document is listed on one node.
3.  High reliability and low latency are critical because Twitter deals with a massive number of users and real-time data. Users expect quick and consistent search results, and any downtime or delays can significantly impact user experience and engagement.
4.  An inverted index is a data structure that maps terms (keywords) to the documents (tweets) containing them. Instead of indexing documents by their primary key, it indexes terms, making it incredibly efficient for searching documents that contain specific words or phrases.
5.  Sorting terms in a search index enables efficient prefix searches. By sorting alphabetically, one can quickly locate all terms starting with a particular prefix using binary search, greatly improving search performance.
6.  Prefix searching involves binary searching the index for a range of terms starting with a specific prefix (e.g., "ap"). This allows the system to quickly identify all documents containing terms that begin with that prefix, like "apple" or "apricot."
7.  Derived data refers to data that is computed or duplicated from a primary data store. A search index is derived data, and it shouldn't be the primary data store because it typically lacks the durability, consistency, and full data attributes required for reliable data storage and retrieval.
8.  Using a queue (like Kafka) and change data capture (CDC) ensures that updates to the primary database are asynchronously propagated to the search index. This avoids atomic commit issues and keeps the search index in sync with the database, ensuring data consistency without impacting database performance.
9.  Elasticsearch heavily utilizes caching to store frequently accessed search results, popular terms, and even entire query results. This reduces the need to recompute search results for common queries, significantly speeding up search operations.
10. The client is the one to upload a tweet through a load balancer. The upload service puts the tweet into a message broker where the tweet object is replicated and partitioned. It is then sent to the feed caches, tweet database, and search index simultaneously.

III. Essay Questions
--------------------

1.  Discuss the trade-offs between using a local index versus a global index for a distributed search engine like Twitter search. Which approach is more suitable given the scale of Twitter, and why?
2.  Explain the role of a log-based message queue (e.g., Kafka) in maintaining data consistency between Twitter's primary data store and its search index. How does change data capture (CDC) contribute to this process?
3.  Describe the various sharding strategies that can be employed for distributing a search index across multiple nodes. What are the advantages and disadvantages of each strategy?
4.  Analyze the design considerations involved in building a search API for Twitter. What are the essential parameters for a search request, and how can the results be sorted and aggregated efficiently?
5.  Elaborate on the techniques used by Elasticsearch to optimize search performance, such as caching, prefix searching, and the use of inverted indexes. How do these techniques contribute to Elasticsearch's ability to handle large-scale search workloads?

IV. Glossary of Key Terms
-------------------------

-   **Inverted Index:** A data structure that maps terms (words) to the documents (e.g., tweets) in which they appear, enabling efficient full-text search.
-   **Sharding:** The process of partitioning a large dataset (e.g., search index) into smaller, more manageable pieces that can be distributed across multiple nodes.
-   **Local Index:** An index where each node holds a portion of the documents and the terms associated with those documents.
-   **Global Index:** An index where a single node contains all documents relating to a specific term.
-   **Derived Data:** Data that is computed or duplicated from a primary data store; a search index is considered derived data.
-   **Change Data Capture (CDC):** A technique for tracking and capturing changes made to a database and propagating those changes to other systems or data stores.
-   **Log-Based Message Queue:** A distributed system (e.g., Kafka) that provides a durable and reliable mechanism for asynchronously propagating messages (e.g., database updates) between different components.
-   **Prefix Search:** A search that retrieves documents containing terms that start with a specific prefix (e.g., searching for "app" to find "apple" and "application").
-   **Levenshtein Distance:** A metric for measuring the similarity between two strings, used for fuzzy searches to find words that are close to the search term.
-   **LRU Cache:** Least Recently Used cache. An in-memory cache that stores the most popular or frequently used search results.

# Dropbox/Google Drive Systems Design Deep Dive

**This YouTube video transcript details a system design approach for a cloud storage service similar to Dropbox or Google Drive.** The speaker outlines functional requirements like file upload/download, sharing, versioning, and offline editing. **It covers key design decisions, including splitting files into chunks for efficient updates and using a relational SQL database for metadata to ensure data consistency and avoid conflicts.** The discussion also addresses capacity estimates, API endpoints, database schema design, and conflict resolution strategies. **The video differentiates itself from other system design explanations by avoiding the use of a global request queue, partitioning response queues by file ID instead of user ID, and using only one version of each document.** It also details the proposed design, and the reasoning behind it, including a diagram showing how clients interact with the storage and metadata services.

Briefing Document
-----------------

**Source:** Excerpts from "Dropbox/Google Drive Design Deep Dive with Google SWE! | Systems Design Interview Question 3"

**Main Themes:**

This source provides a high-level system design for a Dropbox/Google Drive-like service, focusing on the key components and considerations for building a scalable and robust file storage and synchronization platform. The emphasis is on conflict resolution, chunking strategy, data storage and retrieval, and real-time updates. The source critiques common system design approaches, particularly those found in "Grocking the Systems Design Interview," and proposes alternative solutions.

**Key Ideas and Facts:**

-   **Functional Requirements:** The essential features include:
-   Uploading and downloading files.
-   Sharing files with permissions.
-   Automatic propagation of changes across devices ("via a push style of change").
-   Prevention of file corruption.
-   Version history.
-   Offline editing capabilities.
-   **Capacity Estimates:**
-   File sizes up to 1GB.
-   1:1 read/write ratio.
-   4MB file chunks.
-   500 million total users, 100 million daily active users.
-   200 files per user, leading to 100 billion files and 10 petabytes of storage.
-   1 million active connections per minute for real-time updates.
-   **API Endpoints:**
-   **Upload:** Includes user ID, file name/ID, timestamp, and differences (diffs) between versions ("the differences between the older version of the file that we had locally and the newer version of the file"). This could contain chunks of the file.
-   **Download:** Includes user ID, file name/ID, and potentially a list of chunk hashes or the chunks themselves to download minimum necessary data.
-   **Add User:** Adds a user ID with permissions to a specific file.
-   **Database Design:**
-   SQL database (relational) is preferred for metadata storage due to the need for ACID transactions.
-   Transactions are crucial to prevent data corruption when multiple users modify the same file concurrently.
-   Single-leader replication is recommended to ensure consistency, avoiding the "last write wins" problem of leaderless architectures like Cassandra, which could lead to data loss.
-   Tables:
-   Users (user ID, email, password hash).
-   Files (file ID, name).
-   FileUsers (one-to-many relationship between files and users with permissions).
-   FileVersionChunks (mapping from file ID and version to chunk hashes).
-   Chunks (chunk hash as primary key, S3 URL, file ID for partitioning).
-   Partitioning: Partition by files based on the file_id so that if a user wants "for a given file to find all the chunks of it we can just look at one partition." This enhances data locality and reduces cross-partition joins.
-   **Chunking Strategy:**
-   Files are split into small chunks (4MB) to minimize the amount of data that needs to be re-uploaded when changes are made.
-   Chunks are stored in a block storage system like Amazon S3 ("we can go ahead and fetch that chunk from block storage").
-   Hashes of chunks are used to identify and track changes.
-   **Conflict Resolution:**
-   The client checks with the metadata server to ensure it has the latest version of the file before committing changes.
-   If a conflict is detected (the client is out of date), the client must download the newest version, merge the changes, and try again.
-   While storing conflicting versions as siblings is possible, the source argues it adds unnecessary complexity and advocates for a single version history.
-   **Workflow for Modifying a File:**

1.  Client compares its local version to the new changes, determining which chunks need to be uploaded.
2.  Client uploads the differing chunks to S3.
3.  Client sends a request to the metadata service, which checks if the client is updating the newest version.
4.  If the client has the newest version, the changes are committed to the chunks table (within a SQL transaction).
5.  If the client does not have the newest version, it is instructed to redownload and merge.

-   **Real-time Updates:**
-   Instead of using response queues partitioned by user ID (which the source considers inefficient for popular documents), the proposed solution uses response queues partitioned by file ID.
-   Clients subscribe to the message queues corresponding to the files they have access to (potentially up to 200 queues per user).
-   Log-based message queues (e.g., using long polling, server-side events, or webhooks) ensure message persistence and fault tolerance.
-   **Critiques of "Grocking the Systems Design Interview" and Other Approaches:**
-   The source argues against the need for a global request queue for database updates, claiming the update operation to the metadata table is simple enough to be handled directly.
-   It also criticizes the approach of having response queues partitioned by user ID, which is deemed inefficient for popular documents.

**Quotes:**

-   "Additionally all of the changes to files need to be propagated on other devices and that's probably going to be via a push style of change."
-   "...i'm basically just going to go ahead and steal them from rocky again because I don't know they're pretty much arbitrary and then we're obviously going to put more focus into the design section"
-   "splitting files into chunks means that if you make a small change to a file on a local machine you don't have to re-upload the entire file but basically only the modified chunks"
-   "if a client is to ever make a conflicting write you basically say hey your writes about to conflict go ahead and redownload the newer version of the document merge them in and then you should be good to go but we'll obviously make sure to check it again so we don't have any race conditions or anything like that"
-   "for a given file to find all the chunks of it we can just look at one partition"
-   "the differences between the older version of the file that we had locally and the newer version of the file"

Study Guide
===========

Quiz: Short Answer Questions
----------------------------

1.  Why is it important to split files into smaller chunks in a file storage system like Dropbox or Google Drive?
2.  Explain why a relational SQL database with single-leader replication is preferred for storing file metadata in this design.
3.  Describe the purpose of the "file version chunks" table and its key fields.
4.  Why is the chunks metadata table partitioned by file ID? What benefit does this provide?
5.  Explain the role of a CDN (Content Delivery Network) in the context of storing file chunks.
6.  Why does the client need to upload the chunks to S3 before checking with the Metadata Service?
7.  Why does the author think having response queues partitioned by user ID for changes to a document is not a scalable design choice?
8.  Describe the conflict resolution strategy suggested in this design.
9.  What is the purpose of a load balancer in the given architecture?
10. What data must the client possess to determine the changes required to the file and upload to S3?

Quiz Answer Key
---------------

1.  Splitting files into chunks allows for uploading only the modified portions of a file, reducing the amount of data transferred and stored. This is more efficient than re-uploading an entire file for minor changes.
2.  A relational SQL database with single-leader replication provides ACID (Atomicity, Consistency, Isolation, Durability) properties and guarantees transactionality. This is important for ensuring data integrity and preventing data corruption from concurrent updates.
3.  The "file version chunks" table maps a specific file ID and version to the associated chunk hashes that constitute that version of the file. This allows the system to retrieve all the necessary chunks for a particular version of a file.
4.  Partitioning the chunks metadata table by file ID ensures data locality, meaning that all metadata for chunks belonging to the same file resides on the same shard. This reduces the need for cross-partition joins when retrieving chunk information for a file, improving performance.
5.  A CDN speeds up access times for popular file chunks by caching them in geographically distributed servers. This reduces latency for users downloading these files, improving the overall user experience.
6.  The author feels that the client should upload the changes prior to committing them to the database to avoid any unnessecary latency.
7.  Response queues partitioned by user ID are not scalable because popular documents would require sending updates to a potentially massive number of queues simultaneously, creating a bottleneck.
8.  The conflict resolution strategy involves detecting conflicts by checking if the client has the latest version before committing changes. If a conflict is detected, the client must download the newer version, merge changes locally, and then re-attempt the upload.
9.  A load balancer distributes incoming requests to the Metadata Service across multiple servers, preventing any single server from being overwhelmed. This increases the system's availability and scalability.
10. The client should have a limited internal version of the metadata table where they're keeping track of all the chunks and hashes of the chunks that had previously made up the version of the file that they've since edited

Essay Questions
---------------

1.  Compare and contrast the benefits and drawbacks of the proposed conflict resolution strategy (client-side merge) with an alternative approach where the system stores multiple conflicting versions of a file (e.g., using version vectors).
2.  Analyze the scalability bottlenecks in the presented architecture. How could the system be further optimized to handle a significantly larger number of users and files?
3.  Discuss the trade-offs between using a relational SQL database and a NoSQL database for storing file metadata in a system like Dropbox/Google Drive. Consider factors such as consistency, scalability, and performance.
4.  Evaluate the decision to partition the response queues by file ID instead of user ID. What are the advantages and disadvantages of this approach, and under what circumstances might one be preferred over the other?
5.  The design emphasizes client-side logic for determining file differences and uploading chunks. Discuss the benefits and challenges of this approach, and propose alternative strategies that shift more responsibility to the server-side.

Glossary of Key Terms
---------------------

-   **Chunk:** A small, discrete unit of data into which a larger file is divided for storage and transmission.
-   **Metadata:** Data about data; in this context, information about files such as their name, ID, version history, chunk hashes, and user permissions.
-   **ACID Properties:** A set of properties that guarantee database transactions are processed reliably. Atomicity, Consistency, Isolation, and Durability.
-   **Single-Leader Replication:** A database replication strategy where one node is designated as the primary (leader) and all write operations are directed to it. Changes are then replicated to follower nodes.
-   **Partitioning (Sharding):** Dividing a database into smaller, more manageable pieces that can be distributed across multiple servers.
-   **Data Locality:** The principle of storing related data together to minimize the need for cross-server communication during data retrieval.
-   **CDN (Content Delivery Network):** A geographically distributed network of servers that caches content (e.g., file chunks) to improve delivery speed and reduce latency for users.
-   **Load Balancer:** A device or software that distributes incoming network traffic across multiple servers to prevent overload and ensure high availability.
-   **Conflict Resolution:** The process of handling situations where multiple users simultaneously modify the same file, resulting in conflicting changes.
-   **Response Queue:** A message queue used to notify clients of updates or changes to files.
-   **Log-Based Message Queue:** A durable and persistent message queue where messages are stored in a sequential log, allowing clients to replay messages if they become disconnected.
-   **S3:** Amazon Simple Storage Service, a scalable cloud storage service.
-   **Hash:** A unique, fixed-size identifier computed from a piece of data (e.g., a file chunk). Used for data integrity checks and identifying duplicate chunks.
-   **Webhooks:** Automated HTTP requests triggered by events in a system, used to push notifications to clients.
-   **API Endpoint:** A specific URL that provides access to a web service or application.
-   **Two-Phase Commit:** Is a distributed algorithm technique that guarantees all processes in a distributed transaction either commit or rollback.
-   **Diff:** A summary of the differences between two files.
-   **Merkle Tree:** Tree data structure in which each leaf node is a hash of a block of data, and each non-leaf node is a hash of its children.
-   **Anti-Entropy Process:** Is a data synchronization technique that is used to resolve inconsistencies in the contents of a distributed system.
-   **Data locality:** Describes the tendency of CPUs to access the same set of memory locations repetitively over a short period.

# Designing Real-Time Chat Systems: Messenger & WhatsApp Deep Dive

**Jordan's YouTube video offers a systems design walkthrough for creating a real-time messaging service like Facebook Messenger or WhatsApp.** **The video covers functional requirements such as group chats, real-time messaging, and persistent storage of chat history.** **It estimates capacity based on daily active users and message volume, considering API endpoints for sending and fetching messages.** **The video also explores database table design, advocating for NoSQL databases like Cassandra for message storage due to their write performance, while differing slightly from "Grocking the System Design Interview" regarding HBase.** **The discussion includes the selection of technologies like server-sent events or long polling for message delivery and addresses concerns about message ordering and potential race conditions.** **Finally, it presents a system diagram outlining the message flow from sender to receivers, incorporating load balancing and caching mechanisms.**

Briefing Document: 
------------------
**Overview:**

This document summarizes a system design discussion for a real-time chat messaging service (like Facebook Messenger or WhatsApp). The presenter, a Google SWE, generally follows the approach outlined in "Grocking the System Design Interview" but adds specific insights into database choices, data partitioning, and race conditions. The primary focus is on designing a system that can handle a large volume of messages with low latency and high availability.

**1\. Functional Requirements:**

-   **Core Functionality:**
-   Sending and receiving messages in real-time with low latency.
-   Persistent storage of chat history.
-   Group chats (chats with multiple members). The presenter states: "typically this is considered an extended requirement in the past but I'm going to consider it a functional requirement because it's pretty simple group chats so a chat can have a bunch of members it's kind of one of the main features of both these apps"
-   **Extended Requirements (Mentioned but De-emphasized):**
-   Push notifications.
-   Online/Offline status of users (the presenter considers this more of a front-end concern).

**2\. Capacity Estimates:**

-   The presenter borrows capacity estimates from "Grocking the Systems Design Interview."
-   **Assumptions:**500 million daily active users (DAU).
-   40 messages per user per day.
-   100 bytes per message.
-   **Calculations:**20 billion messages per day.
-   2 terabytes of storage per day.
-   3.6 petabytes of total storage over five years. "assume 500 million daily active users with about 40 messages per user per day so i guess we're all hitting up everyone comes out to basically 20 billion messages per day"

**3\. API Endpoints:**

-   **Key Endpoints:**POST /send_message: Handles sending a message. Parameters include user ID, chat ID, timestamp, and message content (text, potentially an S3 link for images). "sending a message which is probably just going to be a post request with something like a user id a chat id a time stamp for the message and basically the message content"
-   GET /fetch_messages: Retrieves messages for a specific chat. Requires a chat ID and a pagination token for efficient retrieval of messages in chunks. "an endpoint to fetch messages so that should have something like a chat id and also some sort of pagination token"

**4\. Database Design:**

-   **Users Table:** (Relational database)
-   Fields: User ID, email, password hash.
-   Potential for a graph database to optimize grouping of related users. "even though we would probably just end up using a relational database for this out of simplicity and data denormalization i think that it could be really useful to use something like a graph database here"
-   **Chats Table:** (Relational database)
-   Fields: Chat ID, chat name, chat photo.
-   **UserChats Table:** (Relational database)
-   Fields: User ID, Chat ID (with indexes on both for efficient lookups).
-   **Messages Table:** (**NoSQL Database - Cassandra Preferred**)
-   Partition Key: Chat ID.
-   Sort Key: Message Timestamp.
-   Fields: Message timestamp, content of the message
-   Justification for NoSQL (Cassandra/HBase): High write volume (due to message sending). LSM tree architecture in NoSQL databases is well-suited for this.
-   **Cassandra vs. HBase:** The presenter disagrees with the original source's preference for HBase. While both are schemaless and columnar, the presenter argues that column-oriented storage doesn't provide much benefit if the entire message row is typically fetched. Cassandra's leaderless replication is preferred for geographically distributed users, allowing faster writes to local data centers. "i think that cassandra may be a little bit of a better choice here than hbase because cassandra uses a leaderless replication schema which means that especially for people who are geographically distributed pretty far away from one another"
-   **Client-Chat Server Mapping:**
-   A data store is needed to track which clients (devices) are connected to which chat servers.
-   Suggests using a load balancer with consistent hashing instead of a dedicated database like Redis.

**5\. System Architecture and Message Delivery:**

-   **Real-time Message Push:**
-   Need to horizontally scale chat servers to handle a large number of concurrent connections (500 million DAU exceeds the capacity of a single server). "it is impossible for one server to be handling you know 500 million web sockets at once instead all of those websocket connections or whatever type of connection we end up using will have to be distributed horizontally over a bunch of application servers"
-   Load balancer with consistent hashing to route users to specific chat servers.
-   **Choice of Technology (WebSockets, Server-Sent Events, Long Polling):**
-   Short polling is dismissed as inefficient.
-   Long polling, WebSockets, or Server-Sent Events (SSE) are considered.
-   **WebSockets:** Bi-directional, but the bi-directional communication isn't needed and has no automatic reconnection.
-   **Server-Sent Events (SSE):** Unidirectional, automatically re-establishes connections after failure. Preferred. "the one cool thing about server sent events is that they will actually re-establish a connection automatically if they're to fail and web sockets don't do that so perhaps with all these things in mind i would stick maybe between server sent events or long polling as opposed to web sockets"
-   **Long Polling:** Requires resending headers with each request, increasing payload size and latency. Could be selected due to mitigating risk of thundering herd problem
-   Potential "thundering herd" problem if chat servers go down and all clients try to reconnect simultaneously (especially with WebSockets). This is a key consideration when choosing between the technologies.
-   **Message Sending Process:**

1.  Client sends message to the load balancer, which routes it to a chat server.
2.  Chat server performs two concurrent actions:

-   Pushes the message to the database.
-   Pushes the message to other clients in the group chat (using the load balancer to route to their respective chat servers and then SSE).
-   **Race Condition Considerations:**Concern about one action succeeding while the other fails (e.g., message delivered to clients but not stored in the database).
-   Suggests prioritizing database write first, then client push, to ensure persistence. Distributed transactions are deemed unnecessary (due to financial risk). "maybe what you could do is upload the message to the database first and then go ahead and upon receiving the success message send that out to all these different clients but i think there is a consideration basically between whether you want to do the database and the client uploading concurrently or whether you want to basically wait for the message to be in the database before sending them out to the clients"
-   **Message Ordering:**
-   Addresses the misconception that message order will be inconsistent among users.
-   Argues that proper front-end implementation (sorting by timestamp) can ensure consistent order, even with potential timestamp inaccuracies. Timestamps are at least consistent.

**6\. Diagram Description:**

-   Sender sends message to load balancer.
-   Load balancer routes the message to a chat server.
-   Chat server finds all user IDs that need to receive the message by querying the User Chat table in cache.
-   The load balancer forwards all of the messages to the various receivers.
-   The load balancer is configured in active/passive configuration.
-   The chat servers communicate with each other using the consistent hashing pattern delegated by the load balancer and then deliver the messages properly.

**Key Takeaways:**

-   The design prioritizes scalability, low latency, and high availability.
-   Cassandra is the preferred database for storing messages due to its ability to handle high write volumes and its leaderless replication model.
-   Server-Sent Events (SSE) offer a good balance between persistent connections and automatic reconnection.
-   Careful consideration is given to potential race conditions and the order of operations when sending messages.
-   A load balancer plays a crucial role in distributing traffic and routing messages to the appropriate servers.

Study Guide
===========

I. Functional Requirements
--------------------------

-   **Group Chats:** Ability for multiple users to participate in a single chat.
-   **Real-time Messaging:** Sending and receiving messages with low latency.
-   **Persistent Storage:** Storing chat history in a database.

II. Capacity Estimates
----------------------

-   **Daily Active Users:** 500 Million
-   **Messages per User per Day:** 40
-   **Total Messages per Day:** 20 Billion
-   **Message Size:** 100 Bytes
-   **Storage per Day:** 2 Terabytes
-   **Total Storage (5 years):** 3.6 Petabytes

III. API Endpoints
------------------

-   **Send Message (POST):**
-   UserID
-   ChatID
-   Timestamp
-   Message Content (text)
-   **Fetch Messages (GET):**
-   ChatID
-   Pagination Token

IV. Database Tables
-------------------

-   **Users:**
-   UserID (Primary Key)
-   Email
-   Password Hash
-   **Chats:**
-   ChatID (Primary Key)
-   Chat Name
-   Chat Photo
-   **User Chats:**
-   UserID (Indexed)
-   ChatID (Indexed)
-   **Messages:**
-   Partition Key: ChatID
-   Message Timestamp
-   Message Content

V. Design Considerations
------------------------

-   **Real-time Message Delivery:**
-   Load balancing user connections across multiple chat servers.
-   Possible technologies: WebSockets, Server-Sent Events (SSE), Long Polling.
-   **Message Sending Process:**

1.  Client sends message to load balancer, which routes to a chat server.
2.  Chat server persists the message to the database.
3.  Chat server forwards the message to other clients in the chat using the load balancer and appropriate connection type (SSE, Long Polling).

-   **Database Choice:**
-   NoSQL database (Cassandra/HBase) is better for high write volume for messages table.
-   Cassandra's leaderless replication is beneficial for geographically distributed users.
-   **Message Ordering:**
-   Use timestamps to order messages on the client-side.

Quiz
----

1.  What are the three main functional requirements discussed for a messaging service like Messenger or WhatsApp?
2.  Given the capacity estimates, why is data partitioning necessary for the messages table?
3.  What information is included in the POST request for the "Send Message" API endpoint?
4.  Explain the purpose of the User Chats table and why it needs to be indexed.
5.  Why is a NoSQL database like Cassandra preferred for the messages table over a relational database?
6.  What is leaderless replication, and why is it beneficial for geographically distributed users?
7.  Why should you avoid short polling as a technique for pushing messages to users and suggest a better alternative and explain its advantages.
8.  Describe the message sending process, including the roles of the client, load balancer, chat server, and database.
9.  Why might a distributed transaction *not* be necessary when persisting a message and sending it to clients?
10. How can timestamps be used to ensure consistent message ordering among all members of a chat?

Quiz Answer Key
---------------

1.  The three main functional requirements are group chats, real-time messaging with low latency, and persistent storage of chat history. These features are critical for providing a seamless and reliable user experience.
2.  Data partitioning is necessary because of the high volume of messages (20 billion per day) which translates to a large storage requirement (3.6 petabytes over five years). Partitioning distributes the data across multiple servers to manage the load.
3.  The POST request for the "Send Message" API endpoint includes the UserID, ChatID, timestamp of the message, and the message content (text). These parameters are needed to identify the sender, the recipient chat, when the message was sent, and the message itself.
4.  The User Chats table links users to the chats they are participating in. Indexing both the UserID and ChatID allows for efficient lookups of all chats for a user and all users within a chat.
5.  A NoSQL database like Cassandra is preferred because of its ability to handle a high volume of writes due to its LSM tree architecture, which is optimized for write-heavy workloads. This is crucial for handling the constant stream of new messages.
6.  Leaderless replication is a database replication scheme where there is no single "leader" node. Writes can be accepted by multiple nodes simultaneously. This improves availability and reduces latency, especially for geographically distributed users because they can write to a nearby replica.
7.  Short polling should be avoided due to its inefficiency of repeatedly sending requests for updates, regardless of whether new messages are available, which wastes resources. Alternatives like Server-Sent Events (SSE) are more efficient because they maintain persistent, one-way connections, allowing the server to push updates to clients as they become available, reducing unnecessary network traffic.
8.  The client sends a message to the load balancer, which routes it to a chat server. The chat server then persists the message to the database and uses the load balancer to forward the message to other clients in the chat.
9.  A distributed transaction might not be necessary because losing a message is not as critical as losing financial data. The system can be designed to prioritize eventual consistency, where the message is eventually stored and delivered, even if there's a temporary failure in one of the processes.
10. Timestamps can be used by having the front end application display messages in the order of their timestamps. This ensures that all participants in the chat see the messages in a consistent order, regardless of the order they were received by their individual devices.

Essay Questions
---------------

1.  Discuss the trade-offs between using WebSockets, Server-Sent Events (SSE), and Long Polling for real-time message delivery in a high-volume messaging application. Consider factors such as connection overhead, bidirectional communication needs, and handling connection failures.
2.  Evaluate the advantages and disadvantages of using a graph database for the Users table in a messaging application. How could a graph database optimize performance for common operations, and what are the challenges associated with its implementation and scalability?
3.  Describe the architecture of a load balancing system for a messaging application with 500 million daily active users. Discuss the different load balancing algorithms that could be used and how to ensure high availability and fault tolerance.
4.  Explain the concept of eventual consistency and how it applies to the message persistence and delivery process in a messaging application. What strategies can be used to minimize the impact of eventual consistency on the user experience?
5.  Design a caching strategy for the User Chats table to optimize the performance of message forwarding. Consider factors such as cache size, eviction policies, and handling cache invalidation.

Glossary of Key Terms
---------------------

-   **API (Application Programming Interface):** A set of rules and specifications that software programs can follow to communicate with each other.
-   **Cassandra:** A NoSQL database known for its high availability and scalability, particularly suitable for write-heavy applications.
-   **Consistent Hashing:** A technique used in load balancing and distributed caching to evenly distribute data across servers and minimize disruption when servers are added or removed.
-   **Data Partitioning:** Dividing a large dataset into smaller, more manageable pieces that can be stored across multiple servers.
-   **Distributed Transaction:** A transaction that affects multiple data sources across a distributed system, ensuring atomicity, consistency, isolation, and durability (ACID) properties.
-   **HBase:** A NoSQL database built on top of Hadoop, offering column-oriented storage and scalability for large datasets.
-   **LSM Tree (Log-Structured Merge Tree):** A data structure used in NoSQL databases like Cassandra and HBase, optimized for high write throughput.
-   **Load Balancer:** A device or software that distributes network traffic across multiple servers to prevent any single server from being overwhelmed.
-   **Long Polling:** A technique where a client makes a request to the server and keeps the connection open until the server has data to send back.
-   **NoSQL Database:** A type of database that differs from traditional relational databases in that they do not use SQL for querying and may not adhere to the ACID properties.
-   **Pagination Token:** A value used to retrieve the next set of results in a paginated API, allowing clients to efficiently fetch large datasets in smaller chunks.
-   **Real-time Messaging:** The ability to send and receive messages with minimal delay.
-   **Relational Database:** A type of database that organizes data into tables with rows and columns and uses SQL for querying.
-   **Server-Sent Events (SSE):** A server push technology enabling a server to automatically send data updates to a client over a single HTTP connection.
-   **Single Leader Replication:** A database replication strategy where one node is designated as the primary (leader) and all writes are directed to it.
-   **Timestamp:** A sequence of characters or encoded information identifying when a certain event occurred, usually giving date and time of day.
-   **WebSockets:** A communication protocol that provides full-duplex communication channels over a single TCP connection.

# TinyURL and PasteBin Design Deep Dive

**The YouTube transcript presents a comprehensive discussion about designing a URL shortener service like TinyURL or Pastebin, a common system design interview question.** The speaker outlines the functional requirements, capacity estimations, API design, and database schema for such a service. **It explores two primary approaches: hashing and a key generation service, analyzing the trade-offs of different database options, including considerations for replication and partitioning.** The discussion covers essential components like caching, load balancing, and the final system design which includes a microservice architecture with considerations for database selection based on replication strategy and conflict resolution, as well as a CDN for Pastebin content. **The video also emphasizes the need for a coordination service like Zookeeper to manage partitioning information and ensure high availability and consistency.**

Briefing Document: 
------------------

**Source:** Excerpts from "TinyURL/PasteBin Design Deep Dive with Google SWE! | Systems Design Interview Question 1"

**Main Themes:**

This video provides a detailed walkthrough of how to approach the TinyURL/Pastebin system design interview question. It focuses on functional requirements, capacity estimation, API design, database selection, short URL generation, replication, partitioning, caching, load balancing, and overall system architecture. The speaker emphasizes the importance of understanding trade-offs, especially concerning database choices and conflict resolution.

**Key Ideas and Facts:**

1.  **Functional Requirements:**

-   **TinyURL:** Given a long URL, generate a unique shorter URL. Redirect users to the original URL when they access the shortened URL. Link expiration is desirable. Analytics (click count, demographics) are a potential add-on.
-   **Pastebin:** Upload text data and receive a unique shortened URL. Allow access to the text via the shortened URL. Link expiration is also desirable. Analytics (click count) are also a potential add-on.
-   The core difference is TinyURL redirects to a website, Pastebin redirects to a text file.

1.  **Five-Step Process:** The speaker uses a five-step process (detailed in another video) to structure the solution, although the specific steps aren't explicitly listed in this excerpt.
2.  **Capacity Estimation (TinyURL Focus):**

-   500 million URL shortenings generated per month.
-   100:1 read-to-write ratio (meaning reads are far more frequent). This translates to approximately 200 writes per second and 20,000 reads per second. The speaker notes "200 writes per second is no joke."
-   500 bytes per URL (the speaker states this is a generous estimate, and 200-250 bytes might be more realistic).
-   Storage: Over five years, this equates to about 15 terabytes of storage. The speaker notes, "which is actually not very much. You can store that on one machine."
-   Caching: Assume 20% of URLs account for 80% of traffic. This means caching 100 million URLs, requiring 50 GB of cache per month. The speaker notes, "50 gigabytes is more than enough to fit on the memory of one server."
-   Pastebin's data size is more variable, so one needs to discuss with the interviewer on what average paste size will be.

1.  **API Design:**

-   Create: Takes an original URL (or text in Pastebin) and returns a shortened URL.
-   Get: Takes a shortened URL and returns a redirect (TinyURL) or the text (Pastebin).
-   Optional parameters: API key (for rate limiting), user ID, expiration time, alias for the link. "But kind of the bread and butter of this is just a create and a get endpoint"

1.  **Database Tables:**

-   Users (optional, for user management).
-   ShortURL to Content Mapping (essential): short_url (primary key/partitioning key), expiration_time, content (original URL or text), user_id (optional).
-   Analytics/Clicks (optional): row per shortened link.
-   The speaker notes, "there's not a lot of relational needs going on. You're not going to be doing a lot of joining. We have a lot of single access patterns here." This suggests a NoSQL database might be suitable.

1.  **Short URL Generation:**

-   **Hashing:** Use a hashing algorithm (SHA or MD5) but truncate the result. Potential for collisions. Problematic if two users shorten the same URL, as they'd get the same short URL. Using Cassandra for Hashing may lead to "right conflict[s] and cassandra uses last right wins".
-   **Key Generation Service:** Pre-generate all possible short URLs and store them in a database. When a user requests a short URL, grab an unused key and mark it as used. Requires atomic operations (compare-and-set or lightweight transactions) to avoid race conditions (double allocation of keys).
-   The speaker notes, "it is very important that we're using some sort of locking mechanism here on a row at a time such that we don't basically double allocate keys"

1.  **Replication:**

-   The speaker argues against multi-leader replication (e.g., Cassandra, DynamoDB) because "eventually because of the fact that they basically all use last right wins to settle conflicts" which can lead to URL collisions and data loss. Instead, "using a single leader replication schema...is probably the best way to actually make sure that we're avoiding conflicts on a single url."
-   Even though NoSQL databases are fine the speaker says " I think using a single leader replication setup where you have the ability to do some sort of lightweight transaction on a single partition is probably the best way to actually make sure that we're avoiding conflicts on a single url and this can be the case for either our key generation setup or our actual hashing schema"

1.  **Partitioning:**

-   Partitioning by the short URL key itself, which should already be relatively evenly distributed (especially if using hashing).
-   Caching can mitigate the impact of hot keys (popular URLs).

1.  **Caching:**

-   Cache the most frequently accessed short URLs.
-   Use a Least Recently Used (LRU) eviction policy. The speaker notes, "a good eviction policy would probably just be least recently used where if a short url hasn't been clicked for a while and the cash is run out of space we can go ahead and just evict it from the cash and put in our new cash entry so that's kind of a good way of making sure that things aren't getting too crowded there and also keeping the relevant data in the cache"

1.  **Database Choice:**

-   The speaker suggests NoSQL architectures using an LSM tree based storage engine with single leader replication. MongoDB and Riak are mentioned as possibilities, with the caveat that Riak requires understanding its conflict resolution logic. The speaker states, " i'm thinking that any type of nosql architecture using an lsm tree based storage engine but still using single leader replication and you know allows for easy partitioning would be good here maybe something like mongodb which is a document database would be good or perhaps you could get away with ryak"
-   Cassandra is possible but requires careful consideration of potential right conflicts.

1.  **Load Balancing:**

-   Decouple the "create" and "fetch" operations into separate microservices due to the significant read-to-write ratio. This allows scaling them independently.
-   Use load balancers for both services.
-   Consider consistent hashing for the "fetch" service to improve cache hit rates on application servers.
-   Use a coordination service (Zookeeper) to manage partitioning information and server status, especially for database replicas. The speaker notes, "something like a coordination service which allows you to basically have this distributed store of truth that we know is going to be not strongly consistent but ordered and linearizable such as zookeeper would be really useful to kind of keep track of the partitioning information per server"

1.  **Final Design (Diagram Summary):**

-   Clients hit a load balancer (active-passive configuration).
-   Separate microservices for key creation and URL fetching.
-   Key creation service uses key generation logic or hashing and interacts with a single-leader replicated database (MongoDB suggested).
-   URL fetching service checks a key cache first. If a cache miss, it retrieves the URL from the database or a replica (using a second load balancer).
-   Zookeeper manages partitioning information.
-   For Pastebin, the paste data is stored in S3 (for large pastes) or a CDN (Content Delivery Network) for particularly popular pastes. The CDN should use an appropriate language based partitioning strategy.

**Conclusion:**

The video provides a comprehensive overview of the TinyURL/Pastebin system design problem, covering essential architectural decisions and trade-offs. It emphasizes the need for unique short URL generation, efficient read operations, and a robust database strategy to handle potential conflicts and scale effectively. The speaker gives a slight preference towards single-leader replication over multi-leader/leaderless replication as well as a slight preference to using a coordination service such as Zookeeper.

Study Guide
===========

I. Core Concepts & Review Questions
-----------------------------------

1.  **URL Shortening:** Explain the primary purpose of URL shortening services like TinyURL. Why are they useful?
2.  **Pastebin Functionality:** How does Pastebin's core functionality differ from that of a URL shortening service? What type of content is it typically used for?
3.  **Functional Requirements:** What are the key functional requirements that both TinyURL and Pastebin share? How do these requirements influence the design choices?
4.  **Capacity Estimation:** Why is capacity estimation a crucial step in system design? How does it impact the selection of technologies and scaling strategies?
5.  **Read/Write Ratio:** Explain the significance of the read-to-write ratio in the context of TinyURL. How does this ratio impact the prioritization of performance optimizations?
6.  **API Design:** Describe the core API endpoints required for a TinyURL service (create and get). What parameters might be included in these endpoints?
7.  **Database Schema:** Outline the key database tables needed for TinyURL. Which table is most critical, and what are its key attributes?
8.  **Short URL Generation:** Discuss the two primary methods for generating short URLs from original URLs (hashing and key generation). What are the trade-offs of each approach?
9.  **Hashing and Collisions:** What is a hash collision, and why is it a concern in URL shortening? How can collisions be handled, and what are the limitations of these solutions?
10. **Key Generation Service:** How does a key generation service work? What are the advantages and disadvantages compared to hashing?
11. **Replication Strategies:** Compare and contrast single-leader and multi-leader replication. Why does the presenter recommend single-leader replication for TinyURL?
12. **Partitioning:** Explain the concept of partitioning in database design. How can keys be effectively partitioned in the TinyURL system?
13. **Caching:** What is caching, and why is it important for TinyURL? What eviction policy is recommended, and why?
14. **Database Selection:** Discuss the factors to consider when choosing a database for TinyURL. Why does the presenter suggest NoSQL options like MongoDB or Riak?
15. **Load Balancing:** Why is load balancing important? What are the benefits of decoupling the create and fetch operations into separate microservices?
16. **Coordination Service:** What is a coordination service (like Zookeeper), and how can it be used in the TinyURL architecture?
17. **CDN for Pastebin:** How does a Content Delivery Network (CDN) improve the performance and availability of Pastebin?

II. Quiz
--------

**Answer each question in 2-3 sentences.**

1.  What is the primary advantage of using a URL shortener, and give a real-world example of its use?
2.  Explain the difference in content that is returned by Pastebin versus TinyURL.
3.  Why is it essential to have unique URL generation in a URL shortening service?
4.  In the context of TinyURL, what is meant by a "read-to-write ratio," and what does a high ratio imply for system design?
5.  Briefly describe the function of the "create" endpoint in the TinyURL API.
6.  Why might relational databases not be as well suited as NoSQL for this particular kind of project?
7.  What is a key drawback of using a hashing algorithm with truncation for URL shortening?
8.  How does a key generation service help to avoid collisions in URL shortening?
9.  Explain why single-leader replication might be preferred over multi-leader in TinyURL.
10. Describe how a CDN improves the performance of the Pastebin Service?

III. Quiz Answer Key
--------------------

1.  A URL shortener makes long URLs more manageable and easier to share, especially on platforms with character limits. For example, shortening a long product link for sharing on Twitter.
2.  Pastebin returns a text file or block of text, while TinyURL redirects the user to another existing website. TinyURL can be used to obfuscate or rebrand URLs, while Pastebin can be used to store and share code snippets, notes, or other textual information.
3.  Unique URL generation is essential to prevent users from being incorrectly redirected to the wrong destination. Without it, links would be unreliable and the service would become unusable.
4.  A read-to-write ratio describes the proportion of read operations (accessing shortened URLs) to write operations (creating shortened URLs). A high ratio implies that the system should be optimized for fast reads, potentially using caching and other read-heavy optimizations.
5.  The "create" endpoint accepts an original URL as input and returns a shortened URL. This endpoint is responsible for generating a unique short URL and storing the mapping between the short URL and the original URL in the database.
6.  There are not a lot of relations within the data itself. This makes the benefits provided by SQL's abstractions less valuable than the lower latency provided by NoSQL architectures.
7.  The primary drawback is the potential for hash collisions, where different original URLs map to the same shortened URL, even when probing is implemented. This requires a mechanism for collision resolution and can lead to increased latency in some cases.
8.  A key generation service pre-generates and stores a pool of unique keys, ensuring that each original URL is assigned a distinct short URL without collisions. The service uses atomic operations or locking mechanisms to prevent double allocation of keys.
9.  Single-leader replication makes it possible to use lightweight transactions to ensure that an atomic operation on a single partition happens without race conditions. If there were race conditions, a collision could occur.
10. A CDN caches the most popular pastes closer to the users who are requesting them. This makes the retrieval of pastes much faster.

IV. Essay Questions
-------------------

1.  Discuss the trade-offs between hashing and a key generation service for generating short URLs in a high-traffic URL shortening service. Consider factors such as collision handling, performance, and scalability.
2.  Evaluate the different database options presented (SQL vs. NoSQL, single-leader vs. multi-leader replication) for a TinyURL service. Justify your preferred choice, considering factors such as data consistency, availability, and performance.
3.  Describe how caching can be used to improve the performance of a URL shortening service. Discuss different caching strategies, eviction policies, and potential challenges in maintaining cache consistency.
4.  Outline the key considerations in designing a scalable and reliable Pastebin service that can handle large text uploads and high read traffic. Focus on aspects such as storage, content delivery, and fault tolerance.
5.  Explain how a coordination service like Zookeeper can be used to manage partitioning and replication in a distributed TinyURL system. Discuss the benefits of using a coordination service and potential alternatives.

V. Glossary of Key Terms
------------------------

-   **URL Shortening:** The process of converting a long URL into a shorter, more manageable link.
-   **Pastebin:** A web application used for storing and sharing text snippets, often used for code, notes, or configuration files.
-   **Functional Requirements:** The specific features and functionalities that a system is expected to provide.
-   **Capacity Estimation:** The process of estimating the storage, processing, and network resources required to support a system's anticipated workload.
-   **Read/Write Ratio:** The proportion of read operations (data retrieval) to write operations (data creation or modification) in a system.
-   **API Endpoint:** A specific URL that exposes a particular function or service of an application programming interface (API).
-   **Database Schema:** The structure and organization of data within a database, including tables, columns, and relationships.
-   **Hashing:** A process of transforming data of arbitrary size to a fixed-size value (the hash) using a hash function.
-   **Hash Collision:** Occurs when two different inputs to a hash function produce the same hash value.
-   **Key Generation Service:** A service responsible for generating unique keys (short URLs) in a URL shortening system.
-   **Replication:** The process of copying data across multiple servers or databases to improve availability and fault tolerance.
-   **Single-Leader Replication:** A replication strategy where one server is designated as the primary leader, and all writes are directed to the leader.
-   **Multi-Leader Replication:** A replication strategy where multiple servers can act as leaders and accept writes.
-   **Partitioning:** The process of dividing a large dataset into smaller, more manageable chunks that can be stored and processed on different servers.
-   **Caching:** A technique for storing frequently accessed data in a fast-access memory location (cache) to reduce latency and improve performance.
-   **Eviction Policy:** A strategy for determining which data to remove from the cache when it reaches its capacity.
-   **NoSQL Database:** A non-relational database management system that provides flexible data models and horizontal scalability.
-   **Load Balancing:** Distributing network traffic across multiple servers to prevent overload and ensure high availability.
-   **Microservices:** An architectural style where an application is composed of small, independent, and loosely coupled services.
-   **Coordination Service:** A distributed system that provides services such as configuration management, service discovery, and distributed locking.
-   **Content Delivery Network (CDN):** A geographically distributed network of servers that caches content closer to users, improving download speeds and reducing latency.
-   **LRU (Least Recently Used):** A caching eviction policy where the least recently accessed items are removed first.
-   **Consistent Hashing:** A hashing technique that minimizes the impact of adding or removing servers in a distributed system.
-   **Zookeeper:** A popular open-source coordination service used for distributed configuration management, synchronization, and naming.
-   **MongoDB:** A popular open-source NoSQL database that stores data in JSON-like documents.
-   **Riak:** A distributed NoSQL database known for its fault tolerance and scalability.
-   **Active Passive Configuration:** A high-availability setup with a primary (active) system and a secondary (passive) system that takes over if the primary fails.
-   **Heartbeat:** A periodic signal sent by a system to indicate that it is still functioning correctly.
-   **Partition Master:** A node responsible for managing a specific partition in a distributed database.
-   **LSM Tree (Log-Structured Merge Tree):** A data structure used in many NoSQL databases that optimizes for write performance.

# Social Media Design Deep Dive: System Architecture Explained

**The YouTube transcript presents a software engineer's approach to designing a social media platform like Twitter or Instagram.** **It outlines functional requirements, such as posting content, following users, and generating a news feed.** **The discussion covers capacity estimation, API design, and database schema considerations.** **The core architectural challenge is optimizing for a high read-to-write ratio, leading to a design emphasizing pre-computing user news feeds.** **The solution involves a hybrid approach using both caching and a NoSQL database like Cassandra, along with stream processing for efficient distribution of posts.** **The transcript culminates in a complex system diagram illustrating the flow of data and the interaction of microservices.**

Briefing Document: 
------------------

**Source:** Excerpts from "Twitter/Instagram/Facebook Design Deep Dive with Google SWE! | Systems Design Interview Question 2"

**Main Themes:**

The video focuses on designing the system architecture for a social media news feed, highlighting the commonalities between platforms like Twitter, Instagram, and Facebook. The primary challenge is efficiently delivering timely updates to users while handling the vast scale of data and user activity. The discussion covers functional requirements, capacity estimations, API design, database schema, and ultimately, a detailed architectural overview using stream processing to address the performance bottlenecks of a naive SQL approach. The video emphasizes the read-heavy nature of the problem and the need to pre-compute news feeds as much as possible.

**Key Ideas and Facts:**

-   **Functional Requirements:**
-   Users can create posts (text, images, video). "users need to be able to make posts of some type that can contain either text or video or images as well"
-   Users can follow other users.
-   News feed generation with low latency. "i am able to basically go ahead and generate this news feed of people that i follow with relatively low latency"
-   (Extended) Liking and replying to posts.
-   **Capacity Estimations (based on modifications of "grocke in the systems design interview" numbers):**
-   200 million daily active users.
-   Average user follows 200 people.
-   100 million new tweets per day.
-   Post size: ~300 bytes (including metadata). "...we can probably cap that individual post size at around 300 bytes per post"
-   30 GB of storage per day, leading to 55 TB over 5 years. "that means that there are going to be 30 gigabytes per day of storage because that is 100 million new tweets per day times 300 bytes that means over five years we're storing 55 terabytes of data"
-   High read-to-write ratio (reads are 100x more frequent than writes), necessitating read optimization. "reads are probably happening like 100 times as often as tweet rights are going down"
-   **API Design:**
-   /create_post: Handles post creation (user ID, auth token, metadata, content).
-   /get_newsfeed: Retrieves the user's news feed (user ID, auth token). "... just going to list that as a get endpoint where it's just like fetch news feed and for that again we would need user id something like um you know an authorization token of some sort like a bear token and then that's more or less it and that's going to go ahead and basically return all the posts on a user's news feed"
-   (Extended) Endpoints for liking and replying.
-   **Database Schema (Evolution):**
-   Initial Relational Approach (Naive):
-   posts table (user ID, content, metadata).
-   users table (user ID, name, email, etc.).
-   user_follow table (user1 ID, user2 ID to represent following relationships).
-   Shift to NoSQL (Cassandra) for Scalability: "instead what we're ultimately probably better off doing is using a nosql database like cassandra"
-   Eliminates the need for joins. However, we need to keep track of lists of followers and followings for each user. Each 'follow' action is going to "...update both of these two lists and so that's going to require some sort of cross-partition transaction"
-   **Architecture (Key Decisions):**
-   **Moving Computation to the Write Path:** Pre-computing news feeds for users to minimize read latency. "instead why don't we do more work on the right path which is something that is going to be happening less frequently because less tweets are being posted"
-   **Hybrid Approach for High-Profile Users:** Direct delivery of tweets to followers' caches for average users, but a read-path join for users with millions of followers to avoid overwhelming the system. "...for the typical average user with only a few hundred followers we can go ahead and deliver that tweet to everyone's cache however if you have you know if you're a verified user and you have millions or billions of followers what we need is for the work to be done in the read path"
-   **Cassandra Partitioning:** Shard by user ID, using timestamp as a sort key. "i personally think you should be shouting by user id and then using timestamp as a sort key"
-   **Caching:**Feed caches (Redis/Memcached, replicated) for each user. "just throw the entire feed of a given user on one of these redis instances which is then replicated to a couple of other redis instances"
-   Hot post cache (Redis) for popular posts.
-   **Stream Processing:** Using Apache Flink to consume and process the stream of new posts. "using a technology called flank which is super useful for basically keeping this this state local to the actual cluster of nodes which are going to be consuming from that other post stream"
-   **Object Storage:** Storing static media (images, videos) in Amazon S3 (or similar). "we should pretty much always be storing static media in an object store like amazon s3"
-   **Microservices:** Separating read and write services to scale them independently. "...we should probably be operating those as separate micro services and that way we can have separate load balancers for each and scale them independently"
-   **Follow Service**: "a follow service because this is kind of fundamental to how we make sure that when we process posts our followers are actually going to be up to date" and that service would "...update the the followers and the followings whenever we make one of these follows put that in our user follows cassandra table"
-   **Eventual Consistency:** The design acknowledges the possibility of eventual consistency with Cassandra, which is deemed acceptable for social media data. "i really don't think it's too huge of a deal and because it's social media you know we're not dealing with financial data I think that eventual consistency is fine here"

**Diagram Elements:**

The final diagram (described but not visually provided) includes these components:

-   Client
-   Load Balancer
-   Follow Microservice
-   User Follows Cassandra Table
-   Log-Based Message Broker
-   Flink Consumer Nodes
-   In-Memory Message Broker (RabbitMQ, SQS)
-   Feed Caches (Redis/Memcached)
-   Posts Cassandra Table
-   Post Cache
-   S3 (Object Storage)
-   CDN

**Conclusion:**

Building a scalable social media news feed requires a complex architecture optimized for reads. Leveraging stream processing, NoSQL databases, and caching strategies are crucial for handling the massive data volumes and high user activity characteristic of these platforms.

Study Guide
===========

Quiz
----

**Instructions:** Answer the following questions in 2-3 sentences each.

1.  What are the three main functional requirements of a social media service like Twitter, Instagram, or Facebook, according to the source?
2.  What is the primary reason for partitioning data in the context of a social media service?
3.  Why are there more reads than writes in a social media platform, and how does this impact system design?
4.  Describe the two main API endpoints that are essential for a social media service.
5.  Explain the purpose of the user follow table and how it supports the functionality of a social media platform.
6.  Explain why a naive solution using a huge SQL relational database might fail for a large-scale social media service.
7.  Explain the hybrid approach of caching and relational data loading to solve the problem of too many followers for an average user.
8.  What is change data and why is it streamed to a log-based message broker like flink?
9.  In the architecture described, what is the role of an in-memory message broker like RabbitMQ or Amazon SQS?
10. Why should you store static media like images or videos on a system like Amazon S3?

Quiz Answer Key
---------------

1.  Users must be able to make posts containing text, video, or images. Users need to be able to follow other users, so posts from those followed appear in the news feed. Users need to generate news feeds with low latency.
2.  Partitioning data is essential because the amount of data generated by a social media service is too large to be stored on a single server or in a single database table, making it necessary to distribute the data across multiple machines.
3.  In social media, users spend more time reading or viewing content than creating it. As a result, the system design must prioritize optimizing read operations (fetching and displaying news feeds) over write operations (posting new content).
4.  The primary API endpoints are for creating a post, which includes user ID, authorization, and content, and retrieving the news feed, which requires user ID and authorization and returns the posts on a user's news feed.
5.  The user follow table tracks which users are following which other users, with columns for "user1 ID" (follower) and "user2 ID" (followee). By indexing these columns, the system can quickly query who is following a given user or who a given user is following.
6.  Every time a user loads their feed, it requires an expensive join operation to find all the people they follow and retrieve their latest tweets. A huge SQL database would be overloaded by the massive number of read requests, becoming a performance bottleneck.
7.  For users with only a few hundred followers, a tweet can be delivered to everyone's cache. For users with millions or billions of followers, the work is done on the read path, where a feed is requested, and some tweets are loaded from the cache while others are loaded from the database and then combined based on timestamps.
8.  Change Data Capture (CDC) involves streaming changes made to a database, like new follows, to a log-based message broker such as Flink. This ensures that consumer services, like the stream processor for posts, can stay up-to-date with the latest user follow relationships, enabling posts to be directed to the proper user feed caches.
9.  An in-memory message broker like RabbitMQ or Amazon SQS is used for faster processing of posts. Once a post is made, it goes into this broker where it will be delivered to the nodes which will load all the followers of that user and place it in the cache.
10. Static media should be stored in object stores due to its elastic nature and speed. They scale out really easily and especially when combined with a CDN ensures geographic proximity to the user, resulting in performant delivery of static files.

Essay Questions
---------------

1.  Discuss the trade-offs between optimizing for reads versus writes in the design of a social media platform. Provide specific examples from the source to support your arguments.
2.  Compare and contrast using a relational database versus a NoSQL database like Cassandra for a social media platform. What are the advantages and disadvantages of each approach?
3.  Describe the architecture of a social media platform using stream processing, including the roles of different components such as message brokers, stream consumers, and caches.
4.  Explain the challenges of handling users with a very large number of followers in a social media platform. How does the hybrid approach (caching and relational loading) address these challenges?
5.  Discuss the importance of data partitioning and replication in the design of a scalable social media platform. Provide examples of how these techniques are applied in the architecture described in the source.

Glossary of Key Terms
---------------------

-   **API (Application Programming Interface):** A set of rules and specifications that software programs can follow to communicate with each other.
-   **Authorization Token:** A credential used to verify that a user or application has permission to access a resource.
-   **Cache:** A high-speed data storage layer that stores a subset of data, typically transient in nature, so that future requests for that data are served up faster than is possible by accessing the data's primary storage location.
-   **Cassandra:** A highly scalable NoSQL database designed to handle large amounts of data across many commodity servers, providing high availability with no single point of failure.
-   **Change Data Capture (CDC):** Tracking and capturing changes made to data within a database and then delivering those changes in real-time to other systems or applications.
-   **CDN (Content Delivery Network):** A geographically distributed network of proxy servers and their data centers. The goal is to provide high availability and high performance by distributing the service spatially relative to end users.
-   **Flink:** A stream processing framework capable of stateful computations over both bounded and unbounded data streams.
-   **Functional Requirements:** Describe what the system should do from the user's point of view.
-   **Latency:** The delay before a transfer of data begins following an instruction for its transfer.
-   **Load Balancer:** A device or software that distributes network or application traffic across multiple servers to ensure no single server is overwhelmed.
-   **Log-Based Message Broker:** A system that provides asynchronous communication and allows for replaying messages, ensuring data durability and fault tolerance. Examples include Kafka.
-   **LRU (Least Recently Used):** A caching algorithm that discards the least recently used items first.
-   **Metadata:** Data about data; provides information about a certain item's content.
-   **Microservices:** An architectural style that structures an application as a collection of small autonomous services, modeled around a business domain.
-   **NoSQL Database:** A database management system that differs from traditional relational databases in some ways.
-   **Object Store:** A storage architecture that manages data as objects, not as blocks within a filesystem.
-   **Partitioning:** Dividing a database or table into smaller, more manageable parts that can be distributed across multiple servers.
-   **RabbitMQ/Amazon SQS:** In-memory message brokers that enable asynchronous communication between different parts of a system.
-   **Replication:** Creating multiple copies of data and storing them on different servers to ensure data availability and durability.
-   **S3 (Amazon Simple Storage Service):** An object storage service offering scalability, data availability, security, and performance.
-   **Sharding:** A type of database partitioning that separates very large database tables into smaller, faster, more easily managed parts called data shards.
-   **Stream Processing:** The real-time analysis and action on data-in-motion (streaming data) that requires continuous query of new events.
-   **Vertical Scaling:** Increasing the resources (CPU, RAM, storage) of a single server.
-   **Websocket:** A computer communications protocol, providing full-duplex communication channels over a single TCP connection.

# Google Docs Design Deep Dive: Real-Time Collaborative Text Editing

**The YouTube transcript explores the design of Google Docs, focusing on the complexities of real-time collaborative text editing.** The speaker, a Google SWE, explains functional requirements like concurrent editing and eventual consistency, along with capacity estimations and API design. **The video contrasts two algorithms for achieving this: Operational Transform (OT) and Conflict-Free Replicated Data Types (CRDTs).** OT requires all edits to go through a central server, while CRDTs allow decentralized writing, providing more scalability at the cost of increased complexity. **The video concludes by providing potential system diagrams for implementing Google Docs using each of these algorithms.**

Briefing Document
-----------------

This briefing document summarizes the key themes and ideas discussed in the "Google Docs Design Deep Dive" video, focusing on the system design considerations for a collaborative real-time text editor like Google Docs. The video explores functional requirements, API design, database schema, and most importantly, algorithms like Operational Transform (OT) and Conflict-Free Replicated Data Types (CRDTs) to achieve real-time collaboration.

### I. Core Concepts and Functional Requirements:

-   **Functional Requirements:** The core requirement is to allow concurrent editing of a text document by multiple users with eventual convergence to a consistent state for all users.
-   "We want to be able to edit some text document and have others be able to edit it concurrently assuming they have access. Additionally, it's important that if concurrent editing does occur...basically what we want to have is that all people who are editing or accessing the document eventually will see the document in a convergent state."
-   **Capacity Estimates:** The video proposes example capacity estimates, such as 1 billion documents created per day and 1 KB storage per document, translating to roughly 1 TB of storage per day and a need for sharding.
-   **API Design:** The video outlines two essential endpoints:
-   fetch document: To load the document state based on a document ID.
-   make edit: To submit a character insertion at a specific position. The video notes this may not be a traditional REST endpoint, and could be implemented with web sockets.
-   **Database Schema:** The video proposes a schema involving users, document_users, and documents tables. The focus shifts to how to represent the document itself to best handle real-time collaborative editing.

### II. Algorithms for Real-Time Collaboration: Operational Transform (OT) vs. CRDTs

The video contrasts two primary algorithms for achieving real-time collaboration:

-   **Operational Transform (OT):Mechanism:** OT involves routing all edits through a central server. The server transforms incoming operations based on the current state of the document and the client's last known state before broadcasting the transformed operation to other clients. This ensures consistency.
-   **Example:** If two users concurrently edit "helo," one inserting "l" at position 3 to make "hello" and the other inserting "!" at position 4 to make "helo!", the server would transform the "!" insertion to position 5 when sending it to the first user to ensure the final document is "hello!".
-   **Advantages:** Proven to work at scale (Google Docs).
-   **Disadvantages:**Limited right throughput due to the single central server.
-   Inflexible network topology: Not suitable for decentralized or mesh network environments.
-   "All the rights have to go through a single server...it certainly limits the right throughput that we can handle"
-   **Conflict-Free Replicated Data Types (CRDTs):Mechanism:** CRDTs guarantee eventual consistency without requiring a central authority. Operations are designed to be commutative, ensuring that the final document state is the same regardless of the order in which operations are applied.
-   **Types:State-based CRDTs:** Each replica holds the entire state, and the entire state is sent over the network on every change. (Not practical for large documents).
-   **Operation-based CRDTs:** Only the operation (e.g., "insert 'l' at position 3") is sent over the network.
-   **Advantages:**Highly scalable due to decentralized operation.
-   Flexible network topology: Suitable for decentralized applications.
-   "As a client you can basically route all of your rights anywhere and as long as they're eventually sent to all the other clients things are good to go because all of these operations all of these insert operations are commutative which means i can receive them in any order and as long as i eventually get them we're good"
-   **Disadvantages:**Operations need to be idempotent or handled to avoid duplicate application.
-   More complex to implement, especially for text editing.

### III. Representing Documents with CRDTs: The Text Editing CRDT

-   **Index Assignment:** The video describes how to uniformly assign indexes to characters in the document from the range of 0 to 1 (e.g., "h-e-l-o" could have indexes 0.2, 0.4, 0.6, and 0.8).
-   **Insertion:** To insert a character, take the average of the indexes of the surrounding characters (e.g., insert "l" between "l" and "o" at (0.6 + 0.8)/2 = 0.7). Adding a small amount of randomness helps avoid conflicts.
-   **Commutativity:** This indexing scheme allows operations to be commutative, as inserting "l" at 0.7 and inserting "!" at 0.9 will result in the same document state regardless of the order in which these operations are received.
-   **Interleaving Issue:** A major challenge is interleaving issues, especially when two clients are inserting words of slightly different lengths simultaneously leading to jumbled text.

### IV. System Design Diagrams

The video presents two potential system diagrams:

-   **Operational Transform Diagram:** This diagram shows three clients sending all writes through a Kafka queue to an OT primary server. A backup OT server listens to the Kafka queue for failover. The primary server propagates changes back to the clients via WebSockets or Server-Sent Events. HBase is used as a persistent document database, leveraging its column-oriented storage to efficiently retrieve characters by index. Sharding by document ID is recommended.
-   **CRDT Diagram:** This diagram depicts a more flexible, decentralized system. Clients can write to multiple Kafka queues. A primary and backup server (likely different servers than the ones storing the documents) also listen to all queues and persist data to Cassandra, a leaderless database providing higher write throughput. Since Cassandra is eventually consistent, the document version also stores queue indices which have been ingested, allowing new clients to get partially up to date documents and then "catch up" by reading remaining messages from the queues.

### V. Conclusion

The video argues that while Operational Transform is proven and reliable, CRDTs offer greater scalability and flexibility for real-time collaborative text editing due to their decentralized nature and commutative operations. The choice depends on specific requirements, such as the desired level of decentralization, the expected write throughput, and the acceptable level of complexity in implementation.

Study Guide
===========

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  What are the primary functional requirements of a real-time collaborative text editor like Google Docs?
2.  Why is sharding necessary for a Google Docs-like system, and how might document IDs be used for this purpose?
3.  Briefly describe the purpose of the "fetch document" and "make edit" API endpoints.
4.  What are the two algorithms discussed in the source that can be used for real-time text editing?
5.  Explain how Operational Transform ensures that all clients eventually see the document in a convergent state.
6.  What are the limitations of Operational Transform in terms of network topology and scalability?
7.  Explain the difference between state-based CRDTs and operation-based CRDTs.
8.  Why is commutativity of operations essential for CRDTs to function correctly?
9.  How does the text editing CRDT presented in the source resolve the issue of concurrent inserts?
10. What are the advantages and disadvantages of using Cassandra over HBase for a CRDT-based Google Docs implementation?

Quiz Answer Key
---------------

1.  The core functional requirements are the ability for multiple users to edit a single document simultaneously and, despite concurrent edits, ensure all users eventually see a consistent, converged document state. This means resolving conflicts and reflecting changes in near real-time.
2.  Sharding is essential to manage the massive scale of documents and data in a system like Google Docs. Sharding on document IDs allows for distributing the storage and processing load across multiple servers, improving performance and scalability since document storage is self-contained.
3.  The "fetch document" endpoint retrieves the current state of a document given its ID, allowing users to load and view the document. The "make edit" endpoint accepts a character and its position to insert into the document, reflecting user edits.
4.  The two algorithms discussed are Operational Transform (OT), which is used by Google Docs and Conflict-free Replicated Data Types (CRDTs), which have two types including state-based and operation-based.
5.  Operational Transform relies on a central server to receive all edits, transform them based on the current document state, and then propagate these transformed operations to all other clients. This ensures that edits are applied in a consistent order, and conflicts are resolved.
6.  Operational Transform is limited by its reliance on a single central server, which can become a bottleneck for write throughput and restricts the flexibility of network topologies. This centralized approach makes it unsuitable for decentralized applications or mesh networks.
7.  State-based CRDTs involve sending the entire state of the data structure whenever a change occurs, while operation-based CRDTs only transmit the specific operation that caused the change. Operation-based CRDTs are typically preferred for documents due to the high cost of transmitting large amounts of data with every edit.
8.  Commutativity ensures that operations can be applied in any order and still result in the same final state. This allows for decentralized systems where edits can be propagated asynchronously without the need for strict ordering or a central authority.
9.  The text editing CRDT assigns each character a unique index from the range of zero to one, splitting the difference to create a new index point for an inserted character. In cases of concurrent inserts where two clients are inserting different words into a document of differing lengths (also called "interleaving"), this algorithm can become jumbled but there are fixes to be found in existing research.
10. Cassandra offers higher write throughput due to its leaderless architecture, which is beneficial for handling frequent edits, while HBase provides strong consistency by virtue of being a single leader. Although HBase is still an option for this use case, the strong consistency is less important for a CRDT-based system where eventual consistency can be managed through message queue indexes when a new client starts ingesting data, making Cassandra a reasonable choice.

Essay Questions
---------------

1.  Compare and contrast the Operational Transform and CRDT approaches for building a real-time collaborative text editor. Discuss their respective advantages, disadvantages, and trade-offs in terms of scalability, consistency, and network topology.
2.  Evaluate the design choices for the database schema and storage backend in a collaborative text editor. Discuss the suitability of relational vs. non-relational databases, and compare the use of HBase and Cassandra as persistent storage solutions for both Operational Transform and CRDT-based systems.
3.  Describe the challenges of implementing CRDTs for text editing and explain the techniques used to address these challenges, such as dealing with interleaving and ensuring commutativity.
4.  Discuss the role of message queues (e.g., Kafka) and real-time communication protocols (e.g., WebSockets, Server-Sent Events) in the architecture of a collaborative text editor. Explain how these components contribute to the overall performance and responsiveness of the system.
5.  Design a decentralized architecture for a real-time collaborative text editor based on CRDTs. Explain how clients can synchronize their states, resolve conflicts, and ensure eventual consistency without relying on a central server.

Glossary of Key Terms
---------------------

-   **Operational Transform (OT):** An algorithm for collaborative editing that transforms operations to maintain consistency when concurrent edits occur.
-   **Conflict-free Replicated Data Type (CRDT):** A data structure that allows concurrent updates from multiple sources without requiring coordination, ensuring eventual consistency.
-   **State-based CRDT:** A type of CRDT where the entire state of the data is sent over the network to synchronize replicas.
-   **Operation-based CRDT:** A type of CRDT where only the operations that modify the data are sent over the network.
-   **Commutativity:** The property of operations where the order in which they are applied does not affect the final result.
-   **Idempotency:** The property of an operation where applying it multiple times has the same effect as applying it once.
-   **Sharding:** A database partitioning technique that separates very large databases into smaller, faster, more easily managed parts called data shards.
-   **Kafka:** A distributed, fault-tolerant, high-throughput streaming platform used for building real-time data pipelines and streaming applications.
-   **HBase:** A non-relational, column-oriented distributed database built on top of Hadoop, designed for storing and managing large amounts of structured and semi-structured data.
-   **Cassandra:** A highly scalable, distributed, and fault-tolerant NoSQL database designed to handle large amounts of data across many commodity servers.
-   **WebSocket:** A communication protocol that provides full-duplex communication channels over a single TCP connection, enabling real-time data transfer.
-   **Server-Sent Events (SSE):** A server push technology that enables a server to automatically send updates to a client's web browser over HTTP.
-   **Eventual Consistency:** A consistency model in distributed computing that guarantees that if no new updates are made to a given data item, eventually all accesses to that item will return the last updated value.
-   **Interleaving:** The problem of overlapping inserts into a document which can create a jumbled document in certain scenarios.

# Shazam Audio Recognition Design Deep Dive

**This YouTube transcript outlines a systems design interview question centered around building a music recognition service similar to Shazam.** The speaker details the functional requirements, capacity estimations, and API design for such a service. **It explains Shazam's algorithm of converting audio into spectrograms, creating constellation maps, and using combinatorial hashing to identify songs.** The discussion covers how to design the database using an index and a comprehensive audio database. **The transcript explores how to address challenges like sharding the index for faster memory access and parallelizing computations for matching songs.** The explanation concludes with a diagram illustrating the flow of a client request and the involvement of various services, including a fingerprint index, matching service, and a database.

Briefing Document
-----------------

This document summarizes the key concepts and design considerations for building a music recognition service like Shazam, as presented in the YouTube video "Shazam Audio Recognition Design Deep Dive with Google SWE! | Systems Design Interview Question 23".

**I. Functional Requirements:**

-   The core function is to identify a song from a user-recorded audio clip.
-   A user records an audio clip via their phone microphone and sends it to the server.
-   The server returns the identified song.

**II. Capacity Estimates & Data Size:**

-   The design assumes a large database of songs (100 million).
-   Each song has approximately 3,000 indexable "fingerprints" (keys).
-   Each key is 64 bits.
-   This results in an index size of approximately 2.4 terabytes: *"basically I'm going to assume that there are 100 million songs that we're you know having under consideration and then basically per song let's imagine that there are 3 000 indexable keys... and if each of those indexable keys is 64 bits then we can basically say that we're going to have to create an index that is 2.4 terabytes"*
-   This large size necessitates sharding for memory processing capabilities and scalability: *"obviously that's going to be pretty big we can't fit it all on one machine or i guess we could with a hard drive but if we plan on doing any sort of processing in memory we're going to have to be doing sharding and in addition 2.4 terabytes is a lot anyway you would probably want to do some sharding regardless because that obviously has an ability to grow"*

**III. API Design:**

-   A simple API to find a song given a recording: find_song(recording).

**IV. Database Design:**

-   The database design should address searching of two things: an index that allows us to search potential song id's given an audio clip and the actual database holding audio files for further search.
-   Requires two key components:

1.  **Index:** To quickly identify potential song matches based on the audio clip's characteristics. This can be implemented using an inverted index.
2.  **Audio Database:** A database to store audio files or metadata about audio files, used to retrieve the identified song's information: *"the first one is going to be some sort of index that allows us to take in our client-side clip of audio and spits out some potential ideas of audio that you know we could be playing and then the second thing is probably going to be some overarching database of all the audio files or something pertaining to the audio that we can go ahead and search afterwards and return back to our user"*

**V. Music Recognition Algorithm (Shazam's Approach):**

1.  **Spectrogram Conversion:** Audio is represented as a spectrogram (3D graph of Time, Frequency, and Amplitude).
2.  **Constellation Map:** The 3D spectrogram is reduced to a 2D "constellation map" by identifying and plotting points of high amplitude (peaks). This simplifies the search space and accounts for variations in audio quality: *"basically they take all of the peaks at certain amplitudes within the graph in this 3d graph and they reduce this 3d graph down to a 2d graph and create something they call a constellation map where basically now we have a 2d scatter plot of all these points that were at really high amplitude"*

-   **Combinatorial Hashing:** Shazam uses pairs of notes (frequencies) and their time difference to create "fingerprints" or hashes for each song: *"shazam does instead to limit down the space is they actually look at pairs of notes and this has a bunch of really useful properties so basically if you take two frequencies so two of those points on the graph and you take the first one which comes before the second one in time and you say the first one is what we'll call the anchor point basically create some sort of tuple where we have you know the anchor frequency the second frequency and then the last thing is going to be the amount of time and offset between them"*An "anchor point" (first frequency) is paired with a second frequency, and the time delta between them is recorded. This addresses the problem of not knowing where in the song the user's clip starts.
-   The tuple (anchor frequency, second frequency, time delta) forms the basis of the hash. For example, a 32-bit hash could be created using 10 bits for each frequency and 12 bits for the time delta.
-   **Inverted Index Lookup:**The user's audio clip is processed to generate its own set of hashes.
-   These hashes are used to query an inverted index: *"if we have an inverted index where basically the inverted index also is taking all of these hashes or these fingerprints of all of the existing songs that we know and says you know for this hash right here here all the song ids that correspond to it or have that that exact hash then we can look up all of the hashes in our user clip find all of the potential songs that we have to you know look at to compare them and then say okay now we have all these potential songs where one hash is matching what is the best match we can get when we look at all of the hashes in our user clip"*
-   The inverted index maps each hash to a list of song IDs that contain that hash.

1.  **Matching & Ranking:** The retrieved song IDs are potential candidates. The system compares the sequence of hashes in the user's clip to the sequences in the candidate songs. The song with the most matching hashes is identified as the best match.

**VI. Distributed Systems Design Considerations:**

-   **Sharding the Index:** Due to the index size (2.4 TB), it needs to be sharded to fit in memory on multiple servers.
-   **Consistent Hashing:** Using consistent hashing based on the "anchor point" of the hash can improve performance, as keys with the same anchor point will likely reside on the same node: *"consistent hashing will basically make sure that all of those keys with the same anchor point are probably going to be on the same node"*
-   **Parallelization:** The index lookup and the comparison of hashes between the user clip and candidate songs should be parallelized to reduce latency: *"we're probably going to have to parallelize the rest of our requests because they may have a different anchor point in that tuple...one way or another we're probably going to have to be making a bunch of different calls to different shards of that index and in an ideal world because they're parallelized it'll all be decently fast and returned back to our server quickly enough"*
-   **Caching:** Frequently accessed song fingerprints should be cached to reduce database load and improve response time: *"for the popular songs you know at a given time a lot of people are going to be wanting to figure out what one particular song is you can actually just go ahead and cache the sequence of fingerprints for that song and as a result it should greatly speed up a lot of those operations and requests to the database"*

**VII. System Architecture Components:**

-   **Client:** Mobile device with microphone to record audio.
-   **Load Balancer:** Distributes incoming requests to the recognition service.
-   **Recognition Service:** The core service that processes the audio clip, generates hashes, queries the index, and identifies the song.
-   **Fingerprint Index (e.g., Redis):** A fast, in-memory database to store the inverted index (hash -> song IDs).
-   **Matching Service:** Compares the user's audio clip hashes with fingerprints of candidate songs to determine the best match.
-   **Database (e.g., MongoDB):** Stores song metadata, including song IDs and their fingerprints. MongoDB's document-oriented nature and B-tree indexing are suggested for data locality and read performance: *"you'll see that i used a mongodb database for the songs here and there are a couple reasons for that the first thing is that a it still uses a b tree even though it's nosql which is nice because b trees in theory at least should be faster for reads than an lsm tree additionally another nice thing about the mongodb is that because it is a document store you should have really nice data locality"*
-   **Hadoop Cluster/Spark Job:** A batch processing system to update the index and database with new songs and their fingerprints.

**VIII. Batch Job for New Songs:**

-   A recurring batch job (e.g., using Spark) extracts fingerprints from new songs, adds them to the index, and stores them in the audio database.

**IX. Conclusion:**

The design of a Shazam-like system involves a complex interplay of algorithmic techniques (spectrogram analysis, combinatorial hashing), distributed systems principles (sharding, caching, parallelization), and database design choices.

Study Guide
===========

I. Key Concepts & Algorithms
----------------------------

-   **Functional Requirements:** The core function of Shazam is to identify music based on a short audio clip provided by the user. The system accepts an audio input and returns the song title and artist.
-   **Capacity Estimation:** The video uses 100 million songs as a baseline, assuming each song generates approximately 3,000 indexable keys (fingerprints). This leads to an index size of about 2.4 terabytes, highlighting the need for sharding.
-   **Spectrogram:** A visual representation of audio, depicting time, frequency, and amplitude (volume).
-   **Constellation Map:** A simplified 2D representation derived from the spectrogram, plotting points of high amplitude (peaks) across time and frequency. This reduces the search space and accounts for audio variations (compression, noise).
-   **Combinatorial Hashing:** A technique used to create song indexes. Instead of focusing on individual frequencies, Shazam uses pairs of frequencies (anchor point and secondary point) and the time delta between them. This combination forms a unique "fingerprint" for a song segment.
-   **Inverted Index:** A database index that maps each unique hash (fingerprint) to a list of song IDs that contain that hash. This enables rapid lookup of potential song matches.
-   **Fingerprints:** The indexable keys (hashes) created from the constellation map. The video discusses a 32-bit hash structure comprising the anchor point, the second point, and the time delta.
-   **Sharding:** The process of dividing a large database or index into smaller, more manageable pieces (shards) that can be distributed across multiple servers. This improves performance and scalability.
-   **Matching Service:** The component responsible for comparing the user's audio clip fingerprints with the fingerprints of candidate songs.
-   **Fingerprint Cache:** A caching layer that stores frequently accessed song fingerprints in memory, reducing the need to fetch them from the database every time.
-   **Hadoop Cluster:** Used for batch processing new songs, extracting fingerprints, and updating the index and database.
-   **Data Locality:** How close similar data points are to one another within a given data storage solution.

II. System Architecture
-----------------------

1.  **Client:** User records an audio clip on their phone.
2.  **Load Balancer:** Distributes incoming requests to the recognition service.

-   **Recognition Service:**Extracts fingerprints from the user's audio clip.
-   Queries the fingerprint index in parallel to find potential song IDs.

1.  **Fingerprint Index (Redis):** Stores the inverted index, mapping fingerprints to song IDs.

-   **Matching Service:**Retrieves the fingerprints for candidate songs from the fingerprint cache or the database.
-   Compares the user's fingerprints with the candidate songs' fingerprints to find the best match.

1.  **Database (MongoDB):** Stores song information, including fingerprints.
2.  **Hadoop Cluster:** Processes new songs in batches to update the index and database.

III. Quiz
---------

Answer the following questions in 2-3 sentences each.

1.  What is the primary functional requirement of the Shazam system?
2.  Explain the purpose of the constellation map in the Shazam algorithm.
3.  How does Shazam use combinatorial hashing for music recognition?
4.  Why is an inverted index a crucial component of the Shazam architecture?
5.  What is sharding, and why is it necessary for Shazam's fingerprint index?
6.  Describe the role of the matching service in the Shazam system.
7.  What is the purpose of the fingerprint cache, and how does it improve performance?
8.  What is Data Locality and why is it a useful feature in database implementation?
9.  How does the Hadoop cluster contribute to the Shazam system?
10. Explain the two-step process used to identify a song.

IV. Quiz - Answer Key
---------------------

1.  The primary functional requirement of the Shazam system is to identify music based on a short audio clip provided by the user, returning the song title and artist. The system needs to be highly available and have low latency.
2.  The constellation map simplifies the audio search space by reducing a complex 3D spectrogram to a 2D scatter plot of high-amplitude points (peaks). This abstraction accounts for variations in audio quality and external noise.
3.  Shazam uses combinatorial hashing by pairing frequencies from the constellation map (anchor point and secondary point) and calculating the time delta between them. This tuple is then converted into a hash, which serves as a fingerprint for a specific song segment.
4.  An inverted index is crucial because it allows for quick lookup of potential song IDs based on the fingerprints extracted from the user's audio clip. This allows the system to efficiently search for candidate songs that contain the user's extracted audio clip.
5.  Sharding is the division of a large database or index into smaller, distributed pieces. It's necessary for Shazam's fingerprint index because the index is too large to fit on a single server, enabling parallel processing and faster queries.
6.  The matching service compares the fingerprints from the user's audio clip with the fingerprints of candidate songs, retrieved from the cache or the database. It calculates a similarity score to determine the best match and identify the song.
7.  The fingerprint cache stores frequently accessed song fingerprints in memory, reducing the need to fetch them from the database for every request. This reduces latency and improves the overall system performance.
8.  Data locality refers to how close similar data points are to one another within a given data storage solution. This is useful in database implementation because it allows for faster retrieval of data, which is especially useful when working with large files.
9.  The Hadoop cluster is used for batch processing new songs, extracting their fingerprints, and updating both the fingerprint index and the song database. This process occurs on a recurring basis to keep the system up to date.
10. The two-step process involves first using the inverted index to retrieve a list of potential song IDs based on the user's audio fingerprints. Then, the matching service compares the user's fingerprints with those of the candidate songs to determine the best match.

V. Essay Questions
------------------

1.  Discuss the trade-offs between using memory-based storage (e.g., Redis) and disk-based storage for Shazam's fingerprint index. Consider factors like cost, performance, and scalability.
2.  Explain the challenges of handling real-time updates to Shazam's index and database, considering the volume of new music being created daily.
3.  How does Shazam's algorithm account for variations in audio quality and background noise when matching user recordings to songs in its database?
4.  Analyze the scalability challenges of the Shazam system as its user base and song library continue to grow. Propose solutions to address these challenges.
5.  Evaluate alternative database technologies (e.g., NoSQL vs. Relational) for storing song metadata and fingerprints in the Shazam system. Justify your recommendations based on performance, scalability, and data model considerations.

VI. Glossary of Key Terms
-------------------------

-   **Amplitude:** The intensity or loudness of a sound wave, often represented as volume.
-   **Anchor Point:** In Shazam's algorithm, the first frequency in a pair of frequencies used for fingerprinting, serving as a reference point.
-   **Combinatorial Hash:** A hashing technique that combines multiple data points (e.g., frequency pairs and time delta) to create a unique identifier.
-   **Constellation Map:** A simplified 2D representation of a spectrogram, highlighting points of high amplitude across time and frequency.
-   **Data Locality:** How close similar data points are to one another within a given data storage solution.
-   **Fingerprint:** A unique identifier created from a song segment, used for indexing and matching.
-   **Fingerprint Cache:** A caching layer that stores frequently accessed song fingerprints in memory for faster retrieval.
-   **Hadoop Cluster:** A distributed computing framework used for processing large datasets in parallel.
-   **Inverted Index:** A database index that maps each unique hash (fingerprint) to a list of song IDs that contain that hash.
-   **Load Balancer:** Distributes incoming network traffic across multiple servers to ensure high availability and performance.
-   **Matching Service:** The component responsible for comparing the user's audio clip fingerprints with the fingerprints of candidate songs.
-   **Secondary Point:** In Shazam's algorithm, the second frequency in a pair of frequencies used for fingerprinting.
-   **Sharding:** Dividing a large database or index into smaller, more manageable pieces (shards) that can be distributed across multiple servers.
-   **Spectrogram:** A visual representation of audio, depicting time, frequency, and amplitude.

# Leaderboard Design: Top K Problem Solutions

**This YouTube video transcript explains approaches to solving the "top k" or leaderboard problem in system design.** **The speaker outlines functional requirements, capacity estimates, and API specifications for a leaderboard service.** **Several solutions are explored, ranging from exact but slower methods using time-series databases to approximate but faster approaches like Count-min sketch.** **The transcript also details using stream processing frameworks like Spark Streaming.** **These different solutions each provide a means of providing real-time and batched calculation methods for different use cases.** **The speaker concludes by describing a lambda architecture combining both batch and stream processing for optimal performance.**

Briefing Document: 
------------------

**I. Overview**

This document summarizes a systems design discussion on building a leaderboard (Top K) system. The core challenge revolves around efficiently identifying and ranking the most frequent events within a given timeframe, given high throughput and the need for flexible time window queries. The video emphasizes that there's no single "perfect" solution; the best approach depends on specific requirements regarding accuracy, latency, and time window granularity. The source material also notes that there are some things that are similar to other system design problems that have been addressed previously, most notably social media news feeds.

**II. Functional Requirements**

-   **General Goal:** Track and rank the most frequent occurrences of an event. This could apply to various scenarios:
-   Most viewed YouTube videos
-   Most frequent search queries
-   Video game scores
-   **High Throughput:** The system must handle a large volume of events (e.g., a billion updates per day).
-   **Time Window Flexibility:** The leaderboard should be retrievable for arbitrary time intervals (not just all-time). As the source mentions, "the leaderboard is not just over the all-time window but it can be on any time interval you might request."
-   **Top K:** The system focuses on identifying the top *k* items, where *k* is typically less than 1000 to keep computation manageable. The presenter mentions "we kind of want the top k count or just the leader board where let's say k is less than a thousand because if k is more than a thousand we're gonna have some really really intense computation."

**III. Capacity Estimates**

-   The problem is inherently open-ended and hard to quantify precisely.
-   Assume a billion updates per day.
-   *k* (number of top items) should be less than 1000.
-   The high volume makes single-server solutions infeasible.

**IV. API Specification**

-   The API will be platform-specific.
-   A key endpoint is for retrieving the leaderboard: getLeaderboard(startTime, endTime, k). The source notes "one endpoint that we are certainly going to have is just to actually get that leaderboard and that's basically just going to take in a start point and an endpoint and I suppose the parameter k as well"

**V. Core Challenges & Solutions (Lambda Architecture)**

The video emphasizes a "Lambda Architecture" approach, combining stream processing for near real-time results and batch processing for accurate results. The speaker says that the following diagram "is what's actually known colloquially as something called the lambda architecture in the sense that it's got both batch and stream processing present and the point of the stream processing is you know to get some instant results and the point of the batch processing is to get accurate results later down the line"

Here's a breakdown of the proposed solutions:

1.  **Exact Leaderboard (All Time):**

-   Use a message broker (e.g., Kafka) to ingest events.
-   Employ a stream processing framework (e.g., Spark Streaming).
-   Partition data by event ID to ensure related events are processed on the same node.
-   Each node maintains local counts of events.
-   Periodically merge the top *k* lists from each node using a batch job (e.g., Spark job). The source notes that each node can maintain the top k items using a min-heap.
-   Store the final merged leaderboard in a cache or database.
-   *Limitations:* Only provides an all-time leaderboard; lacks time window flexibility.

1.  **Exact Leaderboard (Any Time Window):**

-   Instead of stream processing, ingest events into a time-series database. As the source notes "if we need exact numbers and we need to be able to do it over any window a time series database is probably our best bet."
-   Run batch aggregations on the time-series data to generate leaderboards for specific time windows.
-   *Limitations:* Slower than pre-computed approaches as it relies on read-time computation.

1.  **Approximate Leaderboard (Larger Windows):**

-   Use stream processing (Spark Streaming) to aggregate data into larger time windows (e.g., hourly).
-   After each window, reset local counts and upload aggregated data to a database (e.g., MySQL, which is potentially better for reading than an LSM tree based database).
-   Merge leaderboards of "micro windows" to extrapolate data for larger time windows.
-   *Limitations:* Results are approximate due to potential loss of information during window aggregation. This is because there is no guarantee that all of the correct top k items have been accounted for in the smaller time intervals.

1.  **Approximate Leaderboard (Small Windows - Count-Min Sketch):**

-   Employ Count-Min Sketch data structure for efficient, approximate counting with a limited memory footprint. The source notes "by doing this we can basically keep the computation for determining the counts of all of these elements into a fixed amount of memory without having to kind of expand in a linear matter with respect to the number of elements that we're encountering"
-   Use multiple hash functions to map elements to cells in a 2D array.
-   Increment tallies in the corresponding cells for each event.
-   Estimate counts by taking the minimum tally across all cells associated with an element.
-   Aggregate counts over small time windows and then reset the Count-Min Sketch.
-   Replicate the Count-Min Sketch server using state machine replication (via a log-based message broker) for fault tolerance.
-   *Advantages:* Extremely fast, allows for fine-grained time windows.
-   *Limitations:* Approximate results.

**VI. Key Technologies**

-   **Message Brokers:** Kafka (for durable message storage and distribution)
-   **Stream Processing Frameworks:** Spark Streaming
-   **Databases:**Time-series databases (for accurate, but potentially slower, aggregations)
-   MySQL (for reading aggregated data)
-   **Caching:** (for fast retrieval of pre-computed leaderboards)
-   **Load Balancers:** (for distributing requests to leaderboard servers)
-   **Count-Min Sketch:** (for memory-efficient, approximate counting)

**VII. Database Design Considerations**

-   The specific database design is less critical and depends on the chosen implementation.
-   Consider a database for caching historical leaderboard queries.

**VIII. Conclusion**

Designing a leaderboard system involves trade-offs between accuracy, latency, and time window flexibility. The Lambda Architecture, combining stream processing and batch processing, offers a way to address these competing requirements. Different solutions, such as time-series databases, approximate counting algorithms (Count-Min Sketch), and stream processing with varying window sizes, can be employed depending on the specific needs of the application.

Study Guide
===========

I. Quiz
-------

Answer each question in 2-3 sentences.

1.  What are some examples of events that a leaderboard system might track?
2.  Why is calculating the top K events on the fly often infeasible?
3.  What are the key input parameters for the "get leaderboard" API endpoint?
4.  What is the core idea behind pre-computing results for the read path in system design?
5.  What are the benefits of using a min-heap for tracking the top K items on a node?
6.  What are the two key issues with the initial batch processing implementation discussed?
7.  How does using a time-series database help with selecting any time window for the leaderboard?
8.  How does the count-min sketch algorithm minimize the memory footprint for counting elements?
9.  What is state machine replication and how can it be used in conjunction with the Count-Min Sketch to ensure fault tolerance?
10. Briefly describe the lambda architecture in the context of leaderboard design.

II. Quiz Answer Key
-------------------

1.  Examples include views across YouTube videos, searches with a given keyword, or scores in a video game. The core principle is that the leaderboard tracks the most frequently occurring instance of a given event on a platform. High throughput and the need for aggregation over various time intervals are the reason that compiling it on one server is not feasible.
2.  The volume of data (updates to items) is often too large to be processed in real-time on a single server. This necessitates distributed processing and pre-computation to reduce latency.
3.  The "get leaderboard" API endpoint typically takes a start time, an end time, and the value of "k" (the number of top items to retrieve) as input parameters.
4.  Pre-computing results involves performing calculations ahead of time (on the write path) to minimize the work required when a user requests the leaderboard (on the read path). This improves response time and reduces latency.
5.  A min-heap of size k allows for efficient tracking of the lowest element in the top K items, enabling quick replacement of elements that fall out of the top K. This has better time complexity compared to sorting.
6.  The initial batch processing implementation had two main issues: it only provided a leaderboard for all time and it only had a time granularity equal to batch processing speed. It lacks the flexibility to select any arbitrary time window and it is limited by the frequency of the batch process.
7.  Time-series databases use column-oriented storage and indexing to optimize for fast reads, allowing for aggregations to be performed on-the-fly for any specified time window. However, this can be slower compared to pre-computed solutions.
8.  The count-min sketch uses a 2D array and multiple hash functions to approximate the counts of elements with a fixed amount of memory, avoiding linear scaling with the number of elements. By taking the minimum count across the hash functions, the algorithm is able to approximate the count of elements.
9.  State machine replication involves creating a secondary node that performs the same identical calculations of a primary node using the same inputs to ensure fault tolerance. Using a log-based message broker like Kafka guarantees the second node receives the same input as the first node which allows for the second node to take over if the first one fails.
10. The lambda architecture combines both batch and stream processing, using stream processing to generate instant results and batch processing to generate accurate results. It uses stream processing to get instant results and batch processing for accurate results later down the line.

III. Essay Questions
--------------------

1.  Discuss the trade-offs between accuracy and performance in designing a leaderboard system. How do different approaches (e.g., time series databases, count-min sketch) address these trade-offs?
2.  Explain the benefits and drawbacks of using a stream processing framework (e.g., Spark Streaming) for calculating leaderboards. How can sharding and merging of lists be optimized in this context?
3.  Compare and contrast the different solutions presented for handling time window flexibility in leaderboard systems. Which solution would be most appropriate for different use cases, and why?
4.  Describe the count-min sketch algorithm in detail, including its advantages, limitations, and how it can be used to approximate the counts of elements in a high-throughput system.
5.  Outline the steps involved in designing a complete leaderboard system using the lambda architecture. Include considerations for data ingestion, processing, storage, and serving the leaderboard to clients.

IV. Glossary of Key Terms
-------------------------

-   **Leaderboard:** A ranked list of the top performers or most frequent occurrences based on a specific metric.
-   **Top K Problem:** A system design challenge focused on efficiently identifying and retrieving the top K items from a large dataset.
-   **Throughput:** The rate at which data or events are processed by a system, typically measured in events per second or updates per day.
-   **API Spec:** A detailed description of the interfaces and functionality exposed by a service, including endpoints, parameters, and data formats.
-   **Log-based Message Broker:** A system for reliably storing and delivering messages in a sequential order, typically used for asynchronous communication between services (e.g., Kafka).
-   **Stream Processing Framework:** A platform for processing continuous streams of data in real-time (e.g., Spark Streaming).
-   **Min-Heap:** A tree-based data structure where the value of each node is less than or equal to the value of its children, providing efficient access to the minimum element.
-   **Time Series Database:** A database optimized for storing and querying time-stamped data, often used for analyzing trends and patterns over time.
-   **Count-Min Sketch:** A probabilistic data structure used for approximating the frequencies of elements in a stream, with a fixed memory footprint.
-   **State Machine Replication:** A technique for ensuring fault tolerance by replicating the state and operations of a service across multiple nodes.
-   **Lambda Architecture:** A data processing architecture that combines both batch and stream processing to provide both real-time and accurate results.
-   **Kafka Broker:** A central node in an Apache Kafka cluster that stores and manages data streams, providing scalability, fault tolerance, and durability.
-   **Horizontal Sharding:** Also known as database sharding, this divides a database or a table into smaller, more manageable pieces, and the physical location of these shards are on different servers.
-   **Batch Job:** A program that processes a large amount of data in a single operation, typically performed offline or at scheduled intervals.
-   **Fault Tolerance:** The ability of a system to continue operating correctly even in the presence of failures or errors.
-   **Latency:** The time it takes for a request to be processed and a response to be returned by a system.
-   **Node Coordination:** The process of communication and synchronization between different nodes in a distributed system.
-   **Hopping and Tumbling Windows:** Hopping and tumbling windows are both types of windowing techniques used in stream processing to divide a continuous stream of data into smaller, more manageable segments for analysis.
-   **LSM Tree:** Log-Structured Merge Tree, a data structure used in databases that optimize for write performance.
-   **B-Tree:** A self-balancing tree data structure that keeps data sorted and allows searches, sequential access, insertions, and deletions in logarithmic time. B-trees are commonly used in database and file systems.

# Rate Limiter Design: Architecture, Algorithms, and Implementation

**The YouTube transcript outlines the design of a rate limiter system.** **The video explores key considerations like preventing abuse, minimizing latency, and supporting various rate-limiting techniques.** **It addresses capacity estimation, identifies user ID and IP address as targets for rate limiting, and defines a rate-limiting interface.** **The discussion covers architectural choices such as local vs. distributed rate limiting, caching strategies, and database selection with a preference for Redis.** **Finally, the video presents common algorithms like fixed window and sliding window rate limiting, addressing concurrency concerns and concluding with a simple rate limiter design.**

Briefing Document: 
------------------

**I. Executive Summary**

This document summarizes key considerations for designing a rate limiter, a crucial component in modern system architecture for preventing abuse, ensuring service availability, and maintaining a positive user experience. The design involves trade-offs between latency, accuracy, scalability, and complexity. The discussion covers various aspects, including problem requirements, capacity estimates, rate limiting strategies, architectural placement, algorithm choices, and concurrency considerations.

**II. Problem Definition and Requirements**

-   **Purpose:** The primary goal of a rate limiter is to prevent malicious or negligent users from overwhelming a service with excessive requests, which can lead to service degradation or outages.
-   *"We're going to be building a rate limiter that's going to stop bad users malicious ones or potentially just negligent ones from submitting too many requests...that can easily bring down your service."*
-   **Key Requirements:Effectiveness:** Accurately identify and limit abusive traffic.
-   **Low Latency:** Introduce minimal delay to legitimate requests. *"This rate limiter should introduce minimal latency meaning we want to make it as fast as humanly possible."*
-   **Flexibility:** Support multiple rate limiting techniques. *"There are a variety of different rate limiting techniques. Let us try and support at least a couple of those."*

**III. Capacity Estimates**

-   **Scale:** A system with a billion users and 20 rate-limited services (endpoints) is considered.
-   **Data Storage:** Estimations suggest needing to store at least 240 GB of data to track request counts per user per service (based on 8 bytes for user ID + 4 bytes for request count).
-   *"At minimum we're probably looking to store at least 240 GB of data."*
-   **Implication:** This data volume necessitates partitioning for in-memory solutions. *"If we are planning on storing things in memory we're likely going to need to partition out our server."*

**IV. Rate Limiting Strategies**

-   **Identification:** Rate limiting can be applied based on:
-   **User ID:** Effective for authenticated endpoints, allowing blocking of malicious accounts. However, it's ineffective against users creating multiple accounts.
-   *"If I'm a user at home and I create many accounts well the rate limiting on one user ID is now not going to work as well."*
-   **IP Address:** Useful for non-authenticated endpoints and blocking abusive behavior from a network. Potential for false positives (throttling innocent users on a shared network).
-   *"The nice thing about this is that you know if someone is creating a bunch of accounts over the same network uh it's easy to track."*
-   **Hybrid Approach:** Combining IP address rate limiting for non-authenticated endpoints with user ID-based rate limiting for authenticated endpoints.
-   *"Ultimately I think something like a hybrid approach makes the most sense where you use the IP address of a user on non-authenticated endpoints and then the second year logged in you can go and use that user ID endpoint."*
-   **Interface:** A function rateLimit(userID, IPAddress, serviceName, requestTime) that returns a boolean (true if throttled, false otherwise).
-   *"We've got uh this function rate limit which is going to return a Boolean which will be true if the request should be throttled or rate limited and false otherwise."*

**V. Architectural Placement**

-   **Options:Local Rate Limiting (Service-Specific):** Implement rate limiting logic directly within each service.
-   **Pros:** No extra network calls, lower latency.
-   **Cons:** Doesn't shield application servers from spam, tightly couples application and rate limiting scaling, loss of rate limiting data if a service instance goes down.
-   *"If we're getting spammed over and over again...those requests are still making it over here to the service right and so that's going to use up a bunch of network bandwidth."*
-   **Dedicated Distributed Rate Limiter:** A separate, independent service for rate limiting.
-   **Pros:** Shields application servers from traffic bursts, allows independent scaling of rate limiting infrastructure.
-   *"The biggest Pro is that we actually Shield our application servers from large bursts of network traffic."*
-   **Cons:** Introduces an extra network call, increasing latency.
-   **Caching at Load Balancer:** The load balancer can act as a write-back cache, storing request counts for popular user IDs, reducing the load on the rate limiter service.
-   *"We could have the load balancer effectively be a right back cach...the load balancer basically Acts as a partial rate limiter itself."*

**VI. Database Choice & Replication**

-   **In-Memory Database:** Essential for low latency. Redis or Memcached are suitable options. Redis is favored due to built-in data structures (lists, hashmaps) and replication features.
-   *"We could just keep all of our data in memory so that would naturally lead us to either a just using our own like custom application server solution or B employing something like redus or mcast which are both just in memory databases."*
-   **Replication:** Single-leader replication is preferred for accuracy. Multi-leader or leaderless replication can lead to inconsistencies in request counts.
-   *"I think it would be better off to just be accurate in our rate liming Counts from the get-go and the way that we can do that the most easy is by using a single leader."*

**VII. Rate Limiting Algorithms**

-   **Fixed Window Rate Limiting:**Divides time into fixed-size windows (e.g., minutes).
-   Allows a certain number of requests per window.
-   Resets the counter at the beginning of each new window.
-   Simple to implement but can allow bursts of traffic near window boundaries.
-   **Sliding Window Rate Limiting:**More accurate than fixed window.
-   Maintains a queue (linked list) of request timestamps within the window.
-   Removes old requests from the queue as they fall outside the window.
-   Adds new requests to the queue if the number of requests within the window is below the limit.
-   More complex to implement but provides better traffic smoothing.
-   *"The idea here is that every single time a new request comes in we first Purge all of the old requests from the LinkedIn from the LinkedIn from the link list and then after that we go ahead and add the new request to the link list."*

**VIII. Concurrency Considerations**

-   **Race Conditions:** Incrementing request counters and modifying linked lists require careful handling of concurrency.
-   **Locking:** Locking mechanisms are necessary to prevent lost updates and concurrent modification exceptions. Alternatively, single-threaded execution or concurrent data structures can be used.
-   *"Unless we're doing this type of thing atomically...we have the potential for lost updates."*

**IX. Rate Limiter Design Summary**

-   **Architecture:** Load balancer with rate limiter cache in front of a distributed Redis cluster (single-leader replication per partition).

1.  **Workflow:**Request hits the load balancer.
2.  Load balancer checks its rate limiter cache.
3.  If request not rate limited in cache, it's forwarded to the rate limiter service (Redis).
4.  Rate limiter service determines if request should be allowed.
5.  If allowed, the request is forwarded to the backend service.

**X. Conclusion**

Designing an effective rate limiter requires careful consideration of various factors, from capacity planning and algorithm selection to architectural placement and concurrency control. The optimal design depends on the specific requirements and constraints of the system being protected. The presented design offers a robust starting point for building a scalable and reliable rate limiting solution.

Study Guide
===========

I. Review of Key Concepts
-------------------------

-   **Rate Limiting:** Controlling the rate of requests a user or service can make to prevent abuse, ensure fair usage, and maintain system stability.
-   **Latency:** The delay between a request and a response. Minimizing latency is crucial for a good user experience.
-   **Malicious/Negligent Users:** Users who intentionally or unintentionally overload a system with excessive requests.
-   **Authentication:** Verifying the identity of a user or service.
-   **Endpoints:** Specific URLs or entry points to a service (e.g., login, create post, read comment).
-   **User ID vs. IP Address:** Different methods for identifying and rate-limiting users. User ID is suitable for authenticated endpoints, while IP address can be used for non-authenticated ones.
-   **Rate Limiting Interface:** A standard way to interact with different rate-limiting algorithms. The interface typically includes a function that returns a Boolean indicating whether a request should be throttled.
-   **Local vs. Distributed Rate Limiting:** Local rate limiting occurs within each service, while distributed rate limiting uses a dedicated layer.
-   **Load Balancer:** Distributes incoming network traffic across multiple servers.
-   **Caching:** Storing frequently accessed data in a faster storage medium to reduce latency.
-   **Database Choice (Redis):** Selecting an appropriate database for storing rate-limiting data. Redis, an in-memory data store, is a suitable choice due to its speed and data structure support.
-   **Replication:** Creating copies of data across multiple servers to ensure fault tolerance and availability.
-   **Multi-Leader vs. Single-Leader Replication:** Different approaches to data replication. Single-leader replication is preferred for rate limiting due to its consistency guarantees.
-   **Fixed Window Rate Limiting:** A rate-limiting algorithm that resets the request count at fixed time intervals.
-   **Sliding Window Rate Limiting:** A more accurate rate-limiting algorithm that considers a moving time window to prevent bursts of requests near the window boundaries.
-   **Concurrency/Threading Considerations:** Addressing potential race conditions and data inconsistencies when multiple threads access and modify rate-limiting data.
-   **Lost Updates:** A race condition that results in data being lost or overwritten because multiple threads were accessing the same data simultaneously.
-   **Atomic Operations:** Operations that are guaranteed to execute without interruption, preventing race conditions.
-   **Race Condition:** A situation where the outcome of a program depends on the unpredictable order in which multiple threads or processes access shared resources.

II. Quiz (Short Answer)
-----------------------

1.  **What is the primary purpose of a rate limiter in a system?** A rate limiter's main goal is to prevent users (malicious or otherwise) from overwhelming a system with too many requests, thus maintaining stability and performance. It protects against abuse and ensures fair resource allocation.
2.  **Why is it important to minimize latency when implementing a rate limiter?** Rate limiters are a necessary function for a smooth, efficient experience. Adding latency would frustrate users, potentially driving them away and hurting overall user experience.
3.  **Explain the difference between rate limiting based on User ID and IP address. What are the pros and cons of each?** User ID-based rate limiting is good for signed-in users, preventing individual accounts from overloading services, but falls apart when users create multiple accounts. IP address-based rate limiting works for anonymous users, but may incorrectly throttle legitimate users sharing the same IP.
4.  **Describe the basic interface of a rate limiter. What inputs and outputs would it typically have?** A rate limiter interface typically includes a function, 'rate_limit', that takes User ID, IP address, the service name and the request timestamp as input. It returns a Boolean value indicating whether the request should be throttled.
5.  **What are the advantages and disadvantages of local rate limiting compared to distributed rate limiting?** Local rate limiting reduces network calls because it is done in each service, but it does not shield application servers from bursts of traffic and tightly couples application and rate-limiting scaling. Distributed rate limiting protects application servers and allows independent scaling, but introduces extra network calls.
6.  **How can caching be used to improve the performance of a distributed rate limiter? Where in the architecture would this caching typically be implemented?** Caching reduces network calls to the rate limiter. Implementing a write-back cache on the load balancer can store the request counts of frequent users, filtering out many bad actors requests before they ever reach the rate limiter.
7.  **Why is Redis a suitable choice for storing rate-limiting data?** Redis is suitable because it's an in-memory database which is very fast and it offers the right kind of data structures that you would need for rate limiting. The speed helps minimize added latency, plus Redis has support for managing single leader replication.
8.  **Explain the key difference between fixed window and sliding window rate-limiting algorithms.** Fixed window rate limiting resets the request count at fixed time intervals, but allows bursts of requests at the window boundaries. Sliding window rate limiting considers a moving time window, preventing bursts and offering greater accuracy.
9.  **How does the sliding window algorithm prevent the exploitation seen in fixed window algorithms?** The sliding window maintains a linked list, allowing one to accurately calculate the requests made over a sliding window, which is far more precise than a fixed window. The list is constantly culled and updated, which means no exploitation is possible.
10. **What concurrency considerations are important when implementing a rate limiter?** Concurrency requires careful consideration to avoid race conditions and data inconsistencies. The increment of the count in a fixed window algorithm must be done atomically, and linked lists in the sliding window algorithm require locking mechanisms to prevent modification exceptions.

III. Answer Key
---------------

1.  To prevent users (malicious or otherwise) from overwhelming a system with too many requests, thus maintaining stability and performance. It protects against abuse and ensures fair resource allocation.
2.  Rate limiters are a necessary function for a smooth, efficient experience. Adding latency would frustrate users, potentially driving them away and hurting overall user experience.
3.  User ID-based rate limiting is good for signed-in users, preventing individual accounts from overloading services, but falls apart when users create multiple accounts. IP address-based rate limiting works for anonymous users, but may incorrectly throttle legitimate users sharing the same IP.
4.  A rate limiter interface typically includes a function, 'rate_limit', that takes User ID, IP address, the service name and the request timestamp as input. It returns a Boolean value indicating whether the request should be throttled.
5.  Local rate limiting reduces network calls because it is done in each service, but it does not shield application servers from bursts of traffic and tightly couples application and rate-limiting scaling. Distributed rate limiting protects application servers and allows independent scaling, but introduces extra network calls.
6.  Caching reduces network calls to the rate limiter. Implementing a write-back cache on the load balancer can store the request counts of frequent users, filtering out many bad actors requests before they ever reach the rate limiter.
7.  Redis is suitable because it's an in-memory database which is very fast and it offers the right kind of data structures that you would need for rate limiting. The speed helps minimize added latency, plus Redis has support for managing single leader replication.
8.  Fixed window rate limiting resets the request count at fixed time intervals, but allows bursts of requests at the window boundaries. Sliding window rate limiting considers a moving time window, preventing bursts and offering greater accuracy.
9.  The sliding window maintains a linked list, allowing one to accurately calculate the requests made over a sliding window, which is far more precise than a fixed window. The list is constantly culled and updated, which means no exploitation is possible.
10. Concurrency requires careful consideration to avoid race conditions and data inconsistencies. The increment of the count in a fixed window algorithm must be done atomically, and linked lists in the sliding window algorithm require locking mechanisms to prevent modification exceptions.

IV. Essay Questions
-------------------

1.  Discuss the trade-offs between implementing a rate limiter as a local service within each application server versus as a dedicated distributed service. In what scenarios would each approach be more suitable?
2.  Describe the process of scaling a distributed rate limiter to handle a large number of users and requests. What partitioning strategies could be used, and how would a load balancer direct traffic to the appropriate rate limiter instance?
3.  Compare and contrast the fixed window and sliding window rate-limiting algorithms. Explain the advantages and disadvantages of each, and provide examples of situations where one algorithm would be preferred over the other.
4.  Explain how caching can be integrated into a rate-limiting system to improve performance. Describe the potential benefits and challenges of using a write-back cache on the load balancer in this context.
5.  Discuss the concurrency challenges involved in implementing rate-limiting algorithms, particularly when using in-memory data structures like linked lists. Describe the techniques that can be used to address these challenges and ensure data consistency.

# Designing Yelp: A System Design Interview Perspective

**This YouTube transcript explains how to design a system similar to Yelp or Google Places.** **It details the functional requirements, such as finding restaurants by location and reading/posting reviews.** **The video then explores capacity estimates and prioritizes optimizing the read path due to the higher frequency of reading reviews compared to writing them.** **The discussion covers database schema design, geospatial indexing (using geohashes, quad trees, or Hilbert curves), data caching strategies, and geo-sharding to balance load.** **Finally, the transcript describes how to implement restaurant search by name using an inverted index and change data capture to keep the search index up to date.**

Briefing Document: 
------------------

**Overview:**

This document summarizes a system design walkthrough for building a Yelp-like application, focusing on the key considerations and architectural choices involved. The primary goal is to design a system that efficiently handles the read-heavy workload (reading reviews and finding restaurants) while also supporting write operations (posting reviews, updating restaurant information).

**I. Problem Requirements and Constraints:**

-   **Core Functionality:**Find restaurants within a given radius of a location (lat/long).
-   Read reviews for a specific restaurant.
-   Post reviews for a restaurant.
-   **Optimization Goal:** Prioritize the read path (finding restaurants and reading reviews) over the write path (posting reviews). "Because of the fact that we are mostly reading reviews on Yelp relative to the number that are written basically we can prioritize the read path and we can deprioritize the right path."
-   **Tradeoffs:** The design acknowledges the need for tradeoffs between read and write performance.
-   **Scalability and Fault Tolerance:** The system needs to be scalable and fault-tolerant to handle a large number of restaurants, reviews, and users.

**II. Capacity Estimates:**

-   **Number of Restaurants:** Estimated at 10 million globally, 1 million in the US.
-   **Reviews per Restaurant:** Estimated at 100 reviews per restaurant.
-   **Review Size:** Estimated at 1,000 characters (1,000 bytes) per review.
-   **Total Raw Review Data:** Roughly 1 terabyte. "If each review is around 1,000 characters long because people on Yelp just love to be so very wordy that can be around 1,000 bytes and so if we multiply all of those figures together maybe we have a terabyte of raw review data."
-   **Partitioning is a Must:** Even with only 1 TB of data, partitioning is needed for future-proofing.

**III. Key Design Components and Considerations:**

-   **A. Reading Reviews:Database Schema:** Author ID, Restaurant ID, Date, Number of Stars, Content.
-   **Replication:** Single-leader replication for fault tolerance and read speed boost. "Since we're not optimizing for our right speeds I don't really feel the need to do anything complex here I think single leader replication will make sure that we don't have to deal with any sorts of right conflicts while also ensuring that we can still get the read speed boost of having multiple replicas."
-   **Partitioning:** Hash range on Restaurant ID. "We can just go ahead and do a hash range on the restaurant ID the reason for this being that the one time we actually access these reviews is per restaurant so we want to make sure that all the reviews for a given restaurant are on the same node."
-   **Database Choice:** MySQL or another relational database, prioritizing correctness and ease of use over raw performance. "Performance is really not my main concern here I'm more concerned with correctness and ease of use and as a result I think just using something like MySQL or any other type of typical relational database is going to be just fine for this particular problem."
-   **Caching:** A caching layer is introduced to shield the database partition from excessive reads of popular restaurants.
-   **B. Finding Restaurants (Geospatial Indexing):Problem:** Standard database indexes are inefficient for geospatial queries (finding points within a radius).
-   **Solution:** Geospatial Indexing: "The main idea is this right we're looking for all the points near a certain X and y-coordinate in reality it would be lat long but X and Y is going to be a lot easier for us to think about so the idea right here is Imagine we've got this point right this is me and I want to find everything within this circle right here that's exactly what we're looking to do"
-   **Geohash Implementation:**Divide the world into a grid of boxes (quadtree-like structure).
-   Assign a geohash to each box.
-   Sort restaurants based on their geohash.
-   Benefits: Points close in geohash space are also close in physical space.
-   Search becomes a logarithmic operation: "This entire process of figuring out um which geohash our Point actually belongs in is O of log n because every single time we are basically logarithmically limiting the search space that we care about in order to find that geohash."
-   Binary search is then used to search the database index.
-   **Distance Calculation:** Pythagorean theorem (high school geometry!) is used to filter results within the radius.
-   This distance calculation should ideally be done in the SQL to save bandwidth.
-   **Restaurant Data Schema for Geo Index:** Geohash, Restaurant ID, Name, S3 link (photo), Number of Reviews, Total Stars, Latitude, Longitude.
-   **Denormalization:** Denormalizing data (including restaurant name, photo, and aggregated review stats) into the Geo index avoids distributed joins. "It would be better if we denormalize our data a little bit we can use something like change data capture in order to do so."
-   **Incrementing Review Counters:** Because number of reviews and total stars are integer counters that will be incremented, the design allows for either two-phase commit when submitting a review or use change data capture to ensure every single message will eventually get processed.
-   **Database Choice for Geo Index:** Utilize existing database solutions with Geo index support (e.g., Elasticsearch, PostgreSQL). "I would just recommend not Reinventing the wheel a lot of existing database Solutions already allow supporting Geo indexes."
-   Elasticsearch or Postgres are good candidates
-   **C. Data Caching:Purpose:** Cache popular queries for both fetching locations and fetching reviews. "It would be great if we could go ahead and cash certain popular queries the reason being that certain restaurants a lot of people are going to look up the reviews for."
-   **Strategy:** LRU (Least Recently Used) cache.
-   Start with cache misses and pull in data as requested.
-   Popular queries will naturally stay in the cache.
-   **Pre-population:** Potentially pre-populate the cache with known popular locations (e.g., restaurants near Times Square).
-   **Cache Type:** Write-around cache. "I think right around makes the most sense in this case it is going to ensure that our data is going to you know use the database as the source of Truth and unlike a right through cache it is not going to impact our right latencies." This minimizes the impact on write latency.
-   **D. Geo Sharding:Problem:** Balancing load across shards when restaurant density is uneven (e.g., New York City vs. rural areas).
-   **Goal:** Ensure each node has a similar amount of data and handles a similar number of queries. "Ideally what we want is every single node is going to have similar amounts of data in terms of the number of restaurants and what we'd also like is that every single node is going to handle similar numbers of queries"
-   **Partitioning Strategy:**Partition by geohash range to maintain data locality. "We definitely want to be partitioning by geohash range the reason being that points that are close to one another as geohashes are close to one another in physical space."
-   Assign smaller geohash ranges to high-density areas and larger ranges to low-density areas.
-   Goal: Each partition gets similar amounts of load.
-   **Dynamic Partitioning (Optional):**Rebalance partitions as load changes.
-   Requires careful management to ensure correctness (copying data, updating load balancers).
-   Could use libraries (e.g., Google's S2) for computing load per geohash.
-   **Rebalancing:**Graph the load per geohash range.
-   Adjust partition boundaries so each partition handles roughly equal load.
-   Rebalance partitions accordingly.
-   Assumes load metrics don't change dramatically over time.
-   **E. Search:Functionality:** Search restaurants by name.
-   **Implementation:** Inverted index (e.g., Elasticsearch).
-   **Geo-indexing in Search:** Utilize Geo-indexing capabilities of the search index to incorporate geolocation into search scoring.
-   **Derived Data:** Treat the search index as derived data updated via Change Data Capture (CDC). "We should probably just make it some derived data which at least to me tends to ring a bell in my head saying change data capture."

**IV. System Diagram and Data Flow:**

-   **Right Service:**Clients write reviews and restaurant data.
-   Reviews: MySQL sharded on Restaurant ID.
-   Change Data Capture (CDC) -> Stateful Consumer (Spark Streaming) -> Increment num_stars and num_reviews in Restaurants table.
-   Restaurants: MySQL sharded on ID.
-   Change Data Capture (CDC) -> Kafka Queue -> Stateful Consumer (Flink) -> Update Geo index and Inverted Search Index (Elasticsearch).
-   **Read Service:**Clients read reviews, find nearby restaurants, and search by name.
-   LRU Caching Layer (Redis) in front of all read operations.
-   Reviews: Read from MySQL.
-   Nearby Restaurants: Query Geo index (Elasticsearch or PostgreSQL).
-   Search by Name: Query Inverted Search Index (Elasticsearch).

**V. Technologies Mentioned:**

-   MySQL
-   Elasticsearch
-   PostgreSQL
-   Redis
-   Kafka
-   Flink
-   Spark Streaming
-   Change Data Capture (CDC)

**VI. Key Takeaways:**

-   A successful Yelp/Google Places system relies on a well-designed architecture that prioritizes read performance while efficiently handling writes.
-   Geospatial indexing is crucial for quickly finding restaurants within a given radius.
-   Caching plays a vital role in improving read latencies for popular restaurants and locations.
-   Careful consideration of sharding strategies is necessary to balance load across partitions, especially in areas with varying restaurant densities.
-   Change Data Capture (CDC) is a valuable technique for keeping derived data (e.g., search index, aggregated review statistics) up-to-date.

This document provides a high-level overview of the key considerations involved in designing a Yelp-like system. A deeper dive into each component would be necessary for a complete implementation.

Study Guide
===========

Quiz
----

**Answer each question in 2-3 sentences.**

1.  What are the three primary problem requirements for designing a system like Yelp?
2.  Why is the read path prioritized over the write path in the design of Yelp?
3.  What is the initial estimate given for the amount of raw review data that Yelp might store, and why might the actual storage requirement be higher?
4.  Explain why single leader replication is chosen for the reviews database.
5.  Explain how a geospatial index works in the context of finding restaurants.
6.  What are geohashes, and how do they help with geospatial indexing?
7.  Why does denormalizing data within the Geo index improve performance?
8.  What are the benefits of using a caching layer for Yelp's data, and what type of cache is recommended?
9.  Why is it important to partition by geohash range in a geo-sharded system?
10. How does an inverted index help facilitate searching for restaurants by name?

Quiz Answer Key
---------------

1.  The primary requirements are to find restaurants within a given radius of a location, read reviews for a specific restaurant, and allow users to post their own reviews. These features form the core functionality of the platform.
2.  Because users primarily read reviews far more often than they write them, the system design prioritizes optimizing for fast read speeds. This ensures a smooth experience when users are browsing and making decisions based on existing reviews.
3.  The initial estimate is around 1 terabyte of raw review data. The actual storage requirement might be higher due to additional metadata, characters, and other associated data stored with each review.
4.  Single leader replication provides fault tolerance while avoiding complex write conflicts. Since write speeds are not a priority, this setup ensures data consistency and allows for faster reads from multiple replicas.
5.  A geospatial index maps a 2D space (like latitude and longitude) into a single dimension. This is done using techniques like geohashes to ensure that locations close in physical space are also close in the index.
6.  Geohashes are a way to divide a 2D space into a hierarchy of boxes, each represented by a unique string. By mapping each restaurant to a geohash, nearby restaurants will have similar geohashes, making proximity searches more efficient.
7.  Denormalizing data in the Geo index avoids the need for distributed joins, which can slow down queries. By including all necessary data in the Geo index, the system can retrieve restaurant information and location data in a single query.
8.  Caching popular queries improves performance by reducing the load on the database. An LRU (Least Recently Used) cache is recommended to store frequently accessed data, ensuring that popular restaurants and search terms are readily available.
9.  Partitioning by geohash range ensures that restaurants in similar areas are located on the same node. This improves data locality for queries, reducing the need to hit multiple partitions to retrieve nearby restaurants.
10. An inverted index maps search terms to the restaurant IDs that contain those terms. This makes searching for restaurants by name efficient, as the system can quickly identify and retrieve relevant restaurants based on the search query.

Essay Questions
---------------

1.  Discuss the tradeoffs between prioritizing read and write speeds in a system like Yelp. How does the chosen design reflect these tradeoffs, and what alternative approaches could be considered?
2.  Explain the different approaches to geospatial indexing (geohash, quad tree, Hilbert curve). Compare and contrast these methods, discussing their advantages and disadvantages in the context of Yelp's requirements.
3.  Describe the data flow for writing a new review, from the client to the database and cache. How is data consistency maintained, and what are the potential bottlenecks in this process?
4.  Discuss the challenges of geo-sharding a system like Yelp, where restaurant density varies significantly across regions. How can dynamic partitioning and load balancing be used to address these challenges?
5.  Analyze the role of change data capture (CDC) in the design of Yelp. How does CDC facilitate keeping the Geo index and search index up-to-date, and what are the alternative approaches to maintaining data consistency?

Glossary of Key Terms
---------------------

-   **Read Path:** The sequence of operations involved in retrieving data from the system.
-   **Write Path:** The sequence of operations involved in writing or updating data in the system.
-   **Single Leader Replication:** A database replication strategy where one node is designated as the primary (leader) for writes, and changes are propagated to other nodes (followers).
-   **Partitioning:** Dividing a large dataset into smaller, more manageable pieces that can be stored and processed on different nodes.
-   **Geospatial Index:** A specialized index that optimizes queries based on geographic location.
-   **Geohash:** A hierarchical spatial data structure that divides the world into a grid of cells, each represented by a unique string.
-   **Quad Tree:** A tree data structure in which each internal node has exactly four children; commonly used to partition a two-dimensional space.
-   **Hilbert Curve:** A space-filling curve that maps multi-dimensional space to one dimension while preserving locality.
-   **Denormalization:** Adding redundant data to a database table to improve read performance by reducing the need for joins.
-   **Change Data Capture (CDC):** A set of techniques used to identify and track changes to data in a database, allowing those changes to be propagated to other systems.
-   **LRU Cache (Least Recently Used):** A caching algorithm that removes the least recently accessed item when the cache is full.
-   **Geo-sharding:** Partitioning data based on geographic location to distribute load and improve query performance.
-   **Inverted Index:** An index data structure storing a mapping from content, such as words or numbers, to its locations in a database file, document, or a set of documents.
-   **Flink:** A stream processing framework that provides exactly-once processing guarantees.
-   **Kafka:** A distributed, fault-tolerant streaming platform that enables building real-time data pipelines and streaming applications.
-   **Elasticsearch:** A distributed, RESTful search and analytics engine capable of solving a growing number of use cases.
-   **Consistent Hashing:** A technique used to distribute data across a cluster of servers in a way that minimizes the impact of adding or removing servers.
-   **Two-Phase Commit:** A distributed transaction protocol that ensures all participating nodes either commit or rollback a transaction.
-   **Idempotent:** An operation that can be applied multiple times without changing the result beyond the initial application.
-   **S2 Library:** Google's library for spatial indexing and geometry on the sphere.

# Recommendation Engine Design

**The YouTube video transcript details the design of a recommendation engine, contrasting batch computation with real-time approaches.** **It outlines functional requirements, capacity estimations, and API design considerations for such a system.** **The video emphasizes a shift towards real-time recommendation, explaining the retrieval and ranking steps involved in generating personalized suggestions.** **Retrieval utilizes embedding models and indexes to find candidate items, while ranking refines these candidates based on user-specific data.** **The system incorporates filtering mechanisms and distributes computation across multiple servers to optimize performance.** **Finally, user logs are fed back into the system to continuously improve model accuracy and relevance.**

Briefing Document: 
------------------

**Source:** Excerpts from "Recommendation Engine Design Deep Dive with Google SWE! | Systems Design Interview Question 20" (YouTube Video Transcript)

**Date:** October 26, 2024

**Overview:**

This document summarizes a YouTube video discussing the design of a recommendation engine, focusing on system design considerations and contrasting batch computation with real-time approaches. The speaker, presumably a software engineer, outlines functional requirements, capacity estimates, API design, and database schema before diving into the architecture of both batch and real-time recommendation systems. The emphasis is on a real-time system using retrieval and ranking techniques leveraging embeddings and Bloom filters.

**Key Themes and Ideas:**

1.  **Functional Requirements:** The core functional requirement is to "build a recommendation engine for our service" that suggests relevant items (videos, products, etc.) to users based on their past interests.
2.  **Capacity Estimates:** The system is designed for massive scale:

-   1 Billion Users
-   100 Million New Items Added Per Day.
-   This necessitates distributed computation and careful architectural considerations to handle the high volume of data and updates.

1.  **API Design:** A simple API is sufficient, primarily focused on "get recommendations for a given user ID".
2.  **Database Schema:** The schema includes tables for "items" (videos, products, etc.) and potentially "the actual recommendations themselves if we're going to be caching them".
3.  **Batch Computation vs. Real-Time Recommendation:**

-   **Batch Computation (Traditional Approach):** This involves offline computation of recommendations, typically running daily Spark jobs. User and item data are ingested, a model is trained or updated, and a cache is populated with recommendations.
-   **Pros:** Simple to manage, less demanding in terms of availability (no need for 24/7 on-call engineers).
-   **Cons:** Can be computationally wasteful (calculating recommendations for inactive users), doesn't account for rapidly changing content (e.g., new TikTok videos), and provides a poor first-time user experience.
-   Quote: "...every single day we take in our user data we take in our item data and we take in our existing model and then we can run a once per day spark job that basically goes ahead and populates a an existing cache with all of the recommendations that we want"
-   **Real-Time Recommendation (Cutting-Edge Approach):** Focuses on generating recommendations in real-time as a user interacts with the platform. It consists of two major steps: retrieval and ranking.
-   Quote: "...the real-time prediction is done in two major steps...retrieval and ranking..."
-   **Pros:** Incorporates more up-to-date data, provides more relevant suggestions, and offers a better experience for new users.
-   **Cons:** More complex architecture, potentially sacrificing some latency for accuracy.

1.  **Real-Time Recommendation: Retrieval Phase:**

-   Involves loading a large number (e.g. a few thousand) of potential candidates (items) that are generally relevant to the user's past behavior. This is done quickly.
-   Heavily based on the concept of **embeddings**: turning items (words, images, products) into multi-dimensional vectors.
-   Quote: "embeddings are saying take anything that you know you might be trying to provide recommendations on...and turn it into a vector a multi-dimensional vector..."
-   An **embedding model** (trained offline) is used to create these vectors based on the user's past interactions.
-   Candidates are retrieved by finding embeddings that are "closest" (most similar) to the embeddings of items the user has interacted with previously.
-   **Index Optimization:** Instead of using geospatial indexes the suggested optimization is: "for each embedding calculate the Thousand most offline in a batch process and save them in some index and then that way we can really quickly just directly fetch the Thousand for each embedding instead of having to do a sort of binary search like we would with a geospatial index so instead of being a logarithmic complexity it goes to a constant time complexity at the cost of having to use some more space".
-   **Filtering with Bloom Filters:** Used to quickly eliminate undesirable candidates (e.g., items the user has already seen, items with curse words on a children's account, items the user has already purchased). Bloom filters are particularly useful for blacklisting.
-   Quote: "...the point of Bloom filters is that for a bunch of items in a set...what you can do is you hash them all to a few different hash functions and then that way you can very quickly tell if a given item is not in the set..."

1.  **Real-Time Recommendation: Ranking Phase:**

-   The ranking phase takes the candidate set from retrieval and narrows it down to the top N (e.g., 10-15) items to display to the user.
-   This involves more complex, user-specific computations.
-   Ranking servers, that are horizontally scaled-out, perform computations for each candidate.
-   After scores are calculated for each item, the results are sorted and merged, where the top videos with the highest possible scores are passed to the user.
-   Ranking models should be calculated offline.
-   User interaction data is fed back into the system (via Kafka queue into Hadoop cluster) to continuously update the models.

1.  **System Diagram:** The speaker describes a typical architecture with:

-   Client
-   Load Balancer
-   Recommendation Service Nodes (horizontally scaled)
-   User Database (for user history)
-   Embedding Model and Neighboring Index
-   Ranking Servers (horizontally scaled)
-   Kafka Queue
-   Hadoop Cluster (for model training)

**Conclusion:**

The video provides a valuable overview of recommendation engine design, highlighting the trade-offs between batch and real-time approaches. The real-time recommendation engine design, using retrieval and ranking with embeddings and Bloom filters, offers a more dynamic and personalized experience at the cost of increased complexity and potential latency. The speaker emphasizes the importance of continuous model updates based on user feedback.

Study Guide
===========

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  What is the primary goal of a recommendation engine?
2.  Explain the difference between batch computation and real-time recommendation.
3.  Why is real-time recommendation becoming more popular despite the advantages of batch processing?
4.  What are the two major steps involved in real-time prediction?
5.  Describe what embeddings are and their purpose in the retrieval process.
6.  Explain how the embedding model is trained and whether it is part of the read path.
7.  How does the retrieval process use nearest neighbors?
8.  What are Bloom filters, and how are they used as an optimization in the retrieval process?
9.  What is the role of ranking servers in the real-time recommendation process?
10. How are user logs used to improve the machine learning models over time?

Quiz Answer Key
---------------

1.  The primary goal of a recommendation engine is to predict and suggest items (e.g., videos, products) that a user is likely to be interested in based on their past behavior and preferences. This enhances user engagement and satisfaction by providing personalized content.
2.  Batch computation involves pre-computing recommendations offline, typically once per day, for all users. Real-time recommendation generates suggestions dynamically when a user requests them, allowing for more up-to-date and personalized recommendations based on immediate user data.
3.  Real-time recommendation is gaining traction because it incorporates more current data, provides more timely suggestions, and caters to new users immediately. This is particularly important for platforms with rapidly changing content, like TikTok.
4.  The two main steps in real-time prediction are retrieval and ranking. Retrieval involves quickly identifying a few thousand potential candidates based on user history, while ranking narrows down those candidates to the top 10-15 most relevant items for the user.
5.  Embeddings are vector representations of items (e.g., words, images, products) created using machine learning models. They encode the semantic relationships between items, allowing the system to identify similar items based on their proximity in the vector space.
6.  The embedding model is trained offline using machine learning. It analyzes item data to create vector representations, and it is not part of the read path, which means it doesn't slow down user experience.
7.  In retrieval, nearest neighbors are used to find items that are similar to what a user has interacted with in the past. For each item a user has engaged with, the system identifies the closest embeddings, representing similar items that could be potential recommendations.
8.  Bloom filters are a probabilistic data structure used to quickly determine if an item is not in a set. In the context of recommendation engines, they efficiently filter out unwanted candidates (e.g., previously seen videos, blacklisted content) from the retrieval results.
9.  Ranking servers are responsible for scoring the candidate items generated during the retrieval phase. They use machine learning models to evaluate each candidate based on its relevance to the user, and the results are used to determine the top recommendations.
10. User logs, which record user interactions with recommendations (e.g., clicks, views), are fed back into a Hadoop cluster. This data is then processed using Spark jobs to update and refine the machine learning models, ensuring that the recommendations remain relevant and accurate over time.

Essay Questions
---------------

1.  Compare and contrast batch computation and real-time recommendation engines. Discuss the advantages and disadvantages of each approach, and provide examples of scenarios where each would be most appropriate.
2.  Explain the retrieval and ranking steps in detail, emphasizing the role of embeddings and machine learning models. How do these steps work together to deliver personalized recommendations?
3.  Describe the architectural components of a real-time recommendation engine, including the roles of the client, load balancer, recommendation service, user database, embedding model, ranking servers, and message queue.
4.  Discuss the various machine learning considerations involved in designing a recommendation engine, including model training, feature selection, and evaluation metrics.
5.  Imagine you are designing a recommendation engine for a specific platform (e.g., e-commerce, music streaming, social media). Outline the key design decisions you would make, justifying your choices based on the platform's characteristics and user needs.

Glossary of Key Terms
---------------------

-   **Recommendation Engine:** A system designed to predict and suggest items that a user is likely to be interested in.
-   **Batch Computation:** Processing a large volume of data in a single, scheduled job, typically offline.
-   **Real-Time Recommendation:** Generating recommendations dynamically in response to a user's request, using up-to-date data.
-   **API:** Application Programming Interface, a set of rules and specifications that software programs can follow to communicate with each other.
-   **Embedding:** A vector representation of an item (e.g., word, image) in a multi-dimensional space, capturing semantic relationships.
-   **Retrieval:** The process of identifying a set of potential candidate items for recommendation based on user history.
-   **Ranking:** The process of sorting candidate items based on their relevance to the user, using machine learning models.
-   **Candidate:** An item that is being considered for recommendation to a user.
-   **Bloom Filter:** A probabilistic data structure used to quickly determine if an element is not in a set.
-   **Kafka:** A distributed streaming platform used for building real-time data pipelines and streaming applications.
-   **Hadoop:** An open-source framework for distributed storage and processing of large datasets.
-   **Spark:** A fast and general-purpose cluster computing system for big data processing.
-   **Load Balancer:** A device or software that distributes network traffic across multiple servers to prevent overload.
-   **Logistic Regression:** A statistical model that uses a logistic function to model the probability of a binary outcome.
-   **Neural Network:** A computational model inspired by the structure and function of biological neural networks.
-   **Systems Design:** The process of defining the architecture, modules, interfaces, and data for a system to satisfy specified requirements.

# Designing Uber or Lyft

**This YouTube transcript presents a detailed explanation of the system design behind Uber/Lyft.** **The video focuses on key aspects like ride requests, driver matching, and calculating routes.** **It explores geospatial indexing for locating nearby drivers and riders and analyzes the challenge of maintaining real-time location updates.** **The presenter proposes solutions for partitioning data and addressing race conditions during driver selection.** **The video concludes with an intricate discussion of route recreation using Hidden Markov Models to handle imprecise GPS data and uses Kafka, Flink, and other technologies to implement the solutions.**

Briefing Document: 
------------------

**Source:** Excerpts from "10: Design Uber/Lyft | Systems Design Interview Questions With Ex-Google SWE"

**Main Themes:**

This video excerpt focuses on designing the core systems behind a ride-hailing service like Uber/Lyft. The core functionalities addressed are:

1.  **Requesting a Ride & Driver Matching:** How to efficiently find and match a rider with a nearby driver.
2.  **Real-Time Location Tracking:** Handling constant updates of driver and rider locations.
3.  **Route Recreation:** Accurately determining the actual route taken, even with imperfect GPS signals.

The interviewer takes a practical approach, acknowledging that a full implementation is incredibly complex and prioritizes a deep dive into specific challenges and solutions. He emphasizes understanding the *why* behind design choices, including capacity estimation and potential bottlenecks. He stresses the importance of leveraging existing knowledge (specifically, his previous video on Yelp's system design for geolocation data).

**Key Ideas and Facts:**

**1\. Capacity Estimation & Scalability:**

-   The speaker starts with crude calculations to understand the scale of the problem:
-   "Let's imagine that everyone in the world is using Uber and that everyone in the world is taking one 20-minute ride per week...that means that there are going to be on average around 20 million rides at a time."
-   This leads to the realization that simple solutions won't scale and partitioning/replication will be necessary.
-   Keeping track of location of users can require a ton of network requests:
-   "if I'm assuming that there are 20 million ride requests per 20 minutes that means that equates to around 15,000 requests per second...most servers are not going to be able to handle 15,000 requests per second."

**2\. Finding Nearby Drivers (Geolocation Indexing):**

-   The interviewee builds upon the concepts covered in a previous video on Yelp, requiring viewers to have prior knowledge of the topic of Geo indexing.
-   The drawbacks of a standard database index for geospatial data are mentioned: poor data locality.
-   The video discusses the use of geospatial indexes (e.g., using bounding boxes and geohashes):
-   "In a geospatial index we basically take our 2D plane and we break it down into a bunch of different bounding boxes...by breaking down our 2D plane into a bunch of different bounding boxes we can actually just search within a box in order to find all the points that are relevant to us."
-   This allows for efficient range queries.

**3\. Real-Time Location Updates & Partitioning:**

-   Websockets are suggested for real-time bidirectional communication between clients (riders/drivers) and servers to reduce overhead:
-   "websockets are a real-time bidirectional connection between the server and the client...in addition to actually sending out our updates to the server we can now actually see where other drivers on that same server are located"
-   Partitioning the geospatial index is crucial for handling the high volume of location updates.
-   Load is not evenly distributed; high-density areas require special consideration. Partitioning based on geohash range.
-   Secondary sharding (by user ID) within a geographic area is proposed to handle extreme situations (e.g., Time Square on New Year's Eve):
-   "for these situations what you can actually have is within a particular geographic area you can even do a secondary Shard on that and you can just Shard based on user ID"
-   Drivers need to switch partitions as they move, requiring the breaking and re-establishment of websocket connections.

**4\. Data Storage & Availability:**

-   Prioritize speed and low latency over data persistence for location data. Data is constantly updated so some loss is acceptable.
-   Suggests keeping the partitioned geolocation index in-memory. "Ideally we would keep our whole partitioned geolocation index on our servers in memory."
-   Use standby nodes and Zookeeper for failover but be aware of the Thundering Herd problem when nodes reconnect to new replicas.

**5\. Driver Selection (Matching):**

-   Discusses various heuristics for selecting a driver: closest, first to respond, highest rating.
-   Race conditions are possible when multiple drivers respond to a ride request concurrently.
-   Locking mechanisms or database transactions are necessary to prevent multiple drivers from being assigned the same ride. "In order to actually conclusively make sure that no one uh ever thinks they have the ride at another time we need some sort of locking mechanism."
-   Suggests using predicate locks or materializing a conflict to deal with edge cases.

**6\. Route recreation with imperfect GPS signals (Advanced Topic):**

-   Addresses the challenge of accurately determining the route taken when GPS signals are noisy.
-   Proposes using a Hidden Markov Model (HMM) to probabilistically determine the most likely route.
-   Explains the concepts of emission probabilities (likelihood of seeing a GPS signal given a location on the road) and transition probabilities (likelihood of moving from one point on the road to another).
-   Dynamic Programming is used to solve for an optimal route via the Viterbi algorithm.
-   Inter-partition communication: kfka Qs and Apache Flink are recommended: "I'm proposing using Kofa Q's as well as Flink"
-   Kafka Q's provides replayable and distributed logs. Apache Flink will keep state, processing messages at least once.

**Quotes of Particular Importance**

-   "the idea here is that we want similar locations to have similar geohashes...and they should be on the same node and of course this basically just means that we want to be doing partitioning by our geohash range but in a way that actually balances them"
-   "We don't really care if data is persistent...I care about data speed and low latency"
-   "...For me to really be confident that I actually had the most likely sequence of roads I would have to try this for you know a a a a c um a e a d and on and on and on and on and on even though these guys have zero probability for emission if we really want to be uh sure that we haven't seen everything we still have to try them right"

**In summary:** This video provides a high-level overview of the systems design challenges involved in building a ride-hailing service. It emphasizes scalability, real-time data handling, and the use of appropriate data structures and algorithms (geospatial indexing, Hidden Markov Models). The interviewer stresses the importance of understanding the underlying principles and trade-offs involved in each design decision.

Study Guide
===========

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  What are the four key functionalities this system design review aims to cover?
2.  Explain the purpose of crude capacity estimates in the context of system design.
3.  Why is a typical index not efficient for geospatial queries?
4.  How does a geospatial index improve data locality for location-based searches?
5.  Explain why WebSockets are preferred over HTTP requests for updating driver locations.
6.  Describe the partitioning strategy for geolocation data.
7.  What is a "Thundering Herd" problem?
8.  Why is low latency prioritized over data persistence when updating rider and driver locations?
9.  Name three different heuristics for picking a driver and describe some of the pros and cons.
10. How can a database table called claimed_trips help to eliminate race conditions with picking a driver?

### Quiz - Answer Key

1.  The four key functionalities are requesting a ride, seeing nearby drivers, matching with a driver, and providing distance and time estimations. A mapping service like Google Maps provides distance and time estimations, which we will later expand on.
2.  Crude capacity estimates help determine the order of magnitude of the system's requirements. This informs decisions about whether partitioning, replication, or other scaling strategies are necessary.
3.  A typical index, sorted first on the x-position and then the y-position, is not efficient because it cannot guarantee data locality for range queries. Data points within a given radius can be spread out all over the index.
4.  A geospatial index breaks down the 2D plane into a hierarchy of bounding boxes (geohashes). Points close to each other in 2D space are stored next to each other on disk, providing data locality for faster range queries.
5.  WebSockets provide a real-time, bidirectional connection, reducing overhead by avoiding the need to send HTTP headers with every location update. This can be useful for improving the efficiency of server-client communications.
6.  Geolocation data is partitioned based on geohash ranges, ensuring similar locations are on the same partition. Data should be partitioned so that major cities do not overburden any one server.
7.  The "Thundering Herd" problem occurs when multiple clients simultaneously attempt to reconnect to a server. This can overload the server.
8.  When updating location information, low latency is prioritized over data persistence because location data changes frequently. Losing a small amount of data is not a big deal because the location will soon update again.
9.  The first is the closest driver, which is easy to find using a geospatial index. The second is sending the ride request to all drivers and having the first one to accept win the ride. The third is allowing the highest rated drivers to be assigned to the ride. This would require the index to have access to all drivers actual review score, which could add latency.
10. The database table called claimed_trips will first start as an unclaimed trip, and then it becomes a claimed trip when a driver is assigned to it. When the writer asks the server for a driver, the first thing that the server does is assign an ID for that trip.

Essay Questions
---------------

1.  Discuss the trade-offs between different approaches to driver selection (e.g., closest, highest rating). What factors should be considered when choosing a specific approach for Uber/Lyft?
2.  Elaborate on the complexities of route recreation in the context of Uber/Lyft. How does the use of probabilistic models, such as Hidden Markov Models, address the challenges posed by imperfect GPS signals?
3.  Compare and contrast the roles of Kafa and Flink in Uber/Lyft's path-calculation system. What advantages do these technologies offer in terms of data processing, fault tolerance, and scalability?
4.  Analyze the video's discussion of load balancing and partitioning in the context of a real-world example like Time Square on New Year's Eve or the San Francisco Airport. How can the system dynamically adapt to handle localized spikes in demand?
5.  Trace the path of a GPS signal from a rider's phone to the trip's database, detailing the key components and processes involved in route calculation. How is data consistency and accuracy maintained throughout this process?

Glossary of Key Terms
---------------------

-   **Capacity Estimates:** Approximations of the resources (e.g., storage, network bandwidth) required to handle the system's load.
-   **Geospatial Index:** A specialized data structure used to efficiently query data based on location.
-   **Geohash:** A hierarchical spatial index that divides the earth into a grid of bounding boxes.
-   **Data Locality:** The principle of storing related data close together on disk to minimize access times.
-   **WebSockets:** A communication protocol that provides full-duplex communication channels over a single TCP connection.
-   **Partitioning:** Dividing a database or system into smaller, more manageable parts to improve scalability and performance.
-   **Thundering Herd Problem:** A situation where a large number of clients simultaneously try to access a resource, overwhelming the system.
-   **Load Balancer:** A device or software that distributes network traffic across multiple servers.
-   **HTTP:** Hypertext Transfer Protocol - a protocol for sending and receiving web requests and responses
-   **Geolocation:** The process of determining the geographical location of a device or person.
-   **Heuristics:** A problem-solving approach (algorithm) that uses a practical method to achieve immediate goals.
-   **Race Conditions:** A situation where multiple threads or processes access and modify shared data concurrently, leading to unpredictable results.
-   **Transactions:** A database mechanism that ensures a series of operations are treated as a single atomic unit, either all succeeding or all failing.
-   **Predicate Locks:** A database locking mechanism that locks rows based on a specific condition or predicate.
-   **Hidden Markov Model (HMM):** A statistical model used to represent systems that evolve over time, where the system's state is hidden but can be inferred from observed outputs.
-   **Emission Probabilities:** In an HMM, the probability of observing a particular output given a specific hidden state.
-   **Transition Probabilities:** In an HMM, the probability of transitioning from one hidden state to another.
-   **Viterbi Algorithm:** A dynamic programming algorithm used to find the most likely sequence of hidden states in an HMM, given a sequence of observations.
-   **Kafka:** A distributed, fault-tolerant, high-throughput streaming platform.
-   **Flink:** A distributed stream processing framework for stateful computations.
-   **Stateful Consumer:** A process that maintains state across multiple events or messages.
-   **Change Data Capture (CDC):** A technique for capturing and tracking changes made to data in a database.
-   **HDFS:** Hadoop Distributed File System - a distributed file system for storing large datasets.

# Reddit Systems Design: Comment Ranking and Architecture

**The YouTube video presents a systems design problem: designing a comment system similar to Reddit's.** The content creator explores various system design considerations, such as data storage options (graph databases vs. relational databases), indexing strategies for different comment sorting methods (new, top, controversial, hot), and the importance of causal and eventual consistency. **He analyzes trade-offs between read and write speeds**, emphasizing optimizing for frequent reads while keeping write latency reasonable. **The presenter outlines different strategies for managing data flow**, suggesting asynchronous processing with Kafka and Flink to maintain eventual consistency across multiple comment views. **The discussion includes practical considerations like capacity estimations** and caching to improve performance. **The video concludes with an overview of a proposed system architecture**, detailing components like a source-of-truth database, derived data stores, and stream processing for upvote aggregation.

Briefing Document: 
------------------

**Source:** Excerpts from "15: Reddit Comments | Systems Design Interview Questions With Ex-Google SWE" (YouTube transcript)

**Overall Theme:** The video explores the systems design considerations for building a Reddit-like comment system, focusing on how to efficiently handle different comment views (New, Hot, Top, Controversial) while maintaining consistency and optimizing for read-heavy workloads. The speaker emphasizes practical considerations and trade-offs involved in data storage, indexing, and asynchronous processing.

**Key Ideas and Facts:**

1.  **Problem Requirements & Comment Views:** The system needs to support multiple ways of viewing comments:

-   **New:** Comments ordered by the newest first, recursively within each comment thread.
-   **Hot:** Comments with the most upvotes within a recent timeframe (e.g., the last hour).
-   **Top:** Comments with the most upvotes in the entire history of the post.
-   **Controversial:** Comments with a high number of both upvotes and downvotes, resulting in a score near zero.

1.  *"Reddit is going to support a bunch of different ways of viewing the comments with on a within a given post so the first is going to be new which is basically like think of all of these um types of defining comment orders as hierarchical"*
2.  **Capacity Estimates & Partitioning:** The speaker performs rough calculations to estimate the storage requirements for comments.

-   Assumes a maximum of 1 million comments per post (rounds up from a known high of 500,000).
-   Assumes an average comment size of 500 bytes (including metadata like user ID, post ID, timestamp, upvote/downvote counts).
-   This leads to an estimate of 500 MB of data per post, which is considered manageable for a single partition. If replicated for all four views, that would be 2 GB.
-   Alternatively, storing only comment IDs (8 bytes each) for secondary indexes would require approximately 8 MB per post.

1.  *"I did some quick Googling and apparently there are 500,000 comments on the most popular Reddit thread ever but since we're not soft over here we're going to round up to a million and double that"*
2.  **Read vs. Write Optimization:** The speaker prioritizes read performance due to the read-heavy nature of Reddit. Writes should still be reasonably fast.
3.  *"Because it's Reddit uh the majority of people are probably lurking there are posts with a lot of comments but I would say that even more so than that most people are actually just reading and not commenting and so again if reads are going to happen much more frequently than writes we do want to be prioritizing the read path"*
4.  **Consistency Model:** The different comment views do *not* need to be perfectly synchronized. Eventual consistency is acceptable, but causal consistency is crucial.
5.  *"Even though we have four different comment views they don't actually have to be perfectly in sync as long as they're probably eventually consistent with one another"*
6.  *"When I say causal consistency it means that if you have one comment uh that is being displayed on your screen and it relies on some parent comment that the parent comment is also present right"*
7.  **Source of Truth Table:** A centralized data store is required as the single source of truth for all comments. This is non-negotiable, and must be the most reliable and consistent data store in the entire system.
8.  *"Regardless if we of whether we have four different copies of all the comments or whether we have one source of truth table and you know three different secondary indexes we are going to need basically one centralized data store where we can rely on the data"*
9.  **Comment Tree Structure:** Comments are organized as a tree (or graph), with nested replies. This structure influences the choice of database and indexing strategies.
10. *"The interesting thing with comments and Reddit are that they are organized as some sort of tree"*
11. **Graph Database vs. Relational Database:** The video explores two main approaches for storing the comment data:

-   **Graph Database (e.g., Neo4j):**Pros: Naturally handles graph data, uses physical pointers for relationships (edges). Good for arbitrary queries
-   Cons: Jumping from place to place on disk is slower. Can be harder to scale.
-   **Relational Database (with Smart Indexing):**Pros: Easier to scale, good data locality with smart indexes. Ideal for depth-first queries.
-   Cons: Edges are represented using indexes on table joins, which can slow down lookups. Not optimized for breadth-first.
-   **Smart Indexing Strategy:** Use a naming scheme based on parent-child relationships (e.g., parent ID "A", child IDs "AA", "AB", etc.). Sorting these IDs alphabetically creates an index suitable for depth-first search. This can provide really nice data locality.

1.  *"The nice thing about graph databases are that they're good at handling this data the reason being that you actually use physical pointers on your hard drive from node to node this is in contrast to something like a non-native graph database where we basically just use a typical relational database index"*
2.  *"If you definitely know that you're only going to be doing uh depth first type of queries the relational index that I built is going to help a lot if you want to be able to do generalizable queries well maybe it might not be so good"*
3.  **Indexing Strategies for Comment Views:**

-   **New:** Can be derived from the source-of-truth table itself, particularly if using the relational database approach with the hierarchical ID naming scheme.
-   **Top and Controversial:** Two options:
-   **Denormalized:** Create separate graph databases for each view, pre-sorted by upvotes/controversy. Requires updating the edges on every right. Do the extra work on write to make reads quicker.
-   **Normalized:** Keep an in-memory index of comment IDs, sorted by upvotes/controversy. Fetch the actual comment data from the source-of-truth database when needed. This requires a join, but saves disk space and reduces write overhead. Use a hash map in memory to quickly increment counts.

1.  *"If we wanted to actually use a typical index we could do that in absolute terms right so we could very easily build an index to show us the top comments across all comments but to do so at every single level of our graph right is not easy"*
2.  **Hot Comments:** Requires a separate component to track upvotes within a sliding time window.

-   Use a message queue (e.g., Kafka) to buffer incoming upvotes.
-   Use a stateful stream processor (e.g., Flink) to maintain a linked list of upvotes within the time window.
-   As upvotes expire, decrement the upvote count in the database.
-   The database for hot comments can be the same as for Top/Controversial (either a full graph copy or an in-memory index).

1.  *"The idea here is that hot comments have had a lot of upvotes in the last X minutes or maybe hour or something like that and so the idea is that we're going to have to deploy a separate component or a separate piece of functionality that is ultimately going to keep track of when an upvote you know it's been an hour and now that upvote is no longer hot"*
2.  **Data Flow & Consistency:**

-   Avoid synchronous writes to all four comment views.
-   Use asynchronous, derived data approach (writes go to one place, then propagated to others).
-   Maintain causal consistency: Ensure parent comments exist before child comments or upvotes are applied.
-   **Complex Causal Consistency Solution:**Comments are put in a source of truth database.
-   Comments then go into a Kafka queue sharded by post ID
-   Flink uses this comment queue to write data to top, controversial, and hot views.
-   FLink then puts the COMMENT ID and POST ID into a separate upvotes Kafka Queue, sharded by POST ID and COMMENT ID (more granular than just POST ID), to eventually be acknowledged by spark streaming.
-   Uvotes from users come in and are stored in a MySql database.
-   MySql database does Change Data Capture and puts data into Kafka.
-   Only after FLINK acknowledges to SPARK that a comment is present does SPARK aggregate the upvotes for that POST ID/COMMENT ID, and then it uploads them to the various secondary views.

1.  **Upvotes Database:** This stores upvotes and downvotes for every comment.

-   Schema: User ID, Post ID, Comment ID, Upvote/Downvote flag.
-   Sharded by User ID and Post ID for quick retrieval of user votes on a specific post.
-   Change data capture is used to propagate these upvotes to Spark for aggregation.
-   The CHANGE DATA CAPTURE Q should be sharded by POST ID and COMMENT ID so SPARK can easily maintain a total value.

1.  *"I would think that uh basically you would say okay here's our user field here's our post field here's our comment ID field and then here's whether it was an up Vote or a down vote so that's going to be our database schema"*
2.  **Caching:**

-   Cache the initial comments (e.g., first 10) for each view (New, Hot, Top, Controversial) to improve initial load times.
-   Use a read-through cache or write-around cache strategy.
-   Potential client-side optimization: Prefetch the next set of comments while the user is viewing the initial set.
-   Redis is ideal for comment caching. It should use the LRU strategy and should be sharded by POST ID.

1.  *"For a lot of hot posts or or just for posts in general uh you probably want to cash some initial comments right maybe the first 10 of the top view the new view the hot View and the controversial view"*

**In Summary:**

This video presents a detailed overview of designing a Reddit comment system, focusing on scalability, consistency, and efficient data retrieval. The speaker covers key design decisions including database selection, indexing strategies, caching mechanisms, and asynchronous processing, highlighting the trade-offs between different approaches and the importance of optimizing for read-heavy workloads.

Study Guide
===========

Quiz
----

Answer each question in 2-3 sentences.

1.  What are the four ways Reddit supports viewing comments?
2.  Why is it beneficial if all the comment views can be kept on the same database node?
3.  If choosing between optimizing for read or write speeds for Reddit comments, which should you prioritize and why?
4.  What is causal consistency in the context of Reddit comments?
5.  What are the two main options for building out the source of truth table for Reddit comments?
6.  What is one advantage of using a graph database for storing Reddit comments?
7.  What is one advantage of using a relational database for storing Reddit comments?
8.  What is the main challenge when building a "top comments" index for Reddit?
9.  How does the video suggest handling "hot comments"?
10. According to the video, what is the best way to ensure data makes it to different comment indexes or graph database replicas?

Quiz Answer Key
---------------

1.  The four ways Reddit supports viewing comments are **new** (ordered by the newest comments), **hot** (comments with the most upvotes in the last hour), **top** (comments with the most upvotes in the history of the post), and **controversial** (comments with a similar number of upvotes and downvotes).
2.  Keeping all the comment views on the same database node can benefit us because it allows us to **avoid a two-phase commit** if we were writing to all four of those at a time.
3.  You should prioritize **read speeds** because the majority of Reddit users are likely lurking, reading, and not commenting.
4.  Causal consistency means that **if a comment relies on a parent comment, the parent comment must also be present**. A child comment cannot exist without its parent comment being displayed.
5.  The two main options are using a **graph database** or a **normal (relational) database.**
6.  Graph databases can use **physical pointers** on your hard drive from node to node; as a result, this solution is **not slowed down** as the database gets bigger.
7.  With a relational database you can achieve **data locality** when doing depth first searches because everything that is actually a child is **right next to its parent** in the index.
8.  Building a top comments index is challenging because it requires sorting by upvotes at every single level of the comment tree, which is **difficult to achieve with a typical index**.
9.  Hot comments can be handled by using a **sliding window** approach with a stateful consumer like Flink to track upvotes within a certain time period.
10. It is better to ensure **derived data makes it to different comment indexes** using a **Kafka queue** that is partitioned by post ID.

Essay Questions
---------------

1.  Discuss the trade-offs between using a graph database and a relational database for storing and retrieving Reddit comments. Consider factors such as performance, scalability, and query flexibility.
2.  Explain the concept of causal consistency and its importance in a system like Reddit comments. Describe different approaches to ensuring causal consistency across multiple comment views, highlighting the advantages and disadvantages of each.
3.  Design a system for handling "hot comments" on Reddit. Describe the data structures, algorithms, and components involved in identifying and ranking hot comments in real time.
4.  Discuss different strategies for caching Reddit comments to improve read performance. Consider factors such as cache invalidation, cache eviction policies, and client-side optimizations.
5.  Evaluate the proposed architecture for the Reddit comments system, identifying potential bottlenecks and areas for improvement. Suggest alternative approaches or optimizations to enhance the scalability, reliability, and performance of the system.

Glossary of Key Terms
---------------------

-   **Causal Consistency:** A consistency model that ensures that if comment B relies on comment A, then comment A will always be visible before comment B.
-   **Change Data Capture (CDC):** A technique for capturing and propagating changes made to a database to other systems or data stores in near real-time.
-   **Depth-First Search (DFS):** An algorithm for traversing or searching tree or graph data structures.
-   **Breadth-First Search (BFS):** An algorithm for traversing or searching tree or graph data structures.
-   **Eventual Consistency:** A consistency model where data replicas will eventually converge to the same state, but there may be temporary inconsistencies.
-   **Flink:** A stream processing framework used for stateful computations and fault tolerance.
-   **Graph Database:** A database designed to store and query relationships between data entities.
-   **Kafka:** A distributed streaming platform used for building real-time data pipelines and streaming applications.
-   **Neo4j:** A popular graph database management system.
-   **Normalization:** A database design technique that reduces data redundancy by storing related data in separate tables.
-   **Denormalization:** A database design technique that increases data redundancy by storing related data in the same table to improve query performance.
-   **Relational Database:** A database based on the relational model, which organizes data into tables with rows and columns.
-   **Sliding Window:** A technique for analyzing data streams by focusing on a fixed-size window of data that slides over time.
-   **Source of Truth:** The authoritative data store for a particular piece of information.
-   **Spark Streaming:** An extension of the Apache Spark framework that enables real-time data processing.
-   **Two-Phase Commit (2PC):** A distributed transaction protocol that ensures atomicity across multiple databases.
-   **Write-Around Cache:** A caching strategy where writes are made directly to the backing store (database) and not to the cache. Reads are served from the cache if the data is present; otherwise, the data is fetched from the backing store and added to the cache.

# Zoom Systems Design: Video Conferencing Architecture

**The YouTube transcript outlines a systems design interview question focused on building a video conferencing application like Zoom.** **The speaker, an ex-Google SWE, walks through the technical considerations, beginning with networking basics like TCP and UDP.** **The transcript explores peer-to-peer communication limitations, before detailing a centralized chat server architecture using selective forwarding to optimize server load.** **Partitioning and replication strategies ensure scalability and high availability, while video recording is handled by separate servers to avoid overloading the main chat server.** **The design incorporates technologies like WebRTC and Kafka, providing a comprehensive overview of a complex system.**

Briefing Document: 
------------------

**Overview:**

This document summarizes the key considerations and architectural choices involved in designing a video conferencing system similar to Zoom or Skype, based on the provided transcript of a system design discussion by a former Google SWE. The discussion covers network protocols, peer-to-peer vs. centralized architectures, selective forwarding, and video recording strategies, partitioning, replication.

**Key Themes & Ideas:**

1.  **Problem Requirements:**

-   Support one-on-one and group video chats.
-   Support large calls (up to 100 people).
-   Record video calls on the backend and upload to the cloud.

1.  **Networking Refresher (TCP vs. UDP):**

-   **TCP:** Reliable, ordered delivery, re-sends dropped packets, three-way handshake, congestion and flow control.
-   **UDP:** Faster, doesn't guarantee delivery or order. Suitable for video/audio where occasional packet loss is acceptable. "as far as we are concerned with video chats we don't actually really care about getting every single package right...UDP is probably going to be the way to go here at least for the actual sending of the video and the audio itself for other stuff perhaps not"

1.  **Peer-to-Peer (P2P) vs. Centralized Architecture:**

-   **P2P:** Direct communication between clients. Ideal for one-on-one calls, potentially problematic for large groups due to increased load on client devices. Recording also becomes challenging in P2P setups. "if it's us two and your entire company now all of a sudden we're all sending one another packets and that can add a lot of load on our client devices...also what if we wanted to actually record this video footage well now all of a sudden basically whoever is doing that recording probably needs to be ingesting all of the video footage from all these different places right it could be up to 100"
-   **Centralized Server:** Clients send/receive video from a central server. Easier to manage for group calls and recording, but server load becomes a concern.

1.  **Network Address Translation (NAT) and STUN Servers:**

-   **NAT:** Masks private IP addresses behind a public IP address for security and to conserve IPv4 addresses.
-   **STUN (Session Traversal Utilities for NAT):** Helps clients behind NAT discover their public IP address and communicate directly (P2P). However, STUN may not always work depending on the NAT device.

1.  **Selective Forwarding (Key Concept):**

-   The centralized server only sends clients the video streams they care about, in the resolutions they require. This optimizes server load and provides a customizable experience. "option two allows us to have a customizable experience per user because basically what we're going to do now is only send clients the streams they actually care about and in the resolutions that they care about them"

1.  **Selective Forwarding Implementation:**

-   **Option 1 (Server Transcoding):** Client sends one stream, server converts it to multiple formats/resolutions. High CPU load on the server. Not ideal.
-   **Option 2 (Proxy Server):** Client sends one stream to a proxy server, which transcodes and sends to the central server. Adds latency.
-   **Option 3 (Client-Side Encoding - Preferred):** Client sends multiple streams (different resolutions) to the central server. Increases client-side processing but reduces server load. "the client is still going to do all the encoding but it's going to do it all locally so maybe instead of sending out one stream now we're going to send out three streams"

1.  **WebRTC:**

-   Mentioned as technology that abstracts away the underlying complexities of video streaming. However, in-depth knowledge of WebRTC is likely not required for a general systems design interview.

1.  **Partitioning and Replication:**

-   **Partitioning:** Shard the chat servers by chat ID using consistent hashing to distribute load evenly.
-   **Replication:** Use an active-passive configuration with a Zookeeper cluster for failover. The passive backup server doesn't need to maintain significant state because the video streams are stateless. "we can use centralized servers in kind of like a active passive configuration...we have some passive backup that just sits there doing nothing we've got a zookeeper cluster that's constantly getting heartbeats from our main node"

1.  **Video Recording:**

-   Don't perform recording directly on the central chat server to avoid overloading it.
-   Use separate recording servers that subscribe to the video call.
-   For high-definition recording of all user streams, distribute the recording workload across multiple recording servers. Each server captures a subset of streams.
-   Store the video footage in Kafka (partitioned by chat ID) with timestamps for proper alignment. A stateful stream consumer then combines the footage and syncs it to S3. "we can have other servers that subscribe to the video call and also perform the encoding that we need to actually make this recording and put it in S3"

1.  **End-to-End Architecture:**

-   The client goes to a load balancer using a chat ID.
-   The load balancer uses a consistent hashing policy to map the client to a chat server.
-   The client establishes a websocket connection (for selective forwarding) and subscribes to a UDP feed.
-   The client sends multiple variants of their stream over UDP to the chat server.
-   Recording servers also connect and stream the video data into Kafka for processing and archival.

**Quotes that Highlight Key Points:**

-   *On UDP's suitability for video:* "...as far as we are concerned with video chats we don't actually really care about getting every single package right...UDP is probably going to be the way to go here at least for the actual sending of the video and the audio itself for other stuff perhaps not"
-   *On the downside of P2P for group calls:* "if it's us two and your entire company now all of a sudden we're all sending one another packets and that can add a lot of load on our client devices...also what if we wanted to actually record this video footage well now all of a sudden basically whoever is doing that recording probably needs to be ingesting all of the video footage from all these different places right it could be up to 100"
-   *On the benefits of Selective Forwarding:* "option two allows us to have a customizable experience per user because basically what we're going to do now is only send clients the streams they actually care about and in the resolutions that they care about them"
-   *On the client doing the encoding in a selective forwarding system:* "the client is still going to do all the encoding but it's going to do it all locally so maybe instead of sending out one stream now we're going to send out three streams"
-   *On the server replication strategy:* "we can use centralized servers in kind of like a active passive configuration...we have some passive backup that just sits there doing nothing we've got a zookeeper cluster that's constantly getting heartbeats from our main node"
-   *On using separate video recording servers:* "we can have other servers that subscribe to the video call and also perform the encoding that we need to actually make this recording and put it in S3"

**Conclusion:**

Designing a video conferencing system involves carefully balancing trade-offs between client-side and server-side processing, network protocols, and different architectural patterns. Selective forwarding is a crucial optimization for handling large group calls. The system also needs to be designed for scalability, fault tolerance, and efficient video recording.

Study Guide
===========

Quiz
----

**Answer each question in 2-3 sentences.**

1.  What are the key differences between TCP and UDP, and why is UDP preferred for video and audio transmission in Zoom-like applications?
2.  Explain the concept of Network Address Translation (NAT) and its purpose in modern networking.
3.  What is a STUN server, and how does it facilitate peer-to-peer communication behind NAT devices?
4.  Describe the main benefits and drawbacks of using a centralized chat server for video conferencing.
5.  What is selective forwarding, and how does it improve the efficiency of a centralized server in a video conferencing application?
6.  Explain the three options presented for how a client sends data to a centralized server in a video conferencing application, and describe why the third option is the most common implementation.
7.  Why is partitioning and replication essential for a centralized server in a video conferencing application?
8.  Explain how chat IDs can be used to shard a video call across multiple servers and why consistent hashing is beneficial.
9.  Describe the active-passive configuration used for centralized servers and how Zookeeper facilitates failover.
10. Why should video recording not be performed on the central chat server, and what is the alternative approach?

Quiz Answer Key
---------------

1.  TCP provides reliable, ordered delivery with error checking and flow control, while UDP is connectionless and faster but unreliable. UDP is preferred for video/audio because occasional packet loss is tolerable and speed is prioritized over guaranteed delivery.
2.  NAT allows multiple devices on a private network to share a single public IP address. It translates between private and public IP addresses, enhancing security and conserving IPv4 addresses.
3.  A STUN (Session Traversal Utilities for NAT) server helps clients behind NAT discover their public IP address and port. This information is then used to establish direct peer-to-peer connections.
4.  A centralized server simplifies video and audio routing, reducing client-side processing and battery drain. However, it introduces a single point of failure and can become a performance bottleneck under high load.
5.  Selective forwarding allows the server to send only the necessary video streams and resolutions to each client based on their individual view, optimizing bandwidth and reducing server load.
6.  One is for the client to send a single stream and the server converts it, the second is for the client to send one stream through an intermediary server, and the third is that the client does all of the encoding locally and sends three streams to the server. The third is the most common because although the client does more work, it does not overload the central server.
7.  Partitioning distributes load across multiple servers to handle a large number of concurrent video calls. Replication ensures high availability by providing backup servers that can take over in case of a primary server failure.
8.  Chat IDs are used to shard video calls across multiple servers, ensuring each call is handled by a specific server. Consistent hashing ensures that servers receive a relatively even distribution of calls, preventing overloading.
9.  In an active-passive configuration, a passive backup server monitors the primary server's health. Zookeeper detects failures through heartbeats and triggers the passive server to become the new primary, ensuring continuous service.
10. Video recording is intensive and can overload the central chat server. Instead, dedicated recording servers should subscribe to the video call and perform the encoding, uploading the recordings to cloud storage.

Essay Questions
---------------

1.  Discuss the trade-offs between peer-to-peer and centralized server architectures for video conferencing applications, considering scalability, latency, and resource utilization.
2.  Explain how selective forwarding can be implemented in a video conferencing system, and analyze its impact on server load, network bandwidth, and user experience.
3.  Describe the steps involved in setting up a highly available and scalable centralized server architecture for a video conferencing application, including partitioning, replication, and failover mechanisms.
4.  Analyze the different options for video encoding and distribution in a video conferencing system, considering client-side processing, server-side processing, and overall network performance.
5.  Discuss the challenges and solutions involved in recording video conferences at scale, including strategies for distributing the workload, managing storage, and ensuring data integrity.

Glossary of Key Terms
---------------------

-   **TCP (Transmission Control Protocol):** A connection-oriented protocol that provides reliable, ordered, and error-checked delivery of data.
-   **UDP (User Datagram Protocol):** A connectionless protocol that provides a fast but unreliable way to transmit data.
-   **NAT (Network Address Translation):** A technique that allows multiple devices on a private network to share a single public IP address.
-   **STUN (Session Traversal Utilities for NAT):** A protocol used to discover the public IP address and port of a device behind NAT.
-   **Peer-to-Peer Communication:** Direct communication between two devices without an intermediary server.
-   **Centralized Server:** A server that acts as an intermediary, routing data between clients.
-   **Selective Forwarding:** A technique where a centralized server sends only the necessary video streams and resolutions to each client.
-   **WebRTC (Web Real-Time Communication):** A free, open-source project providing real-time communication capabilities for web browsers and mobile applications.
-   **Partitioning (Sharding):** Dividing a large dataset or system into smaller, more manageable parts.
-   **Replication:** Creating multiple copies of data or system components to ensure high availability and fault tolerance.
-   **Consistent Hashing:** A hashing technique that minimizes the impact of adding or removing servers on the distribution of data.
-   **Zookeeper:** A centralized service for maintaining configuration information, naming, providing distributed synchronization, and group services.
-   **Active-Passive Configuration:** A high-availability setup where a passive backup server monitors the primary server and takes over in case of failure.
-   **Kafka:** A distributed streaming platform used for building real-time data pipelines and streaming applications.
-   **Stateful Stream Consumer:** A stream processing application that maintains state and processes events in order.
-   **S3 (Simple Storage Service):** A scalable, high-speed, web-based cloud storage service offered by Amazon Web Services (AWS).
-   **Stream Ingress Server:** A server responsible for receiving and processing incoming video streams.

# Designing a Scalable Bidding Platform Like eBay

**The YouTube transcript presents a systems design interview question focused on creating a bidding platform similar to eBay but with increased scalability.** **The content creator, Jordan, outlines the requirements, data models, and potential challenges, emphasizing the need for synchronous bid processing and high throughput.** **He explores the use of Kafka for ordering bids and a bid engine for validation, addressing fault tolerance through state machine replication.** **The discussion extends to real-time price updates for users and a detailed examination of potential race conditions and solutions.** **Jordan details the different options for how to handle concurrency using pseudocode with the goals of high performance and minimizing lost bids.** **The video concludes with a system-wide summary of the complete bidding platform architecture, including data flow and component responsibilities.**

Briefing Document: 
------------------

**Overview:**

This document summarizes a systems design interview scenario focused on building a high-scale bidding platform. The scenario emphasizes the challenges of handling a large volume of bids per second, especially for popular items, and maintaining data consistency and fault tolerance while providing a synchronous bidding experience for users. The design is heavily influenced by concepts from a previous video on building a stock exchange.

**Key Requirements:**

-   **User Bidding:** Users must be able to bid on items in active auctions.
-   **Concurrency:** The platform must support many active auctions simultaneously.
-   **Winning Bid:** The highest bid at the end of the auction period wins.
-   **Auction End Times:** Auctions can have fixed or variable end times (variable end times extend the auction if a bid is placed near the original end time).
-   **Real-time Updates:** Users with the auction page open receive real-time price updates.
-   **High Throughput:** The system should handle a high number of bids per second, even for popular items. The presenter expresses doubt that eBay realistically handles this level of throughput, stating "...I don't think that any of them are getting like a 100 bids within a second and uh especially getting that type of throughput the entire time."
-   **Synchronous Bidding:** A crucial and challenging requirement is that users receive immediate confirmation (synchronously) of whether their bid was accepted or rejected. This constraint significantly impacts the architecture. The presenter notes "when a user submits the bid um they will know synchronously whether it was accepted or rejected now if I relax this uh premise then this becomes a much easier problem".

**Data Models:**

-   **Bid:**ID: Unique identifier for the bid.
-   Auction ID: The auction to which the bid belongs.
-   User ID: The user who submitted the bid.
-   Price: The bid amount.
-   Server-side Timestamp: Timestamp generated by the server. This is important to prevent malicious clients from manipulating bid order.
-   Status: Accepted (currently winning bid) or Rejected (too low).
-   Important to keep all bids for auditing purposes, storing them in a time series database.
-   **Auction State:**Auction ID
-   Current Bid ID
-   Price
-   End Time
-   The Auction State is small (32 bytes), which enables in-memory storage to support fast bidding logic.

**Architecture & Components:**

1.  **Bid Service:** Entry point for user bids. Routes requests to the Bid Engine.
2.  **Bid Engine (Primary):** The central component responsible for bid validation, order determination, and state management.

-   Maintains the Auction State in memory for speed.
-   Uses locking to handle concurrent bids and ensure data consistency. The presenter specifies "as every single request comes in we're just going to have that request lock grab a lock on the auction state check if the bid is valid".
-   Partitions auctions across multiple Bid Engine instances using consistent hashing on the Auction ID.
-   Publishes bids to Kafka.

1.  **Bid Engine (Backup):** Provides fault tolerance.

-   Uses **State Machine Replication**: The backup listens to the *output* (ordered bids) from the primary bid engine rather than accepting bids directly from users.
-   Connected to Zookeeper to facilitate failover in case the primary goes down.

1.  **Kafka:** Serves as the central message broker for distributing bid information. The presenter says "Kafka is actually going to be the source of truth of our system".

-   Provides fault tolerance and ensures that all interested parties (audit logs, price update services) eventually receive all bid information.
-   Different replication topologies can be used, depending on the requirements for fault tolerance and write speed.

1.  **Stream Consumers:** Process bid data from Kafka.

-   **Audit Log (Time Series Database):** Stores all bid history for auditing and dispute resolution.
-   **Auction State Queue (Filtered):** A second Kafka queue containing *only* the accepted (price-increasing) bids. This reduces the amount of data that needs to be processed by the real-time price update services.

1.  **Auction State Servers:** Maintain the current price for auctions.

-   Listen to the Auction State Queue.
-   Provide real-time price updates to clients via Server-Sent Events (SSE) or WebSockets.

1.  **Load Balancers:** Distribute traffic to the Bid Gateways and Auction State Servers.

**Key Design Decisions and Rationale:**

-   **Synchronous vs. Asynchronous Processing:** The presenter emphasizes the trade-offs between synchronous and asynchronous bid processing. While Kafka offers high throughput and fault tolerance, it introduces latency. The decision to provide synchronous feedback to the user requires a more complex architecture. The presenter explains that the complication that would be caused due to using a synchronous endpoint would lead to the length of the video.
-   **Kafka as Source of Truth:** Kafka is used as a source of truth to ensure bids are not lost in the event of a Bid Engine failure. The bid is persisted to Kafka before the user is notified of acceptance.
-   **Bid Engine Throughput:** The Bid Engine is designed for high throughput by:
-   Keeping the Auction State in memory.
-   Minimizing the amount of work done within the critical section (locking).
-   Offloading tasks like publishing to Kafka to separate threads or processes.
-   **Ordering Guarantees:** Ensuring that bids are processed in the correct order is crucial for fairness. The Bid Engine uses locking to establish a total order. To address the risk of out-of-order messages in Kafka (due to asynchronous publishing), a queue is used in the Bid Engine to publish bid messages to Kafka in the correct order. The presenter describes "what we would want to do is ensure that when we publish to Kafka we're publishing the bids in their proper order".
-   **Fault Tolerance:** Achieved through:
-   State machine replication of the bid engine.
-   Kafka replication.
-   Change Data Capture (CDC) for derived data.

**Challenges and Considerations:**

-   **Race Conditions:** The design must address race conditions that can occur when multiple users bid on the same item concurrently. The presenter says "when we have a bidding system like this you can have two people who without knowing about one another's bids meaning their concurrent rights uh will actually submit a bid for the same price".
-   **Kafka Latency:** Minimizing latency when publishing bids to Kafka is important to maintain a responsive user experience.
-   **Scalability:** The architecture must be scalable to handle a large number of auctions and users.
-   **Complex Endpoints and Data Handling:** Many components are dependent on each other, so all potential issues need to be taken into account.

**Simplified Pseudo Code for Bid Engine:**

The presenter outlines two possible approaches to writing the pseudo code for the Bid Engine. The first approach focused on faster bids, and the second approach focused on Kafka order being reflected by the legitimate order processed by the bidding engine.

**Fast Bids:**

-   Get bid on engine with action ID, bid, and price
-   Lock auction state
-   get the next sequence number of this bid.
-   check if bid is accepted
-   if bid is accepted, update auction state
-   upload bid to Kafka
-   Return accepted or rejected to user

**Reflected Kafka Order:**

-   register a call back function on when we eventually upload to kfka
-   Lock auction state
-   get the next sequence number of this bid.
-   check if bid is accepted
-   if bid is accepted, update auction state
-   call current concurrent Q to Kafka

**Auction Ending Mechanism:**

-   Update auction end time based on bid time stamps.
-   Bid engine will occasionally check the current time versus the end auction time
-   Write winning bid ID to auction database

**Conclusion:**

This design aims to create a scalable and fault-tolerant bidding platform while providing a synchronous experience for users. The key challenges lie in managing concurrency, maintaining data consistency, and minimizing latency. The architecture leverages in-memory caching, Kafka, and state machine replication to achieve these goals. The presenter notes "I understand it's probably not the most practical one and I imagine that in practice someone like eBay is probably just going server database establish a lock" while acknowledging that the presented system was a cool system.

Study Guide
===========

Quiz: Short Answer Questions
----------------------------

1.  **What is the key difference in bidding rules between this proposed system and eBay, and why does this difference matter for the design?** This system requires users to know synchronously whether their bid was accepted or rejected, unlike eBay's maximum bid system. This synchronous requirement introduces significant complexity and demands a totally ordered system.
2.  **Why is it important to store all bids for an auction, even the rejected ones?** Storing all bids is crucial for auditing purposes, especially in case of disputes or investigations. It also allows for the possibility of awarding the item to the second-highest bidder if the initial winner defaults on payment.
3.  **Why is Kafka chosen as a central component in this bidding platform design?** Kafka provides high write throughput and ensures a total ordering of events, which is essential for determining the correct order of bids and preventing race conditions. Its fault tolerance also ensures bids are not lost.
4.  **What is the function of the "bid engine" in this architecture?** The bid engine acts as the central processing unit for all incoming bids. It stores the auction state, validates bids, and determines whether a bid is accepted or rejected.
5.  **Why is the bidding engine designed to be in-memory, and what are the advantages of this approach?** Keeping the bidding engine in memory minimizes latency and maximizes throughput by avoiding disk reads in the critical path of bid validation. The auction state is small, making this feasible.
6.  **How does the system achieve fault tolerance for the bid engine, considering it's primarily an in-memory component?** Fault tolerance is achieved through state machine replication with a backup bidding engine. The backup listens to the output from the primary, ensuring it stays synchronized.
7.  **Why is publishing to Kafka done *before* informing the user if their bid was accepted?** Publishing to Kafka first ensures that the bid is persisted and fault-tolerant before the user is notified. This avoids the situation where a user thinks their bid is successful, but it's lost if the bid engine fails before the bid is persisted.
8.  **Describe one optimization discussed for maintaining high bid engine throughput while also publishing to Kafka.** Using a concurrent queue to buffer bids and upload them to Kafka in the background allows the bid engine to continue processing new bids without waiting for the Kafka write to complete.
9.  **What is the purpose of the "auction state queue" mentioned in the design?** The auction state queue contains only the valid bids that actually increased the price. This reduces the amount of data that user-facing servers need to process, improving their performance and scalability.
10. **How does the system handle variable end-time auctions, where the auction end time is extended based on recent bids?** When a bid is accepted, the bid engine updates the finishing time based on the timestamp of the bid. The bid engine checks the time periodically and writes to an auction database with the winning bid ID once the end time is passed.

Quiz Answer Key
---------------

1.  The system requires users to know synchronously whether their bid was accepted or rejected, unlike eBay's maximum bid system. This synchronous requirement introduces significant complexity and demands a totally ordered system.
2.  Storing all bids is crucial for auditing purposes, especially in case of disputes or investigations. It also allows for the possibility of awarding the item to the second-highest bidder if the initial winner defaults on payment.
3.  Kafka provides high write throughput and ensures a total ordering of events, which is essential for determining the correct order of bids and preventing race conditions. Its fault tolerance also ensures bids are not lost.
4.  The bid engine acts as the central processing unit for all incoming bids. It stores the auction state, validates bids, and determines whether a bid is accepted or rejected.
5.  Keeping the bidding engine in memory minimizes latency and maximizes throughput by avoiding disk reads in the critical path of bid validation. The auction state is small, making this feasible.
6.  Fault tolerance is achieved through state machine replication with a backup bidding engine. The backup listens to the output from the primary, ensuring it stays synchronized.
7.  Publishing to Kafka first ensures that the bid is persisted and fault-tolerant before the user is notified. This avoids the situation where a user thinks their bid is successful, but it's lost if the bid engine fails before the bid is persisted.
8.  Using a concurrent queue to buffer bids and upload them to Kafka in the background allows the bid engine to continue processing new bids without waiting for the Kafka write to complete.
9.  The auction state queue contains only the valid bids that actually increased the price. This reduces the amount of data that user-facing servers need to process, improving their performance and scalability.
10. When a bid is accepted, the bid engine updates the finishing time based on the timestamp of the bid. The bid engine checks the time periodically and writes to an auction database with the winning bid ID once the end time is passed.

Essay Questions
---------------

1.  Discuss the trade-offs involved in choosing between synchronous and asynchronous bid confirmation, and how the chosen approach impacts the overall architecture of the bidding platform.
2.  Explain the role of Kafka in ensuring consistency and fault tolerance in the bidding platform, and evaluate alternative approaches to achieve these goals.
3.  Analyze the scalability challenges presented by popular auctions, and how the proposed architecture addresses these challenges. What other optimizations could be implemented?
4.  Evaluate the proposed system's approach to data storage and retrieval, considering the requirements for auditing, real-time updates, and derived data.
5.  Discuss the potential security considerations in this system design, including bid manipulation and denial-of-service attacks, and propose mitigation strategies.

Glossary of Key Terms
---------------------

-   **Bid Engine:** The core component responsible for processing bids, validating them, and determining the winner of an auction.
-   **Auction State:** The current state of an auction, including the current bid ID, price, and end time.
-   **Kafka:** A distributed streaming platform used for handling real-time data feeds and providing fault tolerance and ordering guarantees.
-   **State Machine Replication:** A technique for achieving fault tolerance by replicating the state of a system across multiple nodes and ensuring that they all execute the same operations in the same order.
-   **Synchronous Endpoint:** A communication pattern where the client waits for a response from the server before proceeding.
-   **Asynchronous Endpoint:** A communication pattern where the client sends a request to the server and doesn't wait for an immediate response.
-   **Time Series Database:** A database optimized for storing and retrieving data that is indexed by time, often used for auditing and historical analysis.
-   **Change Data Capture (CDC):** A technique for identifying and tracking changes to data in a database, allowing for the propagation of these changes to other systems.
-   **Bid Gateway:** A server that routes bid requests to the appropriate bid engine.
-   **Auction State Queue:** A filtered queue containing only valid bids that increased the price, used to reduce the load on user-facing servers providing real-time updates.

# LinkedIn Mutual Connection Search System Design

**The YouTube transcript describes the systems design of a LinkedIn mutual connection search feature.** **It begins by outlining the problem, estimating capacity, and positing a graph database solution.** **The speaker then rejects this solution due to performance and partitioning concerns.** **As an alternative, the transcript presents caching all mutual connections for each user with denormalized profile data in a SQL database.** **The speaker details how new connections and profile updates are handled asynchronously using Kafka, Flink, and batch jobs for eventual consistency.** **Ultimately, this design prioritizes fast read times for search queries by pre-organizing and caching data.**

Briefing Document: 
------------------

**Source:** Excerpts from "30: LinkedIn Mutual Connection Search | Systems Design Interview Questions With Ex-Google SWE"

**Main Theme:** The video discusses the system design challenges and solutions for implementing a mutual connection search feature on a platform like LinkedIn, focusing on scalability and efficiency. It explores the trade-offs between different database approaches, caching strategies, and update mechanisms. The conclusion is the best implementation would be a batch update system, instead of continuous streaming.

**Key Ideas and Facts:**

-   **Problem Definition:** The goal is to enable users to search for connections that are mutually connected with them, based on criteria like education or current employer. It explicitly excludes full-text search for simplicity, focusing instead on searching within pre-defined fields.
-   "Basically the gist is uh we want to be able to search our LinkedIn for connections where uh basically it's going to be specifically a mutual connection right so if this is me I have some sort of connection I'm going to Emmit connection search in this problem because that's you know go to a database and search it based on all my connections uh the mutual connection is one where I'm not actually connected to them but we share a connection in common"
-   **Capacity Estimates:**
-   Assumes 1 billion users.
-   Assumes an average of 500 connections per user.
-   This leads to an estimated 250,000 mutual connections per user (500 * 500, with some overlap accounted for).
-   Assumes a search preview requires approximately 1 KB of data.
-   Assumes each user updates their education or employment information once per year, resulting in approximately 3 million updates per day.
-   "I'm going to assume we've got a billion users on the site that's pretty big...I also think that on average having each user with 500 connections is pretty reasonable...each of your 500 connections also has 500 connections"
-   **Initial Graph Database Approach (and its limitations):**
-   Initially considers a graph database approach, viewing the problem as finding nodes at a distance of two connections.
-   Discusses native vs. non-native graph databases. Non-native graph databases have slow performance due to the binary search needed for traversing connections: O(log n). Native graph databases are better, since they allow you to directly point to the physical location on disk of a user's connections.
-   **Problem:** Highlights the challenges of partitioning the LinkedIn social graph, which is highly interconnected. Partitioning is needed for scalability, but the interconnected nature makes it difficult to isolate users within single partitions.
-   "in reality the LinkedIn social graph is probably highly connected right in reality we can see that it's very intertwined over here and there's not really any good way to split it up such that if I want to find the mutual connections for for this guy I can only read one partition"
-   **Cached Results (Denormalized Data):**
-   Proposes caching pre-organized results for every user to optimize read speeds.
-   Acknowledges this requires significant storage: 250,000 mutual connections * 1KB/connection * 1 billion users = 250 Petabytes of data.
-   Suggests sharding by user ID to keep all mutual connection data for a user on a single database node.
-   Emphasizes denormalization of profile data (name, school, company) within the mutual connection cache to avoid expensive distributed joins during read queries.
-   "We want to keep our reads as fast as possible and the only way to basically do that is to pre-organize for every single user all of their Mutual connections...we can also do this as well is by denormalizing the profile data per Mutual connection"
-   **Mutual Cache Table Schema:**
-   User ID (indexed).
-   Mutual Connection IDs.
-   Denormalized data for each mutual connection (name, school, company).
-   Suggests secondary local indices on name, school, and company for faster filtering.
-   Recommends a SQL database for this cache due to the focus on read performance and the benefits of a B+ tree index.
-   **Adding New Connections (Kafka, Flink, Fan-out):**
-   Uses Kafka to handle new connection events asynchronously and ensure fault tolerance.
-   Employs a stateless consumer to split the connection event into two messages, one for each user involved in the connection.
-   Utilizes Flink to maintain a real-time mapping of users to their connections.
-   When a new connection occurs, Flink determines the new mutual connections and updates the mutual connection database for all affected users using a fan-out approach, writing to multiple partitions. The process requires O(m+n) writes, where m and n are the number of connections of the two newly connected users (on the order of 1000s of writes).
-   "when a client makes a new connection the first thing that we can do is put that connection in Kafka the reason is we just want to put it in there once so we don't have to do a two-phase commit we then have a stateless consumer right...all it's going to do is see a message like 10 to 15 send it into the Q for both 10 and send it into the Q for both 15"
-   **Profile Updates (Batch Processing):**
-   Recognizes that Fanning-out profile updates to all mutual connections would be too expensive (potentially 25,000 writes per update).
-   Instead, proposes batching profile updates daily.
-   Uses Change Data Capture (CDC) to sync profile updates from the main profile database to Kafka.
-   Caches the updates in memory (approximately 3GB per day, which is manageable).
-   Runs a daily batch job to apply these updates to all mutual connection databases. This job can use stored procedures or scripts.
-   The database rows do not need to be locked.
-   "when someone updates their profile they basically have a bunch of mutual connections right...it's probably going to be too expensive to do a fan out of all of our profile updates to all three you know not just all three but all partitions of these Mutual connection databases over on our left so what can we do instead?"
-   **Final Architecture:**
-   The system consists of a client, Kafka, a stateless consumer, Flink, a mutual connection database, a profile edit service, a profile database, change data capture to Kafka, profile edit caches, and a daily batch job.

**Overall Recommendation:**

The briefing favors a system that prioritizes read performance through caching and denormalization, while managing write load through asynchronous processing and batch updates. This provides good query performance with eventual consistency for profile updates. The streaming process of new connections makes it possible to display recent additions, while updates can be done via batch.

Study Guide
===========

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  What is the primary goal of the LinkedIn mutual connection search system described in the source?
2.  Why is a full-text search omitted from the design, according to the source?
3.  What are the key capacity estimates assumed in the design, and why are they important?
4.  Explain why treating this problem as a graph problem using a graph database might not be feasible.
5.  Why is caching the results for all users considered as an alternative to a graph database approach?
6.  What are the advantages of sharding the mutual connection cache by user ID?
7.  Explain the denormalization strategy employed in the mutual cache table and why it is used.
8.  Describe the process of adding a new connection and how Kafka and Flink are utilized.
9.  Why is it necessary to update mutual connections databases when a user updates their profile information?
10. Explain the batch processing approach used to handle profile updates and why it is more efficient than fanning out updates in real-time.

Quiz Answer Key
---------------

1.  The primary goal is to allow users to search their LinkedIn network for mutual connections based on criteria like education or current employer. The system aims to efficiently retrieve these mutual connections, which are defined as people who are not directly connected to the user but share a connection in common.
2.  Full-text search is omitted to simplify the design. Implementing it would involve putting everything in a search index, which is a known process, but the focus is on optimizing the mutual connection search based on hardcoded fields like education or current employer.
3.  The key estimates are 1 billion users, 500 connections per user on average, and 1 kilobyte of data per search preview. These estimates are important for capacity planning, especially when considering the amount of data that needs to be stored and processed for mutual connections.
4.  Treating it as a graph problem and using graph databases may be infeasible because of issues related to random seeks on disk and partitioning. The LinkedIn social graph is highly interconnected, making it difficult to partition effectively and leading to slower query performance.
5.  Caching is considered because it offers the potential to make reads as fast as possible by pre-organizing all mutual connections for every user. This is done to minimize latency during the search process.
6.  Sharding by user ID ensures that all mutual connections for a given user are located on a single database partition. This minimizes the need to read from multiple partitions when retrieving a user's mutual connections, thereby improving read performance.
7.  The denormalization strategy involves storing profile data (name, school, company) directly within the mutual connection cache. This is used to avoid the need for large distributed joins and scatter-gather queries, which would be too expensive and slow down the search process.
8.  When a new connection is added, the event is placed in Kafka, then a stateless consumer splits the event into two messages for each user involved. Flink maintains a mapping of users to connections and updates the mutual connection databases accordingly, all done asynchronously.
9.  Updating mutual connections databases is necessary because profile information is denormalized. Therefore, if a user changes their profile details, those changes need to be reflected in the records of all their mutual connections to ensure accurate search results.
10. The batch processing approach involves caching profile updates in memory and then applying them to the mutual connection databases in a daily batch. This is more efficient because it reduces the number of writes required, avoiding the overhead of fanning out each update in real-time to every mutual connection's record.

Essay Questions
---------------

1.  Discuss the trade-offs between using a graph database approach versus a caching approach for implementing the LinkedIn mutual connection search. In what scenarios might each approach be more suitable?
2.  Explain the role of Kafka and Flink in managing and processing new connections. How do these technologies contribute to the system's fault tolerance and eventual consistency?
3.  Evaluate the denormalization strategy used in the mutual connection cache. What are the benefits and drawbacks of this approach, and are there any potential alternatives?
4.  Describe the challenges of handling profile updates in a denormalized system. How does the proposed batch processing approach address these challenges, and what are its limitations?
5.  Analyze the overall system architecture for the LinkedIn mutual connection search, including the key components, their interactions, and the design decisions made. How well does this architecture meet the requirements of scalability, performance, and consistency?

Glossary of Key Terms
---------------------

-   **Mutual Connection:** A user who is not directly connected to a given user but shares a connection in common with them.
-   **Depth-First Search (DFS):** An algorithm for traversing or searching tree or graph data structures. The algorithm starts at the root node (selecting some arbitrary node as the root node in the case of a graph) and explores as far as possible along each branch before backtracking.
-   **Graph Database:** A database that uses graph structures with nodes, edges, and properties to represent and store data.
-   **Native Graph Database:** A graph database designed to Traverse edges in O(1) time.
-   **Non-Native Graph Database:** A graph database that stores edges as a SQL table.
-   **Partitioning:** Dividing a database or dataset into smaller, more manageable pieces that can be stored and processed independently.
-   **Sharding:** A type of database partitioning that separates very large database tables into smaller, faster, more easily managed parts called data shards.
-   **Two-Phase Commit:** A distributed transaction protocol that ensures all participants in a distributed transaction either commit or roll back the transaction.
-   **Caching:** Storing frequently accessed data in a fast-access memory location to reduce latency and improve performance.
-   **Denormalization:** Adding redundant data to a database to improve read performance.
-   **Load Balancer:** A device or software that distributes network traffic across multiple servers to prevent overload and ensure high availability.
-   **Kafka:** A distributed streaming platform used for building real-time data pipelines and streaming applications.
-   **Flink:** An open-source, distributed stream processing framework for data analytics.
-   **Change Data Capture (CDC):** A set of software design patterns used to determine and track data that has changed in a database.
-   **Eventual Consistency:** A consistency model used in distributed computing to achieve high availability. It guarantees that, if no new updates are made to a given data item, eventually all accesses to that item will return the last updated value.
-   **Batch Job:** A program that processes a large volume of data in a single, non-interactive operation.
-   **Cron Job:** A time-based job scheduler in Unix-like operating systems that allows users to schedule tasks to run automatically at specific intervals.
-   **Hash Join:** A join algorithm used in database systems to efficiently join two tables based on a common key.
-   **Stateless Consumer:** A consumer that does not keep track of state.
-   **Consistent Hashing:** A special kind of hashing such that when a hash table is resized, only K/n keys need to be remapped on average, where K is the number of keys, and n is the number of slots.

# Distributed Priority Queue System Design

**The YouTube video explains how to design a distributed priority queue, a system used to manage asynchronous tasks with varying levels of importance.** The video is based on a Facebook paper and aims to create a scalable and fault-tolerant system for processing jobs in order of priority. **The design involves using a hybrid approach that combines an in-memory sorted index (like a heap) with data stored on disk (similar to a write-ahead log).** Replication strategies, partitioning techniques (primarily round robin to avoid hot spots), and consumer strategies (including long polling) are discussed to optimize throughput and fault tolerance. **The ultimate design includes a load balancer, a MySQL database for storing the queue, and a consensus algorithm (like Paxos or Raft) for replication within each partition.** Consumers then retrieve and process tasks based on priority, potentially managing multiple partitions.

Briefing Document:
------------------

This document summarizes the key concepts and design considerations for a distributed priority queue system, based on excerpts from a video discussing the topic. The video heavily references a Facebook (Meta) paper and design, emphasizing scalability, fault tolerance, and efficient processing of asynchronous jobs with assigned priorities.

**Main Themes:**

-   **Asynchronous Job Processing with Priority:** The core requirement is to build a system that can manage and process tasks (e.g., video encoding) asynchronously, assigning each task a priority level. This allows for prioritization of critical tasks over less urgent ones. "Basically our distributed priority queue is going to allow us to build out a scalable and fault tolerance system with fairly good read and write through putut that's going to allow all of these separate video encoders to basically pick off all of these items and make sure that they all get processed at least once."
-   **Scalability and Fault Tolerance:** The system must be designed to handle massive scale (billions of messages) and remain operational even in the face of failures. This dictates the need for distributed architecture and replication strategies.
-   **Approximate Priority Ordering:** The design prioritizes performance (throughput) over strict priority ordering. Due to the use of multiple consumers, perfect ordering is not guaranteed, but the system aims for "approximate priority que."
-   **Hybrid In-Memory/On-Disk Approach:** To balance speed and storage capacity, the design employs a hybrid approach. A sorted index of priorities (and pointers to data) is maintained in memory for fast retrieval, while the actual task data is stored on disk (e.g., in a write-ahead log).
-   **Replication and Consensus:** To ensure fault tolerance, the system utilizes replication and a consensus algorithm (like Paxos or Raft) to guarantee that messages are durably stored and will eventually be processed.
-   **Partitioning for Throughput:** Data is partitioned across multiple nodes to increase read and write throughput. Round-robin partitioning is favored over partitioning by priority range to avoid hot spots on specific nodes.

**Key Ideas and Facts:**

-   **Problem Requirements:**Ordered list of items with assigned priorities.
-   Support for Enqueuing, Modifying, and Dequeuing items.
-   Guaranteed at-least-once delivery and processing (item potency is handled separately).
-   Scalability and fault tolerance.
-   **Ordered Consumption Trade-off:** Perfect message ordering can only be guaranteed with a single consumer. With multiple consumers, inherent race conditions can lead to out-of-order processing. "Keep in mind that we're not going for ordered consumption we're going for pretty much like an approximate priority que."
-   **Heap vs. B-Tree/LSM-Tree:** While a heap is a natural data structure for a priority queue, it's not ideal for disk-based storage due to frequent random shuffles. B-trees and LSM trees are better for disk but can still be slow.
-   **Hybrid Approach Details:** "in memory over here we have a heap or a sorted list whatever you prefer to do and then over here on the right on disk we basically have something that almost resembles just like a write Ahad log where as nodes come in they get assigned some sort of ID based on their location."
-   In-memory data structure stores a sorted list (heap, sorted linked list with hashmap) of node IDs, ordered by priority.
-   Node IDs point to the actual data stored on disk (similar to a write-ahead log).
-   Disk writes are O(1), while in-memory insertions are O(log n).
-   **Modification of Existing Elements:**Each element on disk has an ID that is indexed in memory.
-   Changing the priority of a node requires an in-memory update.
-   Consider using a sorted linked list with a hashmap for constant-time access to elements for priority updates.
-   **Replication Strategies:**Multi-leader replication is likely not a primary requirement.
-   Leaderless replication (Quorum consistency) can provide strong consistency but may impact performance.
-   Single-leader replication is preferred for faster read/write response times.
-   A consensus algorithm (Paxos/Raft) with synchronous replication to a follower can provide fault tolerance in a single-leader setup. "every single leader has two followers that message has to be synchronously replicated to one of the followers uh before you know we can proceed right because uh that's kind of how something like raft works you have to get a majority of nodes to agree that the message is real before you can proceed and uh consider it a successful right so we can do that"
-   **Partitioning Considerations:**Round-robin partitioning is generally preferred for fair distribution and avoids hot spots.
-   Partitioning by priority range can lead to uneven distribution and concentrated reads on the highest priority shard.
-   **Consumer Handling Multiple Partitions:**Consumers handling multiple partitions need to aggregate data across all partitions.
-   Use a local heap in memory to track priorities across partitions.
-   Employ long polling to fetch the next task from each partition.
-   **System Design Overview:**Clients publish tasks to a load balancer (round robin).
-   The load balancer distributes tasks to priority queue nodes.
-   Each priority queue node stores data in a MySQL database (primary key order, resembling a write-ahead log).
-   In-memory heap indexes the priority score.
-   Paxos/Raft ensures replication within the partition.
-   Consumers use locking to prevent processing the same event concurrently.
-   Consumers handling multiple partitions use long polling to fetch tasks.

**Quotes of Note:**

-   "Basically our distributed priority queue is going to allow us to build out a scalable and fault tolerance system with fairly good read and write through putut that's going to allow all of these separate video encoders to basically pick off all of these items and make sure that they all get processed at least once."
-   "Keep in mind that we're not going for ordered consumption we're going for pretty much like an approximate priority que."
-   "in memory over here we have a heap or a sorted list whatever you prefer to do and then over here on the right on disk we basically have something that almost resembles just like a write Ahad log where as nodes come in they get assigned some sort of ID based on their location."
-   "every single leader has two followers that message has to be synchronously replicated to one of the followers uh before you know we can proceed right because uh that's kind of how something like raft works you have to get a majority of nodes to agree that the message is real before you can proceed and uh consider it a successful right so we can do that"

**Conclusion:**

This briefing document has outlined a design for a distributed priority queue system that prioritizes scalability, fault tolerance, and efficient asynchronous task processing. The system leverages a hybrid memory/disk architecture, consensus-based replication, and careful partitioning strategies to achieve its goals. While perfect priority ordering is not guaranteed, the design aims for near-optimal performance in a distributed environment.

Study Guide
===========

Quiz
----

**Answer each question in 2-3 sentences.**

1.  Why do companies like Facebook or Google need distributed priority queues?
2.  What are the four key operations supported by a distributed priority queue?
3.  What does it mean to say that the distributed priority queue design in the source goes for approximate priority?
4.  Why isn't a simple heap implementation ideal for a distributed priority queue that relies on disk storage?
5.  What is the hybrid approach to implementing a priority queue?
6.  How does the hybrid approach leverage both memory and disk?
7.  How does assigning an ID to each element help with modifying existing elements in the distributed priority queue?
8.  Why is single-leader replication with a consensus algorithm preferred over simple single-leader replication?
9.  Why is round robin partitioning preferred over partitioning by priority range?
10. How does a consumer handle multiple partitions in the distributed priority queue design?

Answer Key
----------

1.  Companies like Facebook or Google need distributed priority queues because they have many automated jobs they want to run asynchronously, and some jobs are more important than others. A distributed priority queue allows them to assign priorities and process these jobs in a scalable and fault-tolerant manner.
2.  The four key operations are: enqueueing (adding items), modifying existing items, dequeuing (removing items), and ensuring that each item is delivered and processed at least once. These operations allow the system to manage the priority and processing of tasks efficiently.
3.  This means that perfect ordering of messages is not guaranteed due to the presence of multiple consumers. While the system attempts to prioritize messages, race conditions and parallel processing can lead to some out-of-order execution, especially across partitions.
4.  A simple heap implementation involves frequent random shuffling of array elements, which is inefficient for disk-based storage. The constant jumping around the list slows down performance, making it unsuitable for a distributed system relying on disk access.
5.  The hybrid approach involves storing a sorted list or heap of primary keys in memory, while the actual data is stored on disk. The in-memory structure acts as an index, pointing to the data location on disk.
6.  The memory component stores a sorted index of node IDs based on priority, allowing quick access to the highest-priority elements. The disk component stores the actual data in a right-ahead log, providing durable storage and efficient writes.
7.  The ID, sorted by ID like an index, makes it easy to locate the element on disk and update its data or metadata. Without the ID, finding the element would require scanning the entire disk.
8.  Single-leader replication with a consensus algorithm ensures fault tolerance. If the leader fails before replicating a message, the consensus algorithm (like Raft or Paxos) ensures that the message is replicated to a follower before the write is considered successful, preventing data loss.
9.  Partitioning by priority range can lead to hot reads on the top shard since all reads would be directed there, which also assumes priority is a continuous variable. Round robin ensures that reads and writes are distributed more evenly across all partitions, balancing the load and preventing bottlenecks.
10. A consumer handling multiple partitions maintains its own local heap in memory. It uses long polling to fetch the highest-priority event from each partition, aggregates them in its local heap, and processes the event with the highest priority across all partitions, writing back to the database after completion.

Essay Questions
---------------

1.  Discuss the trade-offs between using a heap-based in-memory index versus a sorted linked list with a hashmap for a distributed priority queue. Consider factors such as read/write performance, memory usage, and complexity of implementation.
2.  Evaluate the different replication strategies (multi-leader, leaderless, single-leader with consensus) in the context of a distributed priority queue. Which replication strategy is most suitable, and why?
3.  Explain the importance of partitioning in a distributed priority queue and compare the advantages and disadvantages of round robin partitioning versus partitioning by priority range.
4.  Describe the hybrid approach to building a distributed priority queue, including the motivations for using both in-memory and disk storage and the key data structures involved.
5.  Analyze the system design presented, identifying potential bottlenecks and suggesting alternative approaches or optimizations to improve performance, scalability, or fault tolerance.

Glossary of Key Terms
---------------------

-   **Distributed Priority Queue:** A queue that manages items with assigned priorities across multiple nodes in a distributed system, ensuring that higher-priority items are processed before lower-priority ones.
-   **FIFO Queue:** First-In, First-Out queue. A type of queue where the first item added is the first item removed.
-   **Heap:** A tree-based data structure that satisfies the heap property: in a min-heap, the value of each node is greater than or equal to the value of its parent, with the minimum-value element always at the root node.
-   **B-Tree:** A self-balancing tree data structure that maintains sorted data and allows searches, sequential access, insertions, and deletions in logarithmic time. It is commonly used for file system and database indexing.
-   **LSM Tree:** Log-Structured Merge-Tree. A data structure that optimizes for write performance by accumulating data changes in memory and periodically merging them to disk in sorted order.
-   **Write-Ahead Log (WAL):** A log of changes made to a database before those changes are applied to the database itself, used for durability and crash recovery.
-   **Multi-Leader Replication:** A replication strategy where multiple nodes can accept writes, with changes propagated asynchronously to other nodes.
-   **Leaderless Replication:** A replication strategy where writes are sent to multiple nodes, and reads require querying multiple nodes to ensure consistency.
-   **Single-Leader Replication:** A replication strategy where one node is designated as the primary, and all writes are directed to it before being replicated to follower nodes.
-   **Consensus Algorithm:** An algorithm (e.g., Raft, Paxos) that ensures agreement among a distributed group of nodes, typically used for leader election and maintaining consistent state.
-   **Raft/Paxos:** Consensus algorithms used to achieve fault tolerance and consistency in distributed systems.
-   **Round Robin Partitioning:** A partitioning strategy where data is distributed evenly across partitions in a circular fashion.
-   **Long Polling:** A technique where a client makes a request to a server and keeps the connection open until the server has data to send back, reducing the need for frequent polling.
-   **Load Balancer:** A device or software that distributes network traffic across multiple servers to prevent any single server from being overwhelmed.
-   **Quorum Consistency:** A consistency model that requires a minimum number of nodes (a quorum) to agree on a write before it is considered successful, ensuring that reads have access to the most up-to-date data.
-   **Two-Phase Commit (2PC):** A distributed transaction protocol that ensures all participants either commit or rollback a transaction, guaranteeing atomicity.
-   **Kafka:** A distributed streaming platform that provides high-throughput, fault-tolerant data ingestion and processing.
-   **JMS/JMS Brokers (ActiveMQ, RabbitMQ):** Java Message Service. An API for sending messages between applications. ActiveMQ and RabbitMQ are popular message broker implementations.
-   **Item Potency Key/Function:** Designing functions so that they can be repeated without causing unintended effects. This helps ensure that messages are processed exactly once.

# Amazon Payment Gateway: Systems Design

**The YouTube transcript presents a systems design interview question focused on building a payment processing system for an e-commerce website.** **It emphasizes the importance of maintaining a perfect record of payments while avoiding duplicate orders.** **The design considerations prioritize data consistency and fault tolerance over latency, using techniques like distributed consensus algorithms (Raft) and item potency keys.** **The proposed system includes a centralized payments database with change data capture, a pending payments cache using Flink, and derived data tables for tracking buyer orders and seller revenue.** **The video also addresses potential failure scenarios and suggests solutions such as polling Stripe for payment status updates.** **The explanation involves creating item potency keys, webhooks, and a payment service to ensure that users are only charged once and payments can be verified.**

Briefing Document:
------------------

This briefing document summarizes the key themes and ideas discussed in the provided source regarding the design of an internal payment processing system for an e-commerce site similar to Amazon. The system focuses on handling payments, accounting for buyers and sellers, and ensuring data integrity and idempotency.

**I. Main Themes:**

-   **Data Integrity & Reliability:** The system prioritizes maintaining a perfect record of all orders and payments, even at the cost of latency. "We care a lot more that they're never going to lose that payment information and so we're going to optimize for that." This is crucial to avoid customer dissatisfaction and legal issues.
-   **Idempotency:** The system must prevent duplicate payments, ensuring that a user clicking the submit button multiple times doesn't result in multiple charges. "We don't want to allow duplicate orders as well... no double submitting payments AKA we want item potent payments."
-   **Scalability and Throughput:** The design considers the need to handle a large volume of transactions efficiently, especially for both writes and reads. Partitioning and caching mechanisms are proposed to address this.
-   **Asynchronous Processing & Failure Handling:** The system acknowledges that payment processing isn't always instantaneous and anticipates potential failures in communication with external services like Stripe. Polling and webhooks are used for asynchronous communication and error recovery.
-   **Derived Data & Analytics:** The system aims to provide insights into buyer spending and seller revenue trends over time, leveraging change data capture (CDC) and stream processing to populate derived data stores.

**II. Key Ideas & Facts:**

-   **Problem Requirements:**Accurate accounting of payments for buyers and profits for sellers, ordered by time.
-   Perfect record of all orders and payments (no data loss).
-   Prevention of duplicate payments (idempotency).
-   Integration with external payment processors (Stripe for incoming payments, Depal for outgoing).
-   **Design Considerations:**Latency is less critical than data integrity.
-   Payment approval may not be instant (Stripe processing times).
-   A single source of truth (a ledger or payments table) is essential, from which all other data is derived. "As long as we have one source of truth table right with all of our pending payments and our accepted or failed payments everything else that we have can be derived off of that..."
-   **Strong Consistency for the Source of Truth:**Synchronous replication is deemed insufficient due to lack of fault tolerance.
-   Distributed consensus algorithms (Raft, Paxos, Zab) are recommended to ensure strong consistency with fault tolerance. "The answer that we've at least spoken about is using distributed consensus algorithms... one that we've covered in depth on this channel is raft."
-   Raft ensures writes are persisted to a quorum of nodes, minimizing the risk of data loss even if some nodes fail.
-   Increased cluster size increases fault tolerance but also increases the number of network messages, slowing down rights.
-   **Idempotency Implementation:**Utilize Stripe's item potency keys to prevent duplicate payments.
-   Two options for generating item potency keys:
-   **Pre-materialization:** Generate a batch of random keys and distribute them across servers.
-   **On-demand Generation:** Clients generate a hash-based key (user ID + timestamp) and attempt to claim it in the database. Collision resolution is necessary.
-   **Partitioning for Scalability:**Partitioning the payments database by the hash range of the item potency key increases write and read throughput. "Instead of just having one payment table like this guy we can have a second one and then each of them is effectively running their own raft algorithm to make sure that all of their rights are strongly consistent."
-   This distributes load evenly and ensures that concurrent requests for the same key are routed to the same node.

1.  **Payment Workflow:**Client lands on payment page, a pending payment and item potency key are generated and stored.
2.  Payment server sends payment request to Stripe with the item potency key.
3.  Stripe processes the payment and sends a webhook notification to the payment server upon completion or failure.
4.  Payment server updates the status of the payment in the payments database, and CDC propagates the change.

-   **Failure Handling (Stripe Integration):Problem:** Payment server might die before submitting to Stripe, or Stripe webhook might fail to reach the server.
-   **Solution:** Implement a polling mechanism to periodically check the status of pending transactions with Stripe.
-   Avoid premature deletion of pending payment information. Wait some time before polling Stripe in case the payment information is slow to reach Stripe.
-   **Pending Payments Cache (Flink):**Use a pending payments cache to quickly identify payments that require polling from Stripe.
-   Flink is used to store the cache in memory and restore state from S3 in case of failure.
-   Doubly linked list (sorted by timestamp) and hashmap are used for efficient addition, removal, and retrieval of pending payments.
-   **Derived Data (Buyer Spending, Seller Revenue):**Utilize change data capture (CDC) to stream payment updates to derived data stores.
-   Stream consumer processes CDC events and updates user orders and seller revenue tables.
-   Time series databases (or column-oriented databases) are suitable for storing and querying this data.
-   **Diagram Overview:**Client requests item potency key from payment service.
-   Payment service interacts with strongly consistent payments database (key generation, status updates).
-   Stripe processes payment and sends webhook notification.
-   Flink monitors pending payments and polls Stripe for status updates.
-   Stream consumer processes CDC events and updates derived data stores (user orders, seller revenue).

**III. Quotes:**

-   "We care a lot more that they're never going to lose that payment information and so we're going to optimize for that."
-   "We don't want to allow duplicate orders as well... no double submitting payments AKA we want item potent payments."
-   "As long as we have one source of truth table right with all of our pending payments and our accepted or failed payments everything else that we have can be derived off of that..."
-   "The answer that we've at least spoken about is using distributed consensus algorithms... one that we've covered in depth on this channel is raft."
-   "Instead of just having one payment table like this guy we can have a second one and then each of them is effectively running their own raft algorithm to make sure that all of their rights are strongly consistent."

**IV. Conclusion:**

The design emphasizes data integrity, idempotency, and scalability in building a robust payment processing system for a large e-commerce platform. By leveraging distributed consensus, partitioning, caching, and asynchronous communication, the system can handle a high volume of transactions while maintaining a reliable and accurate record of all payments. The design also facilitates data analysis and reporting through the creation of derived data stores.

A Study Guide
=============

Quiz
----

Answer the following questions in 2-3 sentences each.

1.  Why is data loss unacceptable in a payment processing system for an e-commerce site?
2.  What is item potency in the context of payment processing, and why is it important?
3.  Why is latency a secondary concern compared to data integrity in this payment system design?
4.  Explain the issue with using synchronous replication for the payment database in this scenario.
5.  How does a distributed consensus algorithm like Raft help ensure data persistence in the payment system?
6.  How does Stripe's item potency key prevent duplicate payments?
7.  Explain how partitioning the payments database by the hash range of the item potency key helps improve throughput.
8.  Why is the item potency key generated when the confirmation page loads, rather than when the payment button is clicked?
9.  How does a web hook from Stripe notify the payment service about the completion or failure of a payment?
10. What is the purpose of the pending payments cache, and how is it implemented using Flink and Kafka?

Answer Key
----------

1.  Data loss is unacceptable because it can lead to customers being charged incorrectly or orders not being processed, resulting in customer dissatisfaction, disputes, and potential legal issues. Maintaining a perfect record of all orders and payments is essential for trust and compliance.
2.  Item potency means that submitting the same payment request multiple times will only result in a single payment being processed. This prevents users from being charged multiple times due to accidental or intentional resubmissions, ensuring fair and accurate billing.
3.  While speed is desirable, the primary focus is on ensuring that every payment is accurately recorded and processed without errors. The system prioritizes reliability and data integrity over immediate confirmation to prevent data loss and financial discrepancies.
4.  Synchronous replication lacks fault tolerance; if the leader or follower database goes down, writes cannot be processed, halting the payment system. This is unacceptable for an e-commerce site that needs to maintain continuous operation.
5.  Raft ensures that a write is committed to a quorum of nodes in the cluster, so even if the leader node fails, the right will still persist on the follower nodes and be available in subsequent epochs. This provides fault tolerance and guarantees that the data will not be lost due to node failures.
6.  Stripe uses the item potency key to identify duplicate payment requests. If the same key is submitted twice, Stripe recognizes it as a duplicate and ignores the second request, ensuring that the payment is only processed once.
7.  Partitioning distributes the data across multiple database nodes, allowing for parallel processing of read and write operations. By hashing the item potency key, related requests are routed to the same partition, minimizing conflicts and improving overall throughput.
8.  Generating the item potency key when the confirmation page loads ensures that the key is available even if the payment button is clicked multiple times from the same page. This prevents the generation of multiple keys for the same payment attempt, which could lead to duplicate payments.
9.  Stripe uses a web hook to send a notification to the payment service when a payment is completed or has failed. This allows the payment service to asynchronously update its records and notify the user without having to constantly poll Stripe for the status of the payment.
10. The pending payments cache stores a list of payments that are awaiting confirmation from Stripe. This allows the system to efficiently identify payments that have been pending for an extended period and poll Stripe for their status, without locking or interfering with the main payments database. Flink is used to maintain a stateful cache which is persisted to S3 in case of failure, and Kafka is used for change data capture and event streaming.

Essay Questions
---------------

1.  Discuss the trade-offs between strong consistency and eventual consistency in the context of a payment gateway system. How does the design described in the source material balance these trade-offs?
2.  Explain the significance of change data capture (CDC) in the system design. How does it facilitate the creation of derived data and improve the overall efficiency of the system?
3.  Analyze the failure scenarios described in the source material and evaluate the effectiveness of the proposed polling mechanism in addressing these scenarios.
4.  Critically assess the decision to partition the payments database by the hash range of the item potency key. What are the benefits and potential drawbacks of this approach?
5.  Compare and contrast different approaches for generating item potency keys, considering the trade-offs between pre-materialization and client-side generation. Which approach is more suitable for this system, and why?

Glossary of Key Terms
---------------------

-   **Item Potency:** The property of an operation that ensures it has the same effect regardless of how many times it is executed. In payment processing, it ensures that a payment is only processed once, even if the request is submitted multiple times.
-   **Strong Consistency:** A guarantee that any read operation will return the most recent write operation's value.
-   **Eventual Consistency:** A consistency model where data will eventually become consistent across all nodes in a distributed system, but there may be a delay.
-   **Synchronous Replication:** A database replication method where a write operation is not considered complete until it has been successfully replicated to all replicas.
-   **Distributed Consensus Algorithm:** An algorithm (e.g., Raft, Paxos) that allows a group of nodes to agree on a single value, even in the presence of failures.
-   **Raft:** A distributed consensus algorithm that ensures that all nodes in a cluster agree on the same log of operations.
-   **Quorum:** The minimum number of nodes in a distributed system that must agree on a value for a write operation to be considered successful.
-   **Two-Phase Commit (2PC):** A distributed transaction protocol that ensures that all nodes either commit or abort a transaction together.
-   **Item Potency Key:** A unique identifier used to prevent duplicate processing of the same request.
-   **Web Hook:** A user-defined HTTP callback that is triggered by an event. In this context, Stripe uses web hooks to notify the payment service about the completion or failure of a payment.
-   **Polling:** Periodically checking the status of a resource or service. In this context, the payment service polls Stripe to determine the status of pending payments.
-   **Change Data Capture (CDC):** A technique for tracking changes to data in a database and propagating those changes to other systems.
-   **Kafka:** A distributed stream processing platform used for building real-time data pipelines and streaming applications.
-   **Flink:** A distributed stream processing framework used for stateful computations and real-time data analysis.
-   **Stream Consumer:** An application that reads data from a stream processing platform like Kafka.
-   **Time Series Database:** A database optimized for storing and querying time-stamped data.
-   **Partitioning:** Dividing a database into smaller, more manageable pieces that can be stored and processed on different nodes.
-   **Sharding:** A type of database partitioning that distributes data across multiple physical machines.
-   **Derived Data:** Data that is calculated or aggregated from other data sources. In this context, derived data includes revenue per seller and spending per buyer.
-   **Ledger Table:** A central, immutable record of all transactions in the system. In this case, it's the payments database.
-   **Prematerialization:** Generating and storing item potency keys in advance.

# Designing TicketMaster: A System Design Perspective

**The YouTube transcript outlines a system design for TicketMaster/StubHub.** **It explores challenges related to ticket claiming and booking, considering factors like fairness and preventing overload.** **The video presents different approaches, including unfair claiming with time expiration and a fairer method using a FIFO waitlist.** **It then goes into the data models and algorithm design, such as linked lists with hashmaps, for building the system's data structures.** **Partitioning and data storage options, including in-memory caching and asynchronous data synchronization, are discussed to optimize performance and fault tolerance.** **The final architecture involves load balancing, order claim servers, asynchronous replication, change data capture with Kafka, and a partitioned bookings database.**

Briefing Document:
------------------

This document summarizes the key themes and ideas presented in the provided source, a systems design interview walkthrough focused on building a platform like TicketMaster or StubHub. The speaker, Jordan, outlines potential architectures and data structures, emphasizing the complexities arising from fairness, concurrency, and scalability.

### Main Themes:

-   **Ambiguity of Requirements:** The design problem is inherently ambiguous. Different ticket selling sites implement key features like claiming tickets differently. This necessitates exploring multiple approaches.
-   **Concurrency and Claim Management:** A central challenge is handling concurrent requests for the same tickets, particularly during high-demand events. The video explores different claim management strategies and their trade-offs.
-   **Fairness vs. Efficiency:** Implementing a fair system (e.g., FIFO waitlists) is significantly more complex than simpler, potentially "unfair" models.
-   **Scalability and Hotspots:** Handling spikes in traffic for popular events requires careful consideration of partitioning, caching, and asynchronous processing.
-   **Data Consistency and Fault Tolerance:** The design emphasizes achieving data consistency and preventing data loss, through techniques such as replication, write-ahead logs, and change data capture.

### Key Ideas and Facts:

1.  **Problem Requirements:**

-   Users can buy and sell tickets for events.
-   Upon selecting a seat, a user has a limited time (e.g., 5 minutes) to claim and purchase it.
-   The implementation of the "claiming" process itself is where the ambiguity lies.

1.  **Capacity Estimates:**

-   Estimated 100-200k events at one time.
-   Each event has ~100 sections with ~100 seats each.
-   Total data storage for active, bookable seats is estimated at ~20GB.
-   Historic bookings will require significantly more storage.

1.  *Quote:* "...if we're actually only trying to Service uh users for active events we don't actually need that much storage..."
2.  **Claiming Approaches:**

-   **Option 1 (Discarded):** Claiming tickets after payment. Considered a bad user experience as users complete payment information only to find out the ticket is unavailable. "*...probably a pretty bad user experience right because I'm not going to figure out that I didn't get the ticket until I've already done all this work...*"
-   **Option 2 (Unfair Claim):** Claiming tickets with a time-to-expire (TTE) before purchase. This can lead to "Thundering Herd" problems when a claim expires and multiple users simultaneously try to claim the same ticket.
-   **Option 3 (Fair Claim):** Using a FIFO waitlist. The fairest approach, but more complex to implement.

1.  **Unfair Claim Implementation Details:**

-   Database writes for claiming seats must be atomic and serializable.
-   SQL databases (e.g., MySQL, Postgres) are preferred for transactional support.
-   Partitioning can improve write throughput but introduces the risk of distributed transactions.
-   To avoid distributed transactions, keep all seats within the same row on the same partition. Partition key should be a hash of (event ID, section ID, row number).
-   Hotspots (popular events/rows) can be handled with:
-   Repartitioning hot rows.
-   Using a write-back cache (e.g., Redis, Memcached) for hot rows. "*...putting it in memory is going to allow us to speed up those requests.*" The cache acts as the temporary source of truth, asynchronously synced to the SQL database.
-   Claim expiration can be implemented by storing an expiration timestamp in the database, generated by the database itself.

1.  **Fencing Tokens:**

-   Use a combination of claim user ID and claim XPR as a fencing token to prevent malicious or outdated requests. The client must provide the correct fencing token when attempting to book.

1.  *Quote:* "the combination of the claim user ID and the claim XPR is itself a fencing token..."
2.  **Fair Claim (FIFO Waitlist) Implementation:**

-   Complex dependencies arise when users can select specific seats.
-   A proposed solution is to *not* allow users to select specific seats, but instead select a row or section. The backend assigns seats to the user.

1.  *Quote:* "...not letting users select their specific seats but rather they would just select a row or a section and in our back end we assign the user a certain amount of seats based on the number they're asking for..."
2.  **Data Model for FIFO Waitlist:**

-   Use a queue implemented with a doubly-linked list and a hashmap for O(1) removal of users leaving the queue. "*...we can use a doubly link list with a hashmap...*"
-   Consider the impact of multi-threading and locking on the linked list operations.
-   Expire unfillable orders (orders requesting more seats than available) by either waiting until they reach the front of the queue or by actively iterating through the queue.

1.  **Tree Index**

-   Maintain a Treet index of the orders by size to support efficiently identifying unfillable orders. However, this adds complexity and introduces contention due to locking.

1.  *Quote:* "...we do a Treet index of the orders by size..."
2.  **Partitioning and Hot Rows:**

-   Partitioning strategy similar to the unfair claim approach.
-   Isolate hot rows/events onto their own partitions.

1.  **Data Storage and Consistency:**

-   Use the in-memory linked list (waitlist) as the source of truth, rather than relying on a two-phase commit to the database.
-   Asynchronously replicate the in-memory data to another node.
-   Use Change Data Capture (CDC) via Kafka to sync data to the bookings database.
-   Partition the bookings database by user ID for efficient retrieval of user's ticket history.

1.  **System Diagram:**

-   Client requests hit a load balancer.
-   Load balancer directs requests to order claim servers (maintaining waitlists).
-   Real-time connection (e.g., WebSocket) between client and order claim server for queue position updates.
-   Order claim servers asynchronously replicate to a replica node and perform CDC to Kafka.
-   Kafka consumers sync data to the bookings database.
-   A booking service, behind a load balancer, retrieves bookings from the database.

### Conclusion:

Designing a ticketing system like TicketMaster involves tackling complex challenges related to concurrency, fairness, and scalability. The briefing doc synthesizes the main themes and key ideas from the source to address these core challenges. The implementation involves careful choices regarding data structures, caching strategies, and data consistency mechanisms.

Study Guide
===========

Quiz
----

**Instructions:** Answer each question in 2-3 sentences.

1.  What are the four components of the tuple used to represent each seat available for booking?
2.  Why is it important for database writes to be atomic and serializable when claiming a seat?
3.  What is a distributed transaction, and why are they undesirable in a system like TicketMaster/StubHub?
4.  Explain how partitioning can help distribute load and improve throughput in a database.
5.  What is a right-back cache, and how can it be useful in managing hot rows/sections?
6.  Why is it important for the database server to generate the claim expiration timestamp?
7.  What is a fencing token, and how is it used to validate booking requests?
8.  What is a thundering herd problem, and how does it relate to ticket claiming?
9.  What are the advantages of using a doubly linked list with a hashmap for managing the waiting list?
10. Explain how change data capture (CDC) can help reduce dependencies on the database.

Quiz Answer Key
---------------

1.  The tuple consists of the event ID, the section ID, the row ID, and the actual seat number. This allows for unique identification and tracking of individual seats within the system.
2.  Atomic writes ensure that either all claims for a selection of seats succeed or none do, preventing partial claims. Serializable writes prevent concurrent claims from conflicting, ensuring consistent state.
3.  A distributed transaction occurs when a transaction involves multiple partitions across different nodes. They are slow and undesirable because they require a two-phase commit, involving multiple network round trips for coordination.
4.  Partitioning divides the data across multiple database nodes, allowing for parallel processing of requests. By distributing the load, no single node becomes a bottleneck.
5.  A right-back cache stores frequently accessed data (like hot rows) in memory, allowing for faster read and write operations. It improves performance by reducing database load, as claim requests can be handled in-memory.
6.  It prevents malicious clients from setting arbitrarily long expiration times, which would unfairly hold seats and block other users. Generating the timestamp on the server ensures validity.
7.  A fencing token is additional data passed back to the client to be used to validate future booking requests. It ensures that only the client with a valid, unexpired claim can book the tickets.
8.  A thundering herd occurs when multiple clients simultaneously attempt to claim the same resource after it becomes available. In the context of ticket claiming, it can overload the database.
9.  A doubly linked list allows efficient removal of nodes from the middle of the list. A hashmap provides O(1) lookups by order ID for efficient removals.
10. Change data capture asynchronously streams database changes (e.g., bookings) to other systems, such as a booking database. This reduces the need for synchronous writes to the database during the claim/booking process, improving throughput and responsiveness.

Essay Questions
---------------

1.  Discuss the trade-offs between different approaches to handling ticket claims (e.g., claim after payment, timed claims, waitlists). Which approach do you think provides the best balance of user experience and system efficiency, and why?
2.  Explain the challenges of implementing a fair claiming system using a FIFO waitlist, and describe the proposed "middle ground" solution of allowing users to select only rows/sections. What are the pros and cons of this approach?
3.  Describe the data model and data structures used for the waiting list implementation, including the use of a doubly linked list with a hashmap. How do these choices contribute to the efficiency of the system?
4.  Discuss the partitioning strategy for both the order claim servers and the bookings database, and explain how these choices support the system's scalability and performance requirements.
5.  Explain the overall system architecture, including the roles of the load balancer, order claim servers, Kafka, and the bookings database. How do these components work together to provide a reliable and scalable ticket booking experience?

Glossary of Key Terms
---------------------

-   **Atomic Write:** A database operation where either all changes are committed or none are, ensuring data consistency.
-   **Serializable Write:** A database transaction property ensuring that concurrent transactions appear to execute in isolation, preventing conflicts.
-   **Distributed Transaction:** A transaction that spans multiple database nodes, requiring coordination and potentially slow two-phase commits.
-   **Partitioning:** Dividing a database into smaller, more manageable pieces that can be distributed across multiple servers.
-   **Right-Back Cache:** A cache that stores data in memory and writes it to the underlying database asynchronously.
-   **Hot Row/Section:** A database row or section that receives a disproportionately high amount of traffic.
-   **Fencing Token:** A unique identifier provided to a client that must be presented in subsequent requests to prevent unauthorized actions or race conditions.
-   **Thundering Herd:** A phenomenon where a large number of clients simultaneously attempt to access a resource, overwhelming the system.
-   **FIFO (First-In, First-Out):** A queuing principle where the first item added to the queue is the first item removed.
-   **Doubly Linked List:** A linked list where each node has pointers to both the next and previous nodes, enabling efficient traversal in both directions.
-   **Hashmap:** A data structure that stores key-value pairs and allows for efficient lookups using a hash function.
-   **Change Data Capture (CDC):** A process that identifies and captures changes made to a database and streams them to other systems.
-   **Kafka:** A distributed streaming platform used for building real-time data pipelines and streaming applications.
-   **Load Balancer:** A device or software that distributes network traffic across multiple servers to prevent overload and improve performance.
-   **Websocket:** A communication protocol that provides full-duplex communication channels over a single TCP connection.
-   **Write-Ahead Log:** An logging mechanism that logs all database operations before applying them, ensuring durability.
-   **Replication:** The process of copying data from one database server to another to provide redundancy and fault tolerance.
-   **Distributed Consensus:** Guarantees that multiple nodes in a distributed system agree on a single value, even in the presence of failures.

# Netflix and YouTube Systems Design

**This YouTube transcript presents a systems design interview question focused on building a scalable video-watching platform akin to Netflix or YouTube.** The speaker outlines the core requirements, including posting, watching, commenting, and searching for videos, while emphasizing read speed optimization. **Capacity estimates are discussed, targeting a billion users and massive storage needs.** The design incorporates video chunking for adaptive streaming across various devices and network conditions. **Database schema considerations are mentioned, with potential choices like MySQL and Cassandra being weighed for different data storage needs.** The transcript further describes video uploads, processing, aggregation, and content delivery networks (CDNs). **Finally, the design details building a distributed search index and discusses complex system architecture with multiple services and queues.**

Briefing Document:
------------------

This document summarizes the key themes and ideas presented in the provided source, which outlines a systems design approach for building a video-sharing platform like Netflix or YouTube. The discussion centers around scalability, video streaming optimization, database design, and video uploading processes.

**I. Main Themes & Requirements:**

-   **Core Functionality:** The system needs to allow users to post, watch, comment on, and search for videos. The primary focus is optimizing for video watching (read speeds).
-   **Scale:** Designed for billions of users, millions of videos posted daily, and substantial data storage (40 petabytes per year).
-   **Video Streaming Support:**Multiple device types (phones, laptops, tablets).
-   Varying network speeds (2G to Gigabit Ethernet).
-   Dynamic resolution adjustment based on bandwidth.

**II. Capacity Estimates:**

-   **Users:** 1 Billion
-   **Views per video:** 1,000 (average)
-   **Video size:** 100 MB (average)
-   **Videos posted per day:** 1 Million
-   **Storage needed per year:** 40 Petabytes

**III. Video Streaming Optimization:**

-   **Video Chunking:** Videos are split into chunks to enable adaptive streaming, allowing devices to dynamically switch between resolutions based on network conditions.
-   *"So the fact that we actually have all of these different chunks means that with a relatively small granularity we can pick the right chunk at the right time so that we can take advantage of our changing networking speeds"*
-   **Chunking Advantages:**Parallel uploads.
-   Lower barrier to starting a video (play starts after the first chunk loads).
-   Optimal chunk selection based on networking abilities.

**IV. Database Design:**

-   **Key Tables:Subscribers:** Stores user subscriptions, enabling newsfeed generation and pre-CDN pushing for popular content. Needs to efficiently determine who a user is subscribed to.
-   *"First let's start with a subscribers table so imagine that we have a user ID and another user ID that describes who we're subscribed to."*
-   **User Videos:** Tracks videos posted by users.
-   *"So this is just outlining if I'm a user what videos have I actually posted that's going to have a user ID it's going to have a video ID it's going to have the timestamp that we actually posted that video and then of course some metadata about the video like the name and the description"*
-   **Users:** Basic user information.
-   **Video Comments:** Stores comments for each video.
-   *"so for this one we've actually got a video ID and I should preface this with saying that in reality it would probably be like a channel ID and a video ID because eventually that's how I'm planning on uniquely identifying a video is a combination of just some channel name and then also the video ID that comes with it"*
-   **Video Chunks:** Crucial table. Tracks multiple encodings and resolutions for each video chunk and contains a hash to describe the chunk and the URL to where it is stored.
-   *"as you can see we now have a bunch of encodings and resolutions to worry about... by partitioning by video ID we ensure that all the chunks for a given video should be on the same note"*
-   **Database Choices:MySQL:** Suitable for most tables due to its B-tree indexing, which is efficient for read-heavy workloads.
-   *"we can get away with using something like my SQL here because it actually uses a b tree so if you recall uh the B tree is just a big tree on disk and it is a lot easier to iterate this on disk tree than it is to check multiple different SS tables"*
-   **Cassandra:** Recommended for the Video Comments table to handle potential high write loads for popular videos. Uses LSM tree architecture where rights go to memory first and has a leaderless architecture.
-   *"for that type of situation I think you could make the argument to actually use Cassandra here for a couple of reasons one the LSM tree architecture means that all rights first go to memory the second thing is that it's a leaderless architecture"*

**V. Video Upload Process:**

-   **Asynchronous Processing:** Video processing (encoding, resolution changes) is performed in the background to avoid blocking user uploads.
-   **Message Broker:** An in-memory message queue (RabbitMQ, Amazon SQS) is used to distribute video chunks to processing nodes. The author prefers an in-memory message queue.
-   *"In this case I think what we would want is probably going to be an inmemory message CU so if you recall at least from our Concepts videos the the reason behind using something like a Kafka broker or a log based message broker is that there's only one consumer..."*
-   **Object Storage:** Video chunks are stored in an object store (S3) instead of directly in the message broker, due to their size. The broker carries only references (URLs) to the chunks.
-   *"ideally what we would first want to do is upload them to some sort of either distributed file store or some object store and then after doing so we can put a reference to them in the broker"*
-   **Chunk Aggregation:** A system is in place to track when all chunks of a video have finished processing. Kafka and Flink are used for state management and idempotent processing to ensure that all the events are processed. The processors will upload that chunk information to Kafka.
-   *"so the type of message that we would send to this Kafka over here is this guy imagine we have video 10 and we're uploading that I would basically say in advance hey video 10 is three chunks that's what we're going to be looking out for"*
-   **Data Model Updates:** Updates to video metadata and chunk information are written to the database (video_chunks table and user_videos table) after all chunks have been processed.

**VI. Content Delivery Network (CDN) Pushing:**

-   **Preemptive CDN Push:** For videos from popular channels (based on subscriber count), content is proactively pushed to a CDN to ensure faster initial loading times for viewers.
-   *"with a service like YouTube or with a service like Netflix we actually have the privilege of knowing when a particular video is going to be popular in advance precisely because we know how many subscribers they have"*
-   **Subscription Table:** This is the foundation for knowing how popular a video will be.
-   **Order of Operations:** The video needs to be added to the database before pushing to the CDN or else it will be taking up space in the CDN and won't be watchable.

**VII. Search Index Design:**

-   **Inverted Index:** An inverted index is used to enable efficient searching by video title and description.
-   **Partitioning:** The approach to partitioning depends on balancing query speed with storage space.
-   **Denormalization:** Video metadata (title, description) is denormalized and stored directly in the search index to improve read performance.
-   *"in reality we're going to have to store you know a bunch of metadata for the video we're going to have to denormalize our data a little bit or else we're looking at potentially a super expensive query"*
-   **Update Propagation:** Editing video titles/descriptions is slower due to the need to update the search index but is considered acceptable.
-   **Change Data Capture (CDC):** Old and new video name and description can be extracted using CDC.

**VIII. System Diagram and Services:**

-   **Key Services:**User Service (User table, Subscriber table)
-   Comment Service
-   Upload Service
-   Chunk Processing Nodes
-   Elasticsearch
-   **Technologies:**Load Balancer
-   MySQL
-   Cassandra
-   S3
-   RabbitMQ
-   Kafka
-   Flink
-   CDN
-   **Sharding:**User ID and video ID
-   Subscriber table is sharded by the user who is doing the subscribing, not by the channel they are subscribing to.

**IX. Conclusion:**

The design emphasizes a scalable and efficient architecture for a video-sharing platform, addressing key challenges such as adaptive streaming, high write loads, and efficient search. The use of message queues, object storage, and different database technologies allows for optimized performance and fault tolerance. The design also considers the importance of knowing when a video is going to be popular in advance. The author highlights the importance of proper data synchronization, particularly with the order in which different data is written to different databases.

Study Guide
===========

Quiz
----

**Instructions:** Answer the following questions in 2-3 sentences each.

1.  What are the four primary actions users can perform on a video-watching site like YouTube or Netflix according to the video?
2.  Why is optimizing for read speeds more important than write speeds when designing a video-watching platform?
3.  Explain the purpose of video chunking and how it improves the user experience.
4.  Describe two advantages of splitting videos into chunks for uploading and streaming.
5.  Why is it important to have a database or table to keep track of video chunks?
6.  Why might Cassandra be a suitable database choice for managing video comments, particularly for popular videos?
7.  When uploading video chunks, why is an in-memory message queue like RabbitMQ preferred over a log-based message broker like Kafka?
8.  Why upload video chunks to object storage like S3 or HDFS *before* adding the metadata to a message queue like RabbitMQ?
9.  Explain how Flink is used to ensure that all video chunks have been fully processed and aggregated, even in the face of partial failures?
10. How can subscriber data be used to improve Content Delivery Network (CDN) delivery?

Quiz Answer Key
---------------

1.  Users can post videos, watch videos, comment on videos, and search for videos by name. Optimizing these actions ensures a comprehensive and engaging user experience within the platform.
2.  Because the vast majority of users are watching videos (reading data) far more often than users are posting or uploading them (writing data). Optimizing for read speeds ensures a smooth and efficient viewing experience for the majority of users.
3.  Video chunking involves splitting videos into smaller segments to support multiple resolutions and encodings. This allows devices with different bandwidths to dynamically load appropriate footage, improving the user experience.
4.  Splitting videos into chunks allows for parallel uploads, which can potentially speed up the process. It also lowers the barrier to starting a video, as users can begin watching after only the first chunk is loaded.
5.  Tracking video chunks is crucial due to the multiple copies of each video file in different resolutions and encodings. A database helps manage and retrieve the correct chunks for streaming.
6.  Cassandra's LSM tree architecture ensures fast write speeds by initially storing writes in memory, making it suitable for handling the high volume of comments on popular videos. Its leaderless architecture also avoids right conflicts.
7.  An in-memory message queue is favored because the order of processing the chunks doesn't matter, as each chunk processing task is independent. This allows for efficient round-robin load balancing among processing nodes.
8.  Because chunks are large, and object storage provides a more appropriate means of storage before being processed. The RabbitMQ queue would contain only lightweight metadata references to the chunks.
9.  Flink maintains state by tracking the number of expected chunks for each video and marking them as completed when processed. Because Flink processes each message at least once, it ensures that failed processes will be reattempted.
10. Subscriber information can be used to identify videos likely to become popular. These videos can then be proactively pushed to a CDN for faster delivery.

Essay Questions
---------------

1.  Describe the end-to-end process of uploading a video to a platform like YouTube or Netflix, from the initial upload by the user to the video being available for streaming, including all relevant services and data stores.
2.  Explain the challenges of building a scalable search index for a video-sharing platform, and discuss different partitioning strategies to optimize query performance.
3.  Discuss the trade-offs between different database choices (e.g., MySQL, Cassandra) for various components of a video-sharing platform, such as user data, video metadata, and comments.
4.  Describe the role of message queues (e.g., RabbitMQ, Kafka) in the video uploading and processing pipeline, and explain the reasons for choosing different types of queues for different tasks.
5.  Analyze the challenges of maintaining data consistency and fault tolerance in a distributed video-sharing platform, and discuss strategies for ensuring that videos are eventually available for streaming despite potential failures.

Glossary of Key Terms
---------------------

-   **Chunking:** Dividing a video into smaller, manageable segments for easier processing and streaming.
-   **Encoding:** Converting video files into different formats to support various devices and bandwidths.
-   **Resolution:** The quality of a video, such as 480p, 720p, 1080p, or 4K.
-   **B-tree:** A self-balancing tree data structure that maintains sorted data and allows searches, sequential access, insertions, and deletions in logarithmic time.
-   **LSM Tree:** A data structure with log-structured merge trees that optimize for write performance by accumulating data changes in memory and periodically flushing them to disk in sorted order.
-   **Change Data Capture (CDC):** A set of software design patterns used to determine and track data that has changed in a database.
-   **Kafka:** A distributed, fault-tolerant, high-throughput streaming platform used for building real-time data pipelines and streaming applications.
-   **Flink:** A framework and distributed processing engine for stateful computations over unbounded and bounded data streams.
-   **Message Broker:** Software that enables applications, systems, and services to communicate with each other and exchange information.
-   **In-Memory Message Queue:** A type of message broker that stores messages in memory for faster processing.
-   **RabbitMQ:** A popular open-source message broker that supports multiple messaging protocols.
-   **Hadoop Distributed File System (HDFS):** A distributed file system designed to store and process large datasets across clusters of commodity hardware.
-   **Object Storage:** A data storage architecture that manages data as objects, often used for storing unstructured data like videos and images (e.g., Amazon S3).
-   **S3 (Amazon Simple Storage Service):** A scalable, high-speed, web-based cloud storage service designed for online backup and archiving of data and application programs.
-   **CDN (Content Delivery Network):** A geographically distributed network of servers that cache content to deliver it to users more quickly and efficiently based on their location.
-   **Inverted Index:** A database index storing a mapping from content, such as words or numbers, to its locations in a document or a set of documents.
-   **Tokenization:** The process of breaking down a text document into individual words or tokens for indexing and searching.
-   **Elasticsearch:** A distributed, open-source search and analytics engine for all types of data, including textual, numerical, geospatial, structured, and unstructured.
-   **Denormalization:** A database optimization technique in which redundant data is added to one or more tables to avoid costly joins.
-   **Sharding:** A database partitioning technique that separates very large databases into smaller, faster, more easily managed parts called data shards.
-   **Round Robin Load Balancing:** A simple load balancing technique that distributes incoming requests sequentially to each server in a group.
-   **Item potent:** An operation that can be performed multiple times without changing the result beyond the initial application.

# Typeahead Suggestion System Design: Google Search Bar Implementation

**The YouTube transcript discusses the system design of a type-ahead suggestion feature, similar to what's found in search bars or text messaging apps.** **It explores various algorithmic approaches, from simple caching to the use of tries, balancing read speed and memory usage.** **The discussion covers capacity estimations, emphasizing the need for fast read speeds and daily updates of suggestions.** **The transcript also addresses challenges in a distributed environment, like the need for partitioning and sharding to handle large datasets and high traffic.** **Finally, it proposes using range-based partitioning with dynamic repartitioning to handle potential hotspots and a batch job processing pipeline using Hadoop, Kafka, and Flink for updating the suggestion data.**

Briefing Document: 
------------------

**Overview:**

This document summarizes a discussion of designing a typeahead suggestion system, similar to those used in text messaging and Google Search. The discussion covers algorithmic and systems design considerations, emphasizing the need for fast read speeds and exploring different approaches to data storage, retrieval, and updating.

**Main Themes and Ideas:**

1.  **Problem Definition and Requirements:**

-   The system should provide quick suggestions as users type. These can be individual words (like in text messaging) or full search terms (like in Google Search).
-   Latency for updates to the suggestion list is less critical than read speed, with an acceptable update frequency of "at least once per day."
-   The initial design will not incorporate personalized suggestions.
-   Key goal: optimize for read speed

1.  **Capacity Estimation and Storage Considerations:**

-   **Single-Word Suggestions (Text Messaging):** The speaker considers whether the entire dataset of suggestions can be stored in memory on the client device.
-   Estimation: 200,000 English words, average length of 5 characters.
-   Calculations: Caching all prefixes and top 3 suggestions requires approximately 20MB. "20 megabytes is actually not a lot of memory even if we're dealing with phones these days..."
-   Alternative: Storing only the counts of each word requires only 800KB but necessitates on-the-fly computation of top suggestions.
-   **Multi-Word Suggestions (Google Search):**Estimation: 1 billion unique search queries per day, average length of 20 characters.
-   Calculations: Storing all prefixes and top 10 suggestions requires approximately 4TB. "Now that's obviously a lot. Not only is that so much data that we can't store it on client devices, we also couldn't even store it on one server..." This necessitates a distributed approach.

1.  **Implementation Details and Data Structures:**

-   **Non-Caching Approach (Low Memory):**Sort words by search order (alphabetically).
-   Use binary search to find the range of words matching a prefix.
-   Employ a Min-Heap of size *n* (e.g., 3 or 10) to find the top *n* suggestions within that range.
-   Complexity: "o of n but then you also multiply in a factor of you know log of three which is effectively constant so we don't really care about this but the point is it's linear to the number of terms that are suffixed by your search term"
-   **Local Caching Approach (HashMap):**Store a hashmap from prefix to top suggestions.
-   Provides O(1) retrieval *per character* (though hashing the prefix isn't truly O(1)).
-   **Trie Data Structure:**A tree-like structure where each node represents a character, allowing for efficient prefix-based searching.
-   At each node, cache the top suggestions.
-   "At every single node we cach the top terms and then let's imagine that you know I was already making a search right so I already got the top three terms for this and then now all of a sudden I want to get the top terms for AP so it's an O of one jump down to AP and then I already have access to the top terms which is going to be really great"
-   Memory usage is linear in relation to search term
-   Can further compress memory by "dictionary compression"

1.  **Updating the Trie:**

-   Writes can be problematic as a single write can affect parent nodes
-   Writes will likely require locking
-   A batch job is preferred to update the trie
-   Data is uploaded via the following flow: Client -> Server -> Kafka -> Flink -> HDFS
-   Spark job: Take every term, convert into prefixes. Partition by term. Use Min Heap to take top N suggestions
-   New try can be loaded with a lead code problem: While loop creating node at current index

1.  **Distributed System Design (for Google Search):**

-   **Partitioning/Sharding:** Essential due to the large data volume.
-   Websockets can be used for communication: supports bidirectional connections
-   Range-based partitioning is suggested: sort all terms alphabetically
-   Hotspots and repartitioning: create small fix-sized partitions and use another service to monitor the load and copy to smaller nodes
-   **Architecture Diagram:**Client -> Load Balancer -> Suggestion Service via Websocket
-   Suggestion clicks flow into: Load Balancer -> Aggregation Service -> Kafka -> Hadoop
-   Hadoop computes frequencies, the results are sent to Suggestion service which builds the Try.

**Key Quotes:**

-   "20 megabytes is actually not a lot of memory even if we're dealing with phones these days..."
-   "Now that's obviously a lot. Not only is that so much data that we can't store it on client devices, we also couldn't even store it on one server..."
-   "At every single node we cach the top terms and then let's imagine that you know I was already making a search right so I already got the top three terms for this and then now all of a sudden I want to get the top terms for AP so it's an O of one jump down to AP and then I already have access to the top terms which is going to be really great"

**Conclusion:**

The design of a typeahead suggestion system involves trade-offs between memory usage, computational complexity, and read speed. While a trie data structure with caching offers the potential for fast lookups, scaling to large datasets requires a distributed approach with careful consideration of partitioning strategies and update mechanisms.

Study Guide
===========

Quiz
----

1.  What are the two main examples of typeahead suggestions discussed in the source, and how do they differ?
2.  What are the three problem requirements specified for the typeahead suggestion system?
3.  Why is it important to prioritize read speeds in a typeahead suggestion system?
4.  Explain the non-caching approach for implementing typeahead suggestions, and what are its trade-offs?
5.  How does the Trie data structure improve the efficiency of typeahead suggestions compared to a hash map?
6.  Explain dictionary compression as an optimization for reducing memory footprint in a Trie-based typeahead system.
7.  What are the challenges associated with updating the Trie data structure to reflect more recent searches?
8.  Describe the batch job process using Hadoop and HDFS for updating the typeahead suggestions.
9.  Explain how range-based partitioning can be used to distribute the Trie data across multiple servers in the Google search use case.
10. How does the suggested system architecture utilize websockets, a load balancer, Kafka, and Hadoop to deliver typeahead suggestions?

Quiz Answer Key
---------------

1.  The two main examples are auto-suggestions in texting (suggesting words based on prefixes) and the Google search bar (suggesting full search terms). The Google search bar example is more complex because it deals with multiple words and a larger number of potential suggestions.
2.  The requirements are to quickly load the top suggestions as text is typed, to support both individual words and full search terms, and to update the suggestions at least once per day.
3.  Prioritizing read speeds is important because users expect suggestions to appear quickly as they type. Otherwise, the feature becomes less useful, since the speed of modern typing would outpace the suggestions.
4.  The non-caching approach involves storing counts of each word and then computing the top suggestions on the fly by doing a range query and using a min-heap. It saves memory but sacrifices read speed due to the computation needed for each query.
5.  A Trie allows for O(1) jumps between prefixes, as opposed to the O(n) performance hit in the hash map with the cost of generating hashes of multi-character inputs. Each node in the Trie caches the top suggestions for that prefix, enabling quick access.
6.  Dictionary compression involves creating a map from short (two-byte) codes to frequently repeated search terms. Instead of storing the full term at each node, the short code is stored, saving memory.
7.  The challenge lies in the fact that a single write can propagate all the way up the tree to the start node. This requires locking every node or using atomics to increment counters in order to prevent concurrency bugs.
8.  Search terms are uploaded to HDFS using Kafka and Flink, partitioning by the search term name. A Spark job then converts search terms to prefixes, partitions by prefix, and uses a Min Heap to find the top suggestions, storing the results back in HDFS for the client nodes.
9.  Range-based partitioning involves sorting all possible search terms alphabetically and assigning ranges of terms to different servers. This ensures that as users type additional characters, they are likely to stay within the same partition and maintain a persistent websocket connection.
10. Clients connect to a load balancer, which assigns them to a suggestion service with a websocket connection. The load balancer directs click events to an aggregation service, then to Kafka and Hadoop. Hadoop computes frequencies and suggestions, sending them back to the suggestion service to update the Trie.

Essay Questions
---------------

1.  Discuss the trade-offs between memory usage and read speed when designing a typeahead suggestion system. Compare and contrast the caching and non-caching approaches, providing specific examples of when each would be more suitable.
2.  Explain the role of data structures, such as Tries and Min Heaps, in optimizing the performance of a typeahead suggestion system. How do these data structures contribute to efficient suggestion retrieval and ranking?
3.  Describe the challenges involved in scaling a typeahead suggestion system to handle a large volume of data and user traffic. How can techniques like partitioning and replication be used to address these challenges?
4.  Evaluate the proposed architecture for a distributed typeahead suggestion system, including the use of websockets, Kafka, and Hadoop. What are the strengths and weaknesses of this approach, and how could it be further improved?
5.  Discuss the importance of updating typeahead suggestions to reflect recent search trends. What are the challenges associated with real-time updates, and how can batch processing techniques be used to address these challenges effectively?

Glossary of Key Terms
---------------------

-   **Typeahead Suggestion:** A feature that predicts and suggests words or phrases as a user types in a search box or text field.
-   **Prefix:** A sequence of characters that appears at the beginning of a word or phrase.
-   **Capacity Estimates:** Calculations and estimations used to determine the storage and processing requirements of a system based on expected usage patterns.
-   **Caching:** Storing frequently accessed data in memory to reduce access time and improve performance.
-   **Non-Caching Approach:** An approach where suggestions are computed on-the-fly rather than being pre-computed and stored in memory.
-   **Min Heap:** A tree-based data structure where the value of each node is less than or equal to the value of its children.
-   **Trie (Prefix Tree):** A tree-like data structure used for storing strings, where each node represents a character and paths from the root to the leaves represent words or phrases.
-   **Dictionary Compression:** A technique for reducing memory usage by replacing frequently repeated strings with shorter codes.
-   **Batch Job:** A process that performs a large number of operations in a single, scheduled run, typically used for data processing and analysis.
-   **HDFS (Hadoop Distributed File System):** A distributed file system designed for storing and processing large datasets across a cluster of commodity hardware.
-   **Range-Based Partitioning:** A data partitioning technique where data is divided into ranges based on the sorted values of a key.
-   **Hotspot:** A partition or server that receives a disproportionately high amount of traffic or requests, leading to performance bottlenecks.
-   **Websockets:** A communication protocol that provides full-duplex communication channels over a single TCP connection.
-   **Kafka:** A distributed streaming platform used for building real-time data pipelines and streaming applications.
-   **Load Balancer:** A device or software that distributes network traffic across multiple servers to ensure high availability and optimal performance.
-   **Aggregation Service:** A component responsible for collecting and aggregating data from multiple sources into a unified format.
-   **Stateful Server:** A server that maintains information about past client interactions, such as the current position in the Trie.
-   **Commodity Hardware:** Readily available and inexpensive computer hardware components.
-   **Data Locality:** The principle of storing data close to where it is processed to reduce latency and improve performance.
-   **RPC Call:** Remote Procedure Call; a mechanism that allows a computer program to execute a procedure in another address space (commonly on another computer on a shared network) as if it were a normal (local) procedure call
