---
title: My tech note - Security
date: 2025-12-11 10:00:00 +1300
categories: [Tech dev note]
tags: [spring boot, java]
---

## What I've learned today



## Security Basic

Which encrypts and decrypts, Java or PHP? It depends on who has the sensitive data to protect.

Scenario 1: Java Encryption → PHP Decryption (What I'm doing now)
User enters password in the App
↓ 
Java encrypts "password123" → "sdvvzrug456"
↓
Sends to server
↓ 
PHP decrypts → Gets "password123" → Stores in database

Purpose: Protects user-sent data (login, payment, etc.)

Scenario 2: PHP Encryption → Java Decryption (Reverse)
Server has confidential data
↓ 
PHP encrypts "secret" → "vhfuhw"
↓ 
Sends to client
↓ 
Java decrypts → Gets "secret" → Displays to user

Purpose: Protects data returned by the server (API keys, personal information, etc.)

Scenario 3: Two-way Encryption

Java Encryption ────► PHP Decryption

Java Decryption ◄──── PHP Encryption

Purpose: Both parties encrypt communication (most secure)

Why choose Java → PHP as the course? The most common scenario is that the client sends sensitive data to the server:

Examples: Login users send passwords, payment users send credit card numbers, registered users send personal information.

These are all user-to-server interactions, so Java encrypts and PHP decrypts.

In short:
Whoever sends the sensitive data encrypts it.
It can be Java to PHP, or PHP to Java, depending on business requirements. RetryClaude is AI and can make mistakes. Please double-check responses.
