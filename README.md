# How Container Networking is Implemented in Linux

The three building blocks:

- namespace
- virtual interface
- software bridge

### Creating a namespace

List all namespaces on the machine:

```bash

sudo ip netns list
```

Create 2 new namespaces

```bash

sudo ip netns exec add ns1 ns2
```

`ip netns exec` can be simplified into `ip -n`, going forward that is the syntax I will use.<br>
Use the command to list namespaces to confirm they have been created.

### Create a bridge

A bridge is a link device of type `bridge`. To create one:

```bash

sudo ip link add test-bridge type bridge
```

Assign the bridge device an ip address:

```bash

sudo ip addr add 192.168.100.168/24 dev test-bridge
```

Bring up the bridge:

```bash

sudo ip link set test-bridge up
```

### Create virtual interfaces

Create 4 virtual interfaces. Each namespace has one virtual interface inside it, communicating with its peer in the host namespace. The virtual interfaces in the host namespace will be connected to the bridge create above. Include a diagram of the architecture.

Create the 4 virtual interfaces

```bash

sudo ip link add veth-ns1 type veth peer name veth-ns1-br
sudo ip link add veth-ns2 type veth peer name veth-ns2-br
```

Move the two virtual interfaces into their namespaces:

```bash

sudo ip link set veth-ns1 netns ns1
sudo ip link set veth-ns2 netns ns2
```

Add IP addresses to the virtual interfaces and bring them up:

```bash

sudo ip -n ns1 addr add 192.168.100.170/24 dev veth-ns1 && sudo ip -n ns1 link set veth-ns1 up
sudo ip -n ns2 addr add 192.168.100.171/24 dev veth-ns1 && sudo ip -n ns2 link set veth-ns2 up
sudo ip addr add 192.168.100.172/24 dev veth-ns1-br && sudo ip link set veth-ns1-br up
sudo ip addr add 192.168.100.173/24 dev veth-ns2-br  && sudo ip link set veth-ns2-br up
```

Ping the virtual interfaces pairs to confirm they have connectivity

```bash

sudo ip -n ns1 ping 192.168.100.172 # ping veth-ns1-br from inside ns1
ping 192.168.100.170 # ping veth-ns1 from veth-ns1-br

sudo ip -n ns2 ping 192.168.100.173 # ping veth-ns2-br from inside ns2
ping 192.168.100.171 # ping veth-ns2 from veth-ns2-br
```

Attach the two veths to the bridge:

```bash

sudo ip link set veth-ns-br1 master test-bridge
sudo ip link set veth-ns-br2 master test-bridge
```

Try pinging ns1 from ns2 and vice-versa

Add NAT for the two veths in ns1 and ns2:

```bash

sudo iptables -t nat -A POSTROUTING -s 192.168.100.0/24 ! -o test-bridge -j MASQUARADE
```

Download content from a site on the internet from any of the namespaces

Check the connection tracking stats for the namespaces and NAT
