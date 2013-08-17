---
title: "Working with RabbitMQ exchanges and publishing messages from Clojure with Langohr"
layout: article
---

## About this guide

This guide covers the use of exchanges according to the AMQP 0.9.1 specification, including broader topics
related to message publishing, common usage scenarios and how to accomplish typical operations using Langohr.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/clojurewerkz/langohr.docs).


## What version of Langohr does this guide cover?

This guide covers Langohr 1.4.x.


## Exchanges in AMQP 0.9.1 — Overview

### What are AMQP exchanges?

An *exchange* accepts messages from a producer application and routes them to message queues. They can be thought of as the
"mailboxes" of the AMQP world. Unlike some other messaging middleware products and protocols, in AMQP, messages are *not* published directly to queues.
Messages are published to exchanges that route them to queue(s) using pre-arranged criteria called *bindings*.

There are multiple exchange types in the AMQP 0.9.1 specification, each with its own routing semantics. Custom exchange types can be created to deal with
sophisticated routing scenarios (e.g. routing based on geolocation data or edge cases) or just for convenience.

### Concept of Bindings

A *binding* is an association between a queue and an exchange. A queue must be bound to at least one exchange in order to receive
messages from publishers. Learn more about bindings in the [Bindings guide](/articles/bindings.html).

### Exchange attributes

Exchanges have several attributes associated with them:

 * Name
 * Type (direct, fanout, topic, headers or some custom type)
 * Durability
 * Whether the exchange is auto-deleted when no longer used
 * Other metadata (sometimes known as *X-arguments*)


## Exchange types

There are four built-in exchange types in AMQP v0.9.1:

 * Direct
 * Fanout
 * Topic
 * Headers

As stated previously, each exchange type has its own routing semantics and new exchange types can be added by extending brokers with plugins. Custom exchange types
begin with "x-", much like custom HTTP headers, e.g. [x-consistent-hash exchange](https://github.com/rabbitmq/rabbitmq-consistent-hash-exchange) or [x-random exchange](https://github.com/jbrisbin/random-exchange).

## Message attributes

Before we start looking at various exchange types and their routing semantics, we need to introduce message attributes. Every AMQP message has a number
of *attributes*. Some attributes are important and used very often, others are rarely used. AMQP message attributes are metadata
and are similar in purpose to HTTP request and response headers.

Every AMQP 0.9.1 message has an attribute called *routing key*. The routing key is an "address" that the exchange may use to decide
how to route the message . This is similar to, but more generic than, a URL in HTTP. Most exchange types use the routing key to implement routing logic,
but some ignore it and use other criteria (e.g. message content).


## Fanout exchanges

### How fanout exchanges route messages

A fanout exchange routes messages to all of the queues that are bound to it and the routing key is ignored. If N queues are bound to a fanout exchange,
when a new message is published to that exchange a *copy of the message* is delivered to all N queues. Fanout exchanges are ideal for the
[broadcast routing](http://en.wikipedia.org/wiki/Broadcasting_%28computing%29) of messages.

Graphically this can be represented as:

![fanout exchange routing](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/004_fanout_exchange.png)

### Declaring a fanout exchange

There are two ways to declare a fanout exchange:

 * Using the `langohr.exchange/declare` function
 * Using the `langohr.exchange/fanout` convenience function

Here are two examples to demonstrate:

``` clojure
(require '[langohr.exchange :as le])
 
(le/declare ch "activity.events" "fanout")
```

``` clojure
(require '[langohr.exchange :as le])
 
(le/fanout ch "activity.events")
```

### Fanout routing example

To demonstrate fanout routing behavior we can declare ten server-named exclusive queues, bind them all to one fanout exchange and then
publish a message to the exchange:

``` clojure
(ns clojurewerkz.langohr.examples.fanout-routing
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.exchange  :as le]
            [langohr.queue     :as lq]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb]))
 
 
(defn start-consumer
  "Starts a consumer in a separate thread"
  [ch queue-name]
  (let [handler (fn [ch metadata ^bytes payload]
                  (println (format "[consumer] %s received a message: %s"
                                   queue-name
                                   (String. payload "UTF-8"))))
        thread  (Thread. (fn []
                           (lc/subscribe ch queue-name handler :auto-ack true)))]
    (.start thread)))
 
(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        ename "langohr.examples.fanout"]
    (le/declare ch ename "fanout")    
    (dotimes [i 10]
      (let [q (.getQueue (lq/declare ch "" :exclusive false :auto-delete true))]
        (lq/bind    ch q ename)
        (start-consumer ch q)))
    (lb/publish ch ename "" "Ping" :content-type "text/plain")
    (Thread/sleep 2000)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

When run, this example produces the following output:

```
[consumer] amq.gen-wh_xtqSpvRY_N6miyl6vWv received a message: Ping
[consumer] amq.gen-wGO_q4bC7p4TuuEU-uHDbw received a message: Ping
[consumer] amq.gen-w2AFtJSaB1Z8mwWYfb0Vl2 received a message: Ping
[consumer] amq.gen-gVJ7vmuTu1zpclZLWGG3-N received a message: Ping
[consumer] amq.gen-g9_mdSEW9kk7yrcjsk5VGg received a message: Ping
[consumer] amq.gen-g-DMp1pnNxhAMBgPWtWIfR received a message: Ping
[consumer] amq.gen-g-5HZnGHH3e7Rdj52EU6Dg received a message: Ping
[consumer] amq.gen-Q74z06gfqFAXIgr6uDDy0H received a message: Ping
[consumer] amq.gen-AXiszOM5LD5MmISDDnEHwV received a message: Ping
[consumer] amq.gen-AHMourpNZFX6Hiq8NtVTaA received a message: Ping
[main] Disconnecting...
```

Each of the queues bound to the exchange receives a *copy* of the message.


### Fanout use cases

Because a fanout exchange delivers a copy of a message to every queue bound to it, its use cases are quite similar:

 * Massively multiplayer online (MMO) games can use it for leaderboard updates or other global events
 * Sport news sites can use fanout exchanges for distributing score updates to mobile clients in near real-time
 * Distributed systems can broadcast various state and configuration updates
 * Group chats can distribute messages between participants using a fanout exchange (although AMQP does not have a built-in concept of presence, so [XMPP](http://xmpp.org may be a better choice))

### Pre-declared fanout exchanges

AMQP 0.9.1 brokers must implement a fanout exchange type and pre-declare one instance with the name of `"amq.fanout"`.

Applications can rely on that exchange always being available to them. Each vhost has a separate instance of that exchange, it is *not shared across vhosts* for obvious reasons.

## Direct exchanges

### How direct exchanges route messages

A direct exchange delivers messages to queues based on a *message routing key*, an attribute that every AMQP v0.9.1 message contains.

Here is how it works:

 * A queue binds to the exchange with a routing key K
 * When a new message with routing key R arrives at the direct exchange, the exchange routes it to the queue if K = R

A direct exchange is ideal for the [unicast routing](http://en.wikipedia.org/wiki/Unicast) of messages (although they can be used
for [multicast routing](http://en.wikipedia.org/wiki/Multicast) as well).

Here is a graphical representation:

![direct exchange routing](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/005_direct_exchange.png)


### Declaring a direct exchange

 * Using the `langohr.exchange/declare` function
 * Using the `langohr.exchange/direct` convenience function

Here are two examples to demonstrate:

``` clojure
(require '[langohr.exchange :as le])
 
(le/declare ch "imaging" "direct")
```

``` clojure
(require '[langohr.exchange :as le])
 
(le/direct ch "imaging")
```

### Direct routing example

Since direct exchanges use the *message routing key* for routing, message producers need to specify it:

``` clojure
(ns clojurewerkz.langohr.examples.direct-routing
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.exchange  :as le]
            [langohr.queue     :as lq]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb]))
 
 
(defn start-consumer
  "Starts a consumer in a separate thread"
  [ch queue-name]
  (let [handler (fn [ch metadata ^bytes payload]
                  (println (format "[consumer] %s received a message: %s"
                                   queue-name
                                   (String. payload "UTF-8"))))
        thread  (Thread. (fn []
                           (lc/subscribe ch queue-name handler :auto-ack true)))]
    (.start thread)))
 
(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        ename "langohr.examples.direct"]
    (le/declare ch ename "direct")
    (let [q (.getQueue (lq/declare ch "" :exclusive false :auto-delete true))]
      (lq/bind ch q ename :routing-key "pings")
      (start-consumer ch q))
    (lb/publish ch ename "pings" "Ping" :content-type "text/plain")
    (Thread/sleep 2000)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

The routing key will then be compared for equality with routing keys on bindings, and consumers that subscribed with
the same routing key each get a copy of the message.


### Direct Exchanges and Load Balancing of Messages

Direct exchanges are often used to distribute tasks between multiple workers (instances of the same application) in a round robin manner.
When doing so, it is important to understand that, in AMQP 0.9.1, *messages are load balanced between consumers and not between queues*.

The [Queues and Consumers](/articles/queues.html) guide provide more information on this subject.

### Pre-declared direct exchanges

AMQP 0.9.1 brokers must implement a direct exchange type and pre-declare two instances:

 * `amq.direct`
 * *""* exchange known as *default exchange* (unnamed, referred to as an empty string by many clients including amqp Ruby gem)

Applications can rely on those exchanges always being available to them. Each vhost has separate instances of those
exchanges, they are *not shared across vhosts* for obvious reasons.


### Default exchange

The default exchange is a direct exchange with no name (the Langohr refers to it using an empty string) pre-declared by the broker. It has one special
property that makes it very useful for simple applications, namely that *every queue is automatically bound to it with a routing key which is the same as the queue name*.

For example, when you declare a queue with the name of "search.indexing.online", the AMQP broker will bind it to the default exchange using "search.indexing.online"
as the routing key. Therefore a message published to the default exchange with routing key = "search.indexing.online" will be routed to the queue "search.indexing.online".
In other words, the default exchange makes it *seem like it is possible to deliver messages directly to queues*, even though that is not technically what is happening.

The default exchange is used by the "Hello, World" example:

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
 
(defn start-consumer
  "Starts a consumer in a separate thread"
  [conn ch queue-name]
  (let [thread (Thread. (fn []
                          (lc/subscribe ch queue-name message-handler :auto-ack true)))]
    (.start thread)))
 
(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        qname "langohr.examples.hello-world"]
    (println (format "[main] Connected. Channel id: %d" (.getChannelNumber ch)))
    (lq/declare ch qname :exclusive false :auto-delete true)
    (start-consumer conn ch qname)
    (lb/publish ch default-exchange-name qname "Hello!" :content-type "text/plain" :type "greetings.hi")
    (Thread/sleep 2000)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```


### Direct Exchange Use Cases

Direct exchanges can be used in a wide variety of cases:

 * Direct (near real-time) messages to individual players in an MMO game
 * Delivering notifications to specific geographic locations (for example, points of sale)
 * Distributing tasks between multiple instances of the same application all having the same function, for example, image processors
 * Passing data between workflow steps, each having an identifier (also consider using headers exchange)
 * Delivering notifications to individual software services in the network

## Topic Exchanges

### How Topic Exchanges Route Messages

Topic exchanges route messages to one or many queues based on matching between a message routing key and the pattern that was used to bind a queue to an exchange.
The topic exchange type is often used to implement various [publish/subscribe pattern](http://en.wikipedia.org/wiki/Publish/subscribe) variations.

Topic exchanges are commonly used for the [multicast routing](http://en.wikipedia.org/wiki/Multicast) of messages.

![](http://upload.wikimedia.org/wikipedia/commons/thumb/3/30/Multicast.svg/500px-Multicast.svg.png)

Topic exchanges can be used for [broadcast routing](http://en.wikipedia.org/wiki/Broadcasting_%28computing%29), but fanout exchanges are usually
more efficient for this use case.

### Topic Exchange Routing Example

Two classic examples of topic-based routing are stock price updates and location-specific data (for instance, weather broadcasts). Consumers indicate which
topics they are interested in (think of it like subscribing to a feed for an individual tag of your favourite blog as opposed to the full feed). The routing is enabled
by specifying a *routing pattern* to the `langohr.queue/bind` function, for example:

``` clojure
(require '[langohr.exchange :as le])
 
(lq/bind ch "crawler.d98fd46e880ab4541d9570f671a9272811c5c438" "requests.crawling" :routing-key "#.org")
```

In the example above we bind a queue with the name of "americas.south" to the topic exchange declared earlier using the `langohr.queue/bind` function. This means that
only messages with a routing key matching "americas.south.#" will be routed to the "americas.south" queue.

A routing pattern consists of several words separated by dots, in a similar way to URI path segments being joined by slash. A few of examples:

 * asia.southeast.thailand.bangkok
 * sports.basketball
 * usa.nasdaq.aapl
 * tasks.search.indexing.accounts

The following routing keys match the "americas.south.#" pattern:

 * americas.south
 * americas.south.*brazil*
 * americas.south.*brazil.saopaolo*
 * americas.south.*chile.santiago*

In other words, the "#" part of the pattern matches 0 or more words.

For the pattern "americas.south.*", some matching routing keys are:

 * americas.south.*brazil*
 * americas.south.*chile*
 * americas.south.*peru*

but not

 * americas.south
 * americas.south.chile.santiago

As you can see, the "*" part of the pattern matches 1 word only.


Full example:

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
    (.start (Thread. (fn []
                       (lc/subscribe ch queue-name' handler :auto-ack true))))))
 
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

### Topic Exchange Use Cases

Topic exchanges have a very broad set of use cases. Whenever a problem involves multiple consumers/applications that selectively choose which type of messages
they want to receive, the use of topic exchanges should be considered. To name a few examples:

 * Distributing data relevant to specific geographic location, for example, points of sale
 * Background task processing done by multiple workers, each capable of handling specific set of tasks
 * Stocks price updates (and updates on other kinds of financial data)
 * News updates that involve categorization or tagging (for example, only for a particular sport or team)
 * Orchestration of services of different kinds in the cloud
 * Distributed architecture/OS-specific software builds or packaging where each builder can handle only one architecture or OS


## Declaring/Instantiating Exchanges

With Langohr, exchanges can be declared in two ways: by using the `langohr.exchange/declare` or by using a number of convenience function:

  * `langohr.exchange/direct`
  * `langohr.exchange/topic`
  * `langohr.exchange/fanout`
  * `langohr.exchange/headers`

The previous sections on specific exchange types (direct, fanout, headers, etc.) provide plenty of examples of how these methods can be used.

## Publishing messages

To publish a message to an AMQP exchange, use `langohr.basic/publish`:

``` clojure
(lb/publish ch default-exchange-name qname "Hello!" :content-type "text/plain" :type "greetings.hi")
```


The function accepts a channel, an exchange, a routing key (can be an empty string but *not nil*), a body and a number of
message and delivery metadata options.

The body can be either a byte array or a string. The message payload is completely opaque to the library and is not modified in any way.

### Data serialization

You are encouraged to take care of data serialization before publishing (i.e. by using JSON, Thrift, Protocol Buffers or some other serialization library).
Note that because AMQP is a binary protocol, text formats like JSON largely lose their advantage of being easy to inspect as data travels across the network,
so if bandwidth efficiency is important, consider using [MessagePack](http://msgpack.org/) or [Protocol Buffers](http://code.google.com/p/protobuf/).

A few popular options for data serialization are:

 * JSON: [Cheshire](http://github.com/dakrone/cheshire)
 * Protocol Buffers: [clojure-protobuf](http://github.com/flatland/clojure-protobuf)
 * Clojure reader forms: [Nippy](https://github.com/ptaoussanis/nippy)
 * Kryo: [Carbonite](https://github.com/revelytix/carbonite)


### Message metadata

AMQP messages have various metadata attributes that can be set when a message is published. Some of the attributes are well-known and mentioned in the AMQP 0.9.1 specification,
others are specific to a particular application. Well-known attributes are listed here as options that `langohr.basic/publish` takes:

 * `:persistent`
 * `:mandatory`
 * `:content-type`
 * `:content-encoding`
 * `:priority`
 * `:message-id`
 * `:correlation-id`
 * `:reply-to`
 * `:type`
 * `:user-id`
 * `:app-id`
 * `:timestamp`
 * `:expiration`

All other attributes can be added to a *headers table* (in Clojure, a map) that `langohr.basic/publish` accepts as the `:arguments` option.

An example:

``` clojure
(lb/publish ch default-exchange-name qname "Hello!"
            :content-type "text/plain"
            :type "greetings.reply"
            :mandatory true
            :reply-to "greetings.hi"
            :arguments {"server" "app5.myapp.megacorp.com"
                        "cached" false})
```

<dl>
  <dt>:routing-key</dt>
  <dd>Used for routing messages depending on the exchange type and configuration.</dd>

  <dt>:persistent</dt>
  <dd>When set to true, AMQP broker will persist message to disk.</dd>

  <dt>:mandatory</dt>
  <dd>
  This flag tells the server how to react if the message cannot be routed to a queue. If this flag is set to true, the server will return an unroutable message
  to the producer with a basic.return AMQP method. If this flag is set to false, the server silently drops the message.
  </dd>

  <dt>:content-type</dt>
  <dd>MIME content type of message payload. Has the same purpose/semantics as HTTP Content-Type header.</dd>

  <dt>:content-encoding</dt>
  <dd>MIME content encoding of message payload. Has the same purpose/semantics as HTTP Content-Encoding header.</dd>

  <dt>:priority</dt>
  <dd>Message priority, from 0 to 9.</dd>

  <dt>:message-id</dt>
  <dd>
    Message identifier as a string. If applications need to identify messages, it is recommended that they use this attribute instead of putting it
    into the message payload.
  </dd>

  <dt>:reply-to</dt>
  <dd>
    Commonly used to name a reply queue (or any other identifier that helps a consumer application to direct its response).
    Applications are encouraged to use this attribute instead of putting this information into the message payload.
  </dd>

  <dt>:correlation-id</dt>
  <dd>
    ID of the message that this message is a reply to. Applications are encouraged to use this attribute instead of putting this information
    into the message payload.
  </dd>

  <dt>:type</dt>
  <dd>Message type as a string. Recommended to be used by applications instead of including this information into the message payload.</dd>

  <dt>:user-id</dt>
  <dd>
  Sender's identifier. Note that RabbitMQ will check that the [value of this attribute is the same as username AMQP connection was authenticated with](http://www.rabbitmq.com/extensions.html#validated-user-id), it SHOULD NOT be used to transfer, for example, other application user ids or be used as a basis for some kind of Single Sign-On solution.
  </dd>

  <dt>:app-id</dt>
  <dd>Application identifier string, for example, "eventoverse" or "webcrawler"</dd>

  <dt>:timestamp</dt>
  <dd>Timestamp of the moment when message was sent, in seconds since the Epoch</dd>

  <dt>:expiration</dt>
  <dd>Message expiration specification as a string</dd>

  <dt>:arguments</dt>
  <dd>A map of any additional attributes that the application needs. Nested hashes are supported. Keys must be strings.</dd>
</dl>

It is recommended that application authors use well-known message attributes when applicable instead of relying on custom headers or placing information in the message body.
For example, if your application messages have priority, publishing timestamp, type and content type, you should use the respective AMQP message attributes
instead of reinventing the wheel.


### Validated User ID

In some scenarios it is useful for consumers to be able to know the identity of the user who published a message. RabbitMQ implements a feature known as [validated User ID](http://www.rabbitmq.com/extensions.html#validated-user-id).
If this property is set by a publisher, its value must be the same as the name of the user used to open the connection. If the user-id property is not set, the publisher's
identity is not validated and remains private.


### Publishing Callbacks and Reliable Delivery in Distributed Environments

A commonly asked question about RabbitMQ clients is "how to execute a piece of code after a message is received".

Message publishing with Langohr happens in several steps:

 * `langohr.basic/publish` takes a payload and various metadata attributes
 * Resulting payload is staged for writing
 * On the next event loop tick, data is transferred to the OS kernel using one of the underlying NIO APIs
 * OS kernel buffers data before sending it
 * Network driver may also employ buffering

<div class="alert alert-error">
As you can see, "when data is sent" is a complicated issue and while methods to flush buffers exist, flushing buffers does not guarantee that the data
was received by the broker because it might have crashed while data was travelling down the wire.

The only way to reliably know whether data was received by the broker or a peer application is to use message acknowledgements. This is how TCP works and this
approach is proven to work at  enormous scale of the modern Internet. AMQP (the protocol) fully embraces this fact and Langohr follows.
</div>

In cases when you cannot afford to lose a single message, AMQP 0.9.1 applications can use one (or a combination of) the following protocol features:

 * Publisher confirms (a RabbitMQ-specific extension to AMQP 0.9.1)
 * Publishing messages as mandatory
 * Transactions (these introduce noticeable overhead and have a relatively narrow set of use cases)

A more detailed overview of the pros and cons of each option can be found in a [blog post that introduces Publisher Confirms extension](http://bit.ly/rabbitmq-publisher-confirms)
by the RabbitMQ team. The next sections of this guide will describe how the features above can be used with Langohr.


### Publishing messages as mandatory

When publishing messages, it is possible to use the `:mandatory` option to publish a message as "mandatory". When a mandatory message cannot be *routed*
to any queue (for example, there are no bindings or none of the bindings match), the message is returned to the producer.

The following code example demonstrates a message that is published as mandatory but cannot be routed (no bindings) and thus is returned back to the producer:

``` clojure
(ns clojurewerkz.langohr.examples.mandatory-publishing
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb]))
 
(def ^{:const true}
  default-exchange-name "")
 
(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        qname (str (java.util.UUID/randomUUID))
        rl    (lb/return-listener (fn [reply-code reply-text exchange routing-key properties body]
                                    (println "Message returned. Reply text: " reply-text)))]
    (.addReturnListener ch rl)
    (lb/publish ch default-exchange-name qname "Hello!" :content-type "text/plain" :mandatory true)
    (Thread/sleep 1000)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```


### Returned messages

When a message is returned, the application that produced it can handle that message in different ways:

 * Store it for later redelivery in a persistent store
 * Publish it to a different destination
 * Log the event and discard the message

Returned messages contain information about the exchange they were published to. Langohr associates
returned message callbacks with consumers. To handle returned messages, use `langohr.basic/add-return-listener`:

``` clojure
(ns clojurewerkz.langohr.examples.mandatory-publishing
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb]))
 
(def ^{:const true}
  default-exchange-name "")
 
(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        qname (str (java.util.UUID/randomUUID))
        rl    (lb/return-listener (fn [reply-code reply-text exchange routing-key properties body]
                                    (println "Message returned. Reply text: " reply-text)))]
    (lb/add-return-listener ch rl)
    (lb/publish ch default-exchange-name qname "Hello!" :content-type "text/plain" :mandatory true)
    (Thread/sleep 1000)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

A returned message handler has access to AMQP method (`basic.return`) information, message metadata and payload (as a byte array).
The metadata and message body are returned without modifications so that the application can store the message for later redelivery.


### Publishing Persistent Messages

Messages potentially spend some time in the queues to which they were routed before they are consumed. During this period of time, the broker may crash or experience a restart.
To survive it, messages must be persisted to disk. This has a negative effect on performance, especially with network attached storage like NAS devices and Amazon EBS.
AMQP 0.9.1 lets applications trade off performance for durability, or vice versa, on a message-by-message basis.

To publish a persistent message, use the `:persistent` option that `langohr.basic/publish` accepts:

``` clojure
(require '[langohr.basic :as lb])
 
(lb/publish ch "requests.crawling" "datsite.net" payload :content-type "application/json" :type "commands.sitemap.refresh" :persistent true)
```

<div class="alert alert-error">
Note that in order to survive a broker crash, both the message and the queue that it was routed to must be persistent/durable.
</div>

[Durability and Message Persistence](/articles/durability.html) provides more information on the subject.

### Publishing In Multi-threaded Environments

<div class="alert alert-error">
When using Langohr in multi-threaded environments, the rule of thumb is: avoid sharing channels across threads.
</div>

In other words, publishers in your application that publish from separate threads should use their own channels. The
same is a good idea for consumers.


## Headers exchanges

Now that message attributes and publishing have been introduced, it is time to take a look at one more core exchange type in AMQP 0.9.1. It is called the *headers
exchange type* and is quite powerful.

### How headers exchanges route messages

#### An Example Problem Definition

The best way to explain headers-based routing is with an example. Imagine a distributed [continuous integration](http://martinfowler.com/articles/continuousIntegration.html)
system that distributes builds across multiple machines with different hardware architectures (x86, IA-64, AMD64, ARM family and so on) and operating systems.
It strives to provide a way for a community to contribute machines to run tests on and a nice build matrix like [the one WebKit uses](http://build.webkit.org/waterfall?category=core).
One key problem such systems face is build distribution. It would be nice if a messaging broker could figure out which machine has which OS, architecture or
combination of the two and route build request messages accordingly.

A headers exchange is designed to help in situations like this by routing on multiple attributes that are more easily expressed as message metadata
attributes (headers) rather than a routing key string.

#### Routing on Multiple Message Attributes

Headers exchanges route messages based on message header matching. Headers exchanges ignore the routing key attribute. Instead, the attributes used for
routing are taken from the "headers" attribute. When a queue is bound to a headers exchange, the `:arguments` attribute is used to define matching rules:

``` clojure
(require '[langohr.queue :as lq])
 
(lq/bind ch "hosts.ip-172-37-11-56" "requests" :arguments {"os" "Linux"})
```

When matching on one header, a message is considered matching if the value of the header equals the value specified upon binding. An example
that demonstrates headers routing:

``` clojure
(ns clojurewerkz.langohr.examples.headers-routing
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.exchange  :as le]
            [langohr.queue     :as lq]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb]))
 
 
(defn start-consumer
  "Starts a consumer in a separate thread"
  [ch queue-name]
  (let [handler (fn [ch metadata ^bytes payload]
                  (println (format "[consumer] %s received a message: %s"
                                   queue-name
                                   (String. payload "UTF-8"))))
        thread  (Thread. (fn []
                           (lc/subscribe ch queue-name handler :auto-ack true)))]
    (.start thread)))
 
(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        ename "langohr.examples.headers"]
    (le/declare ch ename "headers")
    (let [qname (.getQueue (lq/declare ch "" :auto-delete true :exclusive false))]
      (lq/bind ch qname ename :arguments {"os" "linux" "cores" 8 "x-match" "all"})
      (start-consumer ch qname))
    (let [qname (.getQueue (lq/declare ch "" :auto-delete true :exclusive false))]
      (lq/bind ch qname ename :arguments {"os" "osx" "cores" 4 "x-match" "any"})
      (start-consumer ch qname))    
    (lb/publish ch ename "" "8 cores/Linux" :content-type "text/plain" :headers {"os" "linux" "cores" 8})
    (lb/publish ch ename "" "8 cores/OS X"  :content-type "text/plain" :headers {"os" "osx"   "cores" 8})
    (lb/publish ch ename "" "4 cores/Linux" :content-type "text/plain" :headers {"os" "linux" "cores" 4})
    (Thread/sleep 2000)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

When executed, it outputs

```
[consumer] amq.gen-gPQORYX_S-7GCyC-ch_Tc9 received a message: 8 cores/OS X
[consumer] amq.gen-gPQORYX_S-7GCyC-ch_Tc9 received a message: 4 cores/Linux
[consumer] amq.gen-gQmYGGOXey-gUVOEPVmKTV received a message: 8 cores/Linux
[main] Disconnecting...
```


#### Matching All vs Matching One

It is possible to bind a queue to a headers exchange using more than
one header for matching. In this case, the broker needs one more piece
of information from the application developer, namely, should it
consider messages with any of the headers matching, or all of them?
This is what the "x-match" binding argument is for.

When the `"x-match"` argument is set to `"any"`, just one matching
header value is sufficient. So in the example above, any message with
a "cores" header value equal to 8 will be considered matching.



### Declaring a Headers Exchange

To declare a headers exchange, use `langohr.exchange/declare` and specify the exchange type as `"headers"`:

``` clojure
(require '[langohr.exchange :as le])
 
(le/declare ch "langohr.examples.headers" "headers")
```

### Headers Exchange Routing

When there is just one queue bound to a headers exchange, messages are routed to it if any or all of the message headers match those specified upon binding.
Whether it is "any header" or "all of them" depends on the `"x-match"` header value. In the case of multiple queues, a headers exchange will deliver
a copy of a message to each queue, just like direct exchanges do. Distribution rules between consumers on a particular queue are the same as for a direct exchange.

### Headers Exchange Use Cases

Headers exchanges can be looked upon as "direct exchanges on steroids" and because they route based on header values, they can be used as direct exchanges
where the routing key does not have to be a string; it could be an integer or a hash (dictionary) for example.

Some specific use cases:

 * Transfer of work between stages in a multi-step workflow ([routing slip pattern](http://eaipatterns.com/RoutingTable.html))
 * Distributed build/continuous integration systems can distribute builds based on multiple parameters (OS, CPU architecture, availability of a particular package).


### Pre-declared Headers Exchanges

AMQP 0.9.1 brokers [should](http://www.ietf.org/rfc/rfc2119.txt) implement a headers exchange type and pre-declare one instance with
the name of `"amq.match"`. RabbitMQ also pre-declares one instance with the name of `"amq.headers"`. Applications can rely on that exchange always being available to them.
Each vhost has a separate instance of those exchanges and they are *not shared across vhosts* for obvious reasons.

## Custom Exchange Types

### x-random

The [x-random AMQP exchange type](https://github.com/jbrisbin/random-exchange) is a custom exchange type developed as a RabbitMQ plugin by Jon Brisbin.
To quote from the project README:

> It is basically a direct exchange, with the exception that, instead of each consumer bound to that exchange with the same routing key
> getting a copy of the message, the exchange type randomly selects a queue to route to.

This plugin is licensed under [Mozilla Public License 1.1](http://www.mozilla.org/MPL/MPL-1.1.html), same as RabbitMQ.

## Using the Publisher Confirms Extension

Please refer to [RabbitMQ Extensions guide](/articles/rabbitmq_extensions.html)


### Message Acknowledgements and Their Relationship to Transactions and Publisher Confirms

Consumer applications (applications that receive and process messages) may occasionally fail to process individual messages, or might just crash. Additionally,
network issues might be experienced. This raises a question - "when should the AMQP broker remove messages from queues?" This topic is covered
in depth in the [Queues guide](/articles/queues.html), including prefetching and examples.

In this guide, we will only mention how message acknowledgements are related to AMQP transactions and the Publisher Confirms extension. Let us consider
a publisher application (P) that communications with a consumer (C) using AMQP 0.9.1. Their communication can be graphically represented like this:

<pre>
-----       -----       -----
|   |   S1  |   |   S2  |   |
| P | ====> | B | ====> | C |
|   |       |   |       |   |
-----       -----       -----
</pre>

We have two network segments, S1 and S2. Each of them may fail. P is concerned with making sure that messages cross S1, while the broker (B) and C are concerned
with ensuring that messages cross S2 and are only removed from the queue when they are processed successfully.

Message acknowledgements cover reliable delivery over S2 as well as successful processing. For S1, P has to use transactions (a heavyweight solution) or the more
lightweight Publisher Confirms, a RabbitMQ-specific extension.


## Binding Queues to Exchanges

Queues are bound to exchanges using `langohr.queue/bind`. This topic is described in detail in the [Queues guide](/articles/queues.html).


## Unbinding Queues from Exchanges

Queues are unbound from exchanges using `langohr.queue/unbind`. This topic is described in detail in the [Queues guide](/articles/queues.html).

## Deleting Exchange

### Explicitly Deleting an Exchange

Exchanges are deleted using the `langohr.exchange/delete`:

``` clojure
(require '[langohr.exchange :as le])
 
(le/delete ch "requests.crawling")
```

### Auto-deleted exchanges

Exchanges can be *auto-deleted*. To declare an exchange as auto-deleted, use the `:auto_delete` option on declaration:

``` clojure
(require '[langohr.exchange :as le])
 
(le/declare ch "requests.crawling" "topic" :auto-delete true)
```


## Exchange durability vs Message durability

See [Durability guide](/articles/durability.html)


## RabbitMQ Extensions Related to Exchanges

See [Vendor-specific Extensions guide](/articles/rabbitmq_extensions.html)



## Wrapping Up

Publishers publish messages to exchanges. Messages are then routed to queues according to rules called bindings
that applications define. There are 4 built-in exchange types in RabbitMQ and it is possible to create custom
types.

Messages have a set of standard properties (e.g. type, content type) and can carry an arbitrary map
of headers.

Most functions related to exchanges are found in two Langohr namespaces:

 * `langohr.exchange`
 * `langohr.basic`

## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [Bindings](/articles/bindings.html)
 * [RabbitMQ Extensions to AMQP 0.9.1](/articles/rabbitmq_extensions.html)
 * [Durability and Related Matters](/articles/durability.html)
 * [Error Handling and Recovery](/articles/error_handling.html)
 * [Troubleshooting](/articles/troubleshooting.html)
 * [Using TLS (SSL) Connections](/articles/tls.html)



## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on Twitter](http://twitter.com/clojurewerkz) or the [Clojure RabbitMQ mailing list](https://groups.google.com/forum/#!forum/clojure-rabbitmq)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
