20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden (Ep05: Wazuh + OpenVAS + Nuclei)

Intro (0:00–0:45)
- Greet: “Welcome back to blaqlabs. Episode 05: Wazuh + OpenVAS + Nuclei in a homelab.”
- Context: This episode builds a small, end-to-end detection and response loop using Wazuh as the SIEM, OpenVAS for internal vulnerability discovery, and Nuclei for targeted external HTTP checks.
- Pacing note: We’ll move quickly through build, then inject controlled breaks, show detections, and finish with hardening practicals you can apply tonight.

1) Build (1:00–7:30)
- Objective: Deploy a reproducible lab with Wazuh server + agent, OpenVAS (GVM), and Nuclei. Ensure you have Docker/compose or native installs.
- Prereqs (commands you’ll run):
  - sudo apt-get update && sudo apt-get upgrade -y
  - sudo apt-get install -y curl gnupg containerd.io docker-ce docker-ce-cli docker-compose-plugin
  - Add user to docker group: sudo usermod -aG docker $USER; newgrp docker
- Wazuh (server + agent) deployment (example: Docker Compose):
  - mkdir -p ~/blaqlabs/wazuh; cd ~/blaqlabs/wazuh
  - cat > docker-compose.yml << 'YAML'
  version: '3.8'
  services:
    wazuh:
      image: wazuh/wazuh:4.3.10
      hostname: wazuh
      volumes:
        - wazuh-data:/usr/share/wazuh-manager/data
        - ./wazuh-config:/usr/share/wazuh-api/config
      ports:
        - "55000:55000"  # API
        - "1514:1514/udp" # Syslog
        - "1515:1515/tcp"
    wazuh-api:
      image: wazuh/wazuh-api:4.3.10
      depends_on:
        - wazuh
      ports:
        - "5500:5500"
  volumes:
    wazuh-data:
  YAML
  - docker compose up -d
  - Wait for API ready: curl -k https://localhost:5500 -u foo:bar
- OpenVAS (GVM) deployment (Docker approach, optional host install):
  - docker run -d -p 9392:9392 -p 5432:5432 --name gvmd \
    -e "PASSWORD=changeme" --security-opt seccomp=unconfined \
    immauss/gvm:latest
  - Initialize: curl https://localhost:9392 -k --user admin:changeme
  - After initial setup, enable the Greenbone Vulnerability Manager (scan manager) and start services.
- Nuclei setup (for external checks and templates):
  - sudo apt-get install -y git curl
  - go install -v github.com/projectdiscovery/nuclei/v2/cmd/nuclei@latest
  - export PATH=$PATH:$(go env GOPATH)/bin
  - nuclei -update-templates
  - Quick test: nuclei -t vulnerabilities -u https://example.com -silent
- Network segmentation and agent enrollment:
  - Ensure Wazuh agent installed on a host you’ll monitor (bash example):
    - curl -so /tmp/wazuh-agent.sh https://raw.githubusercontent.com/wazuh/wazuh-agent/master/bin/wazuh-agent
    - bash /tmp/wazuh-agent.sh -a <AGENT_KEY> -k <MANAGER_IP>
  - Validate agent is connected in Wazuh UI (https://localhost:5500) under Agents.
- Data flow and baseline:
  - Configure Wazuh decoders and rules for OpenVAS/Nuclei events (look for OpenVAS logs onboarding, Nuclei findings).
  - Example: enable rule sets in /usr/share/wazuh/rules/openvas_rules.xml and nuclei_rules.xml
- Quick sanity steps:
  - OpenVAS scan: openvas-modules start; run a basic internal host scan
  - Nuclei scan: nuclei -t nuclei-templates/ -u 192.168.1.0/24 -l targets.txt -o results.html
  - Verify alerts: tail -f /var/ossec/logs/ossec.log or check Wazuh dashboard for new alerts.

2) Break (7:30–12:30)
- Purpose: Simulated controlled incidents to validate detection pipeline.
- Simulated event 1: OpenVAS internal scan activity
  - Run a quick internal host scan range: openvas-scan -h 192.168.1.0/24 -T 1
  - Observe: Wazuh should generate a rule-trigger when a scan ports multiple-time or abnormal scan detected on a host.
  - Wazuh query example (watch for GID/Rule): /var/ossec/bin/ossec-control status; tail -f /var/ossec/logs/ossec.log | grep -i "vuln"
- Simulated event 2: External HTTP checks with Nuclei
  - nuclei -t camo.yaml -u http://your-honeypot.local -o nuclei-out.txt
  - Expectation: Nuclei findings exported; Wazuh receives a log payload (via syslog or agent) and correlates with a vulnerability policy.
  - If not auto-forwarded, ship Nuclei results to Wazuh:
    - cat nuclei-out.txt | logger -t nuclei -p info
- Simulated event 3: Agent compromise or misconfiguration
  - Create a non-root user, run a privilege escalation attempt in a lab host
  - Observe: Wazuh alerts on unexpected privilege escalations, file integrity change, or unusual process ancestry.
- Detect and respond loop (live demonstration steps):
  - In Wazuh UI: Go to Alerts -> Folders -> Real-time Alerts
  - Filter: rule.id: "100123" or rule.name contains "openvas" or "nuclei"
  - Playbook: isolate host, revoke credentials, collect for forensics
- Common pitfalls checklist:
  - Time synchronization across all components (NTP)
  - Correct agent-manager communication (certs, keys)
  - Template updates (Nuclei) and OpenVAS feed synchronization
  - False positives vs. true positives: tune decoders before hardening
- Quick remediation examples (on-the-fly):
  - Block suspicious source IPs at firewall: sudo ufw deny from 198.51.100.0/24
  - Tighten OpenVAS login: change admin password; ensure no default credentials
  - Harden Wazuh: enable 2FA for admin, rotate API keys, restrict SSH to trusted hosts

3) Detect (12:30–17:30)
- Build-to-dreach: Focus on what you watch for in real time
- Wazuh detection rules:
  - Real-time: monitor for new OpenVAS findings
  - Nuclei findings: check for exposed endpoints, status codes, and misconfigurations
  - Lateral movement indicators: abnormal SSH logins, new processes, privilege escalation
- Log flow overview:
  - OpenVAS/Nuclei findings push to Wazuh via structured logs
  - Wazuh correlates with agent data (syslog, auth logs, file integrity)
  - Alerts surface in Wazuh UI, sent to you via email or Slack
- Guardrail validation:
  - Run a synthetic alert test by triggering a benign OpenVAS vulnerability or a known-Nuclei template
  - Confirm alert appears in Wazuh, and the response playbook runs (scripts or manual steps)
- Example queries for the audience:
  - Wazuh API demo: curl -X GET "https://localhost:5500/agents?limit=50" -u foo:bar -k
  - Check last 100 alerts: tail -n 100 /var/ossec/logs/ossec.log
  - Nuclei output check: cat nuclei-out.txt | grep -i "http" | head
- Key takeaways for detection:
  - Correlate external scanning (Nuclei) with internal exposure (OpenVAS)
  - Tie findings to asset inventory and risk scoring
  - Enforce least privilege in all scanners and automation
- Transition to hardening:
  - We will harden the lab by tightening access, rotating credentials, and enforcing monitoring rules

4) Harden (17:30–20:00)
- Immediate hardening steps:
  - Wazuh:
    - Enforce TLS for API: edit /var/ossec/api/configuration/authentication/authd.json
    - Rotate admin credentials, enable 2FA, restrict IP range for API access
    - Disable unused rule groups; tune thresholds to reduce noise
  - OpenVAS/GVM:
    - Keep feeds updated: gvm-feed-update
    - Limit admin interface exposure to trusted networks
    - Schedule regular vulnerability scans with retention policies
  - Nuclei:
    - Patch templates regularly: nuclei -update-templates
    - Run targeted scans on trusted hosts only
- Network hardening:
  - Firewall: sudo ufw default deny; sudo ufw allow from 192.168.1.0/24 to any port 5500
  - SSH hardening: PermitRootLogin no; PasswordAuthentication no; Use key-based auth
  - IDS/IPS: enable Wazuh rules to block or alert on repeat patterns
- Hardening commands recap:
  - Wazuh TLS/API: cert generation, nginx config, and cert trust setup
  - OpenVAS/GVM: configure credentialed checks for deeper findings
  - Nuclei: enforce authentication for scans if possible, avoid broad internet exposure
- Final lab sanity:
  - Re-run a controlled OpenVAS scan and Nuclei check
  - Confirm alerts are aligned with the hardening changes
  - Verify access restrictions and that only approved hosts can communicate with Wazuh API

Closing (20:00–20:30)
- Summary: Integrated Wazuh with OpenVAS and Nuclei to detect and harden a homelab.
- CTA: Like, subscribe, and comment with what you want to see next—more hands-on lab coverage, more automation scripts, more detectionPlaybooks.

3-MIN TEASER SCRIPT

- Hook in first 5 seconds:
  - “Want a homelab that actually detects and blocks threats in real time? Wazuh, OpenVAS, and Nuclei—together, we break, detect, and harden.”
- Fast-paced beat: quick cuts through the Build, Break, Detect, Harden steps with on-screen commands:
  - Build: docker compose up -d, openvas-setup, nuclei -update-templates
  - Break: simulate a scan, then watch Wazuh alert
  - Detect: filter alerts in Wazuh UI, curl the API
  - Harden: enforce TLS, rotate credentials, restrict access
- Close with the hook:
  - “Full episode on blaqlabs”.

- End line:
  - “Full episode on blaqlabs”.

LINKEDIN DRAFT

One lesson from Ep05: When you combine Wazuh with proactive scanners like OpenVAS and Nuclei, you don’t just detect vulnerabilities—you create a fast, repeatable response loop. Build a reproducible lab, simulate incidents, verify detections in real time, and then harden. It makes incident response feel like a muscle you can train, not a mystery you guess at.