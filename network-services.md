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
