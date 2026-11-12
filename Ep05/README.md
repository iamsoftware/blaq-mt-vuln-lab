1) 20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden (Ep05: Wazuh + OpenVAS + Nuclei for a homelab)

Title card: Ep05 — Wazuh + OpenVAS + Nuclei in a Homelab

Intro (0:00–0:30)
- Welcome back to blaqlabs. Episode 05: Wazuh, OpenVAS, and Nuclei in a pragmatic homelab workflow.
- Quick scope: secure-by-default mindset, repeatable tooling, not “hacky tricks” but a disciplined SDLC for defense.
- Assumptions: you own the lab, you own the assets, you’re scanning only what you own.

Segment 1: Build (0:30–5:00) — Set up stack in a controlled homelab
- Objective: deploy a minimal, reproducible stack: Wazuh for agent-based + log analysis, OpenVAS/GVM for vulnerability scanning, Nuclei for fast asset-aware checks.
- Targets: your own VMs or containers in a local network (e.g., 192.168.1.0/24). No external hosts without explicit permission.
- Prereqs (on a typical Debian/Ubuntu host):
  - sudo apt update && sudo apt install -y curl wget gnupg lsb-release docker.io docker-compose
  - Enable Docker service: sudo systemctl enable --now docker
- Step A: Deploy Wazuh (manager + Elasticsearch) with Docker-Compose (simple, repeatable)
  - Create a docker-compose.yml:
    - version: "3.8"
    - services:
      - wazuh-manager:
          image: wazuh/wazuh:4.3.0
          container_name: wazuh-manager
          ports: 
            - "55000:55000"  # API
            - "1514:1514/udp" # back-end agent communication
      - wazuh-elasticsearch:
          image: docker.elastic.co/elasticsearch/elasticsearch:7.9.3
          environment:
            - discovery.type=single-node
          ports:
            - "9200:9200"
      - wazuh-apiserver:
          image: wazuh/wazuh-api:4.3.0
          depends_on: ["wazuh-manager"]
    - volumes: ...
  - Bring up: sudo docker-compose up -d
  - Agent installation (on a host in the lab):
    - curl -sSf http://<manager-host>:55000 -o /tmp/wazuh-agent.rpm
    - sudo dpkg -i /path/to/wazuh-agent.deb
    - sudo systemctl enable --now wazuh-agent
    - Edit agent.conf to point to manager:MANAGER_ADDRESS
  - Validate: curl http://localhost:55000/jsentication  (check API)
- Step B: Deploy OpenVAS/GVM (greenbone)
  - Install essentials:
    - sudo apt install -y build-essential libsqlite3-dev cmake pkg-config libglib2.0-dev
  - Install GVM:
    - sudo apt install -y gvm
    - sudo gvm-setup
    - sudo gvm-feed-update
    - sudo gvm-start
  - Verify:
    - sudo gvmd --version
    - openvas-start not required if using gvm
  - Access GUI: https://<lab-ip>:9392 (default admin: admin)
- Step C: Nuclei setup (fast, templated checks)
  - Install Nuclei:
    - GO 1.20+ is recommended:
      - sudo apt install -y golang-1.20-go  (or install via gvm/go1.20)
      - go install -v github.com/projectdiscovery/nuclei/v2/cmd/nuclei@latest
  - Update templates:
    - nuclei -update-templates
  - Create a targets file (targets.txt) with lab hosts or use an internal inventory
  - Run a baseline scan:
    - nuclei -l targets.txt -t nuclei-templates/ -o nuclei-baseline.json
  - Template selection examples:
    - -t cves/ -t router/ -t ports/ (adjust to your lab)
- Step D: Secure networking and user access
  - Disable unused services on lab hosts
  - Enforce SSH key-based login; disable password auth
  - Use a dedicated management network for the stack
  - Enable basic TLS for OpenVAS/Wazuh UI
- Segment transition: how these tools fit together
  - Wazuh: central log collection, file integrity monitoring, alerting on suspicious activity
  - OpenVAS: authenticated vulnerability scanning, CVE checks, risk scoring
  - Nuclei: fast, templated checks against assets to catch misconfig or exposure quickly

Segment 2: Break (5:00–12:00) — What to break and how to think about it
- Real-world risk you’re simulating
  - External exposure: web apps, misconfigured endpoints
  - Misconfig: weak SSH keys, outdated services
  - Lateral movement: weak SMB shares, default creds
- Break plan (non-destructive)
  - Use a safe lab host or intentionally vulnerable container (e.g., OWASP Juice Shop in a separate VLAN)
  - Do not test production networks; isolate and snapshot
- Example attack flow (lab-safe, high-level)
  - Nuclei finds a public-facing misconfiguration on a lab host: misconfigured port 80 redirect or outdated header
  - OpenVAS reports a vulnerability in an internal service (e.g., old OpenSSH)
  - Wazuh triggers an alert on new agent activity or file changes
- Commands you’ll actually run (for breaking in a controlled lab)
  - Nuclei targeted scan (non-destructive):
    - nuclei -l targets.txt -t http-* -o nuclei-http-checks.txt
  - OpenVAS quick check (scan task):
    - gvm-cli --xml "<create_task>...</create_task>"  --target 192.168.1.100
  - SSH key discovery (lab host):
    - ssh-keyscan 192.168.1.100 >> ~/.ssh/known_hosts
  - Validate agent enrollment in Wazuh:
    - curl -s http://localhost:55000/agents | jq .
- Important caveats
  - Never scan devices you do not own or have explicit permission to test
  - Use snapshots/rollbacks before starting intrusive checks
  - Document all actions for reproducibility and post-mortem

Segment 3: Detect (12:00–17:50) — How to detect and correlate
- Detection goals
  - Identify exploit attempts, misconfig exposures, and policy deviations
  - Correlate events across Wazuh, OpenVAS, and Nuclei scans
- Wazuh-focused detection
  - Use decoders and rules for:
    - Unexpected SSH login patterns
    - New file changes in critical paths
  - Example rule concept (high level):
    - if alert.type == “warning” and host in VMs with OpenVAS findings
- OpenVAS findings handling
  - Map CVEs to remediation steps in Wazuh notes
  - Create a watchlist of affected assets and track remediation progress
- Nuclei output handling
  - Normalize outputs into JSON, import into Wazuh via log collector or API
  - Tag findings by template type: http-config, SSL, or exposure
- Practical workflow
  - Step 1: Run Nuclei, generate report
  - Step 2: Run OpenVAS scan, export results
  - Step 3: Wazuh ingests logs from agents and fuses findings
  - Step 4: Create a unified incident in your ticketing system (internal, for dev)
- Commands (detection-oriented)
  - Wazuh health check:
    - docker exec -it wazuh-manager /bin/bash -c "curl -s http://localhost:55000/"
  - OpenVAS/GVM status:
    - gvm-cli --username admin --password <pass> task list
  - Nuclei quick parse:
    - nuclei -l targets.txt -o json:nuclei.json
  - Simple correlation snippet (pseudo):
    - grep -E "SSH|OpenSSH" nuclei.json OpenVAS.json | join by host

Segment 4: Harden (17:50–20:00) — Hardening actions you can apply now
- Wazuh hardening
  - Enable TLS on API and UI
  - Enforce agent enrollment with certificates
  - Disable unused modules and reduce log verbosity on non-prod
- OpenVAS/GVM hardening
  - Regularly update feeds
  - Limit access to the UI to management network
  - Turn on authenticated scans with least privilege
- Nuclei hardening mindset
  - Keep templates updated; prune deprecated ones
  - Use inventory-based targeting to avoid noisy checks
- Lab hygiene steps
  - Snapshot before major changes
  - Version control your configuration (Docker Compose files, Wazuh rules, OpenVAS config)
  - Document remediation steps and verify via re-scan
- Final take
  - This is not a one-off; it’s an iterative hardening loop: deploy -> test -> detect -> remediate -> verify

Closing (20:00–20:30)
- Recap: Wazuh for detection, OpenVAS for vulnerability inventory, Nuclei for fast checks. A disciplined cycle: Build -> Break -> Detect -> Harden.
- Call to action: try the pipeline in your homelab and share your findings.

2) 3-MIN TEASER SCRIPT - Same topic, punchy, hook in first 5 seconds, ends with "Full episode on blaqlabs"

Hook (0:00–0:10)
- You’re about to see a defense-in-depth playbook for your homelab: Wazuh, OpenVAS, and Nuclei in one tight loop.

Punch (0:10–2:40)
- Build a repeatable stack: Wazuh agent fleet, GVM/OpenVAS for weekly checks, and Nuclei for fast, template-based tests.
- I’ll show you exactly how to deploy, how to break in a safe lab, and how to detect and harden without breaking production.
- You’ll see concrete commands you can copy-paste in your own lab: docker-compose up, gvm-setup, nuclei -update-templates, and a simple Wazuh alert test.
- The key: correlation across tools so you can prioritize fixes and prove your hardening in a single cycle.

Close (2:40–3:00)
- If you’re building a robust homelab defense, this is the flow you’ll want. Full episode on blaqlabs

3) LINKEDIN DRAFT - I will post this manually. Short, human, 1 lesson from the episode.

Draft:
Just finished Ep05: Wazuh + OpenVAS + Nuclei for a practical homelab defense loop. One concrete lesson: always run a secure, repeatable build first, then your break tests in a sandbox, and finally correlate detections across tools to confirm when remediation actually works. The power isn’t in any single tool—it’s in the workflow: Build -> Break -> Detect -> Harden. If you’re building a resilient lab, this approach removes guesswork and makes security measurable. Full episode on blaqlabs

Note: Ensure you have rights to test all targets in your environment.