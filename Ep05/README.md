1) 20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden. For a homelab. No fluff, senior engineer tone. Include commands.

Title: Ep05 — Wazuh + OpenVAS + Nuclei in a Homelab: Build, Break, Detect, Harden

Intro (0:00–0:45)
- Welcome back to blaqlabs. Ep05: Wazuh for SIEM, OpenVAS for vulnerability assessment, Nuclei for fast, targeted scanning — all in a controlled homelab.
- Quick scope: no external scans on networks you don’t own. Isolate your lab VLAN, DNS, and hosts. Document baselines.

Part 1 — Build (0:46–5:00)
- Objective: set up a repeatable stack: Wazuh manager, Wazuh agent(s), OpenVAS, Nuclei, and a central logging/alerting view.
- Topology:
  - Host A: Wazuh Manager (server)
  - Host B: OpenVAS Scanner
  - Host C: OpenVAS Web UI (optional, in same host as scanner)
  - Host D: Wazuh Agent(s) on lab endpoints
  - Host E: Nuclei runner (or container)
  - Central: Elastic/ Kibana for dashboards (Wazuh uses Elastic)
- Prerequisites (always in a isolated network):
  - Docker/Podman, git, curl, Python v3.x
  - Sudo access
- Commands (example using Docker Compose for a reproducible stack):
  - Create a dedicated lab network
    - docker network create lab-net
  - Set up Wazuh (manager) with Docker Compose (simplified)
    - mkdir -p ~/lab/wazuh && cd ~/lab/wazuh
    - cat > docker-compose.yml << 'YAML'
      version: '3'
      services:
        wazuh-api:
          image: wazuh/wazuh-api:4.x
          depends_on: [wazuh-manager]
        wazuh-manager:
          image: wazuh/wazuh:4.x
          environment:
            - cluster.name=wazuh
          ports:
            - "55000:55000"
          networks:
            - lab-net
        wazuh-agent:
          image: wazuh/wazuh-agent:4.x
          environment:
            - WAZUH_MANAGER=localhost
          networks:
            - lab-net
      networks:
        lab-net:
          external: true
      YAML
    - docker-compose up -d
  - Install OpenVAS (Greenbone Vulnerability Manager) in a container (simplified)
    - docker run -d --name openvas-scanner --net=lab-net --restart=always \
      -e 'OPENVAS_ADMIN_PASS=MyStrongP@ssw0rd' \
      -p 9390:9390 greenbone/openvas
  - Access OpenVAS web UI (from host on lab-net)
    - OpenVAS UI at https://<container-ip>:9392 (default may vary)
  - Set up Nuclei
    - docker run -d --name nuclei -v /home/user/nuclei:/root/.config/nuclei projectdiscovery/nuclei:latest
  - Validate connectivity
    - ping wazuh-manager
    - curl http://wazuh-manager:55000
- Agent deployment (Host D)
  - Install Wazuh agent on a test host (deb)
    - curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/debian/wazuh-agent_4.x.deb
    - sudo dpkg -i wazuh-agent.deb
    - echo 'WAZUH_MANAGER=host.docker.internal' | sudo tee -a /var/ossec/etc/ossec.conf
    - sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent
  - Register agent to manager (using OSSEC key or API)
    - On manager: /var/ossec/bin/manage_agents
    - Add agent and retrieve key, then on agent: /var/ossec/bin/agent-auth -m <manager> -A <agent-name> -p <key>
- Data flow and baselining
  - Wazuh collects logs (syslog, Windows Event Logs, OSSEC rules)
  - OpenVAS scans schedule (e.g., weekly)
  - Nuclei runs on targeted assets to validate exposure after changes
- Configuration best practices
  - Harden Wazuh agent config: disable root access logs, enable file integrity monitoring for key paths
  - OpenVAS: use least-privilege accounts, limit network access to scanner UI
  - Nuclei: maintain allowed templates only; run with -u and -t for targeted tests
- Commands (examples):
  - Wazuh agent status
    - sudo systemctl status wazuh-agent
  - OpenVAS initial setup (web UI)
    - openvas-setup
  - Nuclei basic scan
    - nuclei -u http://target.local -t cves -o results.txt
  - Verify logs in Kibana (Wazuh app)
    - Access Kibana at http://localhost:5601/app/kibana
- Observability
  - Create a Wazuh rule to alert on failed login attempts
  - Set OpenVAS scan results to a dedicated index and create a dashboard
  - Nuclei results feed into a central alert stream with severity tags

Part 2 — Break (5:01–11:00)
- Objective: simulate a controlled breach in the lab to test detection and containment.
- Scenarios:
  - Lateral movement within lab VLAN
  - Web app misconfig exploitation (non-destructive)
  - Credential stuffing with test accounts
- Safe actions (no real data exfiltration)
  - Use intentionally dummy hosts with harmless services (SSH on port 22, HTTP on 8080)
- Break steps with commands (lab-safe)
  - Create a test user with weak password on a host
    - sudo useradd -m tester
    - echo 'tester:Temp123' | sudo chpasswd
  - Simulate failed SSH logins (generate noise)
    - sudo apt-get install -y fail2ban
    - sudo systemctl start fail2ban
  - Run a simulated vulnerability exploit (safe, in-lab)
    - Use a harmless Nuclei template that only checks for an open port and banner
      - nuclei -u 192.168.100.10 -t http-headers.yaml -o nuclei_break_results.txt
  - Misconfiguration attempt with OpenVAS
    - Initiate a low-risk scan profile against the test host
  - Containment actions
    - Wazuh trigger: on high failed login attempts, isolate host by updating firewall
    - sudo ufw deny from any to 192.168.100.10
  - Verify containment
    - Check firewall status
      - sudo ufw status verbose
      - curl http://192.168.100.10:8080/health
- Observability during break
  - Wazuh alerts for authentication failures or suspicious process
  - OpenVAS findings logged and filtered
  - Nuclei results streamed to central log index

Part 3 — Detect (11:01–16:00)
- Objective: demonstrate detection coverage across the stack.
- Detection surfaces
  - Host-level: file integrity, process anomalies, suspicious logins
  - Network-level: unexpected scanning, port sweeps, beacon behavior
  - Vulnerability-level: new OpenVAS findings, misconfig changes
- Detections to demonstrate
  - Wazuh rule: alert when there are more than 5 failed SSH attempts within 2 minutes
  - OpenVAS export: new critical/high findings after a test run
  - Nuclei: template match generates alert for exposed endpoints
- Commands and checks
  - Wazuh alerts
    - tail -f /var/ossec/logs/alerts/alerts.log | head -n 50
    - curl -s http://localhost:55000  # Wazuh API health
  - OpenVAS
    - openvasmd --migrate
    - gvmd --listen 0.0.0.0
  - Nuclei
    - nuclei -list targets.txt
- Dashboards
  - Wazuh app in Kibana for incident timelines
  - OpenVAS findings panel with risk rating
  - Nuclei results feed into a dedicated alert index with tags: "nuclei.discovered", "high-risk"

Part 4 — Harden (16:01–20:00)
- Objective: apply hardening based on detection signals and baselines.
- Hardening steps
  - Wazuh
    - Enable active-response to IP block on multiple failed logins
    - Enforce MFA for admin access where feasible
    - Regularly rotate Wazuh API keys and certificates
  - OpenVAS
    - Schedule rotation of credentials and rescans
    - Restrict access to UI to trusted subnets
  - Nuclei
    - Limit templates to approved set; remove unsafe templates
    - Run with a controlled rate limit to avoid noise
- Validation
  - Re-run a targeted OpenVAS scan against a known fixed host
  - Run Nuclei against a staging host to verify deprecated endpoints are no longer exposed
  - Confirm alerts quiet down after remediation
- Clean-up and repeatability
  - Commit configuration as IaC (Docker Compose + Ansible/Terraform)
  - Create a runbook to reproduce the lab setup
  - Schedule regular, automated runs of Wazuh, OpenVAS, and Nuclei in CI-like cadence

Closing (20:01–21:00)
- Recap: Build a defensive, observable stack with Wazuh, OpenVAS, and Nuclei. Break safely to validate, Detect with dashboards and alerts, Harden through controlled changes and automation.
- Call to action: like, subscribe, and check the show notes for the playbooks and templates used in Ep05.

2) 3-MIN TEASER SCRIPT - Same topic, punchy, hook in first 5 seconds, ends with "Full episode on blaqlabs".

Teaser Script (3 minutes)
- Hook (0–0:05): What if your homelab could automatically answer: “Are we secure?” in real time — Wazuh, OpenVAS, and Nuclei together.
- Quick cut (0:06–0:25): In Ep05, we build a safe lab, break it with controlled tests, detect every telltale sign, and harden the stack so future changes don’t derail security.
- Mid-roll (0:26–1:20): We blueprint a Wazuh-managed SIEM, spin up Greenbone OpenVAS for scalable checks, and run Nuclei against targeted endpoints. You’ll see concrete commands you can reuse: docker-compose up wazuh, openvas-setup, nuclei -u target -t http-headers.yaml, and agent enrollment steps. We keep it strictly lab-safe: isolated network, dummy hosts, and documented baselines.
- Value prop (1:21–2:20): You’ll leave with a repeatable playbook: how to monitor, how to validate detections, and how to harden after findings. No fluff, just signal, noise reduction, and automation hooks you can drop into CI.
- Call to action (2:21–3:00): Full episode on blaqlabs

3) LINKEDIN DRAFT - I will post this manually. Short, human, 1 lesson from the episode.

LinkedIn Draft
Episode Ep05 recap: Wazuh + OpenVAS + Nuclei in a homelab
One lesson learned: when you bring multiple tooling together, your dashboards must tell a coherent story. Wazuh surfaces host and log signals, OpenVAS flags actual vulns, and Nuclei checks for specific exposure patterns. The magic is in the correlation—link a failed login spike to a relevant OpenVAS finding and a Nuclei hit, then automate containment before it bleeds into production. If you’re building resilience, start with a single, well-defined incident timeline and expand from there.