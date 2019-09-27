# Practical intro to Linux network namespaces

## IP protocol: short recap

* `Node`: anything which has an address and can send/receive IP packets

* IP protocol does not depend on the physical nature of nodes and traffic carrier [RFC 1149](https://tools.ietf.org/html/rfc1149)

* `Node` != `machine`, `Node` != `network interface`

* Thus it's possible to have multiple IP nodes per a machine/interface

* Linux network namespaces implement this idea


## Creating network namespaces

### Volatile/anonymous network namespace

Such a namespace gets automatically destroyed after the last process using it
exits **and** the last network link gets detached/destroyed

```bash
unshare -U -r -n /bin/bash
```

`-n` creates a new network namespace
`-U` creates a new UID namespace
`-r` maps the calling user to `root` in the newly created UID namespace

### Persistent/named network namespace

```bash
sudo ip netns add vns1
```

### Connecting network namespaces

#### Isolated node

```bash
$ unshare -U -r -n /bin/bash
# ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

That's an equivalent of an isolated host. Use cases:

* Running functional tests (which listen the same port) in parallel
* Checking if the build can run in an isolated environment
* Preventing an unstrusted program from sending anything to the network
* Testing if a program handles connectivity problems correctly


#### Two directly linked nodes

* Create the 1st namespace

```bash
$ unshare -U -r -n /bin/bash
```

* Create the 2nd namespace

```bash
$ unshare -U -r -n /bin/bash
# echo $$
20061
```

* Link them with  *veth pair*. In the 1st namespace run

```bash
# ip link add name veth0 type veth peer name veth1 netns 20061
```

* List network interfaces in the 1st namespace

```bash
# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth0@if2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ba:35:7b:d2:7a:d0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

* List network interfaces in the 2nd namespace

```bash
# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth1@if2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 6e:0c:1f:0e:ec:59 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

* Assign IP address and up the link, 1st namespace

```bash
# ip link set up dev veth0
# ip addr add 10.0.2.1/24 brd 10.0.2.255 dev veth0
# ip link set up dev lo
```

* Assign IP address and up the link, 2nd namespace

```bash
# ip link set up dev veth1
# ip addr add 10.0.2.2/24 brd 10.0.2.255 dev veth1
# ip link set up dev lo
```

* Check if namespaces can reach each other. In the 1st namespace

```bash
# ping 10.0.2.2
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
64 bytes from 10.0.2.2: icmp_seq=1 ttl=64 time=0.045 ms
64 bytes from 10.0.2.2: icmp_seq=2 ttl=64 time=0.038 ms
64 bytes from 10.0.2.2: icmp_seq=3 ttl=64 time=0.050 ms
^C
--- 10.0.2.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2026ms
rtt min/avg/max/mdev = 0.038/0.044/0.050/0.007 ms
```

* Check if the root namespace can reach newly created ones

```bash
$ ping 10.0.2.2
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
^C
--- 10.0.2.2 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3058ms
```

```bash
$ ping 10.0.2.1
PING 10.0.2.1 (10.0.2.1) 56(84) bytes of data.
^C
--- 10.0.2.1 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3066ms
```


### Several connected nodes

* Create a *bridge* in the main namespace

```bash
$ sudo brctl addbr virbr100500
$ sudo ip link set up dev virbr100500
```

* Create 3 network namespaces

```bash
$ unshare -U -r -n /bin/bash
# echo $$
20371
```

```bash
$ unshare -U -r -n /bin/bash
# echo $$
20404
```

```bash
$ unshare -U -r -n /bin/bash
# echo $$
20430
```

* Link each namespace to the bridge with veth pair

```bash
$ sudo ip link add name veth0 type veth peer name veth1 netns 20371
$ sudo brctl addif virbr100500 veth0
$ sudo ip link set up veth0
$ sudo ip link add name veth2 type veth peer name veth3 netns 20404
$ sudo brctl addif virbr100500 veth2
$ sudo ip link set up veth2
$ sudo ip link add name veth4 type veth peer name veth5 netns 20430
$ sudo brctl addif virbr100500 veth4
$ sudo ip link set up veth4
```

* Check if the network link is available in each namespace

```bash
# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth1@if7: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether a6:c0:c3:77:f3:e6 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

```bash
# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth3@if8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ce:2d:99:4a:74:37 brd ff:ff:ff:ff:ff:ff link-netnsid 0

```

```bash
# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth5@if9: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 0a:c5:b9:02:97:cd brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

* Assign IP addresses, bring up interfaces

```bash
# ip addr add 10.0.2.1/24 brd 10.0.2.255 dev veth1
# ip link set up dev veth1
```

```bash
# ip addr add 10.0.2.2/24 brd 10.0.2.255 dev veth3
# ip link set up dev veth3
```

```bash
# ip addr add 10.0.2.3/24 brd 10.0.2.255 dev veth5
# ip link set up dev veth5
```

* Check if namespaces can reach each other

```bash
# ping -c 5 10.0.2.2
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
64 bytes from 10.0.2.2: icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from 10.0.2.2: icmp_seq=2 ttl=64 time=0.054 ms
64 bytes from 10.0.2.2: icmp_seq=3 ttl=64 time=0.064 ms
64 bytes from 10.0.2.2: icmp_seq=4 ttl=64 time=0.056 ms
64 bytes from 10.0.2.2: icmp_seq=5 ttl=64 time=0.048 ms

--- 10.0.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4075ms
rtt min/avg/max/mdev = 0.048/0.054/0.064/0.008 ms

# ping -c 5 10.0.2.3
PING 10.0.2.3 (10.0.2.3) 56(84) bytes of data.
64 bytes from 10.0.2.3: icmp_seq=1 ttl=64 time=0.044 ms
64 bytes from 10.0.2.3: icmp_seq=2 ttl=64 time=0.037 ms
64 bytes from 10.0.2.3: icmp_seq=3 ttl=64 time=0.054 ms
64 bytes from 10.0.2.3: icmp_seq=4 ttl=64 time=0.051 ms
64 bytes from 10.0.2.3: icmp_seq=5 ttl=64 time=0.053 ms

--- 10.0.2.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4083ms
rtt min/avg/max/mdev = 0.037/0.047/0.054/0.010 ms
```

* Check if the global namespace can reach any of newly created ones

```bash
$ for addr in 10.0.2.1 10.0.2.2 10.0.2.3; do ping -c 5 $addr; done
PING 10.0.2.1 (10.0.2.1) 56(84) bytes of data.

--- 10.0.2.1 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4098ms

PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.

--- 10.0.2.2 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4076ms

PING 10.0.2.3 (10.0.2.3) 56(84) bytes of data.

--- 10.0.2.3 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4076ms
```

#### Connecting namespaces to the Internet

* Configure NAT via the bridge

```bash
$ sudo sysctl -w net.ipv4.ip_forward=1
$ sudo ip addr add 10.0.2.254/24 brd 10.0.2.255 dev virbr100500
$ sudo iptables -A FORWARD -i virbr100500 -j ACCEPT
$ sudo iptables -t nat -A POSTROUTING -o $OUTPUT_IFACE -s 10.0.2.0/24 -j MASQUERADE
```

* Set the bridge as the default route in namespaces

```bash
# ip route add default via 10.0.2.254 dev veth1
```

```bash
# ip route add default via 10.0.2.254 dev veth3
```

```bash
# ip route add default via 10.0.2.254 dev veth5
```

* Check the connectivity

```bash
# ping -c 5 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=45 time=29.2 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=45 time=29.3 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=45 time=29.1 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=45 time=29.0 ms
64 bytes from 8.8.8.8: icmp_seq=5 ttl=45 time=30.0 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4003ms
rtt min/avg/max/mdev = 29.092/29.369/30.033/0.389 ms
```
