# Chapter 6: Networking: How Computers Communicate

## Where We Are

Part I stayed inside one machine. Chapter 1 drew a web application as cooperating processes separated by network boundaries, and warned that a call across one of those boundaries can end with an **unknown outcome** (Section 1.2). Chapter 2 described the hardware a process runs on. Chapter 3 showed how the operating system shares that hardware among processes and threads. Chapter 4 showed how a process reaches the outside world through **file descriptors**, and taught the socket system calls (`socket`, `bind`, `listen`, `accept`, `connect`), the kernel's SYN queue and accept queue, and the send and receive buffers that every connected socket owns (Section 4.6). Chapter 5 explained what happens when concurrent tasks share memory.

Chapter 4 stopped deliberately at the socket buffer. When a server calls `write(fd, buf, n)`, the kernel copies the bytes into the socket's send buffer and the call returns; when it calls `read(fd, buf, n)`, the kernel copies bytes out of the receive buffer. Everything between one machine's send buffer and another machine's receive buffer was treated as a black box.

This chapter opens the box. Most failures you will debug between machines are mechanisms inside it: a database call that hangs for two minutes and then fails with a connect timeout; an API call that fails in under a millisecond with `Connection refused`; a pooled connection that throws `Connection reset by peer` after sitting idle overnight; a DNS change that some clients still do not see an hour later; a service that runs out of local ports under load and fails with `Cannot assign requested address`. You will trace how bytes leave a socket buffer, get wrapped in headers, cross switches, routers, NAT gateways and firewalls, find their destination through DNS and IP routing, and arrive as an ordered, reliable byte stream (TCP) or as independent datagrams (UDP, and QUIC on top of it).

**After this chapter you can:**
- explain why networks are built in layers, map the TCP/IP model to the OSI vocabulary, and trace encapsulation byte by byte;
- explain how a switch delivers frames, why both MAC and IP addresses exist, and how ARP connects them;
- do CIDR arithmetic by hand, trace a routing decision with longest-prefix matching, and say what changes in a packet at each hop;
- explain TTL, ICMP, `ping`, `traceroute`, fragmentation and Path MTU Discovery, and diagnose a Path MTU black hole;
- trace how DHCP gives a new host its address;
- trace how NAT rewrites addresses and ports, and compare stateless and stateful firewalls;
- trace a DNS lookup from the application to the authoritative server, read `dig` output, and explain why "DNS propagation" is a misleading phrase;
- explain how the 4-tuple lets one server port carry tens of thousands of connections;
- trace TCP's handshake, sequence and acknowledgement numbers, retransmission, flow control, congestion control, teardown and state machine, and explain `TIME_WAIT` and `CLOSE_WAIT`;
- state what TCP does **not** guarantee, and why that forces framing and idempotency onto the application;
- explain when UDP is the right choice, what head-of-line blocking is, and how QUIC avoids it;
- calculate a latency budget, a bandwidth-delay product, and the throughput ceiling that packet loss imposes;
- distinguish L4 from L7 load balancing;
- map every common connection error (refused, connect timeout, read timeout, reset, unknown host, no route) to the event on the wire that caused it, and confirm it with `ping`, `traceroute`, `dig`, `ss` and `tcpdump`.

**Interview questions this chapter prepares you for:**
1. What happens at each layer when you send a packet? How do the TCP/IP and OSI models relate?
2. Why do we need both MAC addresses and IP addresses? How does ARP work?
3. Given `10.0.5.25/22`, how does the host decide whether `10.0.6.100` is local or must go through the gateway?
4. How does `traceroute` work? What is a Path MTU black hole?
5. How does a new machine get an IP address?
6. What is NAT, and how do hundreds of private servers share one public IP?
7. Stateless versus stateful firewall; reject versus drop?
8. How does DNS resolution work? Recursive resolver versus authoritative server?
9. How can one server on port 443 handle 50,000 connections at once?
10. Explain the TCP three-way handshake. Why three messages and not two?
11. Explain the four-way close, `TIME_WAIT` and `CLOSE_WAIT`. Why does `TIME_WAIT` last 2 × MSL?
12. What is the difference between flow control and congestion control?
13. TCP versus UDP? What is head-of-line blocking, and how does QUIC fix it?
14. Does TCP guarantee that a request is processed exactly once?
15. Why does one connection attempt fail instantly with "connection refused" and another hang until a connect timeout?
16. What is ephemeral port exhaustion, and how do you prevent it?

**Scope.** Here: the layered model, Ethernet and ARP, IP addressing and routing, ICMP, DHCP, NAT, firewalls, DNS, ports, TCP in depth, UDP, QUIC as a transport, the network view of sockets, network performance arithmetic, an L4/L7 primer, and wire-level diagnosis. The socket system calls and kernel queues were taught in Chapter 4 (Section 4.6) and are only recalled. HTTP (including HTTP/2 and HTTP/3 framing) is Chapter 7. Cryptography and the TLS handshake are Chapter 8. Load balancer and reverse proxy architecture is Chapter 10. Timeouts, retries and backoff as design tools are Chapter 24; idempotency keys are Chapter 13.

---

## 6.1 Layering and Encapsulation

### Definition

A **network protocol stack** is a set of protocols arranged in layers, where each layer provides one service to the layer above it and uses the service of the layer below it. A **protocol** is an agreement about the format and meaning of messages and the rules for exchanging them (Chapter 1, Section 1.2). **Encapsulation** is what the sender does as data moves down the stack: each layer treats whatever it received from above as an opaque payload and prepends its own **header** (and sometimes appends a **trailer**) of control information. **Decapsulation** is the reverse at the receiver: each layer checks and removes its own header and passes the payload up.

### Why it exists

When an order service sends `{"orderId": 42}` to a payment service in another data centre, many unrelated problems must be solved. Bits must be turned into electrical, optical or radio signals. A switch in the rack must deliver the signal to the right machine. Routers belonging to several organisations must find a path across the world and route around broken links. The receiving machine must put pieces back in order and ask for lost ones again. Its operating system must hand the bytes to the right process out of hundreds. Finally the application must understand the message.

If one piece of software solved all of these together, replacing a copper cable with Wi-Fi would mean rewriting every HTTP library and database driver. Layering separates the problems. Each layer solves one, behind a contract the layer above relies on. TCP gives a reliable byte stream without knowing whether its packets cross Ethernet, Wi-Fi or a submarine cable. A second benefit is that a device in the middle of the network only needs to understand the layers it acts on: a switch reads only the link-layer header; a router reads up to the network-layer header and never parses HTTP.

### Mechanism: the TCP/IP model and the OSI vocabulary

Two layer models appear in every discussion of networking. The **TCP/IP model** has four layers and is what operating systems and the internet actually implement. The **OSI model** (Open Systems Interconnection) is a seven-layer reference model published by ISO; its protocols never displaced TCP/IP, but its **layer numbers** survived as vocabulary. Engineers say "L2", "L3", "L4" and "L7" to say which header a device or rule looks at.

| OSI layer | TCP/IP layer | Unit of data (PDU) | Addresses at this layer | Examples | Devices that act here |
|---|---|---|---|---|---|
| L7 Application | Application | message | host names, URLs | HTTP, DNS, SMTP, SSH, database wire protocols | application processes, L7 proxies |
| L6 Presentation | (part of Application) | | | encodings, compression; TLS is often placed here or "between L4 and L7" | libraries |
| L5 Session | (part of Application) | | | no distinct internet protocol | |
| L4 Transport | Transport | **segment** (TCP), **datagram** (UDP) | ports | TCP, UDP | the host kernel, L4 load balancers, stateful firewalls, NAT |
| L3 Network | Internet | **packet** | IP addresses | IPv4, IPv6, ICMP | routers |
| L2 Data link | Link | **frame** | MAC addresses | Ethernet, Wi-Fi | network interface cards (NICs), switches |
| L1 Physical | (below Link) | bits / signals | | copper, fibre, radio | cables, transceivers |

A **protocol data unit (PDU)** is the name for the unit of data at a given layer. In casual speech everything is a "packet". The precise names matter when someone asks how a TCP segment, an IP packet and an Ethernet frame relate: the segment is the payload of the packet, and the packet is the payload of the frame.

### Concrete: encapsulating one HTTP request

An application sends a 47-byte HTTP request on a connected TCP socket:

```text
GET /health HTTP/1.1\r\nHost: api.example.com\r\n\r\n      (20 + 2 + 21 + 2 + 2 = 47 bytes)
```

(`\r\n` is the two-byte line ending carriage return, line feed; Chapter 7 explains HTTP's format.) The kernel and the NIC wrap it like this before it reaches the cable:

```text
 Application (user space)          write(fd, buf, 47) copies 47 bytes into the send buffer
 ┌───────────────────────────┐
 │ HTTP bytes (47)           │
 └───────────────────────────┘
            │
 Transport (kernel TCP)
 ┌────────────────────┬───────────────────────────┐
 │ TCP header (32)    │ HTTP bytes (47)           │   TCP segment: 79 bytes
 │ src port 49152     │                           │
 │ dst port 80        │                           │   (20-byte base header +
 │ seq, ack, flags    │                           │    12 bytes of options)
 │ window, checksum   │                           │
 └────────────────────┴───────────────────────────┘
            │
 Network (kernel IP)
 ┌──────────────┬────────────────────────────────────┐
 │ IPv4 hdr (20)│ TCP segment (79)                   │   IP packet: 99 bytes
 │ src 10.0.5.25│                                    │
 │ dst 10.0.6.10│                                    │
 │ TTL 64       │                                    │
 │ protocol 6   │                                    │
 └──────────────┴────────────────────────────────────┘
            │
 Link (NIC driver and hardware)
 ┌─────────────────┬────────────────────────────────┬──────────┐
 │ Ethernet hdr(14)│ IP packet (99)                 │ FCS (4)  │  Ethernet frame: 117 bytes
 │ dst MAC         │                                │ CRC-32   │
 │ src MAC         │                                │          │
 │ EtherType 0x0800│                                │          │
 └─────────────────┴────────────────────────────────┴──────────┘
```

Each header contains a field that tells the receiver which protocol the payload belongs to. This is how the receiver knows which code should handle the next layer:

- The Ethernet header's 2-byte **EtherType**: `0x0800` means "the payload is IPv4", `0x86DD` means IPv6, `0x0806` means ARP (Section 6.2).
- The IPv4 header's 1-byte **Protocol** field: `6` means TCP, `17` means UDP, `1` means ICMP (Section 6.3).
- The TCP header's **destination port**, together with the addresses and source port: which socket's receive buffer gets the payload (Section 6.7).

At the receiver, decapsulation runs upward:

1. The NIC checks the 4-byte **frame check sequence** (FCS), a CRC-32 checksum over the frame. A frame damaged by electrical noise fails the check and is silently dropped. If the check passes and the destination MAC address is this NIC's (or a broadcast), the NIC copies the frame into kernel memory by DMA (Chapter 4, Section 4.7) and raises an interrupt (Chapter 3, Section 3.2).
2. The kernel's link layer removes the Ethernet header and trailer, reads EtherType `0x0800`, and hands the 99-byte packet to IPv4.
3. IPv4 verifies its header checksum, checks that the destination address belongs to this host, reads Protocol `6`, removes its header and hands the 79-byte segment to TCP.
4. TCP verifies its checksum, finds the socket matching the addresses and ports, checks that the segment's sequence number is the next one expected, appends the 47 bytes to the socket's receive buffer, schedules an acknowledgement, and wakes any thread blocked in `read` or `epoll_wait` (Chapter 4, Section 4.5).

### Guarantees and non-guarantees

Layering guarantees that upper layers do not change when lower layers do: the application calls the same `read` and `write` whether the bytes cross the loopback interface, a rack switch or an ocean.

It does **not** make lower layers invisible. Every layer adds bytes: a 1-byte payload over TCP, IPv4 and Ethernet costs at least 14 + 20 + 20 + 1 + 4 = 59 bytes, padded to Ethernet's 64-byte minimum frame. Lower-layer limits such as the maximum packet size (Section 6.3) can stall an upper layer that knows nothing about them, and a lower-layer failure surfaces as an exception in application code that names none of the layers.

### Failure and diagnosis

Because each layer depends on the ones below, network diagnosis works layer by layer: can the name be resolved (DNS), is the host reachable (L3), is the port accepting connections (L4), does the protocol answer correctly (L7)? The classic waste of time is debugging HTTP headers or application code when the failure is a missing route or a firewall dropping packets. Section 6.14 turns this into a table from error message to layer.

### Backend connection

Every arrow between two boxes in an architecture diagram is a trip down one stack, across switches and routers, and up another stack. Knowing which layer does what explains, later in this book, why an L4 load balancer is cheaper but less capable than an L7 proxy (Section 6.13, Chapter 10), why TLS hides URLs and headers but not IP addresses and ports (Chapter 8), and why reusing a connection saves work at several layers at once.

---

## 6.2 The Link Layer: Ethernet, MAC Addresses, Switches, ARP, MTU

### Definitions

The **link layer** (L2) moves a frame between two network interfaces on the same local network: interfaces that are directly connected, or connected through switches, with no router in between. The set of interfaces that receive a broadcast frame sent by any one of them is a **broadcast domain**.

A **MAC address** (Media Access Control address) is a 48-bit identifier of a network interface, written as six hexadecimal bytes such as `02:42:ac:11:00:02`. The address with all bits set, `ff:ff:ff:ff:ff:ff`, is the **broadcast address**: every interface on the local network accepts a frame sent to it.

An **Ethernet frame** is the link-layer unit: a 14-byte header (destination MAC, source MAC, 2-byte EtherType), a payload of 46 to 1,500 bytes, and a 4-byte FCS trailer. (An optional 4-byte **VLAN** tag can sit inside the header. A VLAN, or virtual LAN, splits one physical switched network into several separate broadcast domains; frames tagged for one VLAN never reach interfaces in another.)

**ARP** (Address Resolution Protocol) is the protocol an IPv4 host uses to find the MAC address that currently owns an IP address on the same local network.

The **MTU** (maximum transmission unit) is the largest IP packet, in bytes and including the IP header, that an interface will carry in one frame. Standard Ethernet's MTU is 1,500 bytes.

### Why they exist

A common interview question is why both MAC addresses and IP addresses exist.

An **IP address** is a *location* in the network's topology, like a postal address. Addresses are handed out in blocks, so a router on the other side of the world needs one entry for a whole block (say, all 256 addresses that start with `203.0.113.`) rather than one entry per machine. When a machine moves to another network, its IP address changes.

A **MAC address** is a *name* for an interface with no location in it. Originally the manufacturer burned it into the NIC; in clouds and containers, the hypervisor or container runtime assigns it. Flat names cannot be grouped into blocks, so they cannot be used for routing across the world: every router would need an entry for every interface in existence. But they are exactly what a local network needs. A switch can forward on a fixed 6-byte field in hardware at line rate, and a machine that has just booted and has no IP address yet (Section 6.4) can still receive frames.

In short: **IP addresses tell routers where a host is in the global network; MAC addresses tell switches and NICs which interface on this local link should take the frame.**

### Mechanism 1: how a switch forwards frames

An **Ethernet switch** connects machines on a local network and keeps a **MAC address table** (also called a CAM table) that maps MAC address to switch port.

```text
   Host A                 Host B                 Router R1
   10.0.1.10              10.0.1.20              10.0.1.1
   MAC aa:aa:..           MAC bb:bb:..           MAC cc:cc:..
      │ port 1                │ port 2                │ port 3
   ┌──┴───────────────────────┴───────────────────────┴──┐
   │                      switch                         │
   │  MAC table:  aa:aa:.. → 1   bb:bb:.. → 2   cc:cc:.. → 3
   └─────────────────────────────────────────────────────┘
```

When a frame arrives on port 1 with source `aa:aa:..` and destination `bb:bb:..`:

1. **Learn.** The switch records "`aa:aa:..` is on port 1", with a timestamp. Entries that are not refreshed expire after a few minutes.
2. **Forward.**
   - *Known destination:* `bb:bb:..` is in the table at port 2, so the frame goes out of port 2 only. Port 3 never sees it.
   - *Unknown destination:* the switch **floods** the frame out of every port except the one it came in on. Only B's NIC accepts it. When B replies, the switch learns where B is.
   - *Broadcast destination:* the frame is flooded to every port in the broadcast domain.

A switch never changes the frame; it only decides where to send it.

### Mechanism 2: ARP

Host A (`10.0.1.10`) wants to send an IP packet to B (`10.0.1.20`) on the same network. The kernel knows B's IP address, but the Ethernet header needs B's MAC address.

1. **Check the ARP cache.** The kernel keeps a table of recent IP-to-MAC mappings (the **neighbour table**). If B is there and the entry is valid, it fills in the MAC and sends.
2. **Broadcast a request.** On a miss, the kernel parks the outgoing packet in a small queue and broadcasts an ARP request (EtherType `0x0806`) to `ff:ff:ff:ff:ff:ff`: "Who has 10.0.1.20? Tell 10.0.1.10 (at aa:aa:..)."
3. **Unicast reply.** Every host on the local network receives the broadcast. All but `10.0.1.20` ignore it. B records A's mapping in its own cache (it is about to reply to A anyway) and sends a **unicast** reply to A's MAC: "10.0.1.20 is at bb:bb:..".
4. **Cache and send.** A stores the mapping and sends the parked packet.

On Linux an entry is `REACHABLE` for a randomised period of tens of seconds after confirmation; it then becomes `STALE` and is re-checked the next time it is used. If no reply comes after a few attempts about a second apart, the entry becomes `FAILED`.

### Concrete: an ARP exchange on the wire

`tcpdump -e` prints the link-layer header as well as the IP one:

```text
$ tcpdump -enni eth0 'arp or host 10.0.1.20'
10:15:01.000100 02:42:ac:11:00:0a > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42:
    Request who-has 10.0.1.20 tell 10.0.1.10, length 28
10:15:01.000240 02:42:ac:11:00:14 > 02:42:ac:11:00:0a, ethertype ARP (0x0806), length 42:
    Reply 10.0.1.20 is-at 02:42:ac:11:00:14, length 28
10:15:01.000255 02:42:ac:11:00:0a > 02:42:ac:11:00:14, ethertype IPv4 (0x0800), length 74:
    10.0.1.10.49152 > 10.0.1.20.5432: Flags [S], seq 294810112, win 64240, length 0
```

Line 1 is the broadcast request. Line 2 is B's unicast reply, 140 µs later. Line 3 is the packet that was waiting: the first packet of a TCP connection to PostgreSQL's port 5432 (Section 6.8 explains `Flags [S]`).

The neighbour table can be inspected with `ip neigh`:

```text
$ ip neigh show dev eth0
10.0.1.1 lladdr 02:42:ac:11:00:01 REACHABLE
10.0.1.20 lladdr 02:42:ac:11:00:14 STALE
10.0.1.99  FAILED
```

`10.0.1.99 FAILED` means ARP requests went unanswered. A `connect` to that address fails with `EHOSTUNREACH` ("No route to host") after the ARP attempts run out, typically a few seconds.

IPv6 does not use ARP. It uses **NDP** (Neighbor Discovery Protocol), which sends the question to a multicast address that only a small subset of hosts listens to, instead of waking every interface on the network.

### MTU

Standard Ethernet carries at most 1,500 bytes of payload per frame (1,518 bytes including header and FCS). Many data-centre and cloud networks support **jumbo frames** with an MTU around 9,000 bytes, which reduces per-packet overhead for bulk traffic such as storage and replication. Tunnels reduce the usable MTU instead: a VPN or an overlay network (used by many container platforms) wraps each packet in extra outer headers, so the inner packet must be smaller. Section 6.3 shows what happens when a packet meets a link whose MTU is smaller than the packet.

### Guarantees and non-guarantees

Ethernet gives **best-effort** delivery within one local network, with error detection: a corrupted frame is detected by its FCS and dropped. It does not guarantee delivery (a switch whose output buffer is full drops frames), it does not retransmit, and it does not authenticate anything. In particular **ARP has no authentication**: any host on the local network can send a forged reply claiming the default gateway's IP address. This is **ARP spoofing**, and it lets an attacker on the same network receive traffic meant for the gateway. Cloud virtual networks prevent it in the hypervisor's virtual switch, which drops frames whose source addresses were not assigned to that machine. On an untrusted network, only encryption with authentication of the peer (Chapter 8) protects the data.

### Failure and diagnosis

- **Unanswered ARP.** A connection to an address on the same subnet hangs for a few seconds and fails with `No route to host`. Evidence: `ip neigh` shows the address as `FAILED`, and `tcpdump -ni eth0 arp` shows repeated `Request who-has` with no `Reply`. Causes: the machine is down, the IP address is wrong, or the machine is on a different VLAN.
- **MTU mismatch.** Small requests work, large responses hang. Evidence: `ip link show` reports different `mtu` values on the two ends or on a tunnel interface along the path. Section 6.3 covers the mechanism.

### Backend connection

Two link-layer facts matter constantly:

1. **ARP never crosses a router.** Your server never learns the MAC address of a remote client or of a third-party API. Every frame it sends to a remote IP address carries the MAC address of its **default gateway** (Section 6.3).
2. **Containers use the same mechanisms in software.** A container in its own network namespace is usually connected to the host through a `veth` pair attached to a virtual bridge (Chapter 4, Section 4.10). Each container has its own MAC and IP address and runs ARP like a physical machine. High-availability setups use a related trick: when a standby machine takes over a shared "virtual IP", it broadcasts a **gratuitous ARP** (an unsolicited announcement of its own mapping) so that neighbours update their caches immediately.

---

## 6.3 The Network Layer: IP, CIDR, Routing, ICMP, Path MTU

### Definitions

The **network layer** (L3) delivers packets from a source host to a destination host across many interconnected networks.

An **IPv4 address** is a 32-bit number written as four decimal bytes, such as `192.168.1.50`.

A **subnet** is a block of consecutive IP addresses that share the same leading bits. **CIDR** notation (Classless Inter-Domain Routing) writes a block as an address followed by `/N`, the **prefix length**: the first N bits identify the network, and the remaining 32 − N bits identify a host within it. The **subnet mask** is the same information written as a 32-bit number with N one-bits followed by zeros (`/24` is `255.255.255.0`).

A **router** is a device (or a kernel function) connected to two or more networks that forwards each packet toward its destination according to a **routing table**, a list of (destination block → next hop, interface) entries.

The **default gateway** is the router to which a host sends every packet whose destination is not on a directly connected network. In a routing table it is the entry for `0.0.0.0/0`, which matches every address.

**ICMP** (Internet Control Message Protocol) is the network layer's error and diagnostic protocol, carried inside IP packets (Protocol `1`). Routers and hosts use it to report problems such as "destination unreachable" and "time exceeded", and `ping` uses it to test reachability.

### Why they exist

A single broadcast domain does not scale. If the internet were one big Ethernet, every ARP broadcast would reach every machine on Earth and every switch would need an entry for every interface. The network layer divides the world into many networks connected by routers, and gives each network a block of addresses with a common prefix. Outside a block, nobody needs to know about individual hosts inside it; they only need to know how to reach the prefix. This is **hierarchical addressing**, and it is what lets a backbone router work with hundreds of thousands of routes instead of billions.

### Mechanism 1: the IPv4 header

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------+-------+-----------+---+-------------------------------+
|Version|  IHL  |   DSCP    |ECN|         Total Length          |
+-------+-------+-----------+---+-----+-------------------------+
|        Identification         |Flags|     Fragment Offset     |
+---------------+---------------+-----+-------------------------+
|  Time to Live |   Protocol    |        Header Checksum        |
+---------------+---------------+-------------------------------+
|                    Source IP Address                          |
+---------------------------------------------------------------+
|                  Destination IP Address                       |
+---------------------------------------------------------------+
```

The fields worth knowing:

- **Version** is 4. **IHL** (header length) is 5 in units of 4 bytes, so 20 bytes when there are no options.
- **DSCP and ECN.** DSCP marks a traffic class for networks that prioritise traffic. **ECN** (Explicit Congestion Notification) lets a congested router mark a packet instead of dropping it, a signal TCP can use (Section 6.8).
- **Total Length** is the size of the whole packet in bytes. It is 16 bits, so a packet can never exceed 65,535 bytes; in practice the MTU limits it.
- **Identification, Flags and Fragment Offset** support fragmentation (Mechanism 4). One flag is **DF**, "don't fragment".
- **TTL** (time to live) is a **hop counter**, not a time. The sender sets it (Linux uses 64 by default, Windows 128); every router that forwards the packet decrements it by 1; a router that decrements it to 0 drops the packet and sends an ICMP "time exceeded" message to the source. Without TTL, a routing loop (router A forwards to B, B forwards back to A) would keep a packet circulating forever.
- **Protocol** names the payload: 1 ICMP, 6 TCP, 17 UDP.
- **Header Checksum** covers only the header. Because TTL changes at every hop, every router recomputes it. (IPv6 dropped this checksum; the link and transport layers have their own.)
- **Source and Destination addresses** stay the same from end to end, unless a NAT device rewrites them (Section 6.5).

### Mechanism 2: CIDR arithmetic and the local-or-routed decision

Every time a host sends a packet, it first decides: is the destination on my own subnet (deliver directly, using ARP for the destination's MAC), or not (send to a router)? It decides by applying its subnet mask with a bitwise AND to both addresses and comparing the results.

| Prefix | Mask | Host bits | Addresses in block |
|---|---|---|---|
| `/32` | 255.255.255.255 | 0 | 1 (a single host) |
| `/28` | 255.255.255.240 | 4 | 16 |
| `/24` | 255.255.255.0 | 8 | 256 |
| `/22` | 255.255.252.0 | 10 | 1,024 |
| `/16` | 255.255.0.0 | 16 | 65,536 |
| `/8` | 255.0.0.0 | 24 | 16,777,216 |
| `/0` | 0.0.0.0 | 32 | all (the default route) |

In an ordinary subnet two addresses cannot be given to hosts: the **network address** (all host bits zero, such as `10.0.1.0` in `10.0.1.0/24`), which names the subnet itself, and the **broadcast address** (all host bits one, `10.0.1.255`). A `/24` therefore holds 254 hosts. Cloud providers reserve a few more addresses per subnet for their own router, DNS and future use; check your provider's documentation when sizing small subnets.

**Worked example.** Server `app-1` has address `10.0.5.25/22` and default gateway `10.0.4.1`. It sends to a cache at `10.0.6.100` and to a database at `10.0.12.50`.

A `/22` prefix covers 8 + 8 + 6 bits, so the boundary falls inside the third byte, where the mask is `11111100` (252). Only the third byte needs work:

```text
mask, 3rd byte:          252 = 1111 1100

app-1    10.0.5.25       5   = 0000 0101  AND mask → 0000 0100 = 4   → network 10.0.4.0/22
                                                                       (10.0.4.0 – 10.0.7.255)
cache    10.0.6.100      6   = 0000 0110  AND mask → 0000 0100 = 4   → 10.0.4.0/22: same subnet
                                                                       → deliver directly (ARP for 10.0.6.100)
database 10.0.12.50      12  = 0000 1100  AND mask → 0000 1100 = 12  → 10.0.12.0/22: different subnet
                                                                       → send via a router
```

Note that `10.0.5.25` and `10.0.6.100` have different third bytes but are on the same subnet. Reading "same third byte means same subnet" is correct only for `/24`.

### Mechanism 3: the routing table and longest-prefix matching

The kernel makes the decision by looking up the destination in its routing table:

```text
$ ip route show
default via 10.0.4.1 dev eth0 proto dhcp src 10.0.5.25 metric 100
10.0.4.0/22 dev eth0 proto kernel scope link src 10.0.5.25
10.0.12.0/24 via 10.0.4.2 dev eth0
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
```

- `default via 10.0.4.1`: anything not matched elsewhere goes to the gateway `10.0.4.1`.
- `10.0.4.0/22 dev eth0 ... scope link`: this subnet is directly attached to `eth0`; deliver directly.
- `10.0.12.0/24 via 10.0.4.2`: a more specific route sends the database subnet through a second router.
- `172.17.0.0/16 dev docker0`: a local container bridge.

When several entries match, the one with the **longest prefix** (the most specific) wins. This rule is **longest-prefix matching**:

- `10.0.6.100` matches `default` (0 bits) and `10.0.4.0/22` (22 bits). `/22` wins: deliver directly on `eth0`.
- `10.0.12.50` matches `default` and `10.0.12.0/24`. `/24` wins: send to `10.0.4.2`.
- `93.184.216.34` matches only `default`: send to `10.0.4.1`.

"Send to a router" means: put the *final* destination in the IP header, ARP for the *router's* MAC address, and put the router's MAC in the Ethernet header. `ip route get 10.0.12.50` prints the decision for one address, which is the fastest way to check it.

### Mechanism 4: what changes at each hop

A classic interview question: when host A sends to host B across two routers, what happens to the MAC and IP addresses at each hop?

```text
 Host A ──link 1── R1 ──link 2── R2 ──link 3── Host B
 10.0.1.10        .1.1 | .2.1   .2.2 | .3.1    10.0.3.50
 MAC A            R1a  | R1b    R2a  | R2b     MAC B
```

| Link | Source MAC | Destination MAC | Source IP | Destination IP | TTL |
|---|---|---|---|---|---|
| 1 (A → R1) | A | R1a | 10.0.1.10 | 10.0.3.50 | 64 |
| 2 (R1 → R2) | R1b | R2a | 10.0.1.10 | 10.0.3.50 | 63 |
| 3 (R2 → B) | R2b | B | 10.0.1.10 | 10.0.3.50 | 62 |

- **MAC addresses are replaced at every router**, because each link is a separate local network and the frame only has to reach the next device.
- **IP addresses stay the same end to end** (unless NAT rewrites them), while **TTL drops by one per router**, and each router recomputes the header checksum.

Each router does the same thing a host does: look up the destination with longest-prefix matching, find the next hop, ARP for the next hop's MAC on the outgoing link, and send a new frame.

### Mechanism 5: ICMP, ping and traceroute

| ICMP type / code | Name | When it is sent |
|---|---|---|
| 8 / 0 | Echo Request | sent by `ping` |
| 0 / 0 | Echo Reply | the target's answer |
| 3 / 0, 1 | Destination Unreachable (network, host) | a router has no route, or the last router's ARP for the host failed; the sender sees "No route to host" |
| 3 / 3 | Destination Unreachable (port) | a UDP datagram arrived at a port with no socket (TCP answers with a reset instead, Section 6.8) |
| 3 / 4 | Fragmentation Needed and DF set | the packet is bigger than the next link's MTU and may not be fragmented; the message carries the next link's MTU |
| 11 / 0 | Time Exceeded | a router decremented TTL to 0 and dropped the packet |

**How `traceroute` discovers the path.** Routers do not announce themselves, so `traceroute` uses TTL:

1. It sends probes toward the destination with **TTL = 1**. The first router decrements TTL to 0, drops the probe, and sends back ICMP Time Exceeded *from its own address*. `traceroute` prints that address and the round-trip time.
2. It sends probes with TTL = 2. The first router forwards them (TTL becomes 1); the second router drops them and reveals itself.
3. It continues with TTL = 3, 4, ... until a probe reaches the destination, which answers in a way that ends the search: ICMP Port Unreachable for the UDP probes Linux `traceroute` sends by default, Echo Reply for ICMP probes (`-I`), or a TCP reply for TCP probes (`-T`).

```text
$ traceroute -n 93.184.216.34
traceroute to 93.184.216.34 (93.184.216.34), 30 hops max, 60 byte packets
 1  10.0.4.1      0.412 ms  0.398 ms  0.377 ms     ← default gateway
 2  100.65.12.9   1.104 ms  1.090 ms  1.211 ms
 3  * * *                                          ← this router does not answer probes
 4  52.93.7.130   2.987 ms  3.002 ms  2.950 ms
 ...
 9  93.184.216.34 79.811 ms 79.790 ms 79.802 ms    ← destination reached
```

Three columns are three probes. `* * *` means no answer arrived for that TTL: the router may be configured not to send Time Exceeded, or to rate-limit it. If later hops answer, the path through the silent hop works. Latency that jumps at one hop and *stays* high for every later hop points to that link (for example a long-distance one); a jump at one hop that disappears at the next usually means only that the router is slow to *generate* ICMP, not that it is slow to forward traffic.

### Mechanism 6: fragmentation and Path MTU Discovery

Suppose a server sends a 1,500-byte packet and, several hops away, the path enters a VPN tunnel whose MTU is 1,400 bytes because the tunnel adds 100 bytes of outer headers.

- **Fragmentation.** Without the DF flag, the router splits the packet into fragments that the destination reassembles. Fragmentation is harmful: losing any one fragment loses the whole packet, so TCP must resend all of it; reassembly needs memory and is a known denial-of-service target; and many firewalls drop fragments. IPv6 routers never fragment.
- **Path MTU Discovery (PMTUD).** Modern TCP stacks set DF on their packets and let the path tell them its limit. The router before the 1,400-byte link drops the oversized packet and returns ICMP type 3 code 4 ("Fragmentation Needed", next-hop MTU 1400; in IPv6, "Packet Too Big"). The sender's kernel caches the smaller **path MTU** for that destination and resends in smaller pieces.

TCP picks the size of its segments from the path MTU. The largest TCP payload in one packet is the **MSS** (maximum segment size, Section 6.8): the MTU minus 20 bytes of IPv4 header and 20 bytes of TCP header, so 1,460 bytes on a 1,500-byte path and 1,360 bytes on a 1,400-byte path.

### Guarantees and non-guarantees

IP guarantees nothing about delivery. It is a **best-effort, connectionless** service: two packets sent one after the other may take different paths, arrive in the opposite order, arrive twice, or be dropped by a congested router without anyone being told. ICMP error messages are themselves best-effort and are often filtered.

`ping` working does not mean your application will work: firewalls treat ICMP and TCP separately, and a host can answer `ping` while nothing listens on the port you need. `ping` failing does not mean the host is down: many networks and cloud security groups drop ICMP Echo by default.

### Failure and diagnosis

**The Path MTU black hole.** A connection is established (the handshake packets are tiny), a small request works, and then the connection hangs whenever either side sends a full-sized packet. The mechanism: a link on the path has a smaller MTU (a VPN, a tunnel, an overlay network), *and* a firewall somewhere blocks all ICMP, including type 3 code 4. The router drops the 1,500-byte packet, its "Fragmentation Needed" message is dropped by the firewall, and the sender, which never learns the path MTU, keeps retransmitting the same 1,500-byte packet until the application times out.

- *Evidence:* `ping -M do -s 1472 <ip>` sends a 1,500-byte packet (1,472 bytes of data + 8 bytes of ICMP header + 20 bytes of IP header) that may not be fragmented. It times out, while `ping -M do -s 1372 <ip>` (1,400 bytes) succeeds. If the ICMP does get through, Linux `ping` prints `Frag needed and DF set (mtu = 1400)`, which names the limit directly.
- *Fixes:* allow ICMP type 3 code 4 (and ICMPv6 Packet Too Big) through firewalls; make the tunnel router rewrite the MSS announced in TCP handshakes down to what fits (**MSS clamping**); set the correct MTU on overlay interfaces; or enable the kernel's fallback that probes for the path MTU without relying on ICMP (`net.ipv4.tcp_mtu_probing` on Linux).

**"Network is unreachable" versus "No route to host".** `ENETUNREACH` is produced locally, before any packet leaves: the routing table has no matching entry, not even a default route (`ip route get <ip>` fails). `EHOSTUNREACH` means a route existed but the last step failed: ARP for the destination or the gateway got no answer, or a router or firewall returned ICMP Destination Unreachable.

### Backend connection

Every time services are spread across subnets, peered networks, VPNs or container overlay networks, routing tables and MTUs decide whether packets arrive at all. A container network that wraps each packet in a 50-byte outer header must give the containers an MTU 50 bytes smaller than the host's; if it does not, health checks (small) pass while large database results and file uploads stall. When "small works, large hangs", think MTU before anything else.

---

## 6.4 DHCP: How a Host Gets Its Address

### Definition

**DHCP** (Dynamic Host Configuration Protocol) is a client–server protocol, carried over UDP (server port 67, client port 68), that gives a host its IPv4 address, subnet mask, default gateway, DNS server addresses and a **lease** time when it joins a network. A lease is a time-limited loan of the address, renewed while the host keeps using it.

### Why it exists

A host needs four pieces of configuration before it can talk beyond its own cable: its address, its mask (to make the local-or-routed decision of Section 6.3), its gateway, and a DNS server (Section 6.6). Configuring these by hand on every laptop, server and virtual machine is error-prone, and impossible when machines are created and destroyed automatically.

DHCP must solve a bootstrap problem: how does a host that has no IP address, and does not know the DHCP server's address, send a packet to ask for one? It uses broadcast at both layers (destination MAC `ff:ff:ff:ff:ff:ff`, destination IP `255.255.255.255`) with source IP `0.0.0.0`, and identifies itself by its MAC address.

### Mechanism: DORA

```text
 new host (UDP 68)                                   DHCP server (UDP 67), 10.0.1.1
 MAC 02:42:ac:11:00:0a
    │ 1. DISCOVER  (broadcast)                               │
    │    src 0.0.0.0:68  dst 255.255.255.255:67              │
    │    client MAC 02:42:ac:11:00:0a, transaction id x      │
    ├───────────────────────────────────────────────────────►│ picks a free address
    │ 2. OFFER                                               │
    │    your address 10.0.1.10, mask /24, router 10.0.1.1,  │
    │    DNS 10.0.0.2, lease 86400 s, server id 10.0.1.1     │
    │◄───────────────────────────────────────────────────────┤
    │ 3. REQUEST  (broadcast)                                │
    │    "I accept 10.0.1.10 from server 10.0.1.1"           │
    ├───────────────────────────────────────────────────────►│ records the lease
    │ 4. ACK                                                 │
    │    10.0.1.10 is yours for 86400 s                      │
    │◄───────────────────────────────────────────────────────┤
 configures eth0 = 10.0.1.10/24, default route via 10.0.1.1, resolver 10.0.0.2
```

Why four messages (**DORA**: Discover, Offer, Request, Acknowledge) rather than two? Because several DHCP servers may be on the network for availability, and all of them may answer the Discover. The client picks one offer and *broadcasts* its Request naming the chosen server, so the chosen server commits the lease and the others see that their offers were declined and return those addresses to their pools. Many clients then check, with an ARP probe, that no other machine already uses the address.

**Renewal.** At half the lease time (**T1**) the client asks its server directly (unicast) to extend the lease; existing connections are unaffected. If that server does not answer by 87.5% of the lease (**T2**), the client broadcasts the request to any server. Only if the lease expires without renewal must the host stop using the address.

**Relays.** DHCP broadcasts do not cross routers. When the DHCP server is on another subnet, the router runs a **DHCP relay agent** that receives the local broadcast and forwards it as a unicast to the server.

### Guarantees and non-guarantees

DHCP guarantees that a host can obtain a working configuration without manual setup. It does not guarantee that a host keeps the same address across leases, unless the server has a reservation for its MAC address. Standard DHCP does not authenticate the server: on an unprotected network, a **rogue DHCP server** can answer first and hand out its own gateway or DNS server, redirecting traffic. Managed switches defend against this with **DHCP snooping**, which accepts server messages only on trusted ports.

### Failure and diagnosis

If no DHCP server answers, many operating systems give the interface a **link-local** address from `169.254.0.0/16` (chosen at random and checked with ARP, per RFC 3927), so that machines on the same cable can still talk to each other. A server that shows `169.254.x.y` in `ip addr` and cannot reach its gateway has failed to get a DHCP lease: check that the DHCP service or relay is reachable and that the interface is on the right network.

Do not confuse that with **`169.254.169.254`**, the address many cloud providers use for the **instance metadata service**, an HTTP endpoint inside the hypervisor that tells a virtual machine about itself and can hand out temporary cloud credentials. Because it hands out credentials, it is a prime target when an attacker can make your server fetch a URL of their choosing (server-side request forgery, Chapter 13).

### Backend connection

In a cloud network, the hypervisor answers DHCP with the address the cloud's control plane assigned to the instance. Instances are replaced, and replacements get new addresses, so configuration should never hard-code instance IPs; services find each other by DNS name (Section 6.6) or through service discovery (Chapter 26).

---

## 6.5 Private Addressing, NAT, IPv6, and Firewalls

### Definitions

A **public IPv4 address** is globally unique and routable across the internet. A **private IPv4 address** comes from one of three blocks reserved by RFC 1918 (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`); routers on the public internet do not forward packets addressed to them, so every organisation can reuse the same private ranges internally.

**NAT** (network address translation) is the rewriting of IP addresses, and usually ports, in packets as they cross a device at a network boundary. **SNAT** (source NAT) rewrites the source of outgoing packets; **DNAT** (destination NAT) rewrites the destination of incoming packets. The device remembers each translation in a **connection-tracking table** (on Linux, **conntrack**) so that it can reverse the rewrite on the reply packets.

**IPv6** is the successor to IPv4, with 128-bit addresses written as eight groups of four hexadecimal digits.

A **firewall** is a device or kernel subsystem that checks each packet against rules and **allows** it, **rejects** it (drops it and sends an error back), or **drops** it (discards it silently). A **stateless** firewall judges each packet on its own. A **stateful** firewall tracks connections and automatically allows packets that belong to a connection it already allowed.

### Why they exist

IPv4 has 2^32 ≈ 4.3 billion addresses, far fewer than the devices now connected. Two answers emerged. The short-term answer was private addresses plus NAT: every network uses private addresses inside and shares a few public addresses at its edge. The long-term answer is IPv6, with enough addresses for every interface to have a unique one.

Private addressing has a second benefit that keeps it in use even where IPv6 exists: a server with no public address cannot receive unsolicited connections from the internet. Cloud backends typically keep application servers, databases and caches in private subnets, accept user traffic only through a load balancer, and let servers reach the internet only through a NAT gateway.

### Mechanism 1: address ranges to recognise on sight

| Block | Meaning |
|---|---|
| `10.0.0.0/8` | private (RFC 1918); common for cloud networks and Kubernetes clusters |
| `172.16.0.0/12` (`172.16.x` to `172.31.x`) | private; Docker's default bridge uses `172.17.0.0/16` |
| `192.168.0.0/16` | private; home and office networks |
| `100.64.0.0/10` | shared address space for carrier-grade NAT (RFC 6598); also used by some overlay and VPN products |
| `127.0.0.0/8` | **loopback**: `127.0.0.1` is the machine itself; packets never reach a NIC |
| `169.254.0.0/16` | link-local (Section 6.4); `169.254.169.254` is the cloud metadata service |

### Mechanism 2: SNAT with port translation

Two private servers, `app-1` (`10.0.1.10`) and `app-2` (`10.0.1.20`), open connections to an external payment API at `93.184.216.34:443` at the same moment. By coincidence both kernels choose source port 49152.

A reply addressed to `10.0.1.10` could never find its way back across the internet: millions of networks use that address. So the packets leave through a NAT gateway that owns the public address `203.0.113.5`. Because two inside connections must share one outside address, the gateway also rewrites the **source port**. This combined form is called **NAPT** (network address and port translation), or "masquerading".

```text
 private subnet 10.0.1.0/24            NAT gateway                internet
 app-1 10.0.1.10:49152 ──┐         inside 10.0.1.1
                         ├────►    outside 203.0.113.5   ────►   payment API
 app-2 10.0.1.20:49152 ──┘                                       93.184.216.34:443
```

The gateway's conntrack table after both connections start:

| Flow | Inside (as sent by the server) | Outside (as seen by the internet) | State |
|---|---|---|---|
| A | `10.0.1.10:49152 → 93.184.216.34:443` | `203.0.113.5:61001 → 93.184.216.34:443` | established |
| B | `10.0.1.20:49152 → 93.184.216.34:443` | `203.0.113.5:61002 → 93.184.216.34:443` | established |

Step by step for flow A:

1. The outgoing packet arrives with source `10.0.1.10:49152`.
2. The gateway picks a free outside port (`61001`), records the mapping, rewrites the source address to `203.0.113.5` and the source port to `61001`, recomputes the IP checksum and the TCP checksum (TCP's checksum covers the addresses too), and forwards the packet.
3. The reply arrives addressed to `203.0.113.5:61001`.
4. The gateway finds flow A by that address and port, rewrites the destination back to `10.0.1.10:49152`, fixes the checksums, and forwards it inside.

Two consequences:

- **SNAT only works for connections started from inside.** An unsolicited packet from the internet to `203.0.113.5` matches no entry, so the gateway has nowhere to send it. To accept inbound connections, an explicit **DNAT** rule (port forwarding) must map an outside address and port to an inside one. Container runtimes use DNAT when they publish container port 8080 on host port 8080, and Kubernetes uses it to turn a virtual service address into the address of one of the pods behind it.
- **The table has finite size and its entries expire.** Every entry is removed after a period with no packets, the **idle timeout**. Section "Failure and diagnosis" below shows why this matters.

### Mechanism 3: IPv6 basics

```text
full form:            2001:0db8:0000:0000:0000:ff00:0042:8329
leading zeros dropped: 2001:db8:0:0:0:ff00:42:8329
"::" for one run of zero groups: 2001:db8::ff00:42:8329
```

Rules that matter in backend code:

1. **Brackets around literal addresses.** Because IPv6 addresses contain colons, an address with a port is written `[2001:db8::1]:8080`, and a URL is `http://[2001:db8::1]:8080/`. Code that splits `host:port` at the first colon breaks on the first IPv6 address it sees.
2. **`::1` is IPv6 loopback.** Many systems map `localhost` to both `127.0.0.1` and `::1`. A server that listens only on `127.0.0.1` will refuse a client that resolves `localhost` to `::1` and tries it first.
3. **`::` as a bind address** means all IPv6 interfaces (and on Linux, by default, IPv4 too). **`::/0`** is the IPv6 default route. **`fe80::/10`** is link-local.
4. **Subnets are normally `/64`**, so address scarcity inside a subnet disappears.
5. **Dual stack and Happy Eyeballs.** A host with both IPv4 and IPv6 may receive both kinds of address for a name. **Happy Eyeballs** (RFC 8305) tries IPv6 first and, if it does not connect within a short delay (the RFC recommends 250 ms), starts an IPv4 attempt in parallel and uses whichever succeeds first, so that a broken IPv6 path costs a fraction of a second instead of a long timeout.

### Mechanism 4: stateless and stateful firewalls, reject and drop

Rules match on fields such as source and destination address, protocol and ports.

```text
 STATELESS (for example a cloud network ACL)
   each packet judged alone; no memory of connections

   client 198.51.100.7:52000 ──SYN to :443──►  inbound rule "allow TCP to port 443"      pass
   client 198.51.100.7:52000 ◄──SYN-ACK──────  outbound rule needed for the CLIENT's port:
                                               "allow TCP to ports 1024–65535"           otherwise dropped

 STATEFUL (for example a cloud security group, Linux conntrack)
   the first packet is checked against the rules; if allowed, the connection is recorded,
   and later packets of that connection, in both directions, are allowed automatically

   client ──SYN to :443──►  rule "allow TCP to port 443" → record connection
   client ◄──SYN-ACK──────  matches a recorded connection → allowed
```

| | Stateless | Stateful |
|---|---|---|
| Memory | none | table of active connections |
| Return traffic | needs its own rule, covering the clients' ephemeral ports (Section 6.7) | allowed automatically |
| Weak point | easy to forget the return rule | the table can fill up |

How a firewall refuses a packet changes what the client experiences:

- **Reject**: the firewall drops the packet and sends back a TCP reset (Section 6.8) or an ICMP Destination Unreachable. The client's `connect` fails within one round trip with `Connection refused` or `No route to host`.
- **Drop**: the firewall sends nothing. The client cannot tell a blocked packet from a lost one, so its TCP stack retransmits the connection request with growing gaps until the **connect timeout** expires. With Linux defaults (six retries of the first packet, Section 6.8) that is about two minutes unless the application sets its own connect timeout.

Internet-facing perimeters usually drop, so that scanners learn nothing and must wait. Inside a private network, rejecting makes a misconfigured dependency fail in milliseconds instead of tying up callers for seconds; either way, callers must set their own connect timeouts.

### Guarantees and non-guarantees

NAT lets private hosts open connections to the internet while sharing a few public addresses. It does not allow unsolicited inbound connections without DNAT, does not provide unlimited connections (to one destination address and port, one public address has at most about 64,000 source ports to offer, and products often document a lower figure), and does not keep an idle connection's entry forever. A stateful firewall allows return traffic automatically; it gives no guarantee that a connection survives once its entry has expired.

### Failure and diagnosis

**The silent idle drop ("ghost connection").** Every NAT gateway and stateful firewall evicts entries that have been idle for its idle timeout; values range from tens of seconds to hours, depending on the product and its configuration (AWS documents 350 seconds for its NAT gateway at the time of writing). **Eviction sends nothing to either end.** Both the client's connection pool and the server still believe the connection is open. When the client next writes on it, the packet reaches the gateway, matches no entry, and is not a connection-opening packet, so the gateway either drops it (the client sees a **read timeout**) or answers with a TCP reset (the client sees **`Connection reset by peer`**).

- *Evidence:* errors cluster on the first request after a quiet period and disappear on retry; the failing connections were idle for longer than the device's documented timeout; a packet capture shows the request leaving and either nothing or a reset coming back.
- *Fixes:* close pooled connections that have been idle for less than the shortest idle timeout on the path, with a safety margin; or send keepalive probes (TCP keepalive, Section 6.8, or a protocol-level ping) more often than that timeout, which keeps the entry alive. A pool's **maximum idle time** (close connections unused for X seconds) is the right control for this; its **maximum lifetime** (close every connection after Y seconds, used or not) solves a different problem and adds reconnection work if set too low.

**SNAT port exhaustion.** Many servers behind one NAT address open many connections to the *same* destination address and port. The gateway runs out of outside ports for that destination; new connections fail or time out, and the gateway's metrics show port-allocation errors. Fixes: reuse connections (pooling, keep-alive), or give the gateway more public addresses.

**Conntrack table full.** A Linux host or Kubernetes node with stateful packet filtering sees a burst of short connections; the conntrack table reaches its limit (`net.netfilter.nf_conntrack_max`) and the kernel drops new connection packets, logging `nf_conntrack: table full, dropping packet` (visible in `dmesg`, Chapter 4, Section 4.12). Fix: reduce connection churn, and size the table for the real connection rate.

> **Java lens:** Connection pools expose the controls above. HikariCP (a common JDBC pool) has `idleTimeout` (close idle connections), `maxLifetime` (retire every connection after a period) and `keepaliveTime` (periodically test idle connections, which also sends packets that keep middlebox entries alive). HTTP client libraries such as Apache HttpClient offer idle-connection eviction. Set them from the measured idle timeouts of the NAT gateways, load balancers and databases on the path.

### Backend connection

A typical cloud backend runs in private subnets, receives traffic through a load balancer in a public subnet, calls external APIs through an SNAT gateway, and is protected by stateful security groups. Two habits come directly from this section: treat every idle connection as possibly dead, and read the *speed* of a failure as evidence. A refusal in a millisecond means something answered; a hang means something swallowed the packet.

---

## 6.6 DNS: Turning Names into Addresses

### Definitions

**DNS** (Domain Name System) is a distributed, hierarchical database that maps names such as `api.example.com` to IP addresses and other data.

A **resource record** is one entry in that database: a name, a type (such as `A`), a class (always `IN` for internet), a **TTL** (time to live: how many seconds a copy may be cached) and the data.

A **zone** is a part of the name tree managed by one organisation, such as `example.com` and the names below it (unless some of them are delegated to another zone). An **authoritative name server** holds the original records of a zone and answers for them.

A **recursive resolver** is a caching DNS server that accepts a question from a client, finds the answer by asking authoritative servers on the client's behalf, caches the records for their TTL, and returns the result. Examples are your cloud network's resolver and public resolvers such as `8.8.8.8` and `1.1.1.1`.

A **stub resolver** is the minimal DNS client inside the operating system or language runtime (on Linux, the C library's `getaddrinfo`), which forwards questions to a recursive resolver and does not walk the tree itself.

### Why it exists

People and configuration files refer to services by stable names; IP routes packets only by address. Addresses change when servers are replaced or traffic moves to another region, and nobody can redeploy every client in the world with a new hard-coded address. Early ARPANET hosts used a single file of names and addresses (`HOSTS.TXT`) copied to every machine, which stopped scaling long ago. DNS replaces the file with a tree of zones, each controlled by its owner, and with caches near the clients so that most questions never travel far.

### Mechanism 1: the hierarchy and a full lookup

Names are read from right to left. Every name ends in the unnamed **root**, written as a trailing dot: the fully qualified form of `api.example.com` is `api.example.com.`.

```text
                         .  (root; 13 named server identities a–m.root-servers.net,
                         │   each served by many machines in many places)
        ┌────────────────┼────────────────┐
       com.             org.             in.          top-level domains (TLDs)
        │
   ┌────┴─────┐
 example.com.  stripe.com.                            zones run by their owners
   │
 api.example.com.  db.example.com.                    records in the example.com zone
```

A lookup with every cache empty:

```text
 app          stub resolver        recursive resolver      root        .com TLD     example.com
 process      (getaddrinfo)        10.0.0.2                servers     servers      authoritative
   │ 1. resolve    │                     │                    │            │              │
   │ api.example   │                     │                    │            │              │
   │ .com          │ /etc/hosts? no      │                    │            │              │
   ├──────────────►│ local cache? no     │                    │            │              │
   │               │ 2. "A api.example.com?"  (recursion desired)          │              │
   │               ├────────────────────►│ cache miss         │            │              │
   │               │                     │ 3. ask root        │            │              │
   │               │                     ├───────────────────►│            │              │
   │               │                     │ 4. referral: ".com is served by these NS,      │
   │               │                     │    here are their addresses (glue)"            │
   │               │                     │◄───────────────────┤            │              │
   │               │                     │ 5. ask .com        │            │              │
   │               │                     ├────────────────────────────────►│              │
   │               │                     │ 6. referral: "example.com is served by          │
   │               │                     │    ns1.example.com, ns2..."     │              │
   │               │                     │◄────────────────────────────────┤              │
   │               │                     │ 7. ask the authoritative server                │
   │               │                     ├───────────────────────────────────────────────►│
   │               │                     │ 8. answer: api.example.com. 300 IN A 93.184.216.34
   │               │                     │◄───────────────────────────────────────────────┤
   │               │ 9. answer           │ caches it for 300 s (and the referrals          │
   │               │◄────────────────────┤ for their own TTLs)                            │
   │ 10. 93.184... │                     │                                                │
   │◄──────────────┤                     │                                                │
```

Two kinds of query appear:

- Step 2 is a **recursive query**: the stub sets the "recursion desired" flag and says, in effect, "give me the final answer or an error, not a referral". The stub's resolver address comes from `/etc/resolv.conf` on Linux.
- Steps 3–8 are **iterative queries**: the recursive resolver asks each level, and each level either answers or **refers** it one level down by returning the name servers (`NS` records) of the child zone, plus their addresses when the resolver could not otherwise find them (**glue records**).

The next lookup of any name under `.com` skips steps 3–4, because the resolver cached the `.com` referral, and the next lookup of `api.example.com` within 300 seconds is answered from cache in steps 2 and 9 alone. Most real lookups are cache hits. (Modern resolvers also send each level only the part of the name it needs, a privacy measure called QNAME minimisation.)

### Mechanism 2: record types

| Type | Maps | Example | Notes |
|---|---|---|---|
| `A` | name → IPv4 address | `api.example.com. 300 IN A 93.184.216.34` | |
| `AAAA` | name → IPv6 address | `api.example.com. 300 IN AAAA 2001:db8::34` | |
| `CNAME` | name → another name (an alias) | `www.example.com. 3600 IN CNAME lb-123.cloud.example.net.` | the resolver must then resolve the target; common for pointing at load balancers and CDNs |
| `NS` | zone → its name servers | `example.com. 86400 IN NS ns1.example.com.` | delegation |
| `SOA` | zone → administrative data | serial number, refresh timers, negative-caching TTL | one per zone |
| `MX` | domain → mail servers, with priority | `example.com. 3600 IN MX 10 mail1.example.com.` | lower number is preferred |
| `TXT` | name → text | `example.com. 3600 IN TXT "v=spf1 ..."` | domain-ownership checks (certificate issuance, Chapter 8), email policies |
| `SRV` | service → priority, weight, **port**, host | `_pg._tcp.db.internal. 60 IN SRV 10 50 5432 db-1.internal.` | service discovery; `A` records cannot carry a port |
| `PTR` | address → name (reverse DNS) | `34.216.184.93.in-addr.arpa. IN PTR api.example.com.` | used by logs, `traceroute`, mail servers |

**Why you cannot put a `CNAME` on the zone apex.** The **zone apex** is the zone's own name (`example.com` itself). The DNS standard says a name that has a `CNAME` may have no other records, so that the alias and the target cannot contradict each other. But the apex must have `SOA` and `NS` records, and usually has `MX` and `TXT`. So `www.example.com` can be a `CNAME` to a load balancer's name, but `example.com` cannot. Managed DNS providers offer non-standard "alias" record types (called `ALIAS`, `ANAME` or similar): the provider's server looks up the target itself and answers with ordinary `A` records.

### Concrete: reading `dig`

`dig` asks a DNS server directly and prints the full response, bypassing `/etc/hosts` and runtime caches:

```text
$ dig api.example.com A

; <<>> DiG 9.18.24 <<>> api.example.com A
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48219
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;api.example.com.               IN      A

;; ANSWER SECTION:
api.example.com.        184     IN      CNAME   edge-lb.cloud.example.net.
edge-lb.cloud.example.net. 42   IN      A       93.184.216.34

;; Query time: 2 msec
;; SERVER: 10.0.0.2#53(10.0.0.2) (UDP)
;; MSG SIZE  rcvd: 98
```

- **`status`**: `NOERROR` is success. `NXDOMAIN` means the name does not exist. `SERVFAIL` means the resolver could not get an answer (for example, the authoritative servers did not respond or failed validation). `REFUSED` means the server will not answer this client.
- **`flags`**: `qr` this is a response; `rd` recursion was desired; `ra` recursion is available. The absence of `aa` (authoritative answer) shows this came from a resolver's cache, not from the zone's own server.
- **`ANSWER SECTION`**: a `CNAME` chain resolved in one response, with the **remaining** TTL of each cached record (184 and 42 seconds). The number counts down in the resolver's cache; it is not the TTL the zone owner configured.
- **`Query time` and `SERVER`**: which resolver answered, and how long it took. A 2 ms answer is a cache hit; a cold lookup across the internet takes tens to hundreds of milliseconds.

Two variants are used constantly: `dig @ns1.example.com api.example.com` asks a specific (for example, authoritative) server directly, and `dig +trace api.example.com` performs the iterative walk itself from the root, showing every referral.

### Mechanism 3: caching, negative caching, DNS load balancing, and "propagation"

**There is no invalidation in DNS.** When you change `api.example.com` from `203.0.113.10` to `203.0.113.20`, nothing tells the caches. The change spreads only as cached copies expire:

```text
 authoritative servers   ← updated within seconds to minutes, depending on the provider
        ▲   re-asked only after the resolver's copy expires
 recursive resolver cache   ← may hold .10 for up to the OLD TTL
        ▲   re-asked only after the OS cache expires
 OS-level cache (systemd-resolved, nscd, a node-local cache)   ← if present
        ▲   re-asked only when the runtime's cache expires
 language runtime cache, and connections that are already open
```

"Waiting for DNS to propagate" is really waiting for caches, at every layer, to expire. The bottom layer is the one people forget: **an open connection never looks at DNS again.** A connection pool that opened its connections to `.10` keeps using `.10` until those connections close, however long ago the record changed.

> **Java lens:** The JVM caches lookups inside `java.net.InetAddress`, controlled by the security properties `networkaddress.cache.ttl` (successful lookups) and `networkaddress.cache.negative.ttl` (failures, 10 seconds by default). When no security manager is installed, current JDKs cache successful lookups for 30 seconds by default; with a security manager installed the default is to cache forever. Services that depend on DNS-based failover (for example a managed database that moves its hostname to a new primary) should set a short positive TTL explicitly and give pooled connections a maximum lifetime, so that the process does not stay attached to the old address.
>
> **Other runtimes:** Node.js's default `dns.lookup()` calls the operating system's blocking `getaddrinfo` on the libuv thread pool (4 threads by default, Chapter 5, Section 5.13). A slow resolver can occupy all four threads and stall unrelated file-system work that uses the same pool.

**Negative caching.** Resolvers also cache the *absence* of a name. A query that returns `NXDOMAIN` is cached for a time taken from the zone's `SOA` record (RFC 2308: the smaller of the `SOA` record's own TTL and its "minimum" field). If a deployment starts an application before creating the DNS record it needs, the first lookup caches `NXDOMAIN`, and the application keeps failing for that period even after the record exists.

**DNS-based load balancing.** An authoritative server can return several `A` records for one name (clients usually try them in turn), or return different answers depending on the region the resolver seems to be in (**GeoDNS**), or stop returning addresses whose health checks fail. This is useful for spreading users across regions or across several entry load balancers. It is a coarse tool: caching means clients keep using a removed address until their cached copy expires, the resolver (not the end user) is what the server sees, so one large resolver can steer many users to one address, and many clients use only the first address returned. Chapter 10 covers load balancing properly.

**Two security mechanisms, briefly.** **DNSSEC** lets a zone sign its records so that resolvers can verify them; it authenticates data but does not encrypt queries. **DoT and DoH** (DNS over TLS, DNS over HTTPS) encrypt the conversation between a client and its resolver; they hide queries from the local network but do not by themselves prove the data came from the real zone. They protect against different attacks.

### Guarantees and non-guarantees

DNS guarantees that a name resolves to the records its zone publishes, as cached by resolvers for at most each record's TTL, when every cache honours TTLs. It does not guarantee instant change, that every runtime honours TTLs, that open connections follow a change, or that a resolver sees the same answer as a user in another region. Plain DNS is unauthenticated and unencrypted.

### Failure and diagnosis

**The `ndots` amplification trap (Kubernetes).** A pod's `/etc/resolv.conf` typically looks like this:

```text
nameserver 10.96.0.10
search prod.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`search` lists suffixes to try, and `ndots:5` tells the stub resolver: "if a name has fewer than five dots, try it with each suffix *before* trying it as written". A lookup of `api.stripe.com` (two dots) therefore asks for `api.stripe.com.prod.svc.cluster.local`, then `api.stripe.com.svc.cluster.local`, then `api.stripe.com.cluster.local`, each returning `NXDOMAIN`, and only then `api.stripe.com.`. With separate `A` and `AAAA` queries, that is 8 packets instead of 2 for every lookup that is not cached locally. Under heavy outbound traffic this load can overwhelm the cluster's DNS servers, and dropped queries surface as intermittent `UnknownHostException` or `EAI_AGAIN` ("temporary failure in name resolution"). Fixes: lower `ndots` in the pod's DNS configuration, run a DNS cache on each node, or write external names with a trailing dot (`api.stripe.com.`), which marks them as fully qualified.

**UDP works, TCP does not.** DNS normally uses UDP port 53. Responses that do not fit in a UDP message (originally 512 bytes; larger with the EDNS(0) extension, where around 1,232 bytes is a common safe size) come back with the **TC** (truncated) flag set, and the client retries over **TCP port 53**. Large `TXT` records, DNSSEC signatures and names with many addresses trigger this. A firewall that allows UDP 53 but blocks TCP 53 makes small lookups work and large ones time out.

**Slow lookups with a 5-second shape.** The C library's stub resolver waits 5 seconds by default before retrying a query that got no answer. A dropped DNS packet therefore shows up as request latency of almost exactly 5 seconds plus the normal time. Diagnosis Lab 2 at the end of this chapter works through it.

### Backend connection

Before a migration that changes a DNS record, lower the record's TTL well ahead of time (at least one old TTL before the change), so that caches already hold the short TTL when you switch. After the switch, keep the old destination running and watch its traffic: TTLs bound compliant caches, not runtimes that cache longer or connections that are already open. Retire the old destination only when its traffic has stopped and the longest connection lifetime has passed. And remember that a cold DNS lookup is part of a cold request's latency (Section 6.12).


---

## 6.7 The Transport Layer: Ports, Multiplexing, the 4-Tuple

### Definitions

The **transport layer** (L4) delivers data between *processes* on two hosts, on top of IP's delivery between *hosts*. TCP adds reliability, ordering, flow control and congestion control on top of that; UDP adds almost nothing.

A **port** is a 16-bit number (0 to 65535) carried in the TCP or UDP header, once for the source and once for the destination, that identifies an endpoint within one IP address (first mentioned in Chapter 1; taught here in full).

**Multiplexing** is combining the data of many sockets into one stream of packets on the sending host, each packet labelled with its ports. **Demultiplexing** is the reverse on the receiving host: reading the addresses and ports of an arriving packet and handing its payload to the right socket.

The **4-tuple** (source IP, source port, destination IP, destination port) identifies one TCP connection. Adding the protocol gives the **5-tuple**, which also distinguishes TCP from UDP traffic on the same ports.

### Why it exists

An IP address identifies a machine (more exactly, an interface). One machine runs a reverse proxy, an application server, a metrics exporter, an SSH daemon and dozens of outbound connections to databases and caches. When a packet arrives, IP knows it reached the right machine and nothing more. Ports and the 4-tuple let the kernel find the right socket among thousands.

### Mechanism: port ranges and demultiplexing

| Range | Name | Use |
|---|---|---|
| 0–1023 | well-known | standard services: 22 SSH, 53 DNS, 80 HTTP, 443 HTTPS. On Linux, binding below 1024 needs root or the `CAP_NET_BIND_SERVICE` capability (Chapter 4, Section 4.10) |
| 1024–49151 | registered | conventional service ports: 5432 PostgreSQL, 3306 MySQL, 6379 Redis, 8080 application servers |
| 49152–65535 | dynamic | the IANA range for **ephemeral ports**, the source ports a kernel assigns to outgoing connections. Linux uses its own default range, 32768–60999 (`/proc/sys/net/ipv4/ip_local_port_range`) |

How can one server listening on port 443 hold 50,000 connections without mixing up their bytes? Because the kernel never identifies a connection by the server port alone. It uses the whole 4-tuple:

```text
 segments arriving at 10.0.1.15, which listens on port 443

 1: 198.51.100.7:51001  → 10.0.1.15:443  data   ─┐
 2: 198.51.100.7:51002  → 10.0.1.15:443  data   ─┤   kernel looks up the 4-tuple
 3: 203.0.113.99:51001  → 10.0.1.15:443  data   ─┤   in a hash table of connections
 4: 198.51.100.88:60123 → 10.0.1.15:443  SYN    ─┘
            │                     │                    │                      │
            ▼                     ▼                    ▼                      ▼
   socket fd 12            socket fd 13         socket fd 14          listening socket fd 3
   198.51.100.7:51001      198.51.100.7:51002   203.0.113.99:51001    *:443 (LISTEN):
   ↔ 10.0.1.15:443         ↔ 10.0.1.15:443      ↔ 10.0.1.15:443       new connection request
```

- Segments 1 and 2 come from the same client address to the same server port, but with different client ports, so they belong to two different connections.
- Segments 1 and 3 have the same client port but different client addresses: again two connections.
- Segment 4 is a connection request (`SYN`, Section 6.8) that matches no existing connection, so it goes to the listening socket, which starts the handshake and later produces a new connected socket through `accept` (Chapter 4, Section 4.6).

### Guarantees and non-guarantees

The 4-tuple means a server port is not limited to 65,535 connections: every distinct client address and port pair is a distinct connection, and the practical limits are memory and file descriptors (Chapter 4, Section 4.9).

It does **not** help a *client* that opens many connections from one address to one destination address and port, such as an application server talking to one database, or a proxy talking to one backend. There, three of the four fields are fixed, and the only variable is the client's ephemeral port: about 28,000 values with Linux defaults. Section 6.11 shows how that runs out.

### Failure and diagnosis

The common mistake is stating that "a server can only handle 65,535 connections because ports are 16 bits". That limit applies only to one client address connecting to one destination address and port. When a server reports "too many open files", it has hit a file-descriptor limit (Chapter 4, Section 4.9), not a port limit.

### Backend connection

`ss -tan` (Section 6.14) prints every TCP connection as `Local Address:Port` and `Peer Address:Port`, which is the 4-tuple. Connection pools, proxies and NAT gateways all manage a finite space of 4-tuples, and several failures in this chapter (port exhaustion, NAT exhaustion, load imbalance) are about that space.

---

## 6.8 TCP in Depth

### Definitions

**TCP** (Transmission Control Protocol, RFC 9293) is a connection-oriented transport protocol that turns IP's unreliable, unordered packets into a **reliable, ordered, full-duplex byte stream** between two sockets. *Connection-oriented* means both ends set up shared state before data flows. *Full-duplex* means data flows in both directions at once, as two independent streams.

A **sequence number** is a 32-bit field that gives the position, in the sender's byte stream, of the first data byte in the segment.

An **acknowledgement number** is a 32-bit field in which the receiver states the next byte it expects. It is **cumulative**: `ack = 2001` means "I have every byte before 2001".

**Flow control** stops a sender from overrunning the receiver. The receiver advertises the free space in its receive buffer, the **receive window** (`rwnd`), in every segment it sends.

**Congestion control** stops senders from overrunning the network. Each sender keeps a private estimate, the **congestion window** (`cwnd`), of how much data the path can carry without overflowing router queues. At any moment, the data sent but not yet acknowledged (**in flight**) must not exceed min(`rwnd`, `cwnd`).

### Why it exists

IP may drop, reorder or duplicate packets, and it does nothing when a sender is faster than a receiver or than the slowest link on the path. Every application would otherwise need its own retransmission timers, reordering buffers, duplicate detection, pacing and back-off. TCP implements all of that once, in the kernel, so that applications can `write` and `read` a stream of bytes. It also has to share the network fairly with every other connection, which is why congestion control exists at all: without it, senders that responded to loss by sending *more* would collapse the network.

### Mechanism 1: the segment header

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------------------------------+
|          Source Port          |       Destination Port        |
+-------------------------------+-------------------------------+
|                        Sequence Number                        |
+---------------------------------------------------------------+
|                     Acknowledgment Number                     |
+-------+-------+-+-+-+-+-+-+-+-+-------------------------------+
| Data  |       |C|E|U|A|P|R|S|F|                               |
|Offset | Rsrvd |W|C|R|C|S|S|Y|I|            Window             |
|       |       |R|E|G|K|H|T|N|N|                               |
+-------+-------+-+-+-+-+-+-+-+-+-------------------------------+
|           Checksum            |         Urgent Pointer        |
+-------------------------------+-------------------------------+
|                 Options (0–40 bytes)                          |
+---------------------------------------------------------------+
```

The base header is 20 bytes; options usually add 12 (Linux sends timestamps on every segment). The flags:

- **SYN** (synchronise): opens a connection and carries the initial sequence number. A SYN counts as one byte of sequence space, so it must be acknowledged like data.
- **ACK**: the acknowledgement field is valid. Set on every segment after the first.
- **FIN** (finish): "I have no more data to send." Also counts as one byte of sequence space.
- **RST** (reset): abort the connection immediately. No acknowledgement, no waiting.
- **PSH** (push): deliver available data to the application promptly. It does **not** mark a message boundary.
- **ECE/CWR**: used by ECN (Section 6.3) to signal congestion without loss.

The options exchanged in the opening segments:

- **MSS** (maximum segment size): the largest payload this end wants to receive in one segment, normally the MTU minus 40 bytes (1,460 on Ethernet). The timestamp option then takes 12 of those bytes, leaving 1,448 bytes of data per full segment, which is why `ss` (Section 6.14) often shows `mss:1448`.
- **Window scale**: the 16-bit Window field can express at most 65,535 bytes, too small for fast long-distance links (Section 6.12). Each side announces a shift count in its SYN; the real window is the field's value shifted left by that count (shift 7 multiplies by 128). The maximum shift is 14, allowing windows of about 1 GiB.
- **SACK permitted**: both ends may use **selective acknowledgements**, which report blocks received beyond a gap (for example, "I have up to 1001, and also 2001–5000"), so the sender resends only what is missing.
- **Timestamps**: used to measure round-trip time accurately and to reject very old segments.

### Mechanism 2: the three-way handshake

```text
 client 10.0.1.10:49152                                   server 10.0.2.99:5432
 CLOSED                                                   LISTEN
   │ 1. SYN  seq=1000                                         │
   │    options: mss 1460, wscale 7, sackOK, timestamps       │
   ├─────────────────────────────────────────────────────────►│
 SYN_SENT                                                  SYN_RCVD (entry in the SYN queue)
   │ 2. SYN+ACK  seq=5000  ack=1001                           │
   │    options: mss 1460, wscale 7, sackOK, timestamps       │
   │◄─────────────────────────────────────────────────────────┤
 ESTABLISHED (connect() returns)                              │
   │ 3. ACK  seq=1001  ack=5001                               │
   ├─────────────────────────────────────────────────────────►│
   │ 4. first data may follow immediately: seq=1001 ...    ESTABLISHED
   │                                                       (entry moves to the accept queue;
   │                                                        accept() can return it)
```

The client's `connect` returns after **one round trip** (when the SYN+ACK arrives). The client does not wait for its own ACK to arrive; it sends the ACK and can send its first request right behind it.

Two interview questions:

1. **Why three messages and not two?** The connection is full-duplex, so *each* side must choose an initial sequence number *and* learn that the other side received it. Message 1 carries the client's number. Message 2 acknowledges it (`ack = 1001`) and carries the server's number. Message 3 acknowledges the server's number (`ack = 5001`). With only two messages, the server would never learn whether its number arrived. It would also have no defence against an old, delayed SYN from a previous connection arriving out of nowhere: it would create a connection that nobody is on the other end of. With three, the real client's ACK must match the server's fresh number, and a stale SYN gets no such ACK.
2. **Why start at a random number rather than 0?** The **initial sequence number** (ISN) is chosen unpredictably for two reasons. First, so that delayed segments from an earlier connection with the same 4-tuple are unlikely to fall inside the new connection's range of valid numbers. Second, for security: if ISNs were predictable, an attacker who cannot see the traffic could forge segments with a spoofed source address and have them accepted. Kernels derive the ISN from a keyed hash of the 4-tuple plus a clock. (`tcpdump` shows sequence numbers *relative* to the ISN after the first segments, starting at 1, unless you pass `-S`.)

### Mechanism 3: sequence numbers, acknowledgements and retransmission

**Stop-and-wait is too slow.** If the sender sent one 1,460-byte segment and waited for its acknowledgement before sending the next, a path with a 50 ms round trip would carry 1,460 bytes per 50 ms = 29 KB/s, however fast the links are.

**The sliding window.** Instead, the sender keeps sending until min(`rwnd`, `cwnd`) bytes are in flight, and keeps a copy of every unacknowledged byte in its send buffer until an acknowledgement covers it:

```text
 the sender's view of its byte stream

 ┌───────────────────┬─────────────────────────┬────────────────────┬──────────────────┐
 │ sent and ACKed    │ sent, not yet ACKed     │ may be sent now    │ may not be sent  │
 │ (buffer freed)    │ (kept for retransmit)   │                    │ yet              │
 └───────────────────┴─────────────────────────┴────────────────────┴──────────────────┘
                     ▲                         ▲                    ▲
               oldest unACKed           next byte to send     oldest unACKed + min(rwnd, cwnd)
```

As acknowledgements arrive, the left edge moves right and frees buffer space; as the receiver's application reads, `rwnd` grows; as congestion control allows, `cwnd` grows. The window "slides" along the stream.

**A loss and its recovery.** The sender sends five 1,000-byte segments starting at byte 1001; the second is lost:

```text
 sender                                                        receiver (expects 1001)
   │ seg 1  seq=1001 len=1000                                      │
   ├──────────────────────────────────────────────────────────────►│ in order: deliver; expect 2001
   │ seg 2  seq=2001 len=1000                                      │
   ├─────────────────────────── ✕ lost                             │
   │ seg 3  seq=3001 len=1000                                      │
   ├──────────────────────────────────────────────────────────────►│ out of order: hold it
   │                      ACK ack=2001  SACK 3001–4001  (dup 1)    │ still expect 2001
   │◄──────────────────────────────────────────────────────────────┤
   │ seg 4  seq=4001 len=1000                                      │
   ├──────────────────────────────────────────────────────────────►│ hold it
   │                      ACK ack=2001  SACK 3001–5001  (dup 2)    │
   │◄──────────────────────────────────────────────────────────────┤
   │ seg 5  seq=5001 len=1000                                      │
   ├──────────────────────────────────────────────────────────────►│ hold it
   │                      ACK ack=2001  SACK 3001–6001  (dup 3)    │
   │◄──────────────────────────────────────────────────────────────┤
   │ three duplicate ACKs → fast retransmit of seg 2               │
   │ seg 2  seq=2001 len=1000                                      │
   ├──────────────────────────────────────────────────────────────►│ gap filled: deliver 2001–6000
   │                      ACK ack=6001                             │
   │◄──────────────────────────────────────────────────────────────┤
```

The receiver keeps acknowledging 2001 because acknowledgements are cumulative and byte 2001 is still missing; each later segment produces a **duplicate ACK**. The SACK blocks tell the sender exactly what arrived. Segments 3–5 sit in the receiver's buffer, invisible to the application, until segment 2 arrives, because TCP delivers bytes only in order.

TCP has two ways to decide that a segment is lost:

1. **Fast retransmit (three duplicate ACKs).** Why wait for three rather than one? Packets are occasionally *reordered* without being lost; one or two duplicate ACKs may simply mean a segment is late. Three is strong evidence of a loss, and recovery then takes about one round trip.
2. **Retransmission timeout (RTO).** If no later segments arrive to generate duplicate ACKs (the lost segment was the last of a request, or a whole burst was lost), only a timer can detect the loss. The sender keeps a smoothed round-trip estimate (SRTT) and its variation (RTTVAR) and sets RTO ≈ SRTT + 4 × RTTVAR (RFC 6298), with a floor: RFC 6298 recommends 1 second; Linux uses 200 ms. Before any round trip has been measured, as for the first SYN, the RTO is 1 second. **On each consecutive timeout the RTO doubles** (exponential backoff), so a peer that has vanished costs ever longer waits.

Two consequences of the timer are worth remembering. A lost SYN costs a full second before the retry (and Linux, by default, gives up on a connection attempt only after six retries, about 127 seconds in total: 1 + 2 + 4 + 8 + 16 + 32 + 64). And when the peer of an established connection disappears with data unacknowledged, Linux by default keeps retransmitting with backoff for roughly 15 minutes (the `tcp_retries2` setting) before reporting an error. That is why applications need their own timeouts.

### Mechanism 4: flow control and congestion control

| | Flow control (`rwnd`) | Congestion control (`cwnd`) |
|---|---|---|
| Protects | the receiving socket's buffer | the routers and links on the path |
| Overflows when | the receiving application reads slower than the sender writes | the sum of all senders exceeds a link's capacity |
| State kept by | the **receiver**, which advertises it in every segment | the **sender**, privately, inferred from ACKs, round-trip times and losses |
| Throttling | as the receive buffer fills, `rwnd` falls toward 0 (a **zero window**); the sender stops and sends occasional **window probes** until the receiver reads and advertises space again (a **window update**) | the sender starts small, grows `cwnd` as ACKs arrive, and cuts it when it sees loss (or ECN marks) |

Flow control is the mechanism behind Chapter 4's observation that a slow reader makes the writer block (Section 4.6): the receiver's window closes, the sender's send buffer fills, and the sending application's `write` sleeps.

**Congestion control step by step.** A new connection has no idea whether the path is a 100 Gb/s data-centre link or a congested mobile link. The sender keeps two variables: `cwnd`, which starts at 10 segments on modern stacks (RFC 6928, about 14.6 KB with a 1,460-byte MSS), and the **slow-start threshold** `ssthresh`, which starts very large.

1. **Slow start.** For every ACK that acknowledges new data, `cwnd` grows by one segment. Since a window's worth of segments produces a window's worth of ACKs, `cwnd` **doubles every round trip**: 10, 20, 40, 80. The name is historical: it is slow only compared with sending a full window at once.
2. **Congestion avoidance.** Once `cwnd` reaches `ssthresh`, growth becomes **additive**: about one segment per round trip (in classic Reno). Linux's default algorithm, **CUBIC**, grows along a cubic curve instead: fast when far below the last size where loss occurred, slow near it, then faster again as it probes for more.
3. **Loss detected by duplicate ACKs.** Segments are still getting through (they generate the duplicate ACKs), so congestion is moderate. Reno sets `ssthresh` and `cwnd` to half the data in flight; CUBIC reduces to 70%. This **multiplicative decrease** together with additive increase is called **AIMD**, and it is what makes competing flows converge on a fair share.
4. **Loss detected by timeout.** Nothing is getting through. The sender sets `ssthresh` to half of what was in flight, sets `cwnd` to **1 segment**, and starts slow start again.

**`cwnd` over time.** The graph is turned sideways: time runs down the page, one row per round trip, and each `#` is two segments.

```text
 RTT  cwnd  phase
  1    10   #####                                     slow start: doubles each RTT
  2    20   ##########
  3    40   ####################
  4    80   ########################################  ← loss: 3 duplicate ACKs
  5    40   ####################                      ssthresh = cwnd = 40 (Reno halves)
  6    41   ####################                      congestion avoidance: +1 per RTT
  7    42   #####################
 ...
 24    59   #############################
 25    60   ##############################            ← loss: retransmission timeout
 26     1                                             ssthresh = 30, cwnd = 1, slow start
 27     2   #
 28     4   ##
 29     8   ####
 30    16   ########
 31    30   ###############                           capped at ssthresh: avoidance again
 32    31   ###############
```

The sawtooth (climb, halve, climb) is the signature of a loss-based algorithm sharing a path. The collapse to 1 after a timeout is why a single timeout hurts throughput much more than a single fast retransmit.

**BBR**, an alternative algorithm developed at Google and available in Linux, does not treat loss as the main signal. It estimates the path's bottleneck bandwidth and its minimum round-trip time from the ACK stream and paces sending to match, aiming to fill the pipe without filling router queues. It behaves better than loss-based algorithms on paths with random (non-congestion) loss; its estimates can still be wrong, and its behaviour when sharing a link with CUBIC flows is an active area of tuning.

### Mechanism 5: Nagle's algorithm, delayed ACKs, keepalive

Two optimisations from the 1980s can combine to add tens of milliseconds to every request.

- **Nagle's algorithm** (sender side, RFC 896) was designed for interactive terminals, where each keystroke sent a 41-byte packet carrying 1 byte of data. The rule: *while any sent data is unacknowledged, hold new small writes (less than one MSS) in the send buffer until the outstanding data is acknowledged or a full segment has accumulated.*
- **Delayed ACK** (receiver side): instead of acknowledging every segment at once, the receiver waits briefly, hoping to send the ACK together with response data or to acknowledge two segments with one ACK. The delay is implementation-dependent; on Linux it is commonly between 40 and 200 ms.

**The interaction.** A client sends a request as two `write` calls: 200 bytes of header, then 100 bytes of body.

1. The header goes out at once: nothing is in flight.
2. The server receives the header, cannot respond until the body arrives, so it has no data to piggyback on and **delays its ACK**.
3. The client's 100-byte body is smaller than one MSS, and the header is still unacknowledged, so **Nagle holds the body**.
4. Each side waits for the other until the delayed-ACK timer fires, often 40 ms. Then the ACK releases the body and the request completes.

The fix is to write each complete message in **one** `write` call by assembling it in a user-space buffer (Chapter 4, Section 4.7), and to set **`TCP_NODELAY`**, which disables Nagle's algorithm, on connections that carry request–response traffic. Most HTTP servers, proxies and database drivers set `TCP_NODELAY` by default; check yours rather than assuming.

**TCP keepalive versus HTTP keep-alive.** The names are confusingly similar.

- *HTTP keep-alive* (persistent connections, Chapter 7) is reusing one TCP connection for several HTTP requests.
- *TCP keepalive* (`SO_KEEPALIVE`) is a kernel timer: on a connection idle for a set time, the kernel sends a probe segment that carries no new data but obliges the peer to reply. If several probes go unanswered, the kernel declares the connection dead and the next `read` or `write` fails. Probes also refresh NAT and firewall entries (Section 6.5). On Linux the defaults are very long: the first probe after **7,200 seconds** (two hours) of idleness, then 9 probes 75 seconds apart. To protect against a 350-second NAT timeout, the application must enable keepalive on the socket and set the idle time (`TCP_KEEPIDLE`) well below it.

> **Java lens:** `Socket.setTcpNoDelay(true)` sets `TCP_NODELAY`; `Socket.setKeepAlive(true)` sets `SO_KEEPALIVE`. Since JDK 11, `jdk.net.ExtendedSocketOptions.TCP_KEEPIDLE`, `TCP_KEEPINTERVAL` and `TCP_KEEPCOUNT` set the timers per socket on platforms that support them. Frameworks built on Netty expose the same options in their channel configuration.
>
> **Other runtimes:** Go's `net` package sets `TCP_NODELAY` by default and enables keepalive on dialled connections. Node.js exposes `socket.setNoDelay()` and `socket.setKeepAlive(enable, initialDelay)`.

### Mechanism 6: closing, the state machine, `TIME_WAIT`, `CLOSE_WAIT`, reset

**The four-way close.** Each direction of a full-duplex connection is closed independently, with a FIN that the other side acknowledges:

```text
 active closer (calls close() first)                          passive closer
 ESTABLISHED                                                  ESTABLISHED
   │ 1. FIN  seq=2001 ack=8001                                    │
   ├─────────────────────────────────────────────────────────────►│ read() will return 0 (end of file)
 FIN_WAIT_1                                                    CLOSE_WAIT
   │ 2. ACK  seq=8001 ack=2002   (the FIN used one sequence number)
   │◄─────────────────────────────────────────────────────────────┤ sent by the kernel at once
 FIN_WAIT_2                                                       │ the application may still send,
   │                                                              │ then calls close()
   │ 3. FIN  seq=8001 ack=2002                                    │
   │◄─────────────────────────────────────────────────────────────┤
 TIME_WAIT                                                     LAST_ACK
   │ 4. ACK  seq=2002 ack=8002                                    │
   ├─────────────────────────────────────────────────────────────►│
   │ waits 2 × MSL (60 s on Linux), then CLOSED                 CLOSED
```

If the passive side has nothing more to send and closes at once, messages 2 and 3 travel as one segment carrying both FIN and ACK, so a close often takes three packets.

**The state machine.** These are the states `ss` and `netstat` display:

```text
                               ┌──────────┐
                               │  CLOSED  │
                               └────┬─────┘
          passive open:             │            active open:
          listen()                  │            connect() sends SYN
                 ┌──────────────────┴───────────────────┐
                 ▼                                      ▼
           ┌──────────┐                           ┌──────────┐
           │  LISTEN  │                           │ SYN_SENT │
           └────┬─────┘                           └────┬─────┘
   recv SYN,    │                                      │ recv SYN+ACK,
   send SYN+ACK ▼                                      │ send ACK
           ┌──────────┐                                │
           │ SYN_RCVD │                                │
           └────┬─────┘                                │
   recv ACK     │                                      │
                └─────────────────┬────────────────────┘
                                  ▼
                           ┌─────────────┐
                           │ ESTABLISHED │
                           └──┬───────┬──┘
     close(): send FIN        │       │        recv FIN: send ACK
                              ▼       ▼
                     ┌────────────┐ ┌────────────┐
                     │ FIN_WAIT_1 │ │ CLOSE_WAIT │  ← waits for the local application
                     └─────┬──────┘ └─────┬──────┘    to call close()
           recv ACK        │              │ close(): send FIN
                           ▼              ▼
                     ┌────────────┐ ┌────────────┐
                     │ FIN_WAIT_2 │ │  LAST_ACK  │
                     └─────┬──────┘ └─────┬──────┘
           recv FIN,       │              │ recv ACK
           send ACK        ▼              ▼
                     ┌────────────┐ ┌────────────┐
                     │ TIME_WAIT  │ │   CLOSED   │
                     └─────┬──────┘ └────────────┘
          after 2 × MSL    ▼
                        CLOSED

 (rarer paths: both sides send FIN at once → CLOSING → TIME_WAIT;
  both sides send SYN at once → SYN_RCVD from SYN_SENT)
```

Two states cause most production confusion.

**`CLOSE_WAIT`: the peer closed, and *your application* has not.** The side that *received* the first FIN enters `CLOSE_WAIT`. Its kernel has already acknowledged the FIN and made `read` return 0; now it waits for the application to call `close`. Nothing on the network can move a socket out of this state. A few `CLOSE_WAIT` sockets are normal for a moment; thousands that stay for hours mean **the application is leaking sockets**: some code path (often an error path) never closes them. Each holds a file descriptor until the process hits "Too many open files" (Chapter 4, Section 4.9).

**`TIME_WAIT`: normal, on the side that closed first.** The side that sent the first FIN ends in `TIME_WAIT` after sending the last ACK, and stays there for **2 × MSL**. **MSL** (maximum segment lifetime) is the longest time a segment is assumed to survive in the network. Linux fixes `TIME_WAIT` at 60 seconds; other systems use values up to four minutes. It exists for two reasons, and interviews expect both:

1. **To finish the close reliably.** If the final ACK (message 4) is lost, the passive closer, still in `LAST_ACK`, retransmits its FIN. The active closer must still remember the connection so it can send the ACK again. Had it already forgotten, it would answer the FIN with a reset, and the peer would report an error for a connection that closed normally.
2. **To keep old segments out of a new connection.** A segment from the old connection may still be delayed somewhere in the network. If a new connection with the same 4-tuple started immediately, that segment could arrive and be accepted as part of the new stream. Waiting long enough for any such segment to have died protects the next connection. (Sequence numbers and timestamps add further protection, which is what allows the kernel, under controlled conditions, to reuse a `TIME_WAIT` 4-tuple early for a new outgoing connection.)

A timeline of reason 1, with the final ACK lost:

```text
 time   active closer (client)                     passive closer (server)
 0.000  sends FIN                                  → CLOSE_WAIT, app calls close()
 0.001                                             sends FIN, → LAST_ACK
 0.002  receives FIN, sends final ACK  ✕ lost
        → TIME_WAIT (timer: 60 s)
 0.205                                             no ACK within its RTO: retransmits FIN
 0.206  still in TIME_WAIT: recognises the FIN,
        sends the ACK again, restarts its timer
 0.207                                             receives ACK → CLOSED
 60.2   timer expires → CLOSED; the 4-tuple
        may now be used by a new connection

 without TIME_WAIT: at 0.206 the client would have no record of the connection,
 would answer the FIN with a RST, and the server would log a reset on a
 connection that had in fact closed cleanly.
```

`TIME_WAIT` costs little memory, but it **holds the 4-tuple**. For a client that opens and closes many connections to one destination, that becomes the ephemeral port exhaustion of Section 6.11.

**Reset.** Where FIN means "I have finished sending", **RST** means "abort now". A reset is sent:

1. in reply to a segment for a port with no listening socket; for a SYN, the client sees `Connection refused`;
2. when data arrives for a connection that no longer exists on this host, for example because the application closed the socket and the peer keeps writing; the writer's next operation fails with `Connection reset by peer` (`ECONNRESET`) or, on a later write, `Broken pipe` (`EPIPE`, Chapter 4, Section 4.6);
3. when an application closes a socket while unread data is still in its receive buffer, or deliberately requests an abortive close (`SO_LINGER` with a timeout of 0);
4. by a firewall, NAT gateway or load balancer that rejects a packet it does not recognise.

**SYN floods and SYN cookies.** Recall from Chapter 4 (Section 4.6) that a listening socket has a **SYN queue** of half-open connections, waiting for the client's final ACK. A **SYN flood** is an attack that sends huge numbers of SYNs from spoofed source addresses that will never complete the handshake, filling the SYN queue so that real clients cannot connect. Linux defends with **SYN cookies** (on by default, `net.ipv4.tcp_syncookies = 1`). When the SYN queue is full, the kernel stores nothing; it encodes the essential connection parameters (a coded MSS, a coarse timestamp and a keyed hash of the 4-tuple) into the sequence number of its SYN+ACK. A genuine client's ACK returns that number plus one, from which the kernel rebuilds the connection. Some options cannot be encoded, so connections created this way may lose features such as window scaling, unless they ride in the timestamp option; the kernel logs "Possible SYN flooding on port ..." when it falls back to cookies.

### Guarantees and non-guarantees

TCP guarantees that the bytes the receiving application reads are exactly the bytes the sending application wrote, in the same order, without gaps or duplicates, for as long as the connection lasts. Corruption is detected by a 16-bit checksum (plus the link layer's CRC), which catches almost all accidental damage but is not protection against deliberate tampering (Chapter 8).

It does not guarantee that data arrives quickly (retransmissions, backoff), that a dead peer is noticed quickly (without keepalive or application timeouts, an idle connection to a vanished peer looks healthy forever), or anything about what the receiving application did with the bytes. Section 6.9 is about that last point.

### Failure and diagnosis

- **`CLOSE_WAIT` growing without bound.** `ss -tan state close-wait | wc -l` rises steadily until "Too many open files". Cause: application code that does not close sockets after the peer closes, usually on an error path. Find the code that owns them with `ss -tanp state close-wait` (which shows the process) and a thread dump or code review of that client.
- **`TIME_WAIT` piling up on a client or proxy.** Tens of thousands of `TIME-WAIT` entries toward one backend address and port, then `connect` fails with `EADDRNOTAVAIL` ("Cannot assign requested address"). Cause: a new connection per request instead of a pool (Section 6.11).
- **A latency floor of about 40 ms on small requests.** Every request takes at least ~40 ms even inside one rack, with idle CPUs and no loss. Cause: Nagle plus delayed ACK on a message sent in several writes. `tcpdump` shows the second write leaving only when an ACK arrives about 40 ms after the first. Fix: one write per message, `TCP_NODELAY`.
- **Stalls with retransmissions.** `ss -ti` shows growing `retrans` counters and a large `rto`; `nstat` shows `TcpRetransSegs` rising. Cause: loss on the path (Section 6.12).

### Backend connection

HTTP/1.1, HTTP/2, gRPC, PostgreSQL, MySQL, Redis and Kafka connections all run this state machine. Section 6.12 puts numbers on why every new connection is expensive: a round trip for the handshake before the first byte, a congestion window that starts at 10 segments and must grow again, and a `TIME_WAIT` entry when it is closed. Reusing connections through pools and persistent connections avoids those costs on every request after the first. (A long-idle connection may restart with a smaller congestion window, but it still skips the handshake.)

---

## 6.9 What TCP Does Not Guarantee

### Definition

TCP guarantees a reliable, ordered byte stream between two **kernels**. It does not provide three things that applications routinely assume it does:

1. **Message boundaries.** It carries a stream of bytes, not a sequence of messages.
2. **Application-level delivery.** An acknowledgement means the peer's kernel has the bytes, not that the peer's application read or acted on them.
3. **Exactly-once execution.** Nothing at the transport layer prevents a request from being executed twice, or reveals whether it was executed at all, when a connection fails mid-exchange.

### Why it matters

"We use TCP, so messages arrive intact and payments are never lost or duplicated" is one of the most dangerous misunderstandings in backend engineering. TCP knows nothing about JSON documents, HTTP requests or database commits, and nothing about a process that crashes a microsecond after its kernel acknowledged a segment.

### Mechanism 1: no message boundaries

A client makes two calls:

```pseudo
write(fd, "{\"id\":1}", 8)
write(fd, "{\"id\":2}", 8)
```

What does the server's `read(fd, buf, 4096)` return? Any of these is correct TCP behaviour:

```text
 A. both writes coalesced into one segment:
      read → 16 bytes: {"id":1}{"id":2}          a naive parser fails on two documents

 B. the boundary falls inside a message (send-buffer timing, a full window, an MSS boundary):
      read → 5 bytes:  {"id"                     a naive parser fails on half a document
      read → 11 bytes: :1}{"id":2}

 C. exactly as written:
      read → 8 bytes, then read → 8 bytes        what testing on a laptop usually shows
```

The kernel may combine writes (Nagle, or simply because both were in the send buffer when the next segment left), split them (a message larger than one segment arrives as several, and `read` returns whatever has arrived, Chapter 4's short reads in Section 4.1), or both. Therefore **every protocol over TCP defines its own framing**, a rule that lets the receiver find where each message ends:

1. **Length prefix:** each message begins with its length, as a fixed-size integer. PostgreSQL's wire protocol, Kafka, and HTTP/2 frames work this way.
2. **Header delimiter plus declared length:** read until a delimiter (`\r\n\r\n` ends HTTP/1.1 headers), then read exactly the number of bytes the headers declare (`Content-Length`, Chapter 7).
3. **Delimiter only:** separate messages with a byte that cannot appear inside them, such as a newline in newline-delimited JSON.

A correct receiver reads in a loop, appending to a buffer, until it has a complete frame, and keeps any extra bytes as the start of the next one.

### Mechanism 2: an acknowledgement comes from the kernel

```text
 client app        client kernel          network        server kernel          server app
 1. write(req) ──► copy into send buffer;
    returns n         │
                      │ 2. segment ─────────────────────► 3. checksum OK; bytes appended to
                      │                                      the receive buffer; ACK scheduled
                      │ 4. ACK ◄────────────────────────     (the application has not run yet)
                      │    free the copy in the                     │
                      │    send buffer                              ▼
                                                            5. the server process is killed
                                                               (OOM, deploy) before read()
```

- `write` returning `n` means the local kernel accepted `n` bytes into its send buffer (possibly fewer than requested: handle short writes). The bytes may not have left the machine.
- The peer's ACK means the peer's *kernel* has the bytes, in order. It is generated before the application runs, and proves nothing about whether the application read, parsed, committed or replicated anything. In the timeline, the client's kernel has a valid ACK for a request the server application never saw.

The only evidence that an application processed a request is a response from that application.

### Mechanism 3: why TCP cannot give exactly-once

A payment service sends "charge $100 to card X" to a payment gateway. The gateway reads it, charges the card, commits, and writes its response. Before the response reaches the caller, the connection breaks (a NAT entry expires, a proxy restarts) or the caller's read timeout fires:

```text
 your service                                              payment gateway
   │ 1. request: charge $100 ───────────────────────────────►│
   │ 2. TCP ACK (gateway kernel) ◄────────────────────────────┤
   │                                                          ├─► charge committed
   │ 3. response "200 OK" ◄──── ✕ connection lost / read timeout fires
   │
 outcome unknown: did the failure happen before the charge or after it?
```

The caller sees a timeout or a reset. Did the charge happen? **TCP cannot tell you.** The outcome is **unknown** (Chapter 1, Section 1.2), and a timeout is not evidence of failure.

- Retry automatically, without protection, and the customer may be charged twice: the request was delivered **at least once**.
- Never retry, and the customer may have been charged with no order recorded, or not charged at all: **at most once**, with an unknown result.

Getting the effect of "exactly once" is an **end-to-end application property**. The client attaches a unique **idempotency key** to the request, and the server records the key atomically with the business change, so that a retry with the same key returns the stored result instead of charging again (Chapter 13 designs this; Chapter 17 covers the atomicity; Chapter 25 applies the same idea to messaging).

### Guarantees and non-guarantees

| Event | What it tells you | What it does not tell you |
|---|---|---|
| local `write` returns `n` | `n` bytes are in the local send buffer | that they left the machine, arrived, or were read |
| the peer's TCP ACK | the bytes are in the peer kernel's receive buffer, in order | that the peer application read, processed or stored them |
| `read` returns data | these bytes came next in the stream | that they form one complete message |
| a response arrives from the peer application | the application processed the request (and says what happened) | anything, if the response is lost |
| a timeout or reset during a request | the exchange did not complete | whether the operation ran |

### Failure and diagnosis

- **Intermittent parse errors under load** (`Unexpected end of JSON input`, truncated Protobuf). The code assumes one `read` returns one message. It works locally because the loopback interface has a very large MTU (65,536 bytes on Linux) and small test messages arrive whole; on a real network a 4 KB message spans several segments and the first `read` returns part of it. Fix: use a framed protocol, and read until the frame is complete.
- **Duplicate charges or rows after network trouble.** A client library or proxy automatically retried a non-idempotent request after a timeout or reset. Evidence: two server-side records with identical content milliseconds or seconds apart, and a retry logged by the client. Fix: idempotency keys (Chapter 13); retry only operations that are idempotent or protected (Chapter 24).

### Backend connection

Two rules follow. Never hand-write socket reading code that assumes message boundaries: use a framed protocol (HTTP, gRPC, a length prefix) and read until the frame is complete. And never let a retry policy repeat a request that changes state unless the operation is idempotent by design or protected by an idempotency key.

---

## 6.10 UDP and QUIC

### Definitions

**UDP** (User Datagram Protocol, RFC 768) is a connectionless transport protocol that adds only ports and a checksum to IP. Each `send` produces one **datagram**, an independent message that is delivered whole or not at all. There is no handshake, no retransmission, no ordering, and no flow or congestion control.

**Head-of-line (HOL) blocking** is the delay of data that has arrived, because data ahead of it in a single ordered sequence has not. In TCP, one lost segment holds back every later byte on the connection, even bytes belonging to completely independent requests.

**QUIC** (RFC 9000) is a transport protocol that runs over UDP and provides several independent, reliable, ordered byte **streams** within one connection, with TLS 1.3 encryption built in (Chapter 8) and connections that can survive a change of client address. It is the transport of **HTTP/3** (Chapter 7).

### Why they exist

Every TCP guarantee has a cost in latency:

1. **The handshake.** A DNS query is a single small question. Over TCP it would first spend a round trip on the handshake, doubling the lookup time. Over UDP the question goes in the first packet.
2. **Ordering at any price.** In a voice or video call, a packet from 300 ms ago is useless; waiting for its retransmission only freezes everything behind it. Such applications prefer to skip the lost data.
3. **One ordered stream per connection.** When protocols began carrying many independent requests over one connection (HTTP/2, Chapter 7), TCP's single stream made every request wait behind any loss.

### Mechanism 1: the UDP header and its semantics

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------------------------------+
|          Source Port          |       Destination Port        |
+-------------------------------+-------------------------------+
|            Length             |           Checksum            |
+-------------------------------+-------------------------------+
|                         payload                               |
+---------------------------------------------------------------+
```

| | TCP | UDP |
|---|---|---|
| Setup | handshake, one round trip before data | none; the first datagram carries data |
| Unit | byte stream; no boundaries | datagrams; each received datagram keeps its boundaries |
| Reliability and order | retransmits; delivers in order | none: datagrams may be lost, duplicated or reordered |
| Flow and congestion control | built in | none in the protocol; the application (or QUIC) must provide it |
| Header | 20 bytes plus options | 8 bytes |
| Kernel state per peer | a socket and state machine per connection | none required |
| Typical uses | HTTP/1.1 and HTTP/2, gRPC, databases, SSH | DNS, DHCP, NTP, voice and video media, metrics agents, QUIC |

Unlike TCP, **UDP preserves message boundaries**: two `sendto` calls of 500 bytes produce two `recvfrom` results of 500 bytes, if both datagrams arrive. A datagram larger than the path MTU is fragmented by IP (Section 6.3) and lost entirely if any fragment is lost, so UDP protocols keep datagrams small. If the receiving buffer passed to `recvfrom` is smaller than the datagram, the rest is discarded.

### Mechanism 2: head-of-line blocking, and how QUIC avoids it

One connection carries three independent responses: stream A (a large file), stream B (a small JSON response) and stream C (a stylesheet). A packet carrying part of A is lost.

```text
 OVER ONE TCP CONNECTION (for example HTTP/2)
 the kernel sees one numbered byte stream; it does not know the streams exist

   seg 1  A bytes      1–1000     → delivered
   seg 2  A bytes   1001–2000     ✕ lost
   seg 3  B bytes   2001–3000     → arrives, held in the kernel
   seg 4  C bytes   3001–4000     → arrives, held in the kernel

   B and C are complete in the receive buffer but cannot be delivered,
   because bytes 1001–2000 come first in the stream. Everything waits at least
   one round trip for the retransmission.

 OVER QUIC (UDP datagrams; HTTP/3)
 each packet has its own packet number; each frame inside names its stream and offset

   pkt 1  A offset    0–1000     → delivered to stream A
   pkt 2  A offset 1000–2000     ✕ lost
   pkt 3  B offset    0–1000     → delivered to stream B at once
   pkt 4  C offset    0–1000     → delivered to stream C at once

   only stream A waits; its missing data is resent in a new packet.
```

QUIC still orders bytes *within* a stream, so stream A waits, and all streams share one congestion window, so loss slows the whole connection's sending rate. What disappears is one stream waiting for another stream's lost data.

QUIC's other design choices:

1. **Why build on UDP instead of a new IP protocol?** Many NAT devices and firewalls on the internet pass only TCP and UDP and drop unfamiliar protocol numbers. This is **ossification**: the network has stopped accepting new transport protocols. TCP is also implemented in operating system kernels, which take years to update across the world's devices. A protocol carried in UDP datagrams passes existing middleboxes, and can be implemented in user-space libraries that ship with applications and update quickly.
2. **Encryption is part of the transport.** QUIC's handshake carries the TLS 1.3 handshake (Chapter 8) in its first packets. A new HTTPS connection over TCP needs one round trip for TCP and one for TLS before the request can be sent; QUIC needs one in total. When resuming a recent connection, a client may even send data in its first packet ("0-RTT"), which can be replayed by an attacker and is therefore restricted to safe requests (Chapter 8). Almost all of QUIC's header is encrypted or protected too, which also keeps middleboxes from depending on its details.
3. **Connection IDs.** A TCP connection *is* its 4-tuple. When a phone moves from Wi-Fi to a cellular network, its address changes and every TCP connection breaks. A QUIC connection is identified by **connection IDs** carried in each packet, so the server can recognise the connection arriving from a new address, verify the new path, and continue.

### Guarantees and non-guarantees

**UDP** delivers each datagram it delivers intact and with its boundaries, as far as the checksum can tell. It does not guarantee delivery, order, uniqueness or congestion safety, and the checksum is error detection, not protection against tampering.

**QUIC** guarantees reliable, ordered delivery within each stream, congestion control for the connection, and encryption and authentication through TLS 1.3. It does not eliminate loss (the affected stream still waits), it usually costs more CPU per byte than kernel TCP (more work happens in user space, and network cards have historically offered fewer offloads for it), and some networks block UDP traffic to port 443, so clients must be able to fall back to TCP.

### Failure and diagnosis

- **Silent UDP drops at the receiver.** UDP has no flow control. If a DNS server or metrics collector receives datagrams faster than it reads them, the socket's receive buffer overflows and the kernel discards the excess, telling no one. Evidence: `ss -uam` shows the receive buffer full; `nstat -az | grep -i RcvbufErrors` shows `UdpRcvbufErrors` rising. Fix: read faster (more reader threads, `SO_REUSEPORT` with several sockets, Chapter 4, Section 4.6) or enlarge the buffer.
- **Amplification attacks.** Without a handshake, nothing verifies a datagram's source address. An attacker sends a small query to an open server with the *victim's* address as the source, and the server sends a much larger answer to the victim. Open DNS resolvers and similar services have been abused this way. QUIC defends itself: until a client's address is validated, a server must not send more than three times the bytes it has received from that address.

### Backend connection

A common architecture terminates QUIC (HTTP/3) at the edge, at a CDN or load balancer close to mobile and browser users, where loss and network changes are common, and forwards requests to the application over long-lived TCP connections inside the data centre, where loss is rare. Application code usually sees ordinary HTTP either way (Chapter 7); what changes is latency on lossy links, and which UDP ports the firewalls must allow.


---

## 6.11 Sockets from the Network's Point of View

### Definitions

An **ephemeral port** is a source port that the kernel picks automatically, from a configured range, when a socket connects (or sends a datagram) without having called `bind` on a specific local port. On Linux the range is 32768–60999 by default: 28,232 ports.

**Ephemeral port exhaustion** is the failure in which a client has no free source port left for a new connection to a given destination address and port, because every candidate 4-tuple is held by a connection that is open or in `TIME_WAIT`. `connect` then fails at once with `EADDRNOTAVAIL` ("Cannot assign requested address").

### Why this view matters

Chapter 4 (Section 4.6) taught the socket calls as the operating system sees them: file descriptors, the SYN queue and the accept queue, buffers. This section maps the same calls onto what happens on the wire and in the space of 4-tuples, which explains two outages that every backend engineer meets: "Address already in use" when a server restarts, and "Cannot assign requested address" when a client is under load.

### Mechanism 1: the server and client lifecycles on the wire

```text
 SERVER                                         CLIENT
 socket()      no address, nothing on the wire  socket()      nothing on the wire
 bind(0.0.0.0:8080)                             connect(10.0.2.99:5432):
   reserves the local port; nothing sent;         a) route lookup: interface eth0,
   EADDRINUSE if the port is taken                   source address 10.0.1.10 (Section 6.3)
 listen(backlog)                                  b) ephemeral port: pick a free one, e.g.
   socket enters LISTEN; the kernel now              49152, such that 10.0.1.10:49152 →
   answers handshakes on its own                     10.0.2.99:5432 is not in use
                                                  c) send SYN; wait one round trip for
                                                     SYN+ACK (or EINPROGRESS if non-blocking)
 accept()                                       write() / read()
   takes one established connection from          segments flow (Section 6.8)
   the accept queue: a new socket, new fd
 close() on a connection: FIN (Section 6.8)     close(): FIN; this side, if first,
                                                  will spend 60 s in TIME_WAIT
```

A client almost never calls `bind` before `connect`. If it bound itself to a fixed local port, it could have only one connection to a given destination at a time, and after each close it would have to wait out `TIME_WAIT` before reconnecting. Letting the kernel choose a fresh ephemeral port gives every connection its own 4-tuple.

### Mechanism 2: the arithmetic of port exhaustion

A service at `10.0.1.10` opens a **new** database connection for every request, runs one query against `10.0.2.99:5432`, and closes it.

1. In the 4-tuple (`10.0.1.10`, *source port*, `10.0.2.99`, `5432`), three fields are fixed. Only the source port varies.
2. With the default Linux range, there are 60999 − 32768 + 1 = **28,232** source ports.
3. The service closes each connection first, so the service is the active closer and **each closed connection keeps its 4-tuple in `TIME_WAIT` for 60 seconds**.
4. The sustainable rate is therefore about 28,232 ports / 60 s ≈ **470 new connections per second** to that one destination.

Above that rate, free ports run out (not instantly: the pool drains at the rate by which new connections exceed recycled ones), and `connect` fails locally with `EADDRNOTAVAIL`. The exact threshold shifts with reserved ports, other connections from the same host, which side closes first, and the kernel's reuse settings, but the shape does not.

**Fixes, in order of importance:**

1. **Reuse connections.** A pool of 50 persistent connections to `10.0.2.99:5432` serves 25,000 queries per second at 2 ms each, using 50 ports and creating no `TIME_WAIT` entries while in use. This is the real fix; the others only raise the ceiling.
2. **Widen the range or add addresses.** A wider `ip_local_port_range`, more source IP addresses, or more destination addresses or ports multiply the 4-tuple space.
3. **Kernel reuse of `TIME_WAIT` for outgoing connections** (`net.ipv4.tcp_tw_reuse` on Linux) lets a new outgoing connection take over a `TIME_WAIT` 4-tuple when timestamps show it is safe. Recent kernels default it to a loopback-only mode; check the value on your kernel before changing it, and fix the connection churn first.

> **Java lens:** `java.net.http.HttpClient` (JDK 11+) keeps a connection pool per client instance, so create one client and share it; a new client per request means a new pool, new connections and new `TIME_WAIT` entries every time. The same holds for Apache HttpClient and OkHttp, whose pool sizes (total and per destination) must be configured for the expected concurrency. Java reports `EADDRNOTAVAIL` on `connect` as an exception whose message is "Cannot assign requested address" (the exception class has varied between JDK versions).
>
> **Other runtimes:** In Python, reuse one `requests.Session` (and read or close every response so its connection returns to the pool). In Node.js, use a keep-alive HTTP agent shared across requests; recent Node versions enable keep-alive on the default global agent, and the built-in `fetch` uses its own shared connection pool.

### Mechanism 3: `SO_REUSEADDR` and `SO_REUSEPORT` from the wire's side

Chapter 4 (Section 4.6) introduced both options. With the state machine of Section 6.8 we can now say exactly what they do:

| Option | Problem | What the kernel allows |
|---|---|---|
| `SO_REUSEADDR` | A server restarts while connections from its previous run are in `TIME_WAIT`, with local address `10.0.1.15:8080`. Without the option, the new process's `bind` to port 8080 fails with `EADDRINUSE` until those entries expire. | `bind` succeeds as long as no socket is actually **listening** on that address and port, even though old connections in `TIME_WAIT` still use it. Servers set it routinely. |
| `SO_REUSEPORT` | Many worker processes or threads accepting from one shared listening socket contend on one accept queue. | Several sockets, each with its own accept queue, listen on the same address and port (if owned by the same user). For each new connection request, the kernel hashes the 4-tuple to pick one socket. |

### Guarantees and non-guarantees

`SO_REUSEADDR` guarantees that `TIME_WAIT` entries from a previous run will not block a restart. It does not let two live servers listen on the same port (that needs `SO_REUSEPORT` on both), and it does not help when the previous instance is still running and still listening, which is the more common cause of `EADDRINUSE` in practice.

### Failure and diagnosis

- **`Address already in use` at startup.** Run `ss -ltnp 'sport = :8080'` (`-p` needs root to show other users' processes). A listening socket with a process name means an old instance, or a child process that inherited the socket, is still alive. No listener, but `ss -tan 'sport = :8080'` shows `TIME-WAIT` entries, means the server does not set `SO_REUSEADDR`.
- **`Cannot assign requested address` on `connect`.** Count connections to the destination: `ss -tan 'dst 10.0.2.99:5432' | wc -l`, and by state with `ss -tan state time-wait 'dst 10.0.2.99:5432' | wc -l`. A count near 28,000, mostly `TIME-WAIT`, confirms exhaustion. (The same error also appears when code explicitly binds to a local address that the machine does not have, which a look at the code or `ip addr` rules out.)

### Backend connection

Creating a client, with its own pool, inside each request handler defeats pooling: every request pays a handshake, and every close leaves a `TIME_WAIT` entry. Create clients once, share them, bound their concurrency, and close or fully read every response so that its connection goes back to the pool.

---

## 6.12 Latency, Bandwidth, Throughput, and Loss

### Definitions

**Latency** is the time for data to travel from sender to receiver. The **round-trip time** (RTT) is the time for a small packet to go to the peer and for its reply to come back.

**Bandwidth** is the capacity of a link, in bits per second (for example 1 Gb/s or 10 Gb/s).

**Throughput** is the rate at which data is actually delivered, never more than the bandwidth of the slowest link. **Goodput** is the useful part of it: application data, excluding headers and retransmissions.

The **bandwidth-delay product** (BDP), bandwidth × RTT, is the amount of data that must be in flight (sent, not yet acknowledged) to keep a path fully busy.

### Why the distinction matters

A common mistake is to expect more bandwidth to make a slow remote call faster. If a service in Mumbai calls a database in Virginia with a 220 ms RTT, a 500-byte query takes about 220 ms on a 100 Mb/s link and about 220 ms on a 100 Gb/s link. Bandwidth is how wide the road is; latency is how long it is, and its floor is set by the speed of light. You can buy more bandwidth. You cannot buy a shorter distance, only move the endpoints closer together.

### Mechanism 1: what latency is made of

The one-way delay of a packet is the sum of four parts:

1. **Propagation delay**: distance divided by the signal's speed. Light in optical fibre travels at roughly 200,000 km/s, about two-thirds of its speed in a vacuum. That is **5 µs per kilometre**, or **1 ms of RTT per 100 km** of fibre. Cables do not run in straight lines, so real RTTs are larger than the straight-line distance suggests.
2. **Transmission (serialisation) delay**: packet size divided by link bandwidth. A 1,500-byte (12,000-bit) frame takes 1.2 µs to put on a 10 Gb/s link, and 120 µs on a 100 Mb/s link.
3. **Processing delay**: the time a router or host spends looking at the packet; microseconds.
4. **Queuing delay**: time spent waiting in a buffer behind other packets when an output link is busy. Unlike propagation, it varies from moment to moment, and together with retransmissions it is what drives tail latency.

Typical round-trip times (orders of magnitude; measure your own paths with `ping`):

| Path | Typical RTT |
|---|---|
| same host, loopback | tens of microseconds |
| same data centre or availability zone | well under 1 ms |
| between availability zones in one cloud region | around 1 ms |
| across a continent (for example US east to US west coast) | 60–80 ms |
| across an ocean (for example New York to London) | 70–80 ms |
| Europe or the US to India or East Asia | 150–250 ms |

### Concrete 1: the latency budget of a cold HTTPS request at 50 ms RTT

A client requests `https://api.example.com/orders` over a path with a **50 ms RTT**. The server needs **15 ms** to produce a response that fits in the initial congestion window (10 segments, about 14 KB). Assume, for simplicity, that the DNS lookup also takes one 50 ms round trip (in practice a nearby resolver is often faster, or the answer is cached). Compare a **cold** connection, where nothing is cached or open, with a **warm** one, where the client reuses a pooled connection:

```text
 COLD                                                    WARM (pooled connection)
 client                         server                    client                       server
   │ DNS query → resolver                                   │ (DNS cached, TCP open,
   │ ◄ DNS answer                       50 ms               │  TLS session established)
   │ SYN ─────────────────────────►│                        │ request ─────────────────►│
   │ ◄──────────────────── SYN+ACK │    50 ms               │                           │ 15 ms
   │ ACK + TLS ClientHello ───────►│                        │ ◄──────────────── response│ 50 ms
   │ ◄──── TLS ServerHello … Finished│  50 ms (TLS 1.3, Chapter 8)
   │ TLS Finished + request ──────►│                        total: 50 + 15 = 65 ms
   │                               │ 15 ms
   │ ◄─────────────────── response │    50 ms
 total: 4 × 50 + 15 = 215 ms
```

| Phase | Round trips | Cold | Warm |
|---|---|---|---|
| DNS lookup | 1 (assumed) | 50 ms | 0 ms (cached) |
| TCP handshake | 1 | 50 ms | 0 ms (connection open) |
| TLS 1.3 handshake (two round trips with TLS 1.2) | 1 | 50 ms | 0 ms (already encrypted) |
| request and response | 1 | 50 ms | 50 ms |
| server processing | | 15 ms | 15 ms |
| **total** | **4 + processing** | **215 ms** | **65 ms** |

On the cold connection, 150 of the 215 ms pass before the server even sees the request. HTTP/3 over QUIC (Section 6.10) merges the TCP and TLS round trips into one, saving 50 ms here. Connection reuse saves all three.

**Slow start adds round trips for larger responses.** A response bigger than the initial window cannot be sent in one round trip on a new connection. A 100 KB response is about 70 segments; with a window of 10, then 20, then 40 segments per round trip, it needs three round trips of data instead of one, adding 100 ms at 50 ms RTT. A warm connection whose window has already grown avoids this.

### Concrete 2: the bandwidth-delay product and the window ceiling

A backup is copied from Virginia to Frankfurt over a 1 Gb/s link with an RTT of 80 ms.

```text
BDP = 1,000,000,000 bit/s × 0.080 s = 80,000,000 bits = 10,000,000 bytes ≈ 10 MB
```

To keep the link busy, about 10 MB must be in flight at all times. Suppose the receive window is limited to 65,535 bytes (window scaling disabled, or a small receive buffer). TCP may have at most min(`rwnd`, `cwnd`) in flight, so it sends 64 KB, then waits for acknowledgements:

```text
throughput ≤ window / RTT = 65,535 bytes / 0.080 s ≈ 819,000 bytes/s ≈ 6.5 Mb/s
```

The link is 1,000 Mb/s; the connection uses 0.65% of it. To fill it, window scaling must be on and the socket buffer limits (`net.ipv4.tcp_rmem` and `tcp_wmem` on Linux) must allow windows of at least the BDP, 10 MB, for this connection.

### Mechanism 2: how loss limits throughput

Now the buffers are large enough, the link is 1 Gb/s, the RTT is 50 ms, and a faulty link drops **1%** of packets. The naive expectation is that 1% of packets are resent and throughput falls by 1%.

That is wrong for loss-based congestion control (Section 6.8). Every loss cuts the congestion window by 30–50%, and with one loss per hundred packets the window never grows large. A well-known approximation, the **Mathis formula**, bounds a loss-based TCP flow's throughput under random loss probability *p*:

```text
throughput ≈ (MSS / RTT) × (1.22 / √p)

with MSS = 1,460 bytes, RTT = 0.05 s, p = 0.01 (√p = 0.1):

throughput ≈ (1,460 / 0.05) × (1.22 / 0.1) = 29,200 × 12.2 ≈ 356,000 bytes/s ≈ 2.8 Mb/s
```

One percent loss reduces a single flow on a 1 Gb/s, 50 ms path to under 3 Mb/s. Throughput falls with the square root of the loss rate and linearly with the RTT, so the same loss hurts long paths much more than short ones. (Algorithms such as BBR, Section 6.8, are less sensitive to random loss; the formula describes Reno- and CUBIC-style behaviour.)

**Loss and tail latency.** For small requests, throughput does not matter, but loss still shows up, in the slowest requests:

- With no loss, a request inside a data centre takes about one RTT plus processing, say 2 ms.
- If a request's only packet, or its last packet, is lost, no later segments can produce duplicate ACKs, so the sender waits for the retransmission timer: at least 200 ms on Linux, and 1 second for a lost SYN.
- With 1% of such requests affected, the median stays at 2 ms while the 99th-percentile latency (p99, the value 99% of requests beat, Chapter 2, Section 2.8; full treatment in Chapter 26) jumps to over 200 ms, a factor of 100.

When p99 is near a multiple of 200 ms or 1 s and the median is normal, suspect loss and retransmission timers before suspecting the application.

### Guarantees and non-guarantees

More bandwidth raises the capacity available to concurrent flows and to bulk transfers whose windows can open wide enough. It does not shorten the RTT of sequential request–response exchanges, does not skip the handshake or slow start on new connections, and does not recover throughput lost to undersized windows or to packet loss.

### Failure and diagnosis

- **Round trips multiplied.** An endpoint makes 100 sequential cache or database calls across a 1.5 ms link between availability zones. Even if each call takes 0.05 ms of server time, 100 × 1.5 ms = **150 ms** is spent waiting for light. Evidence: a trace (Chapter 26) shows many small sequential calls. Fixes: batch the calls, pipeline them, run them concurrently, or keep latency-critical tiers in the same zone.
- **High p99, normal median, low CPU.** Check retransmissions: `nstat -az TcpRetransSegs TcpOutSegs` (compare the two over an interval) or `ss -ti` for per-connection `retrans` counts. A retransmission rate far above what the network normally shows (inside a healthy data centre it is usually a small fraction of a percent) points to a congested link, a faulty port or cable, NIC queue drops (`ethtool -S eth0` counters), or a cloud instance exceeding its network allowance.

### Backend connection

Before adding a network hop (splitting a service, moving a database to another region, adding a synchronous call to a dependency), count the sequential round trips a request will make and multiply by the RTT. Then remember that any loss on any hop surfaces in your own p99.

---

## 6.13 Load Balancing Primer: L4 versus L7

### Definitions

A **layer 4 (L4) load balancer** chooses a backend for each **connection** (or each UDP flow) using only IP and TCP or UDP header information, and then sends all packets of that connection to that backend without interpreting the application data.

A **layer 7 (L7) load balancer** terminates the client's connection (and, for HTTPS, its TLS session), parses application messages such as HTTP requests, and chooses a backend **per request**, sending each request over its own connection to the backend, usually taken from a pool. It can route on application data: the URL path, the `Host` header, a cookie, a gRPC method. A **reverse proxy** (Chapter 1, Section 1.5) is the same kind of component.

### Why both exist

Chapter 10 covers load-balancing algorithms, health checks and proxy internals. The distinction is introduced here because it is a transport-layer question: does the balancer choose per connection or per request? Getting that wrong causes some of the most confusing production imbalances.

There are two common ways to build an L4 balancer. A *packet-forwarding* L4 balancer rewrites and forwards packets (for example with DNAT, Section 6.5), so the client's TCP connection really ends at the backend. An L4 *TCP proxy* accepts the client's connection and opens a second connection to the backend, copying bytes between them without parsing them. Either way, the decision is made once per connection. An L7 balancer always works with whole requests.

### Mechanism: connection topology

```text
 L4 (packet forwarding)
 the balancer picks a backend when the SYN arrives, records the flow,
 and forwards every packet of that connection to the same backend

   client 198.51.100.7:52000 ═══════ one TCP connection ═══════► backend 1 (10.0.1.11:8080)
                                (passes through the balancer)

 L7
 the balancer ends the client's connection, reads each request,
 and sends each one over a connection of its own choosing

                                          ┌──► pooled connection ──► backend 1  (request 1: /orders)
   client ═══ TCP (+TLS) ═══► [ L7 LB ] ──┤
                                          └──► pooled connection ──► backend 2  (request 2: /users)
```

| | L4 | L7 |
|---|---|---|
| Reads | addresses and ports | the application message: method, path, headers, cookies |
| Unit of balancing | **connection**: every request on a connection goes to the same backend | **request**: two requests on one client connection may go to different backends |
| Connections | one end to end (packet forwarding), or two copied blindly (TCP proxy) | always two: client side and backend side |
| Protocols | any TCP or UDP protocol: PostgreSQL, Redis, TLS passed through unopened | the protocols it can parse: HTTP/1.1, HTTP/2, HTTP/3, gRPC |
| Cost | very cheap per byte | more CPU and memory: TLS termination, parsing, buffering |
| Client address at the backend | preserved by packet forwarding, or passed in a small header (the PROXY protocol) by a TCP proxy | the backend sees the balancer's address; the client's address travels in an HTTP header such as `X-Forwarded-For` (Chapter 10) |

### Guarantees and non-guarantees

An L4 balancer guarantees low overhead and works with any protocol. It does **not** balance requests *within* a connection: one long-lived connection carrying many requests always lands on one backend.

### Failure and diagnosis

**The HTTP/2 and gRPC pinning trap.** A team moves service-to-service calls from HTTP/1.1 to gRPC (which runs over HTTP/2) behind an L4 balancer, such as a Kubernetes `ClusterIP` service, which picks a pod per connection. Under load, one pod runs at 100% CPU while others sit idle, and adding pods does not help. The mechanism: an HTTP/2 client opens one long-lived connection and multiplexes thousands of requests over it (Chapter 7). The L4 balancer made its one decision when that connection's SYN arrived, so every request rides to the same pod. New pods receive nothing, because existing clients never open new connections. Evidence: per-pod request rates are wildly unequal; `ss -tn` on a client shows one connection to the service address. Fixes: balance per request at L7 (a proxy that understands HTTP/2, or a **service mesh**: a set of such proxies running next to every service instance and handling all traffic between services, Chapter 8, Section 8.10), or balance on the client, where the client resolves every pod's address and spreads requests across connections to all of them.

### Backend connection

For every load balancer in front of a service, ask two questions: does it balance per connection or per request? And if it terminates connections, how do its idle timeouts on the client side and the backend side relate to the clients' and backends' own timeouts (Chapter 10 sets out the rules)? The answers explain many imbalance and reset problems before anyone opens a dashboard.

---

## 6.14 Diagnosis: From Error Message to Wire Event

### Why a systematic approach

When a network call fails, the application logs one of a small set of messages: `Connection refused`, `connect timed out`, `Read timed out`, `Connection reset by peer`, `Broken pipe`, `UnknownHostException`, `No route to host`. Each corresponds to a different event on the wire, at a different layer, with different causes. Guessing ("maybe the database is overloaded?" for an `UnknownHostException") wastes the first minutes of an incident. Map the message to the event first, then gather evidence for that event.

### Mechanism: the failure table

| Message (typical exception) | errno | DNS done? | Handshake done? | What happened on the wire | Check first |
|---|---|---|---|---|---|
| Unknown host (`UnknownHostException`; `EAI_NONAME`, `EAI_AGAIN`) | resolver error | **no** | not attempted: no SYN was sent | the resolver returned `NXDOMAIN` or `SERVFAIL`, or did not answer | the name; the record; the resolver's health; `ndots` amplification; a cached `NXDOMAIN` |
| Network is unreachable | `ENETUNREACH` | yes | no: nothing was sent | the local routing table has no route for the destination | `ip route get <ip>`; interface down; missing default route |
| No route to host (`NoRouteToHostException`) | `EHOSTUNREACH` | yes | no | ARP for the destination or gateway failed, or a router or firewall returned ICMP Destination Unreachable | `ip neigh`; a firewall rule that rejects with ICMP; a broken route on the path |
| Cannot assign requested address | `EADDRNOTAVAIL` | yes | no: failed locally | no free ephemeral port for this destination | `TIME_WAIT` counts (Section 6.11) |
| Connection refused (`ConnectException`) | `ECONNREFUSED` | yes | no | a **reset** came back in answer to the SYN, typically within one RTT, from the destination host's kernel (nothing listening on that address and port) **or** from a firewall configured to reject | is the process running? which address and port does it listen on (`ss -ltnp`)? a reject rule? |
| Connect timed out (`SocketTimeoutException: connect timed out`) | `ETIMEDOUT` | yes | **no**: SYNs sent, **nothing** came back | the SYN (or the SYN+ACK) was silently dropped: firewall or security group dropping, a missing return route, a host that is down on a remote subnet; a full SYN queue under attack | firewall and security-group rules; routing; the target's SYN queue and accept queue (Chapter 4, Section 4.6) |
| Read timed out (`SocketTimeoutException: Read timed out`) | (application timer) | yes | **yes** | the connection was open and the request was sent, but the response did not arrive before the client's read timeout | a slow dependency (slow query, lock wait, GC pause, exhausted worker pool); a Path MTU black hole on large messages (Section 6.3); a silently expired NAT entry (Section 6.5) |
| Connection reset by peer | `ECONNRESET` | yes | yes | a reset arrived on an established connection | a pooled connection used after a NAT, load balancer or server idle timeout; a peer process that crashed or was killed; a proxy that closed its side |
| Broken pipe | `EPIPE` | yes | yes (earlier) | the process wrote to a connection that had already been reset (or that it had shut down for writing) | a client or proxy that gave up and closed while the server was still working on the response |

Read the speed of the failure as evidence:

- **`Connection refused` in a millisecond** means a packet came back: routing works, nothing dropped the SYN, and something actively refused it. Most often the destination's kernel had no listener on that address and port. Do not start with DNS or routing; check whether the process is running and what it listens on.
- **A connect timeout after seconds or minutes** means silence: something swallowed the SYN or the reply. Start with firewalls, security groups and routes.
- **A read timeout** means the connection worked. The problem is the peer's speed, or something that breaks only large or late packets.

### Concrete: the tools, annotated

**`ping`: is the host reachable at L3, and what is the RTT?**

```text
$ ping -c 3 10.0.12.50
PING 10.0.12.50 (10.0.12.50) 56(84) bytes of data.
64 bytes from 10.0.12.50: icmp_seq=1 ttl=63 time=0.48 ms
64 bytes from 10.0.12.50: icmp_seq=2 ttl=63 time=0.41 ms
64 bytes from 10.0.12.50: icmp_seq=3 ttl=63 time=0.44 ms

--- 10.0.12.50 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.410/0.443/0.480/0.029 ms
```

`ttl=63` says the reply crossed one router (it left the target at 64). `ping -M do -s <size>` tests the path MTU (Section 6.3). Remember that ICMP may be blocked where TCP is allowed, and the reverse.

**`traceroute`: where along the path does it stop?** Annotated in Section 6.3. Use `traceroute -T -p 443 <host>` to probe with TCP SYNs to the real port when ICMP or UDP probes are filtered.

**`dig`: what does DNS say, and who said it?** Annotated in Section 6.6.

**`ss -tan`: which connections exist, in which state?**

```text
$ ss -tan
State      Recv-Q Send-Q  Local Address:Port    Peer Address:Port
LISTEN     0      4096          0.0.0.0:8080         0.0.0.0:*
ESTAB      0      0           10.0.1.10:8080     10.0.4.21:51432
ESTAB      0      93120       10.0.1.10:49200    10.0.2.99:5432
CLOSE-WAIT 1      0           10.0.1.10:49188   10.0.9.31:443
TIME-WAIT  0      0           10.0.1.10:49152    10.0.2.99:5432
SYN-SENT   0      1           10.0.1.10:49301    10.0.8.20:8080
```

- Row 1: a listening socket (for `LISTEN`, `Recv-Q` is the accept queue's current length and `Send-Q` its limit, Chapter 4, Section 4.6).
- Row 2: a healthy client connection to this server.
- Row 3: 93,120 bytes sent to the database and not yet acknowledged; a large, persistent `Send-Q` means the peer is not reading (its window is closed) or packets are being lost.
- Row 4: the peer closed and this application has not; many such rows mean a socket leak (Section 6.8).
- Row 5: normal leftover from a connection this side closed.
- Row 6: a connection attempt still waiting for a SYN+ACK; many of these mean SYNs are being dropped (a connect timeout is coming).

Filters narrow the view: `ss -tan state time-wait 'dst 10.0.2.99'`, `ss -tan 'sport = :8080'`.

**`ss -ti`: inside one connection.**

```text
$ ss -ti 'dst 10.0.2.99'
ESTAB 0 0 10.0.1.10:49200 10.0.2.99:5432
     cubic wscale:7,7 rto:204 rtt:2.6/1.1 ato:40 mss:1448 pmtu:1500 rcvmss:1448
     advmss:1448 cwnd:10 bytes_sent:184320 bytes_acked:184320 bytes_received:5120
     retrans:0/2 rcv_space:14480
```

`cubic`: the congestion-control algorithm. `wscale:7,7`: both sides scale windows by 2^7. `rto:204`: the current retransmission timeout in ms (near Linux's 200 ms floor on a fast path). `rtt:2.6/1.1`: smoothed RTT and its variation, in ms. `mss:1448`, `pmtu:1500`: segment and path sizes (Sections 6.3 and 6.8). `cwnd:10`: the congestion window in segments. `retrans:0/2`: retransmissions outstanding now / in total. A high `rto` with a small `cwnd` after retransmissions is the signature of loss.

**`tcpdump`: what actually crossed the wire?** A short HTTP exchange with a database-like delay on the server:

```text
$ tcpdump -nn -i eth0 'host 10.0.4.21 and port 8080'
10:20:00.000100 IP 10.0.4.21.51432 > 10.0.1.10.8080: Flags [S], seq 3829101, win 64240,
                options [mss 1460,sackOK,TS val 1 ecr 0,nop,wscale 7], length 0
10:20:00.000180 IP 10.0.1.10.8080 > 10.0.4.21.51432: Flags [S.], seq 771100, ack 3829102, win 65160,
                options [mss 1460,sackOK,TS val 9 ecr 1,nop,wscale 7], length 0
10:20:00.000900 IP 10.0.4.21.51432 > 10.0.1.10.8080: Flags [.], ack 1, win 502, length 0
10:20:00.000950 IP 10.0.4.21.51432 > 10.0.1.10.8080: Flags [P.], seq 1:121, ack 1, win 502, length 120
10:20:00.001700 IP 10.0.1.10.8080 > 10.0.4.21.51432: Flags [.], ack 121, win 509, length 0
10:20:00.013800 IP 10.0.1.10.8080 > 10.0.4.21.51432: Flags [P.], seq 1:351, ack 121, win 509, length 350
10:20:00.014500 IP 10.0.4.21.51432 > 10.0.1.10.8080: Flags [.], ack 351, win 501, length 0
10:20:00.020000 IP 10.0.4.21.51432 > 10.0.1.10.8080: Flags [F.], seq 121, ack 351, win 501, length 0
10:20:00.020600 IP 10.0.1.10.8080 > 10.0.4.21.51432: Flags [F.], seq 351, ack 122, win 509, length 0
10:20:00.021200 IP 10.0.4.21.51432 > 10.0.1.10.8080: Flags [.], ack 352, win 501, length 0
```

How to read it. Flags: `[S]` SYN, `[S.]` SYN+ACK (the `.` means ACK), `[.]` a bare ACK, `[P.]` data with PSH, `[F.]` FIN, `[R]` reset. After the handshake, sequence numbers are shown relative to each side's ISN, so `seq 1:121` means bytes 1 to 120 of the stream, and `length` is the payload size.

- Lines 1–3: the handshake. Line 2 comes 80 µs after line 1, which is roughly the network RTT as seen from the server.
- Line 4: the 120-byte request. Line 5: the server's **kernel** acknowledges it 0.75 ms later. (Section 6.9: this ACK says nothing about the application.)
- Line 6: the 350-byte response, **12.1 ms** after the ACK. That gap is the server application's time (including whatever it waited on), not the network's.
- Lines 8–10: the close, here in three packets because the server sent its FIN together with its ACK.

This one capture separates network time (request to kernel ACK) from application time (kernel ACK to response), which is often the first question in a latency investigation. In a capture of a failure, look for: SYNs with no answer (drops), `[R]` in reply to a SYN (refused), retransmitted segments with the same `seq` (loss), and a window of 0 (`win 0`, a receiver not reading).

### Guarantees and non-guarantees

These tools show what the local host sent and received. They do not show what happened on the other side or in the middle: a capture on one end cannot distinguish a packet dropped on the way out from a reply dropped on the way back. When the evidence is ambiguous, capture on both ends at the same time and compare.

### Backend connection

Every client library and pool should report connect time, time to first byte and total time as separate measurements, with connect timeouts and read timeouts configured separately and explicitly. Then the logs already say which row of the failure table you are in.

---

## Chapter Summary

A network is a stack of layers, each solving one problem behind a contract. The **application layer** defines messages; the **transport layer** delivers bytes between processes, identified by ports; the **network layer** routes **packets** between hosts by IP address; the **link layer** moves **frames** across one local network by MAC address. Sending means **encapsulation** (each layer prepends its header, with a field naming the payload's protocol); receiving means **decapsulation**. OSI layer numbers (L2, L3, L4, L7) are the shared vocabulary for which header a device reads.

On a local network, **switches** learn which MAC address is on which port and forward frames up to the link's **MTU** (1,500 bytes on Ethernet). A host finds the MAC address behind an IP address with **ARP**, which never crosses a router. To send anywhere, a host ANDs addresses with its **CIDR** mask to decide whether the destination is local, and otherwise looks up its **routing table** by **longest-prefix match**, which usually leads to the **default gateway**. At each router, MAC addresses are replaced, IP addresses stay, and **TTL** drops by one, so loops die with an ICMP Time Exceeded, which `traceroute` exploits. TCP avoids fragmentation by **Path MTU Discovery**, which depends on ICMP "Fragmentation Needed" messages; blocking them creates a **black hole** in which small messages work and large ones hang. **DHCP** bootstraps a host's configuration with broadcast **DORA**. Private networks use **RFC 1918** addresses and reach the internet through **NAT**, which rewrites addresses and ports and remembers each flow in a **conntrack** table whose entries expire silently when idle. **Stateful** firewalls allow return traffic automatically; **stateless** ones need explicit rules for it; a **reject** fails fast and a **drop** produces a timeout.

**DNS** turns names into records. A stub resolver asks a recursive resolver, which walks root, TLD and authoritative servers on a cache miss and caches every answer for its **TTL**, including negative answers. Nothing invalidates caches, runtimes add their own, and open connections never look again, so a change takes effect only as all of those expire.

The **4-tuple** identifies a connection, which is why one server port can carry any number of clients but one client address can open only about 28,000 connections to one destination address and port. **TCP** builds a reliable, ordered byte stream on top of IP: a **three-way handshake** exchanges random initial sequence numbers in one round trip; **sequence and cumulative acknowledgement numbers**, SACK, **fast retransmit** and a **retransmission timer** with exponential backoff recover from loss; **flow control** (`rwnd`) protects the receiver and **congestion control** (`cwnd`: slow start, then additive increase and multiplicative decrease, CUBIC or BBR) protects the network. **Nagle** and **delayed ACK** can combine into a 40 ms stall. Closing uses a FIN in each direction; the passive side waits in **`CLOSE_WAIT`** for its application (a growing count is a leak) and the active side holds the 4-tuple in **`TIME_WAIT`** for 2 × MSL so that a lost final ACK can be repeated and old segments cannot leak into a new connection. **RST** aborts. Crucially, TCP has **no message boundaries** (applications must frame), its ACKs come from the **kernel** (they say nothing about processing), and it cannot provide **exactly-once** execution: a timeout leaves the outcome unknown, and only application-level idempotency makes retries safe.

**UDP** offers datagrams with boundaries and nothing else, which suits DNS, media and protocols that bring their own reliability. **QUIC** builds independent encrypted streams on UDP, removing TCP's **head-of-line blocking** between streams, merging the transport and TLS handshakes, and surviving address changes. Performance is governed by **RTT** (bounded by the speed of light), **bandwidth**, the **bandwidth-delay product** that windows must cover, and **loss**, which caps loss-based TCP throughput at roughly MSS / (RTT × √p) and shows up in p99 as retransmission timeouts. **L4** balancers choose per connection and **L7** balancers per request, which is why long-lived HTTP/2 connections pin load behind L4. Finally, every error message (refused, connect timeout, read timeout, reset, unknown host, no route) names a specific event on the wire, which `ping`, `traceroute`, `dig`, `ss` and `tcpdump` can confirm.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Layering | Upper layers unchanged when lower layers change | Invisibility of lower-layer overhead, limits (MTU) or failures |
| Ethernet / switching | Frame delivery within one local network; corruption detected by CRC | Delivery under congestion; retransmission; authenticity of addresses |
| ARP | Maps a local IPv4 address to its current MAC | Anything beyond the local network; protection from forged replies |
| IP | Best-effort, hop-by-hop forwarding by longest-prefix match; loops end via TTL | Delivery, order, uniqueness, or notice of drops |
| ICMP, ping, traceroute | Error reports and probes, where allowed | That a working ping means an open TCP port; that silence means a dead host |
| Path MTU Discovery | Finds the path's smallest MTU when ICMP reaches the sender | Progress when ICMP is blocked (black hole) |
| DHCP | A working configuration without manual setup | The same address forever; protection from rogue servers |
| NAT (SNAT) | Outbound connections for private hosts sharing public addresses | Inbound connections without DNAT; unlimited ports per destination; survival of idle flows |
| Stateful firewall | Return traffic of allowed connections passes automatically | Survival of idle connections after state expiry; immunity to table exhaustion |
| DNS | Answers as published, cached for at most their TTL by compliant caches | Instant change; that runtimes honour TTLs; that open connections follow a change; authenticity (without DNSSEC) |
| 4-tuple | Unlimited clients per server port (within memory and FD limits) | Unlimited connections from one client address to one destination address and port |
| TCP handshake | Both ends are reachable and agree on initial sequence numbers and options | Peer identity (Chapter 8); that the server application has accepted the connection |
| TCP stream and ACKs | In-order, gap-free, duplicate-free bytes between kernels | Message boundaries; that the peer application read or processed anything; exactly-once execution |
| Flow control (`rwnd`) | The sender never overruns the receiver's buffer | Protection of the network (that is `cwnd`) |
| Congestion control (`cwnd`) | Bounded in-flight data; convergence toward a fair share | Full use of long, fast paths with small buffers or random loss |
| TCP keepalive | Eventual detection of a vanished idle peer, if enabled | Fast detection with default timers; liveness of the peer application |
| `TIME_WAIT` | A lost final ACK can be repeated; old segments cannot join a new connection on the same 4-tuple | Unlimited connection churn: it holds the 4-tuple for 60 s on Linux |
| UDP | No setup delay; boundaries of delivered datagrams | Delivery, order, uniqueness, congestion control |
| QUIC | Reliable ordered streams without cross-stream HOL blocking; built-in TLS 1.3; migration across addresses | Freedom from loss; passage through networks that block UDP |
| L4 load balancing | Cheap, protocol-independent distribution of connections | Balanced requests within long-lived multiplexed connections |

---

## Interview Drill

**Q1. What happens at each layer when an application sends a message? How do the TCP/IP and OSI models relate?**
A strong answer must mention:
- TCP/IP is the four-layer model actually implemented (application, transport, internet, link); OSI is a seven-layer reference model whose numbers (L2, L3, L4, L7) survive as vocabulary.
- The units: message → TCP segment (or UDP datagram) → IP packet → Ethernet frame.
- What each header adds: ports, sequence and acknowledgement numbers, window (TCP); addresses, TTL, protocol (IP); MAC addresses, EtherType, and a CRC trailer (Ethernet).
- Demultiplexing at the receiver by EtherType, then IP Protocol, then the 4-tuple.
*Common weak answer:* reciting the seven OSI names. It names no header, address or unit, so it shows memory rather than understanding.

**Q2. Why do we need both MAC and IP addresses? How does ARP work, and what happens to each address at a router?**
A strong answer must mention:
- IP addresses are hierarchical locations that can be grouped into prefixes for routing; MAC addresses are flat interface names used on one local network.
- ARP: a broadcast "who has X?", a unicast reply, a cache entry; it never crosses a router.
- For a remote destination the host ARPs for its gateway, not for the destination.
- At every router the MAC addresses are replaced; the IP addresses stay (barring NAT) and TTL decreases.
*Common weak answer:* "ARP finds the destination server's MAC." For a remote server the host never learns that MAC; it uses the gateway's.

**Q3. A host is configured as `10.0.5.25/22` with gateway `10.0.4.1`. How does it send to `10.0.6.100`, and to `10.0.12.50`?**
A strong answer must mention:
- `/22` means a 22-bit network prefix; the mask is `255.255.252.0`, and the host's network is `10.0.4.0/22` (`10.0.4.0`–`10.0.7.255`).
- `10.0.6.100` is inside that block: deliver directly, ARP for `10.0.6.100`.
- `10.0.12.50` is outside (its prefix is `10.0.12.0/22`): longest-prefix match finds the default route (or a more specific route if there is one); the IP header carries `10.0.12.50`, the frame carries the gateway's MAC.
- Longest-prefix match picks the most specific of several matching routes.
*Common weak answer:* comparing third bytes. That works only for `/24`.

**Q4. How does `traceroute` work, and what is a Path MTU black hole?**
A strong answer must mention:
- TTL is a hop counter; the router that decrements it to 0 drops the packet and returns ICMP Time Exceeded from its own address.
- `traceroute` sends probes with TTL 1, 2, 3, ... so each router in turn reveals itself; `* * *` only means a router does not answer.
- TCP sets DF and relies on ICMP "Fragmentation Needed" (carrying the next-hop MTU) to learn the path MTU.
- If a firewall drops that ICMP, small packets pass and full-size ones vanish: the connection opens, small requests work, large transfers hang. Prove it with `ping -M do -s`; fix by allowing the ICMP, MSS clamping, or correct tunnel MTUs.
*Common weak answer:* "TTL is how many seconds a packet may live." It is a hop count that each router decrements; no router measures time, which is exactly why `traceroute` can use it to probe one hop at a time.

**Q5. How does a newly booted machine get an IP address? Why four messages?**
A strong answer must mention:
- DHCP over UDP 67/68 with broadcast at both layers and source `0.0.0.0`, the client identified by its MAC.
- Discover, Offer, Request, Ack.
- Several servers may offer; the broadcast Request tells the chosen one to commit and the others to release their offers.
- Leases renew at T1 (50%, unicast) and T2 (87.5%, broadcast); a `169.254.x.y` address means no server answered.
*Common weak answer:* listing the four messages without explaining how a host with no address can send at all. Broadcast addressing (source `0.0.0.0`, destination `255.255.255.255`, the client identified by its MAC) is the heart of the mechanism.

**Q6. What is NAT, and how do hundreds of private servers share one public IP address?**
A strong answer must mention:
- RFC 1918 ranges are not routed on the internet.
- SNAT with port translation: the gateway rewrites the source address *and port*, records the mapping in a connection-tracking table, and reverses it for replies; checksums are recomputed.
- The port is what separates two inside hosts talking to the same destination.
- Consequences: no unsolicited inbound connections without DNAT; a limited number of ports per destination; idle entries expire silently, so pooled connections need idle eviction or keepalives shorter than the timeout.
*Common weak answer:* "NAT replaces the private IP with a public one." Without port translation, the gateway could not route replies for two hosts talking to the same server.

**Q7. Compare stateless and stateful firewalls, and reject with drop.**
A strong answer must mention:
- Stateless: each packet judged alone; return traffic needs a rule for the clients' ephemeral ports.
- Stateful: tracks connections; once the first packet is allowed, the connection's packets pass both ways; the state table can fill.
- Reject: a reset or ICMP error comes back, so the caller fails within one RTT.
- Drop: silence, so the caller retransmits its SYN until its connect timeout expires (about two minutes with Linux defaults if the application sets none), tying up threads.
*Common weak answer:* "security groups are stateful" without explaining what breaks on a stateless filter or how a drop shows up in callers. The label does not help in an incident; the consequences (missing return rules, callers hung in `connect`) are what you debug.

**Q8. Walk through a DNS lookup of `api.example.com`. What is the difference between a recursive resolver and an authoritative server? Why is "DNS propagation" misleading?**
A strong answer must mention:
- Local checks first (`/etc/hosts`, OS and runtime caches); then the stub sends a recursive query to the configured resolver.
- On a miss, the resolver queries iteratively: root → referral to `.com` → referral to `example.com`'s servers → answer; it follows `CNAME`s.
- The authoritative server holds the zone's records; the resolver only caches them for their TTL, including negative answers.
- Nothing pushes changes: caches at every level expire on their own, runtimes may cache longer, and open connections never re-resolve. So lower TTLs in advance and keep the old target running.
*Common weak answer:* "the application asks the root servers", or "changing the record moves traffic at once". The stub only asks a resolver, and caches at every level, plus connections that are already open, keep using old answers.

**Q9. How can one server on port 443 hold 50,000 connections? Can one client open 50,000 connections to one database?**
A strong answer must mention:
- Connections are identified by the full 4-tuple, not the server port.
- Each client brings its own address and port, so a server port has no 65,535 limit; memory and file descriptors are the limits.
- One client address to one database address and port varies only the source port: about 28,000 with Linux defaults, including connections in `TIME_WAIT`.
- More capacity needs more source addresses, destinations, or (better) pooling.
*Common weak answer:* "a server can only handle 65,535 connections." The 16-bit limit bites only when three of the four tuple fields are fixed, which is the situation of a client, not of a server.

**Q10. Explain the three-way handshake with sequence numbers. Why three messages, and why random initial numbers?**
A strong answer must mention:
- SYN (seq = x); SYN+ACK (seq = y, ack = x + 1); ACK (ack = y + 1); the SYN consumes one sequence number.
- `connect` returns after one RTT; the client can send data right after its ACK; the server's connection moves from the SYN queue to the accept queue.
- Three because each side must choose a number and learn that the other received it, and because the third message defeats old duplicate SYNs.
- Random ISNs keep old segments out of new connections and stop blind injection by off-path attackers.
*Common weak answer:* drawing three arrows labelled SYN, SYN-ACK, ACK with no numbers and no reason. Without the sequence numbers there is nothing to show why two messages would not be enough.

**Q11. Walk through the close. What are `TIME_WAIT` and `CLOSE_WAIT`, and why does `TIME_WAIT` last 2 × MSL?**
A strong answer must mention:
- FIN → ACK → FIN → ACK (each direction closed separately); the states FIN_WAIT_1, FIN_WAIT_2, TIME_WAIT on the active side and CLOSE_WAIT, LAST_ACK on the passive side.
- `CLOSE_WAIT`: the peer closed and the local application has not called `close`; accumulation is a socket leak in your code.
- `TIME_WAIT`: normal on the side that closed first, 60 s on Linux; reason 1, a lost final ACK must be repeatable when the peer retransmits its FIN; reason 2, delayed segments from the old connection must die before the 4-tuple is reused.
- `TIME_WAIT` holds the 4-tuple, which causes port exhaustion under churn.
*Common weak answer:* treating `TIME_WAIT` as a bug, or giving only one of the two reasons. It is required protocol behaviour; the real bug is the connection churn that creates tens of thousands of entries.

**Q12. What is the difference between flow control and congestion control? How do slow start and congestion avoidance work?**
A strong answer must mention:
- Flow control protects the receiver's buffer through the advertised `rwnd` (down to a zero window when the application stops reading).
- Congestion control protects the network through the sender's private `cwnd`; in-flight data ≤ min(`rwnd`, `cwnd`).
- Slow start: begin at about 10 segments, +1 per ACK, doubling per RTT until `ssthresh` or loss; then congestion avoidance: about +1 segment per RTT (Reno) or a cubic curve (CUBIC).
- Three duplicate ACKs: fast retransmit and a multiplicative cut (half in Reno, to 70% in CUBIC); a timeout: `cwnd` back to 1 and slow start again. BBR paces from measured bandwidth and RTT instead.
*Common weak answer:* "both slow the sender down", without saying who is protected or where the state lives. The two fail for different reasons (a slow reader versus a congested link) and are diagnosed with different evidence.

**Q13. What does TCP not guarantee? Why can't it give message boundaries or exactly-once payments?**
A strong answer must mention:
- It is a byte stream: writes can be merged or split, so protocols need framing (length prefix, declared length, delimiter).
- `write` returning means "in my send buffer"; the peer's ACK means "in the peer kernel's buffer"; neither proves the application processed anything.
- A timeout or reset after sending leaves the outcome unknown; retrying gives at-least-once, not retrying gives at-most-once.
- Exactly-once effect is an application property: idempotency keys stored atomically with the change (Chapter 13).
*Common weak answer:* "TCP is reliable, so if `write` succeeds the server got the request." `write` only reached the local send buffer, and even the peer's ACK comes from its kernel, not from its application.

**Q14. Compare TCP and UDP. What is head-of-line blocking, and how does QUIC address it?**
A strong answer must mention:
- UDP: no setup, datagrams with boundaries, no reliability, ordering or congestion control; fits DNS, media, and protocols that bring their own reliability.
- HOL blocking: over one TCP connection, one lost segment holds back all later bytes, including other requests' bytes already received.
- QUIC runs over UDP, numbers packets and tracks each stream separately, so a loss stalls only its own stream; it still shares congestion control across streams.
- QUIC also combines transport and TLS 1.3 setup (one round trip), uses connection IDs to survive address changes, and runs in user space over UDP because middleboxes block new protocols.
*Common weak answer:* "HTTP/3 is faster because UDP is faster than TCP." QUIC re-implements reliability and congestion control; its gains come from stream independence and fewer handshakes.

**Q15. What is ephemeral port exhaustion? Do the arithmetic and give the fix.**
A strong answer must mention:
- Outgoing connections from one address to one destination address and port differ only in the source port, drawn from about 28,000 values on Linux.
- A client that opens and closes a connection per request is the active closer and holds each 4-tuple in `TIME_WAIT` for 60 s.
- Sustained rate ≈ 28,232 / 60 ≈ 470 new connections per second; beyond it `connect` fails with `EADDRNOTAVAIL`.
- The fix is connection reuse (pools, keep-alive); a wider port range, more addresses or kernel reuse settings only raise the ceiling.
*Common weak answer:* confusing it with file-descriptor exhaustion (`EMFILE`) or with `EADDRINUSE` on a server's listening port. Those produce different errors and need different fixes; port exhaustion is a client-side shortage of 4-tuples.

**Q16. Why might a 1 Gb/s link with an 80 ms RTT carry only a few Mb/s on one TCP connection, and why does 1% loss hurt so much?**
A strong answer must mention:
- BDP = 1 Gb/s × 0.08 s = 10 MB must be in flight to fill the link.
- A 64 KB window caps throughput at window / RTT ≈ 6.5 Mb/s; window scaling and large socket buffers are needed.
- Loss-based congestion control cuts `cwnd` on every loss, so throughput ≈ (MSS / RTT) × 1.22 / √p: about 2.8 Mb/s at 1% loss and 50 ms.
- For small requests, loss shows up in p99 as retransmission timeouts of 200 ms or more (1 s for a lost SYN), while the median stays normal.
*Common weak answer:* "1% loss costs 1% of throughput." Every loss also cuts the congestion window, so throughput falls with the square root of the loss rate, not in proportion to it.

**Q17. What is the difference between L4 and L7 load balancing, and why does L4 break gRPC load distribution?**
A strong answer must mention:
- L4 decides per connection from addresses and ports, without parsing the payload; L7 terminates the connection, parses requests, and decides per request.
- L7 can route on path, host and headers, and sees the client address only via headers; L4 works for any protocol at lower cost.
- HTTP/2 and gRPC multiplex many requests over one long-lived connection, so an L4 balancer sends them all to the backend chosen at the SYN; new backends get nothing.
- Fix: L7 balancing that understands HTTP/2, or client-side balancing across all backend addresses.
*Common weak answer:* "L4 is IP and port, L7 is HTTP", without the per-connection versus per-request consequence. That consequence is exactly what produces the gRPC imbalance.

**Q18. Why does one connection attempt fail instantly with "connection refused" and another hang until a connect timeout? What do a read timeout and a reset mean?**
A strong answer must mention:
- Refused: a reset came back in reply to the SYN within about one RTT, so the path works and the destination (most often its kernel, with nothing listening; sometimes a rejecting firewall) actively refused.
- Connect timeout: SYNs went out and nothing came back: a dropping firewall or security group, a routing problem, or an unreachable host; the client retransmits with backoff until its timeout.
- Read timeout: the connection was established and the request sent, but no response came in time: a slow peer, a Path MTU black hole, or an expired NAT entry.
- Reset on an established connection: the peer or a middlebox aborted it, typically a stale pooled connection or a crashed peer; unknown host means DNS failed and no SYN was ever sent.
*Common weak answer:* treating all of these as "network timeouts" and responding to each the same way. Each one points to a different layer and a different first check.

---

## Scenario Exercises

1. Immediately after a deployment of the inventory service, every call from the checkout service to `10.0.2.50:8080` fails in about 0.8 ms with `Connection refused`. A colleague says someone must have broken a security group or a route table during the deploy. Why is that unlikely, what happened on the wire, and what are the two most likely causes on the inventory host?
2. A service calls a partner API through an IPsec VPN tunnel. Health checks and small requests always succeed in 15 ms. Orders with more than ten line items (a 2,200-byte request body) hang for 30 seconds and fail with a read timeout. Explain the mechanism, show how to prove it from the command line in under a minute, and give two fixes.
3. A new aggregator service makes three outbound HTTP calls per incoming request, each on a fresh connection that it closes afterwards, to one internal service at `10.0.3.15:8080`. At 600 incoming requests per second it runs well for about 16 seconds, then nearly every outbound call fails with `connect EADDRNOTAVAIL`. CPU and memory are under 25% on both services. Explain the 16 seconds (assume one source address, an initially free default Linux port range, the aggregator closing first, and no `TIME_WAIT` reuse), and the fix.
4. You must move `api.example.com` from `198.51.100.10` to a new load balancer at `203.0.113.50`. The `A` record's TTL is 86,400 seconds. A colleague plans to change the record at 02:00 and switch off the old server at 02:15. What will happen? Write a step-by-step plan with a timeline that avoids dropped traffic.
5. A team moves a high-traffic internal service from JSON over HTTP/1.1 to gRPC on 12 Kubernetes pods behind a standard `ClusterIP` service. During a spike, the autoscaler adds 12 pods, and none of the new pods receives any traffic from existing callers. Explain why at the TCP and HTTP/2 level, and give two fixes.
6. An internal RPC client sends each request as a 64-byte header `write` followed by a 120-byte body `write`, then waits for a 32-byte reply. Over loopback one thread achieves 5,000 requests per second. Between two machines in the same rack (`ping` RTT 0.3 ms), with `TCP_NODELAY` not set, one thread achieves about 24 requests per second at 0% CPU and no loss. Explain the stall and give two fixes.
7. A service's median latency is 3 ms and its p99 is 205 ms. CPU is at 20% and the database's own query times are flat. Give three hypotheses and the evidence that would separate them.

---

## Diagnosis Lab

**Lab 1: The firewall change that turned fast failures into an outage**
- *Symptom:* An order service with 64 worker threads calls a non-critical loyalty service on each checkout and continues without loyalty points if the call fails. At 02:00 the loyalty service is stopped for maintenance; the order service logs `Connection refused` in about 0.5 ms per call and keeps serving 500 checkouts per second with no user impact. At 02:15 a network engineer changes the loyalty service's firewall rule to drop traffic during the maintenance. Within a second, the order service stops answering all requests.
- *Hypotheses:* (a) the order service's database deadlocked; (b) the drop turned a sub-millisecond refusal into a long `connect` hang that occupied every worker thread; (c) DNS started failing.
- *Evidence:* A thread dump (Chapter 5, Section 5.14) shows all 64 workers inside the HTTP client's `connect` to the loyalty service's address. `ss -tan state syn-sent` shows 64 sockets in `SYN-SENT` toward `10.0.8.20:8080`. `tcpdump -nn host 10.0.8.20` shows `Flags [S]` retransmitted at growing intervals (1 s, then 2 s, then 4 s) with no reply. The loyalty client's configuration sets a 2-second *read* timeout but no *connect* timeout, so the kernel's default (about two minutes) applies. Database and DNS metrics are normal.
- *Root cause:* While the loyalty process was down but its host was up, the host's kernel answered each SYN with a reset: a fast, cheap failure. The firewall drop made each SYN disappear instead. At 500 requests per second, 64 threads are all stuck in `connect` within about 64 / 500 ≈ 0.13 s, and every other request then queues behind them.
- *Fix:* Set an explicit connect timeout on every client (a few hundred milliseconds is typical inside one data centre), alongside the read timeout; protect non-critical dependencies with a circuit breaker and a bulkhead (Chapter 24) so they cannot take all the threads; prefer reject over drop for internal rules where fast failure is wanted.
- *Prevention:* A code-review or configuration check that every outbound client sets both timeouts; an alert on the number of sockets in `SYN-SENT`.

**Lab 2: Requests that sometimes take exactly five seconds longer**
- *Symptom:* A payment service on Kubernetes has a median latency of 25 ms and a p95 of 45 ms, but its p99 is about 5,030 ms. Traces show the extra time is spent *before* the first byte of the request to `api.partner.com` is written.
- *Hypotheses:* (a) a lost SYN (contradicted: the first SYN retransmission comes after 1 s, so a single loss would add about 1 s, not 5 s); (b) a DNS query that went unanswered (the C library's stub resolver waits 5 seconds before retrying); (c) garbage-collection pauses (contradicted: GC logs show pauses under 10 ms).
- *Evidence:* `tcpdump -nn -i eth0 port 53` inside the pod shows 8 to 10 DNS packets for every outbound request: `api.partner.com.prod.svc.cluster.local`, then the other search suffixes, each for `A` and `AAAA`, because of `ndots:5` (Section 6.6). The HTTP client creates a new connection, and therefore a new lookup, for most requests. Occasionally a query receives no answer; the next query for the same name leaves the pod exactly 5 seconds later. On the node, `nstat` shows `UdpRcvbufErrors` rising on the cluster DNS server's host during traffic peaks, and its metrics show it near its capacity.
- *Root cause:* Lookups multiplied by the search list, combined with unpooled connections, overloaded the cluster DNS servers. Each dropped query cost the stub resolver's full 5-second retry timeout.
- *Fix:* Reuse connections so that lookups happen rarely; set `ndots` to 1 or 2 (or use fully qualified names with a trailing dot) for external names; run a DNS cache on each node; shorten the resolver timeout and allow retries (`options timeout:1 attempts:2` in the pod's DNS configuration).
- *Prevention:* Measure DNS time separately from connect and request time in client metrics; alert on DNS server drops and on the rate of DNS queries per outbound request.

**Lab 3: The first payment after a quiet period fails**
- *Symptom:* A payment service in a private subnet keeps a pool of 20 persistent HTTPS connections to an external bank API, through a NAT gateway. During the day the error rate is zero. At night, when a payment arrives only every 10–15 minutes, the first payment after each quiet gap often fails, either at once with `Connection reset` or after 30 seconds with `Read timed out`. An immediate retry always succeeds.
- *Hypotheses:* (a) the bank API is unreliable at night; (b) pooled connections are being silently expired by a middlebox while idle; (c) the bank closes idle connections and the pool does not notice.
- *Evidence:* The failures happen only on connections that were idle for more than about six minutes, never on fresh ones. The NAT gateway's documentation gives an idle timeout of 350 seconds. A capture on the service host during a failure shows the TLS record of the request leaving, followed either by a reset from the outside or by nothing; in neither case does a FIN from the bank appear earlier (which rules out (c), where the pool would have seen the bank close the connection). The bank's status page shows no incidents.
- *Root cause:* After 350 idle seconds the NAT gateway deleted each connection's entry without telling either end. The next request on a "live" pooled connection reached the gateway with no matching entry; depending on the path, it was dropped (read timeout) or answered with a reset. The retry opened a new connection and a new entry.
- *Fix:* Configure the pool to close connections idle for longer than, say, 240 seconds (comfortably below 350), or enable TCP keepalive with an idle time of about 60–120 seconds so that the entries stay fresh. Also make the request idempotent with an idempotency key (Chapter 13), because a read timeout on a payment leaves its outcome unknown (Section 6.9).
- *Prevention:* Record the idle timeout of every NAT gateway, load balancer and firewall on each outbound path, and derive pool idle limits from the smallest one.

**Lab 4: Cross-region replication stuck at 15 Mb/s on a 10 Gb/s link**
- *Symptom:* A disaster-recovery job streams snapshot files over one TCP connection from Tokyo to Frankfurt (RTT 200 ms) across a dedicated 10 Gb/s link with no measurable loss. Throughput never exceeds about 15 Mb/s.
- *Hypotheses:* (a) disk throughput on either end; (b) a window smaller than the bandwidth-delay product; (c) loss on the link.
- *Evidence:* `iostat -x 1` (Chapter 4, Section 4.12) shows both disks under 2% busy. `ss -ti` on the sender shows `retrans:0/0` (no loss), `wscale:3,7` (the first number is the shift the *receiver* announced: only 3), and a congestion window large enough not to be the limit. A capture of the receiver's ACKs shows its advertised window stuck at 393,216 bytes once scaling is applied. On the receiver, `sysctl net.ipv4.tcp_rmem` shows a maximum of 393,216 bytes, set years ago by a tuning script.
- *Root cause:* The receive window limits the transfer: 393,216 bytes / 0.2 s ≈ 1.97 MB/s ≈ 15.7 Mb/s. The path's BDP is 10 Gb/s × 0.2 s = 2 Gbit = 250 MB, more than 600 times the window.
- *Fix:* Raise the receive (and send) buffer limits so that autotuning can grow the window toward the BDP, within a memory budget per connection; reconnect, because the window scale is fixed at the handshake: Linux chooses the shift it announces from its buffer limit, and with shift 3 the window field cannot express more than about 512 KB (65,535 × 8). New connections made after the change announce a larger shift. If one connection's memory cannot be that large, use several parallel streams. Then re-measure: CPU, disks or the congestion window may become the next limit.
- *Prevention:* Compute the BDP when designing any long-distance bulk transfer, and check buffer limits on both ends before blaming the link.

---

## Solution Sketches

1. A security group that dropped the SYN, or a broken route, would produce silence and a connect timeout after seconds, or an ICMP "no route" error; it would not produce a reset in 0.8 ms (about one local round trip). The fast refusal shows that DNS, routing and the firewall all worked and that the host `10.0.2.50` answered the SYN with a reset because nothing was listening on that address and port. Most likely causes: the new version crashed at startup or has not finished starting (no `listen` yet), or it now binds to `127.0.0.1:8080` (or another port) instead of `0.0.0.0:8080`. Check `ss -ltnp` and the process status on the host.
2. A Path MTU black hole. The tunnel reduces the path MTU to about 1,400 bytes. Small messages fit in small packets; the 2,200-byte body produces a full 1,500-byte packet with DF set, which the tunnel router drops, and its ICMP "Fragmentation Needed" is blocked by a firewall, so the sender keeps retransmitting the same too-large segment until the read timeout. Proof: `ping -M do -s 1472 <partner>` fails while `ping -M do -s 1372 <partner>` succeeds (or prints `Frag needed and DF set (mtu = 1400)` if the ICMP does arrive). Fixes: allow ICMP type 3 code 4 (and ICMPv6 Packet Too Big); clamp the MSS on the VPN gateway; or enable `net.ipv4.tcp_mtu_probing` on the senders.
3. 600 requests/s × 3 = 1,800 new connections per second to one destination. Each closed connection stays in `TIME_WAIT` for 60 s, so none are recycled during the first minute; 28,232 ports / 1,800 per second ≈ 15.7 s until no port is free, after which `connect` fails with `EADDRNOTAVAIL`. Fix: share one client with a keep-alive connection pool, so that a few dozen connections carry all the calls; size its maximum connections for the needed concurrency.
4. Resolvers and runtimes that looked up the name in the previous 24 hours keep `198.51.100.10` for up to 24 hours, so switching it off at 02:15 causes hours of failures for many clients. Plan: at least 24 hours before (one old TTL), lower the TTL to 60 s, and confirm that the authoritative servers serve the new TTL. At the cutover, verify the new load balancer is healthy, then change the record. Keep the old server running (or proxying to the new one) and watch its traffic; it should fall within minutes, with a tail from clients that cache longer and from long-lived connections. Retire it only after its traffic has stayed at zero for longer than the longest expected connection lifetime, with a rollback plan ready. Then raise the TTL again.
5. HTTP/2 (and therefore gRPC) sends all of a client's calls over one long-lived connection. A `ClusterIP` service balances at L4, choosing a pod once per connection, when the SYN arrives. Existing clients never open new connections, so the new pods are never chosen. Fixes: an L7 proxy or service mesh that balances individual HTTP/2 requests across pods; or client-side balancing, with a headless service whose DNS returns every pod's address and a client that keeps connections to all of them and spreads calls across them. (Periodically recycling client connections helps, but only partially.)
6. Nagle plus delayed ACK. The 64-byte header leaves immediately. The server cannot reply until it has the body, so it delays its ACK, by about 40 ms here. The 120-byte body is smaller than one MSS and the header is unacknowledged, so Nagle holds it. When the delayed-ACK timer fires, the body is released and the exchange finishes: about 41 ms per request, so about 24 requests per second. Loopback behaves differently (very large MTU, different ACK timing), which hid the problem. Fixes: assemble header and body in one buffer and send them with one `write`; set `TCP_NODELAY` on the socket.
7. A p99 just above 200 ms, with a normal median and low CPU, matches Linux's minimum retransmission timeout. Hypotheses: (a) packet loss on some path, so a few requests wait for a retransmission timer: check `nstat` retransmission counters, `ss -ti` per-connection `retrans`, NIC drop counters, and compare p99 across instances and paths; (b) a periodic stall in the service (a GC pause, a lock, a cron job) of about 200 ms: check GC logs and whether slow requests cluster in time; (c) a slow dependency other than the database (a cache, DNS, an external API) for a small fraction of requests: check per-dependency timings in traces. Loss produces slow requests spread randomly across time and endpoints, with retransmissions visible in captures; a stall produces bursts of slow requests at the same moments; a slow dependency appears only on the endpoints that call it.

---

**END OF CHAPTER 6**
