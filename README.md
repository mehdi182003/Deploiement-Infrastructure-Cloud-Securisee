# ☁️ AWS Cloud Infrastructure — VPC & Multi-Tier Architecture (IaC)

> Infrastructure-as-Code (IaC) deployment of a production-grade AWS network architecture using CloudFormation — VPC, public/private subnets, NAT instance, Jumpbox, auto-scaling EC2 instances, and a 3-tier application stack.

---

##  Project Overview

This lab deploys a complete, modular AWS infrastructure entirely through **CloudFormation templates** (YAML). The architecture follows AWS best practices for network isolation, security group layering, and multi-tier application deployment.

Completed as part of the **GTI778 Cloud Infrastructure course** at ÉTS Montréal (Winter 2026).

---

##  Architecture

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
┌─────────────────────────────────┐
│           VPC (configurable CIDR)│
│                                 │
│  ┌──────────────┐               │
│  │ Public Subnet│               │
│  │  - Jumpbox   │               │
│  │  - NAT       │               │
│  └──────┬───────┘               │
│         │ NAT                   │
│  ┌──────▼───────────────────┐   │
│  │     Private Subnets      │   │
│  │  - Web Servers (ASG)     │   │
│  │  - License Server        │   │
│  │  - Database (MariaDB)    │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

---

##  Stack Components

### Template 1 — Network Layer (`vpc-lab2.yaml`)
Provisions the foundational network infrastructure:

| Resource | Details |
|----------|---------|
| VPC | Configurable CIDR (default: `10.0.0.0/16`) |
| Internet Gateway | Attached to VPC |
| Public Subnet | Jumpbox + NAT instance |
| Private Subnets (×2) | Isolated compute + data tiers |
| NAT Instance | Outbound internet for private resources |
| Jumpbox Security Group | SSH access control for private instances |
| Route Tables | Public/private traffic routing |

**Parameters**: `ProjectName`, `EnvironmentType` (Test/Prod), `VpcCidr`, `InstanceBits`, `KeyPairName`

**Condition**: `IsProd` — enables production-specific resource configuration

### Template 2 — Application Layer (`serveurs-menu-graphique.yaml`)
Deploys the 3-tier application on top of the network stack:

| Resource | Details |
|----------|---------|
| Web Servers | EC2 (t2.micro), ports 80/8080/8081, public-facing |
| License Server | EC2, port 9090, isolated SG |
| Database (MariaDB) | EC2 (t2.micro), custom port 3360, private subnet only |
| Security Groups | Strict layer-to-layer access rules |
| S3 Install Bucket | Software provisioning via UserData scripts |

**Cross-stack references**: Uses `!ImportValue` to consume VPC/subnet outputs from Template 1.

### Template 3 — NAT Nested Stack (`vpc-lab-2-NatNestedStack.yaml`)
Modular NAT instance configuration deployed as a nested CloudFormation stack.

---

##  Security Design

- **Jumpbox pattern**: Only the Jumpbox SG has SSH access to private instances — no direct public SSH
- **DB isolation**: MariaDB port 3360 only accessible from WebSG — never from public internet
- **License server**: Port 9090 restricted, SSH only via Jumpbox
- **Least privilege**: Each tier's SG only allows traffic from the tier that needs it

---

##  Tech Stack

- **AWS CloudFormation** (YAML) — Infrastructure as Code
- **AWS Services**: EC2, VPC, Subnets, IGW, Route Tables, Security Groups, S3, IAM
- **OS/Software**: Amazon Linux 2023, MariaDB 10.5, UserData bash provisioning
- **Course**: GTI778 — Cloud Infrastructure, ÉTS Montréal

---

##  Project Structure

```
├── vpc-lab2.txt                        # Template 1: Network layer (VPC, subnets, NAT, Jumpbox)
├── vpc-lab-2-NatNestedStack.txt        # Template 2: NAT nested stack
├── serveurs-menu-graphique.txt         # Template 3: Application layer (web, license, DB)
└── README.md
```



##  Deployment Order

```bash
# 1. Deploy network stack first
aws cloudformation deploy \
  --template-file vpc-lab2.yaml \
  --stack-name vpc-lab-2 \
  --parameter-overrides ProjectName=myproject EnvironmentType=Prod

# 2. Deploy application stack (depends on network stack outputs)
aws cloudformation deploy \
  --template-file serveurs-menu-graphique.yaml \
  --stack-name app-layer \
  --parameter-overrides NetworkStackName=vpc-lab-2
```

---

##  Key Concepts Demonstrated

- **Modular IaC**: separation of network and application concerns across stacks
- **Cross-stack references**: `Outputs` / `!ImportValue` pattern for loose coupling
- **Security group chaining**: tiered access control without hardcoded IPs
- **UserData provisioning**: automated software installation at instance launch
- **Nested stacks**: reusable NAT module via `AWS::CloudFormation::Stack`
