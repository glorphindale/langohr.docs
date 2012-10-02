---
title: "Connecting to RabbitMQ from Clojure with Langohr"
layout: article
---

## About this guide

This guide covers connection to an AMQP broker with Langohr, connection error handling, authentication failure handling and related issues.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/clojurewerkz/langohr.docs).


## What version of Langohr does this guide cover?

This guide covers Langohr 1.0-beta9.



## Two ways to specify connection parameters

With Langohr, connection parameters (host, port, username, vhost and so on) can be passed in two forms:

 * As a map of attributes
 * As a connection URI string (à la JDBC)


### Using a Map of Parameters

Map options that Langohr will recognize are

 * `:host`
 * `:port`
 * `:username`
 * `:password`
 * `:vhost`
 * `:requested-heartbeat`
 * `:connection-timeout`

To connect to RabbitMQ with a map of parameters, use the `langohr.core/connect` function:

{% gist e7405705d5b1b7c50ecf %}

The function returns a connection instance that is used to open channels. More about channels later in this guide.


#### Default parameters

Default connection parameters are

{% gist cf84d6455c018afc7448 %}


### Using Connection Strings

It is also possible to specify connection parameters as a URI string using the `:uri` option:

{% gist 807bf5c216f33df24614 %}


Unfortunately, there is no URI standard for AMQP URIs, so while several schemes used in the wild share the same basic idea, they differ in some details.
The implementation used by Langohr aims to encourage URIs that work as widely as possible.

Here are some examples of valid AMQP URIs:

 * amqp://dev.rabbitmq.com
 * amqp://dev.rabbitmq.com:5672
 * amqp://guest:guest@dev.rabbitmq.com:5672
 * amqp://hedgehog:t0ps3kr3t@hub.megacorp.internal/production
 * amqps://hub.megacorp.internal/%2Fvault

The URI scheme should be "amqp", or "amqps" if SSL is required.

The host, port, username and password are represented in the authority component of the URI in the same way as in http URIs.

The vhost is obtained from the first segment of the path, with the leading slash removed.  The path should contain only a single segment (i.e, the only slash in it should be the leading one). If the vhost is to include slashes or other reserved URI characters, these should be percent-escaped.

Here are some examples that demonstrate how `langohr.core/settings-from` parses out the vhost from connection URIs:

<pre>
"amqp://dev.rabbitmq.com"            ;; => vhost is nil, so default ("/") will be used
"amqp://dev.rabbitmq.com/"           ;; => vhost is an empty string
"amqp://dev.rabbitmq.com/%2Fvault"   ;; => vhost is "/vault"
"amqp://dev.rabbitmq.com/production" ;; => vhost is "production"
"amqp://dev.rabbitmq.com/a.b.c"      ;; => vhost is "a.b.c"
"amqp://dev.rabbitmq.com/foo/bar"    ;; => argument exception
</pre>


### Connection Failures

If a connection does not succeed, Langohr will raise one of the following exceptions:

 * [java.net.ConnectException](http://docs.oracle.com/javase/7/docs/api/java/net/ConnectException.html) will be raised if there is no service listening on the remote address/port, for example, due to a port misconfiguration
 * [java.net.UnknownHostException](http://docs.oracle.com/javase/7/docs/api/java/net/UnknownHostException.html) will be raised if the host cannot be resolved
 * com.rabbitmq.client.PossibleAuthenticationFailureException will be raised if the supplied credentials are invalid (or connection was closed before authentication could succeed)


## PaaS Environments

### The RABBITMQ_URL Environment Variable

If no arguments are passed to `langohr.core/connect` but the `RABBITMQ_URL` environment variable is set, Langohr will use it as connection
URI.

## Opening a Channel

Some applications need multiple connections to an AMQP broker. However, it is undesirable to keep many TCP connections open at the same time because
doing so consumes system resources and makes it more difficult to configure firewalls. AMQP 0-9-1 connections are multiplexed with channels that can
be thought of as "lightweight connections that share a single TCP connection".

To open a channel, use the `langohr.channel/open` function that takes a connection:

{% gist a47b53cc03cf4bad8573 %}

Channels are typically long lived: you open one or more of them and use them for a period of time, as opposed to opening
a new channel for each published message, for example.


## Closing Channels

To close a channel, pass it the `langohr.core/close` function.


## Disconnecting

To close a connection, pass it the `langohr.core/close` function.


## Troubleshooting

If you have read this guide and still have issues with connecting, check our [Troubleshooting guide](/articles/troubleshooting.html)
and feel free to ask [on the mailing list](https://groups.google.com/forum/#!forum/clojure-rabbitmq).


## Wrapping Up

There are two ways to specify connection parameters with Langohr: with a map of parameters or via URI string.
Connection issues are indicated by various exceptions. If the `RABBITMQ_URL` env variable is set, Langohr
will use its value as RabbitMQ connection URI.


## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [Queues and Consumers](/articles/queues.html)
 * [Exchanges and Publishing](/articles/exchanges.html)
 * [Bindings](/articles/bindings.html)
 * [RabbitMQ Extensions to AMQP 0.9.1](/articles/rabbitmq_extensions.html)
 * [Durability and Related Matters](/articles/durability.html)
 * [Error Handling and Recovery](/articles/error_handling.html)
 * [Troubleshooting](/articles/troubleshooting.html)
 * [Using TLS (SSL) Connections](/articles/tls.html)



## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on Twitter](http://twitter.com/clojurewerkz) or the [Clojure RabbitMQ mailing list](https://groups.google.com/forum/#!forum/clojure-rabbitmq)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
