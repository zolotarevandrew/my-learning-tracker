# Learning log

|Date |                                        |
|:---:|:---------------------------------------|
|     |Learnt, thoughts, progress, ideas, links|

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
----------------------------------------------------------

## 08 Jan 22

Put my github in order

[Log Index]

----------------------------------------------------------

[Log Index]: https://github.com/zolotarevandrew/my_learning_tracker/blob/main/log-index.md#log-index