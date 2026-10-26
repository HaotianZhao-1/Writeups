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

### Note on Non-Standard Ports

A scan that does not cover the full port range will not show services running on non-standard ports. Services can run on ports other than their usual ones, so a port that is not scanned can hide a service entirely. This is a reason to scan fully rather than relying only on a default port list.

## 2. Service Enumeration

### SMB

SMB is a request-response protocol that allows clients and servers to communicate over TCP/IP, typically to share files, printers, and other resources over a network. The client sends a request, and the server responds to it.

### Telnet

Telnet is another client-server communication protocol. Unlike SMB, which is mainly used for sharing files, printers, and other resources over a network, Telnet is mainly used for remote terminal control: a client connects to a server and controls its terminal remotely.

The Telnet step from this lab is recorded later in this file, after the full port scan.

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

### Listing the profiles Share

After connecting as anonymous, I listed the contents of the `profiles` share:

```text
smb: \> ls
  .                                   D        0  Tue Apr 21 04:08:23 2020
  ..                                  D        0  Tue Apr 21 03:49:56 2020
  .cache                             DH        0  Tue Apr 21 04:08:23 2020
  .profile                            H      807  Tue Apr 21 04:08:23 2020
  .sudo_as_admin_successful           H        0  Tue Apr 21 04:08:23 2020
  .bash_logout                        H      220  Tue Apr 21 04:08:23 2020
  .viminfo                            H      947  Tue Apr 21 04:08:23 2020
  Working From Home Information.txt      N      358  Tue Apr 21 04:08:23 2020
  .ssh                               DH        0  Tue Apr 21 04:08:23 2020
  .bashrc                             H     3771  Tue Apr 21 04:08:23 2020
  .gnupg                             DH        0  Tue Apr 21 04:08:23 2020

                15373236 blocks of size 1024. 7030416 blocks available
```

I also confirmed the current directory:

```text
smb: \> pwd
Current directory is \\10.144.160.209\profiles\
```

The listing shows a user-like home directory, including hidden files and directories such as `.ssh` and `.gnupg`, and a file named `Working From Home Information.txt`. This listing alone does not record the contents of those files.

### Downloading and Reading Working From Home Information.txt

From the `profiles` share, I downloaded the file `Working From Home Information.txt` to my local machine:

```text
smb: \> get "working From Home Information.txt"
```

I then read the downloaded file locally:

```bash
cat working\ From\ Home\ Information.txt
```

### Findings

The file was a note from James, Department Manager, to John Cactus. It said that, because of the pandemic, most of POLO inc. had moved to working from home where possible, and that John Cactus's account had been enabled with SSH access to the main server. For problems, it directed the reader to contact the IT department.

This gave a concrete next lead: an employee account, John Cactus, with SSH access to the main server, rather than only an anonymous SMB share listing.

## 3. Initial Access via SSH with a Private Key

The SSH lead from `Working From Home Information.txt` pointed to John Cactus's account. Inside the `profiles` share, I entered the `.ssh` directory and listed it:

```text
smb: \> cd .ssh
smb: \.ssh\> ls
  .                                   D        0  Tue Apr 21 04:08:23 2020
  ..                                  D        0  Tue Apr 21 03:49:56 2020
  id_rsa                              N     1679  Tue Apr 21 04:08:23 2020
  id_rsa.pub                          N      396  Tue Apr 21 04:08:23 2020
  authorized_keys                     N        0  Tue Apr 21 04:08:23 2020
```

I downloaded the private key file `id_rsa` from that directory to my local machine. The private key content itself is not reproduced here.

I then used that private key as the SSH identity to log in as user `cactus`:

```bash
ssh -i id_rsa cactus@10.144.160.209
```

This worked because the private key matched the account's SSH authentication, turning the anonymous SMB access into authenticated SSH access. Note that this was a private key, not a password hash.

After logging in, I found and read `smb.txt` in the account's home directory:

```bash
cactus@POLOSMB:~$ cat smb.txt
```

That file contained the SMB flag. The flag value is intentionally omitted from this write-up.

## 4. Full Port Scan and Telnet Backdoor

This step puts the earlier note on non-standard ports into practice: a full scan found a service that a default scan would not necessarily surface.

### Full Port Scan

```bash
nmap -p- 10.146.144.67
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-25 16:25 -0700
Nmap scan report for 10.146.144.67
Host is up (0.035s latency).
Not shown: 65534 closed tcp ports (reset)
PORT     STATE SERVICE
8012/tcp open  unknown
```

Only one open port was found, `8012/tcp`, and Nmap labeled its service as `unknown`. That label does not identify the service; it only means Nmap did not recognize it from the port number, which fits a non-standard port.

### Connecting with Telnet

I then connected to port `8012` with Telnet:

```bash
telnet 10.144.190.49 8012
```

Note that the IP in the Telnet command as given differs from the IP in the Nmap command above. Target IPs can change between lab sessions, so each command is recorded with the IP used at the time.

```text
Trying 10.144.190.49...
Connected to 10.144.190.49.
Escape character is '^]'.
SKIDY'S BACKDOOR. Type .HELP to view commands
```

The banner identified the service on port `8012` as SKIDY'S BACKDOOR, and pointed to `.HELP` for its commands. Running `.HELP` itself is not recorded here yet.
