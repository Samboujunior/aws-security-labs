# Lab 2: Securing VPC Resources Using Security Groups

## Overview

This lab focuses on securing EC2 instances inside an Amazon VPC using **Security Groups** and **Network ACLs**. It demonstrates multiple layers of network security — from open HTTP access, to IP-restricted rules, to security group referencing — alongside different methods of accessing EC2 instances securely (Bastion Host, AWS Session Manager).

---

## AWS Services Used

| Service | Purpose |
|---|---|
| Amazon VPC | Isolated network environment (LabVPC) |
| Amazon EC2 | AppServer, BastionHost, ProxyServer instances |
| Security Groups | Stateful firewall rules controlling instance-level traffic |
| Network ACLs | Stateless subnet-level traffic filtering |
| AWS Systems Manager (Session Manager) | Agentless browser-based shell access to EC2 |
| NAT Gateway | Outbound internet access for private subnet instances |

---

## Architecture

```
LabVPC (10.0.0.0/16)
    ├── PublicSubnetA  (10.0.1.0/24)
    │       ├── BastionHost         ← SSH jump server
    │       └── ProxyServer1/2      ← Proxy servers (ProxySG)
    │
    ├── PublicSubnetB  (10.0.2.0/24)
    │
    └── PrivateSubnet  (10.0.11.0/24)
            └── AppServer           ← Web application server
                    └── Route Table: 0.0.0.0/0 → NAT Gateway

Internet
    └── ProxyServer (54.157.53.182)
            └── forwards HTTP → AppServer
```

---

## What Was Demonstrated

### 1. VPC Subnet Structure
- Inspected the **LabVPC** containing three key subnets:
  - **PublicSubnetA** — `10.0.1.0/24` — hosts the Bastion and Proxy servers
  - **PublicSubnetB** — `10.0.2.0/24`
  - **PrivateSubnet** — `10.0.11.0/24` — hosts the AppServer

### 2. Private Subnet Routing Through NAT Gateway
- Selected the **PrivateSubnet** and inspected its route table (`rtb-0675cc49e5b04b01f / Private`).
- Routes configured:
  - `10.0.0.0/16` → `local` (VPC-internal traffic)
  - `0.0.0.0/0` → `nat-0e6280c8576000b1a` (all other traffic goes through NAT Gateway)
- This allows the AppServer to make outbound internet requests without being directly reachable from the internet.

### 3. Initial Security Group — HTTP Open to All
- Viewed the **AppServerSG** security group.
- Initial inbound rule: **HTTP (TCP port 80) from 0.0.0.0/0** — allowing all internet traffic to reach the AppServer.
- This is the baseline insecure configuration that the lab then hardens.

### 4. Web Application Accessed via ProxyServer
- Accessed the AppServer web application through **ProxyServer1** at `54.157.53.182`.
- Browser displayed: *"Hello from the web server running on the AppServer instance!"*
- Confirmed the application was reachable via the proxy while the security group was open.

### 5. Network ACL Blocking Inbound HTTP Traffic
- Created a **Network ACL inbound rule** on the subnet to explicitly **Deny** HTTP (TCP port 80) from `0.0.0.0/0`.
- Rule number: **99** (evaluated before the default allow-all rule).
- This demonstrated that NACLs act as a subnet-level firewall, blocking traffic before it reaches any instance — regardless of security group rules.

### 6. AppServer Security Group — Restricted to Specific IP
- Edited the **AppServerSG** inbound rule to restrict HTTP access to a single internal IP: **`10.0.1.154/32`**.
- This replaced the open `0.0.0.0/0` rule, ensuring only one specific host (a proxy or bastion at that IP) could send HTTP traffic to the AppServer.

### 7. Security Group Referencing — Allowing ProxySG
- Further refined the AppServerSG inbound rule using **security group referencing**.
- Instead of a specific IP, the source was set to **ProxySG** (`sg-032be72e1253b38c7`).
- This means only EC2 instances that are members of the ProxySG security group can send HTTP traffic to the AppServer — a more scalable and dynamic approach than hardcoding IPs.

### 8. SSH Access to AppServer via Bastion Host
- Connected to the **BastionHost** (`100.30.255.2`) over SSH.
- The terminal confirmed connection to Amazon Linux 2 (`[ec2-user@bastion ~]$`).
- From the bastion, SSH can be used to reach the AppServer in the private subnet — the classic jump-server pattern for accessing private instances.

### 9. Direct Access via AWS Session Manager
- Connected directly to the **AppServer** (`i-0389dbe59f54508ab`) using **AWS Systems Manager Session Manager** — no SSH key or open port 22 required.
- Used Session Manager to run a shell command modifying the web server's `index.html`:
  ```bash
  sudo sed -i 's/instance!/instance! Session manager was used to edit this file./g' /var/www/html/index.html
  ```
- This demonstrates that Session Manager provides secure, auditable shell access without needing a Bastion Host or inbound SSH rules.

---

## Key Concepts Learned

**Security Groups vs Network ACLs**

| Feature | Security Group | Network ACL |
|---|---|---|
| Level | Instance | Subnet |
| Statefulness | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Rules evaluated in order by rule number |

**Security Group Referencing**
Rather than specifying an IP CIDR as the source, a security group ID can be used. This dynamically allows traffic from any instance that belongs to the referenced security group — making rules portable and scalable as instances are added or replaced.

**Bastion Host Pattern**
A Bastion (or jump) host sits in a public subnet with SSH open from trusted IPs only. Administrators SSH into the bastion first, then SSH from the bastion into private instances. The private instances never need port 22 open to the internet.

**AWS Session Manager**
Session Manager eliminates the need for open inbound SSH ports and key pairs for administrative access. Access is controlled through IAM policies, and all sessions are logged to CloudTrail and optionally S3 — providing a more secure and auditable alternative to traditional SSH.

**NAT Gateway**
Instances in private subnets have no public IP and cannot receive inbound connections from the internet. A NAT Gateway in the public subnet allows these instances to initiate outbound connections (e.g., for software updates) while remaining unreachable from the outside.

---

## Network Configuration Summary

| Resource | Value |
|---|---|
| VPC | LabVPC — `10.0.0.0/16` |
| PublicSubnetA | `10.0.1.0/24` |
| PublicSubnetB | `10.0.2.0/24` |
| PrivateSubnet | `10.0.11.0/24` |
| AppServer Security Group | AppServerSG (`sg-0548516fdd8fa04e9`) |
| Proxy Security Group | ProxySG (`sg-032be72e1253b38c7`) |
| NAT Gateway | `nat-0e6280c8576000b1a` |
| AppServer Instance | `i-0389dbe59f54508ab` |
| BastionHost IP | `100.30.255.2` |
| ProxyServer Public IP | `54.157.53.182` |
| Region | US East (N. Virginia) — us-east-1 |

---

## Screenshots

| Screenshot | Description |
|---|---|
| `Subnets_configured_inside_the_LabVPC.PNG` | LabVPC subnets — PublicSubnetA, PublicSubnetB, PrivateSubnet |
| `Private_subnet_routing_traffic_through_a_NAT_Gateway.PNG` | PrivateSubnet route table showing NAT Gateway as default route |
| `Initial_AppServer_security_group_allowing_HTTP_from_any_source.PNG` | AppServerSG open to 0.0.0.0/0 on port 80 |
| `Web_application_accessed_through_ProxyServer1.PNG` | AppServer web app reachable via ProxyServer |
| `Network_ACL_blocking_inbound_HTTP_traffic.PNG` | NACL rule 99 denying HTTP from 0.0.0.0/0 |
| `AppServer_access_restricted_to_a_specific_internal_IP.PNG` | AppServerSG restricted to 10.0.1.154/32 |
| `Security_group_referencing_used_to_allow_proxy_servers.PNG` | AppServerSG source set to ProxySG security group ID |
| `SSH_access_to_AppServer_through_Bastion_host.PNG` | SSH session on BastionHost (Amazon Linux 2) |
| `Direct_connection_to_AppServer_using_AWS_Session_Manager.PNG` | Session Manager shell on AppServer — editing index.html |

---

## Summary

This lab demonstrated a layered approach to VPC security. Starting from a wide-open HTTP security group, access was progressively locked down — first using Network ACLs at the subnet level, then restricting the security group to a specific IP, and finally using security group referencing so only trusted proxy instances could reach the AppServer. Two secure remote access methods were also compared: the traditional **Bastion Host** SSH pattern and the modern, keyless **AWS Session Manager** approach. The private subnet's outbound access through a **NAT Gateway** completed the picture of a properly segmented, secure VPC architecture.
