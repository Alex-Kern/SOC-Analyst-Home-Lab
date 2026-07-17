# SOC-Analyst-Home-Lab
Isolated local virtualization laboratory for security analysis.


# Local Defensive Security Laboratory Setup
By Alexander Kern

## Executive Summary
This repository contains my active home lab environment used to test host security, analyze network traffic, and practice command-line diagnostics.

## Architecture Specifications
* Hypervisor: Oracle VirtualBox 7.2.12
* Host Operating System: Windows 11 (64-bit)
* Linux Terminal (Attack/Audit Source): Ubuntu Desktop (22.04.5 LTS)
* Target Windows Server (Defensive Host): Windows Server 2022
* Networking Profile: Private NAT Network Subnet (10.0.2.0/24)
![Network Topology Diagram](./images/Network_Topology.png)

## Structural Exercises Documented
1. Private network configuration and host connection tracking.
2. Administrative command analysis via netstat, tasklist, and ipconfig.
3. Plaintext payload extraction: Wireshark packet capture analysis comparing Telnet cleartext to encrypted SSH streams.
   
---

Command-Line Analysis & Network Sockets

1. Active Diagnostic Environmental Baselines

Ubuntu Linux Terminal Environment
Interface & Ping Diagnostics (`ip a` & `ping -c 4 8.8.8.8`):
  ![Ubuntu Interfaces and Network Ping](./images/ubuntu_network.png)
* **Active Sockets Baseline (`netstat -ano`):**
  ![Ubuntu Socket Map](./images/ubuntu_netstat.png)

## Windows Server 2022 Terminal Environment
* **Interface & Ping Diagnostics (`ipconfig /all` & `ping 8.8.8.8`):**
  ![Windows Configuration and Ping](./images/windows_network.png)
* **Active Sockets Baseline (`netstat -ano`):**
  ![Windows Socket Map](./images/windows_netstat.png)

---

### 2. Technical Concepts & Triage Analysis

## What is a Network Socket?
A network socket is basically an internal connection endpoint that the operating system uses to send and receive data over a network. 
It is created by combining an IP Address with a Port Number (like `10.0.2.4:22` or `10.0.2.15:80`). 
The IP address gets the data packet to the right machine on the network, and the port number makes sure that 
the data gets handed off to the exact application or service waiting for it in memory.

## Tracking Suspicious PIDs Using Native Utilities
When looking at network traffic in a SOC environment, an analyst needs a way to connect network activity back to the actual software running on a computer to catch unauthorized activity or malware:

1. Finding the Connection: Running `netstat -ano` lets you see a live list of every open port and active connection on the system. 
2. Finding the PID: Adding the `-o` flag is critical because it tells the OS to show the Process ID (PID) for each connection. This number acts as the direct link between network traffic and the system process responsible for it.
3. Identifying the Program: 
   * On Windows, you take that PID and run `tasklist` in the command line to see the name of the executable file running it.
   * On Linux, you can run `ps -p [PID]` to see the exact program path. 

This tracking process allows an analyst to look at an unrecognized connection, find the PID, and immediately verify if it belongs to a legitimate system tool or a malicious program hiding on the machine.
## Process Resolution:

   * On Windows, running `tasklist` resolves the target identification number to its parent executable name, which helps catch malicious programs masquerading in temporary directories.
   * On Linux, querying `ps -p [PID]` unmasks the binary origin path, allowing an analyst to verify if a socket belongs to an approved system daemon or an unauthorized process.

### 3. Note on Service Management
Practiced starting, inspecting, and terminating native local services programmatically via `systemctl` on Linux and `Stop-Service` / `Get-Service` on Windows Server to simulate service-layer incident response.

## 4. SIEM Operations: Log Ingestion & Query-Driven Event Triage

### Objective
To move beyond isolated host analysis, I deployed **Splunk Enterprise** on my Windows Server VM to act as my central SIEM platform. The goal was to build a local log ingestion pipeline, simulate a credential brute-force attack, and use Splunk's Search Processing Language (SPL) to reconstruct the event timeline.

### Log Ingestion Architecture
Setting up the data input actually tripped me up more than I expected. I went into Splunk's **Settings > Data Inputs** looking for local event log collection, and my first instinct was to click into **Forwarded Inputs** and click on Windows Event Logs, since that sounded the most like what I wanted to look at in the list. That's meant for receiving logs shipped over from other Splunk forwarders though, not for pulling logs off the local machine itself, so nothing showed up when I tried to configure it that way. After poking around for a bit I found the right section under **Local Inputs > Local Event Log Collections**, which is what actually lets Splunk read straight from the Windows Event Log service on the same box.

Once that was sorted, the setup was simple:
- **Monitored Logs:** Windows `Security` and `System` channels
- **Storage Destination:** Splunk's default index (`index="main"`)

Since this is a standalone VM with no forwarders in the environment, everything stays local, no distributed collection needed:

```text
+-----------------------+      Local Event Ingestion      +----------------------------+
|  Windows Server 2022  | ------------------------------> |     Splunk Enterprise      |
|  Security Event Logs  |     (Local Event Collection)    | (Local Dashboard Port 8000) |
+-----------------------+                                 +----------------------------+
```

### Attack Simulation
To generate some real telemetry to chase down, I locked the Windows Server VM and ran a simulated brute-force attempt, with 6 consecutive failed logins at the lock screen using random passwords, just to throw some noise into the security logs.

### Event Triage & SPL Analysis
After logging back in, I opened up Splunk's Search and Reporting app and started digging.

**Step 1: Isolating the failed logins**

I searched for EventCode 4625 (failed Windows logins) to pull the attack footprint out of the noise:

```spl
index="main" EventCode=4625
```

![Splunk Failed Login Query Output](./images/splunk_failed_logins.png)

This filtered down to a tight cluster of failures, all within about a minute — targeting the `Administrator` account, from the local workstation, with timestamps lining up exactly with when I ran the simulation.

**Step 2: Checking whether it actually got in**

Next question was obvious: did any of those attempts succeed? I pivoted the query to EventCode 4624 (successful logins) over the same time window:

```spl
index="main" EventCode=4624
```

Nothing came back for `Administrator` during that window, so the attack failed to get through, which is what I wanted to confirm.

### Takeaways
Getting the data input wrong at first was honestly a useful mistake. It made me actually understand the difference between forwarded and local inputs instead of just following a checklist. Beyond that, this lab made it clear why SOCs lean so heavily on centralized logging: instead of scrolling through Event Viewer on individual boxes, everything lands in one searchable place. And once I had the query working, it wasn't hard to see how something like this would get turned into a real alert — flag anything with 5+ failed logins in 30 seconds and auto-isolate the host.
