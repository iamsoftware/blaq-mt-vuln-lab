# 1) 20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden. For a homelab. No fluff, senior engineer tone. Include commands.

Episode: Ep05
Topic: Wazuh + OpenVAS + Nuclei

Intro (0:00–0:30)
- Host: Welcome to blaqlabs. Episode 05: Wazuh + OpenVAS + Nuclei in a homelab. We’ll Build the stack, Break it in controlled tests, Detect with the SOC workflow, and Harden the environment. No fluff, straight to the point.
- On-screen: Lab topology sketch (Wazuh server → OpenVAS/GVM → Nuclei agents/scans → SIEM ingestion)

Build phase (0:30–7:30)
- Host: Objective: stand up a repeatable lab with Wazuh, Greenbone (OpenVAS), and Nuclei. We’ll run everything in Docker for speed and reproducibility.

Step A — Prep the host
- Script cue: "Start with a clean Ubuntu 22.04 host."
- Commands:
  - sudo apt-get update && sudo apt-get upgrade -y
  - sudo apt-get install -y curl gnupg lsb-release ca-certificates docker.io docker-compose
  - sudo systemctl enable --now docker

Step B — Deploy Wazuh (SIEM) stack
- Script cue: Deploy Wazuh with ELK-like stack via Docker Compose (official)
- Commands:
  - mkdir -p ~/wazuh-docker && cd ~/wazuh-docker
  - cat > docker-compose.yml <<'YAML'
version: "3.8"
services:
  wazuh:
    image: wazuh/wazuh:4.3.10
    restart: always
    ports:
      - "55000:55000"
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.9
    environment:
      discovery.type: single-node
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
    deploy:
      restart_policy:
        condition: on-failure
  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.9
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
YAML
  - docker-compose up -d
  - echo "Waiting for Wazuh to boot (minutes)…"
  - sleep 180
  - curl -s http://localhost:5601/app/wazuh -o /dev/null || echo "Kibana not ready yet"

Step C — Deploy Greenbone/OpenVAS (GVM)
- Script cue: Use Greenbone Vulnerability Manager for OpenVAS scanning
- Commands (on host; if needed, you can run a separate containerized setup):
  - sudo apt-get install -y build-essential cmake pkg-config libglib2.0-dev
  - sudo apt-get install -y libksba-dev libgpgme-dev atk1.0-dev libssh-gcrypt-dev libgnutls28-dev
  - sudo apt-get install -y redis-server postgresql postgresql-contrib
  - sudo apt-get install -y gvm
  - sudo runuser -u _gvm -- bash -c "gvmd --version; greenbone-nvt-sync; greenbone-feed-sync --type GVMD"
  - sudo systemctl enable --now gvmd gsad greenbone-cert-data-scanner
  - # Create admin user
  - sudo runuser -u _gvm -- gvmd --created-user=admin
  - # Access: https://localhost:9392, default admin/password (change after first login)
  - echo "OpenVAS/GVM ready via port 9392"

Step D — Deploy Nuclei (fast external/internal scanning)
- Script cue: Lightweight, fast scanner for targeted checks
- Commands:
  - sudo apt-get install -y golang-go
  - go env -w GO111MODULE=on
  - go install -v github.com/projectdiscovery/nuclei/v2/cmd/nuclei@latest
  - echo "Nuclei installed: $(dirname $(go env GOPATH))/bin/nuclei"
  - nuclei -version
  - # Prepare a small template repository or use default
  - mkdir -p ~/nuclei-project/templates
  - # Example scan against a local target
  - ~/go/bin/nuclei -t ~/nuclei-project/templates/ -target 192.168.1.100

Step E — Ingest Nuclei/OpenVAS into Wazuh
- Script cue: Centralize findings for detection and response
- Commands:
  - # Example: fetch OpenVAS findings (GVMD export) and feed to Wazuh via JSON log format
  - curl -u admin:password -X GET "https://localhost:9392/gm/gm?scantype=full&format=json" -k
  - # Transform findings into a Wazuh-friendly JSON and push to Wazuh index via Filebeat or Logstash (not shown in full here)
  - # If using Wazuh agent on the same host, ensure directory monitoring for scan results
  - sudo mkdir -p /var/ossec/logs/scan
  - sudo chown -R root:root /var/ossec/logs/scan
  - # Example: echo "SCAN_RESULT: OK" >> /var/ossec/logs/scan/openvas.json

Break phase (7:30–12:00)
- Host: We’ll deliberately break the lab by simulating misconfig, mis-scan, and a credential leak test against the stack.

Step A — Break: misconfig and drift
- Script cue: Simulate an insecure OpenVAS user, weak Wazuh auth, and misconfigured network ACLs
- Commands:
  - # Invalidate Kibana admin password (example)
  - curl -u admin:oldpassword -X POST http://localhost:5601/api/security/v1/users/admin/_password -H "Content-Type: application/json" -d '{"password":"new_secure_#1!"}'
  - # Break Wazuh by disabling the index rotation (simulate retention drift)
  - docker exec -it wazuh-elasticsearch /bin/bash -c "curl -XPUT 'http://localhost:9200/_ilm/policy/wazuh-default' -H 'Content-Type: application/json' -d '{\"policy\":{\"phases\":{\"hot\":{\"min_age\":\"0ms\",\"actions\":{\"rollover\":{\"max_age\":\"7d\",\"max_size\":\"50gb\"}}}}}}'"
  - # Simulate network ACL misconfig
  - sudo iptables -A INPUT -s 0.0.0.0/0 -j DROP

Step B — Break: targeted test
- Script cue: Trigger a controlled vulnerability scan to verify detection
- Commands:
  - nuclei -t /usr/share/nuclei/templates/ -target 192.168.1.0/24 -scan-delay 1s -stop-at-nth 10
  - openvas-cli --host 127.0.0.1 --port 9392 --user admin --password 'password' --scan 

Step C — Break: persistence attempt (safe, in-lab)
- Script cue: Attempt to exfiltrate to a dummy external host (blocked)
- Commands:
  - curl -X POST http://127.0.0.1:6666 -d "test=break"
  - # Observe Wazuh alerts for suspicious outbound attempts

Detect phase (12:00–17:30)
- Host: Now we detect what happened with the stack, correlating data across Wazuh, OpenVAS, and Nuclei.

Step A — Build detection rules
- Script cue: Create a few focused rules in Wazuh to catch misconfig, scan anomalies, and failed logins
- Commands (conceptual):
  - # In Wazuh rules.json, add rules:
  - 'rule': { 'level': 10, 'description': 'OpenVAS scan detected', 'decoded_as': 'json', 'condition': 'openvas_scan:status:failed' }
  - 'rule': { 'level': 8, 'description': 'Weak Kibana admin password detected', 'condition': 'auth.login_event.username:admin and event.password:weak' }

Step B — Run live detections
- Script cue: Let Wazuh ingest logs and show alerts
- Commands:
  - # Check Wazuh alerts
  - curl -u wazuhuser:password -X GET "http://localhost:5601/api/alerts/_find?size=100" -H 'Content-Type: application/json'
  - # Verify Nuclei findings appear in Wazuh
  - tail -n +1 /var/ossec/logs/alerts/alerts.json | grep -i nuclei

Step C — OpenVAS findings flow
- Script cue: Ensure scans show up in Wazuh
- Commands:
  - # OpenVAS/GVM: export a scan as XML/JSON
  - # Expect a log entry or API payload that can be parsed by Wazuh

Harden phase (17:30–22:00)
- Host: Harden the environment against the observed gaps and improve resilience.

Step A — Remove drift
- Script cue: Re-enable proper ACLs, rotate credentials, tighten Wazuh access
- Commands:
  - sudo iptables -D INPUT -s 0.0.0.0/0 -j DROP
  - curl -u admin:new_secure_#1! -X POST http://localhost:5601/api/security/v1/users/admin/_password -H "Content-Type: application/json" -d '{"password":"EvenBetter$ecret9!"}'
  - # Re-enable index rotation policy to default
  - docker exec -it wazuh-elasticsearch /bin/bash -c "curl -XPUT 'http://localhost:9200/_ilm/policy/wazuh-default' -H 'Content-Type: application/json' -d '{\"policy\":{\"phases\":{\"hot\":{\"min_age\":\"0ms\",\"actions\":{\"rollover\":{\"max_age\":\"14d\",\"max_size\":\"100gb\"}}}}}}'"

Step B — Harden OpenVAS/GVM and Nuclei usage
- Script cue: Limit exposure, use authenticated scans, and run templates with safe defaults
- Commands:
  - # Use authenticated scan templates and limit external exposure
  - nuclei -t /usr/share/nuclei/templates/ -target 192.168.1.0/24 -headers "Authorization: Bearer <token>" --safe-seed
  - # Disable unauthenticated OpenVAS admin access from public networks
  - sudo firewall-cmd --permanent --add-rich-rule 'rule family="ipv4" source address="0.0.0.0/0" port port="9392" protocol="tcp" accept'
  - sudo firewall-cmd --reload

Step C — Validation and rollback plan
- Script cue: Confirm detections are stable and create a rollback plan
- Commands:
  - # Validate Wazuh health
  - curl -s http://localhost:5601/app/wazuh -o /dev/null || echo "Wazuh not ready"
  - # Snapshot docker compose state
  - docker-compose ps
- Final note: Document changes, ensure backups, and schedule a follow-up test.

Closing (22:00–23:00)
- Host: Quick recap: Build a repeatable stack, intentionally break in a controlled way, validate with cross-team detections, and harden to reduce blast radius. Questions? See show notes for commands and file references.

End screen
- On-screen: Subscribe, Full episode on blaqlabs

Notes for production
- Adjust IPs, credentials, and domain names to your lab.
- Use secure vaults for credentials; avoid hardcoding in scripts.
- This is a high-signal, low-fidelity script intended for a technical audience; adapt to your environment.

# 2) 3-MIN TEASER SCRIPT - Same topic, punchy, hook in first 5 seconds, ends with "Full episode on blaqlabs".

Teaser (0:00–0:20)
- Host: Want a hacker-proof homelab in 2026? Build Wazuh, OpenVAS, and Nuclei in under 15 minutes and watch it survive a real attack. Here’s how we do it in Ep05.

Teaser (0:20–1:40)
- Host: We’ll build a tight stack, deliberately break it, detect every misstep with a real SOC workflow, and harden the setup so it’s not your first exposure. No fluff, no fluff, just results. Look for the exact commands and the proven sequence in the full episode.

Teaser (1:40–2:40)
- Host: You’ll see Wazuh ingestion of OpenVAS/GVM data, Nuclei scans feeding the defense loop, and a practical hardening checklist you can apply today.

Teaser (2:40–3:00)
- Host: Full episode on blaqlabs

# 3) LINKEDIN DRAFT - I will post this manually. Short, human, 1 lesson from the episode.

LinkedIn draft
One lesson from Ep05: In a homelab, integration beats isolation. Wazuh, OpenVAS, and Nuclei must talk to each other for detection to be timely. The real win isn’t just running scans; it’s wiring the findings into a centralized SOC workflow so misconfig and drift get surfaced before a real break occurs. Full episode on blaqlabs