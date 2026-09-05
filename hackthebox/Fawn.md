# HackTheBox - Fawn

**Platform:** HackTheBox 

**Difficulty:** Very Easy

**⚠️Disclaimer:** This writeup contains direct answers to task questions, step by step solutions, and the final flag.

## Pre-Engagement & FTP Basics

Before interacting with the target, we review the fundamentals of the FTP protocol:

**What does the 3-letter acronym FTP stand for?**
> ✅ **Answer:** `File Transfer Protocol`

**Which port does the FTP service listen on usually?**
> ✅ **Answer:** `21`

**FTP sends data in the clear, without any encryption. What acronym is used for a later protocol designed to provide similar functionality to FTP but securely, as an extension of the SSH protocol?**
> ✅ **Answer:** `SFTP`

---

## Enumeration

First, we need to verify that our target is online and reachable. 

**What is the command we can use to send an ICMP echo request to test our connection to the target?**
> ✅ **Answer:** `ping`

We issue the `ping` command to the target IP to confirm we have a connection.

```bash
ping 10.129.190.126                                                                    
```
```text
PING 10.129.190.126 (10.129.190.126) 56(84) bytes of data.
64 bytes from 10.129.190.126: icmp_seq=1 ttl=63 time=36.5 ms
64 bytes from 10.129.190.126: icmp_seq=2 ttl=63 time=34.1 ms
64 bytes from 10.129.190.126: icmp_seq=3 ttl=63 time=34.5 ms
64 bytes from 10.129.190.126: icmp_seq=4 ttl=63 time=35.2 ms
^C
--- 10.129.190.126 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 34.078/35.070/36.503/0.922 ms
```

Next, we perform a port scan with service detection using `nmap` to identify open ports and the services running on them.

```bash
nmap 10.129.200.95 -sV
```
```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-03 20:15 +0200
Nmap scan report for 10.129.200.95
Host is up (0.037s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 4.04 seconds
```

**Alternative Enumeration:**
As an alternative to Nmap, a custom Python port scanner (`viper.py`) can be used to quickly identify open ports and detect the OS.
```bash
python viper.py  10.129.190.126 -sV -O
```
```text
alternative (mine):
____   ____.__                     
\   \ /   /|__|_____   ___________ 
 \   Y   / |  \____ \_/ __ \_  __ \
  \     /  |  |  |_> >  ___/|  | \/
   \___/   |__|   __/ \___  >__|   
              |__|        \/       v1.3

[*] Starting scan on 10.129.190.126 (10.129.190.126)
[*] Thread count set to 100

[*] OS Detection: Linux/Unix (TTL: 63).
[*] Scanning top 65 common ports... (Use -p to specify a range)

==================================================
SCAN RESULTS
==================================================
OS DETECTION: Linux/Unix (TTL: 63)
==================================================
PORT       STATE      SERVICE
-----------------------------
21         open 220 (vsFTPd 3.0.3)
==================================================
Scan completed in 0.12 seconds
```

Based on our scan results, we can answer the following:

**From your scans, what version is FTP running on the target?**
> ✅ **Answer:** `vsftpd 3.0.3`

**From your scans, what OS type is running on the target?**
> ✅ **Answer:** `Unix`

**What is the command we need to run in order to display the 'ftp' client help menu?**
> ✅ **Answer:** `ftp -?`

---

## Exploitation

FTP servers often allow anonymous login for public file sharing. We will attempt to connect using the default anonymous credentials. 

**What is username that is used over FTP when you want to log in without having an account?**
> ✅ **Answer:** `anonymous`

We connect to the target using the `ftp` client with the `anonymous` username and a blank password.

```bash
ftp 10.129.200.95                                                                      
```
```text
Connected to 10.129.200.95.
220 (vsFTPd 3.0.3)
Name (10.129.200.95:User): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

**What is the response code we get for the FTP message 'Login successful'?**
> ✅ **Answer:** `230`

---

## Flag Retrieval

Once logged in, we need to list the available files on the FTP server. 

**There are a couple of commands we can use to list the files and directories available on the FTP server. One is dir. What is the other that is a common way to list files on a Linux system.**
> ✅ **Answer:** `ls`

We use the `ls` command to list the directory contents and find a `flag.txt` file.

```bash
ftp> ls
```
```text
229 Entering Extended Passive Mode (|||37138|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
226 Directory send OK.
```

To read the flag, we can download it to our local machine. 

**What is the command used to download the file we found on the FTP server?**
> ✅ **Answer:** `get`

*(Note: In the output below, `less` was used to read the file directly from the FTP prompt, revealing the flag's contents).*

```bash
ftp> less flag.txt
```
```text
035db21c881520061c53e0536e44f815
```

**Submit the flag located on the FTP server.**
> ✅ **Answer:** `035db21c881520061c53e0536e44f815`
