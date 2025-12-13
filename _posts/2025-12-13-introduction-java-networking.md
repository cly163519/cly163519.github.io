---
title: Introduction to Java networking
date: 2025-12-13 10:00:00 +1300
categories: [learning, programming, Java]
tags: [java, networking, php/json, get/post]
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


##  Get timestamps via GET request

Java (Client) ──── GET request ────► PHP (Server)
     │                                    │
           ◄──── Returns JSON ────

PHP（Server）- timestamp.php

```php
<?php
// http://localhost/cloud_experiments/timestamp.php
$date = new DateTime('now', new DateTimeZone('Pacific/Auckland'));
$myObj = new stdClass();
$myObj->timestamp = date_format($date, 'm/d/Y h:i:s a');
$myObj->timezone = "Pacific/Auckland";
$myJSON = json_encode($myObj);
header('Content-Type: application/json');
echo $myJSON;
?>
```

```java
// 1. Create URL object, pointing to the PHP script
        URL url = new URI("http://localhost/cloud_experiments/timestamp.php").toURL(); 
        
        //2. Open connection
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();

        //3. Create reader to receive data
        BufferedReader reader = new BufferedReader(new InputStreamReader(conn.getInputStream()));
        
        //4. Read response into StringBuilder
        StringBuilder json_response = new StringBuilder();
        String inputLine;
        while ((inputLine = reader.readLine()) != null) {
            json_response.append(inputLine);
        }
        reader.close();

        // 5. Parse JSON string into object
        JSONParser parser = new JSONParser();
        JSONObject jsonObject = (JSONObject) parser.parse(json_response.toString());

        // 6. Extract timestamp value
        System.out.println("Timestamp from PHP: " + jsonObject.get("timestamp"));
        System.out.println("Timezone from PHP: " + jsonObject.get("timezone"));
```
Why GET? (GET = Retrieving data, no data sent)

| Features | In this example | 
|----------|-----------------|
| Simply retrieve data | Retrieve timestamp |
| No data sent to the server | No data sent via URL |
| No data sent to the serverNo parameters | Direct access via PHP|

In short, PHP generates data, JSON packages data, GET requests transmit data, and Java parses data.

##  Get Menu via GET request

```java

//1. Prepare JSON data
        JSONObject obj = new JSONObject();
        obj.put("food", "milktea");
        obj.put("name", "Jack");
        obj.put("position", "Home");
        String msg = obj.toJSONString();

        //2. Encode JSON data for URL transmission
        String encoded = URLEncoder.encode(msg, "UTF-8");

        //3. Create URL with encoded JSON data as parameter
        String queryUrl = "http://localhost/cloud_experiments/menu.php?jsonData="+encoded;
        URL url = new URI(queryUrl).toURL();

        //4. Open connection and read response
        final URLConnection connection = url.openConnection(); 

        //5. Create reader to receive response
        BufferedReader reader = new BufferedReader(
                new InputStreamReader(connection.getInputStream())
            );  
            
        //6. Read and print response
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }
            
            reader.close();

```

```php

<?php
if(isset($_GET['jsonData']))
{
    echo "(PHP): " .$_GET['jsonData'];
}
 else {
    echo "(PHP): No data received.";
}
?>

```

## Update menu via POST request

```java
            //1. Prepare JSON data
            JSONObject obj = new JSONObject();
            obj.put("food", "milktea");
            obj.put("name", "Jack");
            obj.put("position", "Home");
            String msg = obj.toJSONString();

            // 2. Connection (Note: The URL no longer includes parameters as before)
            String encoded = URLEncoder.encode(msg, "UTF-8"); 
            String queryUrl = "http://3.236.22.28/jsonphp_GET_s3_putObject.php?jsonData=" + encoded;    
            URL url = new URI(queryUrl).toURL();
            HttpURLConnection connection = (HttpURLConnection)url.openConnection();

            //3. Set as a POST request
            connection.setRequestMethod("POST");
            connection.setRequestProperty("Accept", "application/json");
            connection.setDoOutput(true);
            
            // 4. Send data (write to the request body)
            try (OutputStream os = connection.getOutputStream()) {
                byte[] input = msg.getBytes("utf-8");
                os.write(input, 0, input.length);
            }

            // 5.Read the response
            BufferedReader reader = new BufferedReader(
                    new InputStreamReader(connection.getInputStream(), "utf-8")
                );
                
                String line;
                while ((line = reader.readLine()) != null) {
                    System.out.println(line);
                }
                reader.close();


```

```php

        <?php
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            // Read data from the request body
            $postData = file_get_contents("php://input");
            $json = json_decode($postData, true);
            
            echo "Received POST data:\n";
            echo "Name: " . $json['food'] . "\n";
            echo "Age: " . $json['name'] . "\n";
            echo "City: " . $json['position'] . "\n";
        } else {
            echo "Not a POST request";
        }
        ?>

```

## Summary: Compare 3 activities

1. Timestamp (GET - Request Data)

```
Java ────── "Give me time" ────► PHP
         (empty request)          │
                                  │ Generate time
                                  ▼
Java ◄────── timestamp ────────  PHP
         (main data)
```

2. Menu (GET - Send Data)

```
Java ────── menu data ─────────► PHP
         (in URL ?param=)         │
                                  │ Process data
                                  ▼
Java ◄────── "Received" ────────  PHP
         (confirmation)
```

3. Menu Update (POST - Send Data)

```
Java ────── menu data ─────────► PHP
         (in request body)        │
                                  │ Save data
                                  ▼
Java ◄────── "Received" ────────  PHP
         (confirmation)
```

**Main data flow: Java → PHP**

---

## Comparison Table

| | Activity 2 | Activity 3 | Activity 4 |
|--|-----------|-----------|-----------|
| **Method** | GET | GET | POST |
| **Purpose** | Get data | Send data | Send data |
| **Data location** | - | URL `?param=` | Request body |
| **Main data flow** | PHP → Java | Java → PHP | Java → PHP |

---

## GET vs POST

| | GET | POST |
|--|-----|------|
| Data location | In URL | In request body |
| Visible? | Yes | No |
| Security | Less secure | More secure |
| Use case | Query data | Submit forms |

---

## How to Identify Data Flow

| PHP Code | Data Flow |
|----------|-----------|
| Only `echo`, no `$_GET` | PHP → Java |
| Has `$_GET['xxx']` | Java → PHP (URL) |
| Has `file_get_contents("php://input")` | Java → PHP (body) |


