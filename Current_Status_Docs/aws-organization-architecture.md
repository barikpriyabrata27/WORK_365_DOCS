# AWS Organization Architecture Document

Generated: 2026-10-13  
Account observed: `418195269698`  
Region observed: `us-east-1`  
Source files reviewed: `vpcs.json`, `subnets.json`, `route-tables.json`, `internet-gateways.json`, `nat-gateways.json`, `security-groups.json`, `network-acls.json`

## 1. Scope and Evidence

This document summarizes the AWS network and application-facing architecture visible from the provided inventory exports. The interpretation is read-only and based on the supplied JSON files plus the visible prior conversation context.

Important evidence boundary:

- The uploaded networking inventory was available and fully inspected.
- The prior conversation preview referenced EC2, S3, Lambda, and RDS investigation outputs, but those detailed result tables were not exposed in the available conversation preview. Where those service inventories are not visible, this document marks them as unverified instead of inventing details.
- Security group names and descriptions reference ECS, ALB, EFS, Valkey/Redis, Lambda, SQL Server, VPN, DICOM, CloudFront, and MedDream. Those references are treated as evidence of configured network intent, but not as proof that every related service is currently deployed unless the specific service inventory was provided.
- No load balancer, CloudFront distribution, ECS cluster/service, ECR, Route 53, IAM, VPC endpoint, Transit Gateway, VPN gateway detail, or CloudFormation stack inventory file was included. Those areas remain investigation gaps.

## 2. Executive Summary

The account contains two VPCs in `us-east-1`:

| VPC | CIDR | Default | Observed role |
|---|---:|---:|---|
| `vpc-0cd775e7bd52b90dd` | `10.0.0.0/16` | No | CDK-managed POC application network for `AWS-HealthImaging-DICOM-Viewer` / MedDream |
| `vpc-0d3b7a7372d3089e8` | `172.31.0.0/16` | Yes | Default VPC containing RadExcel-style resources and public/default subnets |

The CDK-managed application VPC is structured with public, private, and isolated subnet tiers across `us-east-1a` and `us-east-1b`. Public subnets route directly to an internet gateway. Private subnets route outbound through zonal NAT gateways and also receive a propagated route to `10.240.14.102/32` through virtual private gateway `vgw-051c0a8256e7ef95b`. Isolated subnets have local-only routing.

The default VPC has public default subnets across multiple Availability Zones and one explicitly private subnet named by association with `radexcel-private-rt`. The main route table has a default route to an internet gateway, while the private route table routes outbound through a NAT gateway.

Security groups indicate two likely application domains:

- A CDK-managed MedDream / AWS HealthImaging DICOM Viewer POC in the `10.0.0.0/16` VPC using ALB, ECS tasks, EFS, Redis/Valkey, DICOM port `11112`, and possible CloudFront origin integration.
- A RadExcel imaging environment in the default VPC using a VPN server, application server, SQL Server-style database access on `1433`, an ALB, an ECS Fargate API service, and a backend Lambda security group.

## 3. Organization and Tagging

The CDK-managed VPC and related resources consistently include these tags:

| Tag | Value |
|---|---|
| `Project` | `AWS-HealthImaging-DICOM-Viewer` |
| `Environment` | `POC-Development` |
| `Owner` | `PACS Management` |
| `ManagedBy` | `CDK` |
| `Service` | `Networking` |

The CloudFormation stack name visible on the network resources is:

`meddreampoc-NetworkNestedStackNetworkNestedStackResource0124E108-844YWOTFMZ6R`

This strongly indicates that the MedDream/HealthImaging network was provisioned through AWS CDK and CloudFormation. The RadExcel/default VPC resources are less consistently tagged; several security groups and the `RADEXCEL` subnet appear manually named or separately managed.

## 4. VPC Inventory

### 4.1 CDK Application VPC

| Field | Value |
|---|---|
| VPC ID | `vpc-0cd775e7bd52b90dd` |
| CIDR | `10.0.0.0/16` |
| Default VPC | No |
| State | `available` |
| DHCP options | `dopt-032f2410fb65921cd` |
| Internet gateway block mode | `off` |
| Name tag | `meddreampoc/Network/ApplicationVpc` |

### 4.2 Default VPC

| Field | Value |
|---|---|
| VPC ID | `vpc-0d3b7a7372d3089e8` |
| CIDR | `172.31.0.0/16` |
| Default VPC | Yes |
| State | `available` |
| DHCP options | `dopt-032f2410fb65921cd` |
| Internet gateway block mode | `off` |

## 5. Subnet Architecture

### 5.1 CDK Application VPC Subnets

| Subnet | Name | AZ | CIDR | Type | Public IP on launch | Available IPs |
|---|---|---|---:|---|---:|---:|
| `subnet-0ae524b7dccbc8803` | `meddreampoc/Network/ApplicationVpc/PublicSubnet1` | `us-east-1a` | `10.0.0.0/24` | Public | Yes | 248 |
| `subnet-0c6738c755a5d76aa` | `meddreampoc/Network/ApplicationVpc/PublicSubnet2` | `us-east-1b` | `10.0.1.0/24` | Public | Yes | 248 |
| `subnet-08ae75b62da9ec742` | `meddreampoc/Network/ApplicationVpc/PrivateSubnet1` | `us-east-1a` | `10.0.2.0/24` | Private | No | 247 |
| `subnet-00b7b142a4ed7fd84` | `meddreampoc/Network/ApplicationVpc/PrivateSubnet2` | `us-east-1b` | `10.0.3.0/24` | Private | No | 247 |
| `subnet-096878b935ebe3c1d` | `meddreampoc/Network/ApplicationVpc/IsolatedSubnet1` | `us-east-1a` | `10.0.4.0/24` | Isolated | No | 251 |
| `subnet-0354dc540c9c9deaa` | `meddreampoc/Network/ApplicationVpc/IsolatedSubnet2` | `us-east-1b` | `10.0.5.0/24` | Isolated | No | 251 |

Interpretation:

- Public subnets are suitable for internet-facing load balancers and public NAT gateways.
- Private subnets are suitable for application workloads requiring outbound internet access through NAT.
- Isolated subnets have no default internet route and are suitable for data stores or private-only dependencies.

### 5.2 Default VPC Subnets

| Subnet | Name | AZ | CIDR | Default for AZ | Public IP on launch | Available IPs |
|---|---|---|---:|---:|---:|---:|
| `subnet-06a24eaff2e459059` | Not tagged | `us-east-1a` | `172.31.0.0/20` | Yes | Yes | 4091 |
| `subnet-0168b71f6243569cb` | Not tagged | `us-east-1b` | `172.31.80.0/20` | Yes | Yes | 4091 |
| `subnet-094634f7e317c0230` | Not tagged | `us-east-1c` | `172.31.16.0/20` | Yes | Yes | 4091 |
| `subnet-0c3dab6c1e6eb100a` | Not tagged | `us-east-1d` | `172.31.32.0/20` | Yes | Yes | 4091 |
| `subnet-09c5553eeac452f88` | Not tagged | `us-east-1e` | `172.31.48.0/20` | Yes | Yes | 4091 |
| `subnet-07fc3b3bc97f287d4` | `RADEXCEL` | `us-east-1f` | `172.31.64.0/20` | Yes | Yes | 4086 |
| `subnet-040fdb2c363d96b9e` | Not tagged | `us-east-1f` | `172.31.96.0/24` | No | No | 251 |

Interpretation:

- Most default VPC subnets are public by default because they map public IPs on launch and inherit the main route table with an internet gateway default route.
- `subnet-040fdb2c363d96b9e` is associated with a private route table using a NAT gateway and does not map public IPs on launch.
- `subnet-07fc3b3bc97f287d4` is named `RADEXCEL` and hosts NAT gateway `nat-0cd43a72712c6c078`.

## 6. Internet Gateways and NAT Gateways

### 6.1 Internet Gateways

| Internet gateway | Attached VPC | Tags |
|---|---|---|
| `igw-0bf2cf658bb72326a` | `vpc-0cd775e7bd52b90dd` | CDK/MedDream application VPC tags |
| `igw-05ae6a7861f2e9173` | `vpc-0d3b7a7372d3089e8` | None |

### 6.2 NAT Gateways

| NAT gateway | VPC | Subnet | Private IP | Public IP | State | Created |
|---|---|---|---:|---:|---|---|
| `nat-0bf451b67a50c0669` | `vpc-0cd775e7bd52b90dd` | `subnet-0ae524b7dccbc8803` | `10.0.0.90` | `54.160.176.126` | `available` | `2026-08-22T08:15:24Z` |
| `nat-03cfe05bd7e922a8a` | `vpc-0cd775e7bd52b90dd` | `subnet-0c6738c755a5d76aa` | `10.0.1.106` | `3.87.177.69` | `available` | `2026-08-22T08:15:23Z` |
| `nat-0cd43a72712c6c078` | `vpc-0d3b7a7372d3089e8` | `subnet-07fc3b3bc97f287d4` | `172.31.71.213` | `34.232.100.165` | `available` | `2026-08-06T07:03:22Z` |

The CDK application VPC has two zonal NAT gateways, one in each public subnet. The default VPC has one NAT gateway in the `RADEXCEL` subnet.

## 7. Route Table Architecture

### 7.1 CDK Application VPC Route Tables

| Route table | Association | Key routes |
|---|---|---|
| `rtb-0964adb26eed92373` | `PublicSubnet1` / `subnet-0ae524b7dccbc8803` | `10.0.0.0/16 -> local`; `0.0.0.0/0 -> igw-0bf2cf658bb72326a` |
| `rtb-0ddf188c4dd5e8966` | `PublicSubnet2` / `subnet-0c6738c755a5d76aa` | `10.0.0.0/16 -> local`; `0.0.0.0/0 -> igw-0bf2cf658bb72326a` |
| `rtb-0ace22c30096a63e2` | `PrivateSubnet1` / `subnet-08ae75b62da9ec742` | `10.0.0.0/16 -> local`; `0.0.0.0/0 -> nat-0bf451b67a50c0669`; `10.240.14.102/32 -> vgw-051c0a8256e7ef95b` |
| `rtb-074d3dc828bbcc320` | `PrivateSubnet2` / `subnet-00b7b142a4ed7fd84` | `10.0.0.0/16 -> local`; `0.0.0.0/0 -> nat-03cfe05bd7e922a8a`; `10.240.14.102/32 -> vgw-051c0a8256e7ef95b` |
| `rtb-031018a68fddc2ef9` | `IsolatedSubnet1` / `subnet-096878b935ebe3c1d` | `10.0.0.0/16 -> local` only |
| `rtb-02dfd835dc5ee6826` | `IsolatedSubnet2` / `subnet-0354dc540c9c9deaa` | `10.0.0.0/16 -> local` only |
| `rtb-052392b37fafa8353` | Main route table | `10.0.0.0/16 -> local` only |

Key observation:

- Private subnets have virtual private gateway route propagation enabled from `vgw-051c0a8256e7ef95b`, with an active propagated host route to `10.240.14.102/32`. The purpose of this remote host is not identified in the supplied files.

### 7.2 Default VPC Route Tables

| Route table | Association | Key routes |
|---|---|---|
| `rtb-07693896a99e0646e` | Main route table | `172.31.0.0/16 -> local`; `0.0.0.0/0 -> igw-05ae6a7861f2e9173` |
| `rtb-06c134e4799e44c9f` | `subnet-040fdb2c363d96b9e` | `172.31.0.0/16 -> local`; `0.0.0.0/0 -> nat-0cd43a72712c6c078` |

## 8. Network ACLs

Both VPCs use default network ACLs.

| NACL | VPC | Associated subnets | Rules |
|---|---|---:|---|
| `acl-0767fd47e70d19876` | `vpc-0cd775e7bd52b90dd` | 6 | Allow all inbound and outbound at rule `100`; default deny at `32767` |
| `acl-054f98a5fd314e2e5` | `vpc-0d3b7a7372d3089e8` | 7 | Allow all inbound and outbound at rule `100`; default deny at `32767` |

Interpretation:

- Subnet-level filtering is permissive. Effective network restrictions are primarily enforced through security groups.

## 9. Security Group Inventory and Traffic Intent

### 9.1 CDK Application VPC Security Groups

| Security group | Purpose from name/description | Inbound summary | Egress summary |
|---|---|---|---|
| `sg-0fbbb9c2e8c420d20` | Application Load Balancer | TCP `80` from `0.0.0.0/0`, CloudFront prefix list `pl-3b927c52`, and ECS SG `sg-01b0fa25f5032859c` | TCP `3000`, `8080`, `8088` to ECS SG |
| `sg-01b0fa25f5032859c` | ECS tasks | TCP `8080` from ALB SG and `10.0.0.0/16`; TCP `3000` and `8088` from ALB SG; TCP `11112` from specific external/internal IPs; self-referencing all traffic | All outbound to `0.0.0.0/0` |
| `sg-008bf163bd5dc2a7a` | EFS mount targets | TCP `2049` from ECS SG | Non-standard egress rule described as "Disallow all traffic" to `255.255.255.255/32` |
| `sg-0123bb92a8e968cbf` | Redis/Valkey | TCP `6379` from ECS SG | All outbound to `0.0.0.0/0` |
| `sg-0aef8b9b10db5130d` | Default SG for application VPC | Self-referencing all traffic | All outbound to `0.0.0.0/0` |

Application inference from security groups:

- An ALB fronts ECS tasks.
- ECS task ports include `3000`, `8080`, and `8088`.
- EFS is mounted by ECS tasks over NFS port `2049`.
- Redis/Valkey is reachable from ECS tasks on port `6379`.
- DICOM/SCP traffic appears to use TCP `11112`, with allowed sources including `208.87.85.190/32`, `10.0.0.0/16`, `14.98.215.102/32`, `106.202.6.245/32`, and `10.240.14.102/32`.
- CloudFront is suggested by prefix list `pl-3b927c52`, but actual CloudFront distribution configuration was not provided.

### 9.2 Default VPC / RadExcel Security Groups

| Security group | Purpose from name/description | Inbound summary | Egress summary |
|---|---|---|---|
| `sg-0269b2ed3999134bd` | `RADXL_VPNSVR` | TCP `443`, `3389`, `5555`, `7281` from `0.0.0.0/0` | All outbound to `0.0.0.0/0` |
| `sg-017da45a307ea2d09` | `RADXL_HImgAppSvr` | TCP `80`, `104`, `443` from `0.0.0.0/0`; TCP `7281` and `3389` from `172.31.64.0/20` | All outbound to `0.0.0.0/0` |
| `sg-0dfbf2270fa0074df` | `RADXL_HImgDB` | TCP `1433` from app server SG, ECS API SG, Lambda SG, and `172.31.64.0/20`; TCP `443` and `3389` from `172.31.64.0/20` | All outbound to `0.0.0.0/0` |
| `sg-076c3c5e513de7d82` | `radexcel-poc-imaging-alb` | TCP `80` and `443` from `0.0.0.0/0` | All outbound to `0.0.0.0/0` |
| `sg-084702978384200db` | `radexcel-poc-imaging-api` ECS Fargate service | TCP `8080` from ALB SG | All outbound to `0.0.0.0/0` |
| `sg-0959e723470ed1e81` | `radexcel-lambda-sg` backend Lambda | No inbound | All outbound to `0.0.0.0/0` |
| `sg-0403591b4cab4bb97` | CloudFront VPC Origins service SG | No inbound | All outbound to `0.0.0.0/0` |
| `sg-0ee02a484f0afe25e` | Default SG for default VPC | Self-referencing all traffic | All outbound to `0.0.0.0/0` |

Application inference from security groups:

- A public-facing ALB likely receives HTTP/HTTPS traffic for a RadExcel imaging application.
- The ALB security group is allowed to reach an ECS Fargate API service on TCP `8080`.
- The API service and Lambda security group are allowed to reach a database security group on SQL Server port `1433`.
- The app server security group allows HTTP, HTTPS, and DICOM port `104` from the internet.
- A VPN/server security group exposes ports commonly associated with HTTPS, remote administration, or application-specific services.
- CloudFront VPC Origins is suggested by the `CloudFront-VPCOrigins-Service-SG` name and `aws.cloudfront.vpcorigin=enabled` tag, but the actual CloudFront distribution was not provided.

## 10. EC2 Inventory

The visible prior conversation includes the AWS CLI command that was intended to export EC2 inventory for profile `freelance` in `us-east-1`, but the actual EC2 table output is not present in the accessible conversation preview.

Visible command intent:

```powershell
aws ec2 describe-instances `
  --region us-east-1 `
  --profile freelance `
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,ID:InstanceId,Type:InstanceType,State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}" `
  --output table
```

Document status:

- EC2 instances: unverified in the accessible source material.
- Likely EC2-related roles from security group names: `RADXL_VPNSVR`, `RADXL_HImgAppSvr`, and `RADXL_HImgDB`.
- The exact instance IDs, instance types, states, private IPs, and public IPs should be confirmed from the EC2 export before this section is treated as complete.

## 11. S3 Inventory

The detailed S3 bucket output referenced by the prior conversation is not present in the accessible source material. No S3 bucket export file was included with the networking JSON files.

Document status:

- S3 buckets: unverified.
- Bucket policies, public access block settings, encryption, versioning, lifecycle rules, and application use: unverified.

## 12. Lambda Inventory

The detailed Lambda output referenced by the prior conversation is not present in the accessible source material. However, one security group named `radexcel-lambda-sg` is present.

Confirmed from networking inventory:

- Security group `sg-0959e723470ed1e81` is named `radexcel-lambda-sg`.
- It has no inbound rules.
- It allows all outbound traffic to `0.0.0.0/0`.
- The database security group `sg-0dfbf2270fa0074df` allows SQL Server TCP `1433` from `sg-0959e723470ed1e81`.

Inference:

- A backend Lambda function or functions likely run in the default VPC and connect to the SQL Server/database tier.

Unverified:

- Function names, runtimes, VPC subnet placement, environment variables, IAM roles, triggers, and concurrency settings.

## 13. RDS / Database Result

The detailed RDS output referenced by the prior conversation is not present in the accessible source material. No RDS export file was included.

Confirmed from networking inventory:

- Security group `sg-0dfbf2270fa0074df` is named `RADXL_HImgDB`.
- Inbound SQL Server TCP `1433` is allowed from:
  - `sg-017da45a307ea2d09` (`RADXL_HImgAppSvr`)
  - `sg-084702978384200db` (`radexcel-poc-imaging-api`)
  - `sg-0959e723470ed1e81` (`radexcel-lambda-sg`)
  - CIDR `172.31.64.0/20`

Interpretation:

- The database tier appears to be SQL Server-compatible based on TCP `1433`.
- It may be an EC2-hosted database or RDS SQL Server; the supplied files do not confirm which.

Unverified:

- RDS DB identifier, engine, instance class, endpoint, subnet group, Multi-AZ status, backups, encryption, and public accessibility.

## 14. Possible Deployment Architecture

### 14.1 MedDream / AWS HealthImaging DICOM Viewer POC

Confirmed network facts:

- Runs in CDK-managed VPC `vpc-0cd775e7bd52b90dd`.
- Uses two public, two private, and two isolated subnets across `us-east-1a` and `us-east-1b`.
- Public subnets route to `igw-0bf2cf658bb72326a`.
- Private subnets route through zonal NAT gateways.
- Private subnets also have a propagated route to `10.240.14.102/32` through `vgw-051c0a8256e7ef95b`.
- Security groups define ALB, ECS task, EFS, and Redis/Valkey roles.

Likely flow based on security groups:

```text
Internet / CloudFront prefix list
        |
        v
Application Load Balancer security group sg-0fbbb9c2e8c420d20
        |
        | TCP 3000 / 8080 / 8088
        v
ECS task security group sg-01b0fa25f5032859c
        |
        +--> EFS security group sg-008bf163bd5dc2a7a on TCP 2049
        +--> Redis/Valkey security group sg-0123bb92a8e968cbf on TCP 6379
        +--> DICOM/SCP TCP 11112 from specific external/internal sources
```

Unverified:

- Actual ALB ARN/DNS name, listener certificates, target groups, ECS cluster/service names, task definitions, container images, EFS file system ID, Redis/Valkey cluster details, and CloudFront distribution details.

### 14.2 RadExcel Imaging Environment

Confirmed network facts:

- Runs in default VPC `vpc-0d3b7a7372d3089e8`.
- Has public default subnets across multiple AZs and one private subnet using NAT.
- Security groups define VPN, app server, database, ALB, ECS Fargate API, Lambda backend, and CloudFront VPC Origins roles.

Likely flow based on security groups:

```text
Internet
  |
  +--> radexcel-poc-imaging-alb sg-076c3c5e513de7d82 on TCP 80/443
          |
          v
      radexcel-poc-imaging-api sg-084702978384200db on TCP 8080
          |
          v
      RADXL_HImgDB sg-0dfbf2270fa0074df on TCP 1433

RADXL_HImgAppSvr sg-017da45a307ea2d09
  +--> Exposes TCP 80/104/443 from internet
  +--> Can reach database on TCP 1433

radexcel-lambda-sg sg-0959e723470ed1e81
  +--> Can reach database on TCP 1433
```

Unverified:

- Whether the app server and database are EC2 instances or managed services.
- Actual ALB, ECS, Lambda, CloudFront, and database resource inventory.
- DNS, certificate, WAF, backup, monitoring, and logging configuration.

## 15. Security Observations for Review

These are observations for client review only. They are not change recommendations and should not be altered without approval from the client or system owner.

| Priority | Observation | Evidence | Risk to review |
|---|---|---|---|
| High | Public RDP exposure on VPN/server SG | `sg-0269b2ed3999134bd` allows TCP `3389` from `0.0.0.0/0` | Internet-wide RDP exposure is commonly high risk and should be validated as intentional |
| High | Public custom ports on VPN/server SG | `sg-0269b2ed3999134bd` allows TCP `5555` and `7281` from `0.0.0.0/0` | Public administrative or application ports should be confirmed and documented |
| Medium | Public DICOM port on app server | `sg-017da45a307ea2d09` allows TCP `104` from `0.0.0.0/0` | DICOM exposure should be confirmed against business and partner connectivity needs |
| Medium | Public HTTP/HTTPS to app server | `sg-017da45a307ea2d09` allows TCP `80` and `443` from `0.0.0.0/0` | Confirm whether direct instance exposure is intended or should be via a load balancer |
| Medium | Public HTTP to MedDream ALB SG | `sg-0fbbb9c2e8c420d20` allows TCP `80` from `0.0.0.0/0` | Confirm intended origin access model, especially if CloudFront is expected to front the application |
| Medium | Default NACLs allow all | Both NACLs allow all inbound and outbound at rule `100` | Not inherently wrong, but means security groups carry most enforcement responsibility |
| Medium | Broad outbound egress | Many security groups allow all outbound to `0.0.0.0/0` | Common default, but should be reviewed for regulated workloads and data egress requirements |
| Low | Default VPC still in active use | Multiple RadExcel resources appear in default VPC | Default VPC use can be acceptable, but it should be documented and governed |
| Low | Mixed infrastructure management patterns | CDK-managed MedDream resources plus manually named/default VPC resources | Ownership and drift control should be clarified |

Positive controls observed:

- The CDK application VPC separates public, private, and isolated subnet tiers.
- Private application subnets use NAT gateways rather than direct public routing.
- Database access is scoped mainly by source security group for API, Lambda, and app server paths.
- EFS and Redis/Valkey access is limited to the ECS task security group.
- DICOM TCP `11112` in the CDK VPC is restricted to specific source IPs/CIDRs rather than fully open.

## 16. Gaps and Unknowns

The following items cannot be confirmed from the supplied inventory files:

- EC2 instance list, names, public IPs, private IPs, and attached security groups.
- S3 bucket list, public access posture, encryption, policies, and lifecycle.
- Lambda function list, triggers, runtime, VPC attachment, and environment variables.
- RDS/database inventory and whether `RADXL_HImgDB` maps to EC2 SQL Server or RDS SQL Server.
- Load balancer inventory, listeners, target groups, health checks, and certificates.
- CloudFront distributions, origins, cache policies, WAF, and TLS settings.
- ECS clusters, services, task definitions, container ports, image repositories, service autoscaling, and deployment strategy.
- VPC endpoints, private hosted zones, Route 53 records, ACM certificates, WAF, CloudWatch logs, Config, GuardDuty, CloudTrail, and IAM roles.
- VPN customer gateway / site-to-site VPN details for `vgw-051c0a8256e7ef95b`.

## 17. Recommended Next Investigation Commands

Run these as read-only commands in the same account/profile and region to complete the architecture record. Use the appropriate AWS profile, shown here as `freelance`, and adjust only if the active profile differs.

### EC2

```powershell
aws ec2 describe-instances `
  --region us-east-1 `
  --profile freelance `
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,ID:InstanceId,Type:InstanceType,State:State.Name,VpcId:VpcId,SubnetId:SubnetId,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress,SecurityGroups:SecurityGroups[].GroupId}" `
  --output table
```

### Load Balancers and Target Groups

```powershell
aws elbv2 describe-load-balancers --region us-east-1 --profile freelance --output json
aws elbv2 describe-target-groups --region us-east-1 --profile freelance --output json
aws elbv2 describe-listeners --region us-east-1 --profile freelance --load-balancer-arn <load-balancer-arn> --output json
```

### ECS

```powershell
aws ecs list-clusters --region us-east-1 --profile freelance --output json
aws ecs list-services --region us-east-1 --profile freelance --cluster <cluster-arn> --output json
aws ecs describe-services --region us-east-1 --profile freelance --cluster <cluster-arn> --services <service-arn> --output json
aws ecs describe-task-definition --region us-east-1 --profile freelance --task-definition <task-definition-arn> --output json
```

### Lambda

```powershell
aws lambda list-functions --region us-east-1 --profile freelance --output json
aws lambda get-function-configuration --region us-east-1 --profile freelance --function-name <function-name> --output json
```

### RDS

```powershell
aws rds describe-db-instances --region us-east-1 --profile freelance --output json
aws rds describe-db-subnet-groups --region us-east-1 --profile freelance --output json
```

### S3

```powershell
aws s3api list-buckets --profile freelance --output json
aws s3api get-public-access-block --profile freelance --bucket <bucket-name> --output json
aws s3api get-bucket-encryption --profile freelance --bucket <bucket-name> --output json
aws s3api get-bucket-versioning --profile freelance --bucket <bucket-name> --output json
```

### CloudFront

```powershell
aws cloudfront list-distributions --profile freelance --output json
aws cloudfront get-distribution-config --profile freelance --id <distribution-id> --output json
```

### VPN and Hybrid Connectivity

```powershell
aws ec2 describe-vpn-gateways --region us-east-1 --profile freelance --output json
aws ec2 describe-vpn-connections --region us-east-1 --profile freelance --output json
aws ec2 describe-customer-gateways --region us-east-1 --profile freelance --output json
```

### CloudFormation Stacks

```powershell
aws cloudformation describe-stacks --region us-east-1 --profile freelance --output json
aws cloudformation list-stack-resources --region us-east-1 --profile freelance --stack-name <stack-name> --output json
```

## 18. Conclusion

The provided inventory shows a partially formalized AWS architecture: a CDK-managed MedDream/HealthImaging POC network with clean subnet tiering, NAT, ALB/ECS/EFS/Redis-style security group relationships, and DICOM connectivity; alongside an active default VPC hosting RadExcel resources including ALB/API/Lambda/database/VPN/app-server security groups.

The strongest confirmed design element is the network layer. The main remaining work is to attach concrete service inventories for EC2, S3, Lambda, RDS, ALB, ECS, and CloudFront so the architecture can be promoted from a network/security-group interpretation into a complete end-to-end application architecture document.
