# Operation-Blue-Sentinel-Soc Monitoring & Incident Response Lab
A hands-on SOC engineering lab demonstrating real-world threat detection, threat intelligence, and incident response using Wazuh, VirusTotal, and GoPhish across Linux and Windows environments.

**Engineer:** Adewale Adejoju
**Commander/Analyst:** Terna Akin-Akinbisola
**Facilitator:** Raymond Ebonine
**Last updated:** 2026-07-09

A hands-on SOC build validating a Wazuh + VirusTotal detection pipeline against real internet-background attack traffic and a simulated phishing → brute-force incident, from initial log ingestion through detection, containment, and eradication.

---

## TL;DR

A single exposed SSH port drew 35+ real brute-force attempts over the course of this lab. Zero credentials were compromised. The detection pipeline — Wazuh custom rules + VirusTotal enrichment — caught the activity at the first alert, and the incident was formally declared, contained, and eradicated following NIST SP 800-61 phases.

**Takeaway:** budget isn't the barrier to basic security maturity for resource-constrained orgs (Nigerian SMEs/fintechs included) — visibility and process discipline are. This lab used entirely free/open-source tooling (Wazuh, VirusTotal free tier, GoPhish).

---

## Architecture

<!-- Insert the draw.io diagram from Day 6-7 here -->
![Architecture diagram](./screenshots/architecture-nist.png)

- **Linux VM** (`tpp`) — Azure-hosted, Wazuh agent reporting live to Manager
- **Windows VM** — secondary endpoint, validated separately for cross-platform coverage
- Log flow: `sshd` → syslog → Wazuh agent → Manager (rules + MITRE ATT&CK mapping + severity scoring)

---

## Log Source: SSH Authentication (syslog)

Every SSH connection attempt is logged by `sshd`; Wazuh's agent reads and enriches these events with rule matching, severity, and MITRE mapping.

**Observed pattern:** failed logins using randomized/default usernames (`admin1`, `deployer`, `mohit`, `oracle`) from geographically distributed external IPs — consistent with automated credential-guessing scans, not a targeted attack.

| Detail | Value |
|---|---|
| MITRE ATT&CK mapping | Credential Access — T1110.001 (Password Guessing) |
| Rule level observed | 5 (stable, no escalation pattern) |
| Screenshot evidence | 804 SSH auth-failure hits in 1 hour, agent `tpp`, Jul 7 2026 |

At rule level 5, this is background noise requiring monitoring — not urgent response. It becomes a priority the moment a *real* username is targeted or a login *succeeds*.

![SSH auth failures in Discover](./screenshots/ssh-auth-discover.png)

---

## VirusTotal Integration & Custom Detection Rules

Wazuh's VirusTotal integration checks IP/file reputation against 70+ independent security vendors. Two custom rules were added on top of the default set:

| Rule ID | Meaning | Status |
|---|---|---|
| 87100–87106 | Built-in Wazuh VirusTotal rules (rate limits, credential errors, clean/malicious verdicts) | Default |
| **100092** | Custom: alerts when VirusTotal flags an IP as malicious (level 12) | Added |
| **100093** | Custom: alerts when an IP has positive/suspicious detections (level 8) | Added |

### Validation — IP Reputation Testing

Three IPs were tested end-to-end, including a known-clean control (`8.8.8.8`) to confirm the integration discriminates correctly rather than flagging indiscriminately:

| IP | Malicious detections | Suspicious detections | Verdict |
|---|---|---|---|
| `185.220.101.45` | 17 | 3 | 🔴 Malicious |
| `8.8.8.8` (Google DNS — control) | 0 | 0 | 🟢 Clean |
| `45.33.32.156` | 3 | 0 | 🔴 Malicious |

`185.220.101.45`'s 17-vendor consensus represents high-confidence agreement across independent engines, vs. the lower-confidence 3-vendor flag on `45.33.32.156` — both actionable, but weighted differently in analyst triage.

![IP reputation test terminal output](./screenshots/vt-ip-test.png)
![Wazuh dashboard flagged IPs](./screenshots/vt-dashboard.png)

---

## Cross-Platform Validation — Windows Endpoint

Two failed login attempts were simulated on the Windows VM to confirm Wazuh's Windows Security Event Log collection and alerting worked, not just the Linux SSH path. Wazuh generated a dashboard alert as expected.

![Windows failed login alert](./screenshots/windows-alert.png)

> Rule ID / exact severity for this alert: *[fill in from dashboard]*

---

## Incident Timeline — Phishing → Brute-Force (NovaMart Scenario)

Following NIST SP 800-61 phases: Preparation → Detection → Containment → Eradication.

| Phase | What happened |
|---|---|
| **Prep** | NIST SP 800-61 review; architecture diagram drafted |
| **Declaration** | GoPhish-simulated phishing campaign executed against NovaMart target. Custom rule "NovaMart: Failed login attempt" fired; earliest detection **Jul 12, 2026 @ 06:55:11.045**. Incident Commander opened the Incident Log. |
| **Detection** | Sustained brute-force activity recorded against the target, zero successful logins at any point — full timestamps captured via Wazuh Discover. Detected at first attempt, not after escalation. |
| **Containment** | Incident Commander formally declared the incident and authorized containment. Security Engineer applied a firewall block against the malicious source, targeting the exposed SSH port. |
| **Eradication** | Root cause: exposed/weak SSH port configuration. 35+ sustained attempts, zero credentials compromised. Vulnerable port closed/restricted. Post-eradication scan confirmed a clean environment. |

**Key finding:** a real, sustained attack (35+ attempts across multiple days) exploited a genuine configuration weakness — the SOC stack detected, contained, and eradicated it with zero successful breach. This validates the pipeline under adversarial pressure, not just synthetic testing.

---

## Conclusion & Recommendation

Operation Blue Sentinel established a functional SOC spanning an Azure-hosted Linux VM and a Windows VM, enriched with VirusTotal threat intelligence and validated through a live simulated phishing attack.

Despite 35+ real brute-force attempts against a genuinely exposed port, zero credentials were compromised. The incident was detected at first alert, formally declared, contained via firewall action, and eradicated through root-cause remediation — confirmed clean by a post-eradication scan.

**For resource-constrained orgs** (Nigerian SMEs and fintechs included): attackers don't need sophistication to cause damage — a single exposed port drew 35+ attempts. What determined the outcome wasn't the absence of a vulnerability, but the presence of monitoring, alerting, and a team ready to act. Low-cost, open-source tooling (Wazuh, VirusTotal free tier, GoPhish) was sufficient to detect and stop a real attack.

> **Recommendation:** Close all non-essential exposed ports as standard practice, not only as an incident-response measure. Detection worked here — but prevention would have made this a non-event.

---

## Stack

- **SIEM:** Wazuh (Manager + Agents)
- **Threat Intel:** VirusTotal API (free tier)
- **Phishing Simulation:** GoPhish
- **Infra:** Azure (Linux VM), Windows VM
- **Framework:** NIST SP 800-61 (Incident Handling Guide)

## Repo Structure

```
├── README.md
├── screenshots/
│   ├── architecture-nist.png
│   ├── ssh-auth-discover.png
│   ├── vt-ip-test.png
│   ├── vt-dashboard.png
│   └── windows-alert.png
└── rules/
    └── custom_rules.xml        # 100092 / 100093 rule definitions
```
