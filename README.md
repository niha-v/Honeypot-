# AWS Cowrie Honeypot


## Overview

This is Phase B of the Threat Detection Platform: a low-interaction SSH honeypot deployed on AWS to capture real-world attacker behavior — login attempts, credentials tried, and commands executed — in a fully isolated, contained environment. Unlike Phase A (which analyzed a static malware PCAP sample) this phase generates live, first-hand attack data.


  <img src = "https://github.com/niha-v/Honeypot-/blob/main/Image1.png" width ="500" >


## What It Does

- Presents a fake SSH service on the real SSH port (22) that accepts any username/password combination, mimicking a poorly-secured Linux server.
- Logs every login attempt, source IP and every command an attacker runs inside the fake shell.
- Ships those logs off the instance in near real-time to AWS CloudWatch Logs, so evidence survives even if the honeypot itself is fully compromised.
- Runs entirely isolated from any other AWS resources on the account.

## Architecture

- **Dedicated VPC** (`10.99.0.0/16`) with its own public subnet, internet gateway, and route table — fully separated from any other AWS infrastructure.
- **EC2 instance** (Ubuntu 24.04 LTS, `t3.micro`) running Cowrie.
- Cowrie runs as a dedicated **non-root user**, managed as a `systemd` service so it survives reboots and restarts automatically on failure.
- Real SSH management access moved to **port 2200**, restricted by security group to a single known IP.
- Port **22** (the port attackers actually scan) is redirected via `iptables` NAT to Cowrie's listener on port **2222** — so any traffic hitting the "normal" SSH port lands in the honeypot, not real `sshd`.
- **Least-privilege IAM role** attached to the instance, scoped to exactly one permission: writing log events to a single CloudWatch log group. No other AWS permissions are granted, so even a fully compromised instance has no path to other resources on the account.
- **Outbound traffic** restricted to HTTPS/HTTP only (443/80) — enough for CloudWatch log shipping and system updates, nothing more.

## Tools and Technologies

| Category | Tool |
|---|---|
| Cloud platform | AWS (VPC, EC2, IAM, CloudWatch Logs) |
| Honeypot software | Cowrie (SSH/Telnet honeypot, Python/Twisted) |
| OS | Ubuntu Server 24.04 LTS |
| Host firewall | iptables (NAT redirect), netfilter-persistent |
| Log shipping | Amazon CloudWatch Agent |

## Security & Containment Design

This honeypot was deliberately built with defense-in-depth in mind, since it's intentionally exposed to the public internet:

1. **Network isolation** — separate VPC and CIDR range from any other AWS resource, so lateral movement to other infrastructure is not possible even at the network layer.
2. **Least privilege** — the IAM role can do exactly one thing (write to one specific log group) and nothing else.
3. **Non-root execution** — Cowrie runs as an unprivileged system user, not root, limiting the blast radius of any exploit against Cowrie itself.
4. **Off-instance logging** — logs are shipped to CloudWatch in near real-time, so an attacker deleting local logs to cover their tracks does not destroy the evidence.
5. **Minimal outbound access** — outbound traffic is restricted to just what's operationally needed, limiting what a compromised instance could be used for.
6. **Separated management access** — real administrative SSH access is on a non-standard port, restricted to a single IP via security group, completely separate from the honeypot's attack surface.

## Setup Summary

1. Create an isolated VPC, public subnet, internet gateway, and route table.
2. Create a security group allowing: `2222/tcp` from anywhere (Cowrie), `2200/tcp` from a single management IP (real SSH), and default outbound restricted to `443/80`.
3. Create a least-privilege IAM role scoped to CloudWatch Logs write access for one specific log group.
4. Launch an EC2 instance (Ubuntu 24.04, `t3.micro`) into the VPC, attaching the security group and IAM role.
5. Install Cowrie under a dedicated non-root user, in a Python virtual environment.
6. Move real `sshd` to port 2200; configure Cowrie to listen on 2222.
7. Add an `iptables` NAT rule redirecting inbound port 22 to port 2222, made persistent across reboots.
8. Set Cowrie up as a `systemd` service (auto-start, auto-restart on failure).
9. Install and configure the Amazon CloudWatch Agent to ship Cowrie's log file to a dedicated CloudWatch log group in near real-time.
10. Verify end-to-end: a live SSH login attempt against port 22 is captured by Cowrie and appears in CloudWatch Logs within seconds.

*(Full step-by-step walkthrough with reasoning: see [`BUILD_STEPS.md`](./BUILD_STEPS.md))*

## Results

*To be added once the honeypot has been running long enough to collect a meaningful sample of real attacker traffic. This section will cover: number of unique attacker IPs, most common credentials attempted, most common commands run post-login, and any notable attack patterns.*

## What's Next

Logs captured here will feed into later phases of the platform, including MITRE ATT&CK mapping of observed attacker commands and eventual machine learning-based anomaly detection across combined malware traffic (Phase A) and live honeypot data (this phase).
