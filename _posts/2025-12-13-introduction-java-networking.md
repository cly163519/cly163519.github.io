---
title: Introduction to Java networking
date: 2025-12-13 10:00:00 +1300
categories: [learning, programming, Java]
tags: [java, networking, cloud]
---


## Download webpages to local computer


This Java example is a very typical example of using standard Java libraries (java.net.URL and java.net.URLConnection) to send an HTTP GET request and download web page content.


```java
package Network;

import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.PrintStream;
import java.net.HttpURLConnection;
import java.net.URI;
import java.net.URL;
import java.net.URLConnection;
import java.util.Scanner;

public class net {
    @SuppressWarnings({"UseSpecificCatch", "CallToPrintStackTrace", "ConvertToTryWithResources"})
    public static void main(String[] args) {
        try{
            //1. Create a URL object
            URL url = new URI("https://en.wikipedia.org/wiki/Victoria_University_of_Wellington").toURL(); // creates URL object
            
            //2. Open a connection
            final URLConnection connection = url.openConnection(); // create communication link between program and url
            
            //3. Set request headers (optional)
            String agentName = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36";
            connection.setRequestProperty("User-Agent", agentName); // Sets "User-Agent" header in HTTP
            
            //4. Cast to HttpURLConnection to access HTTP-specific features
            HttpURLConnection httpc = (HttpURLConnection)connection; // Cast to HTTP to access get response methods
           
            //5. Check the response status
            System.out.println(httpc.getResponseCode() + ": " + httpc.getResponseMessage()); // 404 is Error, 200 is OK
            
            //6. Get the input stream to read data
            final InputStream stream = connection.getInputStream(); // get ready to start receiving data
            
            //7.Create a reader and output file
            Scanner reader; // or BufferedReader reader = new BufferedReader(new InputStreamReader(stream));
            reader = new Scanner(new InputStreamReader(stream));
            PrintStream out = new PrintStream("C:\\dev\\Network\\wiki.html"); 

            //8. Read and write line by line
            while(reader.hasNextLine()) {
                    String line = reader.nextLine();
                    //System.out.println(line);
                    out.println(line);
                }
            
            //9. Close resources    
            out.close();
            reader.close();
        } catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

## Java network programming steps: 

URL → Connection → Send → Read

## Why is this a GET request? Why not a POST request?

  1. Default Behavior (GET is the default): In Java's URLConnection and HttpURLConnection: If the HTTP request method is not explicitly set, the connection object uses the GET method by default. You can explicitly set it to POST using `httpc.setRequestMethod("POST");`, but since this line is not present in your code, it follows the default GET behavior.
     
  2. Differences in HTTP Request Methods:
     
      |---------|-------------|--------------|
      | Purpose | Request (retrieve) data from server | Submit (send) data to server |
      | Data Location | Data appended to URL (e.g., `?...&key=value`) | Data sent in Request Body |
      | Security | Less secure (data exposed in URL) | More secure (data not visible in URL) |
      | Idempotency | Yes (repeated requests have no side effects) | No (repeated submissions may create multiple resources) |

  3. The code demonstrates this: No request method is set, so it defaults to GET. Only `connection.getInputStream()` is called to retrieve the server's response data. No `connection.getOutputStream()` is called to write or send any request body data to the server. In summary: The purpose is simply to retrieve the HTML page content from Wikipedia, which perfectly conforms to the semantics of a GET request.
