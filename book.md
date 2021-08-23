Container Networking
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

![Single Host](https://i.imgur.com/IvD1UhQ.png)

Create network namespaces
```shell
sudo ip netns add netns0
sudo ip netns add netns1
ip netns ls
```

Inspect a net stack in a newly created namespace
```shell
sudo nsenter --net=/var/run/netns/netns0 ./inspect-net-stack.sh
```

Create virtual ethernet devices (pairs) in the root net ns.
```shell
sudo ip link add veth0 type veth peer name ceth0
sudo ip link add veth1 type veth peer name ceth1
```

Move container ethernet devices to their namespaces.
Assign IP addresses.
```shell=
sudo ip link set ceth0 netns netns0
sudo ip link set ceth1 netns netns1

# netns0
sudo nsenter --net=/var/run/netns/netns0
ip link set lo up
ip link set ceth0 up
ip addr add 192.168.0.10/24 dev ceth0
exit

# netns1
sudo nsenter --net=/var/run/netns/netns1
ip link set lo up
ip link set ceth1 up
ip addr add 192.168.0.11/24 dev ceth1
exit
```

Create a bridge to connect the namespaces.
Attach containers to the bridge
```shell=
sudo ip link add cbr0 type bridge
sudo ip link set cbr0 up

sudo ip link set veth0 master cbr0
sudo ip link set veth1 master cbr0
sudo ip link set veth0 up
sudo ip link set veth1 up
```

C-to-C L2 connectivity
```shell=
sudo ip netns exec netns0 ping -c2 192.168.0.11
sudo ip netns exec netns1 ip neigh  # ARP cache
```
