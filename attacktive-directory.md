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

> To add: the exact Kerbrute command.

## What I Learned

An open port helps me choose what to investigate next: port **389** suggested **LDAP**, and the LDAP response revealed the **domain name**.
