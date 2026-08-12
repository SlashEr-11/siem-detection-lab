# MITRE ATT&CK Coverage

## Summary

This lab deploys a two-node Wazuh SIEM (1 manager, 1 monitored Ubuntu endpoint)
and validates detection coverage by simulating real attack techniques and
observing both Wazuh's default ruleset and a custom rule written specifically
for this lab.

| MITRE ID | Technique | Detected by Wazuh Default? | Custom Rule? | Validated in lab? |
|----------|-----------|------------------------------|---------------|---------------------|
| T1110.001 | Brute Force: Password Guessing | Partial (rules 5710/5712, per-event, level 5-10) | Yes -- rule 100010 | **Yes** -- fired twice on scripted 8-attempt SSH brute-force |
| T1110.001 / T1021.004 | Successful login after brute-force burst | Partial (rule 5715 covers the login itself) | Yes -- rule 100012 | Written, deployed; correlation window needs further tuning |
| T1078 | Valid Accounts (unauthorized account use) | Yes -- default rules 5901 (new group), 5904 (user info changed) | Not written | Yes -- observed while creating a low-privilege test account |
| -- | Unauthorized privilege escalation (invalid sudo password) | Yes -- default rule (PAM unix_chkpwd failure, level 5) | Not written | Yes -- observed after adding test user to sudoers and entering a wrong password |

## Key finding: default vs. custom detection

Wazuh's out-of-the-box ruleset already flags individual failed SSH logins
(rule 5710, level 5) and even has a generic brute-force rule (5712, level 10).
What it does **not** do by default is correlate a burst of failures from the
**same source IP** into a single, higher-confidence alert with an MITRE
technique ID attached. That gap is what custom rule 100010 closes.

## Detection rule build notes (real debugging log)

Writing rule 100010 required iterating on the base rule ID it correlates
against:

1. First attempt used `<if_matched_sid>5716</if_matched_sid>` based on
   assumption from Wazuh's rule numbering convention -- **did not fire**.
2. Used `sudo /var/ossec/bin/wazuh-logtest` with a hand-crafted sample log
   line to find the actual rule ID Wazuh assigns on this OpenSSH/Ubuntu
   build: rule `5760` (generic "authentication failed").
3. Rewired the rule to `5760` -- **still did not fire** against the real
   attack traffic.
4. Realized the actual attack (SSH to a non-existent username) produces a
   *different* log pattern than a generic "authentication failed" line --
   it matches Wazuh's more specific "Attempt to login using a non-existent
   user" rule, ID `5710`.
5. Rewired to `5710` -- rule fired successfully on the next test.

This mirrors a common real-world SOC/detection-engineering problem: the
log pattern an attacker actually produces is often more specific than the
generic rule you'd guess from documentation alone. `wazuh-logtest` is the
tool that closes that gap -- test against real or representative log lines
before assuming a rule ID.

## How to reproduce the tests

```bash
# T1110.001 -- SSH brute force against a non-existent user (from an
# external host, e.g. the Windows host on a VirtualBox host-only network)
for i in {1..8}; do ssh baduser@<target-ip> -o StrictHostKeyChecking=no; done
# enter any wrong password / press enter at each prompt

# Confirm rule ID for any custom correlation rule before wiring it up
sudo /var/ossec/bin/wazuh-logtest
# paste a representative log line, e.g.:
# Aug 11 16:30:00 target-ubuntu sshd[1234]: Failed password for baduser from <ip> port 54321 ssh2
```

## Next techniques planned (not yet built)

| MITRE ID | Technique | Why it's next |
|----------|-----------|----------------|
| T1053.005 | Scheduled Task | Common persistence mechanism, good Sysmon Event ID 1/11 visibility (Windows target) |
| T1059.001 | PowerShell (encoded/hidden/download execution) | Rules already written (see `rules/` history) for a Windows target; not yet tested against a live Windows agent |
| T1003.001 | LSASS Memory Credential Dumping | High-value SOC skill, needs a Windows target and care around AV/Defender interference |
