20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden
Episode: Ep05 - Wazuh + OpenVAS + Nuclei (Homelab)

Intro (0:00–0:45)
- Hook: “If your homelab isn’t defending itself, you’re building a model with a leaking roof.”
- What you’ll cover: a practical, defensive loop using Wazuh, OpenVAS (GVM), and Nuclei to build visibility, break a controlled target, detect the signals, and harden the environment.
- Prereqs in one line: a small lab with a few VMs or containers, root/sudo access, and a network isolated from your production.

Segment 1: Build (0:45–7:00) – Set the foundation for monitoring, scanning, and discovery
- Goals:
  - Deploy a Wazuh stack (manager + indexer + agent)
  - Stand up OpenVAS/GVM for vulnerability scanning
  - Prepare Nuclei for targeted checks
- Assumptions:
  - Lab target: 192.168.1.100 (Windows/Linux host)
  - Wazuh server: 192.168.1.10
  - Nuclei/OpenVAS hosting on 192.168.1.20
- Steps (with commands):
  1) Update and install prerequisites on the Wazuh server
     - sudo apt-get update && sudo apt-get upgrade -y
     - sudo apt-get install -y curl apt-transport-https lsb-release gnupg
  2) Install Wazuh Manager (Debian/Ubuntu)
     - curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh-archive-keyring.gpg
     - echo "deb [signed-by=/usr/share/keyrings/wazuh-archive-keyring.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
     - sudo apt-get update
     - sudo apt-get install -y wazuh-manager
  3) Install Wazuh Agent on the target (192.168.1.100)
     - sudo apt-get update
     - sudo apt-get install -y wazuh-agent
     - Edit /var/ossec/etc/ossec.conf to point to 192.168.1.10
     - sudo systemctl enable wazuh-agent
     - sudo systemctl start wazuh-agent
  4) Install OpenVAS/GVM (on 192.168.1.20)
     - sudo apt-get update
     - sudo apt-get install -y openvas
     - sudo update-rc.d openvas defaults
     - sudo gvm-setup
     - sudo gvm-feed-update
     - sudo systemctl start gvmd
     - sudo runuser -u _gvm -- gvmd --user=admin --new-password='YourStrongP@ssw0rd'
  5) Install Nuclei (on a control host or same box)
     - sudo apt-get install -y nuclei
  6) Prepare a basic inventory and config
     - Create a simple targets.txt with 192.168.1.100
     - Wazuh agent and manager correlation: ensure active responses and alerting are enabled
  7) Basic onboarding checks
     - curl -sS http://192.168.1.10:55000/agents  # verify Wazuh API access
     - sudo -u gvm greenbone-nvt-sync
  8) Validate network reachability
     - ping -c 3 192.168.1.100
  9) Short caveat on scans
     - Scans in a lab should be non-destructive; use safe/authenticated modes and schedule during maintenance windows
- Notes for the audience:
  - Emphasize isolated lab networks, legal/ethical boundaries, and asset inventory before scanning.

Segment 2: Break (7:00–12:30) – Controlled tests to generate signals
- Goals:
  - Simulate common attack surfaces in a safe way
  - Verify that OpenVAS/Nuclei findings generate observable logs
- Scenarios and commands:
  1) OpenVAS/GVM quick scan against the lab host
     - On 192.168.1.20 (or a dedicated scanner): gvm-cli or web UI
     - Example: run a basic internal scan:
       - sudo runuser -u _gvm -- gvm-cli --xml "<create_scan_request>...</create_scan_request>" --targets 192.168.1.100
     - Or through the UI: New → Target: 192.168.1.100 → Scan Config: "Full and fast" → Launch
  2) Nuclei focused checks
     - nuclei -t templates/ -target 192.168.1.100
     - For example: nuclei -t vulnerabilities/ -target 192.168.1.100 -o results_nuclei.json
     - Use header and auth templates if needed:
       - nuclei -t http-headers.yaml -u http://192.168.1.100
  3) Trigger safe exploits or test vectors in a lab
     - Avoid real exploitation; simulate using harmless payloads or misconfig checks
     - Example safe check: test for default credentials
       - curl -sS http://192.168.1.100:80  # basic service banner
       - Use a controlled script to verify if default creds exist (in a controlled test VM)
  4) Observe Wazuh alerts
     - Check manager API or dashboard for alerts triggered by scans
     - tail -f /var/ossec/logs/alerts/alerts.log
     - curl -sS -H 'Authorization: Bearer YOUR_API_KEY' http://192.168.1.10:55000/alerts
- What to look for:
  - OpenVAS findings should correlate to Wazuh events about vulnerable services, outdated packages, or misconfigurations
  - Nuclei results should appear as log events or API alerts if integrated
- Guardrails:
  - Do not scan production networks
  - Throttle scan intensity to avoid disrupting services
  - Document every test to feed the hardening phase

Segment 3: Detect (12:30–17:30) – Turn signals into alerts and telemetry
- Goals:
  - Verify that Wazuh detects indicators from scans
  - Validate alerting rules, timelines, and enrichment
- Highlights:
  1) Wazuh rules overview
     - Default rules map to log sources: OpenVAS, Nuclei, system events
     - Example: a vulnerability detected by OpenVAS may create a Security alert with a CEF-like payload
  2) How to inspect detections
     - On Wazuh UI: SIEM → Alerts → use filters: rule.id, src.ip, src.hostname
     - sudo tail -f /var/ossec/logs/alerts/alerts.log
     - curl -sS -H 'Authorization: Bearer YOUR_API_KEY' http://192.168.1.10:55000/alerts?from=now-1h
  3) Correlation and enrichment
     - Enrich with asset inventory: hostname, OS, patch level
     - Correlate scan results with active devices in the network map
  4) Example signal flow
     - OpenVAS finds CVE-2023-XXXX on 192.168.1.100
     - Wazuh alerts on vulnerability event tied to that host
     - Nuclei reports a specific endpoint misconfiguration; correlates to the same host
  5) Validation steps
     - Confirm that alerts have meaningful fields: host, service, risk, CVE, and remediation suggestions
     - Test alert routing to a secondary SIEM or a Slack/Teams webhook
- Practical tips:
  - Tag rules with environment and asset groups to reduce noise
  - Regularly rotate credentials and disable unused services to reduce risk

Segment 4: Harden (17:30–25:00) – Mitigations, fixes, and hardening playbook
- Goals:
  - Remediate detected issues
  - Harden the stack to reduce future exposure
- Actions:
  1) Patch and update
     - sudo apt-get update && sudo apt-get upgrade -y
     - For OpenVAS/GVM: sudo greenbone-feed-update && sudo greenbone-nvt-sync
     - Re-scan after patching
  2) Configuration hardening
     - Wazuh agent: enable TLS, limit agent reachability, ensure proper log collection rules
     - OpenVAS/GVM: adjust scan policy to non-destructive, limit scope, disable unnecessary plugins
     - Nuclei: maintain a curated template set, disable anonymous tests if not required
  3) Network controls
     - Firewall rules: ufw allow from 192.168.1.0/24 to 192.168.1.0/24 on required ports
     - Segmentation: isolate scanner subnet from critical services yet allow read access for monitoring
  4) Access controls
     - Use strong, unique passwords; enable MFA where possible
     - Limit admin accounts and monitor for brute-force attempts
  5) Verification
     - Re-run scans to confirm mitigations
     - Check Wazuh alerts drop in frequency or change to remediation-focused events
  6) Documentation and runbooks
     - Update runbooks with what changed, why, and who approved the changes
- Final thoughts:
  - A defensible homelab is built on repeatable processes: visibility, validation, and rapid patching.

Close (25:00–26:00)
- Recap: Build a monitored base, Break safely to test detection, Detect with solid telemetry, Harden with disciplined remediation
- CTA: Like, subscribe, and leave questions about your own Wazuh/OpenVAS/Nuclei workflows
- Mention upcoming content and tease future integrations or advanced playbooks

End card (26:00)
- Display links to related resources, and remind viewers to check the description for commands and configs
- Thank the audience

3-MIN TEASER SCRIPT
Ep05 Teaser – Wazuh + OpenVAS + Nuclei (Homelab)

0–5 seconds: Hook
- “If your homelab isn’t defending itself, you’re flying blind in a storm. Today we stitch together Wazuh, OpenVAS, and Nuclei to build, break, detect, and harden in a safe lab.”

5–25 seconds: Quick pitch
- “We’ll lay down a mirrored defense: deploy a Wazuh stack, run targeted scans with OpenVAS and Nuclei, see how alerts appear, and close the gaps with practical hardening steps.”

25–60 seconds: Value props and outline
- “Build the monitoring backbone, fire off safe, controlled tests to generate signals, verify detection, and implement fixes. It’s a hands-on, repeatable defensive loop for your home environment.”

End with call-to-action
- “Full episode on blaqlabs”

LinkedIn Draft (1 lesson from Ep05)
Note: This is a short post you can customize before posting.
- One lesson from Ep05: “In a homelab, visibility is weaponized security. By integrating Wazuh with OpenVAS and Nuclei, you turn scans into structured signals you can detect, correlate, and fast-track hardening. Start small, automate the workflow, and document every change so you can repeat the defense.”

If you’d like, I can format these into a single editable document or tailor the commands for your exact environment (OS versions, container vs. VM, network topology).