# Cowrie Wazuh honeypot lab
"SSH honeypot (Cowrie) with Wazuh SIEM integration — custom decoders and detection rules for classifying attacker recon, brute-force, and command execution in real time."

Both VMs run on a shared VirtualBox NAT Network, isolated from the host's home network, with port-forwarding rules exposing SSH admin access, the honeypot's SSH port, and the dashboard to the host machine only.


Tech Stack

Honeypot: Cowrie (SSH/Telnet)
SIEM: Wazuh 4.9.2 (Manager, Indexer, Dashboard)
Virtualization: VirtualBox, dual Kali Linux VMs
Networking: VirtualBox NAT Network + port forwarding

Detection Rules Built

A custom decoder (local_decoder.xml) parses Cowrie's raw JSON log lines using Wazuh's JSON_Decoder plugin so these fields become queryable in the dashboard.

Sample Detection

{
  "rule": {
    "level": 10,
    "description": "Cowrie honeypot: possible brute-force attack from single IP",
    "id": "100012",
    "frequency": 5
  },
  "agent": { "name": "homelab", "ip": "10.0.2.3" },
  "data": {
    "eventid": "cowrie.login.failed",
    "username": "root",
    "src_ip": "10.0.2.1"
  }
}

MITRE ATT&CK Mapping

Observed Behavior                                                                                                     Technique
Repeated failed logins                                                                                                T1110 – Brute Force
Successful login with guessed credentials                                                                             T1078 – Valid Accounts
Observed BehaviorTechniqueRepeated failed loginsT1110 – Brute ForceSuccessful login with guessed credentials          T1078 – Valid AccountsCommand execution post-login (whoami, uname -a, etc.)T1059 – Command and Scripting Interpreter
Attempted file download (wget)                                                                                        T1105 – Ingress Tool Transfer

Notable Challenges & Fixes

Worth highlighting — these were the genuinely non-obvious problems solved during the build:


Cowrie's file layout changed in newer versions — cowrie.cfg.dist moved to src/cowrie/data/etc/, and the old bin/cowrie startup script no longer exists; pip install -e . is required to register the CLI entry point.
VirtualBox default NAT mode isolates each VM — switching to a shared NAT Network was required for the honeypot and Wazuh manager to communicate, plus port-forwarding rules to reach both from the host.
Wazuh's JSON decoder always reports as json, not a custom decoder name — rules must match on the actual event field content (eventid) rather than decoded_as.
same_source_ip only works with the reserved srcip field — since Cowrie's JSON uses src_ip, correlation rules had to rely on the base frequency/if_matched_sid mechanism directly.
Disk space exhaustion — Wazuh's Vulnerability Detection module silently downloaded ~18GB of CVE feed data, filling the VM's disk and causing the API service to fail; resolved by clearing the cache and disabling vulnerability-detection for this lab's scope.


Setup Summary

Two VirtualBox VMs (Kali Linux) on a shared NAT Network.
Cowrie installed under a dedicated unprivileged cowrie user, configured with a believable hostname, listening on port 2222.
Wazuh manager, indexer, and dashboard installed via the official all-in-one installer on the second VM.
Wazuh agent installed on the Cowrie VM, configured to monitor cowrie.json.
Custom decoder and rules deployed on the manager to parse and classify Cowrie events.
Verified end-to-end with live SSH sessions against the honeypot.

<img width="1538" height="784" alt="image" src="https://github.com/user-attachments/assets/faf2d95c-dd28-4333-8194-3fafed6ebec0" />
<img width="1540" height="784" alt="image" src="https://github.com/user-attachments/assets/0eefcee9-1371-498f-8d56-860415925f6e" />
<img width="1816" height="735" alt="image" src="https://github.com/user-attachments/assets/5e0af1b8-6cec-4962-895a-fd5c5676eb47" />

Future Improvements

Deploy to a public cloud VPS to capture real internet-sourced attacks instead of simulated local traffic.
Add per-source-IP correlation once on a Wazuh version/config that supports custom same_field correlation reliably.
Expand rule set to cover file upload/download events and privilege escalation attempts.
Add a Kibana/Wazuh dashboard visualization panel dedicated to Cowrie events.

Author

Built by Raul as a hands-on SOC/detection engineering project.
