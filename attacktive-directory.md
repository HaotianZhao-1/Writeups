# Attacktive Directory — TryHackMe

## 1. Port Scanning

I used Nmap to identify open ports on the lab target. One of the ports I found was **389 (LDAP)**.

LDAP is used to query directory information, so I chose to investigate it for details about the target's domain.

> To add: the exact Nmap command and relevant scan output. My original note said `-Np`; I need to check the actual options before including the command here.

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

> To add: the exact `ldapsearch` command I used.

## What I Learned

An open port helps me choose what to investigate next: port **389** suggested **LDAP**, and the LDAP response revealed the **domain name**.
