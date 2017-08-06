---
layout: post
title: "rabbitmq vs kafka"
date: July 20, 2017
---

###Kafka VS Rabbitmq

>Message queue is a well known architectural pattern that provide an asynchronous communication protocol. It means that sender and receiver of the message are completely detached, they do not need to interact with a queueing system at the same time.


####MQ makes message queues (or message brokers) very useful for:

* decoupling of services
* painless scaling
* backpressure management


==============

`Kafka`: - high-throughput distributed messaging system

 `RabbitMQ` - reliable message broker (AMQP based)
 
| Feature | RabbitMq | Kafka |
| :---         |     :---:      |          ---: |
| ` what is it`   |  RabbitMQ is a solid, mature, general purpose message broker that supports several standardized protocols such as AMQP      | Apache Kafka is a message bus optimized for high-ingress data streams and replay   |
| ` Primary use`   |  High-throughput and reliable background jobs, communication and integration within, and between applications.      | Build applications that process and re-process streamed data on disk    |
| ` routing approach`   |  rich: topic -> direct,topic,headers,fanout      | simple: topic -> Consumer Group    |
|` throuput / 1 pc`     | 20000 msg/sec      | 100000 msg/sec     |
| `client SDK support`    | multiple languages      | multiple languages      |
| `message delay`    |  μs      | ms      |
| `federated queues`    |  can be use to migrate cluster      | kafka mirror-maker have same function     |
| `availability and failover`    |    HA with mirror queue    | HA with replica      |
| `message lost`   |   under control    | Under control after 0.11      |
| `message duplicated`    |    under control    |1） not avoidable prior to 0.11 because of at lease once semantics, 0.11 introduced exactly once semantics 2）kafka stream     |
| `Order Message`    |  Exclusive Consumer or Exclusive Queues can ensure ordering      | Ensure ordering of messages within a partition     |
| `message priority`    |  supported      | not supported      |
| `message track`    |  supported      | not supported      |
| `Management and Operation Tools`    |  supported as plugin     | not supported (need to install 3rd party tool e.g. kafka-manager)    |
| `scalibility`    | Depends on the consumer,  you can add many consumers as you want, and there is no limit in theory      | Depends on the partition number, eg: 500 partitions, only 500 threads can consume from the topic, not meaning to add more threads.      |


`Conclusion is simple:`

* Kafka is good for high throuput scenario

* RabbitMQ is good for reliable and flexible topic routing scenario ((surely, rabbitmq can also handle large throuput, but it needs more machines compred with kafka.)

###Our rabbitmq use case:

 * `Notification center service`'
 
    Once message arrives, will notify client
    
 * `Why Rabbitmq in Notification center`
 
    Rabbitmq is mature, and developer is veryfamaliar with python + rabbitmq, and rabbitmq is reliable.
 
 * This function can be implemented with kafka as well, but developer thinks rabbitmq is much easier, and the exchange routing strategy is more flexsible.

### Other rabbitmq use case

  * `User order priority in shopping mall`
  
   
    e.g.: pub A -> Rabbitmq -> consumer B
    
    consumer B get all the orders from rabbitmq to handle, B can take priority of the order from VIP customer, this makes sense.



`Note: There are only two hard problems in distributed systems`

 * 1. Exactly-once delivery 
 
 * 2. Guaranteed order of messages
 
 
####Reference:

 * rabbitmq-hits-one-million-messages-per-second-on-google-compute-engine 

  https://content.pivotal.io/blog/rabbitmq-hits-one-million-messages-per-second-on-google-compute-engine
