# Home Lab Setup

I built this lab to practice detecting attacks like a SOC analyst would, not just launching them like a pentester would. Here's what I set up and why.

# What I built

I have 4 VMs running in VirtualBox, all on the same isolated Host-only network so they can talk to each other but nothing touches my real network.

**Kali Linux** : this is my attacker machine. Nmap comes pre-installed, and I'll use Metasploit and other tools from here later.

**Metasploitable2** : a Linux box that's built intentionally full of vulnerabilities. I attack this one when I just want to practice offense without worrying about breaking anything real. It has no monitoring on it, so it's my "unwatched server" in the lab.

**Windows VM** : this is my stand-in for a real employee's laptop. I installed Sysmon here so I can see deep activity logs (process creation, network connections, etc.) that Windows doesn't log by default. I also installed the Wazuh agent on it so its logs actually get shipped somewhere I can see them.

**Ubuntu Server (Wazuh manager)** : this is my SOC dashboard. Wazuh runs here and collects logs from the Windows agent. I log into its dashboard from my browser to actually investigate what's happening.

# Why manager and agent are separate

I was confused about this at first. The manager is the central place logs get collected and reviewed. The agent is installed on the machine being watched, and its only job is to ship logs back to the manager. You don't put both on the same machine because the whole point of a SIEM is one dashboard watching many machines, not a bunch of separate disconnected setups.

# Why Windows has an agent but Metasploitable2 doesn't

This is on purpose. Windows is my "monitored" victim, same as a real employee's laptop would have monitoring software on it. Metasploitable2 is my "unmonitored" victim, which is also realistic, plenty of real breaches happen on machines nobody thought to watch.

If a machine doesn't have Sysmon or the Wazuh agent installed, I get zero visibility into it. No agent means no logs means no detection, no matter how obvious an attack against it is.

# Problems I ran into and fixed

**Windows VM kept ending up on a different network than Kali/Ubuntu.** Kali and Ubuntu showed 192.168.56.x addresses, but Windows showed 10.0.3.x. Windows had two adapters active one Host-only, one NAT and it was routing through the wrong one. I disabled the NAT adapter so it only used Host-only, matching the others.

**Windows install got corrupted.** I changed boot order and detached the install ISO while Windows was still mid-install, during one of its automatic restarts. That broke it. Had to wipe the disk and reinstall, this time letting the whole install finish completely before touching any settings.

**Ping wasn't working between Ubuntu and Windows even after fixing the network.** Windows Firewall blocks inbound ping by default. Had to go into Windows Defender Firewall → Advanced Security → enable the "Echo Request - ICMPv4-In" rule.

**Wazuh indexer failed to start after a VM reboot, with a timeout error.** Checked the actual error with `journalctl -u wazuh-indexer`, which showed it was hitting systemd's default start timeout before it could fully boot up (it's a resource-heavy service). Fixed by adding an override with `TimeoutStartSec=500` so it gets enough minutes instead of the default.

**Agent install silently failed.** Turned out to be a typo I typed `wauh-agent.msi` instead of `wazuh-agent.msi` in the install command. Once I fixed the filename, it installed and started fine.

## How I confirmed it actually works

Went to the Wazuh dashboard, Server Management → Endpoint Summary, and saw my Windows VM listed as an "Active" agent. Then went to Explore → Discover, filtered by `agent.id: 001` (that's the Windows agent, `000` is always the manager itself), did something on Windows, and watched a real event show up with the correct timestamp. Confirmed the whole pipeline, Sysmon logs something, agent ships it, manager receives it, I can search it.

## What I actually need to remember vs just know exists

Things I want to actually keep in my head: `ip a` / `ipconfig` for checking network config, `ping` for connectivity checks, `systemctl status <service>` and `journalctl -u <service>` for checking what's running and why something failed, where Sysmon's logs live in Event Viewer, and how to search in Wazuh's Discover page.

Things I don't need memorized, just need to know they exist and I can look them up again: exact Wazuh/Sysmon install commands, the systemd override syntax.

## What's next

Lab's fully working now. Next step is running actual attacks against it starting with an Nmap scan and checking what shows up (or doesn't) in Wazuh, then writing that up the same way I did this setup.
