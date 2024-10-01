---
layout: post
title: Decoupling REST API Response Module and Long-Running Task Execution Module
subtitle: Lower coupling in Backend Services
# cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/coupling.png
# share-img: /assets/img/path.jpg
tags: [Backend, Server Software, Micro-service, Design Pattern]
author: Yuchen Jiang
readtime: true
---

## Introduction

As digital demands escalate, backend systems must evolve to manage both immediate and delayed operations effectively. Decoupling REST APIs from long-running task modules is key to boosting system performance and scalability.

Imagine a scenario where a REST API receives a request to call a ai agent(probably with LLM embedded) to process complex task, the waiting time for the response can be intolerable. In such cases, decoupling the REST API from the task execution module can significantly improve system performance and user experience.

## Table of Contents

- [Introduction](#introduction)
- [Table of Contents](#table-of-contents)
- [The Need for Decoupling](#the-need-for-decoupling)
- [Design Patterns for Decoupling](#design-patterns-for-decoupling)
- [Implementing Decoupling with Message Queues](#implementing-decoupling-with-message-queues)
  - [Core Components](#core-components)
  - [Workflow](#workflow)
  - [Benefits](#benefits)
- [Deploying Components on Separate Nodes](#deploying-components-on-separate-nodes)
- [Providing Real-Time Feedback](#providing-real-time-feedback)
  - [Real-Time Communication Methods](#real-time-communication-methods)
  - [Implementation Overview for WebSockets](#implementation-overview-for-websockets)
- [Simple Structure](#simple-structure)
- [Best Practices and Considerations](#best-practices-and-considerations)
  - [Ensuring Reliability and Fault Tolerance](#ensuring-reliability-and-fault-tolerance)
  - [Monitoring and Logging](#monitoring-and-logging)
  - [Scalability Strategies](#scalability-strategies)
  - [Data Consistency and Integrity](#data-consistency-and-integrity)
- [Further Discussions](#further-discussions)
  - [Is it always necessary to do so?](#is-it-always-necessary-to-do-so)
  - [How to handle the failure cases?](#how-to-handle-the-failure-cases)
- [Conclusion](#conclusion)

## The Need for Decoupling

Directly handling long-running tasks in REST APIs can lead to higher latency, scalability challenges, and reduced reliability.

Separating REST APIs from task execution modules enhances responsiveness, scales more efficiently, and isolates failures, ensuring smoother operation.

## Design Patterns for Decoupling

Several design patterns facilitate the decoupling of REST APIs from long-running tasks:

1. **Message Queue Pattern**: Utilizes message brokers like Pulsar, RabbitMQ or Kafka to handle asynchronous communication.
2. **Asynchronous Task Process**: The API server delegates tasks to workers for execution, and responds immediately with task identifiers.
4. **Microservices Architecture**: Structures the system into independent services, each handling specific functionalities.
5. **Event-Driven Architecture (optional)**: Uses events to trigger task execution, promoting loose coupling.

And the focus here is on the **Message Queue Pattern** and **Microservices Architecture**.

## Implementing Decoupling with Message Queues

### Core Components

- **Producer (API Server)**: Sends task messages to the queue.
- **Message Queue**: Acts as an intermediary to store and forward messages.
- **Consumer (Worker)**: Retrieves and processes tasks from the queue asynchronously.

### Workflow

1. **Receive Request**: The API server accepts a client request to perform a task.
2. **Enqueue Task**: The API server publishes a message containing task details to the message queue.
3. **Acknowledge Response**: The API server immediately responds to the client with a task identifier.
4. **Process Task**: Workers consume messages from the queue and execute the tasks asynchronously.
5. **Store Results**: Upon completion, workers update the task status and store results in a database.
6. **Client Polling or Notification**: Clients can query the task status or receive real-time updates.

### Benefits

- Easily scale workers based on task volume without impacting the API server.
- Message queues ensure tasks are not lost and can be retried in case of failures.

The important thing here is how should we design the message protocol, how to ensure the message is delivered and processed correctly, and how to handle the failure cases.

## Deploying Components on Separate Nodes

Once the REST API and task execution modules are decoupled, deploying them on separate nodes offers several advantages:

- **Scalability**: Scale REST APIs and workers independently according to their specific load requirements.
- **Fault Isolation**: Prevent failures in one module from affecting others, prevent cascading failures.
- **Decoupled Technologies**: Use different technologies best suited for each service, such as Node.js for APIs and Python for workers.

## Providing Real-Time Feedback

For the long-running tasks, providing real-time feedback to users is essential for a better user experience. Like the progress of the complex task graph, the status of the task, and the final result.

### Real-Time Communication Methods

A common practice here is to set up a full-duplex communication channel between clients and servers to provide instant updates, like WebSockets. Or maybe Server-Sent Events (SSE).

### Implementation Overview for WebSockets

In my previous project, I set up a WebSocket service on the same REST API server, which caused complex communication logic between client and server, due to the establishment of a new connection. A better approach is to set up a dedicated WebSocket server to manage WebSocket connections and relay messages.

1. **WebSocket Server**: A dedicated service to manage WebSocket connections and relay messages.
2. **Client Connection**: Clients establish a WebSocket connection to receive live updates on task progress.
3. **Message Broadcasting**: Workers publish progress updates to a messaging system (e.g., Redis Pub/Sub), which the WebSocket server listens to and forwards to the appropriate clients.

## Simple Structure

```plaintext
+-------------------+          +-------------------+          +-------------------+
|                   |          |                   |          |                   |
|  REST API Server  |          |  WebSocket Server |          | Task Worker Server|
|                   |          |                   |          |                   |
+-------------------+          +-------------------+          +-------------------+
         |                             |                              |
         |                             |                              |
         +-------------+---------------+--------------+---------------+
                       |                              |
                       |          Message Queue       |
                       |    (Pulsar/RabbitMQ/Kafka)   |
                       +------------------------------+

```

<!-- ### Benefits

- **Immediate Updates**: Clients receive task progress in real-time without polling.
- **Efficient Communication**: Reduces overhead compared to continuous HTTP requests.
- **Enhanced User Experience**: Provides a seamless and interactive interface for users monitoring task progress. -->

## Best Practices and Considerations

### Ensuring Reliability and Fault Tolerance

- **Message Acknowledgments**: Ensure messages are acknowledged only after successful task completion to prevent loss or duplication.
- **Retry Mechanisms**: Implement retries for failed tasks, with exponential backoff strategies to handle transient issues.
- **Dead Letter Queues**: Route unprocessable messages to dead letter queues for later analysis and handling.

### Monitoring and Logging

- **Centralized Monitoring**: Use tools like Prometheus and Grafana to monitor the health and performance of all services.
- **Distributed Tracing**: Implement tracing solutions like Jaeger or Zipkin to track requests across services, aiding in debugging and performance optimization.
- **Comprehensive Logging**: Maintain detailed logs for each service to facilitate issue resolution and system audits.

### Scalability Strategies

- **Auto-Scaling**: Configure auto-scaling policies based on metrics like CPU usage or queue length to dynamically adjust the number of service instances.
- **Load Balancing**: Distribute incoming traffic evenly across service instances using load balancers that support sticky sessions for WebSockets.
- **Efficient Resource Allocation**: Optimize resource allocation based on service-specific needs, ensuring that each service has the necessary capacity to handle its workload.

### Data Consistency and Integrity

- **Idempotent Operations**: Design task processing to be idempotent, ensuring that repeated executions do not lead to inconsistent states.
- **Eventual Consistency**: Embrace eventual consistency where strict immediate consistency is not feasible, ensuring that data converges to a consistent state over time.
- **Transactional Boundaries**: Clearly define transactional boundaries within services to manage data integrity without relying on distributed transactions.

## Further Discussions

### Is it always necessary to do so?

Clearly not. Consider your project based on the scale, the complexity of the task, and the user experience you want to provide. After all it's a trade-off between the complexity of the system and the user experience.

### How to handle the failure cases?

For the message queue part, there are many common practices to handle the failure cases, like the dead letter queue. For the WebSocket part, you can set up a reconnection mechanism to handle the connection failure.

## Conclusion

After decoupling REST APIs from long-running task execution modules, you'll find that your backend services will be much more easy to scale, maintain, and monitor. By leveraging message queues, microservices architecture, and real-time communication methods, you can build a robust and responsive system that meets the demands of modern applications.