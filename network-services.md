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

The open NetBIOS (`139`) and SMB (`445`) ports pointed to SMB as the service to investigate next, which is documented in Section 2. The service names above are Nmap's port labels, not confirmed software versions.

## 2. Service Enumeration

### SMB

SMB is a request-response protocol that allows clients and servers to communicate over TCP/IP, typically to share files, printers, and other resources over a network. The client sends a request, and the server responds to it.
