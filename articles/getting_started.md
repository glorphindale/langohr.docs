---
title: "Getting Started with RabbitMQ and Clojure | Clojure RabbitMQ client"
layout: article
---

## About this guide

This guide is a quick tutorial that helps you to get started with the AMQP 0.9.1 specification in general and the Langohr in particular.
It should take about 15 minutes to read and study the provided code examples. This guide covers:

 * Installing RabbitMQ, a mature popular server implementation of the AMQP protocol.
 * Adding Langohr dependency with [Leiningen](http://leiningen.org) or [Maven](http://maven.apache.org/)
 * Running a "Hello, world" messaging example that is a simple demonstration of 1:1 communication.
 * Creating a "Twitter-like" publish/subscribe example with one publisher and four subscribers that demonstrates 1:n communication.
 * Creating a topic routing example with two publishers and eight subscribers showcasing n:m communication when subscribers only receive messages that they are interested in.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/clojurewerkz/langohr.docs).


## What version of Langohr does this guide cover?

This guide covers Langohr 1.0-beta4.


## Supported Clojure Versions

Langohr was built from the ground up for Clojure 1.3+. The latest stable release is recommended.


## Supported RabbitMQ Versions

Langohr works with RabbitMQ versions 2.x. Some features may only be available in more recent releases
(for example, 2.6 or 2.7).


## Langohr Overview

Langohr is a Clojure client for RabbitMQ that embrace [AMQP 0.9.1 Model](http://bitly.com/amqp-model-explained). It reflects
AMQP 0.9.1 protocol structure in the API and uses established terminology and support all of the [RabbitMQ extensions to AMQP 0.9.1](http://www.rabbitmq.com/extensions.html).

Langohr was designed to combine good parts of several other clients: the official RabbitMQ Java client, [Ruby amqp gem](https://github.com/ruby-amqp/amqp)
and [Hot Bunnies](https://github.com/ruby-amqp/hot_bunnies). It does not, however, try to hide the protocol and RabbitMQ capabilities behind layers
of DSLs and new abstractions.

Langohr may seem like a low level client but in return it is predictable, gives you access too all RabbitMQ features and freedom to design message routing
scheme and error handling strategy that makes sense for you.



## What Langohr is Not

Here is what Langohr *does not* try to be:

 * A replacement for the RabbitMQ Java client
 * Sugar-coated API for task queues that hides all the AMQP machinery from the developer
 * A port of Ruby amqp gem to Clojure



## Installing RabbitMQ

The RabbitMQ site has a good [installation guide](http://www.rabbitmq.com/install.html) that addresses many operating systems.
On Mac OS X, the fastest way to install RabbitMQ is with [Homebrew](http://mxcl.github.com/homebrew/):

    brew install rabbitmq

then run it:

    rabbitmq-server

On Debian and Ubuntu, you can either [download the RabbitMQ .deb package](http://www.rabbitmq.com/server.html) and install it with [dpkg](http://www.debian.org/doc/FAQ/ch-pkgtools.en.html) or make use of the [RabbitMQ apt repository](http://www.rabbitmq.com/debian.html#apt).

For RPM-based distributions like RedHat or CentOS, the RabbitMQ team provides an [RPM package](http://www.rabbitmq.com/install.html#rpm).


<div class="alert alert-error">
<strong>Note:</strong> The RabbitMQ package that ships with some of the recent Ubuntu versions (for example, 10.10 and 11.04) is outdated
and *may not work with Langohr* (you will need at least RabbitMQ v2.0 for use with this guide).
</div>




## Adding Langohr Dependency To Your Project

Langohr artifacts are [released to Clojars](https://clojars.org/com.novemberain/langohr).

### With Leiningen

    [com.novemberain/langohr "1.0.0-beta4"]

### With Maven

Add Clojars repository definition to your `pom.xml`:

{% gist 65642c4b53d26539e5f6 %}

And then the dependency:

    <dependency>
      <groupId>com.novemberain</groupId>
      <artifactId>langohr</artifactId>
      <version>1.0.0.-beta4</version>
    </dependency>

### Verifying Your Installation

You can verify your installation in the REPL:

    $ lein repl
    user=> (require 'langohr.core)
    ;= nil
    user=> langohr.core/*default-config*
    ;= {:host "localhost", :port 5672, :username "guest", :vhost "/", :password "guest"}


## "Hello, World" example

Let us begin with the classic "Hello, world" example. First, here is the code:

{% gist 60f82913ed69f0ff24f6 %}

This example demonstrates a very common communication scenario: *application A* wants to publish a message that will end up in a queue that *application B* listens on. In this case, the queue name is "langohr.examples.hello-world". Let us go through the code step by step:

{% gist e0e0de1c4bb60250d835 %}

defines our example app namespace that requires (loads) several Langohr namespaces:

 * `langohr.core`
 * `langohr.channel`
 * `langohr.queue`
 * `langohr.basic`
 * `langohr.consumers`

and will be compiled ahead-of-time (so we can run it).

Clojure applications are compiled to JVM bytecode. The `-main` function is the entry point:

{% gist f1c5163d024db2825691 %}

A few things is going on here:

 * We connect to RabbitMQ using `langohr.core/connect`. We pass no arguments so connection parameters are all defaults.
 * We open a new channel. Channels is a way to have multiple logical "connections" per AMQP connection.
 * We declare an exchange with the name `"langohr.examples.hello-world"`
 * We start consuming messages from it
 * We publish a message, wait for 2 seconds and disconnect

### Connect to RabbitMQ

{% gist fba12df55cbe66ade12c %}

connects to RabbitMQ at `127.0.0.1:5672` with vhost of `"/"`, username of `"guest"` and password of `"guest"` and returns the connection.

`rmq` is an alias for `langohr.core` (see the ns snippet above).

### Open a Channel

{% gist 4d8b16dc9e8a57ffb1fc %}

opens a new *channel*. AMQP is a multi-channeled protocol (connections have one or more channels). This makes it possible for highly concurrent
clients to reuse a single TCP connection for multiple "virtual connections".

`lch` is an alias for `langohr.channel`.

### Declare a Queue

The code below

{% gist cf8c1ee6fa0ebd751465 %}

declares a *queue* on the channel that we have just opened. Consumer applications get messages from queues. We declared this queue with the "auto-delete"
and "non-exclusive" parameters. Basically, this means that the queue will be deleted when our little app exits.

### Start a Consumer

Now that we have a queue, we can start consuming messages from it:

{% gist c6bdf8805bc61fa435ca %}

We use `langohr.queue/subscribe` to start a blocking consumer. Because the consumer will loop waiting for messages forever, we start it
in a separate thread:

{% gist 5b7618377dcf7d9d508f %}

Finally, here's the handling function:

{% gist f84a0165e1998253ed2a %}

It takes a channel the consumer uses, a Clojure map of message metadata and message payload as array of bytes. We turn it into a string
and printit, as well as a few message properties.

### Publish a Message

Next we publish a message to an *exchange*. Exchanges receive messages that are sent by producers. Exchanges route messages to queues according to rules
called *bindings*. In this particular example, there are no explicitly defined bindings. The exchange that we defined is known as the *default exchange* and
it has implicit bindings to all queues.

Routing key is one of the *message properties* (also called attributes). The default exchange will route the message to a queue that has the same name as the message's
routing key. This is how our message ends up in the `"langohr.examples.hello-world"` queue.

{% gist 0755de32689090f58ac8 %}

### Disconnect

Then we use `langohr.core/close`, a polymorphic function, to close both the channel and connection.

{% gist 3777cfc844b6c23ae7a1 %}


This diagram demonstrates the "Hello, world" example data flow:

![Hello, World AMQP example data flow](http://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/001_hello_world_example_routing.png)

For the sake of simplicity, both the message producer (App I) and the consumer (App II) are running in the same JVM process. Now let us move on to a little bit more
sophisticated example.




## Blabbr: One-to-Many Publish/Subscribe (pubsub) Routing Example

The previous example demonstrated how a connection to a broker is made and how to do 1:1 communication using the default exchange. Now let us take a look at another common
scenario: broadcast, or multiple consumers and one producer.

A very well-known broadcast example is Twitter: every time a person tweets, followers receive a notification. Blabbr, our imaginary information network, models this scenario:
every network member has a separate queue and publishes blabs to a separate exchange. Three Blabbr members, Joe, Aaron and Bob, follow the official NBA account on Blabbr
to get updates about what is happening in the world of basketball. Here is the code:

{% gist 5642bbea1fbac1b9e33f %}

In this example, opening a channel is no different to opening a channel in the previous example, however, we do one extra thing: declare an exchange:

{% gist fc8ff2d86d28626c758e %}

The exchange that we declare above is a *fanout exchange*. A fanout exchange delivers messages to all of the queues that are bound to it: exactly what we want in the
case of Blabbr.

This piece of code

{% gist f22f61986d3efeaeeb5c %}

is similar to the subscription code that we used for message delivery previously, but also does a bit more: it sets up a binding between the queue and the exchange
that was declared earlier. We need to do this to make sure that our fanout exchange routes messages to the queues of all subscribed followers.

Another difference in this example is the routing key we use for publishing: it is an empty string:

{% gist 7c315e370ab1f9f85778 %}

Fanout exchanges
simply put a copy of the message in each queue bound to them and ignore the routing key

A diagram for Blabbr looks like this:

![Blabbr Routing Diagram](http://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/002_blabbr_example_routing.png)


## Weathr: Many-to-Many Topic Routing Example

So far, we have seen point-to-point communication and broadcasting. Those two communication styles are possible with many protocols, for instance, HTTP handles these
scenarios just fine. You may ask "what differentiates AMQP?" Well, next we are going to introduce you to *topic exchanges* and routing with patterns,
one of the features that makes AMQP very powerful.

Our third example involves weather condition updates. What makes it different from the previous two examples is that not all of the consumers are interested in
all of the messages. People who live in Portland usually do not care about the weather in Hong Kong (unless they are visiting soon). They are much more interested
in weather conditions around Portland, possibly all of Oregon and sometimes a few neighbouring states.

Our example features multiple consumer applications monitoring updates for different regions. Some are interested in updates for a specific city, others for a specific
state and so on, all the way up to continents. Updates may overlap so that an update for San Diego, CA appears as an update for California, but also should show up
on the North America updates list.

Here is the code:

{% gist e0c6a8c24835ae7c7433 %}

The first line that is different from the Blabbr example is

{% gist 9e7e0a49411e65f7f0c3 %}

We use a topic exchange here. Topic exchanges are used for [multicast](http://en.wikipedia.org/wiki/Multicast) messaging where consumers indicate
which topics they are interested in (think of it as subscribing to a feed for an individual tag in your favourite blog as opposed to the full feed).
Routing with a topic exchange is done by specifying a *routing pattern* on binding, for example:

{% gist 40c08385c0abca8f0f7d %}

Here we bind a queue with the name of "americas.south" to the topic exchange declared earlier using the `langohr.queue/bind` function.  This means
that only messages with a routing key matching `americas.south.#` will be routed to that queue. A routing pattern consists of several words separated by dots,
in a similar way to URI path segments joined by slashes. Here are a few examples:

 * asia.southeast.thailand.bangkok
 * sports.basketball
 * usa.nasdaq.aapl
 * tasks.search.indexing.accounts

Now let us take a look at a few routing keys that match the "americas.south.#" pattern:

 * americas.south
 * americas.south.**brazil**
 * americas.south.**brazil.saopaolo**
 * americas.south.**chile.santiago**

In other words, the `#` part of the pattern matches 0 or more words.

For a pattern like `americas.south.*`, some matching routing keys would be:

 * americas.south.**brazil**
 * americas.south.**chile**
 * americas.south.**peru**

but not

 * americas.south
 * americas.south.chile.santiago

so `*` only matches a single word. The AMQP 0.9.1 specification says that topic segments (words) may contain the letters A-Z and a-z and digits 0-9.

Another novel piece in this example is the use of a **server-named queue**:

{% gist d075d1b0b5d3d6df9cc1 %}

To declare a server-named queue you pass queue name as an empty string and RabbitMQ will generate a cluster-unique name for you. The name
will be returned to the client and you can access it using Java interop on the value `langohr.queue/declare` returns to you. We do
this to output queue name.

Server-named queues are commonly used for collecting replies to other messages (the "request-reply" pattern, often referred to as "RPC") or
when no application needs to know the exact queue name, it just has to be unique.

When you run this example, the output will look a bit like this:

{% gist b3d899a7c0243076e78a %}

As you can see, some messages were routed to multiple queues and some were not routed to any queues ("deadlettered"). Some queues have
names we picked and some have names generated by RabbitMQ.

A (very simplistic) diagram to demonstrate topic exchange in action:

![Weathr Routing Diagram](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/003_weathr_example_routing.png)


The rest of this guide will cover Langohr API design principles. if you are completely new to AMQP and RabbitMQ, feel free
to skip them for now and go straight to the Wrapping Up section.


## Langohr API Structure

AMQP 0.9.1 operations are grouped into **classes** (no relation to classes in OO languages such as Java or Objective-C):

 * `connection.*`
 * `channel.*`
 * `queue.*`
 * `exchange.*`
 * `basic.*`
 * `tx.*`

With a couple of exceptions, the Langohr API follows this structure:

 * `queue.declare` is accessible via the `langohr.queue/declare` function
 * `exchange.declare` is accessible via `langohr.exchange/declare`
 * `basic.publish` is accessible via `langohr.basic/publish`
 * `basic.cancel` is accessible via `langohr.basic/cancel`

and so on. This makes it easy to predict function names, navigate API reference and source code and communicate with
developers who use other AMQP 0.9.1 clients.

The exceptions to this are:

 * `connection.open` is exposed as `langohr.core/connect`
 * `connection.close` is accessible via `langohr.core/close`
 * `channel.close` is accessible via `langohr.core/close` (the function is polymorphic and works on both connections and channels)



## Synchronous vs Asynchronous APIs in Langohr

AMQP 0.9.1 and messaging in general are inherently asynchronous: applications publish messages as events happen and react to events
elsewhere by consuming messages and processing them. Per [AMQP 0.9.1 specification](http://bit.ly/amqp091spec), some AMQP methods (protocol operations, similar to
`GET` and `POST` in HTTP) are asynchronous (do not block) and some are synchronous (block until a response is received).

There are good reasons for this. For some methods, there may be no response (e.g. publishing messages) or it may arrive at an unknown moment
in the future, possibly hours and days later. Some operations (e.g. declaring a queue) typically finish in a few milliseconds and
applications have to wait for them to finish before they can do anything else. In the case of the `queue.declare` method, it is
not possible to start a consumer on a queue that wasn't declared.

RabbitMQ Java client and Langohr take a pragmatic approach: some API functions block, others do not. A few examples of blocking
operations:

 * `langohr.queue/declare`
 * `langohr.exchange/declare`
 * `langohr.queue/bind`
 * `langohr.queue/purge`
 * `langohr.queue/delete`
 * `langohr.basic/get` ("pull API", synchronous by design)

The following functions do not block the calling thread:

 * `langohr.basic/publish`
 * `langohr.basic/ack`
 * `langohr.basic/reject`
 * `langohr.basic/nack`

these lists are not complete but should give you an idea about Langohr's philosophy: operations that are performance-sensitive or inherintly
asynchronous won't block, operations that are synchronous by nature or required to finish for other operations to proceed as blocking.

This offers developers a good balance of convenience of blocking function calls and good throughput for publishing.


## Wrapping Up

This is the end of the tutorial. Congratulations! You have learned quite a bit about both RabbitMQ and Langohr. This is only the tip of the iceberg.
RabbitMQ has many more features built into the protocol:

 * Reliable delivery of messages
 * Message confirmations (a way to tell broker that a message was or was not processed successfully)
 * Message redelivery when consumer applications fail or crash
 * Load balancing of messages between multiple consumers
 * Message metadata attributes

and so on. Other guides explain these features in depth, as well as use cases for them. To stay up to date with Langohr development, [follow @clojurewerkz on Twitter](http://twitter.com/clojurewerkz)
and [join our mailing list](http://groups.google.com/group/clojure-rabbitmq).


## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [AMQP Concepts](http://www.rabbitmq.com/tutorials/amqp-concepts.html)
 * [Conneciting To The Broker](/articles/connecting.html)
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
