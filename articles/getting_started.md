---
title: "Getting Started"
layout: article
---

## About this guide

This guide is a quick tutorial that helps you to get started with the AMQP 0.9.1 specification in general and the Langohr in particular.
It should take about 20 minutes to read and study the provided code examples. This guide covers:

 * Installing RabbitMQ, a mature popular server implementation of the AMQP protocol.
 * Adding Langohr dependency with [Leiningen](http://leiningen.org) or [Maven](http://maven.apache.org/)
 * Running a "Hello, world" messaging example that is a simple demonstration of 1:1 communication.
 * Creating a "Twitter-like" publish/subscribe example with one publisher and four subscribers that demonstrates 1:n communication.
 * Creating a topic routing example with two publishers and eight subscribers showcasing n:m communication when subscribers only receive messages that they are interested in.


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

![Hello, World AMQP example data flow](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/001_hello_world_example_routing.png)

For the sake of simplicity, both the message producer (App I) and the consumer (App II) are running in the same JVM process. Now let us move on to a little bit more
sophisticated example.




## Blabbr: One-to-Many Publish/Subscribe (pubsub) Routing Example

TBD


## Weathr: Many-to-Many Topic Routing Example

TBD


## Synchronous vs Asynchronous APIs in Langohr

TBD



## Wrapping Up

TBD


## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

TBD



## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on Twitter](http://twitter.com/clojurewerkz) or the [Clojure RabbitMQ mailing list](https://groups.google.com/forum/#!forum/clojure-rabbitmq)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
