---
title: Java Basic Syntax - Wellington Train project exercise
date: 2025-12-13 17:03:00 +1300
categories: [Syntax, Java]
tags: [java, syntax, OOP, Core API]
---

## Wellington Train project

# Phase 1: Read the requirements and highlight key points

First reading: What is it roughly about?

"Wellington Trains is developing a program to answer queries about stations, lines, and timetables."
My understanding:
This is a query system.
Users ask questions, the program answers.
It's related to trains.
Second reading: Underline keywords.
I circled the important words:
Processing three types of information:

* Train stations ← First type
* Train lines ← Second type
* Train services ← Third type

Train stations:

* Name ← Stations have names
* Fare zones ← Stations have zones
* Distance ← Stations have distances

Train lines:

* A series of stations ← A line contains multiple stations
* Order ← Stations have a specific order
* Entering and exiting the station are on two separate lines ← Different directions = different lines

Train services:

* Specific trains run on specific lines ← Services belong to a certain line
* Timetable ← Time at each station
*  -1 means no stops ← Some stations have no stops

Third reading: Look at what the data file looks like

stations.data: Croftton-Towns 3 4.9

→ Three columns: Name, Region, Distance, separated by spaces

train-lines.data:
Melling_Wellington

→ Line name, origin_endpoint

Wellington_Johnsonville-stations.data: One station name per line, in order

Wellington_Johnsonville-services.data: Each line is a train timetable; 1437 means 14:37, -1 means no stop

# Phase 2

introduction

```java

// this is java block

```

# Phase 3

1. this is one
2. this is two
3. this is thress

# Phase 4

java api

```java

```
