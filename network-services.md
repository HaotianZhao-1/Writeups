# Network Services — TryHackMe

> Starter structure only, following the Attacktive Directory file. Sections are placeholders until lab notes are confirmed. Flag values and password hashes are never pasted in this writeup.

## Overview

[Write 2-3 sentences here once the lab is underway: what you enumerated, what service gave you access, and what you ultimately obtained.]

**Attack chain:** [First step] → [Next step] → [Final goal]

> Replace `<IP_ADDRESS>` with the target IP assigned to the current lab session. Commands below retain the IPs used at the time.

## 1. Port Scanning

I used Nmap with `-Pn` to identify open ports on the lab target. The `-Pn` option skips host discovery and treats the target as online.

### Command Used

```bash
nmap -Pn <IP_ADDRESS>
```

Replace `<IP_ADDRESS>` with the target IP assigned to the current lab session.

### Scan Results

```text
[Paste the Nmap scan results here once confirmed.]
```

[Write what the open services suggested, and which service you chose to investigate first and why.]

## 2. Service Enumeration

[Enumerate each exposed service in turn. Every finding below must be followed directly by the next action it led to.]

### SMB

SMB is a request-response protocol that allows clients and servers to communicate over TCP/IP, typically to share files, printers, and other resources over a network. The client sends a request, and the server responds to it.

### Command Used

```bash
[Exact service enumeration command used, with the session IP.]
```

- [Flag for what the command does, if needed.]

### Results

```text
[Paste the confirmed enumeration output here, with any sensitive values redacted.]
```

[Write what this finding meant and what you did next because of it.]

### Troubleshooting

[If a command or path failed at first, write what went wrong and how you fixed it. Delete this subsection if nothing failed.]

## 3. Initial Access

[How you obtained initial access, once confirmed.]

### Command Used

```bash
[Exact command used for initial access, with credentials and hashes redacted.]
```

### Results

```text
[Paste the confirmed result here, with sensitive values redacted.]
```

### Why it worked

[One sentence on the principle: why this technique worked in this scenario.]

## 4. Privilege Escalation / Further Access

[How you progressed after initial access, once confirmed. Delete this section if the lab does not involve a separate escalation step.]

### Command Used

```bash
[Exact command used, with credentials and hashes redacted.]
```

### Results

```text
[Paste the confirmed result here, with sensitive values redacted.]
```

### Why it worked

[One sentence on the principle: why this technique worked in this scenario.]

## 5. Locating and Reading the Lab Flags

After obtaining access, I read each lab flag from its location. Flag values are omitted from this writeup.

| Account / area | Location | Finding |
| --- | --- | --- |
| [e.g. user] | [path] | [e.g. user flag] |
| [e.g. root / admin] | [path] | [e.g. root flag] |

### Commands Used

```text
[Exact command used to read the first flag.]
```

```text
[Exact command used to read the second flag, if applicable.]
```

## What I Learned

- **Enumeration builds the attack path.** [What enumeration taught you in this lab, once confirmed.]
- **A finding is not access.** [What a finding alone did and did not give you, once confirmed.]
- **Why it worked matters.** [One principle you will reuse next time, once confirmed.]
- **Next time faster.** [What you would do faster next time, once confirmed.]
