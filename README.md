# Cheese CTF – TryHackMe Walkthrough

## Overview
Cheese CTF is a Linux-based TryHackMe room that includes web enumeration, authentication bypass, local file inclusion, remote code execution, gaining user access, and privilege escalation to root.

## Objectives
- Gain initial access to the target
- Capture the user flag
- Escalate privileges to root
- Capture the root flag

## Tools Used
- Nmap
- ffuf
- curl
- php_filter_chain_generator
- netcat
- SSH
- MySQL
- systemctl
- xxd

## Target Information
- Target IP: `10.113.130.34`

## Walkthrough Sections
1. Reconnaissance
2. Login Bypass (SQL Injection)
3. Local File Inclusion (LFI)
4. Reading PHP Source Code
5. Remote Code Execution (RCE)
6. Reverse Shell
7. Credential Discovery
8. User Access via SSH Key Injection
9. User Flag
10. Privilege Escalation to Root
11. Root Flag
12. Lessons Learned
