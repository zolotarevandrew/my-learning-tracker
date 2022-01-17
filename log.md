# Learning log

|Date |                                        |
|:---:|:---------------------------------------|
|     |Learnt, thoughts, progress, ideas, links|
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