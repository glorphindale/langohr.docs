---
title: "TLS (SSL) connections to RabbitMQ from Clojure with Langohr"
layout: article
---

## About This Guide

This guide covers TLS (SSL) connections to RabbitMQ with Langohr.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/clojurewerkz/langohr.docs).


## What version of Langohr does this guide cover?

This guide covers Langohr 1.4.x.


## TLS Support in RabbitMQ

RabbitMQ version 2.x and 3.x support TLS/SSL on Erlang R13B or later. Using the most
recent version (e.g. R16B) is recommended.

To use TLS with RabbitMQ, you need a few things:

 * Client certificate (public key) and key (private key)
 * Server certificate and key
 * [Configure RabbitMQ to use TLS](http://www.rabbitmq.com/ssl.html)
 * If server certificate is self-signed, issuing CA's certificate


## Generating Certificates For Development

The easiest way to generate a CA, server and client keys and certificates is by using
[tls-gen](https://github.com/ruby-amqp/tls-gen/). It requires `openssl` and `make` to be
available.

See [RabbitMQ TLS/SSL guide](http://www.rabbitmq.com/ssl.html) for more information
about TLS support on various platforms.


## Enabling TLS/SSL Support in RabbitMQ

TLS/SSL support is enabled using two arguments:

 * `ssl_listeners` (a list of ports TLS connections will use)
 * `ssl_options` (a proplist of options such as CA certificate file location, server key file location, and so on)

 An example:

``` erlang
[
  {rabbit, [
     {ssl_listeners, [5671]},
     {ssl_options, [{cacertfile,"/path/to/testca/cacert.pem"},
                    {certfile,"/path/to/server/cert.pem"},
                    {keyfile,"/path/to/server/key.pem"},
                    {verify,verify_peer},
                    {fail_if_no_peer_cert,false}]}
   ]}
].
```

Note that all paths must be absolute (no `~` and other shell-isms) and be readable
by the OS user RabbitMQ uses.

Learn more in the [RabbitMQ TLS/SSL guide](http://www.rabbitmq.com/ssl.html).

## Connecting to RabbitMQ with Langohr Using TLS/SSL

TBD

 * `:ssl` (a boolean) which, when set to `true`, will enable TLS on the connection and switch it to TLS port (5671)
 * `:ssl-context` (a `javax.net.ssl.SSLContext` instance) which will provide TLS private key and certificate information, and more

An example:

``` clojure
(ns langohr.examples
  (:require [langohr.core :as rmq]))

;; connect to localhost:5671 using TLS (SSLv3)
(rmq/connect {:ssl true})
```

If you configure RabbitMQ to accept TLS connections on a separate port, you need to
specify all three of `:ssl`, `:ssl-context`, and `:port`.

TBD: a convenient way to create an SSL context

## What to Read Next

The documentation is organized as [a number of
guides](/articles/guides.html), covering various topics.


## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on
Twitter](http://twitter.com/clojurewerkz) or [RabbitMQ mailing
list](https://lists.rabbitmq.com/cgi-bin/mailman/listinfo/rabbitmq-discuss).

Let us know what was unclear or what has not been covered. Maybe you
do not like the guide style or grammar or discover spelling
mistakes. Reader feedback is key to making the documentation better.
