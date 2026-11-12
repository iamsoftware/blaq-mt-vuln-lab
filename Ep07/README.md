20-MIN YOUTUBE SCRIPT - Episode Ep07: Active Directory Attack and Detection Lab
Topic: Active Directory Attack and Detection Lab
Tone: Senior engineer, no fluff, direct commands, lab-focused

Introduction (0:00–0:45)
- Host: “Welcome back to BlaqLabs. This is Ep07: Active Directory Attack and Detection Lab. Today we build a controlled AD lab, execute a compact attack chain, observe the telemetry, and harden the environment. All steps are designed for a homelab, not production. You should be running on isolated VMs or a private subnet. Do not scan or exploit real AD domains without explicit authorization.”

Section 1: Build – Lab Setup (0:45–6:00)
- Build goal: Stand up a compact, observable AD testbed with domain controller, a workstation, and a monitoring stack.
- Prereqs (host): 64-bit Windows Server 2019/2022 (DC), Windows 10/11 VM (client), separate management VM for SIEM/EDR, isolated network.
- Network topology: 1x Domain Controller (DC), 1x Internal Client, 1x Defender/EDR sensor, 1x Sysmon + Windows Event Forwarding collector.
- Commands and configuration notes:
  - Prepare hosts:
    - Create isolated NAT/host-only network in your hypervisor.
  - DC setup (PowerShell on DC or via GUI, use a clean install for clarity):
    - Install-WindowsFeature -Name AD-Domain-Services
    - Import-Module ServerManager
    - Rename-Computer -NewName DC01 -Restart
    - Install-ADDSForest `
      -DomainName "LAB.LOCAL" `
      -ForestMode "6" `
      -DomainMode "6" `
      -DomainNetbiosName "LAB" `
      -SafeModeAdministratorAccount "TestAdmin" `
      -DNSDelegation:$false
  - Post-DC hardening (minimal for lab): 
    - Set-LocalUser -Name "TestAdmin" -Password (ConvertTo-SecureString -AsPlainText "P@ssw0rd!" -Force -AsPlainText) -PasswordNeverExpires $true
    - New-ADUser LabUser -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) -Name "Lab User" -Path "CN=Users,DC=LAB,DC=LOCAL" -Enabled $true
  - Client VM:
    - Join-Domain LAB.LOCAL
    - gpupdate /force
  - Monitoring stack:
    - Enable Windows Event Forwarding (WEF) to a collector:
      - On collector: winrm quickconfig
      - On DC/clients: wecutil qc /q
      - Subscription XML (example):
        - <Subscription ...> to collect Security.evtx, System.evtx
    - Deploy Sysmon (from Sysinternals):
      - Sysmon64.exe -i sysmon.xml
      - sysmon.xml should log Process_Create, FileCreate, NetworkConnect, DriverLoad
    - Install a lightweight SIEM/EDR: Windows Defender for endpoint, or a free EDR if available (Falcon Sandbox, etc.)
- Build checklist:
  - DC is reachable: ping LAB-DC.LAB.LOCAL
  - DNS works: nslookup LAB.LOCAL
  - Event forwarding test: wecutil qc, confirm collector receives events
- Validation commands:
  - Get-ADDomainController -Filter * | Select Name,OperatingSystem
  - Test-ComputerSecureChannel -Verbose
  - Get-WinEvent -LogName Security -MaxEvents 10
  - Get-WinEvent -LogName Microsoft-Windows-Sysmon/Operational -MaxEvents 20

Section 2: Break – Controlled Attack Sequence (6:00–12:30)
- Objective: Demonstrate a contained attacker foothold and lateral movement in a lab, with visibility.
- Assumptions: Attacker has valid but limited credentials; Defender has basic telemetry enabled (Sysmon, Windows Security logs, WEF).
- Step A: Initial foothold (Credential access in lab)
  - Attacker VM:
    - Use in-domain user to access a workstation via RDP (simulate/observe)
    - Create a non-admin user for lab (to simulate low-privilege creds)
    - Command examples (on attacker console, not executing on victim systems without consent in your lab):
      - echo "simulate foothold" > foothold.txt
      - # No real exploitation here; we simulate with legitimate actions
- Step B: Privilege escalation (simulate in lab)
  - Defender notes: watch for unusual SVC changes, process hollowing, or new local user creation
  - Defender telemetry to monitor:
    - Process_Create events with suspicious parents (e.g., cmd.exe spawning PowerShell with -NoProfile)
    - Network connections to DC or LDAP ports (389, 636, 3268)
  - Attack steps (simulated safely):
    - Create a scheduled task to mimic persistence:
      - schtasks /Create /SC ONLOGON /TN "LAB_Svc" /TR "C:\Windows\System32\cmd.exe /c echo Hello" /RU LABUser
- Step C: Lateral movement (simulated)
  - Attacker uses valid credentials to access a second host via PSRemoting limited to domain group
  - Commands (lab-only):
    - Enter-PSSession -ComputerName Client01 -Credential (Get-Credential)
- Step D: Data enumeration (defender observe)
  - Attacker enumerates groups and AD structure:
    - Run on attacker or lab sim:
      - powershell -Command "Get-ADUser -Filter * -Properties * | Select Name, DistinguishedName"
      - powershell -Command "Get-ADGroup -Filter * -Properties Members | Select Name, DistinguishedName, Members"
  - Defender telemetry:
    - Look for LDAP query patterns in Security log (4624 Logon, 4625 fail, 4769 Kerberos service ticket)
- Step E: Temporary cleanup
  - Remove test users, scheduled tasks, and revert VMs to snapshot
- Observables for audience:
  - Indicators: unusual process trees, unsanctioned LDAP calls, credential harvesting signals, unknown services
- Remediation hints (for breaking the attack in real-time):
  - Enforce MFA, constrain LSA secrets, monitor for unusual service creation, disable legacy protocols (NTLM)
  - Tighten Kerberos with constrained delegation only where necessary
- Hardened state to demonstrate:
  - Enable Defender for Identity/MDATP features if available
  - Apply LAPS for local admin passwords
  - Centralize logs to WEF collector with strict filtering

Section 3: Detect – Telemetry and Detections (12:30–17:30)
- Telemetry sources:
  - Windows Security.evtx: Event IDs 4624, 4625, 4769, 4719
  - Sysmon: Event IDs 1 (Process creation), 3 (File creation), 7 (Network connect), 11 (Process thread creation), 12 (FileCreate)
  - PowerShell logs: Module and Script Block logging (Microsoft-Windows-PowerShell/Operational)
  - DNS logs and Kerberos event logs for unusual patterns
- Detection patterns (practical):
  - Parent-child process anomalies: cmd.exe spawned by LSASS or wmic.exe with PowerShell
  - Unusual user creation and membership changes in AD
  - New scheduled tasks or services running under non-standard accounts
  - Large LDAP query bursts: Get-ADUser/Get-ADGroup calls from a single host
  - Non-interactive remote sessions (PSRemoting) from user accounts outside normal admin teams
- Demonstration commands to verify telemetry:
  - Get-WinEvent -FilterHashtable @{LogName="Security";Id=4624} -MaxEvents 50
  - Get-WinEvent -FilterHashtable @{LogName="Microsoft-Windows-Sysmon/Operational";Id=1} -MaxEvents 50
  - Get-WinEvent -FilterHashtable @{LogName="System";Id=12} -MaxEvents 50
- Real-time hooks:
  - Show a live Windows Event Forwarding collector tail (if supported):
    - Get-WinEvent -LogName "Security" -MaxEvents 5 -Oldest
  - Demonstrate alert generation in your SIEM/EDR dashboard for suspicious AD activity
- Quick wins:
  - Enable audit policy for AD changes: Audit Directory Service Changes (Success/Failure)
  - Ensure Sysmon config includes process creation and parent process lineage
  - Consolidate logs from all VMs to a central collector and review within 15 minutes of any test activity

Section 4: Harden – Defensive Measures and Lessons (17:30–20:00)
- Immediate hardening steps you can demonstrate live:
  - Enable MFA for all domain-joined devices; enforce via GPO or Intune
  - Deploy and tune Windows Defender ATP/MDATP or your preferred EDR
  - Implement LAPS for local admin passwords and regularly rotate
  - Disable NTLM where not necessary and restrict NTLM traffic
  - Use Group Policy to scope PSRemoting and restrict user access
  - Implement Just-In-Time (JIT) access for admin tasks
  - Enable Credential Guard / Device Guard if supported
- Lab wrap-up recap:
  - What was built: isolated AD testbed with DC, client, monitoring stack
  - What was observed: telemetry signals aligned to the detection rules
  - What to improve: broader telemetry coverage, tighter AD auditing, more granular alerts
- Call to action:
  - Pause, snapshot, and slide into defenses before next episode

Closing (20:00–21:00)
- Host: “That’s Ep07: Active Directory Attack and Detection Lab. Build, break, detect, and Harden in a controlled homelab. If you want a deeper dive into any particular detection rule or hardening technique, drop a comment. See you next episode.”

3-MIN TEASER SCRIPT
- Hook (0–5 seconds): “Ever seen an AD attack unfold in real-time inside a safe homelab? Watch how exposure happens, how to spot it instantly, and how to shut it down—live in Ep07.”
- Quick hook beats (5–25 seconds): “We build a tiny AD lab, simulate a foothold, watch the telemetry scream, then lock it down with concrete hardening steps. No fluff, just real commands and real results.”
- Core tease (25–70 seconds): “From DC setup to Sysmon-logged Twitchy processes, you’ll see process trees, LDAP patterns, and how a defender stamps out attackers with just-in-time access and auditing. All practical, all in your home lab.”
- Close (70–100 seconds): “Full episode on blaqlabs.”

LinkedIn Draft (1 lesson from Episode)
- Post text:
  - Lesson: In a controlled Active Directory lab, visibility beats privilege. If you can’t see what AD users and machines are doing, you can’t stop an attacker quickly. Centralize logs (Sysmon, Windows Security, WEF), audit AD changes, and enable just-in-time access. The payoff is faster detection and safer hardening.
- Tone: human, concise, actionable
- Call to action: If you’re exploring AD security in a homelab, check out Ep07 on BlaqLabs for the hands-on setup, detections, and defenses.

Notes
- These scripts are designed for a dedicated homelab environment. Do not attempt to attack real networks without explicit authorization.
- Adapt commands to your exact lab names, domain, and security policy.
- If you’d like, I can tailor the exact PowerShell or XML for your Sysmon config and WEFS subscription.