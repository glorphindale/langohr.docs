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

This guide covers Langohr 1.5.x.


## Supported Clojure Versions

Langohr requires Clojure 1.4+. The latest stable release is recommended.


## Supported RabbitMQ Versions

Langohr works with RabbitMQ versions 2.x and 3.x. Some features may only be available in more recent releases
(for example, 3.0).


## Langohr Overview

Langohr is a Clojure client for RabbitMQ that embrace [AMQP 0.9.1
Model](http://bitly.com/amqp-model-explained). It reflects AMQP 0.9.1
protocol structure in the API and uses established terminology and
support all of the [RabbitMQ extensions to AMQP
0.9.1](http://www.rabbitmq.com/extensions.html).

Langohr was designed to combine good parts of several other clients:
the official RabbitMQ Java client, [Bunny](http://rubybunny.info) and
[March Hare](http://rubymarchhare.info). It does not, however, try to
hide the protocol and RabbitMQ capabilities behind layers of DSLs and
new abstractions.

Langohr may seem like a low level client but in return it is
predictable, gives you access too all RabbitMQ features and freedom to
design message routing scheme and error handling strategy that makes
sense for you.



## What Langohr is Not

Here is what Langohr *does not* try to be:

 * A replacement for the RabbitMQ Java client
 * Sugar-coated API for task queues that hides all the AMQP machinery from the developer
 * A port of Bunny or amqp gem to Clojure



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

``` clojure
[com.novemberain/langohr "1.4.1"]
```

### With Maven

Add Clojars repository definition to your `pom.xml`:

``` xml
<repository>
  <id>clojars.org</id>
  <url>http://clojars.org/repo</url>
</repository>
```

And then the dependency:

``` xml
<dependency>
  <groupId>com.novemberain</groupId>
  <artifactId>langohr</artifactId>
  <version>1.4.1</version>
</dependency>
```

### Verifying Your Installation

You can verify your installation in the REPL:

    $ lein repl
    user=> (require 'langohr.core)
    ;= nil
    user=> langohr.core/*default-config*
    ;= {:host "localhost", :port 5672, :username "guest", :vhost "/", :password "guest"}


## "Hello, World" example

Let us begin with the classic "Hello, world" example. First, here is the code:

``` clojure
(ns clojurewerkz.langohr.examples.hello-world
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb]))

(def ^{:const true}
  default-exchange-name "")

(defn message-handler
  [ch {:keys [content-type delivery-tag type] :as meta} ^bytes payload]
  (println (format "[consumer] Received a message: %s, delivery tag: %d, content type: %s, type: %s"
                   (String. payload "UTF-8") delivery-tag content-type type)))

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        qname "langohr.examples.hello-world"]
    (println (format "[main] Connected. Channel id: %d" (.getChannelNumber ch)))
    (lq/declare ch qname :exclusive false :auto-delete true)
    (lc/subscribe ch qname message-handler :auto-ack true)
    (lb/publish ch default-exchange-name qname "Hello!" :content-type "text/plain" :type "greetings.hi")
    (Thread/sleep 2000)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

This example demonstrates a very common communication scenario: *application A* wants to publish a message that will end up in a queue that *application B* listens on. In this case, the queue name is "langohr.examples.hello-world". Let us go through the code step by step:

``` clojure
(ns clojurewerkz.langohr.examples.hello-world
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.exchange  :as le]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb]))
```

defines our example app namespace that requires (loads) several Langohr namespaces:

 * `langohr.core`
 * `langohr.channel`
 * `langohr.queue`
 * `langohr.basic`
 * `langohr.consumers`

and will be compiled ahead-of-time (so we can run it).

Clojure applications are compiled to JVM bytecode. The `-main` function is the entry point.

A few things is going on here:

 * We connect to RabbitMQ using `langohr.core/connect`. We pass no arguments so connection parameters are all defaults.
 * We open a new channel. Channels is a way to have multiple logical "connections" per AMQP connection.
 * We declare a queue with the name `"langohr.examples.hello-world"`
 * We start consuming messages from it
 * We publish a message, wait for 2 seconds and disconnect

### Connect to RabbitMQ

``` clojure
(let [conn (rmq/connect)]
  (comment ...))
```

connects to RabbitMQ at `127.0.0.1:5672` with vhost of `"/"`, username of `"guest"` and password of `"guest"` and returns the connection.

`rmq` is an alias for `langohr.core` (see the ns snippet above).

### Open a Channel

``` clojure
(let [conn  (rmq/connect)
      ch    (lch/open conn)]
  (comment ...))
```

opens a new *channel*. AMQP is a multi-channeled protocol (connections have one or more channels). This makes it possible for highly concurrent
clients to reuse a single TCP connection for multiple "virtual connections".

`lch` is an alias for `langohr.channel`.

### Declare a Queue

The code below

``` clojure
(lq/declare ch qname :exclusive false :auto-delete true)
```

declares a *queue* on the channel that we have just opened. Consumer applications get messages from queues. We declared this queue with the "auto-delete"
and "non-exclusive" parameters. Basically, this means that the queue will be deleted when our little app exits.

### Start a Consumer

Now that we have a queue, we can start consuming messages from it:

``` clojure
(lc/subscribe ch queue-name message-handler :auto-ack true)
```

We use `langohr.queue/subscribe` to start a consumer. This creates a new thread which will consume the messages.

Finally, here's the handling function:

``` clojure
(defn message-handler
  [ch {:keys [content-type delivery-tag type] :as meta} ^bytes payload]
  (println (format "[consumer] Received a message: %s, delivery tag: %d, content type: %s, type: %s"
                   (String. payload "UTF-8") delivery-tag content-type type)))
```

It takes a channel the consumer uses, a Clojure map of message metadata and message payload as array of bytes. We turn it into a string
and print it, as well as a few message properties.

### Publish a Message

Next we publish a message to an *exchange*. Exchanges receive messages that are sent by producers. Exchanges route messages to queues according to rules
called *bindings*. In this particular example, there are no explicitly defined bindings. The exchange that we defined is known as the *default exchange* and
it has implicit bindings to all queues.

Routing key is one of the *message properties* (also called attributes). The default exchange will route the message to a queue that has the same name as the message's
routing key. This is how our message ends up in the `"langohr.examples.hello-world"` queue.

``` clojure
(lb/publish ch default-exchange-name qname "Hello!" :content-type "text/plain" :type "greetings.hi")
```

### Disconnect

Then we use `langohr.core/close`, a polymorphic function, to close both the channel and connection.

``` clojure
(rmq/close ch)
(rmq/close conn)
```

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

``` clojure
(ns clojurewerkz.langohr.examples.blabbr
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.exchange  :as le]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb]))

(defn start-consumer
  "Starts a consumer bound to the given topic exchange in a separate thread"
  [ch topic-name username]
  (let [queue-name (format "nba.newsfeeds.%s" username)
        handler    (fn [ch {:keys [content-type delivery-tag type] :as meta} ^bytes payload]
                     (println (format "[consumer] %s received %s" username (String. payload "UTF-8"))))]
    (lq/declare ch queue-name :exclusive false :auto-delete true)
    (lq/bind    ch queue-name topic-name)
    (lc/subscribe ch queue-name handler :auto-ack true)))

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        ex    "nba.scores"
        users ["joe" "aaron" "bob"]]
    (le/declare ch ex "fanout" :durable false :auto-delete true)
    (doseq [u users]
      (start-consumer ch ex u))
    (lb/publish ch ex "" "BOS 101, NYK 89" :content-type "text/plain" :type "scores.update")
    (lb/publish ch ex "" "ORL 85, ALT 88"  :content-type "text/plain" :type "scores.update")
    (Thread/sleep 2000)
    (rmq/close ch)
    (rmq/close conn)))
```

In this example, opening a channel is no different to opening a channel in the previous example, however, we do one extra thing: declare an exchange:

``` clojure
(let [conn  (rmq/connect)
      ch    (lch/open conn)
      ex    "nba.scores"]
  (le/declare ch ex "fanout" :durable false :auto-delete true))
```

The exchange that we declare above is a *fanout exchange*. A fanout exchange delivers messages to all of the queues that are bound to it: exactly what we want in the
case of Blabbr.

This piece of code

``` clojure
(let [queue-name (format "nba.newsfeeds.%s" username)
      handler    (fn [ch {:keys [content-type delivery-tag type] :as meta} ^bytes payload]
                   (println (format "[consumer] %s received %s" username (String. payload "UTF-8"))))]
  (lq/declare ch queue-name :exclusive false :auto-delete true)
  (lq/bind    ch queue-name topic-name)
  (lc/subscribe ch queue-name handler :auto-ack true))```

is similar to the subscription code that we used for message delivery previously, but also does a bit more: it sets up a binding between the queue and the exchange
that was declared earlier. We need to do this to make sure that our fanout exchange routes messages to the queues of all subscribed followers.

Another difference in this example is the routing key we use for publishing: it is an empty string:

``` clojure
(lb/publish ch ex "" "BOS 101, NYK 89" :content-type "text/plain" :type "scores.update")
(lb/publish ch ex "" "ORL 85, ALT 88"  :content-type "text/plain" :type "scores.update")
```

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

``` clojure
(ns clojurewerkz.langohr.examples.weathr
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.exchange  :as le]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb]))

(def ^{:const true}
  weather-exchange "weathr")

(defn start-consumer
  "Starts a consumer bound to the given topic exchange in a separate thread"
  [ch topic-name queue-name]
  (let [queue-name' (.getQueue (lq/declare ch queue-name :exclusive false :auto-delete true))
        handler     (fn [ch {:keys [routing-key] :as meta} ^bytes payload]
                      (println (format "[consumer] Consumed '%s' from %s, routing key: %s" (String. payload "UTF-8") queue-name' routing-key)))]
    (lq/bind    ch queue-name' weather-exchange :routing-key topic-name)
    (lc/subscribe ch queue-name' handler :auto-ack true)))

(defn publish-update
  "Publishes a weather update"
  [ch payload routing-key]
  (lb/publish ch weather-exchange routing-key payload :content-type "text/plain" :type "weather.update"))

(defn -main
  [& args]
  (let [conn      (rmq/connect)
        ch        (lch/open conn)
        locations {""               "americas.north.#"
                   "americas.south" "americas.south.#"
                   "us.california"  "americas.north.us.ca.*"
                   "us.tx.austin"   "#.tx.austin"
                   "it.rome"        "europe.italy.rome"
                   "asia.hk"        "asia.southeast.hk.#"}]
    (le/declare ch weather-exchange "topic" :durable false :auto-delete true)
    (doseq [[k v] locations]
      (start-consumer ch v k))
    (publish-update ch "San Diego update" "americas.north.us.ca.sandiego")
    (publish-update ch "Berkeley update"  "americas.north.us.ca.berkeley")
    (publish-update ch "SF update"        "americas.north.us.ca.sanfrancisco")
    (publish-update ch "NYC update"       "americas.north.us.ny.newyork")
    (publish-update ch "São Paolo update" "americas.south.brazil.saopaolo")
    (publish-update ch "Hong Kong update" "asia.southeast.hk.hongkong")
    (publish-update ch "Kyoto update"     "asia.southeast.japan.kyoto")
    (publish-update ch "Shanghai update"  "asia.southeast.prc.shanghai")
    (publish-update ch "Rome update"      "europe.italy.roma")
    (publish-update ch "Paris update"     "europe.france.paris")
    (Thread/sleep 2000)
    (rmq/close ch)
    (rmq/close conn)))
```

The first line that is different from the Blabbr example is

``` clojure
(le/declare ch weather-exchange "topic" :durable false :auto-delete true)
```

We use a topic exchange here. Topic exchanges are used for [multicast](http://en.wikipedia.org/wiki/Multicast) messaging where consumers indicate
which topics they are interested in (think of it as subscribing to a feed for an individual tag in your favourite blog as opposed to the full feed).
Routing with a topic exchange is done by specifying a *routing pattern* on binding, for example:

``` clojure
(lq/bind ch queue-name' weather-exchange :routing-key topic-name)
```

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

``` clojure
(let [queue-name' (.getQueue (lq/declare ch queue-name :exclusive false :auto-delete true))]
  (comment ...))
```

To declare a server-named queue you pass queue name as an empty string and RabbitMQ will generate a cluster-unique name for you. The name
will be returned to the client and you can access it using Java interop on the value `langohr.queue/declare` returns to you. We do
this to output queue name.

Server-named queues are commonly used for collecting replies to other messages (the "request-reply" pattern, often referred to as "RPC") or
when no application needs to know the exact queue name, it just has to be unique.

When you run this example, the output will look a bit like this:

```
[consumer] Consumed 'San Diego update' from us.california, routing key: americas.north.us.ca.sandiego
[consumer] Consumed 'San Diego update' from amq.gen-AbT8568AYqRY1sC8uB1myf, routing key: americas.north.us.ca.sandiego
[consumer] Consumed 'Berkeley update' from us.california, routing key: americas.north.us.ca.berkeley
[consumer] Consumed 'Berkeley update' from amq.gen-AbT8568AYqRY1sC8uB1myf, routing key: americas.north.us.ca.berkeley
[consumer] Consumed 'SF update' from us.california, routing key: americas.north.us.ca.sanfrancisco
[consumer] Consumed 'SF update' from amq.gen-AbT8568AYqRY1sC8uB1myf, routing key: americas.north.us.ca.sanfrancisco
[consumer] Consumed 'NYC update' from amq.gen-AbT8568AYqRY1sC8uB1myf, routing key: americas.north.us.ny.newyork
[consumer] Consumed 'São Paolo update' from americas.south, routing key: americas.south.brazil.saopaolo
[consumer] Consumed 'Hong Kong update' from asia.hk, routing key: asia.southeast.hk.hongkong
```

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
