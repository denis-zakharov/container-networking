Kubernetes Services
===
When we have a constructed container network which covers pod-to-pod and pod-to-external
communication, we can build kubernetes services atop.

The Kubernetes Service provide a way to abstract the service endpoints behind a stable
service name in DNS which resolves to a virtual IP address.

Kubernetes Networking Model
---
There are 4 distinct networking problems to address:

- Highly-coupled container-to-container communications: this is solved by Pods and localhost communications.
- Pod-to-Pod communications (see the Container Networking document).
- Pod-to-Service communications: this is covered by services (the primary focus of this document).
- External-to-Service communications: this is covered by services (the primary focus of this document).

Kubernetes imposes the following fundamental requirements on any networking implementation (barring any intentional network segmentation policies):

- pods on a node can communicate with all pods on all nodes without NAT
- agents on a node (e.g. system daemons, kubelet) can communicate with all pods on that node
 
Note: For those platforms that support Pods running in the host network (e.g. Linux):

- pods in the host network of a node can communicate with all pods on all nodes without NAT



DNS round-robin? No
---
- Improper handing of cached DNS records or a runtime single lookup is no uncommon on a client side.
- Even if the TTL is handled properly, the low or zero TTLs on the DNS records could impose a high load on DNS.

There is a possibility to opt in DNS RR with 'headless' services (`ClusterIP: None`).

User space proxy mode (legacy)
---

![userspace proxy mode](https://d33wubrfki0l68.cloudfront.net/e351b830334b8622a700a8da6568cb081c464a9b/13020/images/docs/services-userspace-overview.svg)

kubeproxy respects `SessionAffinity` settings on a `Service` object and allows different
balancing modes (rr is default) but at the cost of switching between userspace and kernel mode.

iptables proxy mode
---

![iptables proxy mode](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg)

By default, kube-proxy in iptables mode chooses a backend at random.

Using iptables to handle traffic has a lower system overhead, because traffic is handled by Linux
netfilter without the need to switch between userspace and the kernel space.

If kube-proxy is running in iptables mode and the first Pod that's selected does not respond,
the connection fails (userspace proxy mode can retry, however this problem is largely solved by
readiness probes).

Cluster IP DNAT
```shell=
iptables -n -t nat -L KUBE-SERVICES | grep service-name
KUBE-MARK-MASQ
KUBE-SVC-XXX

# source is NOT pod network and dst is the cluster IP addr
KUBE-MARK-MASQ
MARK all (MARK or 0x4000)

# otherwise to KUBE-SVC-XXX chain (to service endpoints chains with probabilities)
KUBE-SEP-YY1
KUBE-SEP-YY2

# for each service endpoint
KUBE-MARK-MASQ if the source IP is the target endpoint IP mark it for masquerading (SNAT)
becase otherwise the pod will reply directly to itself while expecting to get an answer from
the cluster IP.
DNAT else: dnat to the target endpoint IP

# KUBE-POSTROUTING
if marked -> MASQUERADE (SNAT)
```

Using DNAT fanout for load balancing has several caveats. It has no feedback for the load of
a given backend, and will always map application-level queries on the same connection to the
same backend (e.g. gRPC).

IP Virtual Server proxy mode
---
This is the preferrable default mode now with a fallback to the iptables mode.

![IPVS](https://d33wubrfki0l68.cloudfront.net/2d3d2b521cf7f9ff83238218dac1c019c270b1ed/9ac5c/images/docs/services-ipvs-overview.svg)

IPVS provides different balancing modes, respects `SessionAffinity` settings.

IPVS configures backends through the Linux `netlink` interface (userspace-kernel IPC).


The IPVS proxy mode is based on netfilter hook function that is similar to iptables mode,
but uses a hash table as the underlying data structure and works in the kernel space.
That means kube-proxy in IPVS mode redirects traffic with lower latency than kube-proxy
in iptables mode, with much better performance when synchronising proxy rules. Compared to
the other proxy modes, IPVS mode also supports a higher throughput of network traffic.

**Resources**
- [ipvs proxy mode](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md)


Cilium CNI and eBPF
---
[eBPF](https://ebpf.io/) is a higly efficient sandboxed virtual machine
in the Linux kernel making the Linux kernel programmable at native
execution speed.

An eBPF program has a direct access to syscalls. In addition to scoket filtering, other
supported attach points in the kernel are:
- kprobes (dynamic tracing of internal kernel components)
- uprobes (userspace tracing)
- tracepoints (static kernel tracing with the programmed trace points in the kernel)
- perf_events (timed sampling of data and events)
- XDP (eXpress Data Path: direct access to a network drivers to act directly on packets)

Using eBPF, Cilium CNI plugin implements Kubernetes services without kube-proxy/netfilter/iptables.

Why?

iptables overhead:
- NAT and conntrack
- O(N) time complexity for traversing and inserting rules (linear scan of iptables chains)


Cilium CLI can be launched from the cilium pod (of the cilium daemon set). The CLI is scoped to the
corresponding node in this case.

```shell=
# overall check
cilium status

cilium service list
cilium policy list

# monitor deny policy
cilium monitor -t drop
cilium endpoint list
```


**Resources**
- [Services Networking Concepts](https://kubernetes.io/docs/concepts/services-networking/service/)
- [TGI Kubernetes: Troubleshooting Container Networking (Video)](https://youtu.be/IhbJ3ll4usI)
- [How to Make Linux Microservice-Aware with Cilium and eBPF (Video)](https://youtu.be/_Iq1xxNZOAo)
- [Liberating Kubernetes From Kube-proxy and Iptables - Martynas Pumputis, Cilium (Video)](https://youtu.be/bIRwSIwNHC0)
- [Liberating Kubernetes From Kube-proxy and Iptables (Slides)](https://sched.co/Uaam)
- [Networking and Kubernetes by James Strong and Vallery Lancey](https://learning.oreilly.com/library/view/networking-and-kubernetes/9781492081647/)