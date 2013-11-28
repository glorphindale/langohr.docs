---
title: "Working with RabbitMQ bindings from Clojure with Langohr"
layout: article
---

## About this guide

This guide covers bindings in AMQP 0.9.1, what they are, what role they play and how to accomplish typical operations using Langohr.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/clojurewerkz/langohr.docs).


## What version of Langohr does this guide cover?

This guide covers Langohr 1.7.x.


## Bindings In AMQP 0.9.1

Learn more about how bindings fit into the AMQP Model in the [AMQP Concepts](http://www.rabbitmq.com/tutorials/amqp-concepts.html) guide.


## What Are AMQP 0.9.1 Bindings

Bindings are rules that exchanges use (among other things) to route messages to queues. To instruct an exchange E to route messages to a queue Q,
Q has to *be bound* to E. Bindings may have an optional *routing key* attribute used by some exchange types. The purpose of the routing key is to selectively
match only specific (matching) messages published to an exchange to the bound queue. In other words, the routing key acts like a filter.

To draw an analogy:

 * Queue is like your destination in New York city
 * Exchange is like JFK airport
 * Bindings are routes from JFK to your destination. There may be no way, or more than one way, to reach it

Some exchange types use routing keys while some others do not (routing messages unconditionally or based on message metadata). If an AMQP message cannot
be routed to any queue (for example, because there are no bindings for the exchange it was published to), it is either dropped or returned to the publisher,
depending on the message attributes that the publisher has set.

If an application wants to connect a queue to an exchange, it needs to *bind* them. The opposite operation is called *unbinding*.


## Binding Queues To Exchanges

In order to receive messages, a queue needs to be bound to at least one exchange. Most of the time binding is explcit (done by applications). To bind a queue to an exchange,
use the `langohr.queue/bind` function:

``` clojure
(require '[langohr.queue :as lq])

(lq/bind ch "images.resize" "amq.topic")
```

The same example in context:

``` clojure
(require '[langohr.core    :as rmq])
(require '[langohr.channel :as lch])
(require '[langohr.queue   :as lq])


(let [conn (rmq/connect)
      ch   (lch/open conn)]
  (lq/bind ch "images.resize" "amq.topic"))
```


## Unbinding Queues From Exchanges

To unbind a queue from an exchange use the `langohr.queue/unbind` function:

``` clojure
(require '[langohr.basic :as lb])
(lq/unbind channel queue "amq.topic" "streams.twitter.#")
```

Note that trying to unbind a queue from an exchange that the queue was never bound to will
result in a channel-level exception.


## Bindings, Routing and Returned Messages

### How AMQP 0.9.1 Brokers Route Messages

After an AMQP message reaches an AMQP broker and before it reaches a consumer, several things happen:

 * AMQP broker needs to find one or more queues that the message needs to be routed to, depending on type of exchange
 * AMQP broker puts a copy of the message into each of those queues or decides to return the message to the publisher
 * AMQP broker pushes message to consumers on those queues or waits for applications to fetch them on demand

A more in-depth description is this:

 * AMQP broker needs to consult bindings list for the exchange the message was published to in order to find one or more queues that the message needs to be routed to (step 1)
 * If there are no suitable queues found during step 1 and the message was published as mandatory, it is returned to the publisher (step 1b)
 * If there are suitable queues, a _copy_ of the message is placed into each one (step 2)
 * If the message was published as mandatory, but there are no active consumers for it, it is returned to the publisher (step 2b)
 * If there are active consumers on those queues and the basic.qos setting permits, message is pushed to those consumers (step 3)
 * If there are no active consumers and the message is *not* published as mandatory, it will be left in the queue

The important thing to take away from this is that messages may or may not be routed and it is important for applications to handle unroutable messages.

### Handling of Unroutable Messages

Unroutable messages are either dropped or returned to producers. RabbitMQ extensions can provide additional ways of handling unroutable messages: for example,
the [Alternate Exchanges extension](http://www.rabbitmq.com/extensions.html#alternate-exchange) makes it possible to route unroutable
messages to another exchange. amqp gem support for it is documented in the [RabbitMQ Extensions guide](/articles/extensions.html).

RabbitMQ 2.6 introduced a new feature called "dead letter exchange" where unroutable messages will be put instead of dropping them.

Langohr provides a way to handle returned messages with the *return listener* functions.

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
    (.addReturnListener ch rl)
    (lb/publish ch default-exchange-name qname "Hello!" :content-type "text/plain" :mandatory true)
    (Thread/sleep 1000)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

A returned message handler has access to AMQP method (`basic.return`) information, message metadata and payload (as a byte array).
The metadata and message body are returned without modifications so that the application can store the message for later redelivery.

The [Exchanges guide](/articles/exchanges.html) provides more information on the subject, including code examples.




## Wrapping Up

Bindings is how messages get from exchanges to queues in RabbitMQ. Bindings are dynamic and managed by applications.
When creating a binding, it is important to pay attention to the routing key used and what exchange type is.

If a message is not routable can be either "dead lettered" or returned to the publisher.


## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [RabbitMQ Extensions to AMQP 0.9.1](/articles/rabbitmq_extensions.html)
 * [Durability and Related Matters](/articles/durability.html)
 * [Error Handling and Recovery](/articles/error_handling.html)
 * [Troubleshooting](/articles/troubleshooting.html)
 * [Using TLS (SSL) Connections](/articles/tls.html)
