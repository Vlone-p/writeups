# CyberLearner: LedgerX

**Platform:** [CyberLearner](https://cyberlearner.org)

**Challenge:** [LedgerX](https://cyberlearner.org/challenges#LedgerX-18)

**Difficulty:** Easy (50 points)

**Focus:** Business logic vulnerability in a credit transfer API.

**⚠️Disclaimer:** This writeup contains step by step solutions. The flag has been redacted to encourage active learning.

---

## Overview

ICO's internal finance team built a peer to peer credit exchange for staff. It was audited last year and passed with minor findings. After the audit, a developer refactored the transfer API to simplify input handling. The Classified Archive is gated at 10,000 credits. You start with 200. The API accepts JSON. Find a way in.

This is a straightforward CTF. We just need the flag.

---

## Reconnaissance

We start the lab and get our target URL:

```
https://9d67806cc20d3643e098fc14c3f4d909.lab.cyberlearner.org/login
```

*(Your URL will be different.)*

Visiting the site shows a staff login page:

![LedgerX Login Page](images/login.png)

We don't have credentials, but there's a "Create account" link, so let's register a new account. I'll use:

```
person1:person1
```

---

## Enumeration

After logging in as `person1`, we find a few menu options: Dashboard, Transfer, Archive, Logout. Let's check the balance and the transfer page:

![LedgerX Transfer Page](images/transfer.png)

We start with **200 credits**. The challenge description says the Classified Archive is gated at **10,000 credits**. So we need a lot more than we have.

Let's explore the Transfer page. We can pick a recipient, enter an amount, and add an optional note. The interesting question is: does the app actually validate that we have enough credits before sending?

---

## The Bug

Let's test it. I create a second account to send credits to:

```
person2:person2
```

Now back on the Transfer page as `person1`, I select `person2` as the recipient and try sending **100,000 credits**, even though I only have 200.

It goes through! The app doesn't check whether the sender has enough balance. This is a classic business logic vulnerability: the refactored transfer API trusts the input without validating it server side.

---

## Exploitation

So the plan is simple:

1. Log in as `person1`
2. Transfer a huge amount (say, 20,000+) to `person2`
3. Log out and back in as `person2` (or just use the second account)
4. Visit the Classified Archive at `/archive`

```
https://9d67806cc20d3643e098fc14c3f4d909.lab.cyberlearner.org/archive
```

Since `person2` now has over 10,000 credits, the archive gate opens and reveals the flag.

---

## Flag

**What is the flag?**
> ✅ **Answer:** `ICO{...}`

*(Flag redacted. Walk the exploitation chain yourself to retrieve it!)*

---

## Conclusion

LedgerX is a great example of a business logic flaw. There's no fancy SQL injection or XSS here. The vulnerability is in the application's own logic: the transfer API was refactored to "simplify input handling" and in doing so, it stopped validating the sender's balance. As a result, anyone can mint unlimited credits by sending more than they own.

The takeaway for defenders: always validate business rules server side, even after a refactor. Input handling should never come at the cost of integrity checks.

Thanks to the [CyberLearner](https://cyberlearner.org) portal for hosting this challenge!
