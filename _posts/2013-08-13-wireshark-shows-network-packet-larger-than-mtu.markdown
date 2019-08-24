---
layout: post
title:  "Wireshark Shows Network Packet Larger than MTU"
date:   2013-08-13 16:17:22 +0800
---

While working on my thesis I came across a weird problem, wireshark was showing packet size more than the MTU. One of the possible reason can be underlying network supports jumbo frames. But it wasn't so in my case. 

After some googling I found the cause, Large segmentation offload (LSO) performed by NIC. LSO technique  increases the outbound throughput of high-bandwidth networks by offloading packet processing time from CPU to Network Interface Tag (NIC). When LSO is applied to TCP, it is called TCP Segmentation Offload (TSO). When LSO is applied on TCP, it is called as TCP Segmentation Offload (TSO).

The working of TSO can be explained with the help of an example. Let a unit of 65,536 bytes is to be transmitted by the host device. Assuming MTU of 1500 bytes, this data will be divided into 46 segments of 1448 bytes each before it is transmitted over to network through the NIC. Process of dividing the data into segments before sending it over the network is handed over to NIC instead of CPU. NIC will break down the data into smaller segments, and add corresponding TCP, IP and data link layer protocol headers. This significantly reduces the work done by the CPU. Large Receive Offload (LRO) is a similar technique to LSO, but applied for incoming traffic. 

```bash
ethtool -k <interface> shows the status of  LSO and LRO.  
```
To turn off the segmentation offload and to see packet size of <= MTU in wireshark, use: 

```bash
ethtool -K <interface> tso off and ethtool -K <interface> gso off  ```
