Container Netwroking
===

Basic principles how to construct an overlay network for containers.

Containers are isolated and restricted linux processes with a set control on compute resources.

Isolation is provided through Linux kernel namespaces:
- process
- user
- IPC
- net
- UTC (hostnames)
- mnt

A control on compute resources is implemented through CGroups:
- cpu
- memory
- net
- disk I/O

To construct a container networks it is enough to use a `net` namespace as a container
simulation. Each `net` namespace provides an individual isolated network stack
(net devices, routing tables, firewall rules).

**Resources**
- https://www.nginx.com/blog/what-are-namespaces-cgroups-how-do-they-work/
- [CGroup v1](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/index.html)
as of 2021 still dominating; worth to read to learn concepts.
- [CGroup v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
new CGroup API since Linux kernel 3.16 (Aug 3, 2014).
- [CGroup v2 changes overview](https://medium.com/nttlabs/cgroup-v2-596d035be4d7)

Single-Host Net
---
We start with a single host network which provides connectivity:
- container-to-container on the same host
- host to a network
- container to a network


```

```