# Writeup EN - Ultratech (TryHackMe)

## Description

This writeup presents the exploitation of the Ultratech machine available on TryHackMe. The objective is to compromise the machine by following a structured methodology: enumeration, exploitation of a backend vulnerability, then privilege escalation via system misconfiguration.

**Methodology adopted:** Reconnaissance → Enumeration → Exploitation → Privilege Escalation

---

## Reconnaissance

To begin, I performed a complete TCP port scan with `nmap` to identify exposed services (see `artifacts/scan.xml`).

### Discovered Services

- **FTP (21)**
- **SSH (22)**
- **HTTP service on port 8081**
- **HTTP service on port 31331**

Anonymous FTP access was tested but denied, which allows us to rule out this initial attack vector.

The nmap scan also revealed two HTTP ports: the first, port **8081**, is used by **Node.js**, and **31331** hosts an **Apache** server.

We then move on to web analysis.

---

## Web Enumeration

We use `gobuster` to discover hidden directories on both HTTP ports (see `artifacts/gobuster_8081.txt` and `artifacts/gobuster_31331.txt`).

### Port 8081 — Node.js API

- `/auth` — Authentication
- `/ping` — Availability check

This confirms that there is a backend API that handles authentication and system checks.

### Port 31331 — Apache Server (Frontend)

Many resources were found, including:

- `/robots.txt`
- `/js` directory containing JavaScript files
- `partners.html` page

By consulting `robots.txt`, we discover an additional sitemap that reveals `partners.html`, a login page for partners.

We can deduce that the Apache server acts as a frontend and uses the Node.js API on port 8081.

### JavaScript Code Analysis

By analyzing the `API.js` file (found in `/js`), we can observe how the frontend interacts with the API:

- The `/auth` endpoint handles authentication
- The `/ping` endpoint checks API availability

By accessing `/ping` directly without parameters:

```
http://TARGET_IP:8081/ping
```

We get a server error that displays a complete Node.js stack trace and internal file system paths. The disclosure of this information confirms poor error handling and reveals sensitive details about the backend implementation.

### Identified Vulnerability: Command Injection

The `/ping` endpoint accepts an `ip` parameter that can be controlled by the user, supposedly representing an IP address or hostname. The problem is that this parameter is used directly in a system command without validation, which strongly suggests a **command injection vulnerability**.

The raw output of the executed command is returned by the API, allowing an attacker to interact indirectly with the underlying system.

We can therefore test a command injection such as `ls`, which would give this:

```
http://TARGET_IP:8081/ping?ip=`ls`
```

**Result:** The `ls` command executes and we see the directory files.

---

## Exploitation

*(See associated screenshots in the `artifacts/` folder)*

By exploiting the command injection, we list the files and discover **`utech.db.sqlite`**.

We read the contents of this database:

```
http://TARGET_IP:8081/ping?ip=`cat%20utech.db.sqlite`
```

The database contains authentication credentials including a password hash for the **r00t** user.

We crack the hash (with CrackStation, Hashcat or John the Ripper) and obtain the plaintext password.

### SSH Connection

We can now establish an SSH connection on port 22 with the obtained credentials and password:

```bash
ssh r00t@TARGET_IP
```

**Access obtained!**

---

## Privilege Escalation

Once connected as **r00t**, we can check the current user's groups with the command:

```bash
id
```

This reveals that we belong to the **docker** group.

This membership is critical, as it allows executing Docker containers with elevated privileges, providing a direct path to privilege escalation.

### Exploitation via Docker

We consult **GTFOBins** for the exploitation technique.

We list available images:

```bash
docker images
```

We launch a container by mounting the host's root file system:

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

**Explanation:**

- `-v /:/mnt`: Mounts the host's root in `/mnt` of the container
- `chroot /mnt`: Changes the root to the host's file system

We obtain a shell with root privileges on the host machine.

**Root obtained!**

---

## Notes on Artifacts

The complete set of technical evidence is provided in the `artifacts/` folder:

- Nmap scan results
- Directory enumeration outputs
- Screenshots of key steps

Sensitive information has been intentionally excluded from this document.
