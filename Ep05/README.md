20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden. For a homelab. No fluff, senior engineer tone. Include commands.

Ep05: Wazuh + OpenVAS + Nuclei

INTRO
- Quick orientation: This is a defensive episode focused on integrating Wazuh, OpenVAS (GVM), and Nuclei in a contained homelab. Goal: visibility, initial risk reduction, and repeatable hardening workflows.
- Prereqs (lab only): isolated network, no public targets. VM hosts: one for Wazuh Manager, one for OpenVAS/GVM, one for a scan target, plus a management workstation. Ensure DNS / DHCP is stable. Timeboxes shown below.

BUILD: where we set the stage (approx. 7–9 minutes)
1) Prepare lab and base images
- Ensure a dedicated VLAN or isolated NIC for scanning.
- Create a small target VM (e.g., Ubuntu 22.04, 2–4 CPUs, 4 GB RAM).
- Prepare a management host with SSH and your preferred tools.

2) Deploy Wazuh (Manager + Agent)
- On Manager (Ubuntu/D Debian):
  - sudo apt-get update && sudo apt-get install -y curl apt-transport-https
  - curl -s https://documentation.wazuh.com/current/installation-guide/debian/wazuh-cluster-key | sudo apt-key add -
  - echo "deb https://packages.wazuh.com/4.x/debian/apt stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
  - sudo apt-get update
  - sudo apt-get install -y wazuh-manager
  - sudo systemctl enable --now wazuh-manager

- On a Wazuh agent (target or dedicated agent host):
  - sudo apt-get update
  - sudo apt-get install -y curl apt-transport-https
  - curl -s https://documentation.wazuh.com/current/installation-guide/debian/wazuh-agent-key | sudo apt-key add -
  - echo "deb https://packages.wazuh.com/4.x/debian/apt stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
  - sudo apt-get update
  - sudo apt-get install -y wazuh-agent
  - sudo nano /var/ossec/etc/ossec.conf # set server address to manager
  - sudo systemctl enable --now wazuh-agent
  - On first agent: register with manager (per your environment) using the agent-auth workflow

3) Deploy OpenVAS / GVM
- Install dependencies and GVM components on a dedicated VM:
  - sudo apt-get update
  - sudo apt-get install -y curl wget gnupg2 ca-certificates lsb-release
  - sudo apt-get install -y greenbone-security-assistant gvm \
      gvmd greenbone-feed-diversity greenbone-feed-base
  - sudo gvm-setup
  - sudo systemctl enable --now gvmd
- Post-setup:
  - sudo runuser -u _gvm -- bash -c 'gvmd --xml "<your initial admin user>"' # or follow prompts
  - Create scanner user, and ensure network reachability to your target

4) Deploy Nuclei
- On management host:
  - sudo apt-get update
  - sudo apt-get install -y nuclei
- Optional: install nuclei-templates
  - nuclei --update-templates
- Quick test:
  - nuclei -t urls-backup/ -l targets.txt -qa

5) Integrate data flows
- Wazuh agent rules: enable decoders for OpenVAS results and Nuclei findings in wazuh.conf or via rules files.
- OpenVAS/Nessus-like scanning results: set up a post-scan log export to a log path Wazuh monitors, or use the Wazuh OpenVAS integration if available.
- Centralize alerts: ensure Wazuh manager is reachable from your OpenVAS/GVM and your management host, and create a few baseline rules for:
  - CVSS high alerts
  - Malformed config/unauthorized access attempts
  - Nuclei high-severity templates hitting critical assets

6) Validation
- Check Wazuh API (on manager):
  - curl -s -k "http://localhost:55000" -u admin:yourpass
- Check OpenVAS/GVM status:
  - run via gvmd and ensure feeds are up-to-date
  - rediscover targets and run a quick scan: grøn
- Check Nuclei results:
  - nuclei -u http://target.lab -t external/ -silent

BREAk: break the lab (approx. 4–5 minutes)
7) Break-style drills in a controlled way (no external targets)
- Misconfiguration drift
  - Create a weak SSH key on an agent
  - sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config && sudo systemctl restart ssh
  - Verify in logs: tail -n 200 /var/log/auth.log on the agent and manager
- Misconfigured Wazuh rule
  - Disable a simple alert by editing ruleset: sudo sed -i '/<rule id="100001"/,/<\/rule>/d' /var/ossec/etc/rules/local_rules.xml
  - Restart wazuh-manager: sudo systemctl restart wazuh-manager
- OpenVAS/GVM drift
  - Block network access for the GVM host: sudo iptables -A OUTPUT -d 10.0.0.0/8 -j REJECT
  - Validate impact on scanning and alerting
- Nuclei drift
  - Introduce an intentionally weak template to test detection: copy a simple template into ~/.config/nuclei/templates/ and run nuclei -t your-template.yaml
- Observability checks
  - Confirm that each drift triggers alerts:
    - Wazuh: tail -f /var/ossec/logs/alerts/alerts.json
    - OpenVAS/GVM: check gvmd logs
    - Nuclei: output results to local file and confirm ingestion

DETECT: detection phase (approx. 5–6 minutes)
8) How we detect via the stack
- Wazuh
  - Real-time alerts for suspicious authentication, config drift, and high-severity CVEs
  - Create a quick query: reconstruction of login failures and sensitive file access
  - Example: use the Wazuh API to fetch alerts with severity >= 7 in last 24h
- OpenVAS / GVM
  - Vulnerability reports show up in the UI and can be sent to a SIEM or log collector
  - Look for newly opened critical vulnerabilities vs baseline
- Nuclei
  - High-severity findings captured in scan outputs; correlate with asset inventory
  - Automate: nuclei -t templates/ -l assets.txt -o results.json
9) Correlation workflow
- Use a simple correlation script (pseudo-structure):
  - if wazuh_alert and openvas_vulnerability and nuclei_findings overlap on same host, raise incident ticket
- Verify that the lab host inventory matches the active scans and alerts

HARDEN: hardening steps (approx. 4–5 minutes)
10) Typical hardening actions
- Wazuh
  - Enforce endpoint hardening baselines: disable unused services, enforce password policies
  - Enable two-factor for management access
  - Regularly rotate agent keys and ensure agent enrollment is limited
- OpenVAS / GVM
  - Limit access to the admin UI to trusted networks
  - Schedule regular feed updates and CVE checks
  - Disable auto-download of exploits (in this context, only scanning content)
- Nuclei
  - Use strictly vetted templates; remove any templates with active exploitation payloads
  - Limit scan scope to protected networks and pre-approved targets
- Network and host controls
  - Implement host-based firewalls, egress restrictions, and segment sensitive assets
  - Put in place a baseline monitoring dashboard for all three tools and set up daily reports
11) Validation of hardening
- Re-run a baseline scan on a test host
- Confirm alerts reflect the changed state (fewer false positives)
- Confirm no unintended service disruption on critical paths
12) Wrap-up
- Recap: Wazuh gives centralized detection, OpenVAS/GVM provides vulnerability scanning, Nuclei accelerates targeted checks
- Emphasize the importance of a repeatable, isolated lab flow before production exposure

OUTRO
- Next steps: automate this workflow with a lightweight pipeline, add asset inventory, and integrate with a ticketing system
- Call to action: like, subscribe, and share your homelab setups in the comments

3-MIN TEASER SCRIPT - Same topic, punchy, hook in first 5 seconds, ends with "Full episode on blaqlabs".

Ep05 Teaser (3 minutes)

- Hook (0–5s): If your homelab isn’t talking to itself in a red-amber-green way, you’re flying blind. Today we stitch Wazuh, OpenVAS, and Nuclei into one defensive workflow.
- Build tease (5–60s): We set up a closed lab with a Wazuh manager, a GVM/OpenVAS box, and fast Nuclei scans. I show exact commands to deploy, tie alerts to a single dashboard, and verify end-to-end visibility.
- Break tease (60–110s): We simulate drift and misconfig, then watch in real time how the stack detects, correlates, and surfaces risk. No public targets; everything lives in a sandbox.
- Detect tease (110–150s): We parse alerts from Wazuh, pull vulnerability reports from GVM, and cross-check with Nuclei hits. See the correlation that actually tells you where to fix first.
- Harden tease (150–170s): Quick hardening playbook: rotate keys, restrict UI, tighten network segments, and lay down a repeatable runbook.
- Call to action (170–180s): Full episode on blaqlabs

LINKEDIN DRAFT - I will post this manually. Short, human, 1 lesson from the episode.

LinkedIn draft (one lesson)
One lesson from Ep05: When you bring Wazuh, OpenVAS/GVM, and Nuclei together, you don’t just find vulnerabilities—you get context. The real value is automated correlation: a Wazuh alert paired with a GVM finding and a Nuclei hit points you to the exact asset, the risk it poses, and the fastest remediation path. Build the pipeline in a lab first, then roll it out with guardrails. Full episode on blaqlabs

Note: If you’d like, I can export this as a text file or a CSV with sections clearly delineated for scripting or notes.