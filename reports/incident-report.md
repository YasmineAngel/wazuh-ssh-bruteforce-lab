# Incident Report: SSH Brute-Force Detection and Response

Date: 8 October 2026
Classification: Lab exercise / Portfolio
Analyst: Yasmine
SIEM: Wazuh 4.14.8

## 1. Summary

On 8 October 2026, an SSH brute-force attack was launched from an internal host
(192.168.56.103) against a monitored Ubuntu server (192.168.56.102) in an isolated lab.
The attacker made roughly a dozen rapid password-guessing attempts against the `analyst`
account over SSH. Wazuh detected the activity in real time, escalating from individual
failed-login alerts (rule 5760, level 5) to a brute-force correlation alert (rule 5763,
level 10) and, when a guess succeeded, to a successful-breach-after-failures alert
(rule 40112, level 12). An Active Response rule automatically blocked the attacker's IP
at the host firewall for 120 seconds. On a repeat attack with Active Response enabled,
the attacker was unable to complete the login even with the correct password, because the
firewall dropped its traffic mid-attack. Total time from first failed login to automated
block: under one minute.

## 2. Environment

The lab is fully isolated on a VirtualBox host-only network (192.168.56.0/24) with no route
to the internet or any production network. All three hosts are virtual machines on a single
workstation.

| Host | Role | IP | OS / Tooling |
| --- | --- | --- | --- |
| wazuh-server | SIEM (manager, indexer, dashboard) | 192.168.56.101 | Wazuh 4.14.8 OVA (Amazon Linux 2023) |
| ubuntu-victim | Monitored target | 192.168.56.102 | Ubuntu Server 24.04 + Wazuh agent + OpenSSH |
| kali-attacker | Attacker | 192.168.56.103 | Kali Linux + Hydra 9.6 |

The Wazuh agent on the Ubuntu host forwards its logs (including SSH authentication events
from journald) to the Wazuh manager, which applies detection rules and triggers responses.

## 3. Timeline

All times are from the Wazuh manager (UTC). The attacker host clock was on EDT; events are
the same instants shown in the SIEM's own time.

| Time (UTC) | Event |
| --- | --- |
| 10:19:25 | First failed authentication from 192.168.56.103 (rule 5503 / 5760, level 5) |
| 10:19:25-10:19:31 | Repeated failed SSH logins for user `analyst`, ~8 attempts in a burst |
| 10:19:31 | Brute-force pattern recognized (rule 5763, level 10, MITRE T1110) |
| 10:19:33 | Successful login after failures flagged (rule 40112, level 12) -- attacker guessed the password |
| 10:45:48 | Repeat attack with Active Response enabled; brute force re-detected (rule 5763) |
| 10:45:49 | Active Response fires: firewall-drop add for 192.168.56.103 |
| 10:47:50 | Block auto-removed after 120s timeout (firewall-drop delete) |

In the first run the attacker succeeded (detection only). In the second run, with Active
Response enabled, the attacker's correct password never completed because its traffic was
dropped during the attack: Hydra reported 0 valid passwords found.

## 4. Detection details

Wazuh detected the attack through a chain of rules that escalate in severity as the pattern
becomes clearer. This escalation, from noise to confirmed intrusion, is the core value of a SIEM.

| Rule ID | Level | Description | Meaning |
| --- | --- | --- | --- |
| 5503 | 5 | PAM: user login failed | OS-level authentication failure |
| 5760 | 5 | sshd: authentication failed | One failed SSH login (one Hydra guess) |
| 5763 | 10 | sshd: brute force trying to get access | Correlation: many failures from one source |
| 40112 | 12 | Multiple authentication failures followed by a success | Likely successful compromise |

MITRE ATT&CK mapping (applied automatically by Wazuh):

- T1110 - Brute Force (tactic: Credential Access): the password-guessing activity.
- T1078 - Valid Accounts: flagged when the attacker's guessing ended in a valid login.

The level-12 alert (rule 40112) is the critical one: it signals not just an attempt but a
probable breach, which in production would page an on-call analyst immediately.

## 5. Response

An Active Response rule was configured on the Wazuh manager to block the source IP
automatically when rule 5763 (brute force) fires:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>120</timeout>
</active-response>
```

When triggered, the manager instructs the agent on the targeted host to run the built-in
firewall-drop command, which inserts an iptables rule dropping all traffic from the attacker.
After the 120-second timeout, the agent removes the rule automatically.

The active-responses log on the victim confirmed the full cycle: a firewall-drop "command":"add"
entry naming 192.168.56.103, followed 120 seconds later by a matching "command":"delete".
The effect was decisive: in the second attack the correct password was in the wordlist, yet
Hydra reported 0 valid passwords found, because the firewall dropped the attacker before the
winning attempt could complete.

The 120-second timeout is deliberate for a lab (it lets the block be observed and re-tested).
In production this would typically be longer, or paired with a permanent block and an analyst review.

## 6. Impact assessment

In the detection-only run, the attack succeeded: the attacker guessed the `analyst` password
and obtained an SSH session. In a real environment this would mean a foothold on the host, with
whatever privileges that account holds (and, via sudo, potentially root). From there an attacker
could pivot, exfiltrate data, or deploy persistence.

The attack was only possible because the target allowed password authentication with a weak,
guessable password and placed no rate limit on SSH login attempts. The brute force was fast
(dozens of attempts in seconds), which is typical of automated tools and exactly what the SIEM
is tuned to catch.

With Active Response enabled, the practical impact dropped to near zero: the attacker was blocked
before completing a successful login. Detection alone gives visibility; detection plus response
contains the threat without waiting for a human.

## 7. Recommendations

To prevent or blunt this attack in a production environment:

1. Disable SSH password authentication and require SSH keys. This removes brute force as a viable path entirely.
2. Enforce strong, unique passwords and account lockout where passwords must remain.
3. Rate-limit SSH at the network or host (for example fail2ban, or firewall throttling on port 22).
4. Restrict SSH exposure: bind to management networks or require a VPN/bastion rather than exposing port 22 broadly.
5. Keep Active Response, but tune it: longer or escalating block times, and alerting so an analyst reviews repeat offenders.
6. Monitor and alert on rule 40112 (success after failures) as a high-priority signal of probable compromise.
7. Enable MFA for remote access where supported.

## 8. Evidence

The following artifacts support this report (stored in the repository):

| File | What it shows |
| --- | --- |
| screenshots/dashboard-overview.png | Threat Hunting overview: 13 events, 12 auth failures, MITRE donut |
| screenshots/events-table.png | Event table with rule IDs 5503, 5760, 5763, 40112 and timestamps |
| screenshots/active-response-log.png | active-responses.log showing firewall-drop add for 192.168.56.103 |
| scripts/hydra-run-detection.txt | Hydra run that succeeded (detection only) |
| scripts/hydra-run-blocked.txt | Hydra run with Active Response: 0 valid passwords found |
| configs/active-response.xml | The Active Response block added to ossec.conf |
| reports/active-response-proof.txt | Extracted block log entries |
| reports/active-responses-raw.txt | Full raw active-responses.log |

Note: this was conducted entirely within an isolated, self-owned lab. No external or
third-party systems were targeted.
