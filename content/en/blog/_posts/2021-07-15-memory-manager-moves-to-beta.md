---
layout: blog
title: "Kubernetes Memory Manager moves to beta"
date: 2021-07-15
slug: kubernetes-1-22-feature-memory-manager-moves-to-beta
---

# Memory Manager moves to beta

This blog post elaborates on the internals of the **Memory Manager**, a beta feature released in Kubernetes v1.22. The **Memory Manager** is a component situated in the **kubelet** ecosystem, and it was proposed to enable the feature of guaranteed memory (and hugepages) allocation, for the Pods in [Guaranteed QoS class](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/#qos-classes).

This blog post covers the following topics:

1. The answer to why you really need it!
2. Details on how the **Memory Manager** operates
3. Current gaps
4. Further work planned for the **Memory Manager** development

## The answer to why you really need it!

In order to obtain the highest performance and lowest latency for the workloads running on NUMA-based servers, the container's CPUs, PCI devices, and memory resources should be aligned to the same NUMA node. The kubelet already offers a set of managers to align both CPUs and PCI devices, but it lacked the alignment of the memory resources. By default, the kernel mechanism attempts to allocate memory from the same NUMA node where container's CPUs triggered the allocation, however, it cannot guarantee that there is enough spare memory on this node.

## How does the Memory Manger operate?

The Memory Manager is responsible for the following tasks:
- the provision of the topology hints to the **Topology Manager**
- the allocation of the memory (hugepages) for containers and respective updates to the state

The sequence diagram below demonstrates the main steps in the operation of Memory Manager triggered by the Kubelet.

![MemoryManagerDiagram
](/static/images/blog/2021-07-15-memory-manager-moves-to-beta/MemoryManagerDiagram.svg "MemoryManagerDiagram")

In the Admission phase:

1. First, the **Kubelet** calls the **Topology Manager** `Admit()` method.
2. The **Topology Manager** invokes `GetTopologyHints()` on every hint provider (including the **Memory Manager**).
3. The **Memory Manager** calculates all possible NUMA alignment combinations for every container inside the pod and returns hints to the **Topology Manager**.
4. The **Topology Manager** calls `Allocate()` for every hint provider (including the **Memory Manager**).
5. The **Memory Manager** allocates the memory and updates the state according to the hint selected by the  **Topology Manager**.

In the Creation phase:

1. The **Kubelet** calls `PreCreateContainer()`.
2. The **Memory Manager** returns to **Kubelet** the calculated set of NUMA nodes where to allocate the memory for the container.
3. The **Kubelet** creates the container via CRI with the specification crafted by **Memory Manager**.

### Let's talk about the configuration

By default, the **Memory Manager** runs with the `None` policy, meaning it does nothing. To enable the **Memory Manager**, these **Kubelet** flags should be added:

- `--memory-manager-policy=Static`
- `--reserved-memory="<numaNodeID>:<resourceName>=<quantity>"`

It is quite clear what value should be set for the `--memory-manager-policy` parameter, but it is not fully clear how to configure the `--reserved-memory` parameter. In order to set it up correctly, you should follow the rules:

- The amount of reserved memory for the `memory` resource must be greater than zero.
- The amount of reserved memory for the resource type must be equal to [NodeAllocatable](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/) (`kube-reserved + system-reserved + eviction-hard`) for the resource. The topic is discussed and exemplified in [the official documentation](/docs/tasks/administer-cluster/memory-manager/).

![ReservedMemory
](/static/images/blog/2021-07-15-memory-manager-moves-to-beta/ReservedMemory.svg "ReservedMemory")

## Current gaps

While Beta release brings along enhancements and fixes, the **Memory Manager** still leaves gaps that could be fulfilled.

### Single versus Cross NUMA node allocation

The NUMA node cannot support both single and cross NUMA node allocations at the same time. In fact, when the container memory is pinned to two or more NUMA nodes, we cannot determine from which NUMA node the container will consume the memory.

Example (without the involvement of Memory Manager):

![SingleCrossNUMAAllocation
](/static/images/blog/2021-07-15-memory-manager-moves-to-beta/SingleCrossNUMAAllocation.svg "SingleCrossNUMAAllocation")

1. The `container1` started on the NUMA node 0 and requested *5Gi* of the memory, but it consumes only *3Gi* of the memory.
2. For container2, the memory request is set to 10Gi, but no single NUMA node can satisfy it.
3. The `container2` consumes *3.5Gi* of the memory from the NUMA node 0, but once the `container1` requests more memory, which is not available in fact, the kernel kills one of the containers resulting in the **Out-Of-Memory (OOM)** error.

To eliminate this kind of issue, the **Memory Manager** does not admit (rejects) the `container2` as long as a specific machine cannot be found in a cluster, i.e. a machine with two NUMA nodes but without a single NUMA node allocation overlapping in-between.

### Supports only Guaranteed pods

The **Memory Manager** cannot guarantee memory allocation for Burstable pods, even if the Burstable pod has equal memory `limits` and `requests` configured in the specification.

Let us assume a pair of Burstable pods as an example: `pod1` serves containers with equal memory `requests` and `limits`, and `pod2` serves containers only with the memory `requests` configured. We wish to guarantee memory allocation for the `pod1`, but both pods have the same **OOM** score assigned. So, once the operating system runs out of memory, the kernel can kill either `pod1` or `pod2` (despite the fact that `pod1` has equal `limits` and `requests`).

### Memory fragmentation

A sequence of containers deployed to a single machine can pose the fragmentation of the memory on NUMA nodes on this machine. Currently, no such a component exists in the kubelet ecosystem that could rebalance pods and sustain the correct fragmentation of memory.

## Further work

We are willing to enhance the current state of the **Memory Manager**, and therefore, we list the potential improvements.

### Making the Memory Manager allocation algorithm smarter

The current algorithm does not involve the distances between NUMA nodes in the allocation calculations. We could obtain higher performance if the **Memory Manager** can select a set of the closest NUMA nodes for cross NUMA node allocations.

### Reduction in the number of admission errors

The Kubernetes scheduler is not aware of the node's NUMA topology, which can contribute to a sequence of admission errors due to multiple pod rejections.

A [KEP]([https://https://github.com/kubernetes/enhancements/pull/2787) to address this issue is currently under review, and a prototype implementation is underway to get this feature ready in the near future.


## Conclusion

With the promotion of the **Memory Manager** to Beta in v1.22, we encourage everyone to deploy and trial the **Memory Manager**. Also, we look forward to your feedback. Still, while there are a number of gaps to be fulfilled, we plan to address them in the upcoming releases to provide you with these new features.

If you develop any ideas concerning additional enhancements or a desire for certain feature, please share your thoughts with us as the working group is open to suggestions.

We hope you have found this blog informative and helpful, but do not hesitate to share questions or comments.