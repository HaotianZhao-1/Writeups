# Attacktive Directory — TryHackMe

## 1. Port Scanning

I used Nmap with `-Pn` to identify open ports on the lab target. The `-Pn` option skips host discovery and treats the target as online. One of the ports I found was **389 (LDAP)**.

LDAP is used to query directory information, so I chose to investigate it for details about the target's domain.

> To add: the complete Nmap command and relevant scan output. The confirmed option is `-Pn`.

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

This confirms that `svc-admin` has Kerberos pre-authentication disabled. I obtained an AS-REP hash suitable for offline password cracking, not the plaintext password. Cracking has not yet been documented.


## 5. Identifying the Hashcat Mode

After searching online, I identified the Hashcat mode for the collected AS-REP hash:

| Hashcat Mode | Hash Type |
| --- | --- |
| `18200` | Kerberos 5, etype 23, AS-REP |

The hash begins with `$krb5asrep$23# Attacktive Directory — TryHackMe

## 1. Port Scanning

I used Nmap with `-Pn` to identify open ports on the lab target. The `-Pn` option skips host discovery and treats the target as online. One of the ports I found was **389 (LDAP)**.

LDAP is used to query directory information, so I chose to investigate it for details about the target's domain.

> To add: the complete Nmap command and relevant scan output. The confirmed option is `-Pn`.

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

This confirms that `svc-admin` has Kerberos pre-authentication disabled. I obtained an AS-REP hash suitable for offline password cracking, not the plaintext password. Cracking has not yet been documented.


, which matches this hash type. I will use mode `18200` when attempting to crack it with Hashcat.

> To add: the cracking command and its result once attempted.

## What I Learned

An open port helps me choose what to investigate next: port **389** suggested **LDAP**, and the LDAP response revealed the **domain name**.
