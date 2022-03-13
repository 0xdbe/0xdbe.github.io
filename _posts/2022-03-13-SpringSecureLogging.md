---
layout: post
title: "Spring Boot: Prevent Log Injection Attacks With Logback"
date: 20220313
published: true
categories: [Spring]
tags: Security, AppSec, Spring
canonical_url: https://0xdbe.github.io/SpringSecureLogging/
---

Log Injection is an attack that has been known to everyone for years.
Despite the fact that any application can record logs from user input, for too long many of us had forgotten about the dangers.

But the recently discovered vulnerabilities concerning log4j2 have reminded us of the importance of preventing log injection attacks.
This article describes one concrete way - albeit not the only way - to prevent log injection attacks in a Spring Boot application using Logback.


> This article is written down as an ADR (Architectural Decision Record) because, from my point of view, this is an important security decision.
> This includes the context of how the decision was made and the consequences of adopting the decision.
> So, if you find it useful, you can share this ADR with your team members, and perhaps it will prove to be an effective strategy.

## Context

By default, a Spring Boot application uses Logback as a logging framework, and it does this with SLF4J as the interface between the application and the logging framework.

In this way, an application can easily log anything in two steps:

- Create a logger with ``LoggerFactory.getLogger()``
- Log event with ``logger.debug()`` (where ``debug`` is the log level)

Here is an example that can be found in a simple greeting controller:

```java
@RestController
public class HelloController {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @RequestMapping("/")
    public String index(@RequestParam(value = "name", defaultValue = "World") String name) {
        logger.debug("Hello {}", name);
        return "Hello " + name;
    }

}
```
As you can see, this controller takes the username as the input and then logs it.

Consequently, due to this controller logging data from user input, this application is vulnerable to log injection.


## Attack

Primarily, log injection allows an attacker to forge log entries; this is what we call "log forging.” The easiest way is to forge a new log entry using CRLF injection. 

CRLF injection involves inserting two control characters called ``Carriage Return`` (``%0d`` or ``\r``) and ``Line Feed`` (``%0a`` or ``\n``).

Here is an example of an CRLF injection on our greeting controller:

```shell
curl http://localhost:8080/\?name\=Marty%0d%0aYou%20have%20been%20pwned
```

This request results in the logging of two separate entries in the logger.
This could be annoying when later doing log analysis because there would be additional incorrect log entries.

However, despite being primarily used to forge entries, sometimes, log injection attacks can also be used to inject malicious code that could then be executed either by the logging framework (like log4j2) or later in the logging pipeline.

So, it is vital to prevent log injection attacks with a suitable defensive solution.


## Considered options

There are many ways to prevent log injection attacks, such as:

- **Input validation** involves checking that user inputs are in the expected format (mail, date, price, value from a list, ...). This is a good thing! However, for some types of data (like comments), input validation is too difficult and would not be very effective. Of course, you can keep going with input validation, but this is not enough to secure an application.

- **Input filtering** involves stripping out unsafe characters. This is well-intentioned, but sometimes mangles perfectly good input. For example, if the filtering function strips the ``''``, someone like ``O'Conor`` becomes ``OConor``. Something that I doubt they would appreciate.

These solutions are not acceptable if you want to build secure software.


## Decision

The best way to prevent log injection is *output encoding*.
It is really easy to do with Logback!

Here is the configuration file ``logback.xml`` to encode logs using ``json`` layout:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
          <layout class="ch.qos.logback.contrib.json.classic.JsonLayout">
            <jsonFormatter class="ch.qos.logback.contrib.jackson.JacksonJsonFormatter"/>
            <appendLineSeparator>true</appendLineSeparator>
          </layout>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="console"/>
    </root>

</configuration>
```

And here is the dependencies required by Logback to support JSON:

```groovy
dependencies {
    implementation 'ch.qos.logback.contrib:logback-json-classic:0.1.5'
    implementation 'ch.qos.logback.contrib:logback-jackson:0.1.5'
}
```

With this configuration, each log entry is wrapped up in a JSON object: 

```json
{
  "timestamp":"1643015604000",
  "level":"DEBUG",
  "thread":"http-nio-8080-exec-1",
  "logger":"net.example.logging.HelloController",
  "message":"Hello Marty\r\nYou have been pwned",
  "context":"default"
}
```

As you can see, special characters, like CRLF, are escaped.
This will be the same for quotation marks which will be encoded by ``\"``.


## Consequences

From this point onwards, this application is no longer vulnerable to log injection anymore.
However, there are consequences to consider.

### Unreadable logs

Since each log entry is wrapped up in a JSON object, it becomes difficult to use logs with simple tools like ``cat``, ``tail`` or ``grep``.
So, a log management system, such as ELK (Elasticsearch, Logstash and Kibana), is mandatory.

However, it is still possible to have several configurations using Spring Profile:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

  <springProfile name="!local">
    <!-- Configuration with JSON Layout -->
  </springProfile>
  
  <springProfile name="local">
    <!-- Configuration with Pattern Layout -->.
  </springProfile>

</configuration>
```

In this case, Spring Boot Application uses the default pattern layout when it starts locally.
Thus making logs easier to read for developers.

### Sensitive Data

Keep in mind that this ``JSON Layout`` doesn't mask sensitive data in log entries.
A custom layout will still be required to mask sensitive line PII. 

