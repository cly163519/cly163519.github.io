---
title: My tech note - Security
date: 2025-12-11 10:00:00 +1300
categories: [Tech dev note]
tags: [security, java]
---

## Security Basic: java vs php

Which encrypts and decrypts, Java or PHP? It depends on who has the sensitive data to protect.

**Scenario 1: Java Encryption → PHP Decryption**

```
User enters password in the App
↓ 
Java encrypts "password123" → "sdvvzrug456"
↓
Sends to server
↓ 
PHP decrypts → Gets "password123" → Stores in database
```

Purpose: Protects user-sent data (login, payment, etc.)

**Scenario 2: PHP Encryption → Java Decryption (Reverse)**

```
Server has confidential data
↓ 
PHP encrypts "secret" → "vhfuhw"
↓ 
Sends to client
↓ 
Java decrypts → Gets "secret" → Displays to user
```

Purpose: Protects data returned by the server (API keys, personal information, etc.)

**Scenario 3: Two-way Encryption**

```
Java Encryption ────► PHP Decryption

Java Decryption ◄──── PHP Encryption

```

Purpose: Both parties encrypt communication (most secure)

Why choose Java → PHP as the course? The most common scenario is that the client sends sensitive data to the server:

Examples: Login users send passwords, payment users send credit card numbers, registered users send personal information.

These are all user-to-server interactions, so Java encrypts and PHP decrypts.

In short:
Whoever sends the sensitive data encrypts it.
It can be Java to PHP, or PHP to Java, depending on business requirements. RetryClaude is AI and can make mistakes. Please double-check responses.

## Basic Cipher Implementation

Caeser Cipher

```java
          public class CaesarCipher {
      	public static String encrypt(String message, int offset) {
      		String result = "";
      //		for(int i = 0; i <message.length() ; i++) {
      //			char c = message.charAt(i);
      //			char newChar = (char) (c + offset);
      //			result += newChar;
      //		}
      		char[] chars = message.toCharArray();
      		for(int i = 0; i < chars.length; i++) {
      			char c = chars[i];
      			char newChar = (char) (c + offset);
      			result += newChar;
      		}
      		return result;
      	}
      	public static void main(String[] args) {
      		int offset = 3;
      		String message = "Hello from Java!";	
      		String encrypted = encrypt(message,offset);
      		System.out.println(encrypted);
      		String decrypted = encrypt(message,-offset);
      		System.out.println(decrypted);
      
      	}
      }
```

Transposition Cipher

```java

        public static void main(String[] args) {
        try{
            String plaintext = "JavaCloud";
            int[] key = {8, 1, 3, 0, 2, 6, 4, 7, 5};

            String ciphertext = encrypt(plaintext, key);

            PrintWriter out = new PrintWriter("ciphertext.txt");
            out.println(ciphertext);
            out.close();

        }catch (Exception e) {
                e.printStackTrace();
        }
    }
    public static String encrypt(String text, int[] key){
        String result = "";
        for(int i = 0; i < key.length; i++) {
            int index = key[i];
            result += text.charAt(index);
        }
        return result;
```

## Encrypted communication between Java(client) and PHP(server)

```java

    public class SecurityClient {
      public static void main(String[] args) {
        try{

            //1. Prepare data
            String message = "Hello from Java!";
            int offset = 3;

            int[] key = {15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0};
            String encryptedMessage = TranspositionCipher.encrypt(message, key);


            // 2. Encrypt data
            String encrypted = encrypt(message,offset);
		    System.out.println(encrypted);
            String decrypted = decrypt(encrypted, offset);
            System.out.println(decrypted);

            //3. Send to php server
            String encoded = URLEncoder.encode(encrypted, "UTF-8");
            String queryUrl = "http://localhost/cloud_experiments/security_client.php?data="+encoded;    
            URL url = new URI(queryUrl).toURL();
            HttpURLConnection connection = (HttpURLConnection)url.openConnection();
            
            // 4.Read the response
            BufferedReader reader = new BufferedReader(
                    new InputStreamReader(connection.getInputStream(), "utf-8")
                );
                
                String line;
                while ((line = reader.readLine()) != null) {
                    System.out.println(line);
                }
                reader.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public static String encrypt(String message, int offset) {
		String result = "";
		char[] chars = message.toCharArray();
		for(int i = 0; i < chars.length; i++) {
			char c = chars[i];
			char newChar = (char) (c + offset);
			result += newChar;
		}
		return result;
	}
    public static String decrypt(String message, int offset) {
        return encrypt(message, -offset);
    }
}

```

```php

    <?php
    if (isset($_GET['data'])) {
        $encrypted = $_GET['data'];
        echo "Received encrypted data: " . $encrypted;
    
        // // Encryption function
        // function encrypt($text, $offset) {
        //     $result = "";
        //     for ($i = 0; $i < strlen($text); $i++) {
        //         $char = $text[$i];
        //         $newChar = chr(ord($char) + $offset); 
        //         $result .= $newChar;
        //     }
        //     return $result;
        // }
    
        // Decryption function
        function decrypt($text, $offset) {
            $result = "";
            for ($i = 0; $i < strlen($text); $i++) {
                $char = $text[$i];
                $newChar = chr(ord($char) - $offset); 
                $result .= $newChar;
            }
            return $result;
        }
    
        // $encrypted = encrypt($encrypted, 3);
        // echo "Encrypted data: " . $encrypted . "\n";
    
        $decrypted = decrypt($encrypted, 3);
        echo "Decrypted data: " . $decrypted;
    } else {
        echo "No data received";
    }
?>

```

##  Clear core concept - Complete Data Flow
```
Step 1: Java prepares message
┌─────────────────────────────┐
│  message = "Hello from Java!" 
└─────────────────────────────┘
              │
              ▼
Step 2: Java encrypts (offset +3)
┌─────────────────────────────┐
│  encrypted = "Khoor#iurp#Mdyd$" 
└─────────────────────────────┘
              │
              ▼
Step 3: Java sends encrypted data
┌─────────────────────────────┐
│  URL: server.php?data=Khoor... 
│         (via network)        
└─────────────────────────────┘
              │
              ▼
Step 4: PHP receives encrypted data
┌─────────────────────────────┐
│  $_GET['data'] = "Khoor..."  
└─────────────────────────────┘
              │
              ▼
Step 5: PHP decrypts (offset -3)
┌─────────────────────────────┐
│  decrypted = "Hello from Java!" 
└─────────────────────────────┘
              │
              ▼
Step 6: PHP sends confirmation
┌─────────────────────────────┐
│  echo "Decrypted: Hello..."  
└─────────────────────────────┘
```

---


**Both sides must know the key**

| Side | Key | Action |
|------|-----|--------|
| Java (Client) | offset = 3 | Encrypt |
| PHP (Server) | offset = 3 | Decrypt |

If the keys don't match, decryption fails!

**Encryption and Decryption are opposite**

| Operation | Formula |
|-----------|---------|
| Encrypt | `newChar = char + offset` |
| Decrypt | `newChar = char - offset` |

URL Encoding is required.
```
Problem:  Encrypted text may contain special characters
          "Khoor#iurp" contains # which breaks URL

Solution: URLEncoder.encode() converts special characters
          "Khoor%23iurp" is safe for URL
```


Who encrypts? Who decrypts? It depends on **who has the sensitive data to protect**.

**Scenario 1: Client sends sensitive data**
```
User enters password in App
         │
         ▼
Java encrypts "password123" → "sdvvzrug456"
         │
         ▼
Send to server (encrypted)
         │
         ▼
PHP decrypts → gets "password123" → saves to database
```

Use case: Login, payment, registration

**Scenario 2: Server sends sensitive data**
```
Server has secret API key
         │
         ▼
PHP encrypts "secret-key" → "vhfuhw-nh|"
         │
         ▼
Send to client (encrypted)
         │
         ▼
Java decrypts → gets "secret-key" → uses it
```

Use case: API keys, personal information

**Scenario 3: Both directions**
```
Java encrypts ────────► PHP decrypts
Java decrypts ◄──────── PHP encrypts
```

Use case: Secure chat, banking apps

---

## Summary

| Step | Who | What |
|------|-----|------|
| 1 | Java | Prepare plaintext |
| 2 | Java | Encrypt with key |
| 3 | Java | URL encode |
| 4 | Java | Send via network |
| 5 | PHP | Receive data |
| 6 | PHP | Decrypt with same key |
| 7 | PHP | Process original message |

---

## One Sentence

**Client encrypts data before sending, server decrypts after receiving — both must share the same key!**


