---
title: "Understanding ICMP and TTL with Python"
date: 2025-08-21
categories: tutorial
tags: [ICMP, TTL, Python, Networking]
---

# Understanding ICMP and TTL with Python

> To understand the following, a basic understanding of the OSI Layer & basic Python is essential.

## Table of Contents

1. [What is ICMP](#what-is-icmp)
2. [What is TTL](#what-is-ttl)
3. [Raw Sockets](#raw-sockets)
4. [Sending ICMP ECHO Request](#sending-icmp-echo-request)
5. [Capturing ICMP ECHO Reply](#capturing-icmp-echo-reply)
6. [GitHub Project Structure](#github-project-structure)

---

## What is ICMP

We will be focusing on **ICMPv4**, which uses IPv4.  

ICMP (Internet Control Message Protocol) is a **RFC standard network protocol** used for checking and diagnosing network connectivity.

Some popular tools that use ICMP:

1. **Ping**
2. **Traceroute**

> The logic for both ping and traceroute is similar. The major difference is how TTL works in each case.

---

## What is TTL

TTL (**Time To Live**) is a field found in the **IP header**, not the ICMP header.  

It is mainly used to **prevent routing loops**. If a packet exceeds its TTL before reaching the destination, it is **dropped**. This ensures routers maintain CPU performance.

ICMP uses **IP** and does not use **TCP or UDP**. Some ping tools use TCP/UDP to check service availability, but this tutorial focuses on traditional ICMP.  

TTL is a **1-byte field**, so its maximum value is **255**.

**Note:** Only **Layer 3 devices** (routers/firewalls) utilize TTL. Switches do not decrement TTL.

Predefined TTL values depending on the OS:

| OS       | TTL Value |
|----------|-----------|
| Linux    | 64        |
| Windows  | 128       |
| Cisco    | 255       |

---

## Raw Sockets

Raw sockets are essential to **send custom ICMP requests** and **capture ICMP replies**.

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP)

> The above code snippet creates a socket object sock to send ICMP requests and capture replies.




---

Sending ICMP ECHO Request

Building the ICMP request

import struct
import time
import os

ICMP_ECHO_REQUEST = 8  # Defined in icmp_constants.py

def build_icmp_request(identifier, sequence):
    icmp_type = ICMP_ECHO_REQUEST
    code = 0
    checksum_icmp = 0

    header = struct.pack('!BBHHH', icmp_type, code, checksum_icmp, identifier, sequence)
    send_time = time.time()
    data = struct.pack('!d', send_time) + b'ping_payload'

    checksum_icmp = checksum(header + data)
    header = struct.pack('!BBHHH', icmp_type, code, checksum_icmp, identifier, sequence)

    return header + data

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


---

Sending the request

def send_icmp_request(sock, ip, sequence=0):
    identifier = os.getpid() & 0xFFFF
    icmp_packet = build_icmp_request(identifier, sequence)
    timestamp = struct.unpack('!d', icmp_packet[8:16])[0]

    try:
        sock.sendto(icmp_packet, (ip, 0))
    except OSError as e:
        print(f"ERROR OCCURRED: {e}")
    except Exception as e:
        print(f"EXCEPTION: {e}")

    return identifier, sequence, timestamp, ip


---

Capturing ICMP ECHO Reply

data, addr = sock.recvfrom(1024)

reply_info = {
    "src_ip": src_ip,
    "ttl": ttl,
    "icmp_type": icmp_type,
    "icmp_code": icmp_code,
    "identifier": icmp_identifier,
    "sequence": icmp_sequence,
    "start_time": start_time,
    "reply_time": reply_time
}

> After receiving the reply and unpacking required fields, we store them in a dictionary for easy access.




---

GitHub Project Structure

Mping/
├── .gitignore
├── icmp_constants.py
├── icmp_network.py
├── icmp_parser.py
├── icmp_utils.py
├── mping.py
├── ping_output.py
└── ping_stats.py

GitHub Link: mping project

---