---
layout: post
title: Resource Management of Cloud
date: 2024-12-10 00:00:00
description: 
tags: cloud VM "resource management"
categories: academic
featured: true
---

Cloud computing keeps growing and provides many kinds of products for various workloads. Cloud providers and academic researchers focus on how to provide the given services with less cost, that is more efficiently. Numerous works focus on improving significant components like resource allocation, instance placement, resource harvesting and so on. Specifically, the development of AI promotes the exploration of AI for systems. 

As interested in this important problem, I have surveyed it and tried to figure out some research opportunities. Unfortunately, many of them are done and must be done by companies or the cooperation of companies and academic institutes, because of the limitation of characterizing requests' features, finding practical problems, and validating new ideas in laboratories. However, these works give valuable insights such as how to improve large-scale systems and handle indeterministic workload. 

This article is aimed to summarize some research works and record some of my thoughts about this problem. If there are any errors in this article, feel free to contact me to correct them and I will appreciate it.

## Brief Introduction to Cloud

To abstract and define the problem of resource management, I would like to have a brief introduction of Cloud Computing first.

### Products in the Cloud

There are three main cloud computing models, that is Infrastructure as a Service (IaaS), Platform as a Service (PaaS), and Software as a Service (SaaS). Public cloud providers provide multiple kinds of products in IaaS and PaaS, including:

| Product Type                       | Product Example    | Computing Model |
| ---------------------------------- | ------------------ | --------------- |
| Bare-metal machines                | AWS EC2 Bare Metal | IaaS            |
| Virtual machines                   | AWS EC2            | IaaS            |
| Kubernetes Engine                  | AWS EKS            | PaaS            |
| Serverless / Function as a Service | AWS Lambda         | PaaS            |

### The Deployment of the Products

Given a data center with many resources, how do these different kinds of products run together in the same data center? For the convenience of management and development, the products (may) run hierarchically. Virtual machines are running on the top of physical machines. Containers and container schedulers are running on the top of virtual machines.

The process of deployment is composed of  several steps, including resource allocation, instance placement and preemption & migration.

#### Resource Allocation Specification

Users should specify the resource allocation strategy of the cloud products, such as the size of VMs or containers, and the autoscaling configurations of services.

#### Instance Placement

As the resource pool is composed of multiple nodes, the cloud provider should decide the placement of each instance. The placement should consider both the resource efficiency and the additional requirements from users such as affinity configurations. The placement problem can be modeled as Bin-packing problem, where the nodes are bins and the instances are items.

#### Instance Preemption / Eviction/ Migration

On nodes with an oversubscription mechanism, part of the instances are able to be preempted, or evicted or migrated to release resources for the more important instances. Preemption and migration reduce the reliability of the affected instances. In IaaS, the preemptible instances are called spot VMs or harvest VMs and are cheaper than regular VMs.

### Cloud Targets

#### From User

Users have different targets for cloud projects for various kinds of workloads.

- For computing workload (e.g. batch processing): Less cost
- For latency-sensitive workload (e.g. VMs running user-facing service): No resource contention;  guarantee for providing all the required resource
- For dynamic workload (e.g. serverless/microservice services): Dynamic resource allocation that can scale quickly for burst workload; guarantee for meeting SLO (P95 max response latency is smaller than threshold)

#### From Provider

- Meet users' targets
- Less cost
  - Fewer needed physical machines
  - Less power
  - Less carbon footprint
  - Lower scheduling system overhead
- More profits
  - More product selling
  - Higher resource utilization


## Improvement Opportunities

Referring to the three steps of the deployment of the projects, here are several methods to improve resource management.

### Resource Allocation Optimization

**Insight:** Specifying the resource allocation strategies may be difficult for developers and operators, replacing the process with automatic frameworks can lead to better results.

<details>
<summary>Papers</summary>
<ul>
<li>SelfTune: Tuning Cluster Managers  (MS, NSDI'23)</li>
<li>With Great Freedom Comes Great Opportunity: Rethinking Resource Allocation for Serverless Functions (IST(ULisboa)/INESC-ID and UCLouvain, MS, EuroSys'23)</li>
<li>Karma: Resource Allocation for Dynamic Demands (Cornell, OSDI'23)</li>
<li>Golgi: Performance-Aware, Resource-Efficient Function Scheduling for Serverless Computing (HKUST, WeBank, SoCC'23)</li>
<li>Autopilot: Workload autoscaling at google. (Google, EuroSys'20)</li>
<li>Largescale cluster management at Google with Borg. (Google, EuroSys'15)</li>
<li>Quasar: Resource-efficient and QoS-aware cluster management (ASPLOS'14)</li>
</ul>
</details>


### Oversubscription of Instances to Increase Resource Utilization

**Insight:** As the utilizations of instances on the nodes are always below the highest level, the oversubscription (aka overcommitment) strategy is used to put more instances on the nodes to increase the utilization of the nodes. The simple oversubscription method is to set a fixed oversubscription ratio (> 1), and the scheduler can allocate resources to instances whose sum can exceed the total number of the resources of the node. However, oversubscription introduces the possibility of resource contention when utilizations of multiple instances go high. Some works focus on avoiding the problem through characterizing, profiling, predicting, etc.

##### Papers

- Harmonizing Efficiency and Practicability: Optimizing Resource Utilization in Serverless Computing with Jiagu (SJTU, HW, ATC'24)
- Dynamic Idle Resource Leasing To Safely Oversubscribe Capacity At Meta (Meta, SoCC'24)
- Prediction-Based Power Oversubscription in Cloud Platforms (MS, ATC'21)
- History-based harvesting of spare cycles and storage in large-scale datacenters (OSDI'16)

### Improvement of Underlying Schedulers

**Insight:** The colocation of multiple instances in one nodes depends on the task scheduler of the node. In underlying schedulers, discovering fine-grained improvements can increase resource utilization.

#### Papers

- Maximizing VMs’ IO Performance on Overcommitted CPUs with Fairness (The University of Edinburgh,  HW, SoCC'23)
- Improving resource utilization by timely fine-grained scheduling. (CUHK, SOSP'20)

### Unified Large-Scale Resource management

**Insight:** The unified resource management system that manages all the machines in all datacenters and schedules all workloads of different priorities can lead to a more flexible and optimized resource utilization. 

#### Papers

- Optimizing Resource Allocation in Hyperscale Datacenters: Scalability, Usability, and Experiences (Meta, OSDI'24)
- Gödel: Unified Large-Scale Resource Management and Scheduling at ByteDance (BD, SoCC'23)
- Workload Consolidation in Alibaba Clusters: The Good, the Bad, and the Ugly  (HKUST, Alibaba, SoCC'22)
- Twine: A Unified Cluster Management System for Shared Infrastructure (Facebook, OSDI'20)
- Protean: VM Allocation Service at Scale (MS, OSDI'20)

### Hardware Resource Disaggregation

**Insight:** Resource disaggregation makes instance placement easier and increases the resource utilization. Currently, storage is managed separately, but CPU, memory and GPU resources are coupled in same nodes.

- With Great Freedom Comes Great Opportunity: Rethinking Resource Allocation for Serverless Functions (IST(ULisboa)/INESC-ID and UCLouvain, MS, EuroSys'23)
- LegoOS: A Disseminated, Distributed OS for Hardware Resource Disaggregation (Purdue, OSDI'18)

### Resources Harvesting from Unallocated Resources

**Insight:** Preemptible VMs like spot VMs and harvest VMs are much cheaper but less reliable. Some works are aimed at reducing the damage to the applications caused by the stop of preemptible VMs.

#### Papers

- Unlocking unallocated cloud capacity for long, uninterruptible workloads (CMU, MS, NSDI'23)
- Snape: Reliable and Low-Cost Computing with Mixture of Spot and On-Demand VMs (MS, ASPLOS'23)
- Providing SLOs for Resource-Harvesting VMs in Cloud Platforms (MS, OSDI'20)
- History-Based Harvesting of Spare Cycles and Storage in Large-Scale Datacenters (UMich, MS, OSDI'16)

#### Applications of Harvested Resources

- Parcae: Proactive, Liveput-Optimized DNN Training on Preemptible Instances (CUHK, NSDI'24)
- SpotServe: Serving Generative Large Language Models on Preemptible Instances (CMU, ASPLOS ’24)
- Faster and Cheaper Serverless Computing on Harvested Resources (Cornell, MS, SOSP'21)

## Dataset Resources

### cluster traces:

[Alibaba](https://tianchi.aliyun.com/dataset/6287)

[Alibaba](https://github.com/alibaba/clusterdata)

[Azure](https://github.com/Azure/AzurePublicDataset)

[Google](https://github.com/google/cluster-data)

### serverless traces:

[Alibaba](https://github.com/alibaba/clusterdata)

[Huawei](https://github.com/sir-lab/data-release)

[Azure](https://github.com/Azure/AzurePublicDataset)

## Related Techniques

### Virtualization Techniques



### Resource Contention and Resource Isolation



### Observability

[CPU Utilization is Wrong](https://www.brendangregg.com/blog/2017-05-09/cpu-utilization-is-wrong.html)

[USE Method](https://www.brendangregg.com/usemethod.html)



### Hypervisor's Scheduler

vCPU

vCPU overcommitment

## References

Cloud Computing Models: https://aws.amazon.com/types-of-cloud-computing/

Firecracker: Lightweight Virtualization for Serverless Applications: https://aws.amazon.com/cn/blogs/china/deep-analysis-aws-firecracker-principle-virtualization-container-runtime-technology/



