20-MIN YOUTUBE SCRIPT - Build -> Break -> Detect -> Harden (Ep05: Wazuh + OpenVAS + Nuclei for a homelab)

Intro (0:00-0:40)
- Greet: "What’s up, everyone? This is blaqlabs, Ep05. Today we’re building a compact security monitoring and scanning stack in a homelab using Wazuh, OpenVAS, and Nuclei. No fluff, just actionable steps and real-world caveats for a senior engineer."
- Objective: "Create a reproducible lab: detect, verify, and harden vulnerabilities using a tight triad. Build fast, break safely, detect effectively, harden relentlessly."
- Scope note: "All commands assume a Debian/Ubuntu-like host and root or sudo access. Replace distro-specific parts as needed."
- Quick roadmap: Build the stack, Break a sane misconfiguration, Detect with centralized logging and scan results, Harden the environment.

PART 1: Build (0:40-7:30)
Goal: Get Wazuh manager + agent, OpenVAS (greenbone) scanner, and Nuclei in a cohesive lab, with minimal friction.

A. Prepare the host
- Update and install essentials
  - sudo apt-get update && sudo apt-get upgrade -y
  - sudo apt-get install -y curl gnupg lsb-release ca-certificates apt-transport-https

B. Deploy Wazuh (manager + agent)
- Use the official Wazuh repo for Debian/Ubuntu
  - curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
  - echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list > /dev/null
  - sudo apt-get update
  - sudo apt-get install -y wazuh-manager wazuh-api
- Start and verify
  - sudo systemctl enable --now wazuh-manager wazuh-api
  - sudo tail -f /var/ossec/logs/ossec.log
- Add a local agent (on same host or another VM)
  - sudo apt-get install -y wazuh-agent
  - sudo nano /var/ossec/etc/ossec.conf  # set the server to your manager IP
  - sudo systemctl enable --now wazuh-agent
  - On manager: /var/ossec/bin/manage_agents
  - Add a new agent with name, key; then on agent: /var/ossec/bin/wazuh-control start

C. Deploy OpenVAS (Greenbone Vulnerability Management)
- Install dependencies and repository (assuming Debian/Ubuntu)
  - sudo apt-get install -y software-properties-common
  - sudo add-apt-repository -y ppa:mrazek/openvas
  - sudo apt-get update
  - sudo apt-get install -y openvas
- Initialize and start
  - sudo gvm-setup
  - sudo gvm-check-setup
  - sudo systemctl enable --now gvmd
  - Note: The first run can take a while to fetch feeds; watch for "up-to-date" status
- Access the admin console
  - sudo runuser -u _gvm -- bash -c 'gvmd --user=admin --password=YourP@ssw0rd'
  - Web UI at https://<host>:9392 with admin credentials
- Optional: tailor scanners by creating targeted scan configs for your assets

D. Deploy Nuclei (fast, focused scanning)
- Install Nuclei
  - curl -s https://raw.githubusercontent.com/projectdiscovery/nuclei/main/install.sh | bash
  - sudo mv nuclei /usr/local/bin/
- Run quick, useful templates
  - nuclei -update-templates
  - nuclei -l targets.txt -t cves/ -td  # example for CVE templates
- Integrate with CI or local scripts
  - Create a small script that runs: nuclei -l hosts.txt -t svt.yaml -o nuclei_results.json

E. Wire it together
- Centralize logs/events
  - Wazuh: ensure ossec.conf points to the manager; enable syslog output for OpenVAS/Nuclei results
  - Consider shipping OpenVAS/Nuclei outputs to Wazuh via logs or Syslog
- Basic inventory
  - Maintain a simple hosts.txt with IPs of lab assets
- Basic automation
  - cron jobs to run OpenVAS and Nuclei scans on a schedule and push results to a local directory or to Wazuh via log files

F. Validation quick checks
- Verify Wazuh status:
  - systemctl status wazuh-manager wazuh-api wazuh-agent
- Verify OpenVAS status:
  - systemctl status gvmd greenbone-scan-manager ospd-openvas
- Verify Nuclei version:
  - nuclei -version
- Quick scan sanity:
  - OpenVAS: Run a minimal scan against 127.0.0.1 or a known internal host
  - Nuclei: nuclei -l targets.txt -t http-* -o nuclei_scan.txt
- Check logs for ingestion:
  - tail -f /var/ossec/logs/ossec.log
  - tail -f /var/log/gvm/gsad.log
  - tail -f nuclei_scan.txt

PART 2: Break (7:30-12:30)
Goal: Simulate misconfigurations and ephemeral breakage, then demonstrate how to recover and validate.

A. Break 1 — misconfigured Wazuh agent
- Induce: corrupt the ossec.conf on an agent or disable the agent
  - sudo sed -i 's/.*server.*/# misconfiguration: no server/' /var/ossec/etc/ossec.conf
  - sudo systemctl restart wazuh-agent
- Observe: Wazuh manager shows disconnected agent; logs show connection errors
  - tail -f /var/ossec/logs/ossec.log

B. Break 2 — OpenVAS feed outage
- Induce: disable gvm-feed-sync or network egress
  - systemctl stop greenbone-feed-sync
  - Change DNS or block outbound to feed servers (iptables -A OUTPUT -d feed.server -j REJECT)
- Observe: gvmd may report feed update failures; scans timeout

C. Break 3 — Nuclei mis-scan config
- Induce: point to an invalid template or mis-specified target
  - nuclei -l targets.txt -t non-existent-template.yaml
- Observe: non-zero exit codes; log shows missing template error

D. Recovery steps (safe, guided)
- Wazuh agent: restore ossec.conf to correct server, then restart
  - sudo systemctl restart wazuh-agent
- OpenVAS: re-enable feeds and ensure gvmd is healthy
  - systemctl start greenbone-feed-sync
  - gvmd --update --quiet
- Nuclei: re-run with valid templates and targets
  - nuclei -update-templates
  - nuclei -l targets.txt -t httpx.yaml -o nuclei_recovered.json

E. Validation after recovery
- Check agent status: sudo systemctl status wazuh-agent; Wazuh dashboard shows connected agents
- Check OpenVAS: ensure feeds are up, run a quick scan to confirm results appear
- Check Nuclei: ensure templates load, scan completes successfully
- Confirm centralized visibility: the Wazuh console shows events from hosts, OpenVAS results show up in the console if integrated, and nuclei outputs are captured

PART 3: Detect (12:30-17:30)
Goal: Demonstrate detection workflows and data flow across Wazuh, OpenVAS, and Nuclei.

A. Wazuh detection
- Show agent-generated alerts for failed login attempts, misconfig
  - Go to Wazuh dashboard: Alerts -> Decoders -> look for syslog or agent errors
- Demonstrate a rule for failed SSH
  - If using default rules, show how to search:
    - wazuh-sec-api query 'security: sshd:authentication failure'
- Show how to create a custom rule (high-level)
  - Rule: IF event.category == "authentication" AND event.src_ip != "127.0.0.1" THEN set priority 7 and alert

B. OpenVAS results
- Explain how OpenVAS reports high/severe vulnerabilities
- Demonstrate exporting a report
  - In OpenVAS/GVM: Export as XML/HTML and import into Wazuh or a SIEM light integration
- Show how to map findings to remediation tasks (owner, asset, severity)

C. Nuclei findings
- Demonstrate running a targeted nuclei test against a host
  - nuclei -l hosts.txt -t http-default-favicon.yaml
- Explain how to triage findings: host context, path exposure, misconfig exposure
- Tie to Wazuh by shipping a simple finding annotation (e.g., create a custom log entry when a nuclei result hits a threshold)

D. Centralized workflow demo
- Show a live summary screen (or mock) where:
  - Wazuh alerts show new issues
  - OpenVAS provides high-severity failures
  - Nuclei returns fast, actionable endpoints
- Actionable triage: assign owners, create tickets, and schedule remediation

PART 4: Harden (17:30-20:00)
Goal: Harden lab posture, reduce exposure, and improve future detection fidelity.

A. Wazuh hardening
- Enforce agent-client mutual TLS (use certificates rather than plain agent-key)
- Enable tamper protection and log integrity monitoring
- Harden manager: limit API access, enforce strong auth, rotate keys

B. OpenVAS hardening
- Run scans with least privilege, restrict network access to the scanner
- Disable unnecessary scan types, enable only necessary feeds
- Regularly update feed data; schedule periodic re-checks

C. Nuclei hardening
- Use scoped templates, avoid broad ultra-wide scans in production-like labs
- Use rate limiting and request delays to avoid overloading internal services
- Validate results to prevent notification fatigue (e.g., suppress duplicates)

D. Network and host hardening
- Apply host-based firewall rules: allow only essential ports (SSH, web UI, scanner communication)
- Ensure SSH hardening: disable password login, enforce key-based auth
- Regular patching cadence: schedule OS and software updates; test in a staging lane first

E. Final validation and wrap-up
- Re-run a small, representative set of scans
- Confirm Wazuh alerts align with actual risks observed in OpenVAS/Nuclei
- Confirm dashboards show accurate, timely data after hardening
- Close with recommended next steps: add asset inventory, automate weekly scans, and tune rules

Closing (20:00-20:30)
- Quick recap of Build, Break, Detect, Harden
- Call to action: comments with lab asset lists, favorite templates, and hardening tips

3-MIN TEASER SCRIPT - Same topic, punchy, hook in first 5 seconds, ends with "Full episode on blaqlabs"

- Hook (0-0:05): "If your homelab can’t prove its own security, you’re building on sand."
- Build quick punch (0:05-1:20): "Today we stack Wazuh, OpenVAS, and Nuclei in a tight loop: deploy, validate, and prove you can detect, break, and harden in one sitting. We’re talking quick lab setup, sane defaults, and real commands you can copy-paste."
- Core value (1:20-2:40): "You’ll see how a misconfig in an agent is caught by Wazuh, how OpenVAS surfaces critical flaws, and how Nuclei provides fast, focused checks—then we harden the stack to resist churn."
- CTA (2:40-3:00): "If you want the full depth, tune in for Ep05. Full episode on blaqlabs"

Note: The teaser emphasizes urgency and concrete outcomes, with a crisp, punchy flow, and ends with the requested line.

LINKEDIN DRAFT - I will post this manually. Short, human, 1 lesson from the episode.

One lesson from Ep05: Integrating Wazuh, OpenVAS, and Nuclei in a homelab turns vulnerability scanning from noisy alerts into actionable risk intelligence. When you centralize findings, correlate agent events, and harden the stack, you move from reacting to threats to proactively reducing your attack surface.

If you’d like, I can format these into a downloadable script outline or provide a starter docker-compose snippet to bootstrap Wazuh/OpenVAS/Nuclei in a containerized lab.