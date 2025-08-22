---
layout: post
title: "Building a Custom Ping Tool with Python (ICMP & Raw Sockets)"
date: 2025-08-22
tags: [python, networking, icmp, ping, sockets]
---

> To understand the following, basic knowledge of the **OSI Model** and **Python** is essential.

---

## Table of Contents
1. [What is ICMP](#what-is-icmp)  
2. [What is TTL](#what-is-ttl)  
3. [Raw Sockets](#raw-sockets)  
4. [Sending ICMP Echo Request](#sending-icmp-echo-request)  
5. [Capturing ICMP Echo Reply](#capturing-icmp-echo-reply)  
6. [GitHub Project Structure](#github-project-structure)  

---

## What is ICMP
We’ll be focusing on **ICMPv4**, which uses IPv4.  

ICMP is an **RFC standard network protocol** for checking and diagnosing network connectivity.  

Popular tools that use ICMP:
1. `ping`  
2. `traceroute`  

Both use the same logic — the difference is in how **TTL** works.  

---

## What is TTL
TTL (*Time To Live*) is a **field in the IP header** (not the ICMP header).  

- Prevents routing loops.  
- Each router/firewall **decrements TTL by 1** when forwarding a packet.  
- If TTL reaches 0, the packet is dropped.  

👉 This ensures routers don’t waste CPU endlessly forwarding packets.  

### Key facts:
- TTL is 1 byte (8 bits) → max value = 255.  
- Only **Layer 3 devices** (routers, firewalls) decrement TTL — **switches do not**.  
- Default values by OS:  
  - Linux: 64  
  - Windows: 128  
  - Cisco Routers: 255  

Since ICMP runs on IP and **not TCP/UDP**, it is considered a **network layer protocol**.  

---

## Raw Sockets
To send and receive ICMP packets in Python, we need a **raw socket**:  

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP)
```

We’ll use `sock` to send ICMP requests and capture replies.  

---

## Sending ICMP Echo Request

```python
import os, time, struct

ICMP_ECHO_REQUEST = 8  # defined in icmp_constants.py

def build_icmp_request(identifier, sequence):
    icmp_type = ICMP_ECHO_REQUEST
    code = 0
    checksum_icmp = 0

    header = struct.pack("!BBHHH", icmp_type, code, checksum_icmp, identifier, sequence)
    send_time = time.time()
    data = struct.pack("!d", send_time) + b"ping_payload"

    # calculate checksum
    checksum_icmp = checksum(header + data)
    header = struct.pack("!BBHHH", icmp_type, code, checksum_icmp, identifier, sequence)

    return header + data
```

Here’s the **checksum function**:  

```python
def checksum(data):
    s = 0
    for i in range(0, len(data) - len(data) % 2, 2):
        w = (data[i] << 8) + data[i + 1]
        s += w
    if len(data) % 2:
        s += data[-1] << 8
    s = (s >> 16) + (s & 0xFFFF)
    s += (s >> 16)
    return ~s & 0xFFFF
```

---

### Sending the packet
```python
def send_icmp_request(sock, ip, sequence=0):
    identifier = os.getpid() & 0xFFFF
    icmp_packet = build_icmp_request(identifier, sequence)
    timestamp = struct.unpack("!d", icmp_packet[8:16])[0]

    try:
        sock.sendto(icmp_packet, (ip, 0))
    except OSError as e:
        print(f"ERROR OCCURRED: {e}")
    except Exception as e:
        print(f"EXCEPTION: {e}")

    return identifier, sequence, timestamp, ip
```

---

## Capturing ICMP Echo Reply
We use `recvfrom()` to capture responses:

```python
data, addr = sock.recvfrom(1024)

return {
    "src_ip": src_ip,
    "ttl": ttl,
    "icmp_type": icmp_type,
    "icmp_code": icmp_code,
    "identifier": icmp_identifier,
    "sequence": icmp_sequence,
    "start_time": start_time,
    "reply_time": reply_time,
}
```

---

## GitHub Project Structure

```
Mping/
├── .gitignore
├── icmp_constants.py
├── icmp_network.py
├── icmp_parser.py
├── icmp_utils.py
├── mping.py
├── ping_output.py
└── ping_stats.py
```

🔗 **GitHub Repo:** [melvinjosemt/myping](https://github.com/melvinjosemt/myping)
