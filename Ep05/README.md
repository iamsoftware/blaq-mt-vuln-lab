### 1) 20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden (Ep05: Wazuh + OpenVAS + Nuclei)

Note: This script assumes a homelab environment with a basic Linux host (Ubuntu/Derby) and network access. The flow is Build (set up telemetry and baseline), Break (simulate a breach/test), Detect (collect/monitor telemetry), Harden (mitigate and lock down). Commands are included as concrete steps you can follow live.

- Opening (0:00–0:45)
  - Intro line: “Welcome back to blaqlabs. Today we’re building a lightweight, observable security stack for a homelab using Wazuh, OpenVAS, and Nuclei. Build the baseline, simulate an abuse path, detect it, and harden. No fluff.”
  - Quick requirements recap: Linux host, Docker (optional), basic networking, sudo access.

- Segment 1: Build – Establish baseline and telemetry (0:45–6:00)
  - Goals: Centralized visibility, agent-based/file-collection, vulnerability scanning, and targeted URL/daemon coverage.
  - Step A: Prepare the host
    - sudo apt-get update && sudo apt-get upgrade -y
    - sudo apt-get install -y curl gnuplot jq unzip
  - Step B: Deploy Wazuh (manager + agent)
    - If using Docker (simplified):
      - docker run -d --name wazuh-manager -p 55000:55000 -p 1514:1514/udp -p 1515:1515/tcp wazuh/wazuh:4.x
      - docker run -d --name wazuh-agent --restart unless-stopped --network host wazuh/wazuh-agent:4.x /bin/bash -c "/usr/bin/wazuh-agent"  # adapt as needed
    - If installing from packages (deb repo):
      - curl -s https://packages.wazuh.com/4.x/apt/Wazuh-4.x.deb -o wazuh-agent.deb
      - sudo dpkg -i wazuh-agent.deb
      - sudo /var/ossec/bin/ossec-control start
  - Step C: Configure Wazuh agent to talk to manager
    - Edit /var/ossec/etc/ossec.conf to point to manager IP: <manager_ip> and ensure API port 1515
    - sudo systemctl restart wazuh-agent
  - Step D: Verify Wazuh connectivity
    - sudo tail -n +1 /var/ossec/logs/ossec.log | tail -n 50
    - curl -sS http://<manager_ip>:55000/agents  # depending on setup
  - Step E: Deploy OpenVAS (Greenbone Vulnerability Manager)
    - On Debian/Ubuntu:
      - sudo apt-get update
      - sudo apt-get install -y openvas
      - sudo gvm-setup   (or sudo openvas-setup)
      - sudo gvm-feed-update
      - sudo systemctl start gvmd
      - sudo runuser -u _gvm -- gvmd --listen 0.0.0.0 --port 9392
    - Access: https://<host>:9392 with admin credentials created during setup
  - Step F: Run an initial OpenVAS scan
    - Use the OpenVAS GUI or the command line to create a target and a scan task
    - Example (CLI, adapt to environment):
      - gvm-cli --xml "<create_target><name>Home LAN</name><hosts>192.168.1.0/24</hosts></create_target>"
      - gvm-cli --xml "<create_task><name>LAN baseline</name><config id='d2a...'</config></create_task>"
    - Start scan and monitor status
  - Step G: Deploy Nuclei for fast discovery (CLI)
    - sudo apt-get install -y nuclei
    - nuclei -v
    - nuclei -u http://192.168.1.1 -t /path/to/templates  # specify templates for common misconfigurations
  - Step H: Integrate findings into Wazuh if possible
    - Use Wazuh decoders/alerts to ingest vulnerability data, or use a custom log collector to feed results from OpenVAS and Nuclei into Wazuh (e.g., via Localfile or Syslog)
  - Segment takeaway: You now have a telemetry plane (Wazuh), a vulnerability plane (OpenVAS), and a fast probe plane (Nuclei). The goal is to reduce silent risk by correlating events.

- Segment 2: Break – Simulate a breach in a controlled way (6:00–12:00)
  - Objective: Generate observable indicators without causing real harm.
  - Step A: Simulated lateral movement (harmless)
    - Create a test user and simulate suspicious activity
      - sudo useradd -m pentest
      - sudo -u pentest touch /tmp/.note  # simple file touch as a trace
    - Echo a small beacon file and monitor access
  - Step B: Unauthenticated port exposure
    - sudo netstat -tulpen | grep -E 'LISTEN|ESTABLISHED'
    - Try to access a non-auth local service with curl to simulate a misconfiguration
      - curl -sS http://localhost:1080  # if a non-existent port is opened
  - Step C: OpenVAS/anomaly scenario
    - Initiate a targeted scan against a known but harmless host in the lab (e.g., a sandbox VM with vulnerable intentionally misconfigured services)
  - Step D: Nuclei simulated test
    - nuclei -u http://192.168.1.100 -t templates/web -t templates/misc
  - Segment takeaway: You should see alerts in Wazuh correlated with OpenVAS/Nuclei findings. Use the collisions to validate alerting rules and dashboards.

- Segment 3: Detect – Observability and correlation (12:00–16:30)
  - Step A: Review Wazuh rules and dashboards
    - Ensure alerts for: new user creation, failed login bursts, port scans, vulnerability findings
    - Validate that OpenVAS/Nuclei results are attached to the incident stream
  - Step B: Create a simple SOC-style alert rule
    - If OpenVAS report contains high severity found, trigger a Wazuh rule with tag vulnerability.high
  - Step C: Demonstrate a sample detection
    - Show Wazuh alert in the console: tail -f /var/ossec/logs/alerts/alerts.log
  - Step D: Verify OpenVAS/Nuclei results flow into Wazuh
    - Check for a correlation in the Wazuh dashboard that maps to an OpenVAS finding
  - Step E: Quick triage playbook
    - If alert: isolate host, check OpenVAS fix status, re-run scan after patch, validate again
  - Segment takeaway: Correlation closes the loop from coverage to detection.

- Segment 4: Harden – Remediation and hardening (16:30–20:00)
  - Step A: Wazuh hardening practices
    - Enforce MFA, restrict SSH to key-based login, disable password authentication
    - Example: Edit /etc/ssh/sshd_config
      - PermitRootLogin no
      - PasswordAuthentication no
      - ChallengeResponseAuthentication no
    - sudo systemctl reload sshd
  - Step B: OpenVAS remediation workflow
    - Prioritize critical/high findings
    - Patch or mitigate, disable vulnerable services if not required
  - Step C: Nuclei hardening rules
    - Remove or restrict unnecessary endpoints; apply rate limits and monitor for exposure
  - Step D: Wazuh hardening/configuration hygiene
    - Rotate keys, minimize log retention exposure, ensure TLS for manager-agent communications
  - Step E: Validation
    - Re-run a light scan to verify that critical findings were mitigated
    - Check that alerts no longer trigger for the hardened services
  - Step F: Documentation and playbooks
    - Document findings, patches applied, and next steps for continuous improvement

- Closing (20:00–20:30)
  - Recap: Wazuh gives visibility, OpenVAS finds the weaknesses, Nuclei provides fast coverage; together you get a measured, verifiable hardening loop for a homelab.

- Callouts and tips
  - Use version-pinning and container isolation for safety
  - Regularly rotate credentials and use a dedicated homelab network
  - Keep a small, repeatable playbook to reproduce builds
  - Avoid leaking credentials in logs; use environment variables or secrets management

- End frame (20:30)
  - “If you found this useful, hit like, subscribe, and share your own homelab hardening steps in the comments. Full episode on blaqlabs.”



### 2) 3-MIN TEASER SCRIPT

- Hook (0:00–0:05)
  - “What if your home network could stop threats before they bite, in minutes, not hours?”
- Middle punch (0:05–2:40)
  - “We’ll build a real-world homelab stack with Wazuh for visibility, OpenVAS for vulnerability checks, and Nuclei for fast, targeted scans. Build the baseline, break a controlled scenario, detect the signals, and harden—live.”
  - “No fluff, just concrete commands and proven steps you can deploy today.”
- Closing hook (2:40–3:00)
  - “Full episode on blaqlabs.”

- End frame (the last line)
  - “Full episode on blaqlabs”

### 3) LINKEDIN DRAFT

One lesson from Ep05: When you combine Wazuh, OpenVAS, and Nuclei in a homelab, correlation is the force multiplier. Build the telemetry, run controlled tests, and ensure alerts trigger on real findings so hardening is evidence-based, not guesswork. Full episode on blaqlabs.