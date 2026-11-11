20-MIN YOUTUBE SCRIPT - Ep03: Honeypot Lab (Build -> Break -> Detect -> Harden)

Intro (0:00–0:40)
- Quick drop-in: “Ep03: Honeypot Lab. We’ll build a controlled decoy environment, push safe traffic to it, observe attacker behavior, and harden the surface. No fluff. Senior infra mindset, hands-on commands, production-grade pragmatism.”
- Agenda slide: Build a honeypot lab, perform safe break/tamper tests, detect activity, and harden the setup.
- Safety note: All testing in a sandbox, isolated from real networks. Use logging, backups, and proper containment.

1) Build (0:40–7:30)
Goal: A contained, observable honeypot stack you can trust in a homelab.

Prereqs:
- Host: Linux (Ubuntu 22.04+ or Debian-based)
- Docker + docker-compose
- Network: Isolated VLAN or host-only adapter
- Time sync: NTP

Steps (with commands):
- Prepare host
  - sudo apt update && sudo apt upgrade -y
  - sudo apt install -y docker.io docker-compose git nftables chrony
  - sudo systemctl enable --now docker
  - sudo usermod -aG docker $USER (log out/in)
- Create isolated network (optional)
  - sudo mkdir -p /opt/honeypot && cd /opt/honeypot
  - Create docker-compose.yml for Cowrie (or alternative)
- Example docker-compose.yml (Cowrie-based)
  - version: "3.8"
    services:
      cowrie:
        image: cowrie/cowrie:latest
        container_name: cowrie-honeypot
        networks:
          honeynet:
            ipv4_address: 172.28.0.10
        ports:
          - "2222:2222"   # SSH honeypot port
          - "2223:2223"   # Telnet (if desired; consider disabling by default)
        environment:
          - COWRIE_LOG_LEVEL=INFO
        cap_add:
          - SYS_ADMIN
        restart: unless-stopped
        volumes:
          - ./cowrie/data:/cowrie/var/lib/cowrie  # persistent logs
  networks:
    honeynet:
      driver: bridge
      ipam:
        config:
          - subnet: 172.28.0.0/16
- Initialize and run
  - docker-compose up -d
  - sleep 5
  - docker ps
- Basic monitoring setup
  - sudo journalctl -u docker -f
  - docker logs -f cowrie-honeypot
  - Tail Cowrie log paths (inside container): docker exec -it cowrie-honeypot sh -lc "tail -f /cowrie/var/log/cowrie/cowrie.log"
- Optional: capture network flow for context
  - sudo apt install -y tcpdump
  - sudo tcpdump -i docker0 -n -s0 -w /opt/honeypot/pcap/cowrie.pcap host 172.28.0.10
  - Note: Filter in your SIEM by source/dest to keep signal clean

Architecture notes:
- Network isolation: Use a dedicated bridge (honeynet) and no direct access to your operational network.
- Logging: Centralize to a log folder with rotation.
- Persistence: Mount a volume for data retention.

2) Break (7:30–12:30)
Goal: Observe attacker behavior safely and within scope.

What attackers commonly do against honeypots:
- Credential stuffing, brute force on exposed services (SSH/Telnet)
- Lateral movement attempts if misconfigured
- Payload fetches, CLI commands, beaconing

Safe, controlled test plan (no real exploitation outside lab):
- Use a test attacker VM in the same lab VLAN or a controlled host to generate safe, simulated traffic.
- Examples of safe probes:
  - Nmap/Zenmap scan against the honeypot to verify visible services.
  - Hydra/Patator scans for weak credentials only against locally exposed ports you intend to test (limit rate, target 1–2 known accounts you’ve set up for the test).
  - Generate lightweight SSH login attempts using a known, non-root test account you created for the lab.
- Commands (conduct in a controlled test VM, not on production):
  - nmap -sS -p 22,2222 172.28.0.10
  - hydra -l honeypot_user -P /usr/share/wordlists/rockyou.txt ssh://172.28.0.10 -t 4 -f
  - echo "test" | nc 172.28.0.10 2222
- Observability checks (run in parallel with tests)
  - docker logs -f cowrie-honeypot
  - tail -f /opt/honeypot/cowrie/data/ Honeypot log location depends on Cowrie version
  - grep -R "login" /opt/honeypot/cowrie/var/log -n
- Expected signals to record (in your notes):
  - Unusual login prompts, repeated failed attempts, unexpected user agents in network capture
  - Cowrie's shell prompts, command lines that attackers try, and downloaded payloads
- Safe containment steps if traffic escalates:
  - Block offending IPs at the firewall
  - Pause the honeypot container and take a snapshot of logs
  - Ensure no sensitive data is being exfiltrated

3) Detect (12:30–17:30)
Goal: Turn noise into signal. Build a quick observability baseline.

What to monitor:
- Cowrie logs: authentication attempts, commands executed, TTY sessions
- Network flows: unusual port usage, beaconing patterns
- System-level: file writes, new processes, sudo usage within the container

Practical commands:
- Real-time log watch
  - docker exec -it cowrie-honeypot sh -lc "tail -f /cowrie/var/log/cowrie/cowrie.log"
  - docker exec -it cowrie-honeypot sh -lc "tail -f /cowrie/var/log/cowrie/term.log"
- Simple alert baselines (threshold-based)
  - If you have a local syslog/ELK stack, set an alert: > more_than(5) failed_login_within_10min
- Lightweight sniffing
  - sudo tcpdump -i eth0 -n icmp or -i eth0 -n tcp and port 2222
  - Save as /opt/honeypot/pcap/alerts.pcap
- Basic threat-hunting timeline
  - Record time, IP, port, protocol, and what command was attempted (as seen in logs)
  - Correlate multiple failed attempts from a single source
- Quick hardening pointer during detection
  - If you see a suspicious login sequence, stop and isolate the source IP at the firewall
  - Review Cowrie's shell interactions to see if attacker attempted to fetch payloads

4) Harden (17:30–20:00)
Goal: Reduce risk, improve detection fidelity, and keep the honeypot informative without compromising your network.

Hardening steps:
- Network containment
  - Ensure there is no egress path from the honeypot to sensitive systems
  - Use an explicit egress rule: block all outbound except to your logging/monitoring server
- Access controls
  - Disable root login on the honeypot (in Cowrie config or the host for the vuln-sounding service)
  - Use non-privileged accounts for test logins; ensure password policies and account lockouts
- Logging and data handling
  - Centralize logs to a secure collector; enable log rotation and backups
  - Regularly export/export logs to a secure storage location
- Service hardening
  - If you must expose SSH, disable password authentication and use key-based auth only for legitimate admin access (not the honeypot)
  - Remove extraneous services from the honeypot container
- Detection improvements
  - Add a simple SIEM rule to flag repeated login failures from a single IP within a short window
  - Add anomaly detection for unique commands or payload fetch patterns
- Recovery and hygiene
  - Routine backups of Cowrie data
  - Rebuild containers periodically to minimize drift
  - Have a kill-switch to quickly tear down the honeypot environment if abuse escalates

Closing (20:00)
- Recap: We built a modular, observable honeypot lab, exercised safe attacker traffic, captured critical signals, and hardened the setup for ongoing use. All commands shown are designed for a homelab, with containment and safety in mind.
- Next episode tease: deeper dive into data extraction from honeypot logs and turning signals into actionable mitigations.

3-MIN TEASER SCRIPT

Hook (0:00–0:05)
- “If you want to see the exact mistakes attackers make in a real decoy, this is the lab you need.”

Speed run (0:05–2:40)
- “We build a contained Cowrie honeypot in a homelab, watch safe attacker traffic, and pull the signals that actually matter—without risking your network. Build, test, detect, and harden in one tight episode.”
- Visuals: quick cuts of docker-compose file, a grep of logs, a tcpdump snippet, and a firewall rule.

Close (2:40–3:00)
- “Full episode on blaqlabs”

LINKEDIN DRAFT (one lesson, human tone)

Ep03: Honeypot Lab in the homelab
One lesson: Isolation is not just a network setting — it’s the first line of defense for observable security. When you split the decoy from real assets, you unlock safe, repeatable threat signals you can actually act on.

If you’d like adjustments to tone or depth for any section, I can tailor the commands or details.