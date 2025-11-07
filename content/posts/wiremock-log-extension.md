---
title: "Structured JSON Logging for WireMock: A Custom Extension for Easy Log Aggregation"
date: 2023-11-06T15:00:00+01:00
draft: false
tags: ["Java", "WireMock", "Logging", "JSON", "Observability", "Logstash", "Testing", "DevOps"]
description: "Learn how to build and use a custom WireMock extension to generate clean, single-line JSON logs, perfect for ingestion by systems like Logstash, OpenSearch, and the ELK stack."
---

If you've ever used [WireMock](https://wiremock.org/) for integration testing, you know it's an incredibly powerful tool. However, you've probably also noticed that its default logging, while informative for humans, can be a real headache for machines. It's verbose, multi-line, and unstructured, making it a challenge to parse and analyze in a modern log aggregation system.

In this post, we'll walk through a custom WireMock extension that transforms this noisy output into clean, single-line, structured JSON logs, perfect for systems like Logstash, Fluentd, or an OpenSearch/ELK stack.

### The Problem: Default WireMock Logging

When WireMock serves a request, it logs a detailed, human-readable summary to the console. It looks something like this:

```
2023-11-06 14:30:00.123 ===============================================================================
| Request received:                                                              |
|   GET /api/users/123                                                           |
|   Accept: [application/json]                                                   |
|                                                                                |
| Matched response definition:                                                   |
|   HTTP/1.1 200                                                                 |
|   Content-Type: [application/json]                                             |
|                                                                                |
| Response:                                                                      |
|   HTTP/1.1 200                                                                 |
|   Content-Type: application/json                                               |
|   Body: {"id": 123, "name": "John Doe"}                                        |
================================================================================
```

While great for debugging during development, this format is a nightmare for automated parsing:
- **Multi-line**: A single event is spread across many lines.
- **Unstructured**: It's free-form text, not a key-value format.
- **Inefficient**: Requires complex grok patterns or custom scripts in your log aggregator to even begin to make sense of it.

### The Solution: A Custom `ServeEventListener`

WireMock provides a powerful extension mechanism, and the `ServeEventListener` interface is the perfect hook for our needs. It allows us to execute custom code just before a response is sent, giving us access to the complete request and response objects.

We'll create an extension that:
1.  Hooks into the `beforeResponseSent` event.
2.  Gathers all relevant data from the request and response.
3.  Constructs a JSON object with this data.
4.  Prints the JSON object to the console as a single line.

### Key Features of the Extension

This isn't just a simple JSON converter. It includes intelligent sanitization to keep your logs clean and efficient.

#### 1. Structured, Single-Line JSON Output
Every request-response pair is logged as one line, making it trivial for log shippers to pick up.

#### 2. Intelligent Sanitization of Binary Data
Logging raw binary content is a recipe for disaster. It corrupts JSON, bloats logs, and provides no searchable value. Our extension automatically handles this:

-   **Binary Request Bodies**: For requests like file uploads (`multipart/form-data`, `application/pdf`, etc.), the entire body is replaced with a simple placeholder: `"<binary content not logged>"`.
-   **Large Base64 Fields**: For API responses that include file downloads as Base64 strings, the extension recursively scans the JSON response. If it finds a long string that is valid Base64, it replaces it with `"<base64_data_omitted>"`. This prevents multi-megabyte log entries while still indicating that binary data was present.

#### Example of the Final Log Output

Here’s what a log entry for a file download looks like with our extension. Notice how the request and response bodies are cleanly handled.

```json
{
  "@timestamp": "2025-11-06T18:19:36.046Z",
  "service": "wiremock",
  "wasMatched": true,
  "requestId": "af7dc0f1-81dd-4ab1-9635-3bd44b4414bf",
  "request": {
    "method": "POST",
    "url": "http://localhost:9119/auxi/downloadSign",
    "clientIp": "127.0.0.1",
    "headers": { "Content-Type": "application/json; charset=UTF-8" },
    "body": "{ \"idPF\": \"vbak1ds48m\"}"
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json;charset=UTF-8" },
    "body": "{\"documentoFirmadoBase64\":\"<base64_data_omitted>\"}"
  }
}
```

### How to Use It

#### 1. Get the Code
Clone the repository from GitHub.
```sh
# Replace with your actual GitHub username
git clone https://github.com/bernalvarela/wiremock-log-extension.git
cd wiremock-log-extension
```

#### 2. Build the Extension
The project uses Maven. Build the self-contained "uber-JAR" with one command:
```sh
mvn clean package
```
This will compile the code, run the tests, and create `wiremock-log-extension-1.0.0-SNAPSHOT.jar` in the `target/` directory.

#### 3. Run WireMock with the Extension
1.  Copy the generated JAR into your WireMock `extensions` directory.
2.  Start WireMock using the `--extensions` flag to point to the main class of our listener.

```sh
java -jar wiremock-standalone.jar --extensions dev.bernalvarela.wiremock.extensions.StructuredJsonLoggingListener
```

That's it! WireMock will now output structured JSON logs to the console for every request it serves.

### Bonus: Integrating with Docker and Logstash

This extension truly shines when combined with a log aggregation pipeline. Here’s a sample configuration for sending logs from a Dockerized WireMock to Logstash.

**`docker-compose.yml` for WireMock:**
We use the `syslog` logging driver to forward all console output to Logstash.

```yaml
version: "3.9"
services:
  wiremock:
    image: wiremock/wiremock:3.13.1
    container_name: wiremock
    # ... volumes and other config ...
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://localhost:12201"
        tag: "wiremock-log"
```

**`logstash.conf` Pipeline:**
This Logstash pipeline listens for the syslog stream, reassembles any large logs that were split into multiple lines, and then parses the resulting JSON.

```ruby
input {
    tcp {
        port => 12201
        codec => multiline {
            # A new log event starts with a syslog header and contains "{@timestamp}"
            pattern => "^\\<[0-9]{1,3}\\>.*\\{\\"@timestamp\\""
            negate => true
            what => "previous"
        }
        tags => ["wiremock_multiline_stream"]
    }
}

filter {
    if "wiremock_multiline_stream" in [tags] {
        # 1. Discard WireMock's startup messages
        if [message] !~ /\\{\\"@timestamp\\"/ {
            drop {}
        }

        # 2. Extract the pure JSON from the syslog message
        grok {
            match => { "message" => "(?<json_content>\\{[\\s\\S]*)" }
        }

        # 3. Clean up newlines and parse the JSON
        if [json_content] {
            mutate {
                gsub => [ "json_content", "\\n", "" ]
            }
            json {
                source => "json_content"
            }
        }
        # ... other cleanup ...
    }
}

output {
    opensearch {
        # ... your opensearch config ...
    }
}
```

### Conclusion

With a simple Java extension, we've transformed WireMock's logging from a developer-focused convenience into a powerful, production-ready data source for observability. This approach makes it incredibly easy to monitor, alert on, and analyze your test traffic, bridging the gap between integration testing and production monitoring.

You can find the complete source code, including the Maven project and unit tests, on GitHub.

**[Check out the full project on GitHub!](https://github.com/[your-github-username]/wiremock-log-extension)**
