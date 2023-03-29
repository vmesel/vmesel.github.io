---
published: true
title: Hacking Nameko 101 - How to send messages from other services to Nameko
layout: post
tags:
  - nameko
  - python
categories:
  - nameko
  - python
permalink: hacking-nameko-sending-messages-from-other-services
---

Have you ever tried to send messages to a nameko service and failed? This tutorial will enable you to send messages from every other RPC publisher to Nameko using some specific settings for every message.

To begin with, Nameko will always expect two parameters on the properties of the message:
 - **content_type** - the same as we would send in an HTTP request (by default, every request to a Nameko service will use an *application/json* content type)
 - **reply_to** - this parameter is the name of a routing key that the consumer will get the results from after Nameko processes the requests (you will need to create another queue and routing key for this consumer). By default, the routing key will be the name of your service "." the name of the function on your service (i.e. a service with the name *'roger-that'* and a function exposed with RPC named *'say_something'* would result in: *rpc-roger-that.say_something*)

Whenever sending a message to a Nameko service, you will need to send it to its respective exchange and routing key. By default, Nameko's exchanges will be *"nameko-rpc"* and the default name for the routing keys will start with "rpc" and contain the name of your service.

The content of the message is quite tricky: it must be in the format of a Python dictionary, having two keys: *args* and *kwargs* (yes, the same as you would expect in functions).

 - *args*: receives a list (not a tuple as it's not JSON serializable) of arguments
 - *kwargs*: receives a dictionary/JSON object with the argument's name and content

An example of an empty request:
```python
{"args": [], "kwargs": {}}
```

A usable Python code example:

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.basic_publish(
    'nameko-rpc',
    'service-name.service-function-exposed-to-rpc',
    '{"args": [], "kwargs": {}}',
        pika.BasicProperties(
        content_type='application/json',
        reply_to="x",
        delivery_mode=pika.DeliveryMode.Transient
    )
)
print(" [x] Sent 'Hello World!'")
connection.close()
```