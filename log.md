# Learning log

|Date |                                        |
|:---:|:---------------------------------------|
|     |Learnt, thoughts, progress, ideas, links|
----------------------------------------------------------

## 14 Jan 22
**.net**

Default parameters stores in metadata, so compiler can add them at execution.
params always allocate some space in heap, so better not to use it.

**ElasticSearch:**

Finally implemented search, still dont understand why edge n gram did not solve the problem for contains queries.
Instead wilcard queries with boost was used.

surf coff - (surf* and coff*)^3 or (surf*)^2 or (coff*)^1.

5:08
So the problem was with docs in elastic search, they don't put this "Sometimes, though, it can make sense to use a different analyzer at search time, such as when using the edge_ngram tokenizer for autocomplete or when using search-time synonyms."
into edge n gram docs. 

I just changed search_analyzer to standard. And my query became simple - "query": "surf coff".
So now search became faster, because of using index correctly.


**.net stack**

has a simple array implementation.
Capacity started from 4, and then multiply by 2.

**.net queue**

has a simple array implementation.
has _head and _tail index.
Capacity started from 4, and then multiply by 2.
On enqueue, then increment _tail my method with ref index parameter. 
Has a comment (JIT produces better code than with ternary operator ?:) - i think it is  branch to target operator, should learn more about this.

**.net list**

has a simple array implementation.
Capacity started from 4, and then multiply by 2.
Interesting, they avoid the extra generic instantiation for Array.Empty<T>() - by static instantiation of new T[0];


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

So the problem was with docker container and chrome in it.

Docker container has 512 default pids. But my container created over 1000, because of chrome driver starting by each request.
After execution stops a lot of pids, more than 500 just hang.  

Tested on local docker.

So should never span a lot of processes inside container. Knew it, but just thought it would be okay.

Instead selenoid was used and problem was solved.

**c# records, structs, classes**:

record is just wrapper around class, has some additional methods, and implements iequatable.
record created like this record MyRecord(string Name) - has a deconstruct method, and init only setters.

simple struct has no equality operator ==, should only use Equals (no boxing).
record struct implements IEquatable and has equality operator ==, useful for DDD ValueObjects.

operator with for structs - copy struct and changes values.
operator with for classes, records - copy class by Clone Method and changes values.

readonly ref record struct - useful for immutable DDD ValueObjects.
ref struct - lives only stack, useful for low optimizations.

https://github.com/zolotarevandrew/.net-internals/tree/main/ClassesStructsRecords

**.net**

IL has call and callvirt methods. callvirt called polymorphically.

Sealed class cant use inheritance.
So compiler can optimize virtual methods, such as ToString.

[Log Index]
----------------------------------------------------------
----------------------------------------------------------

## 10 Jan 22

Should learn more about LayoutKind for struct.

**Struct Boxing mechanism:**
- Allocated memory in heap
- Memory size = length of struct + object type ref + sync block indes
- Fields copied to heap
- Ref for object returned

**Struct unboxing mechanism:**
- get object ref
- copy fields from heap to stack

String concat using a objects in args - so should be very attentive to writing code with boxing unboxing

GetType - method from object, so value type be boxed

Сasting to interface - value type be boxed

Using ILDasm to find boxing - good solution.

https://github.com/zolotarevandrew/.net-internals/blob/main/BoxingUnboxing/Program.cs

**ElasticSearch:**

Found and used word delimeter graph token filter, which splits tokens by  non alphanumeric characters and by other good stuff.
for example: favorit—42+ to favorit, 42

Found and used shingle filter to produce n grams by concatenating.
for example:  ka - favorit, favoritka, ka.

Used wildcard search for contains string filter:
(*Query*) OR (Query*) OR (Query)

**.net LinkedList internal structure**

list is Doubly linked circular list, node has next and prev ref.

has mutable and immutable methods.

adding, deleting, searching - nothing special

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/LinkedListInternal.cs


**.net SortedList internal structure**

Default capacity - zero. Then 4, then multiplied by 2.

Capacity increased only after full filling.

Removing - nothing special. Capacity not decreasing (optimization?)

Contains arrays of keys and values. Keys are unique.

Adding is done using a binary search. BinarySearch gives index to insert.

Binary search has interesting implementation with bitwise operators.

If key exists exception throws.

https://github.com/zolotarevandrew/.net-internals/blob/main/DataStructuresInternals/SortedListInternal.cs

**.net HashSet internal structure**

Has slots array (hashcode, next entry index and value) and array of indexes.
Hash code using get hash code and bitwise and operator to store lower 31 bits of hash code. (need learn more)/

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
- Process thread gets a 1mb size stack.
- Name address pushed to stack - m1 local var
- S address pushed to stack - m2 parameter
- Return address M1 pushed to stack
- M2 local variables pushed to stack
- M2 code executed
- Write return address into cpu instruction pointer

**CLR method execution with objects.**
We have Employee and Manager class.
M3()
Employee e = new Manager()
e = Employee.Lookup()
e.GetProgressReport

**Then JIT stage comes**
- Found all types which M3 has
- CLR load all assemblies for found types
- Using metadata CLR creates objects types in heap
- Each object type has - sync block index, object type ref, static fields, methods table.
 M3 compiles and then execution starts
- Employee object created in heap
- Object has sync block index, object type ref, bytes for fields and bytes for base type fields
-  Employee ref pushed to stack - variable e

**Calling static method lookup**
- Search object type by ref
- Search entrypoint to method by method table

**Calling virtual method GetProgressReport**
- using variable go to address
- search object type by ref
- Search entrypoint to method by method table

Object types has ref to System.Type which CLR creates at startup.
System.Type object type too and it refers to itself

compiler has flag /checked+ prevent number overflows

![clrexecution](https://user-images.githubusercontent.com/49956820/148688808-8a5cc92c-e986-4678-b532-cb259c82584b.png)

[Log Index]

----------------------------------------------------------

## 08 Jan 22

Put my github in order

[Log Index]

----------------------------------------------------------

[Log Index]: https://github.com/zolotarevandrew/my_learning_tracker/blob/main/log-index.md#log-index