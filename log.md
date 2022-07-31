# Learning log

|Date |                                        |
|:---:|:---------------------------------------|
|     |Learnt, thoughts, progress, ideas, links|
---------------------------------------------------------
## 24 jul 22
**Redis distributed locks**
The simplest way to use Redis to lock a resource is to create a key in an instance.
The key is usually created with a limited time to live, using the Redis expires feature, so that eventually it will get released.

To acquire the lock, the way to go - SET resource_name my_random_value NX PX 30000.
The command will set the key only if it does not already exist (NX option), with an expire of 30000 milliseconds (PX option).
The "lock validity time" is the time we use as the key's time to live.
Basically the random value is used in order to release the lock in a safe way, with a script that tells Redis: remove the key only if it exists and the value stored at the key is exactly the one I expect to be.

How it works - Multiple processes execute the following redis command.
SETNX lock.foo <current Unix time + lock timeout + 1>
- If setnx returns 1, the process obtains the lock, and setnx sets to the timeout time of the lock.
- If setnx returns 0, it means that other processes have acquired the lock and cannot enter the critical area. 
(Processes can continuously try setnx operations in a loop to obtain locks);

Process is disconnected from redis after obtaining a lock - (if there is no effective mechanism to release the lock, other processes will be in a state of waiting, that is, “deadlock”);
After the lock timeout is detected, the process cannot directly and simply perform the Del delete key operation to obtain the lock.

To solve the problem that multiple processes may acquire locks at the same time:
- P1 has acquired lock.foo first, and then process P1 hangs up;
- P4 executes setnx lock.foo to try to acquire the lock;
- P1 has acquired the lock, P4 returns 0 after executing setnx lock.foo, that is, failed to acquire the lock;
- P4 executes get lock.foo to check whether the lock has expired. If not, wait for a period of time and check again;
- If P4 detects that the lock has expired, (the current time is greater than the value of key lock.foo), P4 will perform the GETSET operation;
- Because of the GETSET semantic, P4 can check if the old value stored at key is still an expired timestamp. If it is, the lock was acquired;

In order to make this locking algorithm more robust, a client holding a lock should always check the timeout didn't expire before unlocking the key with DEL.
Because client failures can be complex, not just crashing but also blocking a lot of time against some operations and trying to issue DEL after a lot of time (when the LOCK is already held by another client).


[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 13 jul 22
**Redis pipelining**
Redis pipelining is a technique for improving performance by issuing multiple commands at once without waiting for the response to each individual command.

Redis is a TCP server using the client-server model.
- The client sends a query to the server, and reads from the socket, usually in a blocking way;
- The server processes the command and sends the response back to the client;

While the client sends commands using pipelining, the server will be forced to queue the replies, using memory. It is better to send a lot of commands as batches. The speed will be nearly the same, but the additional memory used will be at most the amount needed to queue the replies for these 10k commands.

Pipelining improves the number of operations you can perform per second in a given Redis server.

StackExchange.Redis - when used concurrently by different callers, it automatically pipelines the separate requests, so regardless of whether the requests use blocking or asynchronous access, the work is all pipelined.

**Pluses**
- I can easily use StackExchange.Redis for pipelining redis requests;


[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 11 jul 22
**Redis pub/sub**
SUBSCRIBE, UNSUBSCRIBE and PUBLISH implement the Publish/Subscribe messaging paradigm. 
Rather, published messages are characterized into channels, without knowledge of what (if any) subscribers there may be. 

Subscribers express interest in one or more channels, and only receive messages that are of interest, without knowledge of what (if any) publishers there are.

A message is an array-reply with three elements:
- subscribe: means that we successfully subscribed to the channel given as the second element in the reply;
- unsubscribe: means that we successfully unsubscribed from the channel given as second element in the reply;
- message: it is a message received as result of a PUBLISH command issued by another client;
- The second element is the name of the originating channel;
- The third argument is the actual message payload;

Pub/Sub has no relation to the key space.
The Redis Pub/Sub implementation supports pattern matching.

A client may receive a single message multiple times if it's subscribed to multiple patterns matching a published message, 
or if it is subscribed to both patterns and channels matching the message.

Only connected subscribers receive messages. Every connected subscriber receives each message.
Once the message is delivered to all current subscribers, it is deleted from the channel.

If a subscriber unsubscribes (disconnects) and later subscribes to a channel again:
- It will not receive any of the intervening messages that it missed while disconnected, and
- It doesn’t know if it’s missed any messages.
- If there are no current subscribers to the channel,  the message will simply be discarded and not delivered to any subscribers.

The delivery semantics are, "at-most-once" per subscriber.
Because the message must be delivered to all current subscribers before being deleted:
- This will take longer with more subscribers.
- This is unlike a radio broadcast, which delivers content currently at the speed of light to every receiver in range.

**Pluses**
- Redis pub/sub can be used in some simple scenarios, where messages can be lost (simple websockets, games, or other little apps);

**Minuses**
- It is better to use RabbitMQ, because a lot of broker features;
- At most once delivery guarantees;
- Internally uses push notifications, can be bad for perfomance;


https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisPubSub

[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 06 jul 22
**Redis transactions**
Redis Transactions allow the execution of a group of commands in a single step.
Guarantees:
- All the commands in a transaction are serialized and executed sequentially;
- The EXEC command triggers the execution of all the commands in the transaction, so if a client loses the connection to the server in the context of a transaction before calling the EXEC command none of the operations are performed;

Transaction is entered using the MULTI command. Instead of executing next commands, Redis will queue them.
All the commands are executed once EXEC is called.

Calling DISCARD instead will flush the transaction queue and will exit the transaction.

Errors:
- Starting with Redis 2.6.5, the server will detect an error during the accumulation of commands. It will then refuse to execute the transaction returning an error during EXEC, discarding the transaction;

Rollbacks:
- Redis does not support rollbacks of transactions;

Optimistic locking:
- WATCH is used to provide a check-and-set (CAS) behavior to Redis transactions;
- Watched keys are monitored in order to detect changes against them;
- If at least one watched key is modified before the EXEC command, the whole transaction aborts;

**Pluses**
- I can use watch command to implement distributed optimistic locking;
- I can use redis transactions for some atomic batch operaitons;

**Minuses**
- StackExchange.Redis because of multiplexing uses WATCH instead of raw MULTI/EXEC commands;
- Redis does not support rollbacks of transactions;

https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisTransactions

[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 05 jul 22
**Redis persistence**
Persistence options:
- RDB, point-in-time snapshots at specified intervals;
- AOF - append only file, logs every write operation received by the server, that will be played again at server startup, reconstructing the original dataset;
- No persistence;
- RDB + AOF combination;

RDB advantages:
- compact single-file point-in-time representation, perfect for backups;
- disaster recovery, single compact file that can be transferred to far data centers;
- maximizes Redis performances since the only work the Redis parent process needs to do in order to persist is forking a child that will do all the rest;
- faster restarts with big datasets;

RDB disadvantages:
- NOT good if you need to minimize the chance of data loss in case Redis stops working;
- fork() can be time consuming if the dataset is big;

AOF advantages:
- is much more durable: different fsync policies: no fsync at all, fsync every second, fsync at every query;
fsync is performed using a background thread
-  AOF log is an append-only log, so there are no seeks, nor corruption problems if there is a power outage;
- Redis is able to automatically rewrite the AOF in background when it gets too big;
- contains a log of all the operations one after the other in an easy to understand and parse format;

AOF disadvantages:
- AOF files are usually bigger than the equivalent RDB files;
- AOF can be slower than RDB depending on the exact fsync policy;

Snapshotting:
By default Redis saves snapshots of the dataset on disk, in a binary file.
- Redis forks (child process);
- child starts to write the dataset to a temporary RDB file;

Snapshotting is not very durable. 

Log rewriting (log compaction) - while Redis continues appending to the old file, a completely new one is produced with the minimal set of operations needed to create the current data set, and once this second file is ready Redis switches the two and starts appending to the new one.

- redis forks;
- child starts writing the new base AOF in a temporary file;
- parent opens a new increments AOF file to continue writing updates;
- When the child is done rewriting the base file, the parent gets a signal, and uses the newly opened increment file;
- atomic exchange of the manifest files;

Three options, configure how many times Redis will fsync data on disk:
- always, very slow;
- everysec;
- no;

**Pluses**
- I can now choose correct persistence settings for my redis databases;


[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 1 july 22
**Redis streams**
Redis Streams are primarily an append-only data structure.
Being an abstract data type represented in memory, Redis Streams implement powerful operations to overcome the limitations of a log file.

It implements additional, non-mandatory features: a set of blocking operations allowing consumers to wait for new data added to a stream by producers, 
and in addition to that a concept called Consumer Groups.

Every new added entry ID will be monotonically increasing, so in more simple terms, every new entry added will have a higher ID compared to all the past entries.
Because the ID is related to the time the entry is generated, this gives the ability to query for time ranges basically for free.

Reading modes:
- get messages by ranges of time;
- stream of messages that can be partitioned to multiple consumers that are processing such messages;

XREAD - subscribe to new items arriving to the stream.
- A stream can have multiple clients (consumers) waiting for data;
- Can access multiple streams at once;


https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisStreams

**Pluses**
- I can now choose correct redis data structure for my purposes;
- I can use redis sets to implement relations model;

**Minuses**
- StackExchange.Redis doesn't support xread blocking command because of multiplexing...;

[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 30 june 22
**Redis sorted sets**
Elements inside sorted sets are not ordered, every element in a sorted set is associated with a floating point value, called the score.

Elements are ordered according to the following rule:
- different scores, A > B if A.score is > B.score;
- same score,  A > B if the A string is lexicographically greater than the B string.

Sorted sets are implemented via a dual-ported data structure containing both a skip list and a hash table, so every time we add an element Redis performs an O(log(N)) operation.

Redis 2.8, a new feature was introduced that allows getting ranges lexicographically.

Sorted sets scores can be updated at any time. 
Just calling ZADD against an element already included in the sorted set will update its score (and position) with O(log(N)) time complexity. 
As such, sorted sets are suitable when there are tons of updates.


Because of this characteristic a common use case is leader boards.

https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisSortedSet

**Pluses**
- I can use redis sets and sorted set to implement relations model;
- I can use redis sorted set to some interesting queries for sorted data structures;


[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 29 june 22
**Redis sets**
Redis' responsibility to delete keys when data types are left empty, or to create an empty data type if the key does not exist and we are trying to add elements to it.

Sets are good for expressing relations between objects. For instance we can easily use sets in order to implement tags.
A simple way to model this problem is to have a set for every object we want to tag. 
The set contains the IDs of the tags associated with the object.

https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisSet


[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 28 june 22
**Redis lists**
From a very general point of view a List is just a sequence of ordered elements.
Lists are implemented via Linked Lists.
Lists can be taken at constant length in constant time.
When fast access to the middle of a large collection of elements is important, sorted sets can be used.

Common use cases:
- Remember the latest updates posted by users into a social network;
- Communication between processes, using a consumer-producer pattern where the producer pushes items into a list, and a consumer (usually a worker) consumes those items and executed actions;

Capped lists - In many use cases we just want to use lists to store the latest items, whatever they are: social network updates, logs, or anything else.
Redis allows us to use lists as a capped collection, only remembering the latest N items and discarding all the oldest items using the LTRIM command. (Technically O(n)).

Lists have a special feature that make them suitable to implement queues.
Redis implements commands called BRPOP and BLPOP which are versions of RPOP and LPOP able to block if the list is empty.

StackExchange Redis uses multiplexing, which means it only maintains a single connection with the redis server. 
When there are concurrent requests, it will automatically use the pipeline to send each request, and each request needs to wait until the previous request is completed.
For this reason, StackExchange Redis does not provide the corresponding api for BRPOP/BLPOP, because these two operations are likely to block the entire Mulitplexer.

https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisList

**Pluses**
- I can implement simple pub/sub queues by using redis lists;

**Minuses**
- StackExchange.Redis does not support BRPOP operations.


[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 27 june 22
**Redis data types**
Strings - binary safe, can contain any kind of data (max value 512mg).

Sets - unordered collection of Strings (desirable property of not allowing repeated members).

Hashes - are maps between string fields and string values.

Sorted Sets - difference with Set is that every member of a Sorted Set is associated with a score, that is used keep the Sorted Set in order (elements in order, fast existence test, fast access to elements in the middle)
- Retreiving data by ranges.

Bitmaps - handle String values like an array of bits; (realtime analytics,Storing space efficient);

HyperLogLogs - probabilistic data structure used in order to count unique things.

Streams - append only collections.

https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisDataTypes

[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 22 june 22
**Caching strategies**
Two common approaches

Cache-aside or lazy loading (a reactive approach), cache is updated after the data is requested.
The cache contains only data that the application actually requests, which helps keep the cache size cost-effective;
Problem - Overhead is added to the initial response time because additional roundtrips to the cache and database are needed.

Write-through (a proactive approach), updated immediately when the primary database is updated. 

With both approaches, the application is essentially managing what data is being cached and for how long.
Data will be found in the cache, on next access. Performance of database is optimal because fewer reads are performed.
Problem - infrequently-requested data is also written to the cache;

Concurrency:
Optimistic - application checks if data in the cache has changed since it was retrieved (infrequent updates)
Pessmistic - when retrieveing data, the application locks it in the cache to prevent another instance from changing it;

Key Expiration:
Too short Key expiration will make objects to expires sooner. 
Too long will make you used stale old data producing issues.

**Pluses**
- I can use redis as a distributed cache in my future projects;

**Minuses**
- I can't use client side caching good new mechanics for improving my cache strategies in .NET projects now;

https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisCachingAdvanced

[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 21 june 22
**Redis Caching**
two key advantages of client-side caching:
- Data is available with a very small latency;
- database system receives less queries;

Redis client-side caching support is called Tracking, two modes:
- default mode, the server remembers what keys a given client accessed, and sends invalidation messages when the same keys are modified;
- broadcasting mode, clients subscribe to key prefixes such as object: or user:, and receive a notification message every time;

The server remembers the list of clients that may have cached a given key in a single global Invalidation Table.
Inside the invalidation table just store client IDs (each Redis client has a unique numerical ID).

Using the new version of the Redis protocol, RESP3, supported by Redis 6, it is possible to run the data queries and receive the invalidation messages in the same connection.

Broadcasting mode:
- Clients enable client-side caching using the BCAST option;
- Instead of invalidation table, it uses a different Prefixes Table, where each prefix is associated to a list of clients;
- No two prefixes can track overlapping parts of the keyspace (foo, foob);
- Every time a key matching any of the prefixes is modified, all the clients subscribed to that prefix, will receive the invalidation message;

NOLOOP - using this option, clients are able to tell the server they don't want to receive invalidation messages for keys that they modified.

Race conditions:
When implementing client-side caching redirecting the invalidation messages to a different connection, you should be aware that there is a possible race condition.

**Pluses**
- StackExchange.Redis has async methods support, i will use it my new projects.

**Minuses**
- StackExchange.Redis doesnt support client-side caching modifications..

https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisCachingAdvanced

[Log Index]
----------------------------------------------------------
---------------------------------------------------------
## 20 june 22
**Redis Introduction**
Redis - in-memory data structure store used as a database, cache, message broker, and streaming engine.
has built-in replication, transactions, and different levels of on-disk persistence.

works with an in-memory dataset.

Can persist your data either by periodically dumping the dataset to disk or by appending each command to a disk-based log.

Redis accepts clients connections on the configured TCP port.
- Socket is put in the non-blocking state;
- TCP_NODELAY is set to ensure that there are no delays;

When Redis can't accept a new client connection because the maximum number clients (max_clients = 10000 by default) it will send an error.

Processing clients - Once new data is read from a client, all the queries contained in the current buffers are processed sequentially.


https://github.com/zolotarevandrew/databases/tree/main/redis/NetRedis/RedisCaching

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 27 may 22
**REST best practices**
REST - Representational State Transfer. software architectural style.

Any API (Application Programming Interface) that follows the REST design principle is said to be RESTful.

Best practices:
- Use JSON as the Format for Sending and Receiving Data;
- Use Nouns Instead of Verbs in Endpoints, Instead, it should be something like this: /posts;
- Name Collections with Plural Nouns. You can think of the data of your API as a collection of different resources from your consumers: /posts/123;
- Use Status Codes in Error Handling;
- Use Nesting on Endpoints to Show Relationships, posts/postId/comments, avoid nesting that is more than 3 levels deep as this can make the API less elegant and readable;
- Use SSL for Security;
- Be Clear with Versioning, https://mysite.com/v1;
- Provide Accurate API Documentation, use swagger;

**Pluses:**
- I can now use correct naming and http error codes in my projects to improve restful api quality;

**Minuses**
- There a lot misconceptions about how to implement REST;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 26 may 22
**GraphQL Queries**
GraphQL is about asking for specific fields on objects.

**Pluses**
- I can correctly use get/join/mutate options in apps using graphql;

**Minuses**
- If using EntityFramework, some queries can affect perfomance, so every call should be checked and SQL logged;


https://github.com/zolotarevandrew/protocols/tree/main/graphQL/SimpleApi

[Log Index]
----------------------------------------------------------
## 26 may 22
**Kafka consumers**

A consumer group is a set of consumers which cooperate to consume data from some topics. 
The partitions of all the topics are divided among the consumers in the group. 
As new group members arrive and old members leave, the partitions are re-assigned so that each member receives a proportional share of the partitions - rebalancing the group.

The coordinator of each group is chosen from the leaders of the internal offsets topic __consumer_offsets (which is used to store committed offsets).

When the consumer starts up, it finds the coordinator for its group and sends a request to join the group.

Each consumer must send heartbeats to the coordinator. If no hearbeat is received before expiration of the configured session timeout, then the coordinator will kick the member out of the group.

Offsets:

When the group is first created, the position is set according to a configurable offset reset policy (auto.offset.reset).
Consumption starts either at the earliest offset or the latest offset.

The offset commit policy providing the message delivery guarantees. By default, the consumer is configured to use an automatic commit policy, which triggers a commit on a periodic interval.

Processing guarantees:
- At-least-once, records are never lost but may be redelivered. (At-least-once semantics is enabled by default (processing.guarantee="at_least_once);
- Exactly-once, records are processed once. Even if a producer sends a duplicate record, it is written to the broker exactly once (processing.guarantee="exactly_once_v2").
write is not considered successful until it is acknowledged, and a commit is made to “finalize” the write;

By default, the consumer is configured to auto-commit offsets.

When auto-commit disabled and commit uses sync versions, this may reduce overall throughput since the consumer might otherwise be able to process records while that commit is pending.
Solution - the consumer has a configuration setting fetch.min.bytes which controls how much data is returned in each fetch.

The problem with asynchronous commits is dealing with commit ordering. By the time the consumer finds out that a commit has failed, you may already have processed the next batch of messages and even sent the next commit.


if the last commit fails before a rebalance occurs or before the consumer is shut down, then offsets will be reset to the last commit and you will likely see duplicates.

Rebalance:
Each rebalance has two phases: partition revocation and partition assignment.


Configuration:
- session.timeout.ms, default 10s;
- heartbeat.interval.ms, this controls how often the consumer will send heartbeats to the coordinator;
- max.poll.interval.ms, the maximum time allowed time between calls to the consumers poll method, default 300s;
- enable.auto.commit, default true;
- auto.commit.interval.ms, default 5s;
- auto.offset.reset, default latest;


**Pluses**
- i can use kafka consumers to reread my prev messages in my apps (create new app version, or something another);
- I can use kafka consumers as simple message consumers with autocommit enabled in my apps, with at-least-once or exactly-once delivery guarantees;
- I can use kafka consumers to provide at-most-once delivery guarantee with autocommit disabled, and committing before consuming in my apps;

**Minuses**
- I should very careful with, async and sync commits, kafka has a lot login upon it;

https://github.com/zolotarevandrew/kafka/tree/main/Consumers

[Log Index]
----------------------------------------------------------
## 25 may 22
**GraphQL**
A GraphQL service is created by defining types and fields on those types, then providing functions for each field on each type.
Every field and nested object can get its own set of arguments, making GraphQL a complete replacement for making multiple API fetches.

Each GraphQL query passes through three phases: parse, validate and execute.

Schema - endpoint provides a schema used to inform API consumers about the functionality available for clients to consume.

Has three primary operations: 
- Query for reading data;
- Mutation for writing data, used to add, modify, or delete data;
- Subscription for receiving real-time updates, allow a server to send data to its clients, notifying them when events occur.

**Pluses**
- I can use graphql in my apps, which has complex queries, such as mobile apps or other.
- I can use graphql to reduce bandwidth for my future apps;

**Minuses**
- It is difficult to implement caching or rate-limiting;
- A little bit complex;

https://github.com/zolotarevandrew/protocols/tree/main/graphQL/SimpleApi

[Log Index]
----------------------------------------------------------
## 24 may 22
**GRPC streams**
.Net 6 has load balancing mechanism for GRPC and also HTTP3 support was added, but http3 is draft in RFC.

**Pluses**
- I can use streams with async enumerable in .net projects for batch operations;

**Minuses**
- Streams have problems with errors, if there are 2 messages sent and 3 failed, server receive error and stream process will fail;

https://github.com/zolotarevandrew/protocols/tree/main/gRPC/Streams

[Log Index]
----------------------------------------------------------
## 24 may 22
**Kafka topics**
Topics are partitioned, meaning a topic is spread over a number of "buckets" located on different Kafka brokers. This distributed placement of your data is very important for scalability because it allows client applications to both read and write the data from/to many brokers at the same time. When a new event is published to a topic, it is actually appended to one of the topic's partitions. Events with the same event key (e.g., a customer or vehicle ID) are written to the same partition, and Kafka guarantees that any consumer of a given topic-partition will always read that partition's events in exactly the same order as they were written.

Properties:
cleanup.policy - delete or compact, based on log retention;
compression.type - standard compression codecs on uncompressed;
delete.retention.ms - the amount of time to retain delete tombstone markers for log compacted topics. (default 1 day);
file.delete.delay.ms -  time to wait before deleting a file from the filesystem; (default 1min);
flush.messages - calling fsync to prevent os caching, recommended not to use;
index.interval.bytes - how frequently Kafka adds an index entry to its offset index, recommended not to use;
max.message.bytes - the largest record batch size allowed by Kafka;
message.timestamp.difference.max.ms - maximum difference allowed between the timestamp when a broker receives a message and the timestamp specified in the message. CreateTime, a message will be rejected if the difference in timestamp exceeds this threshold. ignored if message.timestamp.type=LogAppendTime;
message.timestamp.type - CreateTime or LogAppendTime;
retention.bytes - the maximum size a partition can grow to before discarding old log segments to free up space;
retention.ms - maximum time retaining a log before discarding old log segments to free up space;
segment.bytes - the segment file size for the log (default 1gb);
segment.index.bytes - size of the index that maps offsets to file positions (default 10mb);
message.downconversion.enable - down-conversion of message formats is enabled to satisfy consume request (broker will not perform down-conversion for consumers expecting an older message format. The broker responds with UNSUPPORTED_VERSION error for consume requests from such older clients);


**Pluses**
- I can use correct properties to create topic by my apps requirements;
- I can use topics to scale my apps correctly using partitions;
- I can use topics to increase my apps reliability;

**Minuses**
- I think i will not use, most of all topic properties in my apps;

https://github.com/zolotarevandrew/kafka/tree/main/Topics

[Log Index]
----------------------------------------------------------
## 23 may 22
**GRPC Introduction**
gRPC is a language agnostic, high-performance Remote Procedure Call (RPC) framework.
main benefits:
- Contract-first API development, using Protocol Buffers by default, allowing for language agnostic implementations.
- Supports client, server, and bi-directional streaming calls.
- Reduced network usage with Protobuf binary serialization.

By default, uses Protocol Buffers, open source mechanism for serializing structured data.

Service definitions:
- Unary RPCs where the client sends a single request to the server and gets a single response back;
- Server streaming RPCs where the client sends a request to the server and gets a stream to read a sequence of messages back (gRPC guarantees message ordering within an individual RPC call);
- Client streaming RPCs where the client writes a sequence of messages and sends them to the server, again using a provided stream;
- Bidirectional streaming RPCs where both sides send a sequence of messages using a read-write stream;

Unary RPC:
- client calling stub;
- server is notified that the RPC has been invoked with the client’s metadata;
- server can send back its initial metadata, or wait for the client’s request message;
- Once the server has the client’s request message, it does whatever work is necessary to create and populate a response;

Timeouts - DEADLINE_EXCEEDED property;

In gRPC, both the client and server make independent and local determinations of the success of the call, and their conclusions may not match.

**Minuses**
- I should always copy or create library to share protobuf contracts for correct sending and receiving;

https://github.com/zolotarevandrew/protocols/tree/main/gRPC/Simple

[Log Index]
----------------------------------------------------------
## 18 may 22
**QUIC**
A UDP-Based Multiplexed and Secure Transport.

Integrates the TLS handshake [TLS13], although using a customized framing for protecting packets, structured to permit the exchange of application data as soon as possible.

Authenticates the entirety of each packet and encrypts as much of each packet as is practical.
QUIC packets are carried in UDP datagrams.

Application protocols exchange information over a QUIC connection via streams, which are ordered sequences of bytes.

Two types of streams can be created: 
- bidirectional streams, which allow both endpoints to send data;
- unidirectional streams, which allow a single endpoint to send data.  

Flow control:
To enable a receiver to limit memory commitments for a connection, streams are flow controlled both individually and across a connection as a whole.

Data flow control:
- Stream flow control, limiting data in each stream;
- Connection flow control, limiting data in all streams;

An endpoint limits the cumulative number of incoming streams a peer can open.

A QUIC connection is shared state between a client and a server.
Each connection starts with a handshake phase, during which the two endpoints establish a shared secret using the cryptographic handshake protocol.

QUIC relies on a combined cryptographic and transport handshake to minimize connection establishment latency.
Uses the CRYPTO frame to transmit the cryptographic handshake.



https://datatracker.ietf.org/doc/rfc9000/

[Log Index]
----------------------------------------------------------
## 17 may 22
**IP**
The internet protocol provides for transmitting blocks of data called datagrams from sources to
destinations, where sources and destinations are hosts identified by fixed length addresses.

Implements two basic functions: addressing and fragmentation.

Uses four key mechanisms in providing its service:  
Type of Service - quality of the service desired. To select the actual transmission parameters for a particular network.
Time to Live - set by the sender of the datagram and reduced at the points along the route where it is processed.
Options - provisions for timestamps, security, and special routing.
Header Checksum - verification that the information used in processing internet datagram has been transmitted correctly.

Errors detected may be reported via the Internet Control Message Protocol (ICMP).

Transmitting:
- Prepare data;
- Prepare datagram header and attach data to it;
- Local network interface creates a local network header, and send datagram to local network;
- Local network strips this header and adding from address;
- Sending to destination host;


The internet module maps internet addresses to local net addresses.
An address begins with a network number, followed by local address (called the "rest" field).

There are three formats or classes of internet
- class a, the high order bit is zero, the next 7 bits are the network, and the last 24 bits are the local address; 
- class b, the high order two bits are one-zero, the next 14 bits are the network and the last 16 bits are the local address;
- class c, the high order three bits are one-one-zero, the next 21 bits are the network and the last 8 bits are the local address.

Some hosts will also have several physical interfaces (multi-homing).

Fragmentation:
The receiver of the fragments uses the identification field to ensure that fragments of different datagrams
are not mixed.  The fragment offset field tells the receiver the position of a fragment in the original datagram.  The fragment offset and length determine the portion of the original datagram covered by this fragment.

To fragment a long internet datagram it creates two new internet datagrams and copies the contents of the internet header fields from the long datagram into both new internet headers.

Header format:
- Version 4 bits, format version 4;
- IHL 4 bits - header length;
- Type of service 8 bits - service quality desired;
- Total length 16 bits - in octets, recommended 576 octets, to remove fragmentation;
- Identification 16 bits - assembling fragments;
- Flags 3 bit - don't fragment, fragment;
- TTL 8 bits - every receive decreases by one.
- Header checksum - checksum;
- Source address 32 bits;
- Dest address 32 bits;

https://datatracker.ietf.org/doc/html/rfc791

[Log Index]
----------------------------------------------------------
## 16 may 22
**UDP**
UDP  is  defined  to  make  available  a datagram   mode  of  packet-switched   computer   communication  in  the environment  of  an  interconnected  set  of  computer  networks.
Provides  a procedure  for application  programs  to send messages  to other programs  with a minimum  of protocol mechanism.
Protocol  is transaction oriented, and delivery and duplicate protection are not guaranteed.

Header Format:
- source port - optional field. If not used, a value of zero is inserted;
- destination port - context of particular dest address;
- length - length in octets including header and data;
- checksum - 16 bit sum of a pseudo header of information from the IP header, the UDP header, and the
data;

The pseudo  header  conceptually prefixed to the UDP header contains the source  address,  the destination  address,  the protocol,  and the  UDP length.
- source address;
- dest address;
- protocol;
- udp length;


In .NET 5, the runtime added the concept of a Pinned Object Heap (POH) which is an area designed to hold buffers intended for native IO operations; it was initially needed for improving the performance of sockets in ASP.NET Core HTTP request handling.


https://github.com/zolotarevandrew/protocols/tree/main/Udp

https://datatracker.ietf.org/doc/html/rfc768
https://enclave.io/high-performance-udp-sockets-net6/

[Log Index]
----------------------------------------------------------
## 13 may 22
**HTTP 3**
HTTP/3 runs over QUIC – an encrypted general-purpose transport protocol that multiplexes multiple streams of data on a single connection.

QUIC - The protocol utilizes space congestion control over User Datagram Protocol (UDP).

QUIC will help fix some of HTTP/2's biggest shortcomings:
- Developing a workaround for the sluggish performance when a smartphone switches from WiFi to cellular data;
- Decreasing the effects of packet loss — when one packet of information does not make it to its destination, it will no longer block all streams of information (“head-of-line blocking”);

Benefits:
- Faster connection establishment: QUIC allows TLS version negotiation to happen at the same time as the cryptographic and transport handshakes;
- Zero round-trip time, for servers they have already connected to, clients can skip the handshake requirement;
- More comprehensive encryption: QUIC’s new approach to handshakes will provide encryption by default — a huge upgrade from HTTP/2 — and will help mitigate the risk of attacks;


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 12 may 22
**HTTP 2**
- Multiple requests (html,css,js) in one tcp connection.
- compresses a lot of unnecessary header frames by HPACK.

Binary protocol:
Binary commands will be more difficult to read than subsequent text commands, but the network will easily generate them and parse frames.
- less network load;
- reduced network latency and increase throughput

Changes how the data is exchanged between the client and server.
- Stream: A bidirectional flow of bytes within an established connection, which may carry one or more messages.
- Message: A complete sequence of frames that map to a logical request or response message.
- Frame: The smallest unit of communication, containing a frame header.

Each stream has a unique identifier and optional priority information that is used to carry bidirectional messages.

Each message is a logical HTTP message, such as a request, or response, which consists of one or more frames.

Allows each stream to have an associated weight and dependency:
- Each stream may be assigned an integer weight between 1 and 256.
- Each stream may be given an explicit dependency on another stream.

The combination of stream dependencies and weights allows the client to construct and communicate a "prioritization tree" that expresses how it would prefer to receive responses.

Provides a set of simple building blocks that allow the client and server to implement their own stream- and connection-level flow control.
- Flow control is directional. Each receiver may choose to set any window size that it desires for each stream and the entire connection;
- Flow control is credit-based. Each receiver advertises its initial connection and stream flow control window;
- Flow control cannot be disabled;

Server push - ability of the server to send multiple responses for a single client request.

*Pluses*
- I can switch to HTTP2 to reduce number of TCP connection, and reduce latency.

*Minuses*
- There are the head of blocking problem with HTTP2;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 11 may 22
**HTTP 1**
HTTP is an application layer protocol designed within the framework of the Internet protocol suite. Its definition presumes an underlying and reliable transport layer protocol.

HTTP resources are identified and located on the network by Uniform Resource Locators (URLs), using the Uniform Resource Identifiers (URI's).

In HTTP/1.0 a separate connection to the same server is made for every resource request.
In HTTP/1.1 instead a TCP connection can be reused to make multiple resource requests.

Persistent connections:

HTTP/1.1 has keep-alive-mechanism, so that a connection could be reused for more than one request/response. Such persistent connections reduce request latency, because the client does not need to re-negotiate the TCP handshake. TCP can put multiple requests (and responses to requests) int one TCP segment;

Since requests are pipelined, TCP segments are more efficient. The overall result is less Internet traffic and faster performance for the user

HTTP1.1 introduces the OPTIONS method, way for a client to learn about the capabilities of a server without actually requesting a resource.

Сaching:
A cache entry is fresh until it reaches its expiration time, at which point it becomes stale.
Uses the more general concept of an opaque cache validator string, known as an entity tag.

The most basic is If-None-Match, which allows a client to present one or more entity tags from its cache entries for a resource. If none of these matches the resource’s current entity tag value, the server returns a normal response; otherwise, it may return a 304 (Not Modified) response with an ETag header that indicates which cache entry is currently valid.

Adds the new Cache-Control header, uses relative expiration times, via the max-age directive.

Range requests allow a client to request portions of a resource. A client makes a range
request by including the Range header in its request, specifying one or more contiguous ranges of bytes.

Includes a new status code, 100 (Continue), to inform the client that the request body should be transmitted. When this mechanism is used, the client first sends its request headers, then waits for a response.

Resolves the problem of delimiting message bodies by introducing the Chunked transfer-coding. The sender breaks the message body into chunks of arbitrary length, and each chunk is sent with its length prepended; it marks the end of the message with a zero-length chunk. The sender uses the Transfer-Encoding: chunked header to signal the use of chunking.

*Pluses*
- I can use http1.1 caching correctly in my next projects.
- I can use http1.1 keep alive mechanism correctly to decrease latency in my next projects.

https://www.ra.ethz.ch/cdstore/www8/data/2136/pdf/pd1.pdf


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 09-10 may 22
**TCP**
TCP provides reliable, ordered, and error-checked delivery of a stream of octets (bytes) between applications running on hosts communicating via an IP network.

Three-way handshake (active open), retransmission, and error detection adds to reliability but lengthens latency.

TCP achieves reliability using a technique known as positive acknowledgement with re-transmission. 
This requires the receiver to respond with an acknowledgement message as it receives the data. 
The sender keeps a record of each packet it sends and maintains a timer from when the packet was sent. 
The sender re-transmits a packet if the timer expires before receiving the acknowledgement. 
The timer is needed in case a packet gets lost or corrupted.

TCP protocol operations may be divided into three phases
1) Connection establishment is a multi-step handshake process
2) data transfer phase
3) connection termination closes the connection and releases all allocated resources

- Before a client attempts to connect with a server, the server must first bind to and listen at a port to open it up for connections: this is called a passive open.
- Connection termination, client transmits a FIN and ACK. After the side that sent the first FIN has responded with the final ACK, it waits for a timeout before finally closing the connection, 
during which time the local port is unavailable for new connections.

Resources:

Whenever a packet is received, the TCP must perform a lookup on table to find the destination process. 
Each entry in the table is known as a Transmission Control Block or TCB. 
It contains information about the endpoints (IP and port), status of the connection, running data about the packets that are being exchanged and buffers for sending and receiving data.

Reliable transmission:
TCP uses a sequence number to identify each byte of data.
Reliability is achieved by the sender detecting lost data and retransmitting it.

Dupack-based retransmission:
Hence the receiver acknowledges packet again on the receipt of another data packet. This duplicate acknowledgement is used as a signal for packet loss. If the sender receives three duplicate acknowledgements, it retransmits the last unacknowledged packet.

Timeout-based retransmission:
When a sender transmits a segment, it initializes a timer with a conservative estimate of the arrival time of the acknowledgement. The segment is retransmitted if the timer expires, with a new timeout threshold of twice the previous value, resulting in exponential backoff behavior.

To assure correctness a checksum field is included.

Flow control:

TCP uses a sliding window flow control protocol. In each TCP segment, the receiver specifies in the receive window field the amount of additionally received data (in bytes) that it is willing to buffer for the connection. The sending host can send only up to that amount of data before it must wait for an acknowledgement and receive window update from the receiving host.

When a receiver advertises a window size of 0, the sender stops sending data and starts its persist timer.

Congestion control:

Acknowledgments for data sent, or lack of acknowledgments, are used by senders to infer network conditions between the TCP sender and receiver. Coupled with timers, TCP senders and receivers can alter the behavior of the flow of data.

Maximum segment size:

For best performance, the MSS should be set small enough to avoid IP fragmentation, which can lead to packet loss and excessive retransmissions.


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 05 may 22
**Kafka introduction**
Event streaming is capturing data in realtime from databases and other resources in the form of stream of events, for later retrieval, routing to 
different sources as needed.

kafka using tcp Connections for server and client.

Servers:
Cluster one more servers. Some servers forms the storage layer - brokers. Other server runs kafka connect to  import export data to integrate with other systems.

Events are durably stored in topics.
Topic can have zero or more producers and consumers. Events are not deleted after consumption.

Topics are partitioned, spread over number Of buckets located in different brokers.

Use cases:
- messaging, better than traditional rabbit, throughput, partitioning, replication, fault tolerance.
- Website activity tracking - original use case
- Metrics
- Log aggregation
- Stream processing
- Commit log

*Pluses*
- I can use kafka for projects which needed realtime processing, because it is faster than RabbitMq.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 04 may 22
**POP3/IMAP**
Popular email providers supports both protocols.

POP - receive emails from a remote server and send to a local client. "store-and-forward" service.
Once the email is on the client, POP3 then deletes it from the server.
Users or an administrator can specify that mail be saved for some time, allowing users to download email as many times as they wish within the specified period.

Port 110 - non encrypted connection.
Port 995 - encrypted connection.

The server starts POP3 service by listening on TCP port 110. 
When a client wishes to use POP3 for email retrieval, it establishes a TCP connection with the server host. Once this connection is established, the POP3 server sends a greeting. At this point, the session enters the authorization state.

When the client issues the quit command, the session enters the update state. The POP3 server releases any resources acquired during the transaction state, and says "goodbye," which is when the TCP connection is closed. After the POP3 session enters the update state, the POP3 server deletes the message.

Pluses
- POP3 useful then users need to accesc ther email offline, and for sending and storing bulk email messages;

Minuses
- POP3 not intent to support sync with server.

IMAP - stores email messages on a mail server and enables the recipient to view and manipulate them as thoug they were stored locally on their device(s).

Enables users to organize messages into folders, flag messages for urgency or follow-up, and save draft messages on the server. Users can also have multiple email client applications that sync with the email server to consistently show which messages have been read or are still unread.

- User sign in to email client, client contact server using IMAP;
- TCP connection is made;
- The headers of all emails are displayed by the email client;
- IMAP only downloads a message to the client when the user clicks on it; attachments are not automatically downloaded.
- Email messages remain on the server unless the user explicitly deletes them;

Port 143 - non encrypted connection.
Port 993 - encrypted connection.

With POP3, email is saved for users in a single mailbox on the server. It is moved from the server to their device when the mail client opens.

Pluses
- emails accessible from multiple devices
- a single mailbox can be shared by multiple users
- users can organize emails on the server by creating folders and subfolders


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 04 may 22
**SMTP/ESMTP**
E-Mailing system has three main parts:
- user agents (browsers, mobile devices and other)
- email servers
- SMTP protocol

User agents - read, answer, resend, send emails.
When user creates new email, agent will send it to email server (queue).

SMTP uses TEXT commands, which is not encrypted.
SMTP requires 7-bit ASCII symbols for every message header. 

Because of SMTP message can contain video images and other data.
Content-Type and Content-Tranfer-Encoding headers have to presented.

First SMTP client creates TCP connection by port 25,587 of email server. 
When client and server do handshake.

Agent delivers message to a transfer agent - TA. 
TA uses DNS to look up the MX (mail exchanger) record for the recipient's domain.
Then it selects a recipient server and connects to it to complete the mail exchange.

Once delivered to the local mail server, the mail is stored for batch retrieval by authenticated mail clients.

SMTP AUTH extension
Usually, servers reject RCPT TO commands that imply relaying unless authentication credentials have been accepted.

Encryption:
Port 587 is often used to encrypt SMTP messages using STARTTLS, which allows the email client to establish secure connections by requesting that the mail server upgrade the connection through TLS.



[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 03 may 22
**DNS**
Dns - hierarhical  distributed database and also application protocol for interaction between hosts and servers

Dns uses 53 port and work over UDP.
Http ftp and other protocol uses dns to get ip addresses.

Functions:
- Host aliases by its canonical name (eu1.host.eu - host.eu)
- Mail aliases 
- Load balancing, if web has replicas with different ip addresses, dns can contain list of ip addresses

Why not centralized?
- Single Point of failure
- Traffic volume
- Remoteness
- Service

Local dns servers - for every internet provider
If provider owns searching host it will give fast answer

Root dns servers - base dns servers about 10 worldwide.

If local dns server cant find host, it sends request to root server and becoming a client. If root server cant find host it sends address of Plenipotentiary server.

Plenipotentiary dns server - server where host has been registered.

Dns requests can be iterative of recursive.

Caching - servers  caches dns Responses

Dns servers store resource records
- Name
- Value
- Type
- TTL
Type A - Name host name, Value ip address
Type Ns - name domain, value host name or dns server
Type CNAME - value canonical name, name alias
Type MX - value canonical name, name alias

*Pluses*
- I can use correct DNS records and its properties in my DNS providers later.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 02 may 22
**FTP**
ftp has control TCP connection and data TCP connections.
Uses authentication based on text.

Passive mode:
- Then client is behind firewall, client uses the control connection to send PASV command to server, and then receives server port and ip address to establish connection.

Active mode:
- clients starts listening for incoming data connections from the server.

FTPS -e xtension to FTP, that adds support TLS.

Implicit: 
- Negotiation is not supported with implicit FTPS configurations. A client is immediately expected to challenge the FTPS server with a TLS ClientHello message. If such a message is not received by the FTPS server, the server should drop the connection.

Explicit:
- FTPS client must "explicitly request" security from an FTPS server and then step up to a mutually agreed encryption method. If a client does not request security, the FTPS server can either allow the client to continue in insecure mode or refuse the connection.

**Pluses**
- I can use FTPS server in my simple case scenarios.

**Minuses**
- FTP deprecated in latest browser versions.
- FTP not secure;

Tried ftps by local IIS - https://www.pcwdld.com/install-secure-ftp-server-using-iis
and filezilla client.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 27 apr 22
**Rabbitmq Streams**
Streams cover 4 use-cases that queue types can not provide:
- Large fan-outs
Users have to bind a dedicated queue for each consumer. If the number of consumers is large this becomes potentially inefficient. Streams will allow any number of consumers to consume the same messages from the same queue in a non-destructive manner. Stream consumers will also be able to read from replicas allowing read load to be spread across the cluster.

- Replay, time-travelling
Streams will allow consumers to attach at any point in the log and read from there.

- Throughput Performance
No persistent queue types are able to deliver throughput that can compete with any of the existing log based messaging systems. Streams have been designed with performance as a major goal.

- Large logs
Streams are designed to store larger amounts of data in an efficient manner with minimal in-memory overhead.

Any consumer can start reading/consuming from any point in the log.

As streams persist all data to disks before doing anything it is recommended to use the fastest disks possible.

*Minuses*
- .NET has a raw library for it;
- Kafka can be a better solution with distributed log type such as rabbit streams;


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 27 apr 22
**Rabbitmq Priority Queues**
RabbitMq has priority queues, by using x-max-priority optional queue argument.
By default, consumers may be sent a large number of messages before they acknowledge any, limited only by network backpressure.

*Pluses*
- I can use message priorities and priority queues for some load-balancing tasks or somethis that has priorities.

*Minuses*
- There is bad documentation for consumer priorities, i need to test it more.

https://github.com/zolotarevandrew/rabbitmq/tree/main/priority-consumers
https://github.com/zolotarevandrew/rabbitmq/tree/main/priority-queues

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 26 apr 22
**Rabbitmq Consumers**
Сonsumer is a subscription for message delivery that has to be registered before deliveries begin and can be cancelled by the application.
A successful subscription operation returns a subscription identifier (consumer tag). It can later be used to cancel the consumer.

When registering a consumer applications can choose one of two delivery modes:
- Automatic
- Manual

With manual acknowledgement mode consumers have a way of limiting how many deliveries can be "in flight" by using prefetch count.

The exclusive flag can be set to true to request the consumer to be the only one on the target queue.
Consuming with only one consumer is useful when messages must be consumed and processed in the same order they arrive in the queue.

Single active consumer "x-single-active-consumer", allows to have only one consumer at a time consuming from a queue and to fail over to another registered consumer in case the active one is cancelled or dies.

Priorities - when consumer priorities are in use, messages are delivered round-robin if multiple active consumers exist with the same high priority.

.NET clients guarantee that deliveries on a single channel will be dispatched in the same order there were received regardless of the degree of concurrency. 

*Pluses*
- I can use exclusive consumers for web sockets or other transient connections in my projects;
- I can use single active consumer to support correct message ordering;

*Minuses*
- I think i will not use automatic acks, because of message loss possibility.


https://github.com/zolotarevandrew/rabbitmq/tree/main/consumers

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 26 apr 22
**Rabbitmq WorkQueues**
By default, RabbitMQ will send each message to the next consumer, in sequence. 
On average every consumer will get the same number of messages. (Round robin).

Interesting note:
"Marking messages as persistent doesn't fully guarantee that a message won't be lost. Although it tells RabbitMQ to save the message to disk, there is still a short time window when RabbitMQ has accepted a message and hasn't saved it yet. Also, RabbitMQ doesn't do fsync(2) for every message -- it may be just saved to cache and not really written to the disk. The persistence guarantees aren't strong, but it's more than enough for our simple task queue. If you need a stronger guarantee then you can use publisher confirms."

RabbitMQ dispatches a message when the message enters the queue. It doesn't look at the number of unacknowledged messages for a consumer. It just blindly dispatches every n-th message to the n-th consumer. To defeat this problem it is better to use prefetch_count, so worker will process only one message at time.

*Pluses*
- I can use work queues in some simple scenarios in projects;

*Minuses*
- There can be situations, then two workers can receive and save same message.

https://github.com/zolotarevandrew/rabbitmq/tree/main/work-queues

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 25 apr 22
**Rabbitmq Routing**
Bindings can take an extra routing_key parameter.
The fanout exchanges, simply ignored its value.

Routing keys are on messages, so the producer has a control by sending message by using correct routing key.

Exchanges compare a messages routing key to each route's binding key to determine if the message should be sent to the queue on that route.

https://github.com/zolotarevandrew/rabbitmq/tree/main/routing/DirectLogging

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 1 apr 22
**Rabbitmq Bindings**
Create binding parameters:
- queue name
- exchange name
- routing key
- arguments

There can be exchange to exchange bindings, exchange to queue bindings.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 31 mar 22
**Rabbitmq Topics**
Internally uses trie.
Partially matching routing key.

Can be "#" - one or more words
Can be "*" - one word, faster than "#".

*Pluses*
- I can use topics for complex pub/sub scenarions;

https://github.com/zolotarevandrew/rabbitmq/tree/main/exchanges/Topic

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 30 mar 22
**Rabbitmq Queues**
Queues in RabbitMq FIFO.

Queue properties:
- Name, can max be 255 bytes
- Durable (survive on restart)
- Exclusive (will be deleted when that connection closes)
- Auto-delete (deleted when last consumer unsubscribes)


Before a queue can be used it has to be declared. 
Declaring a queue will cause it to be created if it does not already exist.

Optional arguments:
- length limitat
- mirroring settings
- max number of priorities
- consumer priorities

Ordering can be affected by the presence of multiple competing consumers, consumer priorities, message redeliveries.

Initial deliveries - redelivered(false), single consumer can process in FIFO order.
Repeated delivery - ordering can be affected by the timing of consumer acknowledgements and redeliveries.

If all of the consumers have equal priorities, they will be picked on a round-robin basis.

Durable queues will be recovered on node boot, including messages in them published as persistent. 
Messages published as transient will be discarded during recovery, even if they were stored in durable queues.

Throughput and latency of a queue is not affected by whether a queue is durable or not in most cases.
The choice between durable and transient queues comes down to the semantics of the use case.

Temporary queues:
- transient clients
- temporary WebSocket connections

Deleting temp queue automatically:
- Exclusive queues (declaring connection is closed or gone (e.g. due to underlying TCP connection loss))
(x-queue-master-locator="client-local" when declaring the queue)
- TTL 
- Auto delete queue (deleted when its last consumer is cancelled)


Queues can have their length limited. Queues and messages can have a TTL.
Queues keep messages in RAM and/or on disk.
Publishing messages as transient suggests that RabbitMQ should keep as many messages as possible in RAM.

Queues can have priorities from 1 to 10.
Publishers specify message priority using the priority field in message properties.

Delivered messages can be acknowledged by consumer explicitly or automatically as soon as a delivery is written to connection socket.

High number of unacknowledged messages will lead to higher memory usage by the broker.
Automatic acknowledgement can be problem if consumer process can't process a lot of messages.
Consumers using higher (several thousands or more) prefetch levels can experience the same overload problem.

*Pluses*
- I can configure properly queue settings for my use cases;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 29 mar 22
**Rabbitmq Exchanges**
Direct - sends message to concrete queue by routing key.
Topic - sends message to concrete queue by routing key template.
Fanout - send message to all queues.
Headers - send message by header parameters;

*Pluses*
- I can choose the suitable exchange type in rabbitmq for my future tasks;

https://github.com/zolotarevandrew/rabbitmq/tree/main/exchanges/Exchanges
[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 28 mar 22
**Rabbitmq AMQP**
AMQP - Advanced message queueing protocol.
It is a binary application layer protocol. 

Provides Deliver guarantees:
- at-most-once (where each message is delivered once or never);
- at-least-once (where each message is certain to be delivered, but may do so multiple times);
- exactly-once (where the message will always certainly arrive and do so only once).

Support authentication and encryption. Based on TCP.

AMQP defines a self-describing encoding scheme, allowing representation of a wide range of commonly used type.

The basic unit of data in AMQP is a frame.
There are 9 AMQP frame bodies:
- open (the connection)
- begin (the session)
- attach (the link)
- transfer
- flow
- disposition
- detach (the link)
- end (the session)
- close (the connection)

link - heart of amqp.
attach - initiate new link, detach - detach new link.
links may be established in order to receive or send messages.
messages sent over a established link using the transfer frame.

each transferred message must eventually be settled. 
settlement ensures that the sender and receiver agree on the state of the transfer, providing reliability guarantees.

Ensuring the message as sent by the application is immutable allows for end-to-end message signing and/or encryption and ensures that any integrity checks (e.g. hashes or digests) remain valid.


Messages are published to exchanges (postoffice, mailbox).

Exchanges distritbute copies to queues using bindings (rules).
Then broker deliver messages to consumers subscribed to queues, or consumers pull messages from queues on demand.

Publisher can specify message metadata.
Application can fail process messages, so message acknowledgements are used.

AMQP programmable protocol, entities routing schemes defines by application, not a broker.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 22 mar 22
**Postgres partitioning**

Types:
- By date;
- By id ranges;
- By first char of name;

By date and By Id:
- Easy to understand;
- Stable number of rows;
- Need to add partitions sometimes;
- Can be table scans if query without date or id;

Partition is separate table, it can't have primary key and be a foreign key.

**Pluses**
- I can create postgres partitions to increate my apps db queries perfomance;

**Minuses**
- In some cases, it is better to optimize db queries, rather than make partitions.
- It is need to add new partitions, based on partition types (can be leverared by DDL scripts)

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 21 mar 22
**Postgres base performance problems, replication**
Metrics:
- High cpu usage >50%;
- Transactions count > 20-30 thousands;
- I/O usage;
- Exclusive Locks count;
- Long running transactions;

Solutions:
- write only needed things;
- read only needed data;
- split writing information;
- removing distinct by joins;
- a lot of aggregations, count, sum and other remove it;
- create different users for different purposes, analytics and other.
- partial indexes;

Replication:
- First we creating async replica and Moving aggregations to them.
- Second we createing async lagging replica for long running operations;
- Third creating sync/async replica with low latency (network should be good);

Sync replica problems:
- vacuum problems on master;
- table index bloat;

Streaming replication - by tcp connection (some transactions can be lost);
- async by it manner;

Cascade replication - by other secondaries replication.
Sync replication - wal should be writed on primary and secondary.
Because of latency, can be problems with locks.


**Pluses**
- I can choose correct replica type, based on datacenters location and other factors;
- I can rely on many new metrics to increase my databases perfomance;


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 18 mar 22
**MongoDB time series collections**
Time series collections efficiently store sequences of measurements over a period of time.
Contains:
- timeField, timestamp for document;
- metaField, should rarely changed;
- granularity, seconds by default, can be hours, minutes;
- expireAfterSeconds, ttl, automatic deletion of documents;

MongoDB treats time series collections as writable non-materialized views on internal collections that automatically organize time series data into an optimized storage format on insert.

The implementation of time series collections uses internal collections that reduce disk usage and improve query efficiency. Time series collections automatically order and index data by time.

https://github.com/zolotarevandrew/databases/blob/main/mongodb/timeSeries.js

**Pluses**
- I can use timeseries collection for some metrics, if i can't use grafana or influxdb or something;


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 17 mar 22
**MongoDB sharding**
Mongodb doesn't have a partition mechanism such as relation databases.
Sharded cluster:
- shard - subset of sharded data, each shard can be a replica set.
- mongos - query router, can use hedged-reads (to get first response from multiple replicas)
- config servers - cluster settings

High Availability:
If one ore more shards replicas becomes unavailable, other shard can process requests.

Once a collection has been sharded, MongoDB provides no method to unshard a sharded collection.

Unsharded collections are stored on a primary shard. Each database has its own primary shard.

*Hashed sharding*
range-based queries can target more than one shard.

*Range based sharding*
Poorly considered shard keys can result in uneven distribution of data, which can negate some benefits of sharding or can cause performance bottlenecks.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 16 mar 22
**MongoDB replication secondaries, oplog, data sync**
Secondaries can be:
- Priority 0, Prevent it from becoming a primary in an election, reside secondary or a cold standby.
- Hidden, prevent reasing from applications;
- Delayed, historical snapshot.

Priority 0:
Might be desired if the particular member is deployed in a data center that is distant from the main deployment and therefore has higher latency.

In some replica sets, it might not be possible to add a new member in a reasonable amount of time. 
A standby member keeps a current copy of the data to be able to replace an unavailable member.

Hidden:
Can vote in elections. Useful for dedicated tasks such as reporting and backups.

Delayed:
Applied operations with delay.

Arbiter:
Vote only secondary, has no data, useful just for voting.

*Oplog*
The oplog (operations log) is a special capped collection that keeps a rolling record of all operations that modify the data stored in your databases.
Retention period can be specified.
Each operation in the oplog is idempotent.

When large oplog needed:
- Updates to multiple documents at once, oplog translate multi-updates into individual operations.
- A lot of deletions and inserts;
- Significant portion of the workload is updates;

Slow oplog operations in secondaries can be found in logs.

**Pluses**
- I can choose correct replica types in mongodb to increase my app availability and resiliency;
- I can use correct combination of replicas, such as arbiters, hidden, delayed secondaries if bussiness really needs it.
- I can change oplog size then a lot of write or delete operations perfomed on collection;


**Minuses**
- I had to find slow connection creations and oplog operations in mongodb logs;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 14 mar 22
**MongoDB replication**
A replica set in MongoDB is a group of mongod processes that maintain the same data set.
Replica sets provide redundancy and high availability.
Replication provides a level of fault tolerance against the loss of a single database server.

A replica set contains several data bearing nodes and optionally one arbiter node.
Of the data bearing nodes, one member is deemed the primary node, while the other secondary nodes.

Primary receives all write operations. Records all changes to its data sets in its operation log, i.e. oplog.
Secondaries replicate the primary's oplog and apply the operations to their data sets asynchronously.

Replication lags - amount of time that it takes to copy (i.e. replicate) a write operation on the primary to a secondary.

Failover:
When a primary does not communicate with the other members of the set for more than the configured electionTimeoutMillis period (10 seconds by default), an eligible secondary calls for an election to nominate itself as the new primary.

Read preference - by default clients read from the primary.

Multi-document transactions that contain read operations must use read preference primary. All operations in a given transaction must route to the same member.
Depending on the read concern, clients can see the results of writes before the writes are durable.

Mirrored Reads reduce the impact of primary elections following an outage or planned maintenance.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 10 mar 22
**MongoDB connection pooling**
MongoClient provides connection pooling.
MongoClient instance per application should be used, unless the application is connecting to many separate clusters.

Tuning:
- High CPU Usage, reduce maxPoolSize;
- load small, increase maxPoolSize;
- slow connection creation, change minPoolSize, to increase connections created in before startup;

Slow connection creation, should found from logs;

Database profiler.
The profiler writes all the data it collects to a system.profile collection, a capped collection in each profiled database. Than means in has fixed amount documents by the time.

When enabled, profiling has an effect on database performance. Profiling also consumes disk space, as it logs to both the system.profile collection and also the logfile.

I think it is better to have logging metrics by some queries and commands than enable this feature.

**Pluses**
- I can profile database queries and hot paths by using database profiler sometimes;
- I can use correctly mongodbdriver knowing the fact of connection pooling is inside driver;

**Minuses**
- I should really think about using profiling in production, because it is can affect perfomance;
- It is hard to find connection startup info inside mongodb logs, to profile connection creation;


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 10 mar 22
**MongoDB explain**
The explain results present the query plans as a tree of stages.
Each stage passes its results to the parent node.
Stages:
- COLLSCAN collection scan;
- IXSCAN scan index;
- FETCH retrieve documents;
- SHARD_MERGE merging results from shards;
- SHARDING_FILTER filtering orphan documents from shards.

When an index covers a query, mongo can return results using only index keys.
So there will be no FETCH stage as parent stage for IXSCAN. (totalDocsExamined = 0)

For index intersection parent stage can be AND_SORTED or AND_HASH.

Sort and Group stage has a additional flag usedDisk, is disk was used.

**Pluses**
- I can use combines indexes to skip documents fetching from the disk;
- I can know if there is collection scan, and add indexes;


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 9 mar 22
**MongoDB statistics**
Collection stats has following informations:
- storage stats, indexes size, colection size, count documents and other;
- query exec stats, collection scans;
- cache stats, currently in cache, reads, writes;
- compression;
- transactions, update conflicts, rollback info;
- index details;

**Pluses**
- I can create metrics based on collection statistics to identify, caching problems, slow queries and other problems;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 8 mar 22
**MongoDB deadlocks**
Nothing special found about deadlocks

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 7 mar 22
**MongoDB locks**
Mongo uses multi-granularity locking and global, database or collection level.
It uses reader writer locks.

Intent locks are at the top of the hierarchy and their purpose is to prevent unnecessary checking of low level locks where possible.

For most read and write operations, mongo uses optimistic concurrency control.
When the storage engine detects conflicts, one will incur a write conflict causing retry that operation.

From mongodb 5 find and aggregate queries are lock free.

Locking modes.

Shared - shared between multiple readers, in read operations.
Exclusive - resource not available for concurrent readers, in write operations.
Intent shared - indicates that the lock holder will read the resource at a granular level.
Intent exclusive - lock holder will modify the resource at a granular level.

Parallel transaction making changes to one field, for one transaction there can be a error
"Plan executor error during findAndModify :: caused by :: WriteConflict error"

**Pluses**
- I can use lock free methods from version 5 mongodb, to increase my apps perfomance;
- I can use atomic operations such as inc and others, to prevent race conditions problems;

**Minuses**
- Sometimes i need to use optimistic concurreny by using replace documents without atomic operations;
- I need know and handle writeconflict exception then using transactions;

https://github.com/zolotarevandrew/databases/blob/main/mongodb/locks.js

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 4 mar 22
**MongoDB views**
Views can be useful:
- For remove joining collection inside app;
- add computed fields or metrics;
- exclude some sensitive data;

Materialized views:
Aggregation pipeline results can be merge into collection.
Data can be replaced every time.


https://github.com/zolotarevandrew/databases/blob/main/mongodb/views.js

**Pluses**
- I can use views for some security reasons, to exclude payment cards, roles and other;
- I can use materialized views to store some frequent slow queries;

**Minuses**
- I had to store aggregation inside my applications to use materialized views;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 3 mar 22
**MongoDB aggregations**
Aggregation pipeline consists of one or more stages.
- Each stage perform an operation on the input docs.
- Outputs are passes to then next stage;

The query planner analyzes an aggregation pipeline to determine if indexes can be used to improve pipeline performance.

Operators which can use indexes:
- match, sort, group,

https://github.com/zolotarevandrew/databases/blob/main/mongodb/aggregations.js


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 2 mar 22
**MongoDB relations**
In mongo related data can be embed in a single document (denormalized model).

Embedded data:
- Contains relationship;
- Can update data in single atomic operation;
- Better performance for read operations;

Subset data - separate collection which contains additional, less frequently-accessed data.

Relations:
- One to one, embedded documents can be used or subset collection;
- One to many, embedded documents can be used or subset collection or references (you can also store top ten elements which needed to query);
- Many to many, embedded documents or references.

For refs lookup operator act as left outer join.

**Pluses**
- I will prefer using embedded documents in my apps, to increase apps perfomance;
- I can use subset of data, to store more needed information in separate collections;

https://github.com/zolotarevandrew/databases/blob/main/mongodb/relations.js

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 1 mar 22
**MongoDB Indexes**
Mongo indexes has sort order, but it is needed only for sorting operations in compound indexes.

Multikey indexes is used for content stored in array, it creates separate index entries for every element of the array.

Text indexes - can be only one per collection, so it should be compound text index for some scenarios. 
They.

Index properties:
- ttl indexes - can be used to automatically remove documents from a collection;
- unique indexes - can't contain duplicate values;
- partial indexes - filter expression indexes, has lower storage requirements and reduced performance costs for index creation and maintenance;
- hidden indexes - hide index from a planner;

**Pluses**
- I can use ttl indexes for some event logs collections;
- I can use partial indexes to increase some queries perfomances;

https://github.com/zolotarevandrew/databases/blob/main/mongodb/indexes.js

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 28 feb 22
**MongoDB Transactions**
From mongo 4.2 transactions can be used across multiple shards.
From mongodb 4.4 collections and indexes can be made inside transactions.

For starting one or more transaction we needed to start a session.

Read concern - by default it uses local concern.

Local read concern - returns the most recent data available from the node but can be rolled back;
Available read concern - no guarantee that the data has been written to a majority of the replica set members can getting orphaned documents;
Majority read concern - data that has been acknowledged by a majority of the replica set members;
Snapshot read concern - data from a snapshot of majority committed data;

Needs to test on cluster with replicas.

https://github.com/zolotarevandrew/databases/blob/main/mongodb/transactions_simple.js

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 25 feb 22
**Postgres connection pooling**
Postgres spawns new process on each connection and it's can be more than 2mb, 
there could be a memory problem because of a lot connections.

Connection pooling, like a thread pooling, solves this problem.

There are two popular types of poolers:
- pgbouncer
- pgpool

pgbouncer has three modes:
- session pooling (default), after client disconnects.
- transaction pooling, after transaction finishes;
- statement pooling, after statement finishes.

session - long living;
transaction - living based on transactions;
statement - short living based on statements;

npgsql - .net postgresl provider has also in memory connection pooling, (100 default).

I tried disable npgsql pooling, it a little bit reduced throghput (10% inserts);

```
SELECT count(*) FROM pg_stat_activity where query <> '<insufficient privilege>'
```
Interesting thing, there was only 30 connections pgbouncer used, even if a run more than 1000 virtual users load test.


**Pluses**
- I can use correct connection pooling by pgbouncer, for increasing my databases performance;
- I can change pooling setting in my npgsql driver, to increase my application and database perfomance and throughput;

**Minuses**
- Session pooling is the default method;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 23 feb 22
**Postgres ef optimistic concurrency**
ef core has a simple mechanism for optimistic concurrency, i simply used rowVersion row.

**Minuses**
- Optimistic concurrency requires user to retry request;

https://github.com/zolotarevandrew/databases/tree/main/postgresql/ef-core/WebApi

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 22 feb 22
**Postgres ef core relations**
Ef core has a good mechanism for db first approach.
I have tested all relations approaches (one to one, one to many, many to many)

https://github.com/zolotarevandrew/databases/tree/main/postgresql/ef-core/BaseRelations

**Minuses**
- I should always check, which sql query ef core produces, because it can be inefficient;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 21 feb 22
**Postgres merge joins and sorting**
merge join works only on joins where results ordered by sort condition key.

Merge join uses only one iteration for each set of internal and external rows.
It uses two pointers for internal and external rows because rows sorted.

Sort types:
- Quick sort, when rows set can be placed in  the memory;
- Top n Heap sort, when rows are limited;
- Merge sort, when rows can't be placed in the memory;

Merge sort - readed rows sorted in memory by quick sort and write to temp file, until all rows readed;

Unique values and grouping:
- distinct fields can get easily by one loop if rows are sorted;

**Pluses**
- I can write better queries with join, because i can decide which type of join database will choose, based on my query. 

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 18 feb 22
**Postgres hash joins**
Base idea is to use hash table to get correct rows.

First stage:
- sequantially read internal rows, and foreach row hash functions is calculated;
- hash key is join condition;
It is effectively working if all rows can be in ram.

Second stage:
- for each external rows, we searching internal row in a hash table by join key.
- all found data return as results;

Two-way hashing:
- on planning stage db calculating if internal rows can fit within the given memory.
- internal rows splits by on separate batches;
- batch count is powers of two.

First stage:
- reading internal rows and bulding hash table;
- if rows is within a batch it goes to a temp file, not to a ram;

Second stage:
- reading external rows;
- all external rows splitted as in a first stage, by temp files if its needed.
- start matching rows in memory;
- start matching internal and external batches;

Parallel hashing:
the hash table is not created in the local memory of the process, but in the shared dynamically allocated memory. So all parallel processes can access needed data.

- Creating hash table in parallel;
- Match external rows in parallel;

Hash join can be used in all join types, but only if join condition is equality operator;


**Pluses**
- I should get only needed fields, when my queries uses hash join, because all rows  goes to ram;


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 17 feb 22
**Postgres nested loop joins**

nested loop:
first loop called (external) and second loop called (internal).
each found pair returns immediatly as part of result.

for cartesian product:
Nested Loop
-> Seq Scan (external set)
−> Materialize (internal set)
Materialize also uses seq scan, but upon repeated access, the node reads the previously memorized rows.

In general, the total cost of the join:
- cost of getting all external rows;
- cost of one time getting all internal rows;
- N-1 cost of repeatedly getting internal rows (N external rows);
- cost of processing each row;

Postgresql 14 has memoize node, which can cache internal rows.

Nested loop can be used in left join, but can't be used in full and right joins, 
because some rows will not be viewed.

Antijoin - returns first set rows, if there is no match for them in another set. 
(can be represented as left join with where is null).

Semijoin - return first set rows, if there is at least one match for them in the second set.

Nested loop allows parallel mode, where external nodes can be paralllel

https://github.com/zolotarevandrew/databases/blob/main/postgresql/joins/simple.sql

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
##16 feb 22
**Postgres relations**
one to one
one to many
many to many
just remembered

https://github.com/zolotarevandrew/databases/tree/main/postgresql/relations

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 15 feb 22
**Postgres mvcc**
Each relation (table, view, index and other) has a multiple forks.
Every fork has a type and it's own data.

Fork can grow up to 1gb, and then new fork will be created.

So tables, indexes more than 1gb would be stored in separate files.
Each file splitted by 8 kb pages.

Fork types:
- Initialization, exists only for UNLOGGED tables;
- Free space map, where the presence of empty space in the pages is marked, when new row is added space reduced, when cleaning started in expands
It used for adding new rows, so db should fast find place to add data.
- Visibility map, one bit marks pages that contain only actual versions of rows;

Pages:
- Header;
- Ref array to row versions (for indexes);
- empty spaces;
- row versions;

Each version of a rows should be inside one page, but if it exceeds page size, TOAST will be used.
TOAST (The Oversized Attributes Storage Technique).
TOASTR it's a separate table with it's own indexes.
If table has text or numeric field, TOAST table will be created.

https://github.com/zolotarevandrew/databases/tree/main/postgresql/mvcc

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 14 feb 22
**Db Other anomalies**

Inconsistent read:
First transaction update data, then second transaction read data, first transaction update data, then second transaction read data.
So the last read for the second transaction will be incosistent, we can solve this by sum agrregation, because we have correct state when transaction starts.

The problem can appear by functions volatility classification (VOLATILE, STABLE, IMMUTABLE), by default function has a VOLATILE behaviour.
If we calling volatile function that has a select statement inside other select operator, it can read incosistent data. So we can solve it by STABLE keyword.
(Should learn more about this)

Inconsistent write:
we have a rule to have sum by accounts greater than zero, we start two repeatable read transaction and remove amount from two separate accounts and the problem comes in.

https://github.com/zolotarevandrew/databases/blob/main/postgresql/acid/inconsistent_read.sql
https://github.com/zolotarevandrew/databases/blob/main/postgresql/acid/inconsistent_read_by_update.sql
https://github.com/zolotarevandrew/databases/blob/main/postgresql/acid/inconsistent_read_volatile_functions.sql
https://github.com/zolotarevandrew/databases/blob/main/postgresql/acid/repeatable_read_incosistent_write.sql

**Pluses**
- I can prevent some incosistent read anomalies in my projects, which use postgresql;
- I can prevent some incosistent write anomalies in my projects, which use postgresql;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 11 feb 22
**Db MVCC/Locks**
Snapshot isolation - isolation protocol based on snapshots.
In postgresql snapshot isolation is multiversion based. 
Db can have multiple versions of the same row.
In fact only changing the same row is blocked.
Writing transaction doesn't block reading transactions.
Reading transactions doesn't block anything.

By using snapshots in postgresql it also resolve phantom read anomalia.

Solutions to some isolation problems:
- don't write code:) just use constraints;
- INSERT/UPDATE/DELETE ON CONFLICT;
- LOCK SELECT FOR UPDATE;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 10 feb 22
**Db locks**
Exclusive lock - no one can read or change data, if some thread entered in exclusive lock.
Shared lock - lock for value, so nobody can change it inside a transaction.

Postgres has a row level lock by - FOR UPDATE statement.

I should always test parallel transactions.
So for example first transaction update somethins gets an exclusive lock, and other transaction blocked
until first transaction not committed. So postgres can refresh update statement before commit and see what row changed,
but he can also  not call refreshing the row if some fields were indexed.

https://github.com/zolotarevandrew/databases/tree/main/postgresql/locks

**Pluses**
- I can solve double/booking and some other problems, using exclusive locks in postgres;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 9 feb 22
**Db B trees/B+ trees**
To fund a row in a large table we perform full table scan. Reading large tables is slow.
Requires many I/O to read all pages.

B tree element has a key and value. The value is usually data pointer to the row.
Data pointer point to primary key or tuple.
A node = disk page.

B tree limitations:
- Store keys and values;
- Internal node take more space this require more IO and can slow traversal;
- Range queries problem 1-5;

B+ trees:
- Store keys in internal nodes;
- Values stored in leaf nodes;
- Internal node are smaller because store only keys;
- Leaf nodes are linked;
- Great for range queries;


*OS*
Critical sections:
- Forbid interruptions, because cpu changes process by timers or other interruptions (it works only on systems which has one cpu);
- Active waiting, spin lock, consumes cpu time, can used for a fast locking;
- Peterson algorithm - cycle plus bool variables;
- TSL - cpu blocks memory bus then executing instruction;
- Semaphores - has up and down methods, up increases counter, down decreases counter, this should be atomic operations;
- Mutex - exclusive lock can use TSL and thread yield instructions to free some threads.


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 8 feb 22
**Postgres statistics**

Query Executing process:
- creating connection (maybe using connection pool);
- parsing query, split to tokens;
- building a tree, going to rewrite system;
- using planner/optimizer;
- the more table you have the more ways to execute the query;
- for each operation estimates number of rows and execution time;
- executor comes in after getting the plan 

Number of rows or pages recalculated by:
- auto-vacuum
- auto-analyze
- created index, or some DDL operation was executed (CREATE, ALTER, DROP, TRUNCATE, COMMENT, RENAME)

inserted + updated + deleted > threshold = run autoanalyze
```threshold = analyze_threshold + reltuples(pgclass)*analyze factor```
analyze_threshold - default 50, can be changed by ALTER TABLE;
analyze_factor - default 0.1

pg_stats has most_common_vals and most_common_freqs values.
if there is a lot of frequent values, we can exclude this data from index.

ANALYZE command updates statistics in pg_statistics catalog, like update statistics in MSSQL.

https://github.com/zolotarevandrew/databases/blob/main/postgresql/statistics/statistics.sql

**Pluses**
- I can optimize my tables by sometimes changing default analyze_threshold or factor, when table grows very fast;
- I can exclude some data from index, to optimize search speed and size of the index.
- I will prefer to use jsonb over json in postgres, because postgres don't store statistics for json fields.

[Log Index]
----------------------------------------------------------

----------------------------------------------------------
## 7 feb 22
**OS**
Interrupt process:
- driver send command to controller
- Controller send interrupt signal when read or write was finished
- If interrupt controller ready to receive interruption, he send signal to cpu
- Interrupt controller send device number to bus
- Then cpu ready command counter goes to stack and then goes to kernel mode
cpu can block and continue interruption.

Processes:
Unix process has
- Text segment - program
- Data segment - variables
- Stack segment

Processes in unix has hierarchy structure and init process is on top.
Windows has no hierarchy, and child process just has parent descriptor.

Process state:
- executing, using cpu
- ready, working but stopped, for executing other process
- blocked until some event will happen
Os has table of processes like array or list.
Os has a vector of interruptions, for each device it store a address for execution procedure.
Cpu time = 1 - p^n
Where n - count of processes


*Db views*

Simple view:
- just a virtual table;
- updated every time then a table updated;
- slow processing;
- not pre-computed;
- no storage space needed;

Materialized view:
- physical copy of a table;
- updated manually or using triggers;
- fast processing;


**Pluses**:
- I will use materialized views for some statistical reports, whicn can be updated by analytics of by cache miss.
- I will use views for some frequently used queries with complex logic;


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 4 feb 22
**Db and Data**

each row has a additionall unique id, for postgres it is tuple_id, pair (block number, tuple index).

```
select *, ctid from sales p ;
```

*Page*
- Database doesn't read a single row, it reads a page or more in a single IO and we get a lot of rows in that IO;
- Each page has a size (8KB in postgres), (8KB in MSSQL).

*I/O*
- request to the disk;
- can't read a single row;
- can goes to OS cache.

*Heap*
- data structure where the table is stored with all its pages;
- traversing expensive;
- that's why we need indexes that help tell us exactly that part of the heap we need to read.

*Index*
- data structure, which has pointers to the heap.
- has a part of data;
- one data was found, you go to the heap to get additional data;
- also store as pages and cost IO;

Learnt about explain analyze and primary and secondary indexes.
https://github.com/zolotarevandrew/databases/blob/main/postgresql/indexes/explain_analyze.sql
https://github.com/zolotarevandrew/databases/blob/main/postgresql/indexes/simple_table.sql


**Pluses**
- I can use indexes with include statements, to prevent index scan and use index only scan.
- I can create index on two or more fields, for better using where statement with AND operator;
- I can use explain, to change by indexes structure;
- I can create index concurrently, which won't block the table inserts.

**OS**

Registers - 1ns, 1kb
Cache - 2ns, 4 mb
Ram - 10ns, 1gb+
Disk - 10 ms, 100gb+

Ram splits up to cache strings by 64 bytes.
Often used strings goes to cpu cache.
It takes two tacts to get data from cache.
If string does not exist in cache, we should go to ram so it is more time expensive.
Modern Cpus has two levels of cache - l1, l2.
Intel uses shared cache, amd uses cache for each core.
I/O devices has a controllers.
Controllers has a different interfaces, thats why drivers were created. Manufacturers are creating drivers for each supported OS.
Os can load drivers dynamically like for usb or load them only then os starts.


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 3 feb 22
**ACID**
Played some examples about ACID, serializable, phantom reads and atomicity;

https://github.com/zolotarevandrew/databases/blob/main/postgresql/acid/atomicity.sql
https://github.com/zolotarevandrew/databases/blob/main/postgresql/acid/phantom_reads.sql
https://github.com/zolotarevandrew/databases/blob/main/postgresql/acid/repeatable_read.sql
https://github.com/zolotarevandrew/databases/blob/main/postgresql/acid/serializable.sql

Interesting thing postgres resolved phantom reads even in repeatable read isolation level.

**OS**
Os can load tasks from disk to free memory space - spooling.
*Cpu*:
It is a computer brain. He chooses commands from memory and execute them. 
Selecting command, decode to find type and operands, executing.
Each cpu has its own commands, so x86 commands cant be executed on ARM. 
Registers intended for storing variables and intermediate result.
Other registers available for programmers:
- Command counter - stores memory address with next executing command;
- Stack pointer - ref to stack top;
- Program status word - bits for cond operators, bits for cpu priority,  kernel and user mode bits and other;
 Modern cpus can execute more than one command at time by using conveyors. 

Also we have superscalar cpus, which has different executing blocks (logical operators, integers).
As a result commands executing in different order


**Pluses**
- I can build better apis, knowing the facts of read replicas are eventual consistent.
- I can use correct isolation levels, to prevent some read phenomenas in my applications.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 2 feb 22
**ACID**

Transaction is a collection of queries, one unit of work, which implements all or nothing rule.
Transactions can be readonly to guarantee you are getting consistent snapshot of data.

*Atomicity* - all queries in a transaction must succeed. 
If one query fails, all prior successful in the transaction should rollback;
If the database went down prior to a commit, all the successful queries in the transaction should rollback;
 
*Isolation* - transaction see changes made by other transactions (even running transactions concurrently).
Read phonomenas:
- Dirty reads, if you wrote something, and it is not really committed, but you see it in your transactions.
- Non-repeatable reads, then you read value, then select sum, which also has a prev readed value, and value was changed during the transaction.
- Phantom reads - when you reads aggregated data, there can be new row inserted.
- Lost updates - when you write and other transaction changes that row, then you will see lost update.

*Dirty reads*:
|PID | qnt | price   |
|:--:|:---:|:-------:|
| P1 | 15  | 5       |
| P2 | 20  | 4       |
```
select pid, qnt*price from sales
p1, 50
p2, 80

then concurrently
update sales set qnt = qnt + 5 where pid = 1
but not commit

select sum(qnt*price) from sales

we get $155, when it should be $130, we read a dirty value
```

*Non repeatable read*:
same table as in dirty reads.
```
select pid, qnt*price from sales
p1, 50
p2, 80

then concurrently

update sales set qnt = qnt + 5 where pid = 1
commit


select sum(qnt*price) from sales

we get $155, when it should be $130, we got non repeatable read, because parallel transaction was committed.
```

*Phantom read*:
|PID | qnt | price   |
|:--:|:---:|:-------:|
| P1 | 10  | 5       |
| P2 | 20  | 4       |
```
select pid, qnt*price from sales
p1, 50
p2, 80

then concurrently
insert into sales('Product3', 10, 1)
and commit

select sum(qnt*price) from sales

we get $140, when it should be $130, we had a phantom read.
```

*Lost updates*
|PID | qnt | price   |
|:--:|:---:|:-------:|
| P1 | 10  | 5       |
| P2 | 20  | 4       |
```
update sales set qnt = qnt + 10 where pid = 1

then concurrently
update sales set qnt = qnt + 5 where pid = 1
and commit
so qnt will be 15, not 20.

select sum(qnt*price) from sales

we get $155, when it should be $180, we had a lost update.
```

Isolation levels:
Read uncommitted - no isolation.
Read committed - only see committed changes by other transactions.
Repeatable read - transaction will make sure than when a query reads a row,
that row will remain unchanged the transaction while it's running.
Snapshot - each query see changes that committed up to the start of the transaction.
Serializable - each transactions run one by one, no concurrency.

Isolation levels vs read phenomena:

|Level              | Dirty | Lost upd | Non repeatable | Phantom |
|:-----------------:|:-----:|:--------:|:--------------:|:-------:|
| Read uncommitted  | +     |  +       | +              | +       |
| Read committed    | -     | +        | +              | +       |
| Repeatable read   | -     | -        | -              | +       |
| Serializable      | -     | -        | -              | -       |


*Consistency* - in reads, in data.
In data:
- defined by user;
- foreign keys;
incorrect isolation level can lead to incosistent data.
replicas leads to eventual consistency.

*Durability* - changes made by committed transactions must be persisted in a durable storage.
Techniques:
- write ahead log (OS cache problem, can be possible data loss).
- Async snapshots.
- Append only file.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 1 feb 22
**.net Exceptions**
Managed exceptions in .NET are implemented on top of the Win32 structured exception handling mechanism.
SEH has two mechanisms, exception handlers and termination handlers.
Nothing interesting was found about exceptions.

**net ThreadPool**
Thread pool has a global task queue and each thread has its own queue.
Then thread queue is empty, it can get tasks from other thread, but its requires synchronization mechanism.

If there is no tasks in all queues, thread goes to sleeping mode. If it lasts long, thread awakes and self destructs.

If some thread locked by sync mechanism, thread pool will create new thread.

https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/ThreadPoolInternals.cs

**Pluses**:
- Another point to avoid locks if possible in my applications, so threads count will be stable (gc will be faster);

**Minuses**:
- Locks increases threads in thread pool.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 31 Jan 22
**.net Threads**
Each thread:
Thread kernel object - os creates internal structure
Thread environment block - address space created in user mode. Local storage for thread. It is only 4 kbs. 
User mode stack - local args and variables, by default 1 mb.
Kernel mode stack - for safety when code goes to kernel mode, all data copied from user mode stack.
Before thread created - windows load each dll by dll main.

Windows executed threads  with highest priority fiest. So there can be thread starvation.
Priority calculated by process priority and thread priority.

There are two types of threads in clr: foreground and background.
Thread poll threads are background.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 28 Jan 22
**.net GC**
GC modes: 
- low latency mode, which ignores 2-nd generation.
- batch, disables parallel execution, for maximum throughput;
- interactive, enabled parallel execution;

GC:
- pause all threads;
- Marking - Go through all heap roots that has objects ref and mark them By Sync block index, so marked objects will stay in heap;
- Compacting - objects which stays at heap moving to near memory blocks;

**Finalization**
If object type has finalizator, before calling ctor, object ref goes to finalization list.
Clr has high priority thread for finalization.
When object added to finalizaition queue this thread awakes.
Object that has finalizers lives more than 2 gcs.

**Minuses**:
- Should investigate each mode by concrete app types;


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 27 Jan 22
**.net GC**
.net GC uses two strategies. 
First, by size - small and large object heap.
Compact - squeeze heap, objects should be copied and will be compacted.
Sweep - use free parts of the heap.
SOH < 85000 bytes, LOH > 85000 bytes.
SOH uses compacting and sweep. LOH uses only sweep.

Because a lot of objects less than 85000, they goes to SOH and that's why generations came.
0 gen - time between creation and nearest GC.
1 gen - relatively long living objects;
2 gen - long living objects;


https://www.youtube.com/watch?v=DVnmGW6964o&t=429s&ab_channel=%D0%9C%D0%B8%D0%BD%D0%B8-%D0%BA%D0%BE%D0%BD%D1%84%D0%B5%D1%80%D0%B5%D0%BD%D1%86%D0%B8%D0%B8CLRium

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 26 Jan 22
**.net SpinWait**
SpinWait internally uses Thread.Yield and Thread.Sleep methods and CLR internall Thread.SpinWait.
Thread.Yield will interrupt the current thread to allow other threads to do work.
Spinning should only be attempted when you have good guarantee thad doing so is more efficient than a context switch,
because is hust utilizes CPU time.

**.net SpinLock**
Microsoft says that SpinLock should only be used for improving performance. It is good for quick locks.

**.net Interlocked**


**Pluses**:
- I can implement some lock free algorithms and structures by using SpinLock and SpinWait.
- I can use Interlocked class for async code, which has stateful data using by many threads.

**Minuses**:
- I think i will rarely use SpinWait in my apps, but it is good to know structure.
- SpinLock internally uses while cycle, so it utilizes CPU time, that is not good.
- Can't use spinlock with async/await because it is exlusive kind of lock.

https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/SpinLockInternals.cs
https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/InterlockedInternals.cs

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 25 Jan 22
**.net CountdownEvent**
Internally it just uses ManualResetEventSlim. On each signal it just uses Interlocked.Decrement, 
when count goes to zero, it calls Set method of ManualResetEventSlim.
It is have wait method which support cancellation token.

https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/CountDownEventInternals.cs


Minuses:
- Simply i can use await Task.WhenAll for simple fork/join scenarios;
- Main thread start hanging, when task completed with exception, Task.WhenAll solves this problem..;

**.net Barrier**:
Internally it also uses ManualResetEventSlim. I think it is very useful primitive, to split jobs between multiple tasks(threads).
It is also has post callback when all jobs has done.

**Pluses**:
- I can use Barrier in cases then jobs count are determined in execution time and can be decreased or increased.

**Minuses**:
- Sometimes simple tasks approach can be a better solution, should do some benchmarks;

https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/BarrierInternals.cs

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 24 Jan 22
**.net ReaderWriterLockSlim**
Unfortunately readerwriterlock has no support for async/await statements.
So i found Microsoft.Extensions.Threading which has Async implementation for reader writer lock, but internally it uses lock.

https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/AsyncReaderWriterLockSlimInternals.cs
https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/ReaderWriterLockSlimInternals.cs

**Minuses**:
- I don't think i will use readerwritelock slim, because of i always using async/await.
- I should compare benchmarks with simple lock and Microsoft.Extensions.Threading.AsyncReaderWriterLock.
- I should learn more about implementation details of Microsoft.Extensions.Threading.AsyncReaderWriterLock.
- WTF when used same code with tasks and AsyncReaderWriterLock, my main thread starts hanging.
- I think there should better option for case a lot of reads and single write, and maybe simple lock would be the case.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 22 Jan 22
**.net strings**
in .net5/6 has optimized equality == operator, which uses spans and it is more faster than Equals method, richter book was outdated:)
In benchmarks slowest was compareTo method with current culture param. So i am going to try avoid this comparisons.

.net 6 has improved interpolated string by DefaultInterpolatedStringHandler, so it is more memory efficient than stringbuilder.
https://github.com/zolotarevandrew/.net-internals/blob/main/Strings/StringComparisonBenchmarks.cs
https://github.com/zolotarevandrew/.net-internals/blob/main/Strings/StringCreationBenchmarks.cs

**Pluses**:
- I should avoid strings comparison with Culture for increase my app perfomance;
- I should use string interpolation instead of strinbguiler for lesser memory allocations in my apps (.net5/6).
- I should use string concat method, because of value types can be boxed, for lesser memory allocations in my apps.



[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 21 Jan 22
**.net EventWaitHandle**
AutoResetEvent and ManualResetEvent just using EventWaitHandle with some enum and nothing else.
It's good thread signaling mechanism.

AutoResetEvent works as exclusive lock, so only one thread can get access to resource at point of time. 
Then thread job has done, we can call set to signal one or more thread, so they can start work.

ManualResetEvent works differently from autoresetevent. In signaled state it allow all threads to execute some work.
ManualResetEventSlim has the cancellation token support. And it has spinning with Monitor implementation.


**Pluses**:
- I can use AutoResetEvent for concurrent operations, if only one thread can get access for a point of time (write to file of something).
- I am going to ManualResetEventSlim for sort of thread signaling operations, because of it has cancellation tokens support.

**Minuses**:
- Should learn more about sleeping state of thread, and how it affect on performance.

https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/AutoResetEventInternals.cs
https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/ManualResetEventInternals.cs
https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/ManualResetEventSlimInternals.cs

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 20 Jan 22
**.net Mutex**
I should learn more about implementation of mutex, but a lot of resources says about it uses kernel mode.
And it so expensive to use. We can use named mutex for inteprocess communication.
The main problem with mutex with async/await code. It is because async continuation can be executed by other thread.
But in mutex release should be called by thread, which called waitone. And this is the problem.

https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/MutexInternals.cs

**.net SemaphoreSlim**
Internal implementation of SemaphoreSlim uses lock and has methods for async code.
Should investigate more about that.

https://github.com/zolotarevandrew/.net-internals/blob/main/Concurrent/SemaphoreSlimInternals.cs

**Pluses**:
- I can use SemaphoreSlim as my mutex, semaphore for async/await methods.

**Minuses**:
- As because i always using async/await i will not use Mutex, because of its problems with async code.

To my surpise, creating SemaphoreSlim executed longer than creating Semaphore. Should investigate more about that.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 19 Jan 22
**.net Monitor**
Watched CLRium video about monitor - 
https://www.youtube.com/watch?v=GkWDcsVHh0M&ab_channel=%D0%9C%D0%B8%D0%BD%D0%B8-%D0%BA%D0%BE%D0%BD%D1%84%D0%B5%D1%80%D0%B5%D0%BD%D1%86%D0%B8%D0%B8CLRium

Monitor.TryEnter works only in os user mode and can use spin lock.
Thread.Sleep uses timer, thread awakes with higher priority.

Monitor.Enter first tries to spinwait. In worst case kernel mode lock allocated.
After allocating we use kernel lock, waiting for specified timeout.
Then it tries use spinwait to get a lock. 
Thread can awakes before timeout elapsed, so it uses while true cycle and reduces timeout on each iteration.

**Pluses**:
- I can reduce locks duration to increase the throughput of my backends;
- I can reduce threads waiting time in kernel mode by timeouts, to increase the throughput of my backends;

**Minuses**:
- I  Need to study the Monitor in more detail, for better use in practice.


[Log Index]
----------------------------------------------------------
----------------------------------------------------------
## 18 Jan 22
**.net concurrent dictionary**
internal classes are sealed, because of performance improvements.
all nodes stored in wrapper class Table, which has array of locks and nodes.
Node has next pointer, so it is implemented in linked list manner.
Table variable uses volatile keyword. Should learn more about this keyword.
Has the bool static variable isValueWriteAtomic, which checks is generic type a value type.

On add tables and locks copied in local variables, because of other threads can change size of this arrays.
Using locks array helps different threads, change different buckets. 
Size of locks array can be changed by concurrency level in constructor.
Index is returned along with lockNo - index in array of locks.
After got index, exclusive lock acquired by lockNo.

It is interesting to use an array of locks so that more threads can do more work.

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/ConcurrentDictionaryInternal.cs

**.net concurrent stack**
Internal classes are also sealed.
Node has next pointer, so it is implemented in linked list manner.
Has a pointer to head Node, variable uses volatile keyword.
Push and pop simply implemented with using spinwait and compare exchange swap.
So it is lock free implementation.

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/ConcurrentStack.cs

**Pluses**:
- I can build effective concurrent cache, or pool of locks by objects;
- I can use technique array of locks, to build more responsive apps;
- I can implement simple lock free implementation for some data structures using learnt techniques;

**Minuses**:
- I didn't quite get use cases for volatile keyword in concurrent environment;

**.ElasticSearch**
Yesterday's problem with term frequency solved by changing index options.
I deployed current solution and we should give this first version to clients.


[Log Index]
----------------------------------------------------------
## 17 Jan 22
**.ElasticSearch**

Edge n gram has issues with exact matches in my analyzer setup.

So when I typed: “Surf”, it is not returned exact match “surf” with highest score.

- "SURFPRIZE, SURFPRIZE, SURFRIZE, SURF, PRIZE, SURFRIZE, RIZE"
- "SURF COFFEE SURFING NEVER ALONE"
- "КАЙТВИНД, КАЙТВИНД, КАЙТ, ВИНД, СЁРФИНГ, MYSURF, SURF, SURFLIFE, MYSURFLIFE"
- "SURFJAZZ, СЁРФДЖАЗ, SURF, JAZZ, СЁРФ, ДЖАЗ"
- "SURF'N'FRIES ORIGINAL NATURAL ORIGINAL FRIES"

So i learnt about similarity models and why it is happened.

Default model - tf/idf. 
Position of words in document are not used, doc is just a bag of words.
Tf - It uses logarithm term frequency.
Idf - total number of docs divided by the number docs contains the word. It also uses logarithm.

But default model in elasticsearch is BM25.
It is probabilistic model and it also uses tf and idf.

Found this:
"Indeed, this is due to the fact that the ngram FILTER writes terms at the
same position (like synonyms) while the TOKENIZER generates a stream of
tokens which have consecutive positions. This gives blablablafoobarbarbar a
larger number of positions and thus a smaller length normalization."

So after changing from filter to tokenizer, problem with scores was solved. 
Also was used multi_match query with most_fields. Most_fields - "useful when querying multiple fields that contain the same text analyzed in different ways". 
For that i added multi_field with standard analyzer to my text field.
But there is problem with space-ignoring search.

Found this good approach, which uses keyword tokenizer.

https://medium.com/@davedash/writing-a-space-ignoring-autocompleter-with-elasticsearch-6c3c28e3a974

But is also has a problem with score (then text contains search term twice or higher).

**.net Priority Queue**
Elements is array of tuples (struct) with Value and its priority.
Uses standard grow factors, starts with 4, then multiplying by 2.
Uses standard heapify algorithm (non-recursive). Parent and child indexes calculated by bitwise operators (index - 1 >> 2).
Uses tuple because it allows simple swap without using third variable. (nodes[nodeIndex] = tuple).

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/PriorityQueueInternal.cs

**.net Object pool**
New feature, very cool that you don't have to write from scratch.
Has a thread safe implementation, using Interlocked.CompareExchange.
Always stores first item.
Elements stored in array of objectwrapper struct. (// PERF: the struct wrapper avoids array-covariance-checks from the runtime when assigning to elements of the array.)
Learnt about array covariance checks.
https://codeblog.jonskeet.uk/2013/06/22/array-covariance-not-just-ugly-but-slow-too/

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/DefaultObjectPoolInternal.cs

**Pluses**:
- I can build simple thread safe object pool using implemented default object pool;
- I can build custom search analyzers for elastic search for a better user experience;
- I can build custom search queries for elastic search for a better user experience;

**Minuses**:
- I can't use PriorityQueue's in concurrent environment;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------

## 15 Jan 22
**.net**

**Events:**
If event delegate used in multithread scenarios, there can be some problems.  

(Good use case, should learn more about memory barriers)

- event handler can be copied to temp var by ref, but compiler can remove this var;
- So it is better to use memory barrier and call Volatile.Read. Processor can not move calling delegate before assign to variable;

For registering and removing event handlers, compilers add methods add, remove, which using interlocked.compareexchange for thread safe swap, old and new handler.

**Generics.**

Each type has object type. 

CLR optimizes generated IL code to not overgrow, Generic that has a ref type can use same code.
But its not true for value types.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------

## 14 Jan 22
**.net**

Default parameters stores in metadata, so compiler can add them at execution.
params always allocate some space in heap, so better not to use it.

**ElasticSearch:**

Finally implemented search, still dont understand why edge n gram did not solve the problem for contains queries.
Instead wilcard queries with boost was used.

surf coff - (surf* and coff*)^3 or (surf*)^2 or (coff*)^1.

So the problem was with docs in elastic search, they don't put this "Sometimes, though, it can make sense to use a different analyzer at search time, such as when using the edge_ngram tokenizer for autocomplete or when using search-time synonyms."
into edge n gram docs. 

I just changed search_analyzer to standard. And my query became simple - "query": "surf coff".
So now search became faster, because of using index correctly.


**.net stack**

has a simple array implementation.
Capacity started from 4, and then multiply by 2.

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/StackInternal.cs

**.net queue**

has a simple array implementation.
has _head and _tail index.
Capacity started from 4, and then multiply by 2.
On enqueue, they increment _tail my method with ref index parameter. 
Has a comment (JIT produces better code than with ternary operator ?:) - i think it is  branch to target operator, should learn more about this.

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/QueueInternal.cs

**.net list**

has a simple array implementation.
Capacity started from 4, and then multiply by 2.
Interesting, they avoid the extra generic instantiation for Array.Empty<T>() - by static instantiation of new T[0];

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/ListInternal.cs

[Log Index]
----------------------------------------------------------
----------------------------------------------------------

## 13 Jan 22

**.net hashtable**

Hashtable was implemented a long ago. It is because bucket key and value has object type.
Bucket is struct.
Then there was no generics, and struct type was boxed.
It is using double hashing technique - h(key, n) = h1(key) + n*h2(key).
h1 - can be any number, h2 - can be only from to table size.

They called func to get hash - InitHash, wtf ?:) 
Because of Bucket implemented as struct.

Was used load factor (0.72 by perfomance tests).
Was used _occupancy - total number of collision bits set.

InitHash - has two out parameters, for h1 and h2. h2 using only then there is collision.

For contains key, buckets copied in stack (Take a snapshot of buckets, in case another thread resizes table)

Insert - Has a lot of strange conditions because of optimizations and collision bits.

Has a lot of complex details, worth a look later. 

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/HashTableInternal.cs


**.net dictionary**
Has a bucket and entry arrays.
Bucket array stores indexes, and entry array stores entries.
Entry struct - hash code, key, value, next - next entry in chain.
Size Expanding uses prime number close to count.
For insertions entries copied in new stack frame (same as for hastable case another thread resizes table).
Collision resolves by chaining, because of entry has a next field.

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/DictionaryInternal.cs

**Pluses**:
- I can use technique to copy my data structs in new stack frame for concurrent environments;
- I can use dictionary more smart, to reduce the use of extra memory;

**Minuses**:
- I am not going to use HashSet, because it is outdated;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------

## 12 Jan 22

**.net expressions**

Interesting topic. But looks very unusual and unreadable.
I have never used even once in production.

Read more about this in enterpise development - https://habr.com/ru/company/jugru/blog/423891/
It is good that expressions way more faster than Activator and other Reflection tools for creating objects.

Should warmup expression because of slow compilation (start from 300 milliseconds).

Should learn more about projections by expressions (how it works in automapper and other providers).

Topic not interesting to me, for now.

https://github.com/zolotarevandrew/.net-internals/blob/main/Expressions/Program.cs

[Log Index]
----------------------------------------------------------
----------------------------------------------------------

## 11 Jan 22

**Docker:**

Looking for a problem - resource temporarily unavailable, cant fork process.

So the problem was with the docker container and chrome in it.

The docker container has 512 default pids. But my container created over 1000, because of chrome driver starting by each request.
After execution stops, a lot of pids, more than 500 just hang.  

Tested on local docker.

So should never span a lot of processes inside container. Knew it, but just thought it would be okay.

Instead selenoid was used and problem was solved.

**c# records, structs, classes**:

record is just wrapper around class, has some additional methods, and implements iequatable.
record created like this: record MyRecord(string Name) - has a deconstruct method, and init only setters.

simple struct has no equality operator ==, should only use Equals (no boxing).
record struct implements IEquatable and has equality operator ==, useful for DDD ValueObjects.

operator with for structs - copy struct and changes values.
operator with for classes, records - copy class by Clone Method and changes values.

readonly ref record struct - useful for immutable DDD ValueObjects.
ref struct - lives only stack, useful for low optimizations.

https://github.com/zolotarevandrew/.net-internals/tree/main/ClassesStructsRecords

**.net**

IL has call and callvirt methods. callvirt called polymorphically.

Sealed class cant use inheritance. So compiler can optimize virtual methods, such as ToString.

**Pluses**:
- I can use sealed .net classes to improve my applications performance;
- I can use readonly record structs as my DDD Value objects.
- I can use ref structs to create low-level optimizations.

**Minuses**:
- I will never use docker container to create a lot of processes inside of it;

[Log Index]
----------------------------------------------------------
----------------------------------------------------------

## 10 Jan 22

Should learn more about LayoutKind for struct.

**Struct Boxing mechanism:**
- Memory allocated in heap;
- Memory size = length of struct + object type ref + sync block indes;
- Fields copied to heap;
- Ref for object returned.

**Struct unboxing mechanism:**
- get object ref;
- copy fields from heap to stack.

String concat uses a objects in args - so should be very attentive to writing code with boxing.

GetType - method from object, so value type be boxed.

Сasting to interface - value type be boxed.

Using ILDasm/sharplab.io to find boxing - good solution.

https://github.com/zolotarevandrew/.net-internals/blob/main/BoxingUnboxing/Program.cs

**ElasticSearch:**

Found and used word delimeter graph token filter, which splits tokens by non alphanumeric characters and by other good stuff.
for example: favorit—42+ to favorit, 42

Found and used shingle filter to produce n grams by concatenating.
for example:  ka - favorit, favoritka, ka.

Used wildcard search for contains string filter:
(*Query*) OR (Query*) OR (Query)

**.net LinkedList internal structure**

list is Doubly linked circular list, node has next and prev ref.

has mutable and immutable methods.

adding, deleting, searching - nothing special.

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/LinkedListInternal.cs


**.net SortedList internal structure**

Default capacity - zero. Then 4, then multiplied by 2.

Contains arrays of keys and values. Keys are unique.

Adding is done using a binary search. BinarySearch gives index to insert.

Binary search has interesting implementation with bitwise operators.


https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/SortedListInternal.cs

**.net HashSet internal structure**

Has slots array (hashcode, next entry index and value) and array of indexes.
Hash code using get hash code and bitwise and operator to store lower 31 bits of hash code. (need learn more)

Contains - gets hash code and iterate over slots.next items until found equal value.

Remove - same approach like contains.

Add - add value if its not present.

Additionally has a variable m_freeList, for fast insertion into free part of array.

Has method trim excess - Sets the capacity of this list to the size of the list (rounded up to nearest prime)

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/HashSetInternal.cs

**Pluses**:
- I can create more optimized classes under .net;
- I can search unnecessary boxing by ILDasm, to improve memory allocations.
- I can create search with ignoring spaces by elastic search shingle token filter.

[Log Index]

----------------------------------------------------------
----------------------------------------------------------

## 09 Jan 22

prologue code - set ups the stack frame of the called function.

epilogue code - restore the stack frame of calling parent function.

Another point to learn assembler.

**CLR simple method execution**
M1()
string name
M2(name)
- Process thread gets a 1 mb size stack;
- Name address pushed to stack - m1 local var;
- S address pushed to stack - m2 name parameter;
- Return address to M1 pushed to stack;
- M2 local variables pushed to stack;
- M2 code executed;
- Write return address into cpu instruction pointer.

**CLR method execution with objects.**
We have Employee and Manager class.
M3()
Employee e = new Manager()
e = Employee.Lookup()
e.GetProgressReport

**Then JIT stage comes**
- Found all types which M3 has;
- CLR load all assemblies for found types;
- Using metadata CLR creates objects types in heap;
- Each object type has - sync block index, object type ref, static fields, methods table. M3 compiles and then execution starts;
- Employee object created in heap;
- Object has sync block index, object type ref, bytes for fields and bytes for base type fields;
- Employee ref pushed to stack - variable e.

**Calling static method lookup**
- Search object type by ref;
- Search entrypoint to method by method table.

**Calling virtual method GetProgressReport**
- using variable go to address;
- search object type by ref;
- Search entrypoint to method by method table.

Object types has ref to System.Type which CLR creates at startup.
System.Type object type too and it refers to itself.

compiler has flag /checked+ prevent number overflows.

![clrexecution](https://user-images.githubusercontent.com/49956820/148688808-8a5cc92c-e986-4678-b532-cb259c82584b.png)

[Log Index]

----------------------------------------------------------

## 08 Jan 22

Put my github in order

[Log Index]

----------------------------------------------------------

[Log Index]: https://github.com/zolotarevandrew/my_learning_tracker/blob/main/log-index.md#log-index