# VPC Configuration Documentation

Detailed technical reference for the VPC deployed in this project thus CIDR allocation, subnets, gateways, route tables, and configuration steps. [README](https://github.com/lexisbil1/aws-secure-vpc-networking/blob/main/README.md) for the project overview and architecture diagram.

---

## 1. VPC Details

| Attribute | Value |
|---|---|
| **Name** | `secure-vpc-prod` |
| **CIDR Block** | `10.0.0.0/16` |
| **Region** | *e.g. eu-west-1* |
| **Tenancy** | Default |
| **DNS Hostnames** | Enabled |
| **DNS Resolution** | Enabled |

> Adjust the CIDR, region, and naming to match your actual deployment before publishing.

---

## 2. Subnet Layout

| Subnet Name | Type | CIDR Block | Availability Zone | Auto-assign Public IP |
|---|---|---|---|---|
| `public-subnet-a` | Public | `10.0.1.0/24` | AZ-a | Yes |
| `public-subnet-b` | Public | `10.0.3.0/24` | AZ-b | Yes |
| `private-subnet-a` | Private | `10.0.2.0/24` | AZ-a | No |
| `private-subnet-b` | Private | `10.0.4.0/24` | AZ-b | No |

**Notes:**
- Public subnets host internet-facing resources (public EC2, NAT Gateway, optional bastion).
- Private subnets host internal resources (application server, future database tier).
- A second AZ pair (`-b`) is included here to support the Multi-AZ future improvement — remove if your current deployment is single-AZ.

---

## 3. Internet Gateway (IGW)

| Attribute | Value |
|---|---|
| **Name** | `secure-vpc-igw` |
| **Attached to** | `secure-vpc-prod` |
| **State** | Attached |

**Purpose:** Provides bidirectional internet connectivity for resources in the public subnets only. Private subnets never route directly to the IGW.

---

## 4. NAT Gateway

| Attribute | Value |
|---|---|
| **Name** | `secure-vpc-natgw-a` |
| **Subnet** | `public-subnet-a` |
| **Elastic IP** | Allocated and associated |
| **Connectivity type** | Public |

**Purpose:** Allows instances in the private subnets to initiate outbound connections to the internet (e.g. OS/package updates) without being reachable from the internet. One NAT Gateway per AZ is recommended in production to avoid a cross-AZ single point of failure and reduce cross-AZ data transfer costs.

---

## 5. Route Tables

### Public Route Table — `public-rt`

Associated with: `public-subnet-a`, `public-subnet-b`

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | Internet Gateway (`secure-vpc-igw`) |

### Private Route Table — `private-rt`

Associated with: `private-subnet-a`, `private-subnet-b`

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | NAT Gateway (`secure-vpc-natgw-a`) |

---

## 6. Security Groups

### `public-sg` — attached to public EC2 / bastion

| Direction | Protocol | Port | Source/Destination | Purpose |
|---|---|---|---|---|
| Inbound | TCP | 22 | Authorized admin IP (`x.x.x.x/32`) | SSH management access |
| Inbound | TCP | 80 | `0.0.0.0/0` | HTTP web traffic |
| Inbound | TCP | 443 | `0.0.0.0/0` | HTTPS web traffic |
| Outbound | All | All | `0.0.0.0/0` | Unrestricted outbound |

### `private-sg` — attached to private application EC2

| Direction | Protocol | Port | Source/Destination | Purpose |
|---|---|---|---|---|
| Inbound | TCP | 22 | `public-sg` | SSH from bastion only |
| Inbound | TCP | 8080 | `public-sg` | Application traffic from web tier only |
| Outbound | All | All | `0.0.0.0/0` | Outbound via NAT Gateway (updates, API calls) |

> Reference the security group by ID (not by CIDR) as the source where possible — this keeps rules valid even if instances are replaced.

---

## 7. Network ACLs

### `public-nacl` — associated with public subnets

| Rule # | Type | Protocol | Port Range | Source/Dest | Allow/Deny |
|---|---|---|---|---|---|
| 100 | Inbound | TCP | 80 | `0.0.0.0/0` | ALLOW |
| 110 | Inbound | TCP | 443 | `0.0.0.0/0` | ALLOW |
| 120 | Inbound | TCP | 22 | Admin IP | ALLOW |
| 130 | Inbound | TCP | 1024–65535 | `0.0.0.0/0` | ALLOW (ephemeral return traffic) |
| * | Inbound | All | All | `0.0.0.0/0` | DENY |
| 100 | Outbound | All | All | `0.0.0.0/0` | ALLOW |

### `private-nacl` — associated with private subnets

| Rule # | Type | Protocol | Port Range | Source/Dest | Allow/Deny |
|---|---|---|---|---|---|
| 100 | Inbound | TCP | 22 | `10.0.1.0/24`, `10.0.3.0/24` | ALLOW (from public subnets) |
| 110 | Inbound | TCP | 8080 | `10.0.1.0/24`, `10.0.3.0/24` | ALLOW |
| 120 | Inbound | TCP | 1024–65535 | `0.0.0.0/0` | ALLOW (ephemeral return traffic via NAT) |
| * | Inbound | All | All | `0.0.0.0/0` | DENY |
| 100 | Outbound | All | All | `0.0.0.0/0` | ALLOW |

> NACLs are stateless — return traffic must be explicitly allowed via the ephemeral port range rule.

---

## 8. EC2 Instances

| Instance | Subnet | Public IP | Security Group | Role |
|---|---|---|---|---|
| `public-web-01` | `public-subnet-a` | Yes | `public-sg` | Public-facing web/bastion host |
| `private-app-01` | `private-subnet-a` | No | `private-sg` | Internal application server |

---

## 9. Configuration Steps (Console / CLI Summary)

1. Create the VPC with CIDR `10.0.0.0/16`, enable DNS hostnames and resolution.
2. Create public and private subnets in each target AZ.
3. Create and attach the Internet Gateway to the VPC.
4. Allocate an Elastic IP and create the NAT Gateway in a public subnet.
5. Create `public-rt`, add the `0.0.0.0/0 → IGW` route, associate with public subnets.
6. Create `private-rt`, add the `0.0.0.0/0 → NAT Gateway` route, associate with private subnets.
7. Create `public-sg` and `private-sg` with the least-privilege rules above.
8. Create `public-nacl` and `private-nacl`, associate with respective subnets.
9. Launch EC2 instances into the appropriate subnets with the appropriate security groups.
10. Validate connectivity (see Section 10).

---

## 10. Validation Checklist

- [ ] Public EC2 instance is reachable via SSH/HTTP from the internet.
- [ ] Private EC2 instance has no public IP and is **not** reachable directly from the internet.
- [ ] Private EC2 instance can reach the internet (e.g. `curl https://example.com`) via the NAT Gateway.
- [ ] SSH from public EC2 (bastion) to private EC2 succeeds; direct SSH from the internet to private EC2 fails.
- [ ] Security group rules reject traffic outside the defined allow list.
- [ ] Route table associations match the intended public/private subnet design.

---

## 11. Cost Considerations

- **NAT Gateway** incurs an hourly charge plus per-GB data processing charges — the main recurring cost in this architecture.
- **Elastic IP** is free while associated with a running NAT Gateway/instance, but billed if left unattached.
- Consider a **NAT instance** instead of a NAT Gateway for lab/portfolio environments to reduce cost, with the trade-off of managing patching and availability yourself.
- **VPC Endpoints** (Gateway endpoints for S3/DynamoDB are free) can reduce NAT Gateway data processing costs for AWS-service traffic.

---

## 12. Next Steps

- Add a second NAT Gateway in `public-subnet-b` for AZ-resilient outbound routing.
- Enable **VPC Flow Logs** to CloudWatch Logs or S3 for traffic auditing.
- Migrate this configuration into **Terraform** or **AWS CloudFormation** for repeatable, version-controlled deployments.
- Add **VPC Endpoints** for S3/DynamoDB to keep AWS-service traffic off the NAT Gateway.

