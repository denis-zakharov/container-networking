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
- a container-to-container connectivity on the same host
- a container to a network
- a network to a container via a port publishing

Note. Here 'a network' is an undelying physical network with
a host-to-host connectivity.

The overlay POD network subnet is 192.168.0.0/16.
Allocated subnets per two hosts: 192.168.0.0/24 and 192.168.1.0/24.

![Single Host](https://i.imgur.com/gK0xwyW.png)


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
Assign IP addresses to containers.
```shell=
sudo ip link set ceth0 netns netns0
sudo ip link set ceth1 netns netns1

# netns0
sudo nsenter --net=/var/run/netns/netns0
ip link set lo up
ip link set ceth0 up
ip addr add 192.168.0.10/16 dev ceth0
exit

# netns1
sudo nsenter --net=/var/run/netns/netns1
ip link set lo up
ip link set ceth1 up
ip addr add 192.168.0.11/16 dev ceth1
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

**C-to-C L2 connectivity**
```shell=
sudo ip netns exec netns0 ping -c2 192.168.0.11
sudo ip netns exec netns1 ip neigh  # ARP cache
```

**C-to-root L3 connectivity**
```shell=
sudo ip addr add 192.168.0.1/16 brd + dev cbr0

# default route via cbr0
sudo ip netns exec netns0 ip route add default via 192.168.0.1
sudo ip netns exec netns1 ip route add default via 192.168.0.1
sudo ip netns exec netns0 ping -c2 10.198.16.144
```

**C-to-network L3 connectivity**
Enable IP-forwarding and traffic filtering over bridged networks
on the host.

```shell=
# enable ip forwarding for NAT
sudo bash -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'
# enable bridge traffic interseption
sudo modprobe br_netfilter
sudo bash -c 'echo 1 > /proc/sys/net/bridge/bridge-nf-call-arptables'
sudo bash -c 'echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables'
```

Enable source NAT for outbound traffic from a container subnet.
```shell=
# source NAT container subnet for outbound traffic
# filter out C-to-C traffic i.e. the out interface is not the cbr0
sudo iptables -t nat -A POSTROUTING \
    -s 192.168.0.0/16 ! -o cbr0 \
    -j MASQUERADE
```

We use the 'default - allow' strategy. In the real world,
container runtimes use the 'default - deny' strategy and enable
routing only for known paths.

**Net-to-C L3 connectivity**
Publishing a port in a root namespace allows to forward
inbound host traffic to a concrete container using
a source NAT.

For example, we publish netns0:5000 as rootns:5000.
```shell=
# External traffic
sudo iptables -t nat -A PREROUTING \
    -d 10.198.16.144 \
    -p tcp -m tcp --dport 5000
    -j DNAT --to-destination 192.168.0.10:5000
    
# Local traffic (since it does not pass the PREROUTING chain)
sudo iptables -t nat -A OUPUT \
    -d 10.198.16.144 \
    -p tcp -m tcp --dport 5000
    -j DNAT --to-destination 192.168.0.10:5000
```

**Resources**
- https://iximiuz.com/en/posts/container-networking-is-simple/
- https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking


VXLAN Overlay Network
---
VXLAN is a tunelling protocol that allows to span L2 network
over several underlying physical networks
(essentialy creating a giant switch).

The ethernet frame (starting with a special VXLAN header)
from the overlay container network are
encapsulated into a UPD packet.

Almost all popular CNIs support VXLAN as an overlay backend:
- flannel
- weave
- calico (in addition to default IP-IP tunelling mode)

CNI plugins collect a network topology (from K8s API or
by peering as WeaveNet), configure VXLAN interfaces on
hosts and allocate IP subnet for pods on each host.

The WeaveNet uses a more sophisticated software L2-L3 switch
(Open vSwitch) instead of an old Linux bridge but essentially
it is still a VXLAN tunnel under the hood.

![Multihost VXLAN Network](https://i.imgur.com/b46pyLK.png)


On host0:
```shell=
sudo ip link add vxlan0 \
    type vxlan id 1 \
    remote 10.198.16.227 \
    dstport 4789 dev ens160
sudo ip link set vxlan0 up
sudo ip link set vxlan0 master cbr0
```

On host1:
```shell=
sudo ip link add vxlan0 \
    type vxlan id 1 \
    remote 10.198.16.144 \
    dstport 4789 dev ens160
sudo ip link set vxlan0 up
sudo ip link set vxlan0 master cbr0
```

Forwarding table
```shell=
bridge fdb show
```

Verify VXLAN
```shell=
# On host1
ping -c2 192.168.0.10
sudo ip netns exec netns0 ping -c2 192.168.0.10

# On host0
sudo tcpdump -i ens160 -XX port 4789
```

[Python multicast example on UDP sockets](https://huichen-cs.github.io/course/CISC7334X/20FA/lecture/pymcast/)
```shell=
./mcastrecv.py <nic_ip> 234.3.2.1 50001
./mcastsend.py <nic_ip> 234.3.2.1 50001 'Test message'
```

IP-in-IP Overlay Network
---
On host0:
```shell=
sudo ip link add name ipip1 type ipip local 10.198.16.144 remote 10.198.16.227
sudo ip link set ipip1 up
sudo ip addr add 172.20.255.1/30 dev ipip1
sudo ip route add 172.20.1.0/24 via 172.20.255.2 dev ipip1
```

On host1:
```shell=
sudo ip link add name ipip1 type ipip local 10.198.16.227 remote 10.198.16.144
sudo ip link set ipip1 up
sudo ip addr add 172.20.255.2/30 dev ipip1
sudo ip route add 172.20.0.0/24 via 172.20.255.1 dev ipip1
```

**Resources**
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-ip-tunnels_configuring-and-managing-networking
- https://itnext.io/kubernetes-network-deep-dive-7492341e0ab5
- [Packet Walks in K8s](https://github.com/jayakody/ons-2019)
- [Service Mesh: todo and not todo](https://github.com/jayakody/ones-2020)
- [Pod-to-Pod: Cisco blog](https://blogs.cisco.com/developer/kubernetes-intro-2)

Routed Overlay Network in an L2 segment
---
TODO

Kubernetes Services
---
TODO: eBPF vs IPTables vs IP Virtual Server