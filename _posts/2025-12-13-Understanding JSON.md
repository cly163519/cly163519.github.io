---
title: Understanding JSON in Client-Server Communication
date: 2025-12-14 10:00:00 +1300
categories: [programming, Java]
tags: [java, json, networking, php]
---

## Traditional Web Development (What I learned before)
```
JS (Frontend/Client)  ←──JSON──→  Java (Backend/Server)
      │                                │
   Send request                    Generate data
   Display data                    Return JSON
```

**Java generates data → JSON packages it → JS receives it**

## What I'm Learning Now (Network Programming)
```
Java (Client)  ←──JSON──→  PHP (Server)
      │                         │
   Send request             Generate data
   Parse data               Return JSON
```

**PHP generates data → JSON packages it → Java receives it**

## Key Concepts

| Concept | Description |
|---------|-------------|
| Client | Whoever **sends the request** is the client |
| Server | Whoever **receives the request and returns data** is the server |
| JSON | **Universal translator** - any language can read it |

## Why Do We Need JSON?
```
PHP data format  ──✗──→  Java cannot read directly
PHP data format  ──→ JSON ──→  Java can read ✓

Java data format ──✗──→  JS cannot read directly
Java data format ──→ JSON ──→  JS can read ✓
```

**JSON = Universal translator that everyone can understand** 🌐

## Summary

| Scenario | Client | Server | JSON Direction |
|----------|--------|--------|----------------|
| Traditional Web | JS | Java | Java → JSON → JS |
| Current Course | Java | PHP | PHP → JSON → Java |
| Can also reverse | Java | PHP | Java → JSON → PHP |

## One Sentence Summary

**JSON is a universal data format. Whoever generates the data converts it to JSON, and the receiver parses the JSON.**

The key insight: In this course, Java becomes the client and PHP becomes the server - the opposite of traditional web development!
