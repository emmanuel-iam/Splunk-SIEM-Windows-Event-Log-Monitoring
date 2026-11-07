# Splunk SIEM & Log Analysis

**Splunk Free · Azure VM · SOC Skills · Security Monitoring**

| Field | Value |
|---|---|
| Certification alignment | CompTIA Security+ · CySA+ · Splunk Core Certified User |
| Free tools | Splunk Enterprise — 60-day trial then permanently free with 500MB/day |
| Time to complete | 4–6 hours across multiple sessions |
| Estimated cost | $0 — Splunk Free licence covers everything in this lab |
| Career relevance | SOC Analyst (Tier 1–3) · Security Engineer · Incident Responder |

## Architecture — How Splunk receives and processes logs

The diagram below shows the complete data flow for this lab. Your Windows Server VM from Lab 1 forwards logs to a separate Ubuntu VM running Splunk. Splunk indexes those logs and makes them searchable through the web UI.

## The Business Problem This Lab Solves

A medium-sized organisation generates millions of log events every day — Windows Event Logs from workstations, authentication logs from Active Directory, firewall logs from network equipment, web server access logs, cloud resource logs. Without a SIEM, those logs sit in separate systems and nobody can search across them, correlate events, or identify patterns that indicate an attack.

The SIEM is the security operations centre's primary tool. When an alert fires, the SOC analyst opens the SIEM and searches the logs to understand what happened, when, from where, and what was affected. When a security team wants to know whether a particular threat has touched their environment, they search the SIEM.

Splunk is the most widely deployed commercial SIEM. Learning it in a free lab environment gives you a concrete, demonstrable skill that appears on job descriptions for almost every security operations role.

| Role | How this lab applies |
|---|---|
| SOC Analyst Tier 1 | Monitoring dashboards for alerts, searching logs for suspicious activity, escalating findings |
| SOC Analyst Tier 2–3 | Building detection rules, correlating events across data sources, threat hunting |
| Cloud Security Engineer | Microsoft Sentinel and AWS Security Hub use the same SIEM concepts — this lab teaches the mental model |
| Incident Responder | Searching logs during an active incident, building a timeline of events, identifying scope of compromise |

## Key Concepts — Read This Before Starting

### What is a SIEM?

SIEM stands for Security Information and Event Management. A SIEM is a platform that collects log data from across your entire environment — servers, workstations, firewalls, cloud services, applications — and makes it all searchable in one place. The two core jobs of a SIEM are: (1) correlation — connecting events across different systems to identify patterns that no single system would reveal alone, and (2) alerting — automatically notifying analysts when suspicious conditions are met. Without a SIEM, a security team would need to log into each system individually and search its logs manually. With a SIEM, one analyst can search across thousands of systems simultaneously.

### What is SPL — Splunk Processing Language?

SPL is the query language you use to ask Splunk questions. It works as a pipeline — you start with a search, then pipe the results through commands that filter, transform, and visualise the data. For example: index=windows_logs EventCode=4624 | stats count by Account_Name | sort -count. This search finds all successful login events, counts them grouped by account name, and sorts highest to lowest. Every SPL search follows the same pattern: find the events, then shape the results. You do not need to know SPL before starting this lab. Each search in the steps below is explained line by line.

### What is a Splunk index?

An index in Splunk is like a database table — it is a named storage bucket where events are kept. When logs arrive at Splunk, they are stored in an index. When you search, you specify which index to search with index=name. You can have multiple indexes for different data sources: one for Windows logs, one for firewall logs, one for web server logs. Keeping data in separate indexes lets you control retention, permissions, and storage separately for each data source. In this lab you will create one index called windows_logs.

### What is the Universal Forwarder?

The Universal Forwarder is a lightweight agent that Splunk provides for free. You install it on any machine whose logs you want to send to Splunk. It monitors log files and Windows Event Logs, compresses and encrypts the data, and forwards it to your Splunk indexer over port 9997. The forwarder uses very little CPU and RAM — it is designed to run invisibly in the background on production servers. In this lab you install it on your Windows Server VM from Lab 1 so that Active Directory security events flow into your Splunk instance automatically.

### What is Windows Event IDs?

Windows records everything that happens on a system as numbered events in the Windows Event Log. Each type of event has a unique ID number. The three IDs you will use constantly in security work are: EventCode 4624 — a user successfully logged on. EventCode 4688 — a new process was created. Used in threat hunting to identify malicious execution. EventCode 4624 — successful logons, including logon type which reveals interactive vs remote vs service logins. EventCode 4688 — new process created. These two event codes together cover authentication and execution — the two areas attackers touch most. Additional event codes are covered in the Appendix at the end of this lab.

### What is inputs.conf?

inputs.conf is a configuration file that tells the Splunk Universal Forwarder what data to collect. It lives on the machine where the forwarder is installed — in this lab that is your Windows Server VM. Each section in the file, marked with square brackets, defines one data source. [WinEventLog://Security] tells the forwarder to collect Windows Security events. The settings underneath control how: disabled=0 means enabled, start_from=oldest means collect historical events not just new ones. You edit this file once during setup and the forwarder reads it automatically. You do not need to restart the forwarder every time you add a new log source — just edit the file and restart once.

## What You Will Learn

| Skill | Real-world application |
|---|---|
| Deploy Splunk and configure a data input | Every Splunk deployment starts with getting data in — the Universal Forwarder is how most enterprise environments feed logs to Splunk |
| Navigate the Splunk interface | Search, dashboards, alerts, reports — understanding the layout is table stakes for any SOC role |
| Write SPL searches | SPL is how you ask Splunk questions. It is the skill that separates analysts who find threats from analysts who stare at dashboards |
| Build security dashboards | Visualising security data at a glance — login failures over time, top source IPs, failed authentication by user |
| Identify failed login attempts | One of the most common security investigations. Distinguishing normal user error from a brute force attack |
| Build an automated alert | Splunk fires an alert when conditions you define are met, rather than waiting for a human to notice |
| Search for account lockout events | A trail of lockout events can indicate a password spray attack in progress |

## Step 1 — Get Splunk Free

Splunk Enterprise is free to download. You get a 60-day full trial, after which it automatically converts to the free licence. The free licence limits you to 500MB per day of data indexing — more than enough for a home lab.

When you go to the Splunk website you will see several products listed. You want Splunk Enterprise — not Splunk Cloud (that is a hosted SaaS product) and not Splunk SOAR. Look for the download labelled Splunk Enterprise and select the version for your operating system. To download, Splunk requires you to create a free account and fill in a registration form. You do not need to use your real email address or personal details for this — use a temporary email address and dummy information instead (see the note below).

- Go to splunk.com/en_us/download/splunk-enterprise.html
- Create a free Splunk account using a temporary email address — see the note below for exactly how to do this
- Download Splunk Enterprise for Linux (the Ubuntu VM you will create below)
- Follow the VM setup and install instructions in the next section

### Use a temporary email address for your Splunk account

Splunk requires you to fill in a registration form before you can download. You do not need to use your real personal details. Use a free temporary email service and fill in the rest of the form with dummy information.

**Step 1 — Get a temporary email address:**

Go to temp-mail.org/en/ — a temporary email address is generated automatically when the page loads. No sign-up required. Copy the email address shown on screen.

**Step 2 — Fill in the Splunk registration form:**

Paste the temp-mail address into the Email field. Fill in the remaining fields (First Name, Last Name, Company, Job Title, Phone) with any dummy information — for example: John Smith, Home Lab, Student, 555-0100. These fields are not verified.

**Step 3 — Confirm your account:**

Splunk will send a confirmation email. Go back to the temp-mail.org tab — the email will appear in the inbox automatically within a minute. Click the confirmation link inside it. You can now log in and access the download page.

### Deploy on an Azure VM (Recommended)

Running Splunk on an Azure VM means it stays running when your laptop is closed, you can access it from anywhere, and you can send logs directly from your Active Directory VM (Lab 1) to the same Splunk instance over the internal Azure network.

| VM Setting | Value |
|---|---|
| OS | Ubuntu 22.04 LTS (free tier eligible) |
| Size | Standard_B2s (2 vCPU, 4GB RAM minimum — Splunk requires at least 4GB) |
| Disk | 30GB minimum |
| Inbound NSG ports | 8000 (Splunk web UI) · 9997 (forwarder input) · 22 (SSH) |

Open port 8000 and port 22 to your IP address only. Open port 9997 to the VNet address range only (10.0.0.0/16 or similar) so only your other Azure VMs can send logs to it — not the public internet.

### How to SSH into your Linux VM

SSH (Secure Shell) is how you connect to and control a Linux VM remotely using a terminal. The tool you use to SSH depends on your operating system.

#### Mac or Linux — no extra tools needed

SSH is built into macOS and Linux. Open Terminal and run:

**Step 1 — Fix the key file permissions first (required or SSH will refuse to connect)**

When you download a .pem key file from Azure or AWS, it arrives with permissions that are too open — other users on your machine can read it. SSH will refuse to use the key until you lock it down. If you skip this step you will see the UNPROTECTED PRIVATE KEY FILE warning and the connection will be denied.

```bash
# Navigate to where you downloaded the key (usually Downloads)
cd ~/Downloads
# Set permissions so only you can read the key file
chmod 400 yourkey.pem
```

What happens if you skip this step:

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for 'yourkey.pem' are too open.
Load key "yourkey.pem": bad permissions
azureuser@20.55.24.67: Permission denied (publickey).
```

**Step 2 — Connect once permissions are fixed**

```bash
ssh -i yourkey.pem azureuser@YOUR_VM_PUBLIC_IP
```

Replace yourusername with the admin username you set when creating the VM in Azure, and YOUR_VM_PUBLIC_IP with the public IP address shown in the Azure portal. Type yes when prompted about the host fingerprint, then enter your password.

#### Windows — install PuTTY first

Windows does not have a built-in SSH client that is easy to use for this purpose. PuTTY is the most widely used free SSH client for Windows. Download and install it before proceeding.

**Installing PuTTY:**

- Go to putty.org and click Download PuTTY
- Under Package files, download the 64-bit Windows installer (.msi file)
- Run the installer as Administrator. The wizard has four screens — follow each one exactly:
- Screen 1 — Default Installation Options: Check the licence box. Select An on-premises Splunk Enterprise instance. Do NOT select A Splunk Cloud instance — that option skips the Receiving Indexer screen so the forwarder has nowhere to send data. Click Next
- Screen 2 — Administrator credentials: Username admin, uncheck Generate random password, set a password and write it down — you need it for CLI commands later. Click Next
- Screen 3 — Deployment Server: Leave completely blank. Do not enter your Splunk VM IP here. Click Next
- Screen 4 — Receiving Indexer: Enter your Splunk VM’s private IP (the 10.x.x.x from the Azure portal) and port 9997. Click Next
- Final screen: Click Install. Takes about 30 seconds. Click Finish when complete
- Open PuTTY from the Start menu when installation is complete

**Connecting with PuTTY:**

- Open PuTTY — the configuration window opens automatically
- In the Host Name (or IP address) field, enter your VM’s public IP address from the Azure portal
- Make sure Port is set to 22 and Connection type is SSH (both are the defaults)
- Click Open — a security alert about the server’s host key will appear the first time. Click Accept.
- A terminal window opens. Type your VM admin username and press Enter, then type your password and press Enter
- You are now connected. The commands in the code block below run inside this PuTTY terminal window

Note: when you type your password in PuTTY, nothing appears on screen — no dots, no asterisks. This is normal Linux behaviour. Type your password and press Enter even though the cursor does not move.

Once you are connected to the VM via SSH, run each command below in order. Each command is explained before you run it so you understand exactly what it does and why.

### Command 1 — Download the Splunk installer

wget is a command-line download tool built into Linux. The -O flag sets the output filename so the downloaded file is saved with a clean name on your VM. The URL in quotes is the direct download link from Splunk. This command downloads the Splunk .deb installer package from Splunk’s servers and saves it in your current directory.

```bash
wget -O splunk-10.2.2-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/10.2.2/linux/splunk-10.2.2-80b90d638de6-linux-amd64.deb"
# NOTE: if this returns 404, Splunk has released a newer version.
# Log into splunk.com, go to Free Trials and Downloads, select Linux .deb,
# and copy the wget command shown on the download page for the current version.
```

### Command 2 — Install the package

dpkg is the Debian/Ubuntu package manager. The -i flag means install. sudo runs the command as root (administrator) because installing software system-wide requires elevated permissions. This command unpacks the .deb file and installs Splunk into /opt/splunk/ on your VM. You will see output showing each installation step. A warning about a missing Python 3.7 path is expected and harmless on Ubuntu 22.04 — Splunk 10.x ships with its own Python.

```bash
sudo dpkg -i splunk-10.2.2-linux-amd64.deb
```

### Command 3 — Start Splunk and accept the licence

This command starts the Splunk service for the first time. The --accept-license flag automatically accepts the Splunk licence agreement so you are not prompted interactively. The --run-as-root flag is required on newer versions of Splunk when running as the root user — without it the start command will fail with a deprecation error. When this runs you will be prompted to create an admin username and password. These are the credentials you will use to log into the Splunk web UI.

```bash
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
```

### Command 4 — Enable Splunk to start on boot

This command registers Splunk as a system service so it starts automatically every time the VM reboots. Without this, you would need to manually SSH into the VM and run the start command every time the VM restarts. After running this, Splunk will always be running when the VM is on — you never need to start it manually again.

```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

### Accessing the web UI

Once the start command completes, Splunk is running. Open a browser on your local machine and navigate to the address below. Replace YOUR_VM_PUBLIC_IP with the public IP address shown on your VM’s Overview page in the Azure portal.

```text
# Open this in your browser on your local machine:
http://<YOUR_VM_PUBLIC_IP>:8000
```

### If the browser shows “This site can’t be reached”

A connection timeout when trying to reach port 8000 is almost always an NSG rule missing in Azure. Work through these checks in order before anything else.

**Check 1 — Add the port 8000 NSG rule in Azure (most common cause)**

Go to portal.azure.com → your Splunk VM → Networking → Inbound port rules

Check whether a rule allowing port 8000 exists. If it does not, click Add inbound port rule and set:Name = Allow-SplunkWebUI-MyIP  •  Source = My IP address  •  Destination port = 8000  •  Protocol = TCP  •  Action = Allow  •  Priority = 310The name tells anyone reading the NSG rules exactly what this rule does and why it exists.

Save the rule and try the browser again — this fixes the issue in the majority of cases

**Check 2 — Confirm Splunk is running**

SSH into your VM and run:

```bash
sudo /opt/splunk/bin/splunk status
# Expected output: splunkd is running
```

If it says stopped, run the start command again:

```bash
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
```

**Check 3 — Confirm Splunk is listening on port 8000**

```bash
sudo ss -tlnp | grep 8000
# Expected: a line showing 0.0.0.0:8000 with splunkd in the process column
```

If this returns nothing, Splunk is not running. If it shows the line, Splunk is up and the issue is the NSG rule from Check 1.

**Check 4 — Confirm you have the correct public IP**

Go to portal.azure.com → your VM → Overview. The Public IP address shown there is what goes in the browser. Azure reassigns public IPs when a VM is stopped and restarted — if you stopped the VM since the last session, the IP will have changed. Always get the current IP from the portal rather than using one you wrote down previously.

**Check 5 — Add port 9997 inbound rule to the Splunk VM NSG**

The Universal Forwarder on your Windows VM sends logs to Splunk on port 9997. If your VMs are in different VNets or subnets this traffic is blocked by default. Add a rule to the Splunk VM’s NSG to allow it.

Find the private IP of your Windows Server VM from its Overview page in the portal (it looks like 10.x.x.x)

Go to portal.azure.com → your Splunk Ubuntu VM → Networking → Inbound port rules → Add inbound port rule and set:

Name = Allow-SplunkForwarder-WindowsVM • Source = IP Addresses • Source IP = [Windows VM private IP] • Destination port = 9997 • Protocol = TCP • Action = Allow • Priority = 320

**Check 6 — Set up VNet Peering if your VMs are in different VNets**

Even with NSG rules in place, two VMs in separate VNets cannot communicate unless VNet Peering is configured. Peering creates a direct private connection between the two VNets. Without it, port 9997 traffic from the forwarder never reaches the Splunk indexer regardless of any rules you set.

Go to portal.azure.com → search Virtual Networks → click your Splunk VM’s VNet

Click Peerings in the left menu → Add

Set the peering link names so they are self-describing: This VNet peering link name = splunk-vnet-to-ad-vnet • Remote VNet peering link name = ad-vnet-to-splunk-vnet

Under Remote virtual network select your Windows Server VM’s VNet → leave all other settings as default → click Add

Azure creates the peering in both directions automatically. Wait until both peering links show status Connected before continuing.

Go to your Windows Server VM and restart the forwarder in PowerShell as Administrator:

```powershell
Restart-Service SplunkForwarder
```

Best practice for future labs: put all your lab VMs in the same VNet from the start, each in their own subnet. One shared VNet (e.g. lab-vnet, 10.0.0.0/16) with a subnet per lab environment (10.0.1.0/24 for AD, 10.0.2.0/24 for Splunk) means the VMs can communicate freely and you only manage NSG rules at the subnet level — no VNet Peering needed.

To avoid the IP changing between sessions: in the Azure portal go to your VM → Public IP address → Configuration and set the Assignment from Dynamic to Static. This keeps the same IP every time the VM starts so you never need to update your NSG rules or browser bookmarks.

## Step 2 — Configure Data Inputs

Splunk is only as useful as the data it receives. The first data source to configure is your Active Directory / Windows Server VM from Lab 1. You will install the Splunk Universal Forwarder on that Windows Server and point it at your Splunk instance.

### Part A — Enable receiving in Splunk first

Before installing the forwarder, tell Splunk to listen for incoming data on port 9997.

- Log into the Splunk web UI at http://your-vm-ip:8000
- Click Settings in the top navigation bar → Forwarding and Receiving
- Click Configure Receiving → New Receiving Port → enter 9997 → Save
- Click Settings → Indexes → Create New Index → name it windows_logs → Save

### Part B — Install the Universal Forwarder on Windows Server

Do this on your Windows Server VM from Lab 1 — not on the Splunk Ubuntu VM.

- On your Windows Server VM, open a browser and go to splunk.com/en_us/download/universal-forwarder.html. You will be prompted to log in — use the same email address and password you used when you created your Splunk account for the Enterprise download
- Download the Windows 64-bit installer (.msi file) — make sure you select the 64-bit version. The page shows multiple options including 32-bit and ARM versions. You want the one labelled Windows 64-bit
- Run the installer — when asked for a Deployment Server, leave this field blank and click Next. Do NOT enter your Splunk VM IP here. Setting a Deployment Server to your own VM IP causes the forwarder to phone home to the wrong address and no data will flow
- When asked for a Receiving Indexer, enter your Splunk VM's private IP address and port 9997 (e.g. 10.2.0.4:9997)
- Complete the installation with the default settings

### Part C — Configure inputs.conf to specify which logs to forward

The inputs.conf file tells the forwarder exactly which Windows Event Logs to collect and send. You need to create or edit this file on your Windows Server VM.

The file path below must be exact. We recommend using Visual Studio Code (VS Code) to create and edit this file — it is free, handles config files cleanly, and is far easier than Notepad for this kind of work.To install VS Code on your Windows Server VM: open a browser inside the VM, go to code.visualstudio.com, download the Windows installer, and run it with the default settings. Once installed, right-click the VS Code shortcut and select Run as Administrator — this is required so VS Code can save files into the Program Files directory.Then open the folder: C:\Program Files\SplunkUniversalForwarder\etc\system\local\ — if the local folder does not exist, create it in Windows Explorer first. In VS Code, go to File → Open Folder and navigate to that location. Create a new file called inputs.conf and paste the configuration below into it. Save with Ctrl+S.

```ini
# File location on Windows Server VM:
# C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
 
# Each section in square brackets defines one log source to collect
 
[WinEventLog://Security]
# The Security log contains all authentication events — logins, failures, lockouts
disabled = 0
# disabled=0 means enabled. disabled=1 would turn this source off.
start_from = oldest
# Collect historical events, not just new ones going forward
current_only = 0
evt_resolve_ad_obj = 1
# Resolve Active Directory object names so usernames appear instead of SIDs
index = windows_logs
# Required — without this line data goes to the main index not windows_logs
 
[WinEventLog://System]
# System log contains OS-level events — service starts/stops, driver failures
disabled = 0
index = windows_logs
 
[WinEventLog://Application]
# Application log contains events from installed applications
disabled = 0
index = windows_logs

# After saving inputs.conf, restart the forwarder to apply the changes
# Run this in PowerShell as Administrator on the Windows Server VM:
# Restart-Service SplunkForwarder
```

: Splunk comes with sample data built in. Go to Search → Data Summary and look for sample indexes. You can complete most of the search exercises below using the built-in sample data while you build out Lab 1.

### Step 2D — Generate Test Log Data

This is a fresh Windows Server VM so the Security, System, and Application logs are mostly empty. Before running the SPL searches in Step 3 you need to generate some activity so Splunk has data to show you.

The PowerShell script below simulates realistic security events — failed logins, account lockouts, successful logins, service events, and application errors. These are the exact event types the searches and dashboard panels in this lab are built to detect. Run it once before moving on to Step 3.

This script creates a temporary local user account called labtest.user, generates login activity against it, then deletes it. It only creates test events in the Windows Event Log — nothing is installed or changed permanently on the VM.

Open PowerShell as Administrator on your Windows Server VM

Right-click the Start menu → Windows PowerShell (Admin) and paste the entire script below. It takes about 60–90 seconds to run.

```powershell
# ============================================================
# ============================================================
# Lab 3 Log Generator - Run as Administrator on the Windows VM
# Output saved to C:\lab3-log-output.txt
# ============================================================
 
$logFile = 'C:\lab3-log-output.txt'
$timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
 
function Log($message, $color = 'White') {
    Write-Host $message -ForegroundColor $color
    Add-Content -Path $logFile -Value "[$timestamp] $message"
}
 
# Clear any previous run output
if (Test-Path $logFile) { Remove-Item $logFile }
Add-Content -Path $logFile -Value "Lab 3 Log Generator - Run started at $timestamp"
Add-Content -Path $logFile -Value '============================================='
 
Log 'Starting log generation...' Green
 
# Create a temporary test user account
$testUser = 'labtest.user'
$testPass = ConvertTo-SecureString 'TempPass123!' -AsPlainText -Force
New-LocalUser -Name $testUser -Password $testPass -Description 'Splunk lab test account' -ErrorAction SilentlyContinue
Log "Created test user: $testUser" Gray
 
# --- Generate failed logon attempts (creates Security log activity) ---
Log 'Generating failed logon attempts (Security log activity)...' Yellow
$wrongPass = ConvertTo-SecureString 'WrongPassword!' -AsPlainText -Force
1..15 | ForEach-Object {
    $cred = New-Object System.Management.Automation.PSCredential($testUser, $wrongPass)
    Start-Process -FilePath 'cmd.exe' -Credential $cred -ArgumentList '/c exit' -ErrorAction SilentlyContinue
    Start-Sleep -Milliseconds 500
}
Log '  Generated 15 failed logon attempts' Gray
 
# --- Generate successful login (Event ID 4624) ---
Log 'Generating successful login event (4624)...' Yellow
$correctPass = ConvertTo-SecureString 'TempPass123!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential($testUser, $correctPass)
Start-Process -FilePath 'cmd.exe' -Credential $cred -ArgumentList '/c whoami' -Wait -ErrorAction SilentlyContinue
Log '  Generated successful login (Event ID 4624)' Gray
 
# --- Generate service start/stop events (Event ID 7036) ---
Log 'Generating service events (7036)...' Yellow
$services = @('Spooler','Schedule','Netlogon')
$services | ForEach-Object {
    Stop-Service -Name $_ -Force -ErrorAction SilentlyContinue
    Start-Sleep -Seconds 2
    Start-Service -Name $_ -ErrorAction SilentlyContinue
    Start-Sleep -Seconds 1
    Log "  Stopped and restarted service: $_" Gray
}
 
# --- Generate application log entries ---
Log 'Generating application log events...' Yellow
$eventSource = 'SplunkLabTest'
if (-not [System.Diagnostics.EventLog]::SourceExists($eventSource)) {
    New-EventLog -LogName Application -Source $eventSource -ErrorAction SilentlyContinue
}
1..5 | ForEach-Object {
    Write-EventLog -LogName Application -Source $eventSource -EventId 1001 `
        -EntryType Warning -Message 'Splunk lab test event - application warning'
    Start-Sleep -Milliseconds 300
}
Log '  Generated 5 application log entries (Event ID 1001)' Gray
 
# --- Lock out the test account (Event ID 4740) ---
Log 'Generating account lockout event (4740)...' Yellow
$badCred = ConvertTo-SecureString 'BadPass!' -AsPlainText -Force
1..20 | ForEach-Object {
    $cred = New-Object System.Management.Automation.PSCredential($testUser, $badCred)
    Start-Process -FilePath 'cmd.exe' -Credential $cred -ArgumentList '/c exit' -ErrorAction SilentlyContinue
    Start-Sleep -Milliseconds 200
}
Log '  Account lockout triggered (Event ID 4740)' Gray
 
# --- Clean up the test account ---
Start-Sleep -Seconds 3
Remove-LocalUser -Name $testUser -ErrorAction SilentlyContinue
Log "Removed test user: $testUser" Gray
 
# --- Restart the forwarder ---
Log 'Restarting forwarder to ship events to Splunk...' Yellow
Restart-Service SplunkForwarder -ErrorAction SilentlyContinue
Log '  SplunkForwarder restarted' Gray
 
$endTime = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
Add-Content -Path $logFile -Value '============================================='
Add-Content -Path $logFile -Value "Run completed at $endTime"
Add-Content -Path $logFile -Value 'Wait 60 seconds then run: index=windows_logs | head 100'
 
Write-Host ''
Write-Host '=============================================' -ForegroundColor Green
Write-Host 'Done. Output saved to C:\lab3-log-output.txt' -ForegroundColor Green
Write-Host 'Wait 60 seconds then search Splunk.' -ForegroundColor Green
Write-Host '=============================================' -ForegroundColor Green
```

After the script finishes wait 60 seconds for the forwarder to ship the events to Splunk. Then set the time range in Splunk to All Time and run: index=windows_logs | head 100. You should see events from the Security, System, and Application logs. If you see events you are ready for Step 3.

## Step 3 — Essential SPL Searches

All searches are typed into the search bar at the top of the Search & Reporting app. Select a time range using the time picker on the right side of the search bar. Each search below is explained so you understand what every part does.

### Confirm data is flowing

```spl
index=windows_logs | head 100
# index=windows_logs  — search only in the windows_logs index you created
# | head 100          — return only the first 100 events
# If this returns results, your forwarder is working correctly
# If it returns nothing, check that the SplunkForwarder service is running on the Windows VM
```

### Find successful logins (EventCode 4624)

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| stats count by Account_Name
| sort -count
```

### Detect after-hours logins

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| eval hour=strftime(_time, "%H")
| where hour < 7 OR hour > 19
| table _time, Account_Name, Account_Domain, ComputerName
| sort -_time
 
# | eval hour=strftime(_time, "%H")   — extract just the hour from the timestamp
# | where hour < 7 OR hour > 19       — keep only events outside 7am-7pm
# Account names ending in $ are computer accounts — normal and expected
# Human account names without $ logging in after hours warrant review
```

## Step 4 — Build a Security Dashboard

Dashboards give you a permanent view of your security posture without running searches manually every time. Build one now using the searches from Step 3.

- In Splunk, click Dashboards in the top navigation
- Click the Create New Dashboard button in the top right corner of the Dashboards page
- In the dialog: set Dashboard Title to Windows Security Overview, leave Description blank, leave Permissions as Private, click Classic Dashboards under Dashboard type, then click Create → click Create Dashboard
- Click Add Panel for each panel below
- For each panel, select New Search, paste the search, choose the visualisation type, and save

| Panel | Search to use | Visualisation |
|---|---|---|
| Account Activity — Last 24h | `index=windows_logs sourcetype=WinEventLog:Security EventCode=4624 \| stats count by Account_Name \| sort -count` | Bar chart |
| Top Processes — Last 24h | `index=windows_logs sourcetype=WinEventLog:Security EventCode=4688 \| stats count by Creator_Process_Name \| sort -count \| head 20` | Events list |
| Login Activity Over Time | `index=windows_logs sourcetype=WinEventLog:Security EventCode=4624 \| timechart count` | Line chart |
| After-Hours Logins | `index=windows_logs sourcetype=WinEventLog:Security EventCode=4624 \| eval hour=strftime(_time, "%H") \| where hour < 7 OR hour > 19 \| table _time, Account_Name, Account_Domain, ComputerName \| sort -_time` | Events list |

## Step 5 — Create an Automated Alert

Alerts automate detection. Instead of checking dashboards manually, Splunk runs a search on a schedule and notifies you when the conditions are met. This is how real SOC detection works.

Run this search in the search bar first to confirm it works:

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4672
| stats count as privilege_logons by Account_Name, ComputerName
| where privilege_logons > 50
# This finds any account with more than 50 privileged logon events
# Privileged logons (4672) fire when admin-level rights are assigned at logon
```

- Click Save As → Alert
- Name: High Privileged Logon Count
- Alert type: Scheduled
- Run every: 15 minutes — select Cron Schedule and enter the cron expression: `*/15 * * * *`
- Trigger condition: Number of Results is greater than 0
- Trigger actions: Add to Triggered Alerts — this logs the alert to Splunk’s built-in alert history at Activity → Triggered Alerts. Every time the condition is met Splunk records it there with a timestamp and the result count. It is the simplest action and does not require any external system like email or a ticketing tool. Think of it as Splunk writing to its own incident log. You can check Activity → Triggered Alerts at any time to see a history of every time this alert fired
- Click Save

The quality of your alerts determines the quality of your security monitoring. An alert that fires too broadly creates alert fatigue — analysts stop paying attention. An alert that is too narrow misses real threats. Setting the threshold at 10 failures is a starting point. In a real environment you would tune this based on how many false positives you see over time.

## Verification — Confirm the Lab is Working

| Check | How to verify |
|---|---|
| Data is flowing into Splunk | Run: `index=windows_logs \| head 10` — should return recent events |
| Login activity search works | Run the EventCode=4624 search — should return results immediately as logon events are generated continuously by the Windows VM |
| Dashboard displays data | Windows Security Overview dashboard shows populated panels with charts |
| Alert is active | Settings → Searches, Reports, and Alerts — your alert appears as Enabled |

Portfolio entry for this lab. Take screenshots of your dashboard and your alert configuration. Export a few interesting searches as reports. Write a one-paragraph summary of what each panel shows and why it matters. This is concrete evidence of SIEM experience before your first job.

## Appendix — Additional Queries for Richer Environments

The two queries below require specific Windows event types that may not be present on a fresh lab VM. They are included here as reference material for when you have a more active environment — for example a machine that has been running for several days, a real Active Directory deployment, or a VM where additional software has been installed.

These queries are not required to complete the lab. All the exercises in Step 3 work with the data already in your index. Come back to these when your environment has had more time to generate activity.

### Detect logon with explicit credentials (EventCode 4648)

Event 4648 fires when credentials are explicitly passed to authenticate — for example using runas, passing credentials over the network, or a process authenticating as a different user. This is a common technique in lateral movement and credential abuse. In a normal environment 4648 events are rare. A cluster of them is worth investigating.

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4648
| table _time, Account_Name, Target_Server_Name, Process_Name
| sort -_time
 
# EventCode=4648         — explicit credential logon
# Target_Server_Name     — the machine or service being accessed
# Process_Name           — the process that initiated the authentication
# Legitimate uses: scheduled tasks, backup agents, some admin tools
# Suspicious uses: lateral movement, pass-the-hash, credential stuffing
```

### Detect process creation (EventCode 4688)

Event 4688 fires every time a new process is launched on the machine. With 2361 events in your index this is the richest data source you have. It is one of the most valuable event codes for threat hunting — malware, ransomware, and attackers all have to run processes. Seeing what ran, when, and under which account is the foundation of endpoint detection.

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4688
| stats count by Creator_Process_Name
| sort -count
| head 20
 
# Creator_Process_Name   — the process that spawned the new process
# Processes run from temp folders, AppData, or unusual paths are suspicious
# cmd.exe and powershell.exe spawned by unexpected parents are red flags
# This query shows your top 20 most-executed processes
```

### Detect service installations (EventCode 4697)

Event 4697 fires when a new service is installed on the system. Services run with elevated privileges and persist across reboots — which makes them a favourite persistence mechanism for malware and ransomware. Legitimate service installs happen during software setup. Unexpected ones outside of change windows warrant immediate investigation.

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4697
| table _time, Account_Name, Service_Name, Service_File_Name
| sort -_time
 
# Service_Name           — the name the service was registered under
# Service_File_Name      — the executable that runs as the service
# Services with random or obfuscated names are a malware indicator
# Services pointing to temp directories or unusual paths are suspicious
# Cross-reference Account_Name to see which user installed the service
```
