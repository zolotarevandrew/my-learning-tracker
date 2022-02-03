# Learning log

|Date |                                        |
|:---:|:---------------------------------------|
|     |Learnt, thoughts, progress, ideas, links|
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