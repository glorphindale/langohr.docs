---
title: "Working with RabbitMQ queues and consumers from Clojure with Langohr"
layout: article
---

## About this guide

This guide covers everything related to queues in the AMQP v0.9.1 specification, common usage scenarios and how to accomplish
typical operations using Langohr. This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/clojurewerkz/langohr.docs).


## What version of Langohr does this guide cover?

This guide covers Langohr 1.0-beta4.


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

{% gist cd461764671162acedc8 %}

The same example in context:

{% gist 34a585bf186de42aaf3c %}


### Server-named queues

To ask an AMQP broker to generate a unique queue name for you, pass an *empty string* as the queue name argument. The returned
value has the `#getQueue` method that can be used to retrieve the generated queue name:

{% gist c852bac0fffa3e50f07b %}

The same example in context:

{% gist 6d48e14ca352d3a176b5 %}


### Reserved Queue Name Prefix

Queue names starting with "amq." are reserved for internal use by the broker. Attempts to declare a queue with a name that violates this rule will
result in a channel-level exception with reply code 403 (ACCESS_REFUSED) and a reply message similar to this:

    ACCESS_REFUSED - queue name 'amq.queue' contains reserved prefix 'amq.*'

### Queue re-declaration with different attributes

When queue declaration attributes are different from those that the queue already has, a channel-level exception with code `406 (PRECONDITION_FAILED)`
will be raised. The reply text will be similar to this:

    PRECONDITION_FAILED - parameters for queue 'langohr.examples.channel_exception' in vhost '/' not equivalent

## Queue life-cycle patterns

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

{% gist 13e785d0a282b1c8b755 %}

The same example in context:

{% gist 319bd470028b95a9a561 %}


## Declaring a Temporary Exclusive Queue

To declare a server-named, exclusive, auto-deleted queue, pass "" (an empty string) as the queue name and use the `:exclusive`:

{% gist c852bac0fffa3e50f07b %}

The same example in context:

{% gist 6d48e14ca352d3a176b5 %}

Exclusive queues may only be accessed by the current connection and are deleted when that connection closes. The declaration of an exclusive queue by other
connections is not allowed and will result in a channel-level exception with the code `405 (RESOURCE_LOCKED)`

Exclusive queues will be deleted when the connection they were declare on is closed.


## Binding Queues to Exchanges

In order to receive messages, a queue needs to be bound to at least one exchange. Most of the time binding is explcit (done by applications). To bind a queue to an exchange,
use the `langohr.queue/bind` function:

{% gist %}






## Wrapping Up

TBD

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
