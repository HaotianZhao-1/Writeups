# Attacktive Directory — TryHackMe

> Lab target IPs can change between sessions. The scan below used `10.145.149.203`; earlier commands in this writeup used `10.146.158.66`. Both refer to the same lab target in different sessions. When repeating the steps, use the IP assigned to the current session.

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

> To add: the complete Nmap command. The confirmed option from the earlier scan is `-Pn`.

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

`backup` is also worth investigating because its name suggests a backup-related account. Its group memberships and effective permissions still need to be checked; the name alone does not establish elevated access.

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

The hash begins with `$krb5asrep$23$`, which matches this hash type. I will use mode `18200` when attempting to crack it with Hashcat.

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

## 9. Inspecting backup_credentials.txt Locally

I tried to use `cat` inside the interactive `smbclient` console, but `cat` is not an smbclient command. I downloaded `backup_credentials.txt` to my own machine and viewed it locally instead.

The file contained a string that appeared to be Base64-encoded:

```text
[REDACTED BASE64 STRING — possible credentials]
```

Base64 is an encoding, not encryption. The original string is omitted from this published writeup because it contains credentials.

### Decoding Results

After decoding the string, I recovered a username and password for the `backup` account:

```text
backup@spookysec.local:[REDACTED]
```

This connects the file's contents to the `backup` username previously identified during enumeration. The credentials were subsequently used successfully with `secretsdump`, as documented in Section 10. The account's exact permissions have not yet been enumerated.

> To add: the exact decoding command or tool used.

### File Download Command

Inside the interactive `smbclient` console, I downloaded the file to my local machine with:

```text
smb: \> get backup_credentials.txt
```

The `get` command copies the remote file to the local working directory unless another local destination is specified.

## 10. Extracting Account Hashes with Impacket secretsdump

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

This demonstrated that the recovered `backup` credentials could be used to extract account hash material in this lab. Having credentials alone does not guarantee this capability: the account must also have sufficient permissions. The single output line does not establish which extraction method was used or the exact permissions granted to `backup`.

> To add: relevant non-sensitive output showing the extraction method. No Administrator login or remote shell has been documented yet.

## What I Learned

An open port helps me choose what to investigate next: port **389** suggested **LDAP**, and the LDAP response revealed the **domain name**.
