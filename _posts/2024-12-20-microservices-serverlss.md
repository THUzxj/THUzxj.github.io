---
layout: post
title: "The difference between microservices and serverless computing: the event-driven architecture"
date: 2024-12-20 00:00:00
description: The event-driven architecture in serverless computing mitigates the overhead introduced by decomposing the monolithic application into microservices.
tags: microservice serverless cloud
categories: academic
featured: true
---

TL;DR: The event-driven architecture in serverless computing mitigates the overhead introduced by decomposing the monolithic application into microservices.

Microservice architecture and serverless computing are two popular architectural styles for building cloud-native applications. Both of them support automatic scaling, manage instances in containers and decompose complex applications, making them able to be used to build scalable and resilient applications. However, while microservices let the developers manage the interaction between services, serverless computing takes over the infrastructure management totally and provides an event-driven architecture.

Microservices decompose the monolithic application into a set of loosely coupled services, while also introducing the network overhead of the communication between services. Different from function calls, the communication between services is usually done through the network. When one service calls another service, it needs to serialize the data, send it over the network, wait for the response, deserialize the data in the response, and continue the execution. What's more, while waiting for the response, the process of callee service blocks execution and still maintains the state of execution in memory. The waiting limits the ability of the service to handle more concurrent requests.

In two cases, this waiting is unnecessary. The first one is that the callee service B returns the response data from service C to service A that calls service B, without any more processing, for example, the confirmation of DB writing. This means that sending the data to service B and deserializing and serializing the data by service B are unnecessary. The second one is that the callee service B only did some simple processes on the response data without using too much or any sensitive context, such as filtering some entries. This waiting is also not necessary if some more APIs are added to service C.

The event-driven architecture in serverless computing avoids the unnecessary overhead of waiting for responses and network communication. Cloud vendors provide various components as event sources and event consumers, provide event distribution and management services, and provide asynchronous and decoupled processing mechanisms. Serverless computing provides immediate results through API Gateway or asynchronous results through various event sources.

Take AWS as an example, event sources include S3, DynamoDB, API Gateway, EventBridge, Simple Queue Service (SQS), Simple Notification Service (SNS), etc. Event consumers include Lambda, Step Functions, ECS, etc. Event distribution and management services include EventBridge, SQS, SNS, etc. Asynchronous and decoupled processing mechanisms include Step Functions, SQS, SNS, EventBridge, etc.

However, the implementation of event-driven computing relies on the modification of application code and the developers' understanding of the complex workflows. Some other disadvantages, including dependence on specific cloud platforms and the difficulty of migrating stateful applications, also prevent the adoption of serverless.  

By the way, developers can benefit from event-driven architecture in microservice. [Vert.x](https://github.com/eclipse-vertx/vert.x), a tool-kit for building reactive applications provided by Eclipse, also uses event-driven API to save resources and reach higher throughput. 

