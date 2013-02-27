---
title: "Working with RabbitMQ queues and consumers from Clojure with Langohr"
layout: article
---

## About this guide

This guide covers everything related to queues in the AMQP v0.9.1 specification, common usage scenarios and how to accomplish
typical operations using Langohr. This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/clojurewerkz/langohr.docs).


## What version of Langohr does this guide cover?

This guide covers Langohr 1.0-beta12.


## Queues in AMQP 0.9.1: Overview

### What are AMQP Queues?

*Queues* store and forward messages to consumers. They are similar to mailboxes in SMTP. Messages flow from producing applications to [exchanges](/articles/exchanges.html)
that route them to queues and finally queues deliver the messages to consumer applications (or consumer applications fetch messages as needed).

Note that unlike some other messaging protocols/systems, messages are not delivered directly to queues. They are delivered to exchanges that route
messages to queues using rules known as *bindings*.

AMQP is a programmable protocol, so queues and bindings alike are declared by applications.

### Concept of Bindings

A *binding* is an association between a queue and an exchange. Queues must be bound to at least one exchange in order to receive messages from publishers. Learn more
about bindings in the [Bindings guide](/articles/bindings.html).

### Queue Attributes

Queues have several attributes associated with them:

 * Name
 * Exclusivity
 * Durability
 * Whether the queue is auto-deleted when no longer used
 * Other metadata (sometimes called *X-arguments*)

These attributes define how queues can be used, what their life-cycle is like and other aspects of queue behavior.

## Queue Names and Declaring Queues

Every AMQP queue has a name that identifies it. Queue names often contain several segments separated by a dot ".", in a similar fashion to URI path segments being separated by a slash "/", although almost any string can represent a segment (with some limitations - see below).

Before a queue can be used, it has to be *declared*. Declaring a queue will cause it to be created if it does not already exist. The declaration will have no effect if the queue does already exist and its attributes are the *same as those in the declaration*. When the existing queue attributes are not the same as those in the declaration a channel-level exception is raised. This case is explained later in this guide.

### Explicitly Named Queues

Applications may pick queue names or ask the broker to generate a name for them.

To declare a queue with a particular name, for example, "images.resize", use the `langohr.queue/declare` function:

``` clojure
(require '[langohr.queue :as lq])
 
(lq/declare ch "images.resize" :exclusive false :auto-delete true)
```

The same example in context:

``` clojure
(require '[langohr.core    :as rmq])
(require '[langohr.channel :as lch])
(require '[langohr.queue   :as lq])
 
 
(let [conn (rmq/connect)
      ch   (lch/open conn)]
  (lq/declare ch "images.resize"  :exclusive false :auto-delete true))
```


### Server-named queues

To ask an AMQP broker to generate a unique queue name for you, pass an *empty string* as the queue name argument. The returned
value is a map that has the `:queue` key that can be used to retrieve the generated queue name:

``` clojure
(require '[langohr.queue :as lq])
 
(let [{:keys [queue]} (lq/declare ch "" :exclusive true)]
  (println (format "Declared a queue named %s" queue)))
```

The same example in context:

``` clojure
(require '[langohr.core    :as rmq])
(require '[langohr.channel :as lch])
(require '[langohr.queue   :as lq])
 
 
(let [conn  (rmq/connect)
      ch    (lch/open conn)
      {:keys [queue]} (lq/declare ch "" :exclusive true)]
  (comment ...))
```

A more convenient way is to use `langohr.queue/declare-server-named` which takes one less argument
(no need to specify queue name as an empty string) and returns the name of the queue:

``` clojure
(require '[langohr.core    :as rmq])
(require '[langohr.channel :as lch])
(require '[langohr.queue   :as lq])
 
 
(let [conn  (rmq/connect)
      ch    (lch/open conn)
      queue (lq/declare-server-named ch :exclusive true)]
  (comment ...))
```


### Reserved Queue Name Prefix

Queue names starting with "amq." are reserved for internal use by the broker. Attempts to declare a queue with a name that violates this rule will
result in a channel-level exception with reply code 403 (ACCESS_REFUSED) and a reply message similar to this:

    ACCESS_REFUSED - queue name 'amq.queue' contains reserved prefix 'amq.*'

### Queue Re-Declaration With Different Attributes

When queue declaration attributes are different from those that the queue already has, a channel-level exception with code `406 (PRECONDITION_FAILED)`
will be raised. The reply text will be similar to this:

    PRECONDITION_FAILED - parameters for queue 'langohr.examples.channel_exception' in vhost '/' not equivalent

## Queue Life-cycle Patterns

According to the AMQP 0.9.1 specification, there are two common message queue life-cycle patterns:

 * Durable message queues that are shared by many consumers and have an independent existence: i.e. they will continue to exist and collect messages whether or not there are consumers to receive them.
 * Temporary message queues that are private to one consumer and are tied to that consumer. When the consumer disconnects, the message queue is deleted.

There are some variations of these, such as shared message queues that are deleted when the last of many consumers disconnects.

Let us examine the example of a well-known service like an event collector (event logger). A logger is usually up and running regardless of the existence of services
that want to log anything at a particular point in time. Other applications know which queues to use in order to communicate with the logger and can rely on those queues
being available and able to survive broker restarts. In this case, explicitly named durable queues are optimal and the coupling that is created between
applications is not an issue.

Another example of a well-known long-lived service is a distributed metadata/directory/locking server like [Apache Zookeeper](http://zookeeper.apache.org),
[Google's Chubby](http://labs.google.com/papers/chubby.html) or DNS. Services like this benefit from using well-known, not server-generated,
queue names and so do any other applications that use them.

A different sort of scenario is in "a cloud setting" when some kind of worker/instance might start and stop at any time so that other applications cannot
rely on it being available. In this case, it is possible to use well-known queue names, but a much better solution is to use server-generated, short-lived queues
that are bound to topic or fanout exchanges in order to receive relevant messages.

Imagine a service that processes an endless stream of events — Twitter is one example. When traffic increases, development operations may start additional application
instances in the cloud to handle the load. Those new instances want to subscribe to receive messages to process, but the rest of the system does not know anything about
them and cannot rely on them being online or try to address them directly. The new instances process events from a shared stream and are the same as their peers. In a case
like this, there is no reason for message consumers not to use queue names generated by the broker.

In general, use of explicitly named or server-named queues depends on the messaging pattern that your application needs. [Enterprise Integration Patterns](http://www.eaipatterns.com/)
discusses many messaging patterns in depth and the RabbitMQ FAQ also has a section on [use cases](http://www.rabbitmq.com/faq.html#scenarios).

## Declaring a Durable Shared Queue

To declare a durable shared queue, you pass a queue name that is a non-blank string and use the `:durable` option:

``` clojure
(require '[langohr.queue :as lq])
 
(lq/declare ch "images.resize" :durable true :auto-delete false :exclusive false)
```

The same example in context:

``` clojure
(require '[langohr.core    :as rmq])
(require '[langohr.channel :as lch])
(require '[langohr.queue   :as lq])
 
 
(let [conn (rmq/connect)
      ch   (lch/open conn)]
  (lq/declare ch "images.resize" :durable true :auto-delete false :exclusive false))
```


## Declaring a Temporary Exclusive Queue

To declare a server-named, exclusive, auto-deleted queue, use `langohr.queue/declare-server-named` (or pass "" (an empty string) as the queue name to `langohr.queue/declare`)
and use the `:exclusive` option:

``` clojure
(require '[langohr.queue :as lq])

(lq/declare-server-named ch :exclusive true)
```

The same example in context:

``` clojure
(require '[langohr.core    :as rmq])
(require '[langohr.channel :as lch])
(require '[langohr.queue   :as lq])
 
 
(let [conn  (rmq/connect)
      ch    (lch/open conn)
      queue (lq/declare-server-named ch :exclusive true)]
  (comment ...))
```

Exclusive queues may only be accessed by the current connection and are deleted when that connection closes. The declaration of an exclusive queue by other
connections is not allowed and will result in a channel-level exception with the code `405 (RESOURCE_LOCKED)`

Exclusive queues will be deleted when the connection they were declare on is closed.


## Binding Queues to Exchanges

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


## Subscribing to receive messages ("push API")

To set up a queue subscription to enable an application to receive messages as they arrive in a queue, one uses the `langohr.basic/consume` function
that takes a *consumer*. Consumer is the name for subscription that the AMQP 0.9.1 specification uses. Consumers last as long as the channel that they were declared on,
or until client cancels them (unsubscribes).

Consumers are identified by *consumer tags* and have a number of events they can react on:

 * Message delivery handler
 * Consumer registration confirmation handler
 * Consumer cancellation handler

### Message Delivery Handler

This is the most important of the three. This handler will process messages that RabbitMQ pushes to the consumer.

``` clojure
(fn [ch metadata ^bytes payload]
  (println metadata (String. payload)))
```

The metadata argument is a combined Clojure map of the message metadata (content type, type, reply-to, etc) and
delivery information (routing key, if the mesasge is redelivered, etc), so it is possible to use map destructuring
on it:

``` clojure
(fn [ch {:keys [type content-type]} ^bytes payload]
  (println type content-type (String. payload)))
```


### Consumer Registration Handler

This handler will be invoked when a confirmation (the `basic.consume-ok` method) arrives from RabbitMQ.
This usually happens within milliseconds after registering a consumer. This handler is used
relatively rarely.

``` clojure
(require '[langohr.consumers :as lcons])
 
(lcons/create-default ch :handle-consume-ok-fn (fn [consumer-tag]
                                                 (println "Consumer registered")))
```

The same example in context:

``` clojure
(require '[langohr.core      :as rmq])
(require '[langohr.channel   :as lch])
(require '[langohr.consumers :as lcons])
 
(let [conn (rmq/connect)
      ch   (lch/open conn)]
  (lcons/create-default ch :handle-consume-ok-fn (fn [consumer-tag]
                                                   (println "Consumer registered"))))
```

### Consumer Cancellation Handler

Consumers can be cancelled by RabbitMQ in some situations:

 * When a consumer is cancelled via the RabbitMQ Management UI
 * When the queue messages are consumed from is deleted

This handler will react to *consumer cancellation notifications* when one of the aforementioned events happen.

``` clojure
(require '[langohr.consumers :as lcons])
 
(lcons/create-default ch :handle-cancel-fn (fn [consumer-tag]
                                             (println "Consumer registered")))
```

The same example in context:

``` clojure
(require '[langohr.core      :as rmq])
(require '[langohr.channel   :as lch])
(require '[langohr.consumers :as lcons])
 
(let [conn (rmq/connect)
      ch   (lch/open conn)]
  (lcons/create-default ch :handle-cancel-fn (fn [consumer-tag]
                                               (println "Consumer registered"))))
```


### Consuming Messages

To start consuming messages, pass a consumer to the `langohr.basic/consume` function:

``` clojure
(require '[langohr.basic     :as lb])
(require '[langohr.consumers :as lcons])
 
(let [consumer (lcons/create-default ch :handle-delivery-fn   (fn [ch metadata ^bytes payload]
                                                                (println "Received a message: " (String. payload)))
                                     :handle-consume-ok-fn (fn [consumer-tag]
                                                             (println "Consumer registered")))]
  (lb/consume ch queue consumer))
```

The same example in context:

``` clojure
(require '[langohr.core      :as rmq])
(require '[langohr.channel   :as lch])
(require '[langohr.basic     :as lb])
(require '[langohr.consumers :as lcons])
 
 
(let [conn     (rmq/connect)
      ch       (lch/open conn)
      consumer (lcons/create-default ch :handle-delivery-fn   (fn [ch metadata ^bytes payload]
                                                                (println "Received a message: " (String. payload)))
                                     :handle-consume-ok-fn (fn [consumer-tag]
                                                             (println "Consumer registered")))]
  (lb/consume ch queue consumer))
```

Then when a message arrives, the message header (metadata) and body (payload) are passed to the *delivery handler*.

`langohr.basic/consume` can take a consumer tag (any unique string) or let RabbitMQ generate one. In both cases,
it returns the consumer tag.


### Convenience Method

The `langohr.consumers/subscribe` function starts a consumer that loops and processes messages forever:

``` clojure
(require '[langohr.consumers :as lcm])
 
(lcm/subscribe ch "images.resize"
               (fn [ch metadata ^bytes payload]
                 (println (format "[consumer] %s received %s" username (String. payload "UTF-8"))))
               :auto-ack true)
```

The same example in context:

``` clojure
(require '[langohr.core    :as rmq])
(require '[langohr.channel :as lch])
(require '[langohr.queue   :as lq])
(require '[langohr.consumers :as lcm])
 
(let [conn       (rmq/connect)
      ch         (lch/open conn)
      queue-name (format "nba.newsfeeds.%s" username)
      handler    (fn [ch metadata ^bytes payload]
                   (println (format "[consumer] Received %s" (String. payload "UTF-8"))))]
  (lq/declare ch queue-name :exclusive false :auto-delete true)
  (lq/bind    ch queue-name topic-name)
  (lc/subscribe ch queue-name handler :auto-ack true))
```

It will block the calling thread, so it is usually started in a separate thread. That thread should take care of handling
I/O exceptions that may arise during the consumer's lifespan.


### Accessing Message Metadata

The *metadata* parameter in the example above provides access to message metadata and delivery information:

 * Message content type
 * Message content encoding
 * Message routing key
 * Message delivery mode (persistent or not)
 * Consumer tag this delivery is for
 * Delivery tag
 * Message priority
 * Whether or not message is redelivered
 * Producer application id

and so on. An example to demonstrate how to access some of those attributes via map destructuring:

``` clojure
(defn message-handler
  [ch {:keys [content-type delivery-tag type] :as meta} ^bytes payload]
  (println (format "[consumer] Received a message: %s, delivery tag: %d, content type: %s, type: %s"
                   (String. payload "UTF-8")
                   delivery-tag
                   content-type
                   type)))
```

The full list of keys (note that most of them are optional and may not be present):

 * `:delivery-tag`
 * `:redelivery?`
 * `:exchange`
 * `:routing-key`
 * `:content-type`
 * `:content-encoding`
 * `:headers`
 * `:delivery-mode`
 * `:persistent?`
 * `:priority`
 * `:correlation-id`
 * `:reply-to`
 * `:expiration`
 * `:message-id`
 * `:timestamp`
 * `:type`
 * `:user-id`
 * `:app-id`
 * `:cluster-id`

### Exclusive Consumers

Consumers can request exclusive access to the queue (meaning only this consumer can access the queue). This is useful when you want a long-lived shared queue
to be temporarily accessible by just one application (or thread, or process). If the application employing the exclusive consumer crashes or loses the
TCP connection to the broker, then the channel is closed and the exclusive consumer is cancelled.

To exclusively receive messages from the queue, pass the `:exclusive` option to `langohr.consumers/subscribe`:

``` clojure
(require '[langohr.consumers :as lcm])
 
(lcm/subscribe ch "images.resize"
               (fn [ch metadata ^bytes payload]
                 (comment ...))
               :exclusive true)
```

If a queue has an exclusive consumer, attempts to register another consumer will fail with an [access refused](http://www.rabbitmq.com/amqp-0-9-1-reference.html#constant.access-refused)
channel-level exception (code: 403).

It is not possible to register an exclusive consumer on a queue that already has consumers.


### Using Multiple Consumers Per Queue

It is possible to have multiple non-exclusive consumers on queues. In that case, messages will be
distributed between them according to prefetch levels of their channels (more on this later in this
guide). If prefetch values are equal for all consumers, each consumer will get about the same # of messages.


### Cancelling a Consumer

To cancel a particular consumer, use the `langohr.basic/cancel` function that takes a channel and a consumer tag to cancel:

``` clojure
(require '[langohr.core      :as rmq])
(require '[langohr.channel   :as lch])
(require '[langohr.basic     :as lb])
(require '[langohr.queue     :as lq])
(require '[langohr.consumers :as lcons])
 
(let [channel  (lch/open conn)
      queue    (.getQueue (lq/declare channel))
      latch    (java.util.concurrent.CountDownLatch. 1)
      consumer (lcons/create-default channel :handle-delivery-fn (fn [ch metadata ^bytes payload]
                                                                    (comment ...)))
      cons-tag (lb/consume channel queue consumer :consumer-tag tag)]
  (lb/cancel channel tag))
```

Consumer tag is returned by `langohr.basic/consume` or may be already known to your application.


### Message Acknowledgements

Consumer applications — applications that receive and process messages ‚ may occasionally fail to process individual messages, or will just crash.
There is also the possibility of network issues causing problems. This raises a question — "When should the AMQP broker remove messages from queues?"

The AMQP 0.9.1 specification proposes two choices:

 * After broker sends a message to an application (using either basic.deliver or basic.get-ok methods).
 * After the application sends back an acknowledgement (using basic.ack AMQP method).

The former choice is called the *automatic acknowledgement model*, while the latter is called the *explicit acknowledgement model*.
With the explicit model, the application chooses when it is time to send an acknowledgement. It can be right after receiving a message,
or after persisting it to a data store before processing, or after fully processing the message (for example, successfully fetching a Web page,
processing and storing it into some persistent data store).

![Message Acknowledgements](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/006_amqp_091_message_acknowledgements.png)

If a consumer dies without sending an acknowledgement, the AMQP broker will redeliver it to another consumer, or, if none are available at the time,
the broker will wait until at least one consumer is registered for the same queue before attempting redelivery.

The acknowledgement model is chosen when a new consumer is registered for a queue. By default, `langohr.consumers/subscribe` will use the *explicit* model.
To switch to the *automatic* model, the `:auto-ack` option should be used:

``` clojure
(require '[langohr.core    :as rmq])
(require '[langohr.channel :as lch])
(require '[langohr.queue   :as lq])
(require '[langohr.consumers :as lcm])
 
(let [conn       (rmq/connect)
      ch         (lch/open conn)
      queue-name (format "nba.newsfeeds.%s" username)
      handler    (fn [ch metadata ^bytes payload]
                   (println (format "[consumer] Received %s" (String. payload "UTF-8"))))]
  (lq/declare ch queue-name :exclusive false :auto-delete true)
  (lq/bind    ch queue-name topic-name)
  (lc/subscribe ch queue-name handler :auto-ack true))
```

To demonstrate how redelivery works, let us have a look at the following code example:

``` clojure
(ns clojurewerkz.langohr.examples.redelivery
  (:gen-class)
  (:require [langohr.core      :as rmq]
            [langohr.channel   :as lch]
            [langohr.queue     :as lq]
            [langohr.exchange  :as le]
            [langohr.consumers :as lc]
            [langohr.basic     :as lb])
  (:import [java.util.concurrent TimeUnit ScheduledThreadPoolExecutor Callable]))
 
(def es (ScheduledThreadPoolExecutor. 4))
 
(defn periodically
  [n f]
  (.scheduleWithFixedDelay es ^Runnable f 0 n TimeUnit/MILLISECONDS))
 
(defn after
  [n f]
  (.schedule es ^Callable f n TimeUnit/MILLISECONDS))
 
(defn start-acking-consumer
  [ch queue id]
  (let [handler (fn [ch {:keys [headers delivery-tag redelivery?]} ^bytes payload]
                  (println (format "%s received a message, i = %d, redelivery? = %s, acking..." id (get headers "i") redelivery?))
                  (lb/ack ch delivery-tag))]
    (.start (Thread. (fn []
                       (lc/subscribe ch queue handler :auto-ack false))))))
 
(defn start-skipping-consumer
  [ch queue id]
  (let [handler (fn [ch {:keys [headers delivery-tag]} ^bytes payload]
                  (println (format "%s received a message, i = %d" id (get headers "i"))))]
    (.start (Thread. (fn []
                       (lc/subscribe ch queue handler :auto-ack false))))))
 
 
(defn -main
  [& args]
  ;; N connections imitate N apps
  (let [conn1    (rmq/connect)
        conn2    (rmq/connect)
        conn3    (rmq/connect)
        ch1      (lch/open conn1)
        ch2      (lch/open conn2)
        chx      (lch/open conn3)
        exchange "amq.direct"
        queue    "langohr.examples.redelivery"]
    (lb/qos ch1 1)
    (lb/qos ch2 1)
    (lq/declare chx queue :auto-delete true :exclusive false)
    (lq/bind    chx queue exchange :routing-key "key1")
    ;; this consumer will ack messages
    (start-acking-consumer   ch1 queue "consumer1")
    ;; this consumer won't ack messages and will "crash" in 4 seconds
    (start-skipping-consumer ch2 queue "consumer2")
    (let [i      (atom 0)
          future (periodically 800 (fn []
                                     (try
                                       (lb/publish chx exchange "key1" "" :headers {"i" @i})
                                       (swap! i inc)
                                       (catch Throwable t
                                         (.printStackTrace t)))))]
      (after 4000 (fn []
                    (println "---------------- Shutting down consumer 2 ----------------")
                    (rmq/close ch2)))
      (after 8000 (fn []
                    (println "Shutting down...")
                     (.shutdownNow es)
                     (lq/purge chx queue)
                     (rmq/close ch1)
                     (rmq/close chx)
                     (rmq/close conn1)
                     (rmq/close conn2)
                     (rmq/close conn3))))))
```

So what is going on here? This example uses three AMQP connections to imitate three applications, one producer and two consumers.
Each AMQP connection opens a single channel. The consumers share a queue and the producer publishes messages to the queue periodically using an `amq.direct` exchange.

Both "applications" subscribe to receive messages using the explicit acknowledgement model. The AMQP broker by default will send each message to
the next consumer in sequence (this kind of load balancing is known as *round-robin*). This means that some messages will be delivered
to consumer #1 and some to consumer #2.

To demonstrate message redelivery we make consumer #1 randomly select which messages to acknowledge. After 4 seconds we disconnect it (to imitate a crash).
When that happens, the AMQP broker redelivers unacknowledged messages to consumer #2 which acknowledges them unconditionally. After 10 seconds, this example
closes all outstanding connections and exits.

An extract of output produced by this example:

```
consumer2 received a message, i = 0
consumer1 received a message, i = 1, redelivery? = false, acking...
consumer2 received a message, i = 2
consumer1 received a message, i = 3, redelivery? = false, acking...
consumer2 received a message, i = 4
---------------- Shutting down consumer 2 ----------------
consumer1 received a message, i = 0, redelivery? = true, acking...
consumer1 received a message, i = 2, redelivery? = true, acking...
consumer1 received a message, i = 4, redelivery? = true, acking...
consumer1 received a message, i = 5, redelivery? = false, acking...
consumer1 received a message, i = 6, redelivery? = false, acking...
consumer1 received a message, i = 7, redelivery? = false, acking...
consumer1 received a message, i = 8, redelivery? = false, acking...
consumer1 received a message, i = 9, redelivery? = false, acking...
consumer1 received a message, i = 10, redelivery? = false, acking...
```

As we can see, consumer #1 did not acknowledge three messages (labelled 0, 2 and 4):

``` clojure
consumer2 received a message, i = 0
consumer2 received a message, i = 2
consumer2 received a message, i = 4
```

and then, once consumer #1 had "crashed", those messages were immediately redelivered to the consumer #1:

```
---------------- Shutting down consumer 2 ----------------
consumer1 received a message, i = 0, redelivery? = true, acking...
consumer1 received a message, i = 2, redelivery? = true, acking...
consumer1 received a message, i = 4, redelivery? = true, acking...
consumer1 received a message, i = 5, redelivery? = false, acking...
consumer1 received a message, i = 6, redelivery? = false, acking...
consumer1 received a message, i = 7, redelivery? = false, acking...
consumer1 received a message, i = 8, redelivery? = false, acking...
consumer1 received a message, i = 9, redelivery? = false, acking...
consumer1 received a message, i = 10, redelivery? = false, acking...
```

To acknowledge a message use `langohr.basic/ack`:

``` clojure
(require '[langohr.basic :as lb])
 
(lb/ack ch delivery-tag)
```

`langohr.basic/ack` takes three arguments: a channel, a message *delivery tag* and a flag that indicates whether or not we want to acknowledge multiple messages at once.
Delivery tag is simply a channel-specific increasing number that the server uses to identify deliveries.

When acknowledging multiple messages at once, the delivery tag is treated as "up to and including". For example, if delivery tag = 5 that would mean "acknowledge messages 1, 2, 3, 4 and 5".

<p class="alert alert-error">
Acknowledgements are channel-specific. Applications must not receive messages on one channel and acknowledge them on another.
</p>

<p class="alert alert-error">
A message MUST not be acknowledged more than once. Doing so will result in a channel-level exception (PRECONDITION_FAILED) with an error message like this: "PRECONDITION_FAILED - unknown delivery tag"
</p>

### Rejecting messages

When a consumer application receives a message, processing of that message may or may not succeed. An application can indicate to the broker that message
processing has failed (or cannot be accomplished at the time) by rejecting a message. When rejecting a message, an application can ask the broker to discard or requeue it.

To reject a message use the `langohr.basic/reject` method:

``` clojure
(require '[langohr.basic :as lb])
 
(lb/reject ch delivery-tag)
```

in the example above, messages are rejected without requeueing (broker will simply discard them). To requeue a rejected message, use the second argument
that `langohr.basic/reject` takes:

``` clojure
(require '[langohr.basic :as lb])
 
(lb/reject ch delivery-tag true)
```

### Negative acknowledgements

Messages are rejected with the `basic.reject` AMQP method. There is one limitation that `basic.reject` has:
there is no way to reject multiple messages, as you can do with acknowledgements. However, if you are using [RabbitMQ](http://rabbitmq.com), then there is a solution.
RabbitMQ provides an AMQP 0.9.1 extension known as [negative acknowledgements](http://www.rabbitmq.com/extensions.html#negative-acknowledgements) (nacks) and
Langohr supports this extension. For more information, please refer to the [RabbitMQ Extensions guide](/articles/rabbitmq_extensions.html).

### QoS — Prefetching messages

For cases when multiple consumers share a queue, it is useful to be able to specify how many messages each consumer can be sent at once before sending the next acknowledgement.
This can be used as a simple load balancing technique  to improve throughput if messages tend to be published in batches. For example, if a producing application
sends messages every minute because of the nature of the work it is doing.

Imagine a website that takes data from social media sources like Twitter or Facebook during the Champions League final (or the Superbowl),
and then calculates how many tweets mention a particular team during the last minute. The site could be structured as 3 applications:

 * A crawler that uses streaming APIs to fetch tweets/statuses, normalizes them and sends them in JSON for processing by other applications ("app A").
 * A calculator that detects what team is mentioned in a message, updates statistics and pushes an update to the Web UI once a minute ("app B").
 * A Web UI that fans visit to see the stats ("app C").

In this imaginary example, the "tweets per second" rate will vary, but to improve the throughput of the system and to decrease the maximum number of messages
that the AMQP broker has to hold in memory at once, applications can be designed in such a way that application "app B", the "calculator",
receives 5000 messages and then acknowledges them all at once. The broker will not send message 5001 unless it receives an acknowledgement.

In AMQP parlance this is know as *QoS* or *message prefetching*. Prefetching is configured on a per-channel (typically) or per-connection (rarely used) basis.
To configure prefetching per channel, use the `langohr.basic/qos` function. Let us return to the example we used in the "Message acknowledgements" section:

``` clojure
(lb/qos ch1 1)
(lb/qos ch2 1)
```

In that example, one consumer prefetches three messages and another consumer prefetches just one. If we take a look at the output that the example produces,
we will see that `consumer1` fetched four messages and acknowledged one. After that, all subsequent messages were delivered to `consumer2`:

``` clojure
consumer2 received a message, i = 0
consumer1 received a message, i = 1, redelivery? = false, acking...
consumer2 received a message, i = 2
consumer1 received a message, i = 3, redelivery? = false, acking...
consumer2 received a message, i = 4
---------------- Shutting down consumer 2 ----------------
consumer1 received a message, i = 0, redelivery? = true, acking...
consumer1 received a message, i = 2, redelivery? = true, acking...
consumer1 received a message, i = 4, redelivery? = true, acking...
consumer1 received a message, i = 5, redelivery? = false, acking...
consumer1 received a message, i = 6, redelivery? = false, acking...
consumer1 received a message, i = 7, redelivery? = false, acking...
consumer1 received a message, i = 8, redelivery? = false, acking...
consumer1 received a message, i = 9, redelivery? = false, acking...
consumer1 received a message, i = 10, redelivery? = false, acking...
```

<span class="alert alert-error">The prefetching setting is ignored for consumers that do not use explicit acknowledgements.</span>


## How Message Acknowledgements Relate to Transactions and Publisher Confirms

In cases where you cannot afford to lose a single message, AMQP 0.9.1 applications can use one or a combination of the following protocol features:

 * Publisher confirms (a RabbitMQ-specific extension to AMQP 0.9.1)
 * Publishing messages as immediate
 * Transactions (noticeable overhead)

This topic is covered in depth in the [Working With Exchanges](/articles/exchanges.html) guide. In this guide, we will only mention how
message acknowledgements are related to AMQP transactions and the Publisher Confirms extension.

Let us consider a publisher application (P) that communications with a consumer (C) using AMQP 0.9.1. Their communication can be graphically represented like this:

<pre>
-----       -----       -----
|   |   S1  |   |   S2  |   |
| P | ====> | B | ====> | C |
|   |       |   |       |   |
-----       -----       -----
</pre>

We have two network segments, S1 and S2. Each of them may fail. P is concerned with making sure that messages cross S1, while broker (B) and C are concerned with ensuring
that messages cross S2 and are only removed from the queue when they are processed successfully.

Message acknowledgements cover reliable delivery over S2 as well as successful processing. For S1, P has to use transactions (a heavyweight solution) or the more lightweight
Publisher Confirms RabbitMQ extension.


## Fetching messages when needed ("pull API")

The AMQP 0.9.1 specification also provides a way for applications to fetch (pull) messages from the queue only when necessary.
For that, use the `langohr.basic/get` function which returns a pair of `[metadata payload]`:

``` clojure
(require '[langohr.basic :as lb])
 
(let [[metadata payload] (lb/get ch "indexing.changes")]
  (comment ...))
```

The same example in context:

``` clojure
(require '[langohr.core    :as rmq])
(require '[langohr.channel :as lch])
(require '[langohr.basic   :as lb])
 
(let [conn               (rmq/connect)
      ch                 (lch/open conn)
      [metadata payload] (lb/get ch "indexing.changes")]
  (comment ...))
```

The metadata map has the same keys as for delivery handlers (see the "Push API" section above).

If the queue is empty, then `nil` will be returned.

## Unsubscribing From Messages

Sometimes it is necessary to unsubscribe from messages without deleting a queue. To do so, use the `langohr.basic/cancel` function:

``` clojure
(require '[langohr.basic :as lb])
 
(lb/cancel channel consumer-tag)
```

The consumer tag is either known to your application ahead of time or generated by the broker and returned by `langohr.basic/consume`.

In AMQP parlance, unsubscribing from messages is often referred to as "cancelling a consumer". Once a consumer is cancelled, messages will
no longer be delivered to it, however, due to the asynchronous nature of the protocol, it is possible for "in flight" messages to be received
after this call completes.

Fetching messages with `langohr.basic/get` is still possible even after a consumer is cancelled.


## Unbinding Queues From Exchanges

To unbind a queue from an exchange use the `langohr.queue/unbind` function:

``` clojure
(require '[langohr.basic :as lb])
(lq/unbind channel queue "amq.topic" "streams.twitter.#")
```

Note that trying to unbind a queue from an exchange that the queue was never bound to will
result in a channel-level exception.

## Querying the Number of Messages in a Queue

It is possible to query the number of messages sitting in the queue by declaring the queue
with the `:passive` attribute set.
The response (`queue.declare-ok` AMQP method) will include the number of messages along with
other attributes. However, Langohr provides a convenience function `langohr.queue/status`:

``` clojure
(require '[langohr.queue :as lq])
 
(let [{:keys [message-count]} (lq/status ch "search.indexer")]
  message-count)
```

## Querying the Number of Consumers On a Queue

It is possible to query the number of consumers on a queue by declaring the queue with the ":passive" attribute set. The response (`queue.declare-ok` AMQP method)
will include the number of consumers along with other attributes. However, Langohr provides a convenience function `langohr.queue/status`:

``` clojure
(require '[langohr.queue :as lq])
 
(let [{:keys [consumer-count]} (lq/status ch "search.indexer")]
  consumer-count)
```

## Purging queues

It is possible to purge a queue (remove all of the messages from it) using the `langohr.queues/purge` function:

``` clojure
(require '[langohr.queue :as lq])
 
(lb/purge channel "events.recent")
```

Note that this example purges a newly declared queue with a unique server-generated name. When a queue is declared,
it is empty, so for server-named queues, there is no need to purge them before they are used.

## Deleting Queues

To delete a queue, use the `langohr.queue/delete` function:

``` clojure
(require '[langohr.queue :as lq])
 
(lb/delete channel "events.summery.22.09.2012")
```

When a queue is deleted, all of the messages in it are deleted as well.


## Queue Durability vs Message Durability

See [Durability guide](/articles/durability.html)


## RabbitMQ Extensions Related to Queues

See [RabbitMQ Extensions guide](/articles/rabbitmq_extensions.html)



## Wrapping Up

AMQP queues can be client-named or server-named. It is possible to either subscribe for
messages to be pushed to consumers (register a consumer) or pull messages from the client
as needed. Consumers are identified by consumer tags.

For messages to be routed to queues, queues need to be bound to exchanges.

Most functions related to queues are found in two Langohr namespaces:

 * `langohr.queue`
 * `langohr.basic`


## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

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
