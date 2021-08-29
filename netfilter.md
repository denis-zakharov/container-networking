# Netfiler

Netfilter is a framework of kernel hooks.

A program registers to specific hooks.

A kernel calls that program on applicable packets.

Netfilter was created jointly with iptables, to separate kernel and userspace code.

## Netfilter Hooks

iptables directly maps its concept of *chains* to netfilter hooks.

![](https://i.imgur.com/9ResaH5.png)

Netfilter takes actions based on a value returned by a program that was triggered by the hook.
- **accept** process further
- **drop** discard and stop processing
- **queue** pass to a userspace program
- **stolen** do not execute other hooks, pass an ownership to a userspace program
- **repeat** re-enter the hook

## Conntrack

- firewall can distinguish between responses and arbitrary packets (one-way connect initiation)
- SNAT
- DNAT

Connection (*flows*) are identified as a tuple:

*src addr, src port, dst addr, dst port, L4 proto*

Hashmap (a tuple to a flow) size is configurable:
- /proc/sys/net/nf_conntrack_max flows
- /sys/module/nf_conntrack/parameters/hashsize

## Conntrack states

| State    | Description | Example |
| -------- | ----------- | ------- |
| NEW      | A valid packet is sent with no response seen | TCP SYN Received  |
| ESTABLISHED | Packets observed in both directions | SYN, SYN/ACK sent |
| RELATED | Additional connection where metadata indicates to original connection | FTP client with ESTABLISHED connection, opens additional data connections |
| INVALID | No match to a state | RST received with no prior connection |

---

## conntrack -L

```
tcp      6 431999 ESTABLISHED 
src=10.0.0.2 dst=10.0.0.1 sport=22 dport=49431
src=10.0.0.1 dst=10.0.0.2 sport=49431 dport=22 [ASSURED] mark=0 use=1

<protocol> <protocol number> <flow TTL> [flow state>]
<source ip> <dest ip> <source port> <dest port>
<expected return packet>
```

# iptables

## iptables tables

|Table	| Purpose | Chains |
| ----- | ------- | ------ |
| Raw | The raw table allows for packet mutation, before connection tracking and other tables are handled. Its most common use is to disable connection tracking for some packets. | PREROUTING, OUTPUT |
|Mangle | The mangle table can perform general-purpose editing of packet headers, but it is not intended for NAT. It can also “mark” the packet with iptables-only metadata. | all |
|NAT    | The NAT table is used to modify the source or destination IP addresses. | all except FORWARD |
|Filter | The filter table handles acceptance and rejection of packets. | all except PREROUTING and POSTROUTING |
| Security | SELinux uses the security table for packet handling. It is not applicable on a machine that is not using SELinux. | * |


## iptables chains

*Chains* are a list of rules. The rules in each chain are evaluated until a terminating target.

The built-in top-level chains are *directly mapped to the netfilter hooks*.

Although tables have chains the order of execution is 'chains then tables'.

![](https://i.imgur.com/DvOjlJi.png)


## Rules

Rule = a match condition + a target (action)

The examples of conditions:
- -i eth0 (enters via the interface)
- -o eth0 (exits via the interface)
- -p proto
- -s src_addr
- -d dst_addr
- -m state --state NEW,ESTABLISHED,RELATED,INVALID (conntrack)
- -m statistics --mode random --probability 0.5


The example of targets:
- DROP, REJECT are finally terminating
- ACCEPT, RETURN are terminating **within their chain**

## Targets

Targets and tables where they can be applied.

- AUDIT - all - record data about accepted, dropped or rejected packets.
- ACCEPT - filter - no modify, continue
- DNAT - nat - modify dst addr
- DROP - filter - discard
- JUMP - all - jump into another chain, then continue from the parent (like a subroutine call).
- LOG - all - contents
- MARK - all - internal label
- MASQUERADE - nat - SNAT with an address of the specified interface (dynamic)
- REJECT - filter - discard and notify
- RETURN - all - stop in a chain
- SNAT - nat - a static source NAT (a fixed address)

## iptables commands

The default table is `filter`.

```shell=
# list all rules in the selected table
# (all chains if not explicitly given with a chane name)
iptables -t <table> -L [chain] --line-numbers

# append, check, delete, replace _rules_
iptables [-t table] {-A|-C|-D|-R} chain rule-spec

# insert a new rule into a line 2 moving down an existing rule
iptables -t nat -I POSTROUTING 2 <rule-spec>
```

Other flags:
* -n no DNS lookup
* -S [chain], --list-rules [chain] print all rules in the selected chain or all chains.
  A short form of `-L`; `iptables-save` format.
* -F [chain], --flush [chain]
* -X [chain], --delete-chain [chain] delete the user-defined chain
* -E, --rename-chain [old] [new_name]

## subchains
Subchains are set of custom rules grouped together for reuse (similar to a function).

Filter chains end in a default action, e.g. `DROP` the packet if no prior target matched.

Chains will default to `ACCEPT` if no default is specified.

`iptables -P <chain> <target>` sets the default target for a chain.

```shell=
# Create incoming-ssh chain.
$ iptables -N incoming-ssh

# Allow packets from specific IPs.
$ iptables -A incoming-ssh -s 10.0.0.1 -j ACCEPT
$ iptables -A incoming-ssh -s 10.0.0.2 -j ACCEPT

# Log the packet.
$ iptables -A incoming-ssh -j LOG --log-level info --log-prefix "ssh-failure "

# Drop packets from all other IPs.
$ iptables -A incoming-ssh -j DROP

# Evaluate the incoming-ssh chain, if the packet is an 
# inbound TCP packet addressed to port 22.
$ iptables -A INPUT -p tcp --dport 22 -j incoming-ssh
```

**Resources**
- [iptables tutorial](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html)

# IP Virtual Server

![IPVS](https://i.imgur.com/4qIxug1.png)

IPVS supports packet forwarding modes:
- NAT
- DR (direct routing in L2)
- IP-in-IP tunneling

```shell=
# {-A,-E,-D} add, edit, delete a VIP
# {-a,-e,-d} add, edit, delete a backend IP
ipvsadm -A -t 1.1.1.1:80 -s lc
ipvsadm -a -t 1.1.1.1:80 -r 2.2.2.2 -m -w 100
ipvsadm -a -t 1.1.1.1:80 -r 3.3.3.3 -m -w 100

# list configuration
ipvsadm -L --stats
```