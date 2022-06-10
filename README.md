# Checking network topology from the viewpoint of endpoint machine

## Note

Server mentioned here can be a machine running a service, or a client machine connecting to a server. These tools can be used between client machines, server machine, or a client and a server machine.

## Gather information

### Synopsis

When figuring out how communication is routed between 2 endpoints, it is important to understand their IP assignments, and routing. Particularly important is the understanding on whether both servers consider each other on the same switched network, or whether they require to go through a router in order to contact each other.

### IP and subnet

```bash
$ ip addr sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

The above example shows the IP of eth0 is 172.17.0.2/16 with broadcast address of 172.17.255.255.

In this example, the subnet is `172.17.0.0/16`, i.e. `172.17.0.0 - 172.17.255.255`. This means that should this server try to contact any machine on this subnet, it will do that at L2, using IP->MAC resolution and then sending the packets directly to the destination's MAC address that it obtained by performing an `arp` lookup. Any machine outside of this range will need to go through a router instead.

### Routes

```bash
$ ip route ls
default via 172.17.0.1 dev eth0 
172.16.0.0/24 via 172.17.52.1 dev eth0 
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2 
```

The above shows the following (read from the bottom):
* when contacting `172.17.0.0/16` network IPs, go out via eth0, using source IP `172.17.0.2` (our server IP). This is a known subnet we are part of, so any communication is done on L2 with IP->MAC arp lookup
* when going out to `172.16.0.0/24`, use interface `eth0` and contact this range usign a router at IP `172.17.52.1`
* for all other destinations, send traffic via router with IP `172.17.0.1` using device `eth0`

## L2 check, arp poisoning

### Synopsis

Local communications do not happen directly. Local subnet traffic simply means we are not performing routing via our L3 router. There may exist any number of the following devices between 2 servers: L2 firewalls, switches, hubs, DPI devices

### Troubleshooting

Firstly check the arp tables on each server:

```
$ arp -a
? (172.17.0.3) at 02:42:ac:11:00:03 [ether] on eth0
? (172.17.0.1) at 02:42:09:10:70:1a [ether] on eth0
```

NOTE: If you do not see an IP->MAC resolve, try to `ping 172.17.0.3` to force the arp lookup. Whether ping succeeds is not relevant.

This indicates that the server knows about 2 IPs and their MAC addresses. To ensure arp is not poisoned, go to the router `172.17.0.1` as well as the other server `172.17.0.3` and check their IP and MAC addresses. For example, the following can be observed on `172.17.0.3`:

```
$ ip addr sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

This indicates the real MAC of `172.17.0.3` is `02:42:ac:11:00:02`. The arp table on machihe `172.17.0.2` stated it was `02:42:ac:11:00:03`. This means that there is arp poisoning somewhere. It may just be on the one machine, or may be on a switch/firewall with arp caching. Either way, this machine will not be able to talk to `172.17.0.3`. Assuming that `172.17.0.3` has got the correct arp table, it will be able to send packets one way to `172.17.0.2`.

Note that should the arp table be poisoned to the router instead, this machine would be able to talk to local subnet IPs, but nowhere beyond that (as it would not be able to talk to the router).

### Fix

First try to clear the table:
```
$ arp -v -d 172.17.0.1    
arp: SIOCDARP(dontpub)
```

Then try to `ping 172.17.0.3`. Whether ping works is irrelevant in this case. Check the arp table again. If it's poisoned again, one of the switches/firewalls has a poisoned arp cache and needs to be cleared.

## Routing check

### Synopsis

Should the 2 machines be on different networks, and each machine be able to reach it's first hop (the router), the communication between them will use a router, or set of routers. Note, multiple switches, firewalls, l2 firealls and DPI devices may also exist on the path to the destination.

### Simple check

The first check, assuming that ICMP protocol is enabled (it should! ICMP is used for MUCH MORE than just ping, including packet fragmentation in case of MTU mismatch) is to try and ping the destination.

```
$ ping 172.16.0.5
PING 172.16.0.5 (172.16.0.5) 56(84) bytes of data.
64 bytes from 172.16.0.5: icmp_seq=1 ttl=64 time=0.154 ms
```

If ping works, at least basic ICMP works. Verify that the RTT of the packets is not too high. Here it is `0.154 ms`. If this is too high, communication will be disrupted between servers, and timeouts may occur on low-latency software.

### Checking connectivity

Basic connectivity checks can be performed between 2 servers first, to ensure ports are open.

If `172.17.0.3` is running a service on ports `3001,3002` and want to test connectivity from `172.17.0.2`, run this:
```
nc -zv 172.17.0.3 3001
nc -zv 172.17.0.3 3002
```

If `172.17.0.3` is not currently running a service, use `nc` to listen on those ports on that box first, before retrying the above steps.

So, on `172.17.0.3` run this:
```
nc -4 -v -l -k 172.17.0.3 3001
```

The repeat the above connectivity test steps from above from `172.17.0.2` again.

### Route tracing

It may be benficial to perform a traceroute between the 2 servers and compare the results. The commands for this are:

Command | Meaning | Notes
--- | --- | ---
`traceroute -I 172.16.0.5` | trace route using ICMP packets | ICMP ping be enabled
`traceroute -U 172.16.0.5` | trace route using UDP packets | should work by default to the last hop, unless packets are specifically dropped; if they are, modify port using `-p 3002` for example
`traceroute -T 172.16.0.5` | trace route using TCP packets | should work by default to the last hop, unless packets are specifically dropped; if they are, modify port using `-p 3002` for example

Typically, these commands allow you to find the routes that each server takes when communicating with another server. Traceroute also allows to check which hop has latencies and which hop may be dropping or blocking packets. Traceroute should be checked from server A to B and from B to A, as different routes may be present in each direction.

## Other

Consider using [iperf](https://discuss.aerospike.com/t/benchmarking-throughput-and-packet-count-with-iperf3/2791) to test performance of your link.

Consider checking packet loss by using `netstat -s` for different protocols on each server.

For network issues, always check `dmesg` for hardware faults. and `iptables -L -vn` for rules on the local machine.

Also check `mpstat -P ALL` to see load on the cores and io-wait.

Further checks should include investigating `/proc/interrupts` to ensure interrupts are correctly balanced on all the servers.
