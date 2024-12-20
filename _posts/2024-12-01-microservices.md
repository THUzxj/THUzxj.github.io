---
layout: post
title: A Survey of Research on Microservices
date: 2024-12-01 00:00:00
description: A Survey of Research on Microservice
tags: microservice cloud
categories: academic
featured: true
---

## Introduction of Microservice

Microservice architecture splits complex monolithic applications into multiple smaller, independent services. Microservices are loosely coupled and can be developed, deployed, and scaled independently. They collaborate through calling well-defined APIs with RPC, HTTP, or message queues.

### Technology Stack

#### Hosting

- Container (e.g., Docker, containerd): Resource isolation
- Kubernetes: Orchestration

#### Communication

- service mesh (e.g., Istio, Linkerd): Traffic management, security, observability
- RPC (e.g., gRPC, Thrift)
- Message queue (e.g., Kafka, RabbitMQ)
- RESTful API

#### Observability

- Tracing (e.g., Jaeger, Zipkin, OpenTelemetry, Google Dapper)

## Research Topics

As microservices have multiple advantages, they also bring challenges, such as the complexity and overhead introduced by the communication between services. However, the decoupling of services enables us to perform fine-grained management of services and networks. The following are some research topics in microservices.

### Resource management

Kubernetes enables the autoscaling of the services. Vertical scaling changes the size of the resources of a single instance, such as the CPU and memory allocation of the container, while horizontal scaling changes the number of instances. The widely-used simple autoscaling algorithm needs the users to configure the range of vertical or horizontal scaling and refers to the CPU and memory usage of the services or other custom metrics to make decisions. The autoscaling algorithm helps the service adjust to dynamic or unpredictable workloads. When the workload is low, it can allocate fewer resources. When the workload increases, it can allocate more resources automatically to handle the request spikes.

However, the decisions of autoscaling are only aware of single services, rather than the SLO of the requests and the resource usage of the whole service. One whole service can be presented as a DAG. One request can be processed by the services in parallel or in serial. The latency of the requests is the sum of the latency of each service layer in the process. 

What's more, the configurations of the microservices are too complex for users. Users only know their SLO when deploying the services, and don't know the SLO of each service, even the suitable resource allocation for each service. 

The basic problem is how to save as many resources as possible while guaranteeing the SLO of the service. A series of research works have been done to solve the related problems. 

#### Methodology

The basic methodology includes three components. The first component involves exploring the observability of services, which provides insights into their current status. For example, leveraging new monitoring metrics to know the status more clearly, like using run queue length to guess the waiting task number (SHOWAR), and using gateway network traffic to guess the current request number (Nodens). 

The second one is the profiling and modelling of the services to predict their performance in different conditions. This component bridges the gap between the concerned performance metrics that we want to optimize, the workloads we need to handle, and the resource allocation that we can control. For each service, we can collect the data points online or offline, and model them with simple functional relationships (Cilantro, Erms). A performance model is not always necessary, as it can be replaced by a feedback-based mechanism.

The third one is the scheduler to solve the optimization problem, that is reducing the resource usage in the constraints of guaranteeing SLO. As microservices give complex and huge action space, many works use neural networks, especially Deep Reinforcement Learning (FIRM, AWARE, Autothrottle). The DL methods have potential to solve complex strategy problems without artificial design. There are also non-DL algorithms to reach the goals (Cilantro, Erms, Derm). These methods introduce more expert knowledges of autoscaling.

#### Evaluations

The evaluation of the performance of these works is challenging. Due to the importance of the SLO and the potential side effects of adopting new methods, these studies typically experiment with dedicated experimental systems. The experimental systems always include microservice benchmarks and load generators simulating the real load traces, running on several hosts.

#### Comments

These works give us valuable insights into the resource management of microservices. They consider the QoS of services and propose various methods to manage resources efficiently.

However, some disadvantages still hinder the practical application of these methods. Here are some of them:

1. **The complexity of the system.** These methods always give a redesign of every component in the scheduler for slightly different goals, thus these methods are not compatible with each other. Moreover, compared to the potential resource savings, the cost of implementing complex algorithms may outweigh the benefits. If we want to gain the advantages of several systems, the combined system needs to be designed with effort and will be much more complex.
2. **The tradeoff of saving resources and throttling the services.** Where do the saved resources come from? Part of them are from the higher resource utilization. Another part of them is from the throttling of services. Because the target is set to reduce SLO violations, the real latencies of the services are ignored if they satisfy the SLO. Thus, the average latency of the services under these schedulers is higher because some services are throttled to save resources.
3. **The difficulty of proving the effectiveness of the methods.** The evaluation of the methods is challenging and current evaluations are not very convincing. The experimental systems evaluate the methods, which are much simpler than the real-world systems. Their generalizability to microservices with different designs, scale and dynamic changes, can not be evaluated. In particular, the DL models' low explainability limits their further usage in production because of the possible unexpected outputs.
4. **The ignorance of exploring the lower-level details of the services.** The latency of the services, the basis of SLO, needs more inspection. The latency contains the waiting time in the running queue, the program processing time, the network latency and so on. To have a further understanding of microservices, besides experiments, we need more explanations from the viewpoint of queue theory, probability theory, OS scheduling, the host network and so on.
5. **To be Continued.**



#### Papers

- ATOM: Model-Driven Autoscaling for Microservices (ICDCS'19)
- PARTIES: QoS-Aware Resource Partitioning for Multiple Interactive Services (ASPLOS'19)
- GrandSLAm: Guaranteeing SLAs for Jobs in Microservices Execution Frameworks (EuroSys'19)
- FIRM: An Intelligent Fine-grained Resource Management Framework for {SLO-Oriented} Microservices (OSDI'20)
- SHOWAR: Right-Sizing And Efficient Scheduling of Microservices (SoCC'21)
- SINAN: ML-based and QoS-aware resource management for cloud microservices (ASPLOS'21)
- ORION and the Three Rights: Sizing, Bundling, and Prewarming for Serverless DAGs (OSDI'22)
- Erms: Efficient Resource Management for Shared Microservices with SLA Guarantees (ASPLOS'23)
- Cilantro: Performance-Aware Resource Allocation for General Objectives via Online Feedback (OSDI'23)
- Nodens: Enabling Resource Efficient and Fast {QoS} Recovery of Dynamic Microservice Applications in Datacenters (ATC'23)
- AWARE: Automate Workload Autoscaling with Reinforcement Learning in Production Cloud Systems (ATC'23)
- Autothrottle: A Practical Bi-Level Approach to Resource Management for SLO-Targeted Microservices (NSDI'24)
- Derm: SLA-aware Resource Management for Highly Dynamic Microservices (ISCA'24)

### Network performance

Microservice architecture introduces more network communication between services. Network communications introduce more overhead than function calls in monolith applications, including the inter-node and intra-node network overhead. If a sidecar mechanism is used, the network overhead will be greater. Thus the average overall latency of the service is higher. 

Placing services with close relationships on the same host can reduce inter-node network traffic and latency. However, it may destroy the independence of the services, which is one of the principle of microservice architecture.

Reducing the host network overhead as much as possible is also a promising direction. Kernel bypass technologies have been applied to RPC.

#### Papers

- A Cloud-Scale Characterization of Remote Procedure Calls (SOSP'23)
- Remote Procedure Call as a Managed System Service (NSDI'23)
- MuCache: A General Framework for Caching in Microservice Graphs (NSDI'24)
- HydraRPC: RPC in the CXL Era (ATC'24)


### Configuration tuning

The configuration space of microservice is huge, thus it is meaningful to build tools to explore the configuration automatically and easily.

#### Papers

- Blueprint: A Toolchain for Highly-Reconfigurable Microservices (SOSP'23)
- OPTIMUSCLOUD: Heterogeneous Configuration Optimization for Distributed Databases in the Cloud (ATC'19)
- μTune: Auto-Tuned Threading for OLDI Microservices (OSDI'18)

### Tracing

Tracing of the request processing in the microservices is important for extracting dependencies, troubleshooting and performance analysis. 

Current tracing systems like OpenTelemetry require modification of the application code to actively report the request information and timestamps.

#### Papers

- Network-Centric Distributed Tracing with DeepFlow: Troubleshooting Your Microservices in Zero Code (SigComm'23)
- CRISP: Critical Path Analysis of Large-Scale Microservice Architectures (ATC'22)
- The Benefit of Hindsight: Tracing Edge-Cases in Distributed Systems (NSDI'23)

### Fault localization

This field focuses on using the relationships between services to locate the position of the performance or logic problem. The unique problem in microservices is that one fault in a service can spread in the call graph and cause multiple faults in related services, thus localizing the root causes is challenging. 

#### Comments

Personally speaking, without experience in production microservices, current fault localization works don't convince me of their motivation about the complexity of finding where the root causes are located. As the model of web services is relatively simple, the status of each service, such as workload overwhelming, resource contention and degration caused by other services, can be inferred through the metrics of the services, including request number, resource usage, response latency. Judging whether the performance problem is caused by the service itself or the other services seems not difficult.

#### Papers

- Sage: Practical & Scalable ML-Driven Performance Debugging in Microservices (ASPLOS'21)
- Murphy: Performance Diagnosis of Distributed Cloud Applications (Sigcomm'23)
- Explainit! – a declarative root-cause analysis engine for time series data. (SIGMOD'19) 

### Simulator of microservices

Simulation is an important tool for understanding the performance of microservices. However, currently there are few actively maintained simulators for microservices.

#### Papers

- PerfSim: A Performance Simulator for Cloud Native Microservice Chains
- μqSim: Enabling Accurate and Scalable Simulation for Interactive Microservices

### Demo microservices for research

- [DeathStarBench](https://github.com/delimitrou/DeathStarBench)
- [Train-Ticket](https://github.com/FudanSELab/train-ticket)
- [Microservice-demo](https://github.com/microservices-demo)

