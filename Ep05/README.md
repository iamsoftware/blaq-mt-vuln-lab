20-MIN YOUTUBE SCRIPT - Ep05: Wazuh + OpenVAS + Nuclei (Build -> Break -> Detect -> Harden)

Intro (0:00–0:45)
- Opening hook: "In 2026, your home lab is your security perimeter. Today we build a small SOC with Wazuh, we scan with OpenVAS, and we fuzz with Nuclei. All in a contained homelab." 
- Episode title slate: Ep05 — Wazuh + OpenVAS + Nuclei
- Quick plan: Build a compact stack in Docker, Break some noisy hotspots safely, Detect with alerts, Harden to sleep at night.

Part 1: Build (0:45–6:00)
- Goal: Stand up a compact, reproducible stack for log collection, asset discovery, and vulnerability scanning.
- Assumptions:
  - Host: Linux (Ubuntu/Duster) with docker and docker-compose installed
  - Networks: 172.20.0.0/16 for services; host network for some scans if needed
- Prereqs (commands):
  - sudo apt-get update && sudo apt-get upgrade -y
  - sudo apt-get install -y docker.io docker-compose curl wget unzip
  - sudo systemctl enable --now docker
- Docker-compose (example) — create docker-compose.yml (content snippet):
  - Explain that you’ll use Wazuh, Elasticsearch, OpenVAS (GVM), and Nuclei.
  - Provide a minimal docker-compose snippet (without full YAML to keep it concise; you’ll paste full in repo):
  - Key services: wazuh-manager, wazuh-agent, elasticsearch, kibana, gvm (OpenVAS), nuclei (or run nuclei in a container)
- Commands to write and run:
  - cat > docker-compose.yml << 'YAML'
    version: "3.8"
    services:
      wazuh-manager:
        image: wazuh/wazuh:4.3.10
        environment:
          - MANAGER=true
        ports:
          - "1514:1514/udp"
          - "55000:55000"
        networks:
          - wazuh
        restart: unless-stopped
      elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:8.6.3
        environment:
          - discovery.type=single-node
        ports:
          - "9200:9200"
        networks:
          - wazuh
        restart: unless-stopped
      kibana:
        image: docker.elastic.co/kibana/kibana:8.6.3
        ports:
          - "5601:5601"
        networks:
          - wazuh
        depends_on:
          - elasticsearch
        restart: unless-stopped
      gvm:
        image: securecomms/gvm:21.4
        ports:
          - "9392:9392"
          - "443:443"
        networks:
          - wazuh
        restart: unless-stopped
      nuclei:
        image: projectdiscovery/nuclei
        networks:
          - wazuh
        entrypoint: ["sh", "-c", "while true; do sleep 60; done"]
        restart: unless-stopped
    networks:
      wazuh:
        driver: bridge
    YAML
  - docker-compose up -d
- Verification:
  - docker ps
  - curl -s http://localhost:5601/api/status  (Kibana status)
  - curl -s http://localhost:9392  (GVM admin)
- Observations:
  - Wazuh should start ingesting logs; GVM should be ready to accept scans; Nuclei container ready for on-demand scans.

Part 2: Break (6:00–11:30)
- Objective: Simulate a controlled breach within the homelab to generate telemetry, without venturing into live external assets.
- Safe practice: Use a deliberately vulnerable host inside the docker-compose network or a dedicated test VM.
- Setup notes:
  - Create a deliberately vulnerable test service, e.g., a container running an old version of web app.
  - Example: a simple vulnerable nginx/JCVE app inside a separate container or VM.
- Commands to create a degraded host for break simulation:
  - docker run -d --name vuln-app --network wazuh -p 8080:80 httpd:2.4
  - docker exec vuln-app bash -lc "apt-get update && apt-get install -y openssl && echo 'VULN' > /var/www/html/index.html"
- Simulated break actions (safe, non-destructive):
  - From attacker perspective (in-lab): use a controlled scan with OpenVAS (GVM) against vuln-app
  - Commands to initiate a scan from GVM:
    - Open GVM UI at http://localhost:9392
    - Create target: 172.20.0.0/16 or specific 172.20.0.XX
    - Use a conservative scan policy (Low/Medium)
  - Nuclei quick tests (harmless templates):
    - docker exec nuclei nuclei -u http://172.20.0.XX -t templates/common/ (adjust path to templates)
- Break telemetry focus:
  - Expect increased authentication failures in Wazuh alert rules if surrogate brute-force is attempted (keep to localhost)
  - Look for vulnerability findings from OpenVAS (GVM) like outdated software, misconfig, default creds
- Realistic but safe steps:
  - Do not expose to internet; confine to docker network
  - Document any suspicious patterns: repeated login failures, outdated services, weak config

Part 3: Detect (11:30–16:30)
- Data sources:
  - Wazuh: logs from OS, apps, auth, SSH, and OpenVAS/GVM findings
  - OpenVAS/GVM: vulnerability reports, scan results
  - Nuclei: HTTP/URL checks with templates
- Detect workflow:
  - Wazuh rule: detect failed SSH logins and configuration drift
  - OpenVAS: collect scan results; create a rule to alert on critical/high findings
  - Nuclei: alert on critical templates (CVE-level)
- Commands (hands-on verification):
  - Check Wazuh agent status:
    - docker exec -it <agent> /usr/share/wazuh-manager/bin/ossec-control status
  - Retrieve Wazuh alerts:
    - curl -s -u user:pass http://localhost:5601/api/alerts/_search?size=50
  - OpenVAS/GVM scan results:
    - Use GVM UI to export results (PDF/CSV)
  - Nuclei checks:
    - docker exec nuclei nuclei -l /path/to/targets.txt -t templates/ -silent
- Observations to capture:
  - Time-stamped alerts for failed auth or misconfig
  - CVEs detected and remediation suggestions
  - Nuclei hits across your test host: note which templates hit and why
- Response playbook (alerting side):
  - If critical CVEs found: isolate affected container, rotate keys, patch, re-scan
  - If suspicious login: enforce 2FA for access to admin interfaces; enable Wazuh rules for IP-based blocking

Part 4: Harden (16:30–20:00)
- Harden goals:
  - Reduce attack surface; improve logging fidelity; enforce policy
- Harden steps:
  - Network hardening:
    - Restrict container intercommunication where not required
    - Use docker-compose networks with least privilege
  - Wazuh hardening:
    - Enable agent->manager TLS; ensure TLS certs; restrict API access
    - Enforce strong password policies for admin interfaces
  - OpenVAS/GVM hardening:
    - Use up-to-date feed; disable unauthenticated interfaces; limit scan scope
  - Nuclei hardening:
    - Use allowlists for templates; disable dangerous templates in production
- Commands to implement hardening:
  - Edit docker-compose.yml to enable TLS for Wazuh:
    - volumes:
      - ./certs:/etc/ssl/certs
      - ./wazuh-rules:/usr/share/wazuh/rules
  - Regenerate certificates (example with OpenSSL):
    - openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout cert.pem -out cert.pem
  - Apply firewall rules (on host):
    - sudo ufw allow 22,5601,9200,9392/tcp
    - sudo ufw deny to any from 0.0.0.0/0 to 0.0.0.0/0
- Validation:
  - Verify TLS on Wazuh API
  - Validate that only necessary ports are exposed
  - Re-run OpenVAS/Nuclei scans to ensure no regression
- Wrap-up:
  - Summarize learnings: Wazuh for visibility, OpenVAS for risk posture, Nuclei for breadth checks
  - Provide next steps for automation and CI integration
- End screen CTA:
  - Subscribe, join the BlaqLabs community, and check episode notes for repo links

Notes
- All commands assume a homelab with Docker and Docker-Compose installed and a safe, isolated network.
- Adapt paths and versions to your environment; avoid exposing scan interfaces to the public internet.

3-MIN TEASER SCRIPT - Ep05: Wazuh + OpenVAS + Nuclei

- Hook (0:00–0:05): "Your home lab is a target. Today we build a tiny SOC with Wazuh, OpenVAS, and Nuclei—in under 3 minutes."
- Quick promise (0:05–0:45): "We’ll build, break safely, detect in real time, and harden the stack so you sleep at night."
- Fast cuts: show docker-compose snippet, a quick grep of alerts, a scan starting in GVM, a nuclei hit
- Call-to-action (0:45–1:00): "Full episode on blaqlabs"

Full teaser (UI cue for visuals)
- Scene 1: terminal shows docker-compose up -d
- Scene 2: a simulated alert pops in Kibana
- Scene 3: a GVM scan progress bar completes
- Scene 4: hardened config saved, warning-free indicator
- End card: "Full episode on blaqlabs"

LINKEDIN DRAFT - Ep05 takeaway

- One lesson from Ep05: Co-locating Wazuh, OpenVAS (GVM), and Nuclei in a controlled homelab gives you end-to-end visibility: logs, vulnerabilities, and web-fortex checks all in one stack. The key is safe, contained testing and consistent hardening—don’t forget TLS, least privilege, and targeted scans. Full episode details in the BlaqLabs repo.