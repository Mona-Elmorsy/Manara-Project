# Scalable Web Application with ALB and Auto Scaling

**AWS Solutions Architect – Associate Graduation Project**

A secure, highly available, multi-tier web application architecture on AWS, designed around a public edge layer, a private compute tier, and a Multi-AZ data layer. The design spans two Availability Zones, separates public and private resources correctly, and removes direct SSH exposure to the internet in favor of IAM-based access.

![Architecture Diagram](./architecture-diagram.svg)

---

## About This Project

This repository documents the architecture design for the **"Scalable Web Application with ALB and Auto Scaling"** graduation project (EC2-based track). The deliverable for this project is the **architecture itself** — a diagram plus the reasoning behind every decision in it — rather than an implementation, so this repository contains no infrastructure code. Everything below walks through what is in the diagram and why it is built that way.

---

## Architecture Walkthrough

### 1. Edge layer — Route 53 → CloudFront → WAF
A request first resolves through **Route 53** (DNS, with health checks so DNS can react to failures). It then hits **CloudFront**, which caches static assets at edge locations to cut latency for users far from the origin region. Before any request reaches application infrastructure, **AWS WAF** filters it against OWASP Top 10-style rules (SQL injection, XSS, bad bots), so malicious traffic is rejected at the edge rather than inside the VPC.

### 2. Load balancing — Application Load Balancer
The **ALB** sits in the public subnets and is the single entry point into the VPC. It terminates Layer 7 traffic and distributes it across healthy EC2 targets in both Availability Zones, using target group health checks to automatically stop routing to an unhealthy instance.

### 3. Network design — VPC across two Availability Zones
The VPC (`10.0.0.0/16`) is split into:
- **Public subnets** (one per AZ) — host only the NAT Gateways. No application server lives here, so nothing compute-related is directly internet-facing.
- **Private subnets** (one per AZ) — host the application tier. Outbound internet access (for patches, package installs) goes through the NAT Gateway in the same AZ; nothing can initiate a connection in from the internet.

Spreading both subnet types across two AZs is what makes the architecture *highly available*: the loss of a single AZ still leaves a working public subnet, private subnet, and database replica.

### 4. Compute — EC2 + Auto Scaling Group
Application servers run as an **Auto Scaling Group** of EC2 instances spread across both private subnets. The ASG adds or removes instances based on demand (target tracking), so the app scales out under load and scales in to save cost when idle, without manual intervention.

### 5. Data layer — RDS (Multi-AZ)
The database is **RDS in Multi-AZ mode**: a primary instance in one AZ, synchronously replicated to a standby in the other. Both Auto Scaling Groups talk to **one single endpoint only — the primary**. The standby is never queried directly by the application; if the primary fails, AWS re-points the same endpoint to the standby automatically. This is the correct way to consume RDS Multi-AZ — the failover mechanism is designed around a single, stable connection endpoint, not an app that chooses between two databases itself.

### 6. Instance access — Systems Manager Session Manager (no bastion, no SSH)
Instead of a bastion/jump host with an open SSH port, administrative access to the EC2 instances goes through **AWS Systems Manager Session Manager**. No inbound port is opened for SSH anywhere in this design — instances only need outbound connectivity to the SSM endpoints (through the NAT Gateway), and every session is authorized by IAM and logged. This removes an entire class of attack surface (an exposed bastion host) that a traditional jump-host design would otherwise introduce.

### 7. Observability — CloudWatch + SNS
**CloudWatch** collects metrics, logs, and alarms across the ALB and EC2 fleet. When a threshold is crossed (e.g. unhealthy host count, high latency), CloudWatch triggers **SNS** to notify the team — so failures are surfaced automatically instead of being discovered by users first.

### 8. Storage — S3
Static assets and application files are stored in **S3**, served to end users through CloudFront rather than directly, keeping static content delivery off the application servers entirely.

---

## Key Design Decisions

**Why Systems Manager Session Manager instead of a bastion/jump host?**
A bastion host still means an open SSH port reachable from the internet (even if IP-restricted), plus one more instance to patch, monitor, and secure. Session Manager removes that attack surface entirely: no inbound port is ever opened for admin access, and every session is authenticated and authorized through IAM with a full audit trail in CloudTrail. This directly reflects the "bastion-free access alternative" called out as a learning outcome for this project track.

**Why a single RDS endpoint instead of connecting to both Primary and Standby?**
In RDS Multi-AZ, the application should only ever talk to the **primary endpoint**. The standby exists purely for synchronous replication and automatic failover — AWS re-points the same DNS endpoint to the standby if the primary fails. Having application servers connect to the standby directly would bypass how Multi-AZ failover is actually designed to work, and could read stale or inconsistent data outside of a real failover event.

**Why WAF in front of the ALB rather than only relying on security groups?**
Security groups filter by IP/port — they can't inspect request content. WAF adds request-level filtering (SQL injection patterns, XSS payloads, rate-based rules) that a security group simply cannot express, closing a gap that pure network-layer controls leave open.

---

## Key AWS Services

| Layer | Service | Purpose |
|---|---|---|
| DNS & CDN | Route 53 | DNS routing and health checks |
| | CloudFront | Caches static assets, reduces latency globally |
| Edge Security | AWS WAF | Filters traffic against OWASP Top 10 rules |
| Compute | Application Load Balancer | Layer 7 routing across Availability Zones |
| | EC2 + Auto Scaling Group | Application servers, scaled by demand |
| Access | Systems Manager Session Manager | Secure, IAM-based instance access (no SSH) |
| Networking | VPC | Public/private subnets, route tables, NAT Gateways |
| Data | RDS (Multi-AZ) | Primary/standby database with automatic failover |
| Storage | S3 | Static assets and application files |
| Observability | CloudWatch + SNS | Monitoring, dashboards, and alerts |

---

## Security Group Design

| Security Group | Inbound | Outbound |
|---|---|---|
| ALB SG | HTTP/HTTPS (80/443) from `0.0.0.0/0` | Allow all |
| EC2 SG | HTTP (80) from ALB SG only — no SSH from the internet | Allow all (NAT, RDS, SSM endpoints) |
| RDS SG | DB port (3306/5432) from EC2 SG only | Allow all |

Each layer can only be reached from the layer directly in front of it — the database is unreachable from anywhere except the application tier, and the application tier is unreachable from anywhere except the load balancer.

---

## Traffic Flow

1. `Internet → Route 53 → CloudFront → WAF → ALB`
2. `ALB → EC2 (HTTP 80)` — load balanced across both Availability Zones
3. `EC2 → RDS primary endpoint` — both AZs read/write through the same endpoint
4. `RDS Primary → Standby` — synchronous replication (AWS-managed, not app-facing)
5. `Session Manager → EC2` — admin access, no inbound SSH required
6. `CloudFront → S3` — static asset delivery

---

## AWS Services Used

`Route 53` · `CloudFront` · `AWS WAF` · `Application Load Balancer` · `EC2` · `Auto Scaling` · `VPC` (subnets, route tables, NAT Gateway) · `RDS (Multi-AZ)` · `S3` · `Systems Manager (Session Manager)` · `CloudWatch` · `SNS` · `IAM`

---

## Learning Outcomes

- Designing a VPC with correct public/private subnet separation and routing
- Building highly available architectures across multiple Availability Zones
- Configuring ALB listener rules and target group health checks
- Implementing Auto Scaling with target tracking policies
- Replacing bastion-host SSH access with Systems Manager Session Manager
- Designing least-privilege security groups between network layers
- Understanding RDS Multi-AZ failover and why the application only ever talks to the primary endpoint
- Applying WAF as a request-level filtering layer in front of the load balancer
