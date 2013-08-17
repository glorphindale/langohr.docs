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

This guide covers Langohr 1.4.x.



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





## Wrapping Up

TBD

## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [Durability and Related Matters](/articles/durability.html)
 * [Error Handling and Recovery](/articles/error_handling.html)
 * [Troubleshooting](/articles/troubleshooting.html)
 * [Using TLS (SSL) Connections](/articles/tls.html)



## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on Twitter](http://twitter.com/clojurewerkz) or the [Clojure RabbitMQ mailing list](https://groups.google.com/forum/#!forum/clojure-rabbitmq)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
