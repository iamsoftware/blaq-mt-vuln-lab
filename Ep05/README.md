## 1) 20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden (Homelab; Wazuh + OpenVAS + Nuclei)

Intro (0:00–0:45)
- Opening hook: “Today we wire up a real homelab stack: Wazuh for SIEM, OpenVAS for vulnerability discovery, and Nuclei for fast, templated checks. Build, break, detect, and harden—without bleeding edge fluff.”
- Quick prerequisites: Ubuntu 22.04+, sudo access, enough CPU/RAM for a small SIEM stack (2–4 CPU, 8–16 GB RAM recommended).
- High-level plan: build a compact lab, intentionally introduce a few test vulns, detect them with Wazuh/OpenVAS/Nuclei, and harden.

Segment A: Build (0:46–7:30)
- Goals: deploy a reproducible lab with Wazuh (manager + agent), OpenVAS (Greenbone Vulnerability Manager), and Nuclei, all accessible from a single host network.

1) Environment prep
- Commands:
  - sudo apt update && sudo apt -y upgrade
  - sudo apt install -y docker.io docker-compose curl gnupg software-properties-common
  - sudo systemctl enable --now docker

2) Wazuh stack (manager + elasticsearch + Kibana) via Docker Compose
- Create a docker-compose.yml (example minimal setup):
  - Version 3
  - Services: wazuh-manager, wazuh-api, elasticsearch, kibana, filebeat (optional)
- Example snippets (adjust to your network/file paths):
  - mkdir -p ~/wazuh && cat > ~/wazuh/docker-compose.yml << 'YAML'
  - version: '3.8'
  - services:
    wazuh-manager:
      image: wazuh/wazuh:4.3.10
      environment:
        - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      ports:
        - "1514:1514/udp"
        - "55000:55000"
      depends_on:
        - elasticsearch
        - kibana
    elasticsearch:
      image: docker.elastic.co/elasticsearch/elasticsearch:8.6.2
      environment:
        - discovery.type=single-node
        - ES_JAVA_OPTS=-Xms512m -Xmx512m
      ulimits:
        memlock:
          soft: -1
          hard: -1
      deploy:
        resources:
          limits:
            memory: 1g
    kibana:
      image: docker.elastic.co/kibana/kibana:8.6.2
      ports:
        - "5601:5601"
      depends_on:
        - elasticsearch
  YAML
- Bring up:
  - cd ~/wazuh && docker-compose up -d
- Verify:
  - docker ps
  - curl -s http://localhost:5601/api/status | jq
1) Wazuh agent on host
- Commands (agent registration depends on manager):
  - sudo apt install -y curl
  - curl -so wazuh-agent-setup.sh https://packages.wazuh.com/3.x/wazuh-agent-setup.sh
  - sudo bash wazuh-agent-setup.sh -i http://localhost:55000
  - sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent

2) OpenVAS (Greenbone) deployment
- Install dependencies and GVM
  - sudo apt install -y software-properties-common
  - sudo add-apt-repository -y ppa:mrazek/gvm
  - sudo apt update
  - sudo apt install -y gvmd openvas-scanner greenbone-security-assistant
- Initialize:
  - sudo runuser -u _gvm -- gvmd --version
  - sudo greenbone-nvt-sync
  - sudo greenbone-feed-sync
  - sudo runuser -u _gvm -- gvmd --listen=0.0.0.0 --port=9392 --migrate || true
- Web UI:
  - Access http://localhost:9392 with admin user set during setup.

3) Nuclei setup
- Install:
  - sudo apt install -y unzip curl
  - curl -sL https://github.com/projectdiscovery/nuclei/releases/latest/download/nuclei-$(uname -s | tr '[:upper:]' '[:lower:]')-$(uname -m).zip -o nuclei.zip
  - unzip nuclei.zip -d ~/nuclei && sudo mv ~/nuclei/nuclei /usr/local/bin/nuclei
  - nuclei -update-templates
- Quick usage:
  - nuclei -u http://localhost -t templates/http/
  - nuclei -u http://192.168.1.100 -t templates/vuln/ -json -o ~/nuclei_results.json

4) Networking and basic access validation
- Commands:
  - curl -k -I http://localhost:5601
  - sudo docker logs wazuh-manager 2>&1 | tail -n 50

5) Baseline inventory
- Collect host list:
  - cat /etc/hosts
  - hostnamectl

Segment B: Break (7:31–12:30)
- Intent: simulate vulnerabilities in a controlled way, so detection has something to catch.

1) Create a deliberately vulnerable local host (safe lab host or container)
- Example: enable insecure Nginx/Apache with default creds (do not use real admin credentials).
- Commands (simulate vulnerability):
  - docker run -d --name vulnhost -p 8080:80 nginx:latest
  - docker exec vulnhost bash -lc "apt-get update && apt-get install -y apache2-utils && echo 'Test' > /var/www/html/index.html"

2) OpenVAS/Nuclei checks against vulnhost
- OpenVAS scan
  - Access the OpenVAS UI and create a new task targeting 127.0.0.1:8080
  - Start scan and wait for results; export as XML/CSV
- Nuclei checks
  - nuclei -u http://127.0.0.1:8080 -t templates/http/ -o ~/nuclei_http_scan.txt
  - nuclei -u http://127.0.0.1:8080 -t templates/vuln/ -o ~/nuclei_vuln_scan.json

3) Wazuh rule alerting
- Ensure Wazuh rules cover vulnerability findings:
  - Wazuh decoders for OpenVAS and Nuclei results
  - If using Filebeat, ship OpenVAS/Nuclei output to Wazuh
- Example: tail -f /var/ossec/logs/alerts/alerts.log
- Map critical findings to a dedicated alert rule in Wazuh manager:
  - If nvt: "CVE-YYYY-XXXX" or risk: "High" then trigger alert level 10

Segment C: Detect (12:31–17:30)
- Visualization and correlation
- Kibana dashboards
  - Open http://localhost:5601/app/kibana and verify Wazuh dashboards exist
  - Look for:
    - OpenVAS findings correlated to affected hosts
    - Nuclei execution results
    - Real-time alert feed in Wazuh app

- Practical detections to demonstrate:
  - Failed login attempts from the vulnhost
  - Outbound scans or unexpected open ports discovered by OpenVAS
  - Nuclei templates matched against common misconfigurations

- Commands to validate detections locally:
  - journalctl -u wazuh-agent -f
  - sudo tail -f /var/ossec/logs/alerts/alerts.log
  - curl -s http://localhost:5601/api/dashboards/list | jq

Segment D: Harden (17:31–20:00)
- Remediation steps that close the gaps demonstrated

1) Wazuh hardening
- Enable real-time monitoring, tune decoders, suppress non-critical noise
- Example:
  - edit /var/ossec/etc/ossec.conf to enable syslog for your environment
  - Put in place a reminder to rotate and encrypt logs

2) Patch management workflow
- OpenVAS/Nuclei results drive patching
  - apt update && apt upgrade -y on vulnerable hosts
  - Expand to a weekly scan cadence

3) Network hardening
- Minimal services exposed:
  - disable unnecessary ports, firewall rules
  - ufw enable; ufw default deny; ufw allow 22/tcp; ufw allow 5601/tcp; ufw allow 9392/tcp

4) Containment and monitoring
- Segmentation: place the lab in a dedicated VLAN/host-only network
- Ensure agent-based monitoring on all hosts so Wazuh sees lateral movement indicators

5) Verification
- Re-run OpenVAS/Nuclei scans to confirm remediation
- Validate no critical findings remain and alerts are properly gen’d

Closing (20:00)
- Quick recap of the Build-Break-Detect-Harden cycle
- Mention next episode tease

Notes
- Commands shown are for a lab environment. Adapt versions and paths to your setup.
- Replace placeholder IPs with your lab IPs.
- Ensure you have permission to run scans on all hosts in your environment.

---

## 2) 3-MIN TEASER SCRIPT

- Hook (0:00–0:10): “What if your homelab could practically self-dolve a vulnerability chain before it bites you? Wazuh, OpenVAS, and Nuclei—together in one power-packed cycle.”
- Core message (0:11–2:40): “We’ll build a compact lab, break it with a safe, controlled vuln, detect it with a real SIEM and scanners, then lock it down with concrete hardening steps. No fluff, just the workflow a senior engineer relies on.”
- Quick hits (2:41–4:30): “Dockerized Wazuh, Greenbone/OpenVAS, and Nuclei templates. We’ll show exact commands to spin it up, trigger a test vuln, and confirm you’re getting alerts in your SIEM.”
- Call to action (4:31–5:00): “Full episode on blaqlabs”

- End frame (final second): “Full episode on blaqlabs”


---

## 3) LINKEDIN DRAFT

One lesson from Ep05 (Wazuh + OpenVAS + Nuclei in a homelab):
- Lesson: A layered detection-and-harden approach pays off only when you close the loop between what you scan and how you alert. In this episode we demonstrated wiring Wazuh with OpenVAS and Nuclei, then translating findings into real-time alerts and concrete remediation steps. The key takeaway: automate your discovery, automate your response, and validate with repeatable hardening checks.  
- If you want to discuss the workflow or need a copy of the commands I used, drop a comment and I’ll share a sanitized setup you can try in your own lab.