---
title:  tcpdump
layout: default
---

# tcpdump

[tcpdump][tcpdump]: Utility for inspecting network packet traffic.

## `tcpdump` Parameters

| Flag   | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `-nn`  | Tells tcpdump to: <br/>1. Not resolve hostnames <br/>2. Not resolve port names.<br/><br/>Why: To help speed up the capture process by avoiding DNS and service name lookups.                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `-i`   | (--interface) Specifies the NIC tcpdump should listen on. (i.e., eth0, ensp05, etc.)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `-w`   | Specifies that tcpdump should write the packets to file rather than printing them out to stdout.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `-v`   | When parsing and printing, produce (slightly more) verbose output. For example, the time to live, identification, total length and options in an IP packet are printed. Also enables additional packet integrity checks such as verifying the IP and ICMP header checksum. When writing to a file with the `-w` option and at the same time not reading from a file with the `-r` option, report to stderr, once per second, the number of packets captured. In Solaris, FreeBSD and possibly other operating systems this periodic update currently can cause loss of captured packets on their way from the kernel to tcpdump. |
| `-vv`  | Even more verbose output. For example, additional fields are printed from NFS reply packets, and SMB packets are fully decoded.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `-vvv` | Even more verbose output. For example, telnet SB...SE options are printed in full. With `-X` telnet options are printed in hex as well.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `-x`   | When parsing and printing, in addition to printing the headers of each packet, print the data of each packet (minus its link level header) in hex. The smaller of the entire packet or snaplen bytes will be printed. Note that this is the entire link-layer packet, so for link layers that pad (e.g. Ethernet), the padding bytes will also be printed when the higher layer packet is shorter than the required padding. In the current implementation this flag may have the same effect as `-xx` if the packet is truncated.                                                                                               |
| `-xx`  | When parsing and printing, in addition to printing the headers of each packet, print the data of each packet, including its link level header, in hex.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `-X`   | When parsing and printing, in addition to printing the headers of each packet, print the data of each packet (minus its link level header) in hex and ASCII. This is very handy for analysing new protocols. In the current implementation this flag may have the same effect as `-XX` if the packet is truncated.                                                                                                                                                                                                                                                                                                               |
| `-XX`  | When parsing and printing, in addition to printing the headers of each packet, print the data of each packet, including its link level header, in hex and ASCII.                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |


### `tcpdump` Examples:

#### Capture All Traffic on a Specific Interface

```shell
tcpdump -i eth0
```

#### Capture and Display the Traffic in Real-Time

```shell
tcpdump -i any -c 100
```

#### Save Captured Packets to a File

```shell
tcpdump -i eth0 -w myfile.pcap
```

#### Read Packets from a File
```shell
tcpdump -r myfile.pcap
```

#### Capture Only IP Packets
```shell
tcpdump -i eth0 ip
```

#### Capture Traffic to or from a Specific IP
```shell
tcpdump -i eth0 host 192.168.1.1
```

#### Capture All TCP Traffic on a Specific Port
```shell
tcpdump -i eth0 tcp port 80
```

#### Capture Traffic Based on Source and Destination
```shell
tcpdump -i eth0 src 192.168.1.1 and dst port 22
```

#### Capture with Verbose Output
```shell
tcpdump -i eth0 -v
```

#### Capture Packets of a Specific Protocol
```shell
tcpdump -i eth0 icmp
```


[tcpdump]: https://www.tcpdump.org/manpages/tcpdump.1.html