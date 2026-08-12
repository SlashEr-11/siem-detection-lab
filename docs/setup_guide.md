# Lab Setup Guide

This is the final, working sequence -- written after iterating through
several networking issues (documented in "Troubleshooting notes" at the
end). Follow this order to avoid them.

## Architecture

```
[Windows Host]  ---(SSH, browser)--->  [VirtualBox Host-Only Network: 192.168.56.0/24]
                                                  │
                        ┌─────────────────────────┴─────────────────────────┐
                        │                                                   │
              wazuh-manager (192.168.56.10)                target-ubuntu (192.168.56.20)
              - Wazuh Manager/Indexer/Dashboard              - Wazuh Agent
```

Both VMs also have a NAT adapter (Adapter 1) for internet access during
package installs -- only the Host-Only adapter (Adapter 2) is used for
VM-to-VM and VM-to-host traffic.

## 1. Prerequisites

- VirtualBox installed on the host
- Ubuntu Server 24.04 LTS ISO downloaded
- A VirtualBox Host-Only Network already exists (check under
  Tools -> Network Manager, or it's auto-created the first time a VM
  requests a Host-Only Adapter)

## 2. Create each VM (manager, then target)

Repeat for both `wazuh-manager` and `target-ubuntu`:

- New VM: Linux / Ubuntu 64-bit
- Manager: 2048 MB RAM, 20 GB disk. Target: 1024 MB RAM, 15 GB disk.
- Settings -> Storage: attach the Ubuntu Server ISO
- Settings -> Network:
  - Adapter 1: NAT
  - Adapter 2: Enable, attach to Host-Only Adapter

## 3. Install Ubuntu with a static IP set during install

This avoids the DHCP/netplan issues encountered in earlier attempts.

During the installer's **Network configuration** screen, two interfaces
appear (e.g. `enp0s3`, `enp0s8`). Select the host-only one (`enp0s8`) and
choose **Edit IPv4 -> Manual**:

- Subnet: `192.168.56.0/24`
- Address: `192.168.56.10` for the manager, `192.168.56.20` for the target
- Gateway: leave blank
- Name servers: `8.8.8.8`

Continue through the rest of the installer:
- Use entire disk for storage
- Set a **unique hostname** per machine (e.g. `wazuh-manager` and
  `target-ubuntu` -- do not leave both on a default/identical hostname,
  see troubleshooting notes)
- Set username/password, write it down
- **Check "Install OpenSSH server"** on every VM

## 4. Confirm networking after first boot

```bash
ip a
```
Confirm the host-only interface shows the static IP you set.

## 5. Connect from Windows via SSH (recommended over the VirtualBox console)

```powershell
C:\Windows\System32\OpenSSH\ssh.exe <username>@<vm-ip>
```

If `ssh` isn't found in PowerShell, enable the Windows OpenSSH client
(admin PowerShell, then reboot):
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

## 6. Install Wazuh Manager (on the manager VM only)

```bash
sudo apt update && sudo apt upgrade -y
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash ./wazuh-install.sh -a -i
```
`-a` installs manager + indexer + dashboard together. `-i` skips the OS
compatibility check (Ubuntu isn't on Wazuh's officially tested list but
runs fine). Takes 10-20 minutes. Save the admin password from the summary
block at the end; if missed, retrieve it later with:
```bash
tar -xvf wazuh-install-files.tar
cat wazuh-install-files/wazuh-passwords.txt
```

## 7. Install Wazuh Agent (on the target VM only)

```bash
curl -sO https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb
sudo WAZUH_MANAGER='192.168.56.10' dpkg -i wazuh-agent_4.7.5-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Confirm it registered:
```bash
sudo tail -f /var/ossec/logs/ossec.log
```
Look for `Connected to the server`.

## 8. Confirm in the dashboard

Browser -> `https://192.168.56.10` -> accept the self-signed cert warning
-> log in as `admin` -> Agents section should show `target-ubuntu` as
**Active**.

## 9. Deploy custom detection rules

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```
Paste the contents of [`rules/local_rules.xml`](../rules/local_rules.xml),
save, then:
```bash
sudo systemctl restart wazuh-manager
```

Before assuming a rule ID is correct, validate it:
```bash
sudo /var/ossec/bin/wazuh-logtest
```
Paste a representative log line and confirm which rule ID actually fires
-- see `docs/mitre_mapping.md` for why this step matters in this lab.

## Troubleshooting notes (issues actually hit while building this lab)

- **`ping`/dashboard unreachable after setup**: usually Adapter 1 and
  Adapter 2 attachment types were swapped (Adapter 1 must be NAT, Adapter
  2 must be Host-Only) -- check Settings -> Network on the affected VM.
- **`enp0s8` has no IPv4 address after boot**: the Host-Only network's
  DHCP server may be disabled, or the adapter wasn't present when Ubuntu
  was installed so netplan never picked it up. Setting the static IP
  *during* the installer (step 3) avoids this entirely.
- **Agent shows "Invalid agent name" / "Duplicate agent name"**: both VMs
  had the same hostname (`wazuh-manager` by default from an earlier
  install). Fix with `sudo hostnamectl set-hostname <unique-name>`, then
  remove any stale agent entry on the manager with
  `sudo /var/ossec/bin/manage_agents` (option `R`), and restart the agent.
- **Dashboard says "Wazuh dashboard server is not ready yet"**: usually
  transient after a VM reboot while indexer/manager/dashboard services are
  still starting. Restart in order (indexer, then manager, then
  dashboard) and wait ~30-60s before retrying.
