# 🔐 Secure VPC Networking on AWS

> A production-style Virtual Private Cloud (VPC) built from scratch on AWS, demonstrating secure network segmentation, controlled internet access, and least-privilege traffic control.

![AWS](https://img.shields.io/badge/AWS-VPC-orange?logo=amazonaws)
![Status](https://img.shields.io/badge/status-complete-brightgreen)
![IaC](https://img.shields.io/badge/IaC-Terraform%20%2F%20CloudFormation-informational)

---

## 📖 Project Overview

Modern cloud applications require secure and scalable networking. This project demonstrates the deployment of a production-style VPC on Amazon Web Services using **public and private subnets**, **route tables**, **security groups**, and a **NAT Gateway**.

The objective is to provide secure internet access to public-facing resources while protecting backend workloads from direct internet exposure — following AWS's core network segmentation and least-privilege best practices.

---

## 🏗️ Architecture

> 📌 *Insert your architecture diagram here (e.g. `docs/architecture-diagram.png`), then embed it below:*
> `![Architecture Diagram](docs/architecture-diagram.png)`

```
                                   Internet
                                       │
                              Internet Gateway
                                       │
        ┌──────────────────────────── VPC ────────────────────────────┐
        │                     10.0.0.0/16                             │
        │                                                              │
        │   ┌────────────────── Public Subnet ───────────────────┐    │
        │   │            10.0.1.0/24 (AZ-a)                      │    │
        │   │                                                     │    │
        │   │   ┌────────────────┐        ┌──────────────────┐   │    │
        │   │   │  Bastion Host  │        │   NAT Gateway     │   │    │
        │   │   │   (optional)   │        │                   │   │    │
        │   │   └────────────────┘        └──────────────────┘   │    │
        │   │   ┌────────────────┐                                │    │
        │   │   │   Public EC2   │                                │    │
        │   │   └────────────────┘                                │    │
        │   └─────────────────────────────────────────────────────┘    │
        │                                                              │
        │   ┌────────────────── Private Subnet ──────────────────┐    │
        │   │            10.0.2.0/24 (AZ-a)                      │    │
        │   │                                                     │    │
        │   │   ┌──────────────────┐     ┌──────────────────┐    │    │
        │   │   │ Application EC2  │     │ Database (future)│    │    │
        │   │   └──────────────────┘     └──────────────────┘    │    │
        │   └─────────────────────────────────────────────────────┘    │
        │                                                              │
        │   Route Tables · Security Groups · Network ACLs             │
        └──────────────────────────────────────────────────────────────┘
```

---

## ☁️ AWS Services Used

| Service | Purpose |
|---|---|
| **Amazon VPC** | Isolated virtual network |
| **Public Subnet** | Hosts internet-facing resources |
| **Private Subnet** | Hosts internal application/backend resources |
| **Internet Gateway (IGW)** | Provides internet connectivity to the VPC |
| **NAT Gateway** | Enables outbound-only internet access for private instances |
| **Route Tables** | Directs traffic between subnets, the IGW, and the NAT Gateway |
| **Security Groups** | Instance-level, stateful firewall |
| **Network ACLs** | Subnet-level, stateless firewall |
| **EC2** | Compute instances (public web tier and private app tier) |

---

## 🎯 Project Objectives

- [x] Build a secure VPC from scratch
- [x] Create public and private subnets across an Availability Zone
- [x] Configure secure routing between subnets, the IGW, and the NAT Gateway
- [x] Deploy EC2 instances in both public and private subnets
- [x] Restrict inbound traffic to only what is required
- [x] Enable outbound internet access for private instances via NAT Gateway
- [x] Apply least-privilege networking principles throughout

---

## 🌐 Network Design

| Component | Description |
|---|---|
| **VPC** | Custom CIDR block (e.g. `10.0.0.0/16`) |
| **Public Subnet** | Hosts internet-facing resources (e.g. `10.0.1.0/24`) |
| **Private Subnet** | Hosts backend resources (e.g. `10.0.2.0/24`) |
| **Internet Gateway** | Enables inbound and outbound internet traffic for the public subnet |
| **NAT Gateway** | Allows outbound-only internet access for private instances |
| **Route Tables** | Direct traffic between resources — public route table points to the IGW, private route table points to the NAT Gateway |
| **Security Groups** | Stateful, instance-level traffic control |
| **Network ACLs** | Stateless, subnet-level traffic control as a defense-in-depth layer |

### Example Route Tables

| Route Table | Destination | Target |
|---|---|---|
| Public RT | `10.0.0.0/16` | `local` |
| Public RT | `0.0.0.0/0` | Internet Gateway |
| Private RT | `10.0.0.0/16` | `local` |
| Private RT | `0.0.0.0/0` | NAT Gateway |

### Example Security Group Rules

| Security Group | Direction | Port | Source | Purpose |
|---|---|---|---|---|
| Public-SG | Inbound | 22 | Authorized IP / Bastion SG | SSH access |
| Public-SG | Inbound | 80/443 | `0.0.0.0/0` | Web traffic |
| Private-SG | Inbound | 22 | Public-SG (Bastion) | SSH via bastion only |
| Private-SG | Inbound | Custom app port | Public-SG | App tier traffic only |
| Private-SG | Outbound | All | `0.0.0.0/0` (via NAT) | Updates, package installs |

---

## 🛡️ Security Features

- Private EC2 instances have **no public IP addresses**.
- Security Groups follow the **principle of least privilege** — only required ports/sources are allowed.
- Network ACLs provide an **additional, stateless security layer** at the subnet boundary.
- The Internet Gateway is attached and routable **only from public resources**.
- Private workloads reach the internet **only through the NAT Gateway** (outbound-only).
- SSH access is **restricted to authorized sources**, or brokered through a Bastion Host.

---

## 🚀 Deployment Steps

1. **Create the VPC** with a custom CIDR block.
2. **Create public and private subnets** within the VPC (ideally across multiple AZs).
3. **Attach an Internet Gateway** to the VPC.
4. **Configure route tables** — public subnet → IGW, private subnet → NAT Gateway.
5. **Deploy a NAT Gateway** in the public subnet with an Elastic IP.
6. **Launch EC2 instances** — one in the public subnet, one in the private subnet.
7. **Configure Security Groups** for each tier following least privilege.
8. **Configure Network ACLs** as a subnet-level backstop.
9. **Validate internet connectivity** for both public and private instances.
10. **Test communication** between the public and private instances (e.g. SSH hop via bastion, app-tier connectivity).

---

## ✅ Validation

- [x] Public EC2 is accessible from the internet.
- [x] Private EC2 **cannot** be reached directly from the internet.
- [x] Private EC2 can reach the internet through the NAT Gateway.
- [x] Security Groups permit only authorized traffic.
- [x] Route tables function as expected for both subnets.

---

## 📚 Lessons Learned

- The practical difference between **Security Groups** (stateful, instance-level) and **Network ACLs** (stateless, subnet-level).
- The importance of **network segmentation** in limiting blast radius.
- How to design **secure routing** using public and private subnet pairs.
- **NAT Gateway vs. Internet Gateway** — outbound-only access vs. full bidirectional internet access.
- How to design networks with **scalability and security** in mind from day one.

---

## 🔭 Future Improvements

- [ ] Multi-AZ deployment for high availability
- [ ] VPC Flow Logs for network traffic visibility and auditing
- [ ] AWS Network Firewall for advanced traffic inspection
- [ ] AWS WAF for application-layer protection
- [ ] AWS Transit Gateway for multi-VPC connectivity
- [ ] Site-to-Site VPN for hybrid connectivity
- [ ] VPC Endpoints (Gateway/Interface) to keep AWS service traffic off the public internet
- [ ] Infrastructure as Code using **Terraform**
- [ ] Infrastructure as Code using **AWS CloudFormation**

---

## 🧠 Skills Demonstrated

`AWS Networking` `Amazon VPC` `Cloud Security` `Network Design` `Security Groups` `Network ACLs` `Route Tables` `NAT Gateway` `Internet Gateway` `EC2` `Cloud Architecture` `AWS Best Practices`

---

## 📂 Repository Structure

```
secure-vpc-networking-aws/
├── README.md
├── docs/
│   └── architecture-diagram.png
├── terraform/            # (future) Terraform IaC
├── cloudformation/       # (future) CloudFormation templates
└── screenshots/          # Console screenshots for validation steps
```

---

## 👤 Author

## **Alex Agyei**

Cloud Engineering Student (AWS & Azure) | Core Banking Systems Developer
