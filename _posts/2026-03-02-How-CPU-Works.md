# How Does a CPU Work? The 3-Step Cycle: Fetch, Decode, Execute

> A beginner-friendly guide to understanding what a CPU actually does, explained as simply as possible.

---

## A Simple Setup

![Uploading image.png…]()


Imagine memory as a row of numbered boxes. Each box can hold either **data** or an **instruction**.

```
Box Number    Content
    10          2          ← data
    11          3          ← data
    12          0          ← data (empty, waiting to be written)
    ...
   100        LOAD 10     ← instruction
   101        ADD  11     ← instruction
   102        STORE 12    ← instruction
```

The CPU's job: start from address 100, read instructions one by one, and do what they say.

---

## Why Start at Address 100, Not Address 10?

Because `2`, `3`, `0` in the data boxes are just **nouns**. The CPU sees them and has no idea what to do with them.

`LOAD 10`, `ADD 11`, `STORE 12` are **verb + noun** combinations. They tell the CPU "what to do" and "to whom".

The CPU needs to see the verb first before it can work with the nouns.

So how does the CPU know to start at 100? That's the job of the **PC (Program Counter)** — it's like a bookmark that says "the next instruction is here". When the program starts, PC is set to 100. After each instruction, PC automatically increments by 1.

---

## The Cast of Characters Inside the CPU

| Role | Full Name | What It Does | Analogy |
|------|-----------|-------------|---------|
| **CU** | Control Unit | Directs all other components | The manager |
| **PC** | Program Counter | Remembers where the next instruction is | A bookmark |
| **MAR** | Memory Address Register | Holds the memory address to access | The house number in the assistant's hand |
| **MDR** | Memory Data Register | Carries data between CPU and memory | The delivery window |
| **CIR** | Current Instruction Register | Holds the instruction being processed | The work order on the manager's desk |
| **ACC** | Accumulator | Stores the calculation result | The manager's desk |
| **ALU** | Arithmetic Logic Unit | Does the math | A calculator |

---

## Instruction 1: LOAD 10 (Read data from address 10)

### Step 1: Fetch — Go get the instruction

```
PC = 100          → "Next instruction is at address 100"
MAR ← 100         → Tell memory: I need box 100
Memory sends LOAD 10 to MDR
MDR → CIR         → Instruction LOAD 10 arrives at CIR
PC = 101           → Bookmark moves to the next page
```

The manager asks the assistant to fetch a work order from cabinet 100. The assistant brings it back and places it on the manager's desk.

### Step 2: Decode — Understand the instruction

```
CIR contains LOAD 10
CU breaks it apart:
  Opcode  = LOAD → The action is "read"
  Operand = 10   → The target is "address 10"
```

The manager reads the work order and understands: "I need to get the data from box 10." This step happens entirely inside the CU — the door is closed, no one else is involved.

### Step 3: Execute — Make it happen

```
CU tells MAR ← 10           → Assistant notes down house number 10
CU tells memory: I want to read → Memory sends the value at address 10 (which is 2) to MDR
MDR value (2) → ACC          → Data 2 is placed on the manager's desk
```

The manager opens the door and starts making calls. At the end, ACC = 2.

---

## Instruction 2: ADD 11 (Add the data from address 11)

### Fetch

```
PC = 101 → MAR ← 101 → Memory returns ADD 11 → CIR = ADD 11 → PC = 102
```

### Decode

```
CU breaks it apart: ADD = addition, 11 = address 11
```

### Execute

```
MAR ← 11 → Memory returns value at address 11 (which is 3) → MDR = 3
ALU adds ACC (2) and MDR (3) → 2 + 3 = 5
ACC = 5
```

---

## Instruction 3: STORE 12 (Write the result back to address 12)

### Fetch

```
PC = 102 → MAR ← 102 → Memory returns STORE 12 → CIR = STORE 12 → PC = 103
```

### Decode

```
CU breaks it apart: STORE = write, 12 = address 12
```

### Execute (This is the step that confuses most beginners!)

```
MAR ← 12              → Target address is 12
MDR ← ACC value (5)   → The data to write is 5
MDR sends 5 → written into memory address 12
```

**Key point:** MDR does NOT read the 0 from memory address 12. Instead, it goes the other direction — it takes the 5 from ACC and writes it INTO memory. That's because STORE means **CPU → Memory**, not Memory → CPU.

Simple rule:

- **LOAD** = Memory → CPU (read) — walk in and pick something up
- **STORE** = CPU → Memory (write) — walk in and put something down

After execution, memory address 12 changes from 0 to 5.

---

## The Final Result

```
Address 10: 2    (unchanged)
Address 11: 3    (unchanged)
Address 12: 5    (changed from 0 to 5)
```

The CPU just solved a simple math problem: **2 + 3 = 5**.

---

## The Big Picture

A CPU does three things, over and over, forever:

```
┌──────────────────────────────────┐
│                                  │
│   Fetch    →  Go get instruction │
│      ↓                           │
│   Decode   →  Understand it      │
│      ↓                           │
│   Execute  →  Do it              │
│      ↓                           │
│   Back to Fetch, next one ...    │
│                                  │
└──────────────────────────────────┘
```

From the simplest 8-bit microcontroller to the 64-bit Intel i5 in your laptop, every CPU does these same three things. The only difference is speed and scale:

- **8-bit microcontroller**: 256 memory boxes, a few MHz, millions of cycles per second
- **Your laptop (i5-1235U)**: 17 billion memory boxes, 1.3 GHz, 1.3 billion cycles per second

The principle never changed. Only the scale did.

---

*This article was written based on real questions I had while learning how CPUs work from scratch. Completed with the assistance of Claude Opus4.6.*



# CPU是怎么工作的？三步循环：取指、解码、执行

> 用最简单的话，给零基础的你讲清楚CPU到底在干什么。

---

## 一个小故事

假设内存是一排带编号的格子，每个格子里可以放**数据**或者**指令**。

```
格子编号    内容
  10        2         ← 数据
  11        3         ← 数据
  12        0         ← 数据（空的，等着被写入）
  ...
 100        LOAD 10   ← 指令
 101        ADD  11   ← 指令
 102        STORE 12  ← 指令
```

CPU的任务是：从地址100开始，一条一条读指令，然后照着做。

---

## 为什么从地址100开始，而不是直接去读地址10？

因为格子里的 `2`、`3`、`0` 只是**名词**，CPU看到它们不知道该干什么。

而 `LOAD 10`、`ADD 11`、`STORE 12` 是**动词+名词**，告诉CPU"做什么"和"对谁做"。

CPU必须先看到动词，才知道怎么处理名词。

那CPU怎么知道从100开始？答案是**PC（程序计数器）**，它就像一个书签，告诉CPU"下一条指令在哪"。程序启动时PC被设为100，每执行完一条指令PC自动+1。

---

## CPU里有哪些角色？

| 角色 | 全名 | 干什么 | 类比 |
|------|------|--------|------|
| **CU** | Control Unit 控制单元 | 指挥所有人干活 | 总经理 |
| **PC** | Program Counter 程序计数器 | 记住下一条指令的地址 | 书签 |
| **MAR** | Memory Address Register | 存放要访问的内存地址 | 助理手里的门牌号 |
| **MDR** | Memory Data Register | 搬运数据的中转站 | 传递窗口 |
| **CIR** | Current Instruction Register | 存放当前正在处理的指令 | 总经理桌上的工单 |
| **ACC** | Accumulator 累加器 | 存放计算结果 | 总经理的办公桌 |
| **ALU** | Arithmetic Logic Unit | 做数学运算 | 计算器 |

---

## 第一条指令：LOAD 10（把地址10的数据读出来）

### Step 1：Fetch（取指）— 去拿指令

```
PC = 100        → "下一条指令在地址100"
MAR ← 100       → 告诉内存：我要找100号格子
内存把100号格子里的 LOAD 10 送到 MDR
MDR → CIR       → 指令 LOAD 10 送到CIR
PC = 101         → 书签翻到下一页
```

就像总经理让助理去100号档案柜取工单，助理取回来放在总经理桌上。

### Step 2：Decode（解码）— 看懂指令

```
CIR里是 LOAD 10
CU拆开理解：
  操作码 = LOAD → 动作是"读取"
  操作数 = 10   → 目标是"地址10"
```

总经理看了一眼工单，明白了："要去10号格子取数据"。这一步是CU自己在想，门关着，没跟任何人说话。

### Step 3：Execute（执行）— 指挥干活

```
CU告诉MAR ← 10        → 助理记下门牌号10
CU告诉内存：我要读数据   → 内存把地址10的值（2）送到MDR
MDR的值（2）→ ACC       → 数据2放到办公桌上
```

总经理打开门，开始打电话派任务。最终，ACC = 2。

---

## 第二条指令：ADD 11（把地址11的数据加上来）

### Fetch

```
PC = 101 → MAR ← 101 → 内存送回 ADD 11 → CIR = ADD 11 → PC = 102
```

### Decode

```
CU拆开：ADD = 加法，11 = 地址11
```

### Execute

```
MAR ← 11 → 内存送回地址11的值（3）→ MDR = 3
ALU把ACC的2和MDR的3相加 → 2 + 3 = 5
ACC = 5
```

---

## 第三条指令：STORE 12（把结果存回地址12）

### Fetch

```
PC = 102 → MAR ← 102 → 内存送回 STORE 12 → CIR = STORE 12 → PC = 103
```

### Decode

```
CU拆开：STORE = 写入，12 = 地址12
```

### Execute（这一步最容易困惑！）

```
MAR ← 12          → 目标地址是12
MDR ← ACC的值（5） → 要写入的数据是5
MDR的5 → 写入内存地址12
```

**注意：** 这里MDR不是去内存读地址12里的0，而是反过来——把ACC的5通过MDR写进去。因为STORE的方向是 **CPU → 内存**，不是内存 → CPU。

简单记忆：

- **LOAD** = 内存 → CPU（读）— 进门取东西
- **STORE** = CPU → 内存（写）— 进门放东西

执行完毕后，内存地址12从0变成了5。

---

## 最终结果

```
地址10：2       （没变）
地址11：3       （没变）
地址12：5       （从0变成了5）
```

CPU完成了一道小学算术题：**2 + 3 = 5**。

---

## 全局总结

CPU永远在做三件事，不断循环：

```
┌─────────────────────────────────┐
│                                 │
│   Fetch    →  去拿指令           │
│      ↓                          │
│   Decode   →  看懂指令           │ 
│      ↓                          │
│   Execute  →  照着做             │
│      ↓                          │
│   回到 Fetch，拿下一条指令 ...    │
│                                 │
└─────────────────────────────────┘
```

从最简单的单片机到你电脑里的64位Intel i5，干的都是这三件事。区别只是速度和规模：

- **8位单片机**：256个格子，几MHz，每秒几百万次循环
- **你的电脑（i5-1235U）**：170亿个格子，1.3GHz，每秒13亿次循环

原理没变，规模大了而已。

---

*这篇文章基于我学习CPU基础知识时的真实问题和思考整理而成。由Claude Opus4.6协助完成*
