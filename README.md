# Home SOC Lab — Linux-Only  

**Status:** 🚧 In Progress — Phase 3 (Wazuh Server Installation)

A home-built Security Operations Center lab: Wazuh SIEM, Sysmon-for-Linux
telemetry, and MITRE ATT&CK-mapped attack simulation, built on
resource-constrained hardware . This repo will be updated
as each phase is completed.

---

## Environment

| VM | Role | RAM | Disk | Static IP |
|---|---|---|---|---|
| `Ubuntu-Victim` | Attack target / telemetry source | 1024 MB | 15 GB | `192.168.56.20` |
| `Wazuh-Server` | SIEM (indexer, manager, dashboard) | 2048 MB | 25 GB | `192.168.56.5` |

Both VMs run on an isolated VirtualBox host-only network
(`192.168.56.0/24`), with a temporary NAT adapter attached to each for
package downloads.

## Progress So Far

### ✅ Phase 1 — Ubuntu Victim VM
- Built `Ubuntu-Victim` (Ubuntu Server, 1024 MB RAM, 1 CPU, 15 GB disk)
- Host-only + NAT adapters configured
- Static IP assigned (`192.168.56.20`) via netplan
- Confirmed connectivity between VMs (`ping` — 0% packet loss)

### ✅ Phase 2 — Sysmon for Linux
- Installed Microsoft's Sysmon for Linux on `Ubuntu-Victim`
- Verified service running and logging live activity via `journalctl`

### 🚧 Phase 3 — Wazuh Server (in progress)
- Built `Wazuh-Server` VM (2048 MB RAM, 1 CPU, 25 GB disk)
- Static IP assigned (`192.168.56.5`)
- System packages updated
- Downloaded `wazuh-install.sh` (v4.9.2)
- Ran installer with `-i` flag to bypass the minimum hardware check
  (installer recommends 4 GB RAM / 2 CPU; this lab intentionally runs
  below that to fit the host machine)
- **Current blocker:** Wazuh indexer, manager, and Filebeat all installed
  and started successfully — but the **Wazuh dashboard installation step
  failed**, triggering the installer's automatic rollback (it removed the
  manager, indexer, and Filebeat it had just installed).
- Actively investigating the root cause via `/var/log/wazuh-install.log`
  and system memory logs (`dmesg`) — suspected RAM exhaustion during the
  dashboard's Node.js asset build step, given the constrained 2048 MB
  allocation.

### ⬜ Not Yet Started
- Phase 4 — Connect victim agent to Wazuh
- Phase 5 — Verify full pipeline
- Phase 6 — Run attack simulations
- Phase 7 — Investigate alerts & MITRE ATT&CK mapping
- Phase 8 — Incident reports
- Phase 9/10 — GitHub & LinkedIn publishing

---

## Lessons Learned (so far)

- **`sudo` only applies to the command immediately following it** —
  chaining `sudo cmd1 && cmd2` does *not* run `cmd2` as root. Each command
  needing elevated privileges needs its own `sudo`.
- **`curl -O` (capital O, save to file) vs `curl -0` (zero, force
  HTTP/1.0)** look nearly identical in most terminal fonts but do
  completely different things — the latter dumps file contents straight
  to the terminal instead of saving them, which cost significant
  troubleshooting time before being caught.
- The `-s` (silent) flag on `curl` suppresses error output, which can mask
  a failed download as if nothing happened. Useful to drop that flag when
  debugging.
- On a 2 GB RAM SIEM server, individual install phases (indexer especially)
  can take 10+ minutes — this is expected, not a hang.

---

## Repo Structure (planned)

```
home-soc-lab/
├── README.md
├── architecture-diagram.png
├── incident-reports/
├── detection-rules/
│   └── local_rules.xml
├── mitre-mapping/
│   └── attack-navigator-layer.json
├── screenshots/
└── lessons-learned.md
```

*(Folders will be populated as later phases are completed.)*

---

## Tooling

- VirtualBox
- Ubuntu Server 22.04/24.04 LTS
- Sysmon for Linux (Microsoft)
- Wazuh 4.9.2 (SIEM — indexer, manager, dashboard)

---

*This README will be updated as the lab progresses through each phase.*
