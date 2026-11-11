1) 20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden. For a homelab. No fluff, senior engineer tone. Include commands.

Episode: Ep05 — Topic: Wazuh + OpenVAS + Nuclei

Introduction (0:00–0:45)
- Greet: “Hello, I’m [Name], senior infra vuln engineer. Welcome to blaqlabs Ep05. Today we’re integrating Wazuh, OpenVAS, and Nuclei in a homelab to build, break, detect, and harden a small attack surface.”
- State objective: “We’ll deploy a monitoring stack (Wazuh) alongside a vulnerability scanner (OpenVAS) and an automation scanner (Nuclei) to demonstrate end-to-end defense-in-depth in a home lab.”
- Quick prerequisites: “Ubuntu 22.04+ host or a set of VMs, Docker or bare-metal, root or sudo access, basic networking, and a defensive mindset.”

Section 1: Build — baseline architecture and prerequisites (2:00)
- Describe the target setup:
  - Wazuh manager + Wazuh agents (on hosts in lab)
  - OpenVAS (Greenbone) scanner, with a dedicated scanning VM
  - Nuclei for rapid, rule-based testing
  - ELK/Elastic stack or Wazuh Kibana dashboard for visualization
- Show high-level topology: 
  - Vulnerable assets: host1 (lab workstation), host2 (server), host3 (docker host)
  - Sensor: Wazuh agent on each asset
  - Scanner: OpenVAS VM, Nuclei runner on a control host
- Commands: system prep
  - Update and install dependencies
    - sudo apt-get update && sudo apt-get upgrade -y
    - sudo apt-get install -y curl wget apt-transport-https lsb-release gnupg software-properties-common
- Install Docker and docker-compose (optional but recommended):
  - sudo apt-get install -y docker.io
  - sudo systemctl enable --now docker
  - sudo usermod -aG docker $USER
  - # logout/login or newsession
  - docker --version; docker-compose --version
- Prepare Wazuh repository and initial install (using Docker compose for speed):
  - mkdir -p ~/wazuh-dashboard && cd ~/wazuh-dashboard
  - curl -s https://packages.wazuh.com/4.x/wazuh-docker-compose.yml -o docker-compose.yml
  - sed -i 's/host.docker.internal/host.docker.internal/' docker-compose.yml  # if needed
  - docker-compose up -d
  - Wait for services: curl -s http://localhost:55000/  # Wazuh API sanity
- OpenVAS (Greenbone) quick start:
  - sudo apt-get install -y openvas
  - sudo gvm-setup
  - sudo gvm-feed-update
  - sudo gvm-manage-certs -a
  - sudo systemctl start gvmd
  - gvmd --version
  - OpenVAS web interface at https://<host>:9392 and default admin creds from setup
- Nuclei install:
  - sudo apt-get install -y golang-go
  - GO111MODULE=on go get -u -v github.com/projectdiscovery/nuclei/v2/cmd/nuclei
  - nuclei -version
  - Prepare templates: nuclei -update-templates

Section 2: Break — simulate attacker actions and break the lab minimal risk (5:00)
- Outline realistic attack scenarios:
  - Scanning a misconfigured host with OpenVAS to generate vulnerabilities
  - Running Nuclei templates to probe for common misconfigurations
  - Generating benign but realistic traffic to trigger Wazuh alerts
- Demonstration steps (safe, non-destructive):
  - Nuclei quick scan against internal targets:
    - nuclei -u http://192.168.1.50 -t nuclei-templates/http-headers.yaml
    - nuclei -l targets.txt -t cves/ -o results/nuclei_results.txt
  - OpenVAS quick scan:
    - OpenVAS UI: scan new target 192.168.1.40 with a standard “Full and Quick” policy
    - After scan: review criticals and highs
  - Trigger a Wazuh alert by creating a benign event:
    - echo "ssh_test: test event" | logger -t wazuh-test
    - tail -f /var/ossec/logs/ossec.log | grep -i test
  - Demonstrate a weakness you would expect to be caught, e.g., default credentials on a service (but do not expose actual credentials)
- Observations to capture:
  - OpenVAS finds CVEs and misconfigurations
  - Nuclei detects misconfigurations in web assets
  - Wazuh correlates events and generates an alert

Section 3: Detect — how Wazuh ingests, correlates, and surfaces alerts (5:00)
- Wazuh deployment details:
  - Agents on hosts
  - Manager with rules and decoders
  - Indexers (Elasticsearch) and Kibana for dashboards
- Key commands and checks:
  - Check agent status from manager:
    - sudo /var/ossec/bin/OSSEC-control status
  - Check agent connection on a host:
    - sudo tail -f /var/ossec/logs/ossec.log | grep 'agent'
  - Wazuh API quick query:
    - curl -k -u foo:bar "https://localhost:55000/security/users" | jq .
  - View alerts in Kibana:
    - Open http://localhost:5601/app/kibana#/app/home or the Wazuh app
- Demonstrate a sample alert flow:
  - OpenVAS creates vulnerability N, Nuclei finds issue Y, Wazuh correlates as “Vuln scanners triggered on asset”
  - Show how the alert would appear in the dashboard, with severity, asset, and remediation steps
- Logging and telemetry considerations:
  - Ensure agent-to-manager TLS with certificates
  - Enable decoded events for OpenVAS with proper decoders
  - Normalize and tag events for easier search in Kibana

Section 4: Harden — practical hardening steps post-detection (6:30)
- Immediate actions after a vulnerability discovery:
  - Isolate affected host (if critical): firewall block, update patch, rotate credentials
  - Apply patches for OpenVAS scanner host and any vulnerable assets
  - Ensure vulnerable services are disabled or restricted (principle of least privilege)
- Hardening steps to implement in the lab:
  - Wazuh hardening:
    - Enforce TLS for Wazuh manager-agent communications
    - Enable file integrity monitoring on critical config paths
    - Disable unused decoders and rules to reduce noise
  - Nuclei hardening:
    - Use allowlists for templates; prune obsolete templates
    - Run authenticated checks where possible
  - OpenVAS hardening:
    - Use authenticated scans when possible
    - Limit scanner permissions and ensure the scanner VM is isolated from production-like networks
- Remediation workflow example:
  - Create a runbook snippet:
    - Step 1: Verify alert in Wazuh
    - Step 2: Validate vulnerability details from OpenVAS
    - Step 3: Patch or mitigate (disable, rotate credentials, restrict network)
  - Document actions in a shared repo for repeatability
- Validation after hardening:
  - Re-scan the same assets with OpenVAS and Nuclei
  - Confirm alerts drop in Wazuh
  - Ensure no new criticals appear

Section 5: Wrap-up and takeaways (2:30)
- Recap the production-ready workflow:
  - Build baseline with Wazuh, OpenVAS, Nuclei
  - Break safely to verify detection
  - Detect and investigate with dashboards and API
  - Harden in a controlled, auditable loop
- Metrics you’d track:
  - Time-to-detect (TTD)
  - False positives rate
  - Mean time to remediation (MTTR)
- CTA: “If you want the full, hands-on walkthrough with scripts and exact docker-compose files, stick around for the next episodes. Full episode on blaqlabs.”

End screen (final 15 seconds)
- Quick sign-off: “Thanks for watching. Subscribe for more advanced homelab blue-team practices. Ep05 ends here.”

2) 3-MIN TEASER SCRIPT - Same topic, punchy, hook in first 5 seconds, ends with "Full episode on blaqlabs".

Teaser: Ep05 — Wazuh + OpenVAS + Nuclei in a Homelab
- [0:00–0:05] Hook: "What if your home lab could auto-detect and fix its own vulnerabilities in real-time?"
- [0:05–0:20] Fast build: "In Ep05, I spin up Wazuh, OpenVAS, and Nuclei in a single homelab—build, break, detect, and harden—live."
- [0:20–0:50] Quick hit: 
  - "I’ll show a safe attacker simulate: misconfig checks with Nuclei, a vulnerability scan with OpenVAS, and alerts flowing into Wazuh."
  - "Then I’ll harden the stack with TLS, pruning rules, and authenticated checks."
- [0:50–1:10] Value: "You’ll walk away with a repeatable playbook to shorten detection-to-remediation time in your own lab."
- [1:10–1:45] Tease steps:
  - "Baseline build in minutes, a controlled breach for verification, and a hardened end-state you can reuse."
- [1:45–2:30] Closer: "If you’re building defensive muscle in your homelab, you won’t want to miss this."
- [2:30–3:00] CTA: "Full episode on blaqlabs."

3) LINKEDIN DRAFT - I will post this manually. Short, human, 1 lesson from the episode.

LinkedIn draft:
- Lesson from Ep05 (Wazuh + OpenVAS + Nuclei): In a small homelab, building a defense-in-depth loop that combines a host-based monitor (Wazuh), a vulnerability scanner (OpenVAS), and a fast probing tool (Nuclei) lets you detect and harden faster. The key is to automate the triad: baseline monitoring, continuous scanning, and rapid remediation playbooks. When alerts flow into a single dashboard, you shorten detection-to-remediation and lock down risk before it compounds. Full episode on blaqlabs.
