20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden. For a homelab. No fluff, senior engineer tone. Include commands.

Episode Ep05: Wazuh + OpenVAS + Nuclei

INTRO (0:00–0:45)
- Opening shot: dim-lit desk, multiple monitors, terminal glow.
- Host: “Welcome back to blaqlabs. This is Ep05. Today we build a lightweight blue-team stack in a homelab and push it through a controlled break to validate detection and harden the surface. We’ll wire Wazuh for host-based SIEM, OpenVAS for asset-wide vulnerability assessment, and Nuclei for fast, authenticated/unauthenticated checks. No fluff, just actionable steps you can reproduce. Let’s begin.”

BUILD: environment setup and integration (0:45–8:30)
- Plan overview: “Goal: a small lab with Wazuh manager, Wazuh agent on host, OpenVAS scanner, Nuclei container, and a target VM. We’ll wire logs to Wazuh, run a baseline OpenVAS scan, then run Nuclei for focused checks.”
- Prereqs: “Ubuntu 22.04+ host or similar; at least 4 cores, 8GB RAM for modest lab.”
- Commands sequence:
  1) Update & install prerequisites
     - sudo apt-get update && sudo apt-get upgrade -y
     - sudo apt-get install -y curl gnupg apt-transport-https ca-certificates lsb-release
  2) Install Docker (required for Nuclei/OpenVAS containers)
     - sudo apt-get install -y docker.io
     - sudo systemctl enable --now docker
     - sudo usermod -aG docker $USER
  3) Install Wazuh Manager (Docker) and Wazuh Agent
     - docker pull wazuh/wazuh-manager:4.x
     - docker run -d --name wazuh-manager -p 55000:55000 -v /wazuh/config:/usr/share/wazuh/config -v /wazuh/logs:/var/ossec/logs wazuh/wazuh-manager:4.x
     - docker run -d --name wazuh-agent --link wazuh-manager:wazuh-manager -e ADDRESS=127.0.0.1 -e MANAGER_IP=127.0.0.1 -v /wazuh-agent/data:/var/ossec cisco/xxx  (Note: actual agent image varies; replace with wazuh/wazuh-agent and proper config)
     - On target host: install Wazuh agent
       - curl -so wam-agent.sh https://packages.wazuh.com/4.x/install.sh
       - sudo bash wam-agent.sh
       - # During enrollment, point to manager: https://<manager-ip>:55000
  4) Install OpenVAS (Greenbone Vulnerability Management)
     - sudo apt-get install -y openvas
     - sudo gvm-setup
     - sudo gvm-feed-update
     - sudo runuser -u _gvm -- gvmd --listen 0.0.0.0 --port 9392
     - sudo runuser -u _gvm -- greenbone-nvt-sync
     - # Access: https://<host>:9392 (default admin: admin)
  5) Install Nuclei (fast, powerful scanning tool)
     - sudo curl -sSfL https://raw.githubusercontent.com/projectdiscovery/nuclei-cli/master/install.sh | sudo bash
     - nuclei -version
     - docker run -t -i projectdiscovery/nuclei:latest -version
     - # Quick authenticated/unauthenticated scans with templates
  6) Basic integration checks
     - curl -s http://localhost:55000; echo
     - openvas-check-setup
     - nuclei -update-templates

- Logging and data flow
  - “Wazuh agent ships logs to manager; OpenVAS reports findings; Nuclei results can be ingested into Wazuh via decoders or manual parsing.”
  - sudo tee /wazuh/config/wazuh-agent.conf << 'EOF'
  - manager: wazuh-manager
  - registry: 127.0.0.1
  - EOF
  - systemctl restart wazuh-agent (on host)
- Verification (8:30–12:00)
  - Wazuh dashboard: https://<manager>:55000 (default creds)
  - OpenVAS: run a quick scan against 192.168.1.0/24
  - Nuclei: run a few templates
    - nuclei -u http://192.168.1.10 -t templates/cves/ -o nuclei-results.txt
    - nuclei -l targets.txt -t http://nas.local/.well-known/core -o nuclei-core.txt
- Quick baseline results: summarize expected findings and how to triage in Wazuh

BREAK (12:00–16:30)
- We simulate a breach scenario in-lab
  - Attack vector: misconfigured SSH (weak password) or outdated OpenVAS finding
  - Steps:
    - On a victim VM, attempt login with a weak user:
      - ssh 192.168.1.20
      - Password: try "password", "admin123"
    - If allowed, map the lateral movement: touch a test file in /tmp, exfil via netcat
  - Commands to simulate
    - sshpass -p 'password' ssh -o StrictHostKeyChecking=no user@192.168.1.20 'whoami; id; uname -a'
    - echo "TEST" > /tmp/poc.txt
    - nc -lvp 4444 < /tmp/poc.txt
- Observing detection
  - Wazuh alert: SSH login failure or new user activity
  - OpenVAS findings may show exploitable services (if misconfigured)
  - Nuclei can catch exposed endpoints
- Immediate containment steps
  - Block offending IPs with iptables
  - Disable weak accounts
  - Patch vulnerabilities via OpenVAS remediation suggestions
  - Rotate secrets and enforce MFA where possible

DETECT (16:30–20:00)
- How Wazuh detects the break
  - File integrity monitoring triggers on /etc, /root, /home
  - Real-time alerts for new users, sudo activity, and unusual processes
  - Example rule triggers:
    - useradd or adduser execution
    - new SSH login from unfamiliar IP
    - privileged commands executed (sudo)
- How to validate detections
  - Run a test SSH login, verify Wazuh alert in the UI
  - OpenVAS vulnerability status update
  - Nuclei results ingestion into Wazuh via decoders
- Tuning tips
  - Customize rules for lab assets
  - Create suppression for known benign activity
  - Schedule periodic scans and ensure log retention

HARDEN (20:00–25:00)
- Practical hardening steps
  - Wazuh hardening
    - Enable host integrity monitoring; set alert thresholds
    - Use TLS for manager-agent communications
    - Rotate and store secrets securely (e.g., via a vault)
  - OpenVAS hardening
    - Keep GVM components updated
    - Limit scan scope to known assets
    - Use authenticated scans where possible for accuracy
  - Nuclei hardening
    - Regularly prune old templates; pin safe versions
    - Use least privilege when running containers
- Remediation workflow
  - Patch critical CVEs per OpenVAS findings
  - Disable unused services; close open ports
  - Implement firewall rules on host and network
- Final lab recap
  - Confirm that Wazuh detects the breach containment
  - Re-scan with OpenVAS and Nuclei; verify reduced risk
  - Document the playbook and keep it versioned

OUTRO (25:00–27:00)
- “That’s Ep05. Wazuh + OpenVAS + Nuclei in a compact homelab. Build, Break, Detect, Harden. Reproduce, verify, and iterate.”
- CTA: “If you found this useful, hit like, subscribe, and share your lab topology in the comments.”

3-MIN TEASER SCRIPT - Same topic, punchy, hook in first 5 seconds, ends with "Full episode on blaqlabs".

Ep05 Teaser (3 minutes)

- Hook (0–5s): “What if your home lab could spot a breach before it even happens? Today we build a lightweight shield with Wazuh, OpenVAS, and Nuclei.”
- Fast pace setup (0:05–1:20)
  - “In this episode we’ll deploy Wazuh for host-based detection, run OpenVAS around your lab assets, and add Nuclei for fast, focused checks. No fluff, just hands-on steps you can reproduce.”
- Quick hit points (1:20–2:20)
  - “We’ll wire logs from the agent to the manager, scan a test target with OpenVAS, and kick off Nuclei scans to surface obvious abuse surfaces.”
  - “Then we simulate a breach in a controlled way, and watch how alerts fire in Wazuh, how OpenVAS findings get triaged, and how to harden in real-time.”
- Call to action (2:20–3:00)
  - “If you want the full, step-by-step lab build with commands, head to the full episode. Full episode on blaqlabs”

LINKEDIN DRAFT - I will post this manually. Short, human, 1 lesson from the episode.

LinkedIn draft

One lesson from Ep05: Wazuh, OpenVAS, and Nuclei together reveal that layered visibility is the true defense. In a homelab, instrumenting host analytics (Wazuh), network/app findings (OpenVAS), and fast, targeted checks (Nuclei) creates a feedback loop: detect early, triage fast, and harden continuously. Actionable takeaway: start with a lightweight Wazuh-agent-to-manager setup, run a scoped OpenVAS assessment, and integrate Nuclei scans into your alert workflow to close the loop between discovery and remediation.