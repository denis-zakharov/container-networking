Kubernetes Services
===
A short overview of Kubernetes service implementations.

DNS round-robin? No
---
- Improper handing of cached DNS records or a runtime single lookup is no uncommon on a client side.
- Even if the TTL is handled properly, the low or zero TTLs on the DNS records could impose a high load on DNS.

User space proxy mode
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

IP Virtual Server proxy mode
---

![IPVS](https://i.imgur.com/4qIxug1.png)

IPVS provides different balancing modes, respects `SessionAffinity` settings.

IPVS configures backends through the Linux `netlink` interface (userspace-kernel IPC)


The IPVS proxy mode is based on netfilter hook function that is similar to iptables mode,
but uses a hash table as the underlying data structure and works in the kernel space.
That means kube-proxy in IPVS mode redirects traffic with lower latency than kube-proxy
in iptables mode, with much better performance when synchronising proxy rules. Compared to
the other proxy modes, IPVS mode also supports a higher throughput of network traffic.


Cilium CNI and eBPF
---
eBPF is a higly efficient sandboxed virtual machine in the Linux kernel making the Linux kernel
programmable at native execution speed.

Using eBPF, Cilium CNI plugin implements Kubernetes services without kube-proxy/netfilter/iptables.

Why?

iptables overhead:
- NAT and conntrack
- O(N) time complexity for traversing and inserting rules (linear scan of iptables chains)


**Resources**
- [Services Networking Concepts](https://kubernetes.io/docs/concepts/services-networking/service/)
- [TGI Kubernetes: Troubleshooting Container Networking (Video)](https://youtu.be/IhbJ3ll4usI)
- [How to Make Linux Microservice-Aware with Cilium and eBPF (Video)](https://youtu.be/_Iq1xxNZOAo)
- [Liberating Kubernetes From Kube-proxy and Iptables - Martynas Pumputis, Cilium (Video)](https://youtu.be/bIRwSIwNHC0)