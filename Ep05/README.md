## 1) 20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden (Ep05: Wazuh + OpenVAS + Nuclei)

Episode outline: Ep05. Topic: Wazuh + OpenVAS + Nuclei. Homelab, senior engineer tone. Include commands.

- Opening (0:00–0:45)
  - Welcome back. This episode covers a wired-up homelab stack using Wazuh for SIEM, OpenVAS for vulnerability scanning, and Nuclei for fast, configurable content scans. Objective: show the build, how to break it safely in a lab, how to detect with Wazuh, and harden with concrete mitigations.
  - Prereqs: a small, isolated network (no public IP exposure in this video), Ubuntu 22.04+ on a host or VM, enough CPU/RAM (4 vCPU, 8 GB RAM min). 
  - Topology we’ll implement:
    - Wazuh manager + agent (local lab)
    - OpenVAS scanner (Greenbone) on a separate host or container
    - Nuclei for rapid, templated scans
  - Commands you’ll see today are representative; adapt to your environment.

- Build phase: Set up the stack (0:45–8:00)
  - Step 1: Prepare the host and install prerequisites
    - sudo apt-get update && sudo apt-get upgrade -y
    - sudo apt-get install -y curl wget apt-transport-https ca-certificates gnupg lsb-release
  - Step 2: Deploy Wazuh (manager + agent)
    - Wazuh repo setup (manager)
      - curl -s https://packages.wazuh.com/4.x/apt/conf/wazuh.list | sudo tee /etc/apt/sources.list.d/wazuh.list
      - curl -s https://packages.wazuh.com/4.x/apt/gpgkey | sudo apt-key add -
      - sudo apt-get update
      - sudo apt-get install -y wazuh-manager
      - sudo systemctl enable wazuh-manager && sudo systemctl start wazuh-manager
    - Wazuh agent (on same host for simplicity or remote)
      - sudo apt-get install -y wazuh-agent
      - sudo sed -i 's/host=YOUR_MANAGER_IP/host=127.0.0.1/' /var/ossec/etc/ossec.conf
      - sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent
    - Manager–Agent connection health check
      - sudo tail -f /var/ossec/logs/ossec.log
  - Step 3: Deploy Wazuh dashboard (Elasticsearch/Kibana)
    - Optional: use Wazuh Docker stack for simplicity
      - docker run -d --name wazuh-dashboard -p 5601:5601 -p 5602:5602 \
        -e ELASTICSEARCH_URL=http://elasticsearch:9200 \
        -e KIBANA_URL=http://kibana:5601 \
        --network wazuh-net ghcr.io/wazuh/wazuh-docker:latest
  - Step 4: Deploy OpenVAS (Greenbone Vulnerability Management)
    - Install prerequisites
      - sudo apt-get install -y software-properties-common
    - Install OpenVAS
      - sudo add-apt-repository -y ppa:mrazavi/openvas
      - sudo apt-get update
      - sudo apt-get install -y openvas
      - sudo runuser -l _openvas -c 'gvmd --version'  # verify
    - Initialize OpenVAS
      - sudo greenbone-nvt-sync
      - sudo openvasmd --create-user admin
      - sudo openvasmd --user admin --new-password adminpass
      - sudo openvas-start
    - Basic usage
      - sudo omp -u admin -w adminpass -s /analysis
      - OpenVAS web UI at https://<scanner_ip>:9392
  - Step 5: Deploy Nuclei for fast scanning
    - Install Nuclei
      - go install -v github.com/projectdiscovery/nuclei/v2/cmd/nuclei@latest
      - export PATH=$PATH:$(go env GOPATH)/bin
    - Run a baseline template scan against your internal range
      - nuclei -update-templates
      - nuclei -t templates/web-*.yaml -target 10.0.0.0/24
  - Step 6: Basic integration scaffolding
    - Wazuh agent log transport (local logfile as example)
      - echo '2026-11-12: test log msg' | logger -t lab-test
    - OpenVAS results ingest into Wazuh (basic approach)
      - Use a script to parse OpenVAS reports and create alerts in Wazuh via local API
      - curl -u wazuh:wazuh -X POST "http://localhost:55000/security/events?filter=pluginid:10000"
  - Step 7: Network hardening baseline for lab
    - SSH hardening
      - sudo sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config
      - sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
      - sudo systemctl reload sshd
    - Firewall
      - sudo ufw default deny incoming
      - sudo ufw default allow outgoing
      - sudo ufw allow 2222/tcp  # your SSH port
      - sudo ufw enable
    - Wazuh manager hardening (minimum)
      - Disable unused rule actions in /var/ossec/rules
      - Ensure TLS for agent-manager traffic
  - Step 8: Validation checks
    - Wazuh agent status
      - sudo systemctl status wazuh-agent
      - sudo tail -f /var/ossec/logs/ossec.log
    - OpenVAS scanner status
      - sudo service openvas-scanner status
      - sudo openvas-start
    - Nuclei quick check
      - nuclei -update-templates
      - nuclei -t templates/high-impact.yaml -target 10.0.0.0/24
  - Step 9: Quick recap of the flow
    - Build a centralized visibility stack with Wazuh, OpenVAS, and Nuclei
    - Run regular scans with OpenVAS and Nuclei
    - Feed findings into Wazuh for correlation and alerting
    - Harden by tightening SSH, firewall, and IDS rules
  - Step 10: Safe teardown / note
    - In a lab, ensure no external exposure
    - Use isolated networks or VLANs
    - Regularly rotate credentials and review alerts

- Break phase (8:00–12:00)
  - Simulated breach flow for demonstration (no real exploitation)
    - Step A: Asset discovery
      - nmap -sS -Pn 10.0.0.0/24 -oA netscan
    - Step B: Vulnerability check (OpenVAS)
      - Run a targeted scan against a lab host, export report
      - openvas-report --format pdf --output lab_scan.pdf
    - Step C: Nuclei content check
      - nuclei -t templates/ -target 10.0.0.5
  - Step D: Basic alerting sanity check
    - Generate a test event to Wazuh
      - logger -t lab-test "TEST: alert sanity check"
    - Check Wazuh dashboard for new alert
      - curl -sS -u admin:admin http://localhost:5601/api/status
  - Step E: Attack surface review (no exploits)
    - Review common misconfigurations found by OpenVAS
    - Note: this is defensive testing; do not attempt exploitation

- Detect phase (12:00–16:00)
  - Correlating events in Wazuh
    - Create a rule snippet example (conceptual)
      - When an OpenVAS high-risk finding appears, trigger a console alert
      - Rules in /var/ossec/rules/local_rules.xml
  - Verify alert delivery
    - tail -f /var/ossec/logs/alerts.log
    - Check email/Slack webhook if configured
  - Anomaly detection basics
    - Look for unusual login times, failed SSH attempts from OpenVAS host, or unexpected OpenVAS-related user activity

- Harden phase (16:00–20:00)
  - Apply mitigations per finding
    - Remove or restrict exposed services discovered by OpenVAS
    - Harden OpenVAS with least-privilege user roles and TLS
    - Tighten Nuclei scan templates to only needed domains
  - Wazuh best practices
    - Enable TLS for manager-agent traffic
    - Enforce strong agent authentication
    - Regularly rotate API keys and disable unused integrations
  - SSH and host hardening recap
    - Non-root sudo users
    - SSH on non-standard port with key-based authentication
  - Network segmentation
    - Separate OpenVAS and Wazuh components with internal firewalls
    - Use VLANs to isolate critical assets
  - Final validation
    - Re-run a quick OpenVAS scan, Nuclei sweep, and verify Wazuh alerts
    - Confirm no sensitive data is leaking in logs

- Closing (20:00–21:00)
  - Key takeaways
    - Wazuh centralizes visibility; OpenVAS and Nuclei provide proactive scanning
    - Automate ingestion and alerting for rapid dwell-time reduction
    - Harden iteratively and validate with repeatable checks
  - Call-outs
    - Like, subscribe, and share if this stack helps your homelab
    - Links to code/scripts in the description
  - Sign-off
    - Ep05 complete. See you next time.

— End of script —

---

## 2) 3-MIN TEASER SCRIPT - Same topic, punchy, hook in first 5 seconds, ends with "Full episode on blaqlabs"

- Opening hook (0:00–0:15)
  - "What if your homelab could see every vulnerability before an attacker does? Wazuh, OpenVAS, and Nuclei—the triple-stack that turns blind spots into edge-to-edge visibility."
- Quick promise (0:15–0:40)
  - "In 3 minutes, build the stack, run a controlled breach-in-lab, and show you exactly how detection and hardening kick in with real commands you can reuse."
- Core slice (0:40–2:20)
  - "We install Wazuh for centralized logging, spin up Greenbone OpenVAS for vulnerability scanning, and enable Nuclei for fast, templated checks. Then we fake a threat in a safe lab, verify alerts, and lock things down with concrete hardening steps."
  - Brief command flavor: "apt-get install wazuh-manager, openvas-start, nuclei -t templates/web-*.yaml, curl to Wazuh API, ufw rules"
- CTA (2:20–2:40)
  - "If you want the full build-out, the integration scripts, and the test reports, tune in."
  - "Full episode on blaqlabs"

- Outro hook (2:40–3:00)
  - "Full episode on blaqlabs"

---

## 3) LINKEDIN DRAFT - I will post this manually. Short, human, 1 lesson from the episode.

One lesson from Ep05: Wazuh + OpenVAS + Nuclei creates end-to-end visibility in a homelab by combining centralized log correlation with proactive vulnerability scanning and fast content checks. The key is automating the ingestion and alerting so you can detect and harden before real threats materialize. Full episode on blaqlabs.