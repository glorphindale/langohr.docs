---
title: "Langohr: all documentation guides"
layout: article
---

## Guide list

[Langohr documentation](https://github.com/clojurewerkz/langohr.docs) is organized as a number of guides, covering various topics.

We recommend that you read these guides, if possible, in this order:


###  [Getting started](/articles/getting_started.html)

An overview of Project Name with a quick tutorial that helps you to get started with it. It should take about
15 minutes to read and study the provided code examples.


### [AMQP Concepts](http://www.rabbitmq.com/tutorials/amqp-concepts.html)

This guide covers:

 * AMQP 0.9.1 model overview
 * What are AMQP channels
 * What are AMQP vhosts
 * What are AMQP queues
 * What are AMQP exchanges
 * What are AMQP bindings
 * What are AMQP 0.9.1 classes and methods



### [Conneciting To The Broker](/articles/connecting.html)

This guide covers:

 * How to connect to RabbitMQ with Langohr
 * How to use connection URI to connect to RabbitMQ (also: on Heroku)
 * How to open a channel
 * How to close a channel
 * How to disconnect


### [Queues and Consumers](/articles/queues.html)

This guide covers:

 * How to declare AMQP queues with Langohr
 * Queue properties
 * How to declare server-named queues
 * How to declare temporary exclusive queues
 * How to consume messages ("push API")
 * How to fetch messages ("pull API")
 * Message and delivery properties
 * Message acknowledgements
 * How to purge queues
 * How to delete queues
 * Other topics related to queues


### [Exchanges and Publishing](/articles/exchanges.html)

This guide covers:

 * Exchange types
 * How to declare AMQP exchanges with Langohr
 * How to publish messages
 * Exchange propreties
 * Fanout exchanges
 * Direct exchanges
 * Topic exchanges
 * Default exchange
 * Message and delivery properties
 * Message routing
 * Bindings
 * How to delete exchanges
 * Other topics related to exchanges and publishing


### [Bindings](/articles/bindings.html)

This guide covers:

 * How to bind exchanges to queues
 * How to unbind exchanges from queues
 * Other topics related to bindings


### [Durability and Related Matters](/articles/durability.html) (TBD)

This guide covers:

 * Topics related to durability of exchanges and queues
 * Durability of messages


### [Error Handling and Recovery](/articles/error_handling.html) (TBD)

This guide covers:

 * AMQP 0.9.1 protocol exceptions
 * How to deal with network failures
 * Other things that may go wrong


### [Using TLS (SSL) Connections](/articles/tls.html) (TBD)

This guide covers:

 * How to use TLS (SSL) connections to RabbitMQ with Langohr



### [RabbitMQ Extensions to AMQP 0.9.1](/articles/rabbitmq_extensions.html) (TBD)

This guide covers [RabbitMQ extensions](http://www.rabbitmq.com/extensions.html) and how they are used in Langohr:

 * How to use Publishing Confirms with Langohr
 * How to set per-queue message TTL
 * How to use exchange-to-exchange bindings
 * How to the alternate exchange extension
 * What are consumer cancellation notifications and how to use them
 * Message *dead lettering* and the dead letter exchange


### [Troubleshooting](/articles/troubleshooting.html) (TBD)

This guide covers:

 * What to check when your apps that use Langohr and RabbitMQ misbehave



## Tell Us What You Think!

Please take a moment to tell us what you think about this guide on Twitter or the [Langohr mailing list](https://groups.google.com/forum/?fromgroups#!forum/clojure-rabbitmq)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
