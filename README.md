<h1 align="center">🛡️ Home SOC Lab: Linux-Only</h1>

<h3 align="center">Wazuh SIEM · MITRE ATT&amp;CK attack simulation · NIST incident response</h3>

<p align="center">
  <img alt="Status: complete" src="https://img.shields.io/badge/status-complete-brightgreen">
  <img alt="Wazuh 4.9.2" src="https://img.shields.io/badge/Wazuh-4.9.2-blue">
  <img alt="MITRE ATT&CK" src="https://img.shields.io/badge/MITRE-ATT%26CK-red">
  <img alt="NIST SP 800-61" src="https://img.shields.io/badge/NIST-SP%20800--61-lightgrey">
</p>

<p align="center">
  <a href="#headline-results">Results</a> ·
  <a href="#environment">Environment</a> ·
  <a href="#attack-chain-and-detection-coverage">Detection coverage</a> ·
  <a href="#key-findings">Findings</a> ·
  <a href="#lessons-learned">Lessons learned</a>
</p>

---

## Overview

A home-built Security Operations Center lab: a Wazuh SIEM, auditd and file integrity monitoring (FIM) telemetry, and a MITRE ATT&CK-mapped attack simulation, all built on resource-constrained hardware. A red team workstation brute-forced SSH and simulated ransomware, and the blue team detected, investigated and recovered using the NIST incident response lifecycle.

**Status:** ✅ Complete: attack simulated, detected, investigated and written up

📄 **Full incident report:** [`incident-reports/Incident_Report_Home_SOC_Lab.pdf`](incident-reports/Incident_Report_Home_SOC_Lab.pdf)

> All activity was an authorised exercise on an isolated host-only network. The "ransomware" only renames files to `.locked`; nothing destructive was used.

---

## Headline results

| Metric | Result |
|---|---|
| First failed login to level 10 escalation (rule 5763) | **84 s** |
| First file delete to ransom note (attack duration) | **64 ms** |
| First delete to level 12 ransomware alert (custom rule 100041) | **43 ms** |
| Files restored with original sizes and dates | **4 / 4** |

Wazuh detected every stage of the attack at least once. The review also found real gaps (a false positive, a brute-force threshold gap and lost SSH login events), each with a fix or recommendation in the report.

---

## Environment

| Host | Role | RAM | Disk | IP |
|---|---|---|---|---|
| `ubuntu-victim` | Monitored endpoint (Wazuh agent 001), auditd, realtime FIM, SSH | 1024 MB | 15 GB | `192.168.56.20` |
| `wazuh-server` | Wazuh manager, indexer and dashboard | 2048 MB | 25 GB | `192.168.56.5` |
| Red team host | Windows workstation running PowerShell and plink (PuTTY CLI) | n/a | n/a | `192.168.56.1` |

All hosts sit on an isolated VirtualBox host-only network (`192.168.56.0/24`). The Wazuh server runs below the recommended 4 GB RAM / 2 CPU, installed with the `-i` flag to bypass the hardware check, so the lab fits the host machine.

---

## What was built

- **Phase 1: Ubuntu victim VM.** Ubuntu Server, static IP via netplan, connectivity verified between VMs.
- **Phase 2: Endpoint telemetry.** Sysmon for Linux installed; auditd command logging and realtime FIM on `/home/jdoe/company_docs` configured for detection.
- **Phase 3: Wazuh server.** Wazuh 4.9.2 (indexer, manager, dashboard) on a 2 GB VM. *(Add 1–2 lines here on how you fixed the dashboard install failure.)*
- **Phase 4–5: Agent and pipeline.** Victim agent connected to the manager and the alert pipeline verified end to end.
- **Phase 6: Attack simulation.** SSH brute force, valid-account login, discovery and ransomware-style impact.
- **Phase 7: Investigation and MITRE ATT&CK mapping.** Alerts analysed and mapped to techniques.
- **Phase 8: Incident report.** Full red team / blue team report following NIST SP 800-61.

---

## Attack chain and detection coverage

| Red team action | MITRE | Wazuh rule | Result |
|---|---|---|---|
| SSH brute force | T1110 | 5760, 5503, 5551, 5763 | Detected (built-in; 84 s to level 10) |
| Login after failures | T1078 | 40112 (level 12) | Detected (built-in) |
| Login after 7 failures (test burst) | T1078 | 5760, 5503 only | **Missed** (below threshold; success event lost) |
| Discovery in `company_docs` | T1083, T1005 | 100040 (level 9) | Detected (custom rule) |
| Rename to `.locked` | T1486 | 100041 (level 12) | Detected (custom rule; 43 ms after first delete) |

### Custom detection rules

- **100040** (level 9): flags any audited command touching `company_docs`.
- **100041** (level 12): flags three file deletions within 60 s, built on rule 553.

Rule source: [`detection-rules/local_rules_final.xml`](detection-rules/local_rules_final.xml)

---

## Key findings

| # | Finding | Priority | Status |
|---|---|---|---|
| 6.1 | Rule 100041 cannot tell an admin restore from ransomware (false positive) | High | Open |
| 6.2 | Default brute-force threshold leaves a 7-failure burst at level 5 | Medium | Open |
| 6.3 | Successful SSH logins lost to journald rotation | Medium | **Fixed and confirmed** (`/var/log/auth.log` added as a second source; confirmed by reboot test) |
| 6.4 | Detection is faster than any manual response; containment must be automatic | High | Open |
| 6.5 | Weak password accepted for `jdoe` | High | Open |

The full evidence, screenshots and action plan are in the report.

---

## Repo structure

```
home-soc-lab/
├── README.md
├── incident-reports/
│   └── Incident_Report_Home_SOC_Lab.pdf
├── detection-rules/
│   └── local_rules_final.xml
├── evidence/                 # log extracts referenced in the report
├── screenshots/              # optional: key figures from the report
└── lessons-learned.md        # optional
```

---

## Lessons learned

### Build phase
- **`sudo` only applies to the command immediately following it.** Chaining `sudo cmd1 && cmd2` does not run `cmd2` as root.
- **`curl -O` (save to file) vs `curl -0` (force HTTP/1.0)** look almost identical but behave very differently; confusing them cost significant troubleshooting time.
- **The `-s` flag on `curl` can mask a failed download.** Drop it when debugging.
- **On a 2 GB RAM SIEM, individual install phases (the indexer especially) can take 10+ minutes.** That is expected, not a hang.

### Detection phase
- **Speed is the real constraint.** Detection at +43 ms is not useful for containment unless the response is automatic.
- **Default thresholds are tuned for volume, not for a careful attacker.** Seven failures followed by a success can pass at level 5.
- **Test detections against the matching admin action.** Rule 100041 caught the attack but also fired on the legitimate restore.
- **Logging can fail silently.** A missing alert was only found by comparing Wazuh against the source log (`journalctl`).

---

## Tooling

- VirtualBox
- Ubuntu Server 22.04/24.04 LTS
- Wazuh 4.9.2 (SIEM: indexer, manager, dashboard)
- auditd and Wazuh FIM (realtime)
- Sysmon for Linux (Microsoft)
- PowerShell and plink (red team host)
- MITRE ATT&CK, NIST SP 800-61

---

## Author

Senamile N Maphoso: SOC analyst (blue team) and red team operator
