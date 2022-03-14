# Learning log

|Date |                                        |
|:---:|:---------------------------------------|
|     |Learnt, thoughts, progress, ideas, links|
----------------------------------------------------------
## 14 mar 22
**MongoDB replication**
A replica set in MongoDB is a group of mongod processes that maintain the same data set.
Replica sets provide redundancy and high availability.
Replication provides a level of fault tolerance against the loss of a single database server.

A replica s44et contains several data bearing nodes and optionally one arbiter node.
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