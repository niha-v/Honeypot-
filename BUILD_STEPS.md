# AWS Cowrie Honeypot — Build Steps

This document walks through how the honeypot was built, step by step, from a bare AWS account to a fully running, isolated Cowrie SSH honeypot with off-instance logging.

---

## 1. Network Isolation — VPC Setup

- Created a dedicated VPC (`10.99.0.0/16`) separate from any other AWS resources.
- Created a public subnet (`10.99.1.0/24`) within it.
- Created and attached an Internet Gateway to the VPC.
- Configured the route table for the subnet with a `0.0.0.0/0` route pointing to the Internet Gateway, making the subnet publicly reachable.
  
**Why:** Keeping the honeypot in its own VPC with a distinct CIDR range means that even in a worst-case compromise, there's no network path back to any other AWS infrastructure.

---

## 2. Security Group — Controlled Exposure

Created a security group (`honeypot-sg`) with the following inbound rules:

| Port | Purpose | Source |
|------|---------|--------|
| 2222 | Cowrie's actual listener | `0.0.0.0/0` (open to the world — this is the intended attack surface) |
| 2200 | Real SSH management access | Restricted to a single known IP |

Outbound was later restricted to ports **443** and **80** only — enough for CloudWatch log shipping and system updates, nothing more.

<img src = "https://github.com/niha-v/Honeypot-/blob/main/Inbound%20Rules.png" width ="500" >

**Why:** The honeypot should only ever expose exactly what it's meant to expose. Management access is kept completely separate from the honeypot's attack surface.

---

## 3. Least-Privilege IAM Role

<img src = "https://github.com/niha-v/Honeypot-/blob/main/IAM%20role.png" width ="500" >

Created an IAM policy granting only:

- `logs:CreateLogGroup`
- `logs:CreateLogStream`
- `logs:PutLogEvents`
- `logs:DescribeLogStreams`

Scoped to a single log group ARN (`/honeypot/cowrie`), and attached to a new role (`honeypot-ec2-role`) used only by this EC2 instance.

**Why:** If the instance is ever fully compromised, this role limits the blast radius to "can write logs to one place" — nothing else in the AWS account is reachable via instance credentials.

---

## 4. EC2 Instance Launch

<img src = "https://github.com/niha-v/Honeypot-/blob/main/Ubuntu.png" width ="500" >

- Launched an Ubuntu Server 24.04 LTS instance (`t3.micro`, free-tier eligible).
- Deployed into the honeypot VPC/subnet with a public IP assigned.
- Attached `honeypot-sg` and `honeypot-ec2-role`.

<img src = "https://github.com/niha-v/Honeypot-/blob/main/Network%20Settings.png" width ="500" >

---

## 5. Cowrie Installation

- Installed system dependencies (`python3-venv`, `build-essential`, `libssl-dev`, `libffi-dev`, etc.).
- Created a dedicated **non-root** system user (`cowrie`) to run the honeypot.
- Cloned the Cowrie repository and set up a Python virtual environment.
- Installed Cowrie's dependencies and the package itself into the venv.

**Why:** Running the honeypot as a non-root, unprivileged user limits what could be affected if the Cowrie process itself were ever exploited.

---

## 6. Cowrie Configuration

- Copied the default config template and set a generic, believable hostname (avoiding anything that would tip off an attacker).
- Confirmed Cowrie's SSH listener was bound to port `2222`.
- Copied the default user database, which accepts a wide range of username/password combinations — matching real-world attacker expectations of a poorly-secured server.

---

## 7. Separating Real Management Access from the Honeypot

- Moved the real OpenSSH daemon from port 22 to port **2200**, reachable only from a single trusted IP.
- Verified the new port worked *before* touching anything else, to avoid getting locked out.

**Why:** Once port 22 is redirected to the honeypot, it can no longer serve as real management access — this had to be separated out first.

---

## 8. Redirecting Attacker Traffic into the Honeypot

- Added an `iptables` NAT rule redirecting inbound traffic on port **22** (the port real attackers scan) to Cowrie's listener on port **2222**.
- Made the rule persistent across reboots using `iptables-persistent`.

**Why:** Most internet-wide SSH scanning targets port 22 specifically. This redirect ensures that traffic lands in the honeypot, while true SSH access stays isolated on 2200.

---

## 9. Running Cowrie as a Managed Service

- Created a `systemd` unit file for Cowrie, running as the dedicated `cowrie` user.
- Enabled the service so it starts automatically on boot and restarts automatically if it crashes.

**Why:** A honeypot that silently stops running produces silently incomplete data. Systemd management ensures continuous uptime without manual intervention.

---

## 10. Off-Instance Log Shipping

- Installed the Amazon CloudWatch Agent.
- Configured it to tail Cowrie's log file and ship new entries to a dedicated CloudWatch Logs group (`/honeypot/cowrie`) in near real-time.
- Enabled the agent as a service so it also survives reboots.

**Why:** If an attacker fully breaks out of the honeypot's emulated environment, local logs could theoretically be deleted to cover tracks. Shipping logs off-instance in real time protects the evidence regardless of what happens locally afterward.

---

## 11. End-to-End Verification

- Confirmed a real SSH connection to port 22 was transparently redirected into Cowrie's fake shell.
- Confirmed login attempts and every command typed were captured in Cowrie's local log.
- Confirmed the same events appeared in CloudWatch Logs within seconds, verifying the full pipeline — from attacker connection, to capture, to durable off-instance storage — works end to end.

---

## Result

A fully isolated, least-privilege, self-healing SSH honeypot capturing real attacker behavior in real time, with logs preserved independently of the instance itself.
