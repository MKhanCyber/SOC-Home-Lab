# SOC Home Lab

A self-built Security Operations Center (SOC) home lab, created to gain hands-on experience with network security, firewall configuration, endpoint monitoring, and threat detection tools. Built as part of the "Building a SOC Home Lab" course by LetsDefend, with my own configuration decisions, troubleshooting, and notes added throughout.

## Why I Built This

I wanted practical, hands-on experience with the tools and architecture used in real SOC environments — not just theory. This lab simulates a small enterprise network (firewall, domain controller, workstation, and monitoring/detection tools) so I could practice configuring, securing, and monitoring it end-to-end.

## Tools & Technologies Used

- **Oracle VirtualBox** — virtualization platform hosting all VMs
- **pfSense (Netgate CE)** — firewall / router, network segmentation
- **Active Directory** — domain services
- **[BadBlood](https://github.com/davidprowe/BadBlood)** — populates AD with realistic (fake) users, groups, and misconfigurations for analysis practice
- **Windows Workstation** — client endpoint
- **Sysmon** — endpoint logging and telemetry
- **CrowdSec** — intrusion detection / threat response

## Network Architecture

**Network layout:**
- **WAN** interface (em0) — DHCP, simulates internet-facing connection
- **LAN** interface (em1) — static IP `192.168.1.1/24`, internal network for domain controller and workstation
- **OPT1** interface (em2) — reserved for future segmentation (e.g. isolated monitoring segment)

## Build Log

### 1. pfSense Installation & Firewall Setup

**What I did:**
- Created a VM in VirtualBox and mounted the pfSense (Netgate CE) installer ISO
- Ran through the guided installer to install pfSense onto a 20GB virtual disk
- Assigned virtual network interfaces: WAN → em0, LAN → em1, OPT1 → em2
- Configured the LAN interface with a static IP:
  - IPv4 address: `192.168.1.1`
  - Subnet: `/24` (255.255.255.0)
  - Gateway: `none` (LAN interfaces don't need an upstream gateway)
  - IPv6: disabled
- Disabled the DHCP server on LAN (to be configured later once client VMs are added)
- Kept the web configurator on **HTTPS** (default, more secure than reverting to HTTP)

**Problems I ran into & how I solved them:**
- **Boot failure on first launch:** VirtualBox reported it couldn't find the ISO file. Cause: the pfSense download was a compressed `.iso.gz` file that hadn't been extracted. Fixed by extracting it with 7-Zip and pointing VirtualBox at the resulting `.iso`.
- **"Installation has failed" error mid-install:** The Netgate installer threw an error and pointed to `/tmp/install-log.txt`. Investigated the log via the installer's shell (`cat /tmp/install-log.txt`) and found the run had actually completed successfully on a later attempt — an earlier failed pass had been logged, but the final log entries confirmed "pfSense Post Installation setup .. done."
- **VM re-booting into the installer instead of the installed system:** After a successful install, the VM kept re-launching the installer on reboot. Cause: the installer ISO was still attached to the virtual optical drive. Fixed by ejecting it via VirtualBox Settings → Storage → removing the disk from the IDE controller before restarting the VM.

### 2. Active Directory

**What I did:**
- Installed Windows Server on a new VM (`SOC_AD`)
- Configured a static IP on the LAN-facing adapter:
  - IPv4 address: `192.168.1.10`
  - Subnet: `255.255.255.0`
  - Gateway: `192.168.1.1` (pfSense's LAN interface)
  - DNS: `127.0.0.1` (points to itself, ahead of promotion to Domain Controller)
- Installed the Active Directory Domain Services (AD DS) role via Server Manager
- Promoted the server to a Domain Controller (new forest, domain `soc.lab`, NetBIOS name `SOC`), verified success with `dcdiag`
- Used [BadBlood](https://github.com/davidprowe/BadBlood) to populate the fresh AD environment with a realistic, messy dataset (thousands of users, groups, computers, OUs, permissions, SPNs, and ASREP-roastable accounts) to use for analysis in later lessons

**Problems I ran into & how I solved them:**
- **VirtualBox ISO mount error again:** Same issue pattern as pfSense — needed to properly mount the Windows Server ISO via Devices → Optical Drives.
- **VM stuck on a black screen after mounting the ISO:** Traced this to the VM having UEFI enabled, which requires a different boot path than legacy BIOS. Fixed by disabling UEFI/Secure Boot in Settings → System → Motherboard, which let the BIOS-based Boot Device Order actually take effect.
- **Wrong ISO downloaded initially:** Grabbed a "LOF_PACKAGES_OEM" (language/optional-features) ISO from the Microsoft Evaluation Center instead of the actual bootable installer. Re-downloaded the correct "ISO downloads → 64-bit edition" file, which resolved the endless "no bootable medium" loop.
- **Ctrl+Alt+Del not reaching the VM:** The host OS always intercepts that key combo. Used VirtualBox's **Input → Keyboard → Insert Ctrl+Alt+Del** menu option instead, which sends it to the guest.
- **Ping to pfSense gateway failing ("destination host unreachable") despite correct IP config:** Root cause was a VirtualBox internal network name mismatch — SOC_AD's adapter was attached to an internal network named `GREEN` while pfSense's LAN interface was on `intnet`. Renamed SOC_AD's adapter to `intnet` to match, then confirmed connectivity with a successful ping after a full power-cycle of both VMs.
- **AD DS promotion silently incomplete:** Left the VM mid-wizard and closed it; on return, `dcdiag` reported the server "is not a Directory Server." Re-ran the promotion wizard from the Server Manager notification flag and completed it in one sitting this time — verified with a clean `dcdiag` pass across all domain/forest tests.
- **No internet access inside the VM to download BadBlood:** SOC_AD is isolated on the internal `intnet` network with no working route out through pfSense yet. Worked around it by downloading BadBlood on the host PC and transferring it in via a VirtualBox **Shared Folder** (Settings → Shared Folders), after drag-and-drop (via Guest Additions) proved unreliable.
- **Guest Additions install crashing VirtualBox:** Installing Guest Additions triggered an application error/crash on reboot. VM recovered fine on relaunch, but Drag and Drop never became available — used the Shared Folder method instead as a reliable alternative.
- **BadBlood script crashing the VM mid-run (twice):** Populating ~2500 users/groups/computers was resource-intensive; the VM powered off abruptly during the run. Increased VM RAM from default to 8GB (host has 16GB) and re-ran the script — it completed successfully on the next attempt (re-running is safe; it skips objects that already exist and continues).
- **PowerShell script appeared frozen mid-run:** Caused by accidentally clicking inside the console window, which triggers PowerShell's QuickEdit Mode and pauses execution. Fixed by clicking back into the window and pressing Enter to resume.

### 3. Windows Workstation

**What I did:**
- Built a Windows 10 installation ISO using Microsoft's official Media Creation Tool (English (United States), Windows 10, 64-bit, saved as an ISO file)
- Created a new VM (`SOC_Win10`) in VirtualBox — 2GB+ RAM, 50GB virtual disk
- Attached the network adapter to the same **Internal Network (`intnet`)** used by pfSense and SOC_AD, so this workstation sits on the same LAN
- Ran through Windows 10 setup, setting all privacy/telemetry options to minimal (location, diagnostics, tailored experiences, advertising ID all disabled)
- Configured a static IP on the LAN-facing adapter:
  - IPv4 address: `192.168.1.20`
  - Subnet: `255.255.255.0`
  - Gateway: `192.168.1.1` (pfSense's LAN interface)
  - DNS: `192.168.1.10` (SOC_AD, acting as the domain's DNS server)
- Confirmed connectivity with a successful ping to pfSense (`192.168.1.1`)
- Joined the workstation (`SOC-WIN10`) to the `soc.lab` domain via System Properties → Computer Name → Change, authenticating with a domain admin account, then restarted and verified membership with `systeminfo | findstr /B /C:"Domain"`, which reports `soc.lab`

**Problems I ran into & how I solved them:**
- **Domain join failing with "The user name or password is incorrect":** The join prompt wants *domain* credentials, not the workstation's local account. Fixed by entering the full domain name `soc.lab` and the domain admin account in `SOC\username` format.
- **Ping failing with "General failure" despite a seemingly correct static IP:** `ipconfig /all` showed the adapter was still on DHCP with a self-assigned APIPA address (`169.254.x.x`), meaning the static IP set through the Settings GUI never actually saved. Switched to setting it directly via command line instead:
  ```
  netsh interface ip set address name="Ethernet" static 192.168.1.20 255.255.255.0 192.168.1.1
  netsh interface ip set dns name="Ethernet" static 192.168.1.10
  ```
- **Duplicate IP conflict:** On the first `netsh` attempt, a typo assigned `192.168.1.10` (SOC_AD's own IP) instead of `192.168.1.20`, which Windows correctly flagged as `(Duplicate)` and refused to fully activate. Re-ran the command with the correct IP, which resolved it — the duplicate detection was actually useful confirmation that the network itself was working correctly.

### 4. Sysmon

**What I did:**
- Downloaded Sysmon (Sysinternals) and SwiftOnSecurity's `sysmonconfig-export.xml` on the host PC, since the VMs have no direct internet route yet
- Transferred both into `SOC_Win10` through a VirtualBox **Shared Folder** (same workaround used for BadBlood on `SOC_AD`), then copied them to a local `C:\Sysmon` folder
- Installed Sysmon from an Administrator PowerShell:
  ```
  cd C:\Sysmon
  .\Sysmon64.exe -accepteula -i
  ```
- Loaded the SwiftOnSecurity ruleset into the running service:
  ```
  .\Sysmon64.exe -c sysmonconfig-export.xml
  ```
- Verified it works: `Sysmon64` and `SysmonDrv` started, and Event Viewer under `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational` is populating with events (74 events logged at first check)

**Problems I ran into & how I solved them:**
- **Shared folder not visible inside the VM:** The folder I added in VirtualBox didn't show up in the guest at first. Shared folders need to be added from the VM's **Devices → Shared Folders** menu (or Settings → Shared Folders) and depend on Guest Additions being available in the guest.
- **`sysmonconfig-export.xml` "is not recognized as the name of a cmdlet":** I typed the config file name on its own, so PowerShell tried to run it as a command. The config is a *parameter* for Sysmon, not a program. PowerShell also doesn't run files from the current folder without the `.\` prefix.
- **Installed Sysmon without the config file:** My first install command left off the config, so Sysmon ran with bare defaults. No need to uninstall — I applied the ruleset to the running service with `Sysmon64.exe -c sysmonconfig-export.xml`.

**Next:** consider installing Sysmon on `SOC_AD` as well for domain controller telemetry.

### 5. CrowdSec

**What I did:**
- Created a free CrowdSec console account (community plan)
- Downloaded the CrowdSec Windows `.msi` on the host PC and moved it into `SOC_Win10` through the VirtualBox Shared Folder, then copied it to a local `C:\CrowdSec` folder
- Installed it from an Administrator PowerShell:
  ```
  cd C:\CrowdSec
  msiexec /i .\<filename>.msi
  ```
- Enabled the Windows Firewall detection collection from `C:\Program Files\CrowdSec`:
  ```
  .\cscli.exe collections install crowdsecurity/windows-firewall
  ```
  (It reported "Nothing to install or remove" because the installer had already enabled it.)
- Reviewed `C:\ProgramData\CrowdSec\config\acquis.yaml`, which tells CrowdSec which logs to read. The default file already covers RDP, Windows Firewall, SQL Server and IIS sources
- Restarted the service and confirmed `Get-Service crowdsec` reports **Running**
- Turned on Windows Firewall logging for dropped connections so CrowdSec has a log to analyze:
  ```
  netsh advfirewall set allprofiles logging droppedconnections enable
  ```
  and confirmed `C:\Windows\System32\LogFiles\Firewall\pfirewall.log` now exists
- Restored `SOC_Win10`'s DNS to the domain controller (`192.168.1.10`) after the install
- Rebooted the VM as the lesson instructs and confirmed both `crowdsec` and `Sysmon64` still report **Running** after the restart
- Installed the **Windows Firewall Bouncer** so CrowdSec can act on its decisions (blocking), not just detect. Downloaded `cs_windows_firewall_bouncer_setup.msi` from the project's GitHub releases page plus the .NET runtime it depends on, moved both in through the Shared Folder, installed the runtime first and then the bouncer, and confirmed the CrowdSec services report **Running**

**Problems I ran into & how I solved them:**
- **"Verify that you have sufficient privileges to install system services":** CrowdSec installs a Windows service, so launching the `.msi` normally (from the shared folder) wasn't enough. Copied it to a local folder and ran `msiexec /i` from an **Administrator** PowerShell.
- **Installer hung, VM froze, and a half-installed service was left in a `Stopped` state:** Traced to DNS. The installer downloads detection rules from the internet, and name lookups were timing out. Diagnosis:
  - `ping 8.8.8.8` succeeded, so the route out through pfSense was fine
  - `nslookup hub-data.crowdsec.net` timed out against `192.168.1.10` (SOC_AD), but `nslookup hub-data.crowdsec.net 8.8.8.8` worked, so only the domain controller's DNS wasn't resolving outside names
  - `SOC_AD` had also been powered off, likely from memory pressure on the host (16GB with all three VMs running)

  Fixed by temporarily pointing `SOC_Win10`'s DNS at `8.8.8.8` (Network Connections → Ethernet → Properties → IPv4), uninstalling the partial install, and reinstalling cleanly. The proper long-term fix is a DNS **forwarder** on the domain controller (DNS Manager → server Properties → Forwarders → `8.8.8.8`).
- **Firewall log file didn't exist:** `Test-Path` on `pfirewall.log` returned `False` because Windows Firewall logging is off by default, so the `windows-firewall` acquisition source had nothing to read. Enabling dropped-connection logging with `netsh advfirewall` created the file.
- **The course's "bundle file" for the bouncer doesn't exist anymore:** The course dates from 2022, and the bouncer's latest release notes say the .NET bundle was removed and only an MSI ships. Installed the .NET runtime separately first, then the bouncer MSI.
- **"This installation package could not be opened":** Caused by a typo in my `msiexec` command (no space between `/i` and the file path). Re-ran it as `msiexec /i .\cs_windows_firewall_bouncer_setup.msi` and the installer opened normally.
- **`Restart-Service crowdsec` left the service stopped after editing `acquis.yaml`:** Ran `.\crowdsec.exe -t` (configuration test), which reported `non-map value is specified` for `acquis.yaml`. `Get-Content` on the file showed an extra firewall block I had appended in the wrong format (the path wrapped onto its own un-indented line), and it duplicated an entry that already existed in the default config. Removed the extra block, re-tested, and the service started.

**Next (optional):** link the agent to my CrowdSec console account, add a DNS forwarder on `SOC_AD`, run a simulated attack to generate real alerts for this README, and consider installing Sysmon and CrowdSec on `SOC_AD` as well.

## What I Learned

- How to install and perform initial configuration of pfSense as a virtual firewall/router
- Troubleshooting common VirtualBox/ISO mounting and boot-order issues
- Reading and interpreting installer logs to diagnose failed installs
- Basic network interface assignment and static IP configuration on a firewall appliance
- Diagnosing network connectivity issues by distinguishing error types (e.g. "General failure" vs. "destination host unreachable") to identify whether the problem is at the adapter, IP configuration, or routing level
- Setting static IP configuration reliably via command line (`netsh`) when GUI settings fail to persist
- Installing and verifying Sysmon with the SwiftOnSecurity configuration, and checking its events in Event Viewer
- Separating "no internet" from "no DNS" (ping an IP address, then `nslookup` a hostname, then `nslookup` against a known DNS server) to isolate a domain controller that wasn't forwarding outside lookups
- Validating a service's configuration (`crowdsec.exe -t`) and reading the file it complains about, instead of guessing, when a service won't start
- Setting up CrowdSec end to end on Windows: agent, detection collections, log acquisition (`acquis.yaml`), enabling Windows Firewall logging as a log source, and the firewall bouncer for blocking
- Reading release notes when a course is out of date (the bouncer's "bundle" had been replaced by an MSI plus a separate .NET runtime)

## About

Course followed: *Building a SOC Home Lab* by LetsDefend (prepared by Julien Garcia).
Lab built and documented independently as a personal learning/portfolio project.
