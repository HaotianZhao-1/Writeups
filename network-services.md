# Network Services — TryHackMe

## 1. Port Scanning

I used Nmap with `-Pn` to identify open ports on the lab target. The `-Pn` option skips host discovery and treats the target as online.

### Command Used

```bash
nmap -Pn 10.144.162.168
```

### Scan Results

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-24 16:01 -0700
Nmap scan report for 10.144.162.168
Host is up (0.040s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 1.42 seconds
```

I found ports `139` and `445` interesting because they are associated with SMB. That suggested the target might be running an SMB server, such as a Samba implementation, so identifying the SMB service and version became the next step, documented in Section 2. The open ports alone do not confirm Samba or any specific version, and the service names above are Nmap's port labels, not confirmed software versions.

## 2. Service Enumeration

### SMB

SMB is a request-response protocol that allows clients and servers to communicate over TCP/IP, typically to share files, printers, and other resources over a network. The client sends a request, and the server responds to it.

### Workgroup Enumeration with enum4linux

This step used target IP `10.146.165.216`, which differs from the IP used in the earlier Nmap session. Target IPs can change between lab sessions, so the IP in each command is the one used at the time.

### Command Used

```bash
enum4linux -a 10.146.165.216
```

### Results

The enumeration returned the domain/workgroup name:

```text
WORKGROUP
```

This gave a workgroup name to use for further SMB enumeration, rather than assuming a domain name.

### Machine Identification from srvinfo

Further enumeration for `10.146.165.216` returned OS information from srvinfo. The machine name was `POLOSMB`, and the server was identified as Samba on Ubuntu:

```text
POLOSMB        Wk Sv PrQ Unx NT SNT POLOSMB server (Samba, Ubuntu)
platform_id     :       500
os version      :       6.1
server type     :       0x809a03
```

This confirms the earlier possibility raised by ports `139` and `445`: the SMB service here is Samba, not merely a service suggested by open ports.

### SMB Share Listing

The share listing for `10.146.165.216` returned the following shares:

```text
Sharename       Type      Comment
---------       ----      -------
netlogon        Disk      Network Logon Service
profiles        Disk      Users profiles
print$          Disk      Printer Drivers
IPC$            IPC       IPC Service (POLOSMB server (Samba, Ubuntu))
```

The listing also included an SMB1 negotiation message:

```text
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server 10.146.165.216 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

This SMB1 failure occurred during the subsequent workgroup-listing attempt. It does not invalidate the share list already returned.

The `profiles` share stood out because it is described as Users profiles, which may contain user-related data. Investigating that share became the next step of interest.

### Accessing the profiles Share as Anonymous

Using the information from the share listing, I connected to the `profiles` share as an anonymous user, without supplying a password. This step used target IP `10.144.160.209`, which differs from the IP used in earlier enumeration sessions.

### Command Used

```bash
smbclient //10.144.160.209/profiles -U anonymous -p 445
```

- `-U anonymous`: specify the anonymous username.
- `-p 445`: specify the SMB port.

This connection used the `profiles` share identified in the earlier share listing, rather than assuming access to a different share.
