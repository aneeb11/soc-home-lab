# SOC Home Lab : Detection & Blue Team Practice

# About This Repo

This repo documents my hands-on journey building a home lab for SOC Analyst / Detection Engineering practice, attacking my own isolated environment, capturing what it looks like from a defender's side, and writing up what I'd do about it as an analyst.

I'm a cybersecurity student currently working toward an entry-level SOC Analyst role, with a longer-term goal of Detection Engineering / Threat Hunting. This lab is where I turn theory (networking, Linux, Windows internals) into practice by simulating real attack techniques and learning to detect them myself.

Each folder in this repo represents one phase of the lab or one attack simulation setup notes, what I ran, what showed up in the logs, and what I'd flag as an analyst.


# Lab Architecture

| VM | Role | Purpose |
|---|---|---|
| Kali Linux | Attacker | Runs Nmap, Metasploit, brute-force tools, MITM tools |
| Metasploitable2 | Unmonitored victim | Intentionally vulnerable Linux target, no logging agent used for pure attack-technique practice |
| Windows 10 (eval VM) | Monitored victim | Represents a real employee workstation runs Sysmon + Wazuh agent |
| Ubuntu Server | SOC dashboard | Runs Wazuh manager collects and displays logs/alerts from Windows |

All VMs run in VirtualBox on an isolated Host-only network, no connection to my real network.

Tools used:
 1. Sysmon(SwiftOnSecurity config) deep Windows logging: process creation, network connections, registry/file changes
 2. Wazuh free/open-source SIEM, collects logs centrally and applies detection rules
 3. Nmap, Metasploit, Hydra, Bettercap (Kali, built-in) attack simulation tools
 4. MITRE ATT&CK used as the reference framework to name and structure each technique I practice


# The Loop I Follow for Each Simulation

1. Pick a technique (referencing MITRE ATT&CK where applicable)
2. Run the attack against my own lab
3. Check Sysmon/Wazuh did it get detected? What did the log actually look like?
4. If nothing was detected by default, research why, and where possible, write a basic custom detection rule
5. Document: what I did, what I saw, what I'd recommend to a SOC team watching for this


# Why This Repo Exists

I believe understanding *why* something happens matters more than memorizing tool syntax. This repo is my record of building that understanding including the mistakes and troubleshooting, not just clean final results because that's the actual skill a SOC role requires.
