# Investigation: JetBrains TeamCity Web Server Exploitation

Source: CyberDefenders lab : Network Traffic Analysis (Wireshark)

## Scenario

Analyzed a pcap capturing an attacker exploiting a public-facing TeamCity CI/CD server, from initial access through persistence and privilege escalation attempts.

## What I did

Filtered traffic by `http` and followed the lab's hints to find a POST request involving a suspicious admin/plugin upload path. Used Follow HTTP Stream on the relevant requests to read full request/response bodies instead of just single packets.

## Timeline of the attack

**1. Version fingerprinting**
Attacker queried `/app/rest/server` and got back the server version: `2023.11.3`.

**2. Vulnerability identification**
I initially assumed CVE-2023-42793 since it's the well-known TeamCity auth bypass, but that CVE only affects versions before 2023.05.4 didn't match. Cross-checked the version against known CVEs and found **CVE-2024-27198**, an authentication bypass affecting versions before 2023.11.4, which does match.

**3. Auth bypass → account creation**
Attacker sent a POST to `/app/rest/users` (via a path-traversal-style URL abusing the vulnerability) creating a new user:
- Username: `c91oyemw`
- Password: `CL5vzdwLuK`
- Role: `SYSTEM_ADMIN`

No authentication was required for this step, that's the actual vulnerability.

**4. Token generation and authenticated access**
Attacker generated an auth token for the new account and used it (Bearer token in the `Authorization` header) for all further requests.

**5. Recon**
Queried `/app/rest/debug/jvm/systemProperties`, pulled back server environment details (OS, Java version, file paths, running as `root`).

**6. Persistence webshell upload**
Uploaded a malicious plugin (`NSt8bHTg.zip`) via `/admin/pluginUpload.html`, containing a JSP webshell (`NSt8bHTg.jsp`). The webshell accepts a `cmd` parameter and executes it via `ProcessBuilder`, giving the attacker arbitrary command execution.

**7. Privilege escalation / container escape attempts**
Using the webshell, the attacker tried three different Docker-based container escape techniques:
```
docker run --rm -it --privileged ubuntu
docker run --rm -it -v /:/host ubuntu chroot /host
docker run -v /var/run/docker.sock:/var/run/docker.sock -it ubuntu
```
The second command (mounting the host filesystem and chrooting into it) is the actual container escape attempt. Followed by basic recon (`ls`, `whoami`), which returned `root`, confirming they already had root inside the container context, though it's not fully clear from the traffic alone whether the escape itself succeeded.

**8. Data manipulation**
Attacker checked and then overwrote `/tmp/Creds.txt`:
```bash
bash -c 'echo "username:a1l4m,password:youarecompromised" > /tmp/Creds.txt'
```
New credentials written: username `a1l4m`, password `youarecompromised`.

## IOCs

- Attacker-created username: `c91oyemw`
- Password: `CL5vzdwLuK`
- Webshell filename: `NSt8bHTg.jsp` / plugin: `NSt8bHTg.zip`
- Tampered file: `/tmp/Creds.txt`
- New credentials written into tampered file: `a1l4m` / `youarecompromised`

## MITRE ATT&CK mapping

- **Initial Access T1190:** Exploit Public-Facing Application (CVE-2024-27198 auth bypass)
- **Persistence T1505.003:** Server Software Component: Web Shell
- **Privilege Escalation T1611:** Escape to Host (container escape attempts)
- **Impact T1565.001:** Stored Data Manipulation (credentials file tampering)

## What this investigation covers vs. doesn't

This was detection and analysis, reconstructing what happened and how, not remediation. In a real incident, next steps would be: patch TeamCity past 2023.11.4, remove the malicious admin account and plugin, rotate any potentially compromised credentials, and check whether the attacker pivoted elsewhere using their root access.

## Takeaway

This mapped to a real, actively-exploited CVE (CISA KEV-listed), not a synthetic scenario. The investigation reinforced reading raw HTTP streams, decoding URL-encoded commands, and recognizing a recon → exploit → persistence → escalation → tampering pattern, the same general shape I'd expect to see across different attack types going forward.
