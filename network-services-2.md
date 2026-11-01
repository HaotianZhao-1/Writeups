# Network Services 2 — TryHackMe

Note: All target IP addresses in this write-up are shown as `<IP_ADDRESS>`, because the lab target IP changes between sessions.

## 1. NFS

NFS stands for Network File System. It is a system that allows files on a remote machine to be accessed as if they were local files. With NFS, files can be shared with other machines on the same network.

The NFS step from this lab is recorded in Section 3.

## 2. Enumeration

Enumeration is the systematic gathering of detailed information about a target, such as its services, shares, users, and permissions. It is the information-gathering phase that comes before exploitation: the attacker is preparing to attack, but is not launching an exploit yet.

This phase is important because it is the final discovery step before getting into a machine. Its findings determine which entry point or exploit is worth trying next, so a missed finding here can mean a missed way in.

## 3. NFS Enumeration and Mounting

### Full Port Scan

I scanned all ports and listed only the open ones:

```bash
nmap -p- --open <IP_ADDRESS>
```

Seven open ports were found:

```text
PORT      STATE SERVICE
22/tcp    open  ssh
111/tcp   open  rpcbind
2049/tcp  open  nfs
38241/tcp open  unknown
38869/tcp open  unknown
43943/tcp open  unknown
56205/tcp open  unknown
```

Port `2049/tcp` was the target port here, because it is the NFS service.

### Finding the NFS Export with showmount

I used `showmount` to list the NFS shares exported by the target:

```bash
/usr/sbin/showmount -e <IP_ADDRESS>
```

```text
Export list for <IP_ADDRESS>:
/home *
```

This showed that the `/home` directory was exported, and the `*` means it was available to any host, rather than restricted to a specific client.

### Mounting the NFS Share

I mounted the exported `/home` directory to `/tmp/mount` on my machine:

```bash
sudo mount -t nfs <IP_ADDRESS>:/home /tmp/mount -o nolock
```

### Listing the Mounted Share

Listing the mounted directory returned:

```text
total 12
drwxr-xr-x  4 root root 4096 Jun 22  2025 .
drwxrwxrwt 13 root root  280 Oct 31 17:57 ..
drwxr-xr-x  5 tian tian 4096 Jun 22  2025 cappucino
drwxr-xr-x  4 1001 1002 4096 Jun 22  2025 ubuntu
```

The mounted share contained two user-like directories, `cappucino` and `ubuntu`. Their contents are not recorded here yet.
