# SIEM Detection Engineering Lab — MITRE ATT&CK Mapped Rules

A hands-on SOC/detection-engineering lab: deployed Wazuh as a two-node SIEM
(manager + monitored endpoint), simulated real attack techniques, identified
gaps in the default ruleset, and wrote/debugged a custom detection rule
mapped to MITRE ATT&CK -- validated end-to-end against live attack traffic.

## Why this project

Most beginner security portfolios stop at "I installed a SIEM." This lab
goes further: it documents *what the default ruleset caught, what it
didn't, and the debugging process* behind a custom rule that closes that
gap -- which is the actual day-to-day work of a SOC analyst / detection
engineer.

## Architecture

```
Attacker (Windows host, host-only network)
        │  SSH brute-force / login attempts
        ▼
Target VM  "target-ubuntu"  (192.168.56.20)
  - Ubuntu Server 24.04
  - Wazuh Agent
        │  forwards logs
        ▼
Manager VM  "wazuh-manager"  (192.168.56.10)
  - Wazuh Manager + Indexer + Dashboard
        │
        ▼
  Dashboard (https://192.168.56.10) -- alerts, agent status, MITRE view
```

Both VMs run in VirtualBox on a single host machine (8GB RAM), networked
via a Host-Only Adapter (VM-to-VM / VM-to-host) plus NAT (internet access).

## Techniques Covered

| MITRE ID | Technique | Detected by Default? | Custom Rule? | Validated? |
|----------|-----------|------------------------|---------------|------------|
| T1110.001 | Brute Force: Password Guessing | Partial (per-event only, rules 5710/5712) | Yes | **Fired live, 2 alerts** |
| T1110.001 / T1021.004 | Successful login after brute-force burst | Partial | Yes | Deployed, correlation timing still being tuned |
| T1078 | Valid Accounts (test account creation/misuse) | Yes (default rules 5901, 5904) | -- | Observed |

Full detection logic, reasoning, and the real debugging trail:
[`docs/mitre_mapping.md`](docs/mitre_mapping.md)

## Repo Structure

```
siem-detection-lab/
├── README.md
├── rules/
│   └── local_rules.xml       # custom Wazuh detection rules (validated)
├── docs/
│   ├── setup_guide.md         # full manager + agent install walkthrough
│   ├── mitre_mapping.md       # coverage table, debugging notes, test steps
│   └── screenshots/           # dashboard + alert evidence
```

## Setup

Full walkthrough (manager install, agent install, static networking,
troubleshooting): [`docs/setup_guide.md`](docs/setup_guide.md)

Quick reference:

```bash
# --- On the manager VM ---
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash ./wazuh-install.sh -a -i

# --- On the target VM (point at the manager's IP) ---
curl -sO https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb
sudo WAZUH_MANAGER='<manager-ip>' dpkg -i wazuh-agent_4.7.5-1_amd64.deb
sudo systemctl enable --now wazuh-agent
```

## Lessons Learned

- **Rule IDs aren't always what the docs/convention suggest.** Two guesses
  (`5716`, then `5760`) were wrong before `sudo /var/ossec/bin/wazuh-logtest`
  confirmed the real rule ID (`5710`) that Ubuntu 24.04 + OpenSSH actually
  triggers for a login attempt against a non-existent user. Testing against
  representative log lines beats guessing from documentation.
- **Networking is most of the setup work.** VirtualBox's NAT vs. Host-Only
  adapters, DHCP not being enabled on the host-only network, and a hostname
  collision between the two VMs (both defaulted to the same name, causing
  Wazuh to reject the second agent as a duplicate) each cost real debugging
  time -- and each mirrors a real infrastructure problem, not a toy one.
- **Severity levels tell a story on their own.** Comparing what Wazuh
  assigned to a routine login (level 3), a single wrong sudo password
  (level 5), a new group/account being created (level 8), and a correlated
  brute-force burst (level 10, via the custom rule) made the "noise vs.
  signal" distinction concrete rather than theoretical.
- **Default rules cover more than expected.** Before writing any custom
  rule, Wazuh's built-in ruleset already flagged the SSH brute-force
  attempt and the test-account creation. The value of a custom rule is in
  *raising confidence and adding correlation/MITRE context*, not in
  detecting something from nothing.

## Next Steps

- Tune the timing window on rule 100012 (successful login after brute-force)
  and confirm it fires end-to-end
- Add a Windows target with Sysmon to validate the PowerShell detection
  rules (encoded command, hidden window, download-execute) written for
  this lab but not yet tested live
- Add T1053.005 (Scheduled Task) and T1003.001 (LSASS credential dumping)
  once a Windows target is available

---
*Built as a hands-on cybersecurity portfolio project. Rules and setup are
for lab/educational use.*
