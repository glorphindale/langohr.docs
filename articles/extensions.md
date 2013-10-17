---
title: "Working with RabbitMQ extensions from Clojure with Langohr"
layout: article
---

## About this guide

Langohr supports all [RabbitMQ extensions to AMQP 0.9.1](http://www.rabbitmq.com/extensions.html):

  * [Publisher confirms](http://www.rabbitmq.com/confirms.html)
  * [Negative acknowledgements](http://www.rabbitmq.com/nack.html) (basic.nack)
  * [Exchange-to-Exchange Bindings](http://www.rabbitmq.com/e2e.html)
  * [Alternate Exchanges](http://www.rabbitmq.com/ae.html)
  * [Per-queue Message Time-to-Live](http://www.rabbitmq.com/ttl.html#per-queue-message-ttl)
  * [Per-message Time-to-Live](http://www.rabbitmq.com/ttl.html#per-message-ttl)
  * [Queue Leases](http://www.rabbitmq.com/ttl.html#queue-ttl)
  * [Consumer Cancellation Notifications](http://www.rabbitmq.com/consumer-cancel.html)
  * [Sender-selected Distribution](http://www.rabbitmq.com/sender-selected.html)
  * [Dead Letter Exchanges](http://www.rabbitmq.com/dlx.html)
  * [Validated user_id](http://www.rabbitmq.com/validated-user-id.html)

This guide briefly describes how to use these extensions with Langohr.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/ruby-amqp/rubybunny.info).


## What version of Langohr does this guide cover?

This guide covers Langohr 1.5.x.




## Publisher Confirms (Publisher Acknowledgements)

In some situations it is essential that messages are reliably
delivered to the RabbitMQ broker and not lost on the way. The only
reliable ways of assuring message delivery are by using publisher
confirms or [transactions](http://www.rabbitmq.com/semantics.html).

The [Publisher Confirms AMQP
extension](http://www.rabbitmq.com/blog/2011/02/10/introducing-publisher-confirms/)
was designed to solve the reliable publishing problem in a more
lightweight way compared to transactions.

Publisher confirms are similar to message acknowledgements (documented
in the [Queues and Consumers](/articles/queues/) guide), but involve a
publisher and a RabbitMQ node instead of a consumer and a RabbitMQ
node.

<a href="https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/006_amqp_091_message_acknowledgements.png">
  <img src="https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/006_amqp_091_message_acknowledgements.png" style="max-width: 100%" alt="RabbitMQ Message Acknowledgements"/>
</a>

<a href="https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/007_rabbitmq_publisher_confirms.png">
  <img src="https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/007_rabbitmq_publisher_confirms.png" style="max-width: 100%" alt="RabbitMQ Publisher Confirms"/>
</a>

### How To Use It With Langohr

To use publisher confirms, first put the channel into confirmation
mode using the `` function:

``` clojure
(ns langohr.examples
  (:require [langohr.confirms :as lcf]))

(lcf/select ch)
```

From this moment on, every message published on this channel will be
given a unique identifier (delivery tag). Unacknowledged delivery tags
are tracked in a set and removed from the set when a confirmation (or
negative confirmation) arrives.

`#waitForConfirms` is a method that can be used to make current thread
wait until all outstanding confirms arrive. It returns true if all
confirmations were positive and false otherwise.

Starting with Langohr 1.5, the same can be done with the
`langohr.confirm/wait-for-confirms` function.

### Example

``` clojure
(ns clojurewerkz.langohr.examples.publisher-confirms
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.confirm   :as lcf]
            [langohr.queue     :as lq]
            [langohr.exchange  :as lx]
            [langohr.basic     :as lb]))

(def ^{:const true}
  default-exchange-name "")

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (doto (lch/open conn)
                (lcf/select))
        q     (lq/declare-server-named ch)]
    (dotimes [n 1000]
      (lb/publish ch default-exchange-name q "msg"))
    (.waitForConfirms ch)
    (println "All confirms arrived...")
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

In the example above, the `Bunny::Channel#wait_for_confirms` method blocks (waits) until all of the published messages are confirmed by the RabbitMQ broker. **Note** that a message may be nacked by the broker if, for some reason, it cannot take responsibility for the message. In that case, the `wait_for_confirms` method will return `false` and there is also a Ruby `Set` of nacked message IDs (`channel.nacked_set`) that can be inspected and dealt with as required.

### Learn More

See also rabbitmq.com section on [Publisher Confirms](http://www.rabbitmq.com/confirms.html)





## Queue Leases

Queue Leases is a RabbitMQ feature that lets you set for how long a
queue is allowed to be *unused*. After that moment, it will be
deleted. *Unused* here means that the queue

 * has no consumers
 * is not redeclared
 * no message fetches happened (using `basic.get` AMQP 0.9.1 method, that is, `langohr.basic/get` in Langohr)

### How To Use It With Langohr

Use the `"x-expires"` optional queue argument to set how long the
queue will be allowed to be unused in milliseconds. After that time,
the queue will be removed by RabbitMQ.

``` clojure
(ns langohr.examples
  (:require [langohr.queue :as lq]))

# 500 milliseconds
(lq/declare ch "a.queue" :arguments {"x-expires" 500})
```

### Example

``` clojure
(ns clojurewerkz.langohr.examples.queue-ttl
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.shutdown  :as lsh]))

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        qname "clojurewerkz.langohr.examples.queue-ttl"]
    (lq/declare ch qname :arguments {"x-expires" 500})
    (Thread/sleep 600)
    (try
      (lq/declare-passive ch qname)
      (catch java.io.IOException ioe
          (let [shutdown-ex (.getCause ioe)
                code        (-> (lsh/reason-of shutdown-ex)
                                .getMethod
                                .getReplyCode)]
            (when (= code 404)
              (println "Queue no longer exists")))))
    (println "[main] Disconnecting...")
    (when (rmq/open? ch)
      (rmq/close ch))
    (rmq/close conn)))
```

### Learn More

See also rabbitmq.com section on [Queue Leases](http://www.rabbitmq.com/ttl.html#queue-ttl)



## Consumer Cancellation Notifications

### How To Use It With Langohr

In order to use consumer cancellation notifications, you need to use
consumer objects (documented in the [Queues and Consumers
guide](/articles/queues.html)).  When a consumer is cancelled, an
event handler will be executed.  To register such a callback, use
`:handle-cancel-fn` when registering a consumer with
`langohr.consumers/subscribe` and similar functions:

``` clojure
(ns langohr.examples
  (:require [langohr.consumers :as lcons]))

(lcons/subscribe ch q
                    (fn [ch {:keys [delivery-tag]} ^bytes payload]
                      (comment "No op"))
                    :auto-ack true
                    :handle-cancel-fn (fn [consumer-tag]
                                        (println (format "Consumer %s has been cancelled" consumer-tag))))
```

### Example

``` clojure
(ns clojurewerkz.langohr.examples.consumer-cancel-notifications
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.exchange  :as lx]
            [langohr.basic     :as lb]
            [langohr.consumers :as lcons])
  (:import [java.util.concurrent CountDownLatch TimeUnit]))

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        q     (lq/declare-server-named ch)
        latch (CountDownLatch. 1)]
    (lcons/subscribe ch q
                     (fn [ch {:keys [delivery-tag]} ^bytes payload]
                       (comment "No op"))
                     :auto-ack true
                     :handle-cancel-fn (fn [consumer-tag]
                                         (println (format "Consumer %s has been cancelled" consumer-tag))
                                         (.countDown latch)))
    (lq/delete ch q)
    (.await latch 200 TimeUnit/MILLISECONDS)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

### Learn More

See also rabbitmq.com section on [Consumer Cancellation Notifications](http://www.rabbitmq.com/consumer-cancel.html)




## Per-queue Message Time-to-Live

Per-queue Message Time-to-Live (TTL) is a RabbitMQ extension to AMQP 0.9.1 that allows developers to control how long
a message published to a queue can live before it is discarded.
A message that has been in the queue for longer than the configured TTL is said to be dead. Dead messages will not be delivered
to consumers and cannot be fetched using the *basic.get* operation (`langohr.basic/get`).

Message TTL is specified using the *x-message-ttl* argument on declaration. With Langohr, you pass it
to `langohr.queue/declare`:

``` clojure
(ns langohr.examples
  (:require [langohr.queue :as lq]))

# 1000 milliseconds
(lq/declare ch "a.queue" :arguments {"x-message-ttl" 1000})
```

When a published message is routed to multiple queues, each of the
queues gets a _copy of the message_. If the message subsequently dies
in one of the queues, it has no effect on copies of the message in
other queues.

### Example

The example below sets the message TTL for a new server-named queue to
be 500 milliseconds. It then publishes a message that is
routed to the queue and counts messages in the queue after waiting for
600 milliseconds:

``` clojure
(ns clojurewerkz.langohr.examples.per-queue-message-ttl
  (:gen-class)
  (:require [langohr.core    :as rmq]
            [langohr.channel :as lch]
            [langohr.queue   :as lq]
            [langohr.basic   :as lb]))

(def ^{:const true}
  default-exchange-name "")

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        qname "clojurewerkz.langohr.examples.per-queue-message-ttl"]
    (lq/declare ch qname :arguments {"x-message-ttl" 500} :durable false)
    (lb/publish ch default-exchange-name qname "a message")
    (Thread/sleep 50)
    (println (format "Queue %s has %d messages" qname (lq/message-count ch qname)))
    (println "Waiting for 600 ms")
    (Thread/sleep 600)
    (println (format "Queue %s has %d messages" qname (lq/message-count ch qname)))
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

### Learn More

See also rabbitmq.com section on [Per-queue Message TTL](http://www.rabbitmq.com/ttl.html#per-queue-message-ttl)




## basic.nack

The AMQP 0.9.1 specification defines the basic.reject method that
allows clients to reject individual, delivered messages, instructing
the broker to either discard them or requeue them. Unfortunately,
basic.reject provides no support for negatively acknowledging messages
in bulk.

To solve this, RabbitMQ supports the `basic.nack` method that provides
all of the functionality of basic.reject whilst also allowing for bulk
processing of messages.

### How To Use It With Langohr

Langohr exposes `basic.nack` via the `langohr.basic/nack` function,
similar to `langohr.basic/ack` and `langohr.basic/reject`:

``` clojure
(ns langohr.examples
  (:require [langohr.basic :as lb]))

# requeue multiple messages at once
(lb/nack ch delivery-tag true true)

# requeue a single message at once
(lb/nack ch delivery-tag false true)
```

### Example

``` clojure
(ns clojurewerkz.langohr.examples.basic-nack
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.basic     :as lb]
            [langohr.consumers :as lcons]))

(defn consumer1-fn
  [ch {:keys [delivery-tag]} ^bytes payload]
  (when (>= delivery-tag 29)
    (println "Requeueing all previously received messages...")
    (lb/nack ch delivery-tag true true)))

(defn consumer2-fn
  [ch {:keys [delivery-tag]} ^bytes payload]
  (println (format "Consumer 2 got delivery: %d" delivery-tag))
  (lb/ack ch delivery-tag))

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        qname (:queue (lq/declare ch "clojurewerkz.langohr.examples.basic-nack.q" :exclusive true))]
    (lcons/subscribe ch qname consumer1-fn)
    (lcons/subscribe ch qname consumer2-fn)
    (dotimes [n 30]
      (lb/publish ch "" qname "a message"))
    (Thread/sleep 200)
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

### Learn More

See also rabbitmq.com section on [basic.nack](http://www.rabbitmq.com/nack.html)





## Alternate Exchanges

The Alternate Exchanges RabbitMQ extension to AMQP 0.9.1 allows developers to define "fallback" exchanges
where unroutable messages will be sent.

### How To Use It With Langohr

To specify exchange A as an alternate exchange to exchange B, specify the 'alternate-exchange' argument on declaration of B:

``` clojure
(ns langohr.examples
  (:require [langohr.exchange :as lx]))

(lx/fanout ch x2
              :durable false
              :arguments {"alternate-exchange" x1})
```

### Example

``` clojure
(ns clojurewerkz.langohr.examples.alternate-exchange
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.exchange  :as lx]
            [langohr.basic     :as lb]))

(defn -main
  [& args]
  (let [conn (rmq/connect)
        ch   (lch/open conn)
        x1   "clojurewerkz.langohr.examples.alternate-exchange.x1"
        x2   "clojurewerkz.langohr.examples.alternate-exchange.x2"
        q    (lq/declare-server-named ch)]
    (lx/fanout ch x1 :durable false)
    (lx/fanout ch x2
               :durable false
               :arguments {"alternate-exchange" x1})
    (lq/bind ch q x1)
    (lb/publish ch x2 "_" "a message")
    (Thread/sleep 50)
    (println (format "Queue %s has %d message(s)" q (lq/message-count ch q)))
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

### Learn More

See also rabbitmq.com section on [Alternate Exchanges](http://www.rabbitmq.com/ae.html)






## Sender-Selected Distribution

Generally, the RabbitMQ model assumes that the broker will do the routing work. At times, however, it is useful
for routing to happen in the publisher application. Sender-Selected Routing is a RabbitMQ feature
that lets clients have extra control over routing.

The values associated with the `"CC"` and `"BCC"` header keys will be added to the routing key if they are present.
If neither of those headers is present, this extension has no effect.

### How To Use It With Bunny 0.9+

To use sender-selected distribution, set the `"CC"` and `"BCC"` headers like you would any other header:

``` clojure
(lb/publish ch ex routing-key "a message" :headers {"CC" ["two" "three"]})
```

### Example

``` clojure
(ns clojurewerkz.langohr.examples.sender-selected-distribution
  (:gen-class)
  (:require [langohr.core    :as rmq]
            [langohr.channel :as lch]
            [langohr.queue   :as lq]
            [langohr.basic   :as lb]))

(def ^{:const true}
  default-exchange-name "")

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        q1    "clojurewerkz.langohr.examples.sender-selected-distribution1"
        q2    "clojurewerkz.langohr.examples.sender-selected-distribution2"
        q3    "clojurewerkz.langohr.examples.sender-selected-distribution3"]
    (lq/declare ch q1 :durable false)
    (lq/declare ch q2 :durable false)
    (lq/declare ch q3 :durable false)
    (lb/publish ch default-exchange-name "won't-route-anywhere" "a message" :headers {"CC" [q2 q3]})
    (Thread/sleep 50)
    (println (format "Queue %s has %d messages" q1 (lq/message-count ch q1)))
    (println (format "Queue %s has %d messages" q2 (lq/message-count ch q2)))
    (println (format "Queue %s has %d messages" q3 (lq/message-count ch q3)))
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

### Learn More

See also rabbitmq.com section on [Sender-Selected Distribution](http://www.rabbitmq.com/sender-selected.html)




## Dead Letter Exchange (DLX)

The x-dead-letter-exchange argument to queue.declare controls the exchange to which messages from that queue are 'dead-lettered'.
A message is dead-lettered when any of the following events occur:

The message is rejected (basic.reject or basic.nack) with requeue=false; or the TTL for the message expires.

### How To Use It With Langohr

Dead-letter Exchange is a feature that is used by specifying additional queue arguments:

 * `"x-dead-letter-exchange"` specifies the exchange that dead lettered messages should be published to by RabbitMQ
 * `"x-dead-letter-routing-key"` specifies the routing key that should be used (has to be a constant value)

``` clojure
(lq/declare ch "a-queue" :arguments {"x-dead-letter-exchange" dlx})
```

### Example

``` clojure
(ns clojurewerkz.langohr.examples.dead-letter-exchange
  (:gen-class)
  (:require [langohr.core     :as rmq]
            [langohr.channel  :as lch]
            [langohr.queue    :as lq]
            [langohr.exchange :as lx]
            [langohr.basic    :as lb]))

(def ^{:const true}
  default-exchange-name "")

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        q1    "clojurewerkz.langohr.examples.dlx.q1"
        q2    "clojurewerkz.langohr.examples.dlx.q2"
        dlx   "clojurewerkz.langohr.examples.dlx"]
    (lq/declare ch q1 :durable false :arguments {"x-dead-letter-exchange" dlx
                                                 "x-message-ttl" 300})
    (lq/declare ch q2 :durable false)
    (lx/fanout ch dlx :durable false)
    (lq/bind ch q2 dlx)
    (lb/publish ch default-exchange-name q1 "a message")
    ;; expired messages are dead lettered
    (Thread/sleep 450)
    (println (format "Queue %s has %d messages" q1 (lq/message-count ch q1)))
    (println (format "Queue %s has %d messages" q2 (lq/message-count ch q2)))
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

### Learn More

See also rabbitmq.com section on [Dead Letter Exchange](http://www.rabbitmq.com/dlx.html)


## Exchange-To-Exchange Bindings

RabbitMQ supports [exchange-to-exchange
bindings](http://www.rabbitmq.com/e2e.html) to allow even richer
routing topologies as well as a backbone for some other features
(e.g. tracing).

### How To Use It With Langohr

Langohr exposes it via `langohr.exchange/bind` which is semantically
the same as `langohr.queue/bind` but binds two exchanges:

``` clojure
(ns my.example
  (:require [langohr.exchange :as lx]))

;; x1 is the source, x2 is the destination,
;; the same argument order as in langohr.queue/bind
(lx/bind ch x2 x1 :routing-key "unsorted")
```

### Example

``` clojure
(ns clojurewerkz.langohr.examples.exchange-to-exchange-bindings
  (:gen-class)
  (:require [langohr.core     :as rmq]
            [langohr.channel  :as lch]
            [langohr.queue    :as lq]
            [langohr.exchange :as lx]
            [langohr.basic    :as lb]))

(defn -main
  [& args]
  (let [conn  (rmq/connect)
        ch    (lch/open conn)
        x1    "clojurewerkz.langohr.examples.dlx.x1"
        x2    "clojurewerkz.langohr.examples.dlx.x2"
        qname "clojurewerkz.langohr.examples.dlx.q"]
    (lx/direct ch x1 :durable false)
    (lx/fanout ch x2 :durable false)
    (lq/declare ch qname :exclusive true)
    (lq/bind ch qname x2)
    (lx/bind ch x2 x1 :routing-key "unsorted")
    (lb/publish ch x1 "unsorted" "a message")
    (Thread/sleep 50)
    (println (format "Queue %s has %d message(s)" qname (lq/message-count ch qname)))
    (println "[main] Disconnecting...")
    (rmq/close ch)
    (rmq/close conn)))
```

### Learn More

See also rabbitmq.com section on [Exchange-to-Exchange Bindings](http://www.rabbitmq.com/e2e.html)




## Wrapping Up

TBD

## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [Durability and Related Matters](/articles/durability.html)
 * [Error Handling and Recovery](/articles/error_handling.html)
 * [Troubleshooting](/articles/troubleshooting.html)
 * [Using TLS (SSL) Connections](/articles/tls.html)
