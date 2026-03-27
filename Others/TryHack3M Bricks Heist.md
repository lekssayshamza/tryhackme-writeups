# Nmap
```bash
nmap -sS -sV -O 10.112.150.47
```


```bash
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http     Python http.server 3.5 - 3.10
443/tcp  open  ssl/http Apache httpd
3306/tcp open  mysql    MySQL (unauthorized)
```
# Whatweb
```bash
whatweb 10.112.150.47
```

```bash
WordPress 6.5
```
# ffuf
```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u https://10.112.150.47/FUZZ
```

```bash
0                       Status: 301
admin                   Status: 302
atom                    Status: 301
dashboard               Status: 302
embed                   Status: 301
favicon.ico             Status: 302
feed                    Status: 301
index.php               Status: 301
login                   Status: 302
page1                   Status: 301
phpmyadmin              Status: 301
rdf                     Status: 301
rss                     Status: 301
robots.txt              Status: 200
rss2                    Status: 301
wp-admin                Status: 301
wp-content              Status: 301
wp-includes             Status: 301
```
# Possible Usernames
```bash
ffuf -w /usr/share/seclists/Usernames/cirt-default-usernames.txt -X POST -d "log=FUZZ&pwd=pass" -H "Content-Type: application/x-www-form-urlencoded" -u https://bricks.thm/wp-login.php -fw 218
```

```
ADMINISTRATOR
Administrator
administrator
```