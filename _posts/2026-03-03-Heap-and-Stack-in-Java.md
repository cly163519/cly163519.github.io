---
layout: post
title: "Heap and Stack in Java"
date: 2026-03-03
categories: [computer-architecture, hardware, software]
tags: [cpu, memory, jvm, heap, stack, alu]
lang: en
permalink: /en/heap-and-stack-in-java/
---

# Heap and Stack in Java

## Question
How do concepts like the stack and heap relate to the ALU?  
Is all memory solely divided into the heap and stack? Are there no other spaces?  
Do all programming activities involve manipulating the heap and stack?

Java programmers often feel like they are “manipulating objects,” but the JVM actually operates on:

- **Stack** (stack frames for each thread)
- **Heap** (objects/arrays)
- **Method area / Metaspace** (class information)
- **Some native (off-heap) memory**

The CPU/ALU mainly handles **address calculations** and **data processing**. The JVM translates your code into these read/write operations.

---

## Table of Contents
- [1) What do Java's stack and heap store?](#1-what-do-javas-stack-and-heap-store)
- [2) Besides heap and stack, what other memory spaces exist in Java?](#2-besides-heap-and-stack-what-other-memory-spaces-exist-in-java)
- [3) Example: references on the stack, objects on the heap](#3-example-references-on-the-stack-objects-on-the-heap)
- [4) Does all Java programming activity operate on the heap and stack?](#4-does-all-java-programming-activity-operate-on-the-heap-and-stack)
- [5) Common misconception](#5-common-misconception)
- [中文版本](#中文版本)

---

## 1) What do Java's stack and heap store?

### Stack (per thread)
Each thread has its own stack. It mainly contains **stack frames**.  
Each method call creates a stack frame holding:

- **Local variables**: primitive values (int/double/boolean, etc.) and **object references**
- **Operand stack**: temporary values used during bytecode execution
- Return information and bookkeeping data

**Key point:** the stack usually does **not** store the object body. It stores **references** (addresses/handles).

### Heap (shared)
The heap is shared by all threads. It holds:

- Objects created with `new`
- Arrays
- Object fields

**Key point:** when you write `new Person()`, the object body is on the heap.

---

## 2) Besides heap and stack, what other memory spaces exist in Java?

### Method Area / Metaspace
- Class metadata: class names, method tables, field info, constant pools, bytecode, etc.
- HotSpot JVM replaced PermGen with **Metaspace** starting from Java 8 (Metaspace uses native memory)

### Native / Off-Heap Memory
- Direct memory: `ByteBuffer.allocateDirect()`
- Memory allocated by JNI (C/C++)
- JVM internal structures (e.g., JIT code cache)

### (Hardware layer) Registers and Caches
These are not classified as heap/stack, but they exist and affect performance:

- CPU registers
- L1 / L2 / L3 cache

---

## 3) Example: references on the stack, objects on the heap


```java
class Person { int age; }

void f() {
  int x = 3;
  Person p = new Person();
  p.age = x + 2;
}
```

Runtime overview:

x is a primitive value: stored in the stack frame’s local variables (or optimized into a register)
p is a reference: stored in the stack frame’s local variables (like a pointer/handle to a heap object)
The object created by new Person() resides on the heap
Behind the line p.age = x + 2:
Read x (from stack/register)
Compute x + 2 (ALU performs addition)
Use p to find the object’s address (ALU often helps compute addresses: base + offset)
Write the result into the object field age (write to heap memory)
So the ALU’s role is mainly:
Calculate values (x + 2)
Calculate addresses (object base address + field offset)

---

## 4) Does all Java programming activity operate on the heap and stack?
Most business logic state is indeed mainly located in:
Stack (local variables, references, temporary values)
Heap (objects, arrays)
But Java programs also use other areas indirectly:
Class metadata / constant pools (method area / Metaspace)
Native memory (direct buffers, JNI)
Kernel buffers (behind file/network I/O)
Device memory (e.g., libraries that use GPUs)
Java hides many of these details, and you access them via APIs.

## 5) Common misconception
Many people believe: “Local variable objects must reside on the stack.”
In Java, a more accurate model is:
Local variable references are on the stack
Objects typically live on the heap

Note: with JVM optimizations (escape analysis, scalar replacement), some objects may not actually be allocated on the heap. This is an optimization detail and does not change the basic mental model.

Completed with assistance from ChatGPT 5.2


## 问题: 栈,堆这些概念, 和ALU什么关系?所有的内存只分为堆和栈?没有其他空间?人们所有的编程活动, 都是在操作堆和栈?
Java 程序员“感觉自己在操作对象”，但 JVM 实际是在操作：栈（每个线程的栈帧）+ 堆（对象/数组）+ 方法区/元空间（类信息）+ 一些本地内存。
CPU/ALU 只负责算地址和算数据，JVM 把你的代码翻译成这些读写。

### 下面按你问的三点展开。
## 1) Java 里“栈”和“堆”分别放什么？
栈（Stack）：每个线程都有一条 栈里主要是栈帧（stack frame），每次方法调用都会创建一个栈帧，里面放： 局部变量表（local variables）：基本类型值（int/double/boolean 等），以及对象引用 reference 操作数栈（operand stack）：执行字节码时的临时计算 返回地址、一些 bookkeeping 信息
✅ 重点：栈里一般不放对象本体，只放“引用（地址/句柄）”。
堆（Heap）：所有线程共享 堆里放： new 出来的对象 数组 对象的字段（fields）
✅ 重点：你写 new Person()，Person 的“本体”在堆上。

## 2) Java 里除了堆和栈，还有什么“内存空间”？
有，而且很重要：
方法区 / 元空间（Metaspace） 类
的元数据：类名、方法表、字段信息、常量池、字节码等（不同 JVM 细节不同）
HotSpot JVM 从 Java 8 起用 Metaspace（在本地内存里）替代了永久代 PermGen（历史名词）
本地内存（Native / Off-Heap）
直接内存：ByteBuffer.allocateDirect() JNI 调用 C/C++ 分配的内存 JVM 自己的一些结构（JIT 代码缓存等）
还有：寄存器/缓存（硬件层） 这些不按“堆/栈”划分，但真实存在，并且性能影响巨大： CPU registers / L1 L2 L3 cache ## 3) 用一个

Java 例子把“引用在栈、对象在堆”讲透:

```
Java
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
new Person() 产生的对象本体在 堆 上 p.age = x + 2
这句背后发生了： 读 x（栈/寄存器） 计算 x + 2（ALU 做加法） 用 p 的引用找到堆上对象地址（ALU 常参与地址计算：base + offset）
把结果写到对象内存里的 age 字段位置（写堆内存）
所以 ALU 的关系是： 算值（x+2） 算地址（对象起始地址 + 字段偏移）

## 4) “Java 所有编程活动都在操作堆和栈吗？”
大部分业务代码的状态确实主要落在： 栈（局部变量、引用、临时计算） 堆（对象、数组） 但你也经常间接在用： 类元数据/常量池（方法区/Metaspace） 本地内存（direct buffer、JNI） 内核缓冲区（网络/文件 IO 背后） 设备内存（比如某些库调用 GPU） 只是 Java 把这些细节藏起来了，你通过 API 间接使用。

## 5) 一个特别关键的“Java误区纠正”
很多人以为：“局部变量对象一定在栈上” 在 Java 里一般不这么说。
正确理解是： 局部变量“引用”在栈上 对象通常在堆上 但 JVM 有优化（逃逸分析）时，某些对象可能被“标量替换”，甚至不真的在堆上分配（这属于优化细节，不影响你理解模型）


**在ChatGPT5.2协助下完成*


