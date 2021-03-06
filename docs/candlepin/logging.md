---
categories: developers
title: Logging in Candlepin
---
{% include toc.md %}

# Logging in Candlepin
We use [SLF4J](http://www.slf4j.org/) in Candlepin.  SLF4J is a logging facade
that can bridge to multiple logging implementations (including Log4j,
java.util.logging, and others).  We are using [Logback](http://logback.qos.ch/)
as our implementation.

## SLF4J
When possible, only have your class rely on classes from SLF4J.  Using facade
gives us flexibility.  Under most circumstances, this is all you will need to
do.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Foo {
    private static Logger log = LoggerFactory.getLogger(Foo.class);

    public static void main(String[] args) {
        log.info("Hello world")
    }
}
```

Note that the log object is declared as static.  There is [some
debate](http://www.slf4j.org/faq.html#declared_static) about this practice, but
on Candlepin we stick to statics because our code isn't meant to be deployed on
a shared classpath.

Now let's say you want to print out an object.  That's where SLF4J shines: parameterized messages.  Here's the old way:

```java
//DO NOT DO THIS ANYMORE
Object x = new Something();
log.debug("Hello " + x);
```
{:.alert-bad}

The old way incurs the cost of the toString() operation as soon as the debug()
method is called *even if logging is not set at debug level* (whereupon the
message wouldn't be logged at all).  You can get around this with a
`if(log.isDebugEnabled())` but that gets really ugly fast.  With parameterized
messages the cost of the toString() operation is deferred until we know that we
are actually going to log the message.  Here is the new way:

```java
Object x = new Something();
log.debug("Hello {}", x);
```

What if you don't want to log any message, but just the object itself?  Easy: `log.debug("{}", x)`

If you are going to perform some expensive method call in the logging statement
and you want to avoid the cost when possible, then you *are* going to have to
use an if statement because the nested method call is going to get resolved
before debug() is called.

```java
Object x = new Something();
if (log.isDebugEnabled()) {
    log.debug("Hello {}", x.doSomethingExpensive());
}
```

That's about all you need to know about SLF4J but I recommend reading the [FAQ](http://www.slf4j.org/faq.html)

## Logback
First, read [the manual](http://logback.qos.ch/manual/introduction.html).
Specifically the chapters on architecture (which is very similar to Log4J's),
configuration, filters, and Mapped Diagnostic Contexts.  The rest you can just
refer to when you need it.

If you want to do something fancy with logging, then you are going to need to
access the underlying implementation.  SLF4J doesn't really allow you to
configure loggers.  To interact with Logback, you need to get the logging
context first.

```java
LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
```

With the context, you can access Loggers, attach new Appenders, etc.

### Logback Gotchas and Tips
* If you create something new, you're probably going to need to ```start()```
  it.  If the Logback class you're using implements LifeCycle (and the Filters,
  Appenders, Evaluators, TurboFilters, and Encoders all do), you need to set it
  up and then call `start()` otherwise it's not going to do anything.
  Loggers are the exception to this rule.
* If your logging configuration is wrong or messed up, Logback will print error
  messages to STDERR.  With Tomcat, those messages end up in catalina.out and
  **not** in candlepin.log!
* If you really want to see what's going on during set up, you can add the
  `debug="true"` attribute to the `configuration` element in
  logback.xml.  See <http://logback.qos.ch/manual/configuration.html#automaticStatusPrinting>
* Logback has really powerful filtering using Janino where you can place Java
  expressions in the filter definition and they will be evaluated at logging
  time to determine whether or not to log a statement.  *We do not use these*.
  Janino would be another dependency that we'd have to manage.  If you want to
  use Janino evaluators during testing (you'd have to add the Janino library to
  Candlepin's classpath) that's fine but don't check them in.
* TurboFilters trump log level!  If a TurboFilter accepts a log event, then it
  will get logged.  If the TurboFilter denies it, then it won't.  So be careful
  with your TurboFilters.  In some cases it may be necessary to add an
  additional Filter to an Appender to keep out events accepted by TurboFilters.
* If you are an extreme power user, you can use Logback's JMX functionality to
  alter logging configuration at run-time.  See <http://logback.qos.ch/manual/jmxConfig.html>

## Our Customizations
We have a few customizations to logging.

* Logger levels are set primarily in /etc/candlepin/candlepin.conf.

  ```properties
  log4j.logger.org.candlepin.servlet.filter=DEBUG
  ```

  The "log4j.logger" part is just a prefix; the rest is the Logger name and
  then the level you want.  The LoggingConfig class handles setting the
  Loggers.  This means that you should hardly ever actually need to touch
  logback.xml which is where the basic configuration is.
* The LoggingFilter is a servlet filter that "tees" the ServletRequest and
  ServletResponse so that we can log them.  It also adds a request UUID to the
  MDC.
* Superadmins can enable different logging levels for different orgs.  To set to debug:

  ```console
  $ curl -k -u admin:admin -X PUT "https://localhost:8443/candlepin/owners/{owner_key}/log"
  ```

  You can specify other levels with a level query param.  For example

  ```console
  $ curl -k -u admin:admin -X PUT "https://localhost:8443/candlepin/owners/{owner_key}/log?level=TRACE"
  ```

  To turn org level logging off

  ```console
  $ curl -k -u admin:admin -X DELETE "https://localhost:8443/candlepin/owners/{owner_key}/log"
  ```

* The AuthInterceptor then authenticates you and sees if you have authorization
  to make the request you made.  If you do, it will place your org key and org
  logging level in the MDC.  Keep in mind that if you fail authentication or
  authorization, we probably won't know your org or org log level because
  AuthInterceptor aborts as soon as there is a failure.
* **To mitigate this problem, if you are writing a method with multiple
  `@Verify` annotated parameters put the parameters in order of likelihood that
  the user will pass them.  That way we will have the org if a subsequent
  parameter fails its `@Verify` check.**  Most methods don't have multiple
  `@Verify` annotations so this shouldn't be a common occurrence.
* The org level logging is handled by the LoggerAndMDCFilter class which is a TurboFilter.
