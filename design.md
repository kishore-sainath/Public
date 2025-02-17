# Analysis of Distributed Systems Design
Briefing Document: 
------------------

**Overall Summary:**

This document consolidates insights from several design deep dives focusing on key elements in the design of various distributed systems. The reviewed systems encompass:

-   Distributed Locking
-   Distributed Job Scheduling
-   Metrics and Logging
-   Unique Active User Counting
-   E-Commerce Product Catalog
-   Mapping Applications (Google Maps)
-   Push Notification Services
-   File Sharing
-   Audio Recognition (Shazam)
-   Leaderboards
-   Rate Limiters
-   Ride Sharing Applications
-   Social Media Comment Sections
-   Distributed Priority Queues
-   Payments Processing
-   Video Streaming
-   Typeahead Suggestions

**I. Distributed Locking System**

-   **Theme:** Ensuring Exclusivity and Fault Tolerance in Distributed Environments
-   **Key Ideas:Exclusivity:** Preventing concurrent modifications of shared resources is critical in distributed systems.
-   **Fencing Tokens:** Monotonically increasing sequence numbers are assigned to lock acquisitions to invalidate delayed or failed operations from previous lock holders. "S3 is going to keep track of all the fencing tokens it's seen when machines edit files on it...s3 can go ahead and disregard those saying oh wait that's an old fencing token i don't want to worry about those this is invalid and it's going to mess things up."
-   **Timestamps vs. Fencing Tokens:** Timestamps are unreliable due to clock synchronization issues.
-   **Raft vs. Quorums:** Raft is preferred over quorums because it incorporates a commit phase, increasing reliability.
-   **Thundering Herd Problem:** Addressed using a queue lock.
-   **Queue Lock:** Using a linked list queue with notifications to avoid a surge of requests upon lock release. "As opposed to just telling a machine that wants to grab the lock when the lock is not available...what you can actually do on raft is effectively keep a linked list of all the requests for the lock."

**II. Distributed Job Scheduler**

-   **Theme:** Ensuring "At-Least-Once" Job Execution in the Face of Failures
-   **Key Ideas:Functional Requirements:** Job scheduling (instant and calendar-based), "at-least-once" execution.
-   **Architecture:** Client interaction via HTTP/RPC, binary storage in S3 ("since it's a static file it should be going in something like S3 which is just an elastic object store"), metadata storage in MySQL ("We're going to need some sort of database and what the database is going to do is store metadata about each job"), message queue (RabbitMQ/SQS), consumer nodes, and a distributed lock (ZooKeeper).
-   **Failure Handling:** Database queries for retries, timeouts, ZooKeeper for monitoring. The system queries the database to find jobs "that haven't started yet with a timestamp that is less than the current time and also go ahead and find all the jobs that have started however their timestamp that kind of you know they were last checked into the database so for example you know they're in queueing timestamp plus the actual you know hard set in queueing timeout which we've set in our system is going to be less than the current time".
-   **Claim Service:** Prevents duplicate job executions by implementing distributed locking. "...we don't want multiple nodes running that same job at the same time effectively what they're doing is they are hitting a distributed lock which we can manage behind the scenes with something like zookeeper and saying I am currently the only one running this job and I should be the only one running this job".
-   **Amortized Cron-like Scheduling:** The author describes this as "... amortize this computation where let's say I have a job that's supposed to be run every two weeks when one version of that job completes you know like the the first one and it's successful or fails then basically what's going to happen is the service that goes ahead and updates the metadata base to say that this job succeeded or failed is also going to add an additional row in the database saying here's the next one that has to be done two weeks from now so basically the previous job is going to put the next job that has to be run in the database".
-   **Idempotency:** "...the only way I can think of to prevent this is to take some extra measures to ensure item potency of your code where basically perhaps you're consulting like a different database table or something to see if something with that exact request id has been run before".

**III. Metrics and Logging System**

-   **Theme:** Centralized Data Collection, Processing, and Storage for Insights
-   **Key Ideas:Functional Requirements:** Client-generated logs and metrics, accessibility for analysis, long-term persistence. *"Clients or even possibly devices if we're talking about metrics are able to generate both logs and metrics that other you know internal people working in the company will eventually be able to check out and hopefully gain some insights from...they should also be available in a way that is persistent for further analytical exploration down the line."*
-   **Architecture:** Kafka for message brokering, Flink/Spark Streaming for processing, time-series database and S3 data lake for storage.
-   **Message Broker Choice:** Log-based message broker (Kafka) is preferred over in-memory (RabbitMQ) due to durability and replayability. "...for our case it seems like a log based message broker may be a little bit better...one of which is the durability of messages because everything is persisted to disk we can be sure that none of our metrics or number logs that we have are getting lost...having a message broker that allows us to replay messages that were in the past will allow us to a add new types of consumers that can do new types of processing on the data and b basically um you know get their state back if they were to crash."
-   **Stream Processing:** Stream enrichment and time-based aggregations using Flink/Spark Streaming. *"a lot of these messages are going to have things like a user id but they're probably not going to include a bunch of relevant data with that user which we may actually want in our logs so we don't have to query a sql table every time we see a given log...Flink is handling messages in real time whereas spark is handling messages in mini batches"*
-   **Time-Series Database:** Optimized for timestamped data, improving read/write performance. *"the really useful thing about a time series database is that is specifically created in order to handle this timestamp data like metrics and like logs so that it can be both written and read really quickly...all of these writes for you know a given day are probably only going to be going to a couple of these mini indexes and as a result we can cache the entire index in memory and get really really good write performance and read performance by doing that"*
-   **Data Lake (S3):** *"the better place to put it would be something like s3 and we would use s3 as something known as a data lake where we're basically just throwing in a bunch of jumbled unformatted data and then we know that we're going to be doing some processing on it later"*
-   **Data Encoding & Archiving:** Use data encoding frameworks (Avro, Protocol Buffers, Thrift) and data archival. *"you can encode a lot of these messages using data encoding frameworks like avro protocol buffers or thrift and that's going to be super useful because not only does it reduce the amount of data that has to be sent over the network but more importantly it is going to reduce the amount of data that's actually being held on disk"*

**IV. Unique Active User Counting**

-   **Theme:** Scalable Approximation Techniques for Real-time Analytics
-   **Key Ideas:Challenges:** Counting unique users at scale while handling multiple devices per user.
-   **Bad Implementation:** Full table scan is resource-intensive and slow.
-   **HyperLogLog:** Approximation algorithm balancing accuracy and space efficiency.
-   **Sharded HashMap:** Sharded hash map with consistent hashing ensures that each user ID is consistently assigned to the same node, avoiding duplicate counting when aggregating results from different nodes.

**V. E-commerce Product Catalog**

-   **Theme:** Balancing data consistency, read speed, and write frequency in a massive product catalog.
-   **Key Ideas:Partitioned MongoDB:** Used for storing product information.
-   **CRDTs (Conflict-Free Replicated Data Types):** CRDTs with multiple database leaders ensure that add/delete/re-add scenarios are properly handled in cart services. Clients should read from the same leader they write to, ensuring they see their own changes.
-   **Kafka and Flink:** Flink (stream processing) ensures that orders are processed in a non-concurrent fashion by buffering in Kafka and processing orders one at a time.
-   **Caching and Indexes:** Popular items cached in Redis, inverted index for text search. *Local indexing* is chosen, where each node contains a subset of the products.
-   *"I think single leiter replication should be fine I think that um using a b Tree in theory is going to be optimal..."*
-   *"This is one of the very few cases where I'm interested in no SQL solely for the data model or the flexibility there and uh I think that mongod DB is going to be the choice here..."*

**VI. Mapping Applications (Google Maps)**

-   **Theme:** Efficient Route Calculation and Real-Time Traffic Updates at Scale
-   **Key Ideas:Data as a Graph:** Modeling the map as a graph of nodes (locations) and edges (roads). "The point is we have a crap ton of data and for all of these nodes in our graph and for all of these edges we're going to need a decent amount of metadata."
-   **Contraction Hierarchies:** Simplify the search space by removing unimportant edges and nodes and creating shortcut edges. "The idea here is we can basically do two things to reduce the search space for our Dyers algorithm the first thing is that we can basically just straight up remove certain edges right...another thing that we can actually do is get rid of certain nodes and edges entirely by creating something known as shortcut edges."
-   **Multi-Level Sparse Graphs:** Multiple layers of sparse graphs to handle different distances. If I want to run dyras on that it's just not possible right and on the other hand if I wanted to just go and stay within New York that might be a little bit easier and if I wanted to drive 10 minutes from my house that might be even easier."
-   **Graph Database Choice:** Graph database (Neo4j) is used for storing the base graph. "The reason I say use a graph database is because probably storing this memory is going to be out of question...the reason I say a graph database is because we have to run traversals." The advantage of native graph databases is the use of pointers that allow constant lookups.
-   **Partitioning by Geohash:** Groups nearby points, improving efficiency.
-   **Real-Time ETA Updates:** Kafka and Flink calculate average road speeds. "if we have a shortcut Road and the shortcut road is made up of a variety of individual real roads if one of those real roads all of a sudden gets a ton of traffic on it we may want to update that shortcut Edge because all of a sudden we don't want all of our cars being routed through that traffic we'd to wrap them elsewhere."

**VII. Push Notification Services**

-   **Theme:** Scalable fan-out message delivery with item potency
-   **Key Ideas:Fan-Out Pattern:** Delivery of a message to many subscribed users. "building a notification service is a pattern that we've now seen three different times on this channel...the fan out pattern AKA when you deliver a message to a variety of users that are all subscribed or interested to some sort of topic."
-   **Kafka and Flink:** Kafka (sharded by topic ID) is used as a stream, with Flink as a consumer responsible for routing. we'll take our topic subscription table we can Shard It Out by user ID...but then when we actually push this change data to Kafka we can basically rehart it by our topic ID and so this goes in sharted by topic ID so that Flink can easily consume to just one kofka Q". Flink is sharded on topic ID.
-   **Popular vs. Unpopular Topics:** Popular topics are polled by clients to prevent pushing to many subscribers. "for certain topics uh we're going to have way too many users subscribing to it right such that it doesn't actually make sense to deliver that message to each user individually the cost is just going to be too expensive."
-   **Bloom Filters:** Used to optimize item potency database reads. "we could use something like a bloom filter so a bloom filter is going to allow us to ideally get rid of this step one where we have to read the database to check if uh that item potency key has been seen already".
-   **Data Storage:** Topic subscriptions in MySQL, notifications in Cassandra.
-   *"Additionally all of the changes to files need to be propagated on other devices and that's probably going to be via a push style of change."*

**VIII. File Sharing System**

-   **Key Ideas:Chunking:** Splits files into chunks for efficient upload/download. "splitting files into chunks means that if you make a small change to a file on a local machine you don't have to re-upload the entire file but basically only the modified chunks"
-   Conflict Resolution: "if a client is to ever make a conflicting write you basically say hey your writes about to conflict go ahead and redownload the newer version of the document merge them in and then you should be good to go but we'll obviously make sure to check it again so we don't have any race conditions or anything like that"
-   Partitioning: Partitioning ensures that for a given file to find all the chunks of it, we can just look at one partition.
-   Diff: Only apply changes to the differences between the older version of the file that we had locally and the newer version of the file
-   *"...i'm basically just going to go ahead and steal them from rocky again because I don't know they're pretty much arbitrary and then we're obviously going to put more focus into the design section"*

**IX. Audio Recognition (Shazam)**

-   **Theme:** Indexing and Matching Audio Fingerprints for Fast Song Identification
-   **Key Ideas:Spectrogram Conversion:** Audio converted to a spectrogram.
-   **Constellation Map:** Spectrogram reduced to a 2D constellation map.
-   **Combinatorial Hashing:** Pairs of notes and time differences create unique fingerprints. *"shazam does instead to limit down the space is they actually look at pairs of notes and this has a bunch of really useful properties so basically if you take two frequencies so two of those points on the graph and you take the first one which comes before the second one in time and you say the first one is what we'll call the anchor point basically create some sort of tuple where we have you know the anchor frequency the second frequency and then the last thing is going to be the amount of time and offset between them"*
-   **Inverted Index Lookup:** Fingerprints are used to query an inverted index. *"if we have an inverted index where basically the inverted index also is taking all of these hashes or these fingerprints of all of the existing songs that we know and says you know for this hash right here here all the song ids that correspond to it or have that that exact hash then we can look up all of the hashes in our user clip find all of the potential songs that we have to you know look at to compare them and then say okay now we have all these potential songs where one hash is matching what is the best match we can get when we look at all of the hashes in our user clip"*
-   **Sharding and Parallelization:** Index sharded and lookups parallelized to handle large song libraries. *"basically I'm going to assume that there are 100 million songs that we're you know having under consideration and then basically per song let's imagine that there are 3 000 indexable keys... and if each of those indexable keys is 64 bits then we can basically say that we're going to have to create an index that is 2.4 terabytes"*

**X. Leaderboards**

-   **Key Ideas**The system needs to efficiently identify and retrieve the top K items from a large dataset, efficiently process data, and scale well.
-   Log-based message broker for asynchronous communication and a stream processing framework for real-time performance.

**XI. Rate Limiters**

-   **Key Ideas**Controlliing the rate of requests to prevent abuse
-   Local vs. Distributed: Local rate limiting is done within each service, but it does not shield application servers from bursts of traffic and tightly couples application and rate-limiting scaling. Distributed rate limiting protects application servers and allows independent scaling, but introduces extra network calls.
-   Fixed vs. Sliding Windows

**XII. Ride Sharing Applications**

-   **Key Ideas**Data locality should be kept in mind for range queries
-   WebSockets provide a real-time, bidirectional connection
-   Partitioning major cities
-   Prioritizing low latency over data persistence

**XIII. Social Media Comment Sections**

-   **Key Ideas**Prioritize read speeds
-   Causal consistency is crucial for a parent to child relationship
-   Graph databases
-   Sliding windows
-   Ensuring data makes it to the correct location via a Kafka Queue

**XIV. Distributed Priority Queues**

-   **Key Ideas**The system needs to be scalable and fault tolerant, while maintaining an ordered list of items with priorities.
-   Not going for 100% ordered consumption
-   Hybrid Approach: store everything in disk but index into the disk in memory
-   Partitioned with a consenus algorithm.
-   Clients publish tasks to a load balancer (round robin).

**XV. Payments Processing**

-   **Key Ideas**Ensures that the payment is processed only once
-   A database is needed to store the payment to prevent double processing
-   Synchronously write a message after an operation
-   Webhooks and polling are options

**XVI. Video Streaming**

-   **Key Ideas**Chunking and encoding are necessary for video and should be stored and served to the user
-   If the website has high loads then Cassandra could be used for storing video comments

**XVII. Typeahead Suggestions**

-   "At every single node we cach the top terms and then let's imagine that you know I was already making a search right so I already got the top three terms for this and then now all of a sudden I want to get the top terms for AP so it's an O of one jump down to AP and then I already have access to the top terms which is going to be really great"
-   *Writes can be problematic as a single write can affect parent nodes, writes will likely require locking, and a batch job is preferred to update the trie*
-   **Implementation Details and Data Structures:****Trie Data Structure:**A tree-like structure where each node represents a character, allowing for efficient prefix-based searching. At each node, cache the top suggestions *Range-based partitioning, use a small fix-sized partition to monitor load

**General Takeaways:**

-   **Scalability is paramount:** All designs emphasize the importance of handling large datasets and high user volumes through sharding, partitioning, and efficient algorithms.
-   **Trade-offs are inevitable:** Designs often involve trade-offs between consistency, latency, and resource utilization.
-   **Data Locality matters:** Storing related data together improves performance.
-   **Fault Tolerance is critical:** Replication, consensus algorithms, and careful failure handling are essential for system reliability.
-   **Message queues and stream processing are key components** for distributing and processing data. Kafka and Flink are often mentioned.

This briefing document should provide a solid overview of the key distributed systems design concepts discussed in the provided sources.

Study Guide
-----------

#### Distributed Locking Design and Supporting Concepts & Distributed Job Scheduler Design Quiz

Answer the following questions in 2-3 sentences each.

1.  What is a distributed lock, and why is it useful?
2.  Provide two examples of scenarios where a distributed lock would be beneficial.
3.  What is a fencing token, and how does it prevent issues arising from delayed operations?
4.  Explain why timestamps are not a reliable method for generating fencing tokens in a distributed system.
5.  Briefly describe how the Raft consensus algorithm works to maintain consistency in a distributed system.
6.  Why are quorums considered unreliable for distributed locking systems?
7.  Describe the "thundering herd" problem in the context of lock acquisition.
8.  Explain how the linked list of lock requests solves the thundering herd problem.
9.  What technology is suggested to notify a node that it can acquire a lock in the lock request queue?
10. Summarize how fencing tokens work to reject write requests with examples.

#### Distributed Job Scheduler Design Quiz

Answer the following questions in 2-3 sentences each.

1.  What are the three main functional requirements of the distributed job scheduler?
2.  Why is S3 used to store the binary in the distributed job scheduler?
3.  Why is RabbitMQ or SQS preferred over Kafka for the queueing service?
4.  What role does ZooKeeper play in the distributed job scheduler design?
5.  What is the purpose of indexing the timestamp column in the database?
6.  Describe the purpose of the Claim Service.
7.  How does the system handle job retries, and what metadata is essential for this process?
8.  When is a new row added to the database for the job?
9.  What happens if the consumer running the job fails?
10. Describe the term idempotency.

#### Metrics/Logging Design Quiz

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

#### Unique Active User Counting System Quiz

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

#### Designing A Stock Exchange Quiz

1.  What is the primary goal of an exchange matching engine?
2.  What two protocols does the author explore as potential data transfer methods?
3.  Explain the difference between UDP and TCP in the context of order transmission to the matching engine.
4.  How does state machine replication ensure fault tolerance in the matching engine?
5.  Explain how to speed up message transfer and increase CPU efficiency of the message sending service.
6.  Why is partitioning (sharding) necessary when designing a stock exchange?
7.  What is high-frequency trading, and what implications does it have on the design of a stock exchange?
8.  What is the purpose of a retransmitter, and why is it needed?
9.  What does the glossary refer to as an "Only Fans?"
10. What does the glossary refer to as "Karina Kopp?"

#### Building A Push Notification Service Quiz

1.  What is the central architectural pattern used in the notification service?
2.  How is Kafka used in this architecture?
3.  How does Flink pre-populate a table that avoids expensive database lookups?
4.  Why does the system differentiate between popular and unpopular topics, and how are they handled differently?
5.  How do the notifications get delivered when the topic has a lot of subscribers?
6.  What is Item potency?
7.  Explain the trade-offs of implementing Item potency on either the server or the client side.
8.  How do clients determine which notification server they should connect to?
9.  What is a bloom filter and what can it tell you?
10. Describe the different data storage services and their uses.

#### File Storage Design Quiz

1.  Why is the system designed to split files into chunks?
2.  Explain conflict resolution, and how that functionality is implemented in the file storage system.
3.  How do you find all the chunks for a given file?
4.  What is the role of the hash in chunk metadata?
5.  What is a log based message queue?
6.  Explain the role of the coordination service.
7.  What is an API Endpoint?
8.  What is a two-phase commit?
9.  What is a "diff," and how is it used in the context of file synchronization?
10. What is Data locality?

#### Shazam Audio Recognition Design Deep Dive Quiz

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

#### Top K Frequent Items Leaderboard Design Quiz

1.  What is the goal of the Top K leaderboard system?
2.  Discuss what the author refers to when exploring capacity requirements, and how can these affect your choice of architecture?
3.  What is the API spec for a given item?
4.  What is a log-based message broker, and how is it used to populate the batch processing system?
5.  What is the difference between a Min-Heap and a Max-Heap?
6.  Describe the Count-Min Sketch.
7.  How does the system ensure it delivers exactly one message and maintains the database?
8.  Why use Flink and Kafka as data streaming tools?
9.  Where do all of the events get dumped for final storage and analysis?
10. The author suggests the architecture is eventually consistent; what are the properties of eventual consistency?

#### Rate Limiter System Design Quiz

1.  What is the primary purpose of a rate limiter in a system?
2.  Why is it important to minimize latency when implementing a rate limiter?
3.  Explain the difference between rate limiting based on User ID and IP address. What are the pros and cons of each?
4.  Describe the basic interface of a rate limiter. What inputs and outputs would it typically have?
5.  What are the advantages and disadvantages of local rate limiting compared to distributed rate limiting?
6.  How can caching be used to improve the performance of a distributed rate limiter? Where in the architecture would this caching typically be implemented?
7.  Why is Redis a suitable choice for storing rate-limiting data?
8.  Explain the key difference between fixed window and sliding window rate-limiting algorithms.
9.  How does the sliding window algorithm prevent the exploitation seen in fixed window algorithms?
10. What concurrency considerations are important when implementing a rate limiter?

#### Designing A Recommendation Engine Quiz

1.  Describe the difference between a high performance database index and a geo location database index.
2.  If the design is for a very small system of about 10 million users, will a complex partitioning scheme be worth the time?
3.  Why use WebSockets in this design?
4.  What guarantees low latency?
5.  Why partition by geohash?
6.  How can you avoid a thundering herd problem?
7.  In this design, should you prioritize data persistence, or low latency?
8.  How does a geospatial index work?
9.  Describe the various drivers that are chosen, given a ride request.
10. Why is a table called "claimed_trips" created?

#### Building A Comments Section Quiz

1.  How many comment view orders does Reddit typically support?
2.  Why is keeping all the comment views on the same database node beneficial?
3.  When designing the comments section for a hypothetical new social media site, should read speeds or write speeds be prioritized?
4.  In reference to the hypothetical new social media site, what must be maintained when displaying nested comments, and how is that enforced?
5.  What type of database is generally the most efficient for reading nested comments?
6.  What physical property of graph databases makes them efficient?
7.  What property of relational databases makes them efficient?
8.  Why is building a "top comments index" challenging?
9.  How can you efficiently sort for hot comments?
10. How should derived data be ensured to different comment indexes?

#### Distributed Priority Queue Quiz

1.  How does distributed message queueing help video encoders perform their job?
2.  Does the distributed priority queue need to be ordered?
3.  In the hybrid approach that was discussed in the video, why is the hybrid approach suggested rather than just a heap/b-tree?
4.  Why are memory indexes used, given that the data can just be stored on disk?
5.  When modifying existing elements in the queue, how do the memory and disk sections interact?
6.  What synchronous replication consensus algorithm was mentioned?
7.  When the server is trying to assign a job, how does it know which job to assign?
8.  What is the most important reason for load balancing?
9.  How should the client ensure the message is only delivered at least once?
10. Should multiple leaders be implemented, or should data just be replicated to all the followers?

#### Designing a Payment System Quiz

1.  What is the purpose of optimizing for "never going to lose that payment information?"
2.  What kind of "double submitting" do we want to avoid in a payment processing system, and what property does that result in?
3.  Where should pending payments be stored?
4.  What distributed consensus algorithm is used in the implementation?
5.  Instead of just having one payments table, what is the other approach?
6.  What are web hooks and why are they important in the stripe design?
7.  Explain the trade offs between pushing the webhook and pulling the payment details from stripe.
8.  Why store each table in a time series database partitioned by user ID?
9.  What Kafka and Flink used for?
10. What is the meaning of prematerialization with respect to the item potent key?

#### Designing A Video Upload Service Quiz

1.  What are the key tables needed to store all the uploaded video files?
2.  Why should all of the chunks for a given video be on the same node?
3.  Why are there encodings and resolutions for a single video chunk?
4.  In this design, why is Cassandra recommended over SQL?
5.  Why is the object storage layer in the cloud instead of a local device?
6.  Why should users subscribe for pre-CDN pushing popular content?
7.  Why partition the database when there is just one terabyte of data?
8.  How does one uniquely identify a video?
9.  Does the recommendation system support item potency or item importance?
10. What are you balancing with a round robin load balancing system?

#### Designing a Typeahead System Quiz

1.  What are the four main requirements for a typeahead design?
2.  Describe the formula for caching all prefixes for single word typeahead.
3.  Does storing only the counts of each word require the creation of new suggestions, or do the original suggestions persist?
4.  What is the advantage of searching via a Trie Data structure?
5.  Does memory usage increase linearly or logarithmically using Trie Data structure?
6.  With respect to the Trie Data structure, what is the importance of having a batched process that updates the table?
7.  Does this model explore personalized data?
8.  What technology is used to aggregate user clicks?
9.  What is the preferred communications method used by the client, load balancer, and suggestion service?
10. With range-based partitioning, how are hotspots avoided?

### Quiz Answer Key

#### Distributed Locking Design and Supporting Concepts & Distributed Job Scheduler Design Quiz - Answer Key

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

#### Distributed Job Scheduler Design Quiz - Answer Key

1.  The three main functional requirements are: scheduling jobs to run on a cluster, supporting both instant and calendar-based scheduling, and ensuring every job is executed at least once. The system needs to reliably execute jobs.
2.  S3 is used for storing binaries because it is an elastic object store that serves static files efficiently. This allows the system to decouple binary storage from the scheduler's processing logic.
3.  RabbitMQ or SQS are preferred over Kafka because of their in-memory nature and support for automatic retries. The job scheduler doesn't require message durability or strict ordering, and in-memory brokers are generally faster.
4.  ZooKeeper manages distributed locks to ensure that only one worker processes a job at a time, preventing duplicate executions. It also monitors consumer node heartbeats to detect failures.
5.  Indexing the timestamp column enables efficient querying of jobs that need to be run. This allows the system to quickly identify jobs whose scheduled time has passed or that have timed out.
6.  The Claim Service manages distributed locks, ensuring that only one consumer node is working on a particular job at any given time, preventing duplicate execution of jobs.
7.  The system queries the database to find jobs that haven't started or have timed out. Essential metadata includes the job ID, S3 URL, current status, and timestamps.
8.  A new row is added to the database after a scheduled job completes (successfully or unsuccessfully) to represent the next scheduled execution of the job. This ensures that recurring jobs are automatically rescheduled.
9.  If the consumer running the job fails, ZooKeeper detects the missed heartbeat. The metadata database is updated to reflect the failure, and the job is put back into the queue for retry.
10. Idempotency is the property of an operation that allows it to be executed multiple times without changing the result beyond the initial application. This is important to ensure the system does not rerun a job for which the user already got results.

#### Metrics/Logging Design Quiz - Answer Key

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

#### Unique Active User Counting System Quiz - Answer Key

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

#### Designing A Stock Exchange Quiz - Answer Key

1.  The primary goal is to efficiently and accurately match buy and sell orders.
2.  The author explores UDP Multicast and TCP as potential data transfer methods.
3.  TCP guarantees reliable, ordered, and error-checked delivery, but is slower. UDP Multicast allows for simultaneous transmission to multiple recipients without individual connections, making it faster but less reliable.
4.  State machine replication involves replicating all operations across multiple nodes, ensuring that even if one node fails, others can continue processing orders based on the consistent state.
5.  Sharding is required to distribute the load across multiple servers, each responsible for a subset of securities, thus improving the system's overall throughput.
6.  To speed up message transfer and increase CPU efficiency of the message sending service, batching the messages rather than sending them one at a time can be explored.
7.  High-frequency trading involves algorithmic trading characterized by high speeds and turnover rates. It demands extremely low latency and high throughput from the exchange.
8.  A retransmitter stores and re-broadcasts messages that clients may have missed, ensuring that all clients receive all order updates.
9.  The glossary refers to Only Fans as an internet content subscription service.
10. The glossary refers to Karina Kopp as an internet content creator on Only Fans.

#### Building A Push Notification Service Quiz - Answer Key

1.  The central architectural theme is the "fan-out pattern", delivering a message to multiple subscribed users.
2.  Kafka is used as a stream for handling notifications, sharded by topic ID. It allows for durability and scalability in managing the flow of notifications.
3.  Flink is pre-populated with a sharded topic subscription table by user ID, allowing Flink to do fast database queries.
4.  Differentiating between popular and unpopular topics optimizes notification delivery. Popular topics, with many subscribers, are polled, while unpopular topics are pushed in real-time.
5.  Popular topics are delivered via a notification polling service that allows clients to retrieve popular notifications on a periodic basis. This avoids overwhelming the system with push notifications.
6.  Item potency ensures that each notification is processed and delivered only once, even with failures and retries, preventing duplicate notifications.
7.  Client-side storage of Item potency keys can be limited by memory constraints. Server-side storage risks failures where the key is written, but the message isn't sent.
8.  Clients connect to notification servers via a routing server, which uses consistent hashing based on a routing server and consistent hashing to determine server load.
9.  A Bloom filter is a probabilistic data structure used to determine if an item potency key has been seen before. It tells you that a key definitely has *not* been seen, avoiding database lookups.

-   MySQL database, sharded by user ID, to store topic subscriptions.
-   Cassandra, partitioned by topic ID and sorted by timestamp, to store notifications.
-   Redis cache to store popular notifications in front of the Cassandra database.

#### File Storage Design Quiz - Answer Key

1.  Splitting files into chunks means that if you make a small change to a file on a local machine you don't have to re-upload the entire file but basically only the modified chunks"
2.  If a client is to ever make a conflicting write you basically say hey your writes about to conflict go ahead and redownload the newer version of the document merge them in and then you should be good to go but we'll obviously make sure to check it again so we don't have any race conditions or anything like that"
3.  For a given file to find all the chunks of it we can just look at one partition"
4.  The role of the hash in chunk metadata is used for data integrity checks and identifying duplicate chunks.
5.  A log based message queue is a durable and persistent message queue where messages are stored in a sequential log, allowing clients to replay messages if they become disconnected.
6.  Clients connect to notification servers via a routing server, which uses consistent hashing based on a routing server and consistent hashing to determine server load.
7.  An API Endpoint is a specific URL that provides access to a web service or application.
8.  Two-Phase Commit Is a distributed algorithm technique that guarantees all processes in a distributed transaction either commit or rollback.
9.  A "diff" is a summary of the differences between two files.
10. The most efficient configuration is that of Data locality, where each chunk of a larger file is stored sequentially with its adjacent neighbors on the same server.

#### Shazam Audio Recognition Design Deep Dive Quiz - Answer Key

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

#### Top K Frequent Items Leaderboard Design Quiz - Answer Key

1.  The goal is to provide a continuously updated, ranked list of the most frequent items, allowing users to see the current top performers.
2.  The calculations are based on the number of users, number of searches, and number of items. Calculations can affect design choices because a tiny, single server system doesn't need the data processing tools of a giant system that has high loads.
3.  The API spec for a given item could be the user's unique ID, and the date of the update, and the number of counts the item has.
4.  A log-based message broker is a system for storing and delivering messages in a specific sequence. This way you can re-deliver those events again and again to different parts of your system.
5.  In a min-heap, the smallest element is always at the root, while in a max-heap, the largest element is always at the root. They are used differently based on whether you need to efficiently track minimum or maximum values.
6.  A Count-Min Sketch is a probabilistic data structure that approximates the frequencies of elements in a stream while using a limited amount of memory. It enables efficient tracking of item frequencies.
7.  A 2PC across the consumer and database is an optimal way to guarantee delivery of exactly one message. It can also be ensured by storing some side data (for example, an item potent key), that indicates whether or not a job has been run before.
8.  Flink allows for real-time processing and aggregation of item frequencies, while Kafka serves as a durable message queue to ensure no data is lost during processing. They are used for both scalability and reliability in stream processing.
9.  All of the events ultimately get dumped for storage and analysis into HDFS, Hadoop's filesystem. This is useful for historical record keeping.
10. The properties are: If no new updates are made, all accesses will eventually return the last updated value. There is no guarantee of when an individual access will reflect an update, but eventually, all accesses will.

#### Rate Limiter System Design Quiz - Answer Key

1.  A rate limiter's main goal is to prevent users (malicious or otherwise) from overwhelming a system with too many requests, thus maintaining stability and performance. It protects against abuse and ensures fair resource allocation.
2.  Rate limiters are a necessary function for a smooth, efficient experience. Adding latency would frustrate users, potentially driving them away and hurting overall user experience.
3.  User ID-based rate limiting is good for signed-in users, preventing individual accounts from overloading services, but falls apart when users create multiple accounts. IP address-based rate limiting works for anonymous users, but may incorrectly throttle legitimate users sharing the same IP.
4.  A rate limiter interface typically includes a function, 'rate_limit', that takes User ID, IP address, the service name and the request timestamp as input. It returns a Boolean value indicating whether the request should be throttled.
5.  Local rate limiting reduces network calls because it is done in each service, but it does not shield application servers from bursts of traffic and tightly couples application and rate-limiting scaling. Distributed rate limiting protects application servers and allows independent scaling, but introduces extra network calls.
6.  Caching reduces network calls to the rate limiter. Implementing a write-back cache on the load balancer can store the request counts of frequent users, filtering out many bad actors requests before they ever reach the rate limiter.
7.  Redis is suitable because it's an in-memory database which is very fast and it offers the right kind of data structures that you would need for rate limiting. The speed helps minimize added latency, plus Redis has support for managing single leader replication.
8.  Fixed window rate limiting resets the request count at fixed time intervals, but allows bursts of requests at the window boundaries. Sliding window rate limiting considers a moving time window, preventing bursts and offering greater accuracy.
9.  The sliding window maintains a linked list, allowing one to accurately calculate the requests made over a sliding window, which is far more precise than a fixed window. The list is constantly culled and updated, which means no exploitation is possible.
10. Concurrency requires careful consideration to avoid race conditions and data inconsistencies. The increment of the count in a fixed window algorithm must be done atomically, and linked lists in the sliding window algorithm require locking mechanisms to prevent modification exceptions.

#### Designing A Recommendation Engine Quiz - Answer Key

1.  A high performance database index has low latency for a specific table, whereas a geo location database index is optimized for 2D data.
2.  If the system is going to handle a small amount of data, then a simple index should be used. However, with a high amount of data, the index needs to have partitioning for multiple machines.
3.  Websockets has persistent connections to support quick transmission of data without having to set up new connections every time.
4.  The use of web sockets, and quick lookup indexes in the database guarantees low latency.
5.  Partition by geohash ensures similar locations are on the same partition and are not stored on a separate machine.
6.  Implement client side code that implements a time period between attempting to connect to the machine.
7.  Data persistence should be traded off in favor of low latency, because the users location data changes frequently. It does not have to be persistent to offer the most updated data.
8.  A geospatial index can break down 2D data into a hierarchy of bounding boxes which can be used to identify areas quickly.
9.  The first option was to assign the closest driver to the ride. The second option was to contact every single driver and having the first person to respond win. The third was to contact the highest rated drivers and have them win the job.
10. The "claimed_trips" helps determine when a user has been assigned and should no longer look for new trips.

#### Building A Comments Section Quiz - Answer Key

1.  Reddit typically supports four comment view orders: new, hot, top, and controversial.
2.  Keeping all the comment views on the same database node is beneficial because it avoids the need for a two-phase commit, which simplifies data consistency.
3.  When designing a comments section, read speeds should generally be prioritized, as most users primarily browse and read comments.
4.  Causal consistency must be maintained; a child comment cannot exist without its parent comment being displayed.
5.  A graph database is generally the most efficient for reading nested comments due to its optimized traversal capabilities.
6.  Graph databases can use physical pointers, which minimizes slowdown as the database size increases.
7.  Relational databases offer data locality, ensuring that related data (parent and child comments) are stored contiguously in the index.
8.  Building a "top comments index" is challenging because it requires continuously sorting comments by upvotes at every level of the comment tree.
9.  Hot comments can be efficiently tracked using a sliding window approach with a stateful consumer, such as Flink, to monitor upvotes within a specific time window.
10. It's important to ensure derived data makes it to different comment indexes using a Kafka queue, partitioned by post ID, to guarantee reliable and ordered delivery.

#### Distributed Priority Queue Quiz - Answer Key

1.  Distributed message queueing allows video encoders to pick off items and process them, ensuring that each video is encoded at least once.
2.  The distributed priority queue does not need to be perfectly ordered; an approximate ordering based on priority is sufficient.
3.  The hybrid approach is suggested rather than just a heap/b-tree because it offers a good balance between in-memory access speed (heap) and efficient disk storage (B-tree/LSM-tree).
4.  Memory indexes are used because accessing the index is much faster in memory than on disk, allowing for quick prioritization and retrieval of tasks.
5.  When modifying elements, the in-memory index is updated first to reflect the new priority, and the corresponding data on disk is then updated asynchronously.
6.  Paxos/Raft is a consensus algorithm that ensures that writes are replicated across the leader and its followers, maintaining fault tolerance.
7.  When the server is trying to assign a job, it aggregates tasks from multiple partitions using a local heap.
8.  Load balancing is important to distribute tasks across priority queue nodes, preventing overload and improving system throughput.
9.  The client will assign an ID for each job when the write is sent and as a result it will be delivered at least once.
10. Data replication is preferred over multiple leaders.

#### Designing a Payment System Quiz - Answer Key

1.  The purpose of optimizing for "never going to lose that payment information" is to ensure the reliability and integrity of financial transactions.
2.  The system is designed to prevent multiple payments, which results in "item potent payments."
3.  Pending payments should be stored in a source of truth table, often referred to as the ledger table, to ensure consistency and accuracy.
4.  The distributed consensus algorithm is not explicitly specified.
5.  The other approach involves having a second payment table, with each table running

FAQs: 
-----

### Distributed Locking

-   **What is a distributed lock, and why is it important in distributed systems?** A distributed lock is a mechanism that provides exclusive access to a resource across multiple nodes in a cluster, preventing concurrent modifications that could lead to data corruption or ensure exclusive operations in distributed environments. They are important because they allow you to coordinate actions across multiple independent machines.
-   **What are fencing tokens, and how do they prevent issues caused by delayed or failed lock holders?** Fencing tokens are monotonically increasing sequence numbers assigned to each lock acquisition. The target service tracks these tokens and ignores requests with older tokens, preventing data corruption by ensuring only the most recent operation is valid, even if a previous lock holder experiences a delay or failure.
-   **Why are timestamps unreliable for generating fencing tokens?** Timestamps are unreliable in distributed systems because clock synchronization across different nodes is challenging. If the timestamp server fails and another takes over, it might issue older timestamps, breaking the monotonicity requirement of fencing tokens.
-   **Explain the "thundering herd" problem and how a queue lock solves it.** The "thundering herd" problem occurs when a lock is released, and all waiting nodes simultaneously try to acquire it, overwhelming the locking service with requests. A queue lock maintains a linked list of lock requests within the Raft cluster, notifying only the next node in the queue when the lock is released, preventing the simultaneous bombardment of requests.

### Distributed Job Scheduler

-   **What are the critical functional requirements for a distributed job scheduler?** The primary functional requirements are scheduling jobs to run on a dedicated cluster, supporting multiple scheduling types (instant and calendar-based), and ensuring every job is executed at least once, even in the face of failures.
-   **Why is ensuring "at-least-once execution" so important, and what mechanisms are used to guarantee it?** At-least-once execution is critical to ensure that all jobs are completed, even if failures occur. The system uses database queries to find failed or timed-out jobs, message queues for retries, and ZooKeeper for monitoring worker health and triggering retries when failures are detected.
-   **Explain the role of a distributed lock in a distributed job scheduler, and how it ensures jobs are not executed more than once?** The Claim Service uses a distributed lock (e.g., with ZooKeeper) to ensure only one worker processes a job at a time, preventing duplicate execution. A consumer hits a distributed lock and says "I am currently the only one running this job and I should be the only one running this job."
-   **Describe how the job scheduler handles scheduled jobs (Cron-like), ensuring they are executed at the correct time and interval.** After a scheduled job completes, the service updates the metadata and adds a new row to the database representing the next scheduled execution of the job, allowing the cost of the computation of scheduling to be amortized. Transactions are used to ensure that both the completion update and the next job creation succeed or fail together.

### Metrics and Logging

-   **What are the two primary functional requirements for a distributed metrics and logging system?** The two primary requirements are for clients to generate logs and metrics for internal analysis, and for these logs and metrics to be persistently stored for further analytical exploration (e.g., batch processing) to support in-depth analysis over time.
-   **Why is a log-based message broker, like Kafka, preferred over an in-memory message broker for this system?** A log-based message broker offers durability because messages are persisted to disk, ensuring no data loss. This is critical for accurate metrics and logs and allows consumer nodes to replay past messages, enabling new types of processing and recovery from crashes.
-   **Explain the concept of stream enrichment and provide an example of how it enhances log data.** Stream enrichment involves adding relevant contextual data to log messages. For example, enriching a log entry containing a user ID with additional user information (e.g., location, demographics) from a separate database avoids repetitive SQL queries.
-   **What are the advantages of using a time series database specifically for metrics and logging data compared to a general-purpose database?** Time series databases are specifically designed to handle timestamped data like metrics and logs, optimizing both read and write performance. Their architecture allows for faster data visualization and analysis of time-dependent trends compared to general-purpose databases.

### Unique Active User Counting

-   **What are the main functional requirements for the unique active user counting system?** The functional requirements include counting unique users and search terms at scale, tracking user counts historically via timestamp, and accounting for users with multiple devices. The system needs to provide both real-time and historical data.
-   **Why is performing a full table scan on a database to count unique users considered a bad implementation?** A full table scan consumes significant database resources, takes a long time to complete (especially with large tables), and requires substantial memory for storing user IDs, making it inefficient.
-   **Describe the basic principle behind the HyperLogLog algorithm for approximating unique counts.** HyperLogLog estimates unique counts by analyzing the rightmost zeros in the binary representation of data elements. It finds the maximum number of consecutive rightmost zeros and estimates the cardinality of the set based on that value.
-   **How does the system architecture use both stream processing and batch processing to provide both approximate and accurate results?** Stream processing uses HyperLogLog for real-time, approximate results, while batch processing, using Hadoop/Spark, performs accurate calculations on historical data. The system combines these results to offer both speed and accuracy.

### Google Maps Design

-   **What are the three primary requirements for Google Maps as identified in the video?** The three requirements are to find the quickest possible route between two locations, provide an estimated time of arrival (ETA) for the route, and incorporate real-time updates (traffic, weather) into the ETA.
-   **Why is Dijkstra's algorithm not always feasible for long-distance route calculations?** Dijkstra's algorithm can become too slow for long-distance route calculations because it has to traverse a large number of nodes and edges; therefore, the search space is too large, resulting in unacceptable latency.
-   **Explain the concept of "contraction hierarchies" and how they are used to optimize route calculations.** Contraction hierarchies involve simplifying the map by removing unimportant edges and nodes and creating shortcut edges to reduce the search space for Dijkstra's algorithm. This helps to focus the search on major roads.
-   **Explain why a graph database is preferred for storing the base graph in a Google Maps system.** A graph database is preferred for storing the base graph because it is optimized for graph traversals, using pointers to nodes on disk. The native graph database architecture ensures fast retrieval of connected nodes.

### Notification Service

-   **What is the "fan-out pattern," and how does it relate to building a notification service?** The "fan-out pattern" is where a single message needs to be delivered to multiple subscribers. It's central to a notification service, as a single event triggers notifications to many interested users.
-   **How does the system handle popular topics differently from unpopular topics, and why?** Popular topics are polled by clients instead of pushed in real-time to avoid overwhelming the notification servers with individual push notifications to every subscriber. This approach is more scalable for topics with a large subscriber base.
-   **What is item potency, and why is it important in a notification service?** Item potency is ensuring that each notification is processed and delivered only once, even in the face of failures or retries. It prevents users from receiving duplicate notifications, ensuring a good user experience.
-   **How are Bloom filters used to optimize database reads for item potency checks?** Bloom filters are used to quickly determine if an item potency key has been seen before, avoiding unnecessary database lookups. They are probabilistic data structures and can have false positives, so they only avoid reads and don't guarantee correctness.

### File Sharing Service

-   **What are the advantages of splitting files into chunks for a file-sharing service?** Splitting files into chunks means that if you make a small change to a file, you don't have to re-upload the entire file but only the modified chunks. This improves efficiency by reducing bandwidth usage.
-   **How does the system handle conflicting writes from multiple clients on the same file?** If a client makes a conflicting write, the system informs the client to download the newer version of the document, merge their changes, and re-upload it, checking again to avoid race conditions.
-   **What is a Merkle Tree and how is it used for anti-entropy processes in this design?** Merkle Tree is a tree data structure in which each leaf node is a hash of a block of data, and each non-leaf node is a hash of its children. Using the tree structure you only need to compare root nodes of the tree to be certain that all underlying chunks are in sync, preventing the need to compare underlying individual chunks.
-   **What advantage does S3 provide?** S3 is an object store that offers high availability and scalability, plus its low cost makes it an excellent choice for the storage of files. S3's high availability and scalability makes it an excellent choice for file storage.

### Shazam System Design

-   **Explain the purpose of the constellation map in the Shazam algorithm.** The constellation map simplifies the audio search space by reducing a complex 3D spectrogram to a 2D scatter plot of high-amplitude points (peaks). This abstraction accounts for variations in audio quality and external noise.
-   **How does Shazam use combinatorial hashing for music recognition?** Shazam uses combinatorial hashing by pairing frequencies from the constellation map (anchor point and secondary point) and calculating the time delta between them. This tuple is then converted into a hash, which serves as a fingerprint for a specific song segment.
-   **Why is an inverted index a crucial component of the Shazam architecture?** An inverted index is crucial because it allows for quick lookup of potential song IDs based on the fingerprints extracted from the user's audio clip. This allows the system to efficiently search for candidate songs that contain the user's extracted audio clip.
-   **What is sharding, and why is it necessary for Shazam's fingerprint index?** Sharding is the division of a large database or index into smaller, distributed pieces. It's necessary for Shazam's fingerprint index because the index is too large to fit on a single server, enabling parallel processing and faster queries.

### Rate Limiter Design

-   **What is the primary purpose of a rate limiter in a system?** A rate limiter's main goal is to prevent users (malicious or otherwise) from overwhelming a system with too many requests, thus maintaining stability and performance. It protects against abuse and ensures fair resource allocation.
-   **Explain the difference between rate limiting based on User ID and IP address. What are the pros and cons of each?** User ID-based rate limiting is good for signed-in users, preventing individual accounts from overloading services, but falls apart when users create multiple accounts. IP address-based rate limiting works for anonymous users but may incorrectly throttle legitimate users sharing the same IP.
-   **How can caching be used to improve the performance of a distributed rate limiter? Where in the architecture would this caching typically be implemented?** Caching reduces network calls to the rate limiter. Implementing a write-back cache on the load balancer can store the request counts of frequent users, filtering out many bad actors' requests before they ever reach the rate limiter.
-   **Explain the key difference between fixed window and sliding window rate-limiting algorithms.** Fixed window rate limiting resets the request count at fixed time intervals but allows bursts of requests at the window boundaries. Sliding window rate limiting considers a moving time window, preventing bursts and offering greater accuracy.

### Yelp Review Storage

-   **Why is partitioning important for storing Yelp reviews, even with a relatively small initial dataset?** Even with a small dataset, partitioning is important for future-proofing the system. It allows for horizontal scaling as the dataset grows, distributing the load across multiple servers to maintain performance.
-   **Explain the use of a geospatial index for location-based queries in a system like Yelp.** A geospatial index is a specialized index that optimizes queries based on geographic location. This is essential for finding nearby restaurants or businesses efficiently.
-   **Describe how denormalization can improve read performance in the Yelp review system.** Denormalization involves adding redundant data to a database table to improve read performance by reducing the need for joins. It can improve read speed, but needs to be kept in sync.
-   **Explain the trade offs with using batch processing versus Kafka with Flink.** With Kafka and Flink you get continuous reads, while batch processing only processes at a defined time. This adds a high amount of throughput but at a lower latency.

### Recommendation Engine

-   **Describe the main functional requirement for the Recommendation Engine.** The core functional requirement is to "build a recommendation engine for our service" that suggests relevant items (videos, products, etc.) to users based on their past interests.
-   **Explain the difference between batch computation and real-time recommendation approaches for building a recommendation engine.** Batch computation (traditional approach) involves offline computation of recommendations, typically running daily Spark jobs, and is simple to manage. Real-time recommendation (cutting-edge approach) focuses on generating recommendations in real-time as a user interacts with the platform and incorporates more up-to-date data.
-   **What is an embedding, and how is it used in the retrieval phase of real-time recommendation?** Embeddings are vector representations of items (words, images, products) that capture their semantic meaning. In the retrieval phase, embeddings are used to find items that are "closest" (most similar) to the embeddings of items the user has interacted with previously.
-   **Describe how Bloom filters can be used in the real-time recommendation system to improve performance and filter items that have not been seen by a particular user.** Used to quickly eliminate undesirable candidates (e.g., items the user has already seen, items with curse words on a children's account, items the user has already purchased). Bloom filters are particularly useful for blacklisting.

### Ride-Sharing Service

-   **What are the four key functionalities for a ride-sharing application that are mentioned in the video?** The four key functionalities are requesting a ride, seeing nearby drivers, matching with a driver, and providing distance and time estimations.
-   **Why are capacity estimates important in the design process for a ride-sharing service?** Crude capacity estimates help determine the order of magnitude of the system's requirements. This informs decisions about whether partitioning, replication, or other scaling strategies are necessary.
-   **What benefits are achieved with the use of a geospatial index for finding drivers?** A geospatial index breaks down the 2D plane into a hierarchy of bounding boxes (geohashes). Points close to each other in 2D space are stored next to each other on disk, providing data locality for faster range queries.
-   **What advantage can WebSockets provide over HTTP for handling real-time location updates from drivers?** WebSockets provide a real-time, bidirectional connection, reducing overhead by avoiding the need to send HTTP headers with every location update. This can be useful for improving the efficiency of server-client communications.

### Reddit Comments

-   **What are the four ways Reddit supports viewing comments?** The four ways Reddit supports viewing comments are **new** (ordered by the newest comments), **hot** (comments with the most upvotes in the last hour), **top** (comments with the most upvotes in the history of the post), and **controversial** (comments with a similar number of upvotes and downvotes).
-   **What options are available for deciding to store comments using a graph database or a normal (relational) database?** The two main options are using a **graph database** or a **normal (relational) database.**
-   **How can you achieve data locality to speed up a database that stores relational data?** With a relational database you can achieve **data locality** when doing depth first searches because everything that is actually a child is **right next to its parent** in the index.
-   **What are the benefits of ensuring derived data makes it to different comment indexes?** It is better to ensure **derived data makes it to different comment indexes** using a **Kafka queue** that is partitioned by post ID.

### Distributed Priority Queue

-   **Describe the hybrid approach combining in-memory and disk-based storage for implementing a distributed priority queue?** The system uses an in-memory data structure that stores a sorted list (heap, sorted linked list with hashmap) of node IDs, ordered by priority. Node IDs point to the actual data stored on disk (similar to a write-ahead log).
-   **What replication strategies are discussed for ensuring fault tolerance in the distributed priority queue?** Multi-leader replication is likely not a primary requirement. Leaderless replication (Quorum consistency) can provide strong consistency but may impact performance. Single-leader replication is preferred for faster read/write response times. A consensus algorithm (Paxos/Raft) with synchronous replication to a follower can provide fault tolerance in a single-leader setup.
-   **What are the two methods of modifying existing elements?** Each element on disk has an ID that is indexed in memory. Changing the priority of a node requires an in-memory update.
-   **What must consumers do when handling multiple partitions?** Consumers handling multiple partitions need to aggregate data across all partitions. Use a local heap in memory to track priorities across partitions.

### E-Commerce Payment Service

-   **What is item potency in the context of processing payments, and why is it important?** Item potency is the property of an operation that ensures it has the same effect regardless of how many times it is executed. In payment processing, it ensures that a payment is only processed once, even if the request is submitted multiple times, preventing duplicate charges.
-   **What are the guarantees does a distributed consensus algorithm provide?** Strong consistency, as it ensures that all nodes in a cluster agree on the same log of operations and is ideal for protecting against data loss.
-   **How is a two-phase commit (2PC) used to ensure that payments are item potent**? With a two-phase commit (2PC), the ledger needs to be locked before the credit card can be charged. This may be overkill and not necessary to ensure that each ledger table knows that they're the only ones charging a single credit card.
-   **What is Kafka used for?** Kafka is used for asynchronous message queues. It also guarantees that it can do at least one delivery.

### Video Upload and Streaming

-   **Describe the database schema for a system supporting video uploads and streaming, focusing on key tables.** Key Tables include Subscribers table, User Videos table, Users table, Video Comments table, and Video Chunks table. Video Chunks is a crucial table because tracks multiple encodings and resolutions for each video chunk and contains a hash to describe the chunk and the URL to where it is stored.
-   **Explain why MySQL is suitable for most tables, while Cassandra is recommended for the Video Comments table.** MySQL is Suitable for most tables due to its B-tree indexing, which is efficient for read-heavy workloads. Cassandra, recommended for the Video Comments table to handle potential high write loads for popular videos, uses LSM tree architecture where rights go to memory first and has a leaderless architecture.
-   **How are videos chunked for storage?** Videos are partitioned by video ID, and multiple encodings and resolutions are stored in each chunk.
-   **What is the flow that new data is uploaded via?** The data is uploaded via the following flow: Client -> Server -> Kafka -> Flink -> HDFS

### Typeahead Suggestions

-   **What is the primary functional goal of a typeahead suggestion system?** The system should provide quick suggestions as users type, where the suggestions can be individual words (like in text messaging) or full search terms (like in Google Search). This optimizes for read speed and efficiency.
-   **Describe a Trie data structure and how it's used in a typeahead system?** A Trie data structure is a tree-like structure where each node represents a character, allowing for efficient prefix-based searching. At each node, cache the top suggestions.
-   **How is data uploaded into the typeahead structure?** Data is uploaded via the following flow: Client -> Server -> Kafka -> Flink -> HDFS
-   **Why is caching useful?** Caching stores the top terms and with a Trie data structure and each node, the system will only have to jump down to that portion of the Trie.
