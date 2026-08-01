# Brute Force Detection (T1110) — Splunk SIEM

## Overview
End-to-end detection engineering lab: simulated a brute force attack 
against a Windows Server target, built a full log pipeline into Splunk, 
detected the attack, and wrote a Sigma detection rule mapped to MITRE 
ATT&CK T1110.

This project supersedes an earlier Wazuh-based version (see 
`/brute-force-detection`) — pivoted to Splunk after infrastructure 
constraints made Wazuh unreliable in this environment. Splunk is also 
more widely requested in SOC Analyst job postings.

## Lab Architecture
- **Kali Linux** — attacker machine (Hydra, Nmap)
- **Windows Server 2012 R2** — target (RDP/SMB enabled)
- **Ubuntu + Splunk Enterprise** — SIEM
- **Splunk Universal Forwarder** — installed on Windows Server, forwards 
  Security Event Logs to Splunk over port 9997

## Attack Simulation
1. Confirmed SMB (port 445) reachable on target via Nmap
2. Ran Hydra brute force against Windows Server SMB using a 10-password 
   list against the Administrator account
3. All 10 attempts failed (expected — the goal is to generate detectable 
   failed-authentication traffic, not to crack the password)

## Detection
Windows logs failed authentication as **Event ID 4625**. Confirmed all 
10 failed attempts appeared in Splunk, sourced from the Kali attacker IP.

### Sigma Rule
See [`sigma-rule/brute_force_t1110.yml`](./sigma-rule/brute_force_t1110.yml)

Modeled after a reviewed SigmaHQ community rule — added a `timeframe` and 
count threshold rather than alerting on every single failure, plus 
documented `falsepositives` (forgotten passwords, vulnerability scanners, 
misconfigured applications).

### Splunk Detection Query (SPL)
```spl
index=main source="WinEventLog:Security" EventCode=4625 
| bucket _time span=10m 
| stats count by Source_Network_Address, _time 
| where count > 5
```
This flags any source IP with more than 5 failed logins within a 
10-minute window — translating the Sigma rule's `timeframe` and 
`count()` logic directly into Splunk's search language.

## Result
Splunk correctly identified the attacker (Kali, `10.0.0.10`) with 10 
failed login attempts within the detection window — exceeding the 
threshold and confirming the detection logic works against real 
generated attack traffic.

## MITRE ATT&CK Mapping
- **Tactic:** Credential Access
- **Technique:** T1110 — Brute Force

## Key Learning
A Sigma rule detects the *outcome* of an attack (failed authentication), 
not the *method* used to generate it. This same rule would catch a 
brute force attempt over RDP, SMB, or SSH — the detection logic doesn't 
care which protocol was used, only that Windows logged a failed logon.

## Incident Response Notes
If this were a real alert:
1. Isolate the target machine from the network
2. Reset the targeted account's password
3. Check for any successful login (Event ID 4624) from the same source 
   — indicates possible compromise
4. Check for lateral movement to other internal hosts
5. Escalate to Tier 2 with source IP, timeline, and account details

## Screenshots
See `/screenshots` folder for attack execution, Splunk detection 
results, and the Sigma rule file.
