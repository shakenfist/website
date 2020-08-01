# Networking Explained

Shaken Fist networking is complicated. Its actually less complicated that OpenStack Neutron networking, and its about as simple as we can get away with, but in order to allow virtual networks to use overlapping network ranges we are forced to do some vaguely complicated things with network namespaces. This document attempts to incrementally describe how Shaken Fist networking works, so that I can remember later.

## VXLAN

Shaken Fist networking is based on a VXLAN mesh. VXLAN is like a successor to VLANs, except that you can have 1.6 million virtual networks, it doesn't use an IP header field to divide the networks up, and it is transported inside UDP packets between the members of the mesh. Normally VXLAN meshes are implemented using multicast UDP, but that doesn't work in public clouds where Shaken Fist was born, so we instead use unicast meshes that we lovingly hand maintain.

## Our worked examples

For this document, we will assume there are three Shaken Fist nodes, named sf-1, sf-2, and sf-3. Its a total coincidence that this is the default size for the installer ansible at the time of writing and the exact size of all of the production clusters we are aware of. sf-1 is configured as the "network node", which is just a hypervisor like every other node, except that it is also where packets to and from the virtual networks route in and out of the mesh.

## The simplest case: a virtual network with no DHCP and no NAT

Let's assume you want a new virtual network with no network services. Its just two instances talking to each other.

The basic flow is like this -- you create a virtual network. We allocate you a VXLAN network id (called the vxid in various places in the code):

```
sf-1 # sf-client network create 192.168.0.0/24 demonet --no-dhcp --no-nat
uuid        : 41f41085-b6d1-4c47-83ea-c8065bd54292
name        : demonet
vxlan id    : 2
netblock    : 192.168.0.0/24
provide dhcp: False
provide nat : False
namespace   : system
state       : initial

Metadata:
```

So in this case we were allocated VXLAN id 2, and have a network UUID of 41f41085-b6d1-4c47-83ea-c8065bd54292. The state of the network is "initial" as it has not been created anywhere yet. If you wait a few seconds, you'll see it transition to a "created" state. You can see the new state with a show command:

```
sf-1  sf-client network show 41f41085-b6d1-4c47-83ea-c8065bd54292
uuid        : 41f41085-b6d1-4c47-83ea-c8065bd54292
name        : demonet
vxlan id    : 2
netblock    : 192.168.0.0/24
provide dhcp: False
provide nat : False
namespace   : system
state       : created

Metadata:
```

And you can see the steps we went through to create the network in the events listing:

```
sf-1 # sf-client network events 41f41085-b6d1-4c47-83ea-c8065bd54292
+----------------------------+------+------------------------+------------+----------------------+-----------+
|         timestamp          | node |       operation        |   phase    |       duration       |  message  |
+----------------------------+------+------------------------+------------+----------------------+-----------+
| 2020-07-31 23:39:52.771882 | sf-1 |          api           |   create   |         None         |    None   |
| 2020-07-31 23:39:52.810563 | sf-1 | create vxlan interface |   start    |         None         |    None   |
| 2020-07-31 23:39:52.834701 | sf-1 | create vxlan interface |   finish   | 0.023019790649414062 |    None   |
| 2020-07-31 23:39:52.851094 | sf-1 |  create vxlan bridge   |   start    |         None         |    None   |
| 2020-07-31 23:39:52.921939 | sf-1 |  create vxlan bridge   |   finish   | 0.07030558586120605  |    None   |
| 2020-07-31 23:39:52.928207 | sf-1 |      create netns      |   start    |         None         |    None   |
| 2020-07-31 23:39:53.013959 | sf-1 |      create netns      |   finish   |  0.0834047794342041  |    None   |
| 2020-07-31 23:39:53.032187 | sf-1 |   create router veth   |   start    |         None         |    None   |
| 2020-07-31 23:39:53.257493 | sf-1 |   create router veth   |   finish   | 0.22433686256408691  |    None   |
| 2020-07-31 23:39:53.274471 | sf-1 |  create physical veth  |   start    |         None         |    None   |
| 2020-07-31 23:39:53.369914 | sf-1 |  create physical veth  |   finish   | 0.09386920928955078  |    None   |
| 2020-07-31 23:39:53.404596 | sf-1 |   add mesh elements    |    None    |         None         | 10.2.1.11 |
| 2020-07-31 23:39:53.409215 | sf-1 |      update dhcp       |   start    |         None         |    None   |
| 2020-07-31 23:39:53.497732 | sf-1 |      update dhcp       |   finish   | 0.08712530136108398  |    None   |
| 2020-07-31 23:39:53.518700 | sf-1 |          api           |  created   |         None         |    None   |
| 2020-07-31 23:40:41.118869 | sf-1 |          api           | get events |         None         |    None   |
+----------------------------+------+------------------------+------------+----------------------+-----------+
```

You can see here that the network node (sf-1) has created some network elements, and an IP (10.2.1.11) has been added to the mesh. That IP is sf-1, and its part of the network node being joined to the mesh. If we look on sf-1, we should now have a VXLAN interface and bridge.

```
sf-1 # ip addr show vxlan-2
257: vxlan-2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc noqueue master br-vxlan-2 state UNKNOWN group default qlen 1000
    link/ether c2:8a:fb:94:2b:94 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::c08a:fbff:fe94:2b94/64 scope link 
       valid_lft forever preferred_lft forever

sf-1 # ip addr show br-vxlan-2
258: br-vxlan-2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 92:12:c3:6b:0f:f3 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::9012:c3ff:fe6b:ff3/64 scope link 
       valid_lft forever preferred_lft forever
```

The vxlan-2 interface is the VXLAN mesh, and the br-vxlan-2 bridge is how VMs and veths will connect to the mesh on this local machine. Its important to note that MTU matters here. The MTU for the mesh network is 1500 bytes, and most client VMs will default to that as well. Therefore the underlying network needs to have a MTU greater than that. We default to an MTU of 9000 bytes in our installs, but 1550 would in fact be sufficient in this case. You can see this in the MTU for vxlan-2, which is our 9000 byte underlying MTU, with 50 bytes deducted for the VXLAN encapsulation.

We can also ask the mesh for its current state:

```
sf-1 # bridge fdb show brport vxlan-2
c2:8a:fb:94:2b:94 master br-vxlan-2 permanent
c2:8a:fb:94:2b:94 vlan 1 master br-vxlan-2 permanent
00:00:00:00:00:00 dst 10.2.1.11 self permanent
92:12:c3:6b:0f:f3 dst 127.0.0.1 self 
1e:f7:33:ce:c5:d2 dst 127.0.0.1 self 
c2:8a:fb:94:2b:94 dst 127.0.0.1 self
```

The current members of the mesh are:

* c2:8a:fb:94:2b:94: this is the mac address for vxlan-2. It appears three times for reasons.
* 00:00:00:00:00:00 dst 10.2.1.11: this is a mesh entry for the node with IP 10.2.1.11 (sf-1)
* 92:12:c3:6b:0f:f3: this is the outside mac address of a veth between br-vxlan-2 and a network namespace on sf-1 
* 1e:f7:33:ce:c5:d2: this is the inside mac address of that veth

What is this network namespace? Well, Shaken Fist needs to create a network namespace to contain routing, NAT, and DHCP for the virtual network. It's actually not strictly required in this simplest case, but we always create it. It is named for the UUID of the virtual network:

```
sf-1 # ip netns exec 41f41085-b6d1-4c47-83ea-c8065bd54292 ip addr list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
259: veth-2-i@if260: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 1e:f7:33:ce:c5:d2 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.0.1/24 scope global veth-2-i
       valid_lft forever preferred_lft forever
    inet6 fe80::1cf7:33ff:fece:c5d2/64 scope link 
       valid_lft forever preferred_lft forever
261: phy-2-i@if262: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether ae:0d:cb:4a:a1:3e brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

The veth between the VXLAN mesh and this namespace is named veth-2-i (the interface inside the network namespace) and veth-2-o (the interface outside the network namespace). There is another veth named phy-2-i and phy-2-o, which is a link between the namespace and the outside world, but we'll talk about that more when we enable NAT. For those who are new to veths, think of them like patch cables -- so what we have here is a VXLAN mesh, which is patched into a network namespace, which is in turn patched into the outside world.

We also do some things with iptables, especially around NAT. Here's the current state of iptables in the network namespace:

```
sf-1 # ip netns exec 41f41085-b6d1-4c47-83ea-c8065bd54292 iptables -L -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination 
```

That's empty for now because we're not doing any NAT yet, but watch this space. Next let's now start an instance on sf-2. This instance can't use DHCP to get an address because we have that disabled for this network.

```
sf-1 # sf-client instance create inst-on-sf-2 1 1024 -d 8@cirros -n 41f41085-b6d1-4c47-83ea-c8065bd54292 -p sf-2
uuid        : 0d7fc92d-9577-4d17-97af-1a7ff3bdb157
name        : inst-on-sf-2
namespace   : system
cpus        : 1
memory      : 1024
disk spec   : [{'base': 'cirros', 'bus': None, 'size': 8, 'type': 'disk'}]
node        : sf-2
power state : on
state       : created
console port: 44150
vdi port    : 44787

ssh key     : None
user data   : None

Metadata:

Interfaces:

    uuid    : c051356d-58fe-439b-8a6b-e772858298b0
    network : 41f41085-b6d1-4c47-83ea-c8065bd54292
    macaddr : 00:00:00:e0:b7:9e
    order   : 0
    ipv4    : 192.168.0.100
    floating: None
```

You can see that our instance (inst-on-sf-2) has been placed on sf-2 because we asked nicely (the -p is a placement option to the command), and has been allocated an IP (192.168.0.100). The virtual network still allocates IPs, even if DHCP is disabled. It has also been allocated a MAC address (00:00:00:e0:b7:9e). What is the state of the mesh on the network node now?

```
sf-1 # bridge fdb show brport vxlan-2
c2:8a:fb:94:2b:94 master br-vxlan-2 permanent
c2:8a:fb:94:2b:94 vlan 1 master br-vxlan-2 permanent
00:00:00:00:00:00 dst 10.2.1.11 self permanent
00:00:00:00:00:00 dst 10.2.1.12 self permanent
00:00:00:e0:b7:9e dst 10.2.1.12 self
92:12:c3:6b:0f:f3 dst 127.0.0.1 self 
62:74:69:88:5b:53 dst 10.2.1.12 self 
1e:f7:33:ce:c5:d2 dst 127.0.0.1 self 
c2:8a:fb:94:2b:94 dst 127.0.0.1 self 
```

The following entries there are new:

```
00:00:00:00:00:00 dst 10.2.1.12 self permanent
00:00:00:e0:b7:9e dst 10.2.1.12 self
62:74:69:88:5b:53 dst 10.2.1.12 self
```

These new entries:

* Add sf-2 (10.2.1.12) to the mesh with a broadcast MAC (00:00:00:00:00:00)
* Add our new instance to the mesh (00:00:00:e0:b7:9e)
* And add vxlan-2 on sf-2 to the mesh (62:74:69:88:5b:53)

To repeat some commands from above but on sf-2, we now have two new network interfaces over there:

```
sf-2 # ip addr show vxlan-2
94: vxlan-2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc noqueue master br-vxlan-2 state UNKNOWN group default qlen 1000
    link/ether 62:74:69:88:5b:53 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::6074:69ff:fe88:5b53/64 scope link 
       valid_lft forever preferred_lft forever

sf-2 # ip addr show br-vxlan-2
95: br-vxlan-2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc noqueue state UP group default qlen 1000
    link/ether 62:74:69:88:5b:53 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::6074:69ff:fe88:5b53/64 scope link 
       valid_lft forever preferred_lft forever
```

And the mesh looks like this:

```
sf-2 # bridge fdb show brport vxlan-2
62:74:69:88:5b:53 vlan 1 master br-vxlan-2 permanent
62:74:69:88:5b:53 master br-vxlan-2 permanent
00:00:00:00:00:00 dst 10.2.1.11 self permanent
00:00:00:00:00:00 dst 10.2.1.12 self permanent
00:00:00:e0:b7:9e dst 127.0.0.1 self 
62:74:69:88:5b:53 dst 127.0.0.1 self
```

When we launched our test instance, we were told that it had a console port of 44150. We can connect to that to get an interactive remote serial console:

```
sf-1 # telnet sf-2 44150
Trying 10.2.1.12...
Connected to sf-2.
Escape character is '^]'.

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
inst-on-sf-2 login: cirros
Password: 
$ ip addr list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 00:00:00:e0:b7:9e brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.100/24 brd 192.168.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::200:ff:fee0:b79e/64 scope link 
       valid_lft forever preferred_lft forever
```

Here you can see that instance has an interface named eth0, which has the IP address that Shaken Fist allocated earlier. How did it get an IP address without DHCP? Well, Shaken Fist always attaches a config drive to the instance, and this contains a JSON file with the IP address in it. cloud-init running on boot of cirros has use this to configure the interface. Before we poke more at this instance, let's start another instance on sf-3 so we can do some more testing...

```
sf-1 # sf-client instance create inst-on-sf-3 1 1024 -d 8@cirros -n 41f41085-b6d1-4c47-83ea-c8065bd54292 -p sf-3
uuid        : 989386d3-75df-46c7-99e1-c29de10301af
name        : inst-on-sf-3
namespace   : system
cpus        : 1
memory      : 1024
disk spec   : [{'base': 'cirros', 'bus': None, 'size': 8, 'type': 'disk'}]
node        : sf-3
power state : on
state       : created
console port: 40602
vdi port    : 34793

ssh key     : None
user data   : None

Metadata:

Interfaces:

    uuid    : 8fce045a-4f0b-4019-85b9-dcdd0f301840
    network : 41f41085-b6d1-4c47-83ea-c8065bd54292
    macaddr : 00:00:00:e3:63:67
    order   : 0
    ipv4    : 192.168.0.33
    floating: None
```

And now the mesh on sf-1 looks like this:

```
sf-1 # bridge fdb show brport vxlan-2
c2:8a:fb:94:2b:94 master br-vxlan-2 permanent
c2:8a:fb:94:2b:94 vlan 1 master br-vxlan-2 permanent
00:00:00:00:00:00 dst 10.2.1.11 self permanent
00:00:00:00:00:00 dst 10.2.1.12 self permanent
00:00:00:00:00:00 dst 10.2.1.13 self permanent
00:00:00:e0:b7:9e dst 10.2.1.12 self 
3a:25:6d:58:b4:10 dst 10.2.1.13 self 
00:00:00:e3:63:67 dst 10.2.1.13 self 
62:74:69:88:5b:53 dst 10.2.1.12 self
```

Hopefully you can read these now, but you can see that sf-3 (10.2.1.13) has been added to the mesh, and that the interface MAC for the new instance (00:00:00:e3:63:67) has been added as well. Now would be a good time to point out that these meshes learn over time. If we lookup the mesh state on sf-2 for example, you can see that it has sf-3 as well now (which is manually added by Shaken Fist), but it also has an entry for the new instance. This is because the mesh has learnt this without us explicitly requiring it.

```
sf-2 # bridge fdb show brport vxlan-2
62:74:69:88:5b:53 vlan 1 master br-vxlan-2 permanent
62:74:69:88:5b:53 master br-vxlan-2 permanent
00:00:00:00:00:00 dst 10.2.1.11 self permanent
00:00:00:00:00:00 dst 10.2.1.12 self permanent
00:00:00:00:00:00 dst 10.2.1.13 self permanent
00:00:00:e0:b7:9e dst 127.0.0.1 self 
3a:25:6d:58:b4:10 dst 10.2.1.13 self 
00:00:00:e3:63:67 dst 10.2.1.13 self 
```

For completeness, here's the state of the mesh on sf-3 after the second instance was started:

```
sf-3 # bridge fdb show brport vxlan-2
3a:25:6d:58:b4:10 vlan 1 master br-vxlan-2 permanent
3a:25:6d:58:b4:10 master br-vxlan-2 permanent
00:00:00:00:00:00 dst 10.2.1.11 self permanent
00:00:00:00:00:00 dst 10.2.1.13 self permanent
00:00:00:00:00:00 dst 10.2.1.12 self permanent
00:00:00:e0:b7:9e dst 10.2.1.12 self 
3a:25:6d:58:b4:10 dst 127.0.0.1 self 
00:00:00:e3:63:67 dst 127.0.0.1 self
```

This is where I get unstuck for now. The instance on sf-3 did not get an IP address from config drive as I expected it to, and I am not sure why.