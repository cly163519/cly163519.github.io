---
layout: post
title: "Heap and Stack in Java
date: 2026-03-03
categories: [Computer components, hardware, software]
tags: [CPU, memory, MMIO, bootstrapping, binary]
lang: en
---

## Question: How do concepts like the stack and heap relate to the ALU? Is all memory solely divided into the heap and stack? Are there no other spaces? Do all programming activities involve manipulating the heap and stack?

Java programmers “feel like they're manipulating objects,” but the JVM actually operates on: the stack (stack frames for each thread) + the heap (objects/arrays) + the method area/metaspace (class information) + some native memory.
The CPU/ALU only handles address calculations and data processing; the JVM translates your code into these read/write operations.

### Below are answers to your three points.

## 1) What do Java's “stack” and “heap” store?
Stack: Each thread has its own.
Primarily contains stack frames. Each method call creates a stack frame holding:
Local variables: Primitive values (int/double/boolean, etc.) and object references.
Operand stack: Temporary calculations during bytecode execution
Return addresses and some bookkeeping information
✅ Key point: The stack generally does not hold object bodies; it only holds “references (addresses/handles)”.

Heap: Shared by all threads
The heap holds:
Objects created with `new`

Arrays
Object fields
✅ Key point: When you write `new Person()`, the “body” of the Person is on the heap.

## 2) Besides the heap and stack, what other “memory spaces” exist in Java?
Yes, and they're crucial:
Method Area / Metaspace
Class metadata: class names, method tables, field information, constant pools, bytecode, etc. (details vary across JVMs)
HotSpot JVM replaced PermGen (historical term) with Metaspace (in native memory) starting from Java 8

Native / Off-Heap Memory
Direct memory: ByteBuffer.allocateDirect()
Memory allocated via JNI calls to C/C++
Some JVM structures (JIT code cache, etc.)
Also: Registers/Caches (hardware layer)

These are not classified as “heap/stack” but genuinely exist and significantly impact performance:
CPU registers / L1 L2 L3 cache

## 3) Clarifying “References on the Stack, Objects on the Heap” with a Java Example

```Java
class Person { int age; }

void f() {
  int x = 3;
  Person p = new Person();
  p.age = x + 2;
}
```

# Runtime overview:
x is a primitive type, stored in the stack frame's local variable table (or optimized into a register)
p is a reference, stored in the stack frame's local variable table (like a “pointer/handle to a heap object”)
The object created by new Person() resides on the heap
Behind the line `p.age = x + 2`:
Read x (from stack/register)

Compute x + 2 (ALU performs addition)
Use p's reference to find the heap object's address (ALU often participates in address calculation: base + offset)
Write the result to the object's memory at the age field location (write to heap memory)
So the ALU's role is:
Calculate value (x+2)
Calculate address (object base address + field offset)

## 4) “Does all Java programming activity operate on the heap and stack?”

Most business logic state is indeed primarily located in:
Stack (local variables, references, temporary calculations)
Heap (objects, arrays)
But you also frequently use these indirectly:
Class metadata/constant pool (method area/Metaspace)
Native memory (direct buffers, JNI)
Kernel buffers (behind network/file I/O)
Device memory (e.g., libraries calling GPUs)
Java hides these details, allowing you to use them indirectly via APIs.

## 5) A particularly critical “Java misconception correction”
Many believe: “Local variable objects must reside on the stack.”
This isn't generally how it's described in Java. The correct understanding is:
Local variable “references” reside on the stack
Objects typically reside on the heap

However, during JVM optimizations (like escape analysis), certain objects may undergo “scalar replacement” and may not actually be allocated on the heap (this is an optimization detail that doesn't affect your understanding of the model).

**Completed with assistance from ChatGPT 5.2*



## 问题: 栈,堆这些概念, 和ALU什么关系? 所有的内存只分为堆和栈?没有其他空间?人们所有的编程活动, 都是在操作堆和栈?

Java 程序员“感觉自己在操作对象”，但 JVM 实际是在操作：栈（每个线程的栈帧）+ 堆（对象/数组）+ 方法区/元空间（类信息）+ 一些本地内存。
CPU/ALU 只负责算地址和算数据，JVM 把你的代码翻译成这些读写。

### 下面按你问的三点展开。

## 1) Java 里“栈”和“堆”分别放什么？
栈（Stack）：每个线程都有一条
栈里主要是栈帧（stack frame），每次方法调用都会创建一个栈帧，里面放：
局部变量表（local variables）：基本类型值（int/double/boolean 等），以及对象引用 reference
操作数栈（operand stack）：执行字节码时的临时计算
返回地址、一些 bookkeeping 信息
✅ 重点：栈里一般不放对象本体，只放“引用（地址/句柄）”。

堆（Heap）：所有线程共享
堆里放：
new 出来的对象

数组
对象的字段（fields）
✅ 重点：你写 new Person()，Person 的“本体”在堆上。

## 2) Java 里除了堆和栈，还有什么“内存空间”？
有，而且很重要：
方法区 / 元空间（Metaspace）
类的元数据：类名、方法表、字段信息、常量池、字节码等（不同 JVM 细节不同）
HotSpot JVM 从 Java 8 起用 Metaspace（在本地内存里）替代了永久代 PermGen（历史名词）

本地内存（Native / Off-Heap）
直接内存：ByteBuffer.allocateDirect()
JNI 调用 C/C++ 分配的内存
JVM 自己的一些结构（JIT 代码缓存等）
还有：寄存器/缓存（硬件层）

这些不按“堆/栈”划分，但真实存在，并且性能影响巨大：
CPU registers / L1 L2 L3 cache

## 3) 用一个 Java 例子把“引用在栈、对象在堆”讲透

```Java
class Person { int age; }

void f() {
  int x = 3;
  Person p = new Person();
  p.age = x + 2;
}
```

# 运行时大致是：
x 是基本类型，存在 栈帧的局部变量表里（或被优化到寄存器）
p 是引用，存在 栈帧局部变量表里（像一个“指向堆对象的指针/句柄”）
new Person() 产生的对象本体在 堆 上
p.age = x + 2 这句背后发生了：
读 x（栈/寄存器）

计算 x + 2（ALU 做加法）
用 p 的引用找到堆上对象地址（ALU 常参与地址计算：base + offset）
把结果写到对象内存里的 age 字段位置（写堆内存）
所以 ALU 的关系是：
算值（x+2）
算地址（对象起始地址 + 字段偏移）

## 4) “Java 所有编程活动都在操作堆和栈吗？”

大部分业务代码的状态确实主要落在：
栈（局部变量、引用、临时计算）
堆（对象、数组）
但你也经常间接在用：
类元数据/常量池（方法区/Metaspace）
本地内存（direct buffer、JNI）
内核缓冲区（网络/文件 IO 背后）
设备内存（比如某些库调用 GPU）
只是 Java 把这些细节藏起来了，你通过 API 间接使用。

## 5) 一个特别关键的“Java误区纠正”
很多人以为：“局部变量对象一定在栈上”
在 Java 里一般不这么说。正确理解是：
局部变量“引用”在栈上
对象通常在堆上

但 JVM 有优化（逃逸分析）时，某些对象可能被“标量替换”，甚至不真的在堆上分配（这属于优化细节，不影响你理解模型）

**在ChatGPT5.2协助下完成*
