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
* Networking Profile: Private Host-Only Subnet (192.168.199.0/24)

## Structural Exercises Documented
1. Private network configuration and host connection tracking.
2. Administrative command analysis via netstat, tasklist, and ipconfig.
3. Plaintext payload extraction: Wireshark packet capture analysis comparing Telnet cleartext to encrypted SSH streams.
