# Lab Build Guide

A step-by-step guide to reproduce this lab: a Wazuh SIEM detecting (and optionally
blocking) an SSH brute-force attack, using three VirtualBox VMs on an isolated
host-only network.

> All activity stays inside a self-contained lab with no route to the internet or any
> other network. Only attack machines you own.

## Overview

| VM | Role | IP (example) | Stack |
| --- | --- | --- | --- |
| wazuh-server | SIEM (manager, indexer, dashboard) | 192.168.56.101 | Wazuh 4.14.8 OVA |
| ubuntu-victim | Monitored target | 192.168.56.102 | Ubuntu Server 24.04 + agent + OpenSSH |
| kali-attacker | Attacker | 192.168.56.103 | Kali Linux + Hydra |

## Prerequisites

- A 64-bit host with hardware virtualization enabled, ~16 GB RAM, ~80 GB free disk.
- VirtualBox installed.
- The Wazuh OVA and an Ubuntu Server ISO downloaded. (Verify the OVA against its
  published SHA512 checksum before importing.)

## 1. Create the isolated network

Create a VirtualBox host-only network and enable its DHCP server so VMs get addresses
automatically. Example range: server IP `192.168.56.2`, leases `192.168.56.101-200`,
mask `255.255.255.0`. The host itself sits at `192.168.56.1`.

```powershell
$vbox = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"
$if   = "VirtualBox Host-Only Ethernet Adapter"
& $vbox dhcpserver add --interface="$if" --server-ip=192.168.56.2 `
  --netmask=255.255.255.0 --lower-ip=192.168.56.101 --upper-ip=192.168.56.200 --enable
```

## 2. Import the Wazuh server

Import the OVA and set VirtualBox-recommended options: VMSVGA graphics, hardware clock
in UTC (keeps alert timestamps accurate), and attach it to the host-only network.

```powershell
& $vbox import .\wazuh-4.14.8.ova --vsys 0 --vmname "wazuh-server"
& $vbox modifyvm "wazuh-server" --memory 8192 --cpus 4 `
  --graphicscontroller vmsvga --rtcuseutc on `
  --nic1 hostonly --hostonlyadapter1 "$if"
& $vbox startvm "wazuh-server"
```

Find its IP (match the VM's MAC in the host ARP table), then confirm services are up over SSH:

```bash
ssh wazuh-user@192.168.56.101          # default password: wazuh
systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard
```

Access the dashboard at `https://<server-ip>` (default `admin` / `admin`). The
self-signed certificate warning is expected in a lab.

## 3. Build and enroll the Ubuntu victim

Install Ubuntu Server with OpenSSH enabled, on the host-only network. Then install the
Wazuh agent pointed at the manager. Because the lab has no internet, download the agent
package on a connected host and copy it over (`scp`), then install:

```bash
sudo WAZUH_MANAGER='192.168.56.101' WAZUH_AGENT_GROUP='default' \
  dpkg -i wazuh-agent_4.14.8-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

Confirm the agent shows as **Active** in the dashboard under Agents.

## 4. Prepare the attacker

Set the Kali VM's network adapter to the same host-only network. Confirm reachability
and that Hydra is installed:

```bash
ping -c 3 192.168.56.102
hydra -h
```

## 5. Run the attack

Create a small wordlist and run Hydra against the victim's SSH. Place a valid password
last so failures precede the success:

```bash
hydra -l analyst -P passwords.txt ssh://192.168.56.102 -t 4 -V
```

## 6. Review the detection

In the dashboard, open **Threat Hunting** and filter by the attacker's IP
(`data.srcip:192.168.56.103`). Add `rule.id`, `rule.level`, and `rule.description` as
columns. The expected escalation:

| Rule ID | Level | Description |
| --- | --- | --- |
| 5503 | 5 | PAM: user login failed |
| 5760 | 5 | sshd: authentication failed |
| 5763 | 10 | sshd: brute force trying to get access |
| 40112 | 12 | Multiple authentication failures followed by a success |

Wazuh maps these to MITRE ATT&CK T1110 (Brute Force) and T1078 (Valid Accounts).

## 7. (Optional) Automatic blocking with Active Response

On the manager, add an Active Response block to `/var/ossec/etc/ossec.conf` that fires
the built-in `firewall-drop` command when rule 5763 triggers:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>120</timeout>
</active-response>
```

Restart the manager, then re-run the attack with enough attempts to last past the
detection threshold. The agent inserts an iptables rule dropping the attacker's traffic,
visible in `/var/ossec/logs/active-responses.log` as a `firewall-drop ... "command":"add"`
entry, removed automatically after the timeout.

## Notes

- Back up `ossec.conf` before editing it.
- Shut VMs down cleanly (ACPI shutdown) rather than closing the window with a hard power-off.
- Keep real IPs and credentials out of any screenshots published publicly.
