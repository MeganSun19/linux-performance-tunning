RHEL KVM Performance Tuning Guide

This repository documents a set of practical performance-tuning techniques applied on KVM hypervisors running on RHEL-based systems.
The work originated from a real-world deployment of a large, distributed, microservices-based management cluster that is both storage-intensive and latency-sensitive.

During installation and runtime, several mandatory IOPS checks and storage throughput validations did not meet expected baselines.
To address these bottlenecks, we implemented a series of optimizations across the following areas:

1. CPU Optimization

Disable HyperThreading

CPU governor configuration (performance)

CPU isolation (isolcpus, nohz_full, rcu_nocbs)

CPU pinning for vCPUs and I/O threads

2. NUMA Optimization

NUMA-aligned CPU selection

NUMA-aware VM placement

Ensuring locality for memory, interrupts, and I/O paths

3. Storage & I/O Optimization

Using cache=none and io=native for virtio-based storage

Ensuring NUMA-aligned NVMe access and interrupt distribution

This repository summarizes the optimization steps, the rationale behind each method, and example commands/scripts used to improve storage and virtualization performance on RHEL KVM environments.