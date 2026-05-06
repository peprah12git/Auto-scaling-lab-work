# Auto-Scaling Web Tier on AWS

A highly available, auto-scaling web application tier built with **Amazon EC2 Auto Scaling** and an **Application Load Balancer (ALB)**, provisioned entirely through AWS CloudFormation and deployed via **CloudFormation GitSync**.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Parameters](#parameters)
- [Deployment](#deployment)
- [Accessing the Application](#accessing-the-application)
- [Demonstrating Auto Scaling](#demonstrating-auto-scaling)
- [Outputs](#outputs)
- [Security Design](#security-design)
- [Cost Considerations](#cost-considerations)

---

## Architecture Overview

```
                          Internet
                             │
                    ┌────────▼────────┐
                    │  Internet Gateway │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │   Application Load Balancer  │
              │  (Public Subnets – 2 AZs)   │
              └──────────────┬──────────────┘
                             │  Round-robin HTTP
              ┌──────────────▼──────────────┐
              │      Auto Scaling Group      │
              │  EC2 instances (Apache)      │
              │  (Private Subnets – 2 AZs)  │
              └──────────────┬──────────────┘
                             │
                    ┌────────▼────────┐
                    │   NAT Gateway    │
                    │  (Public Subnet) │
                    └─────────────────┘
```

| Component | Detail |
|---|---|
| VPC CIDR | `10.0.0.0/16` |
| Public Subnets | `10.0.1.0/24` (AZ-1), `10.0.2.0/24` (AZ-2) |
| Private Subnets | `10.0.11.0/24` (AZ-1), `10.0.12.0/24` (AZ-2) |
| Load Balancer | Internet-facing ALB across both public subnets |
| EC2 Instances | Amazon Linux 2, `t3.micro`, Apache HTTP Server |
| ASG Capacity | Min: 1 · Desired: 1 · Max: 4 |
| Scale-out trigger | Average CPU utilisation > 30% |
| Internet egress | Regional NAT Gateway in Public Subnet 1 |

---

## Repository Structure

```
├── template.yaml       # CloudFormation template (all infrastructure resources)
├── deployment.yaml     # CloudFormation GitSync deployment configuration
└── README.md
```

### `template.yaml`

Single CloudFormation template that provisions:

- VPC, Internet Gateway, and gateway attachment
- 2 public subnets and 2 private subnets across 2 Availability Zones
- Public and private route tables with appropriate routes
- Elastic IP and NAT Gateway (placed in Public Subnet 1)
- ALB security group (allows HTTP from `0.0.0.0/0`)
- EC2 security group (allows HTTP from ALB security group only — no SSH)
- Internet-facing Application Load Balancer
- Target Group with HTTP health checks on `/`
- ALB Listener on port 80
- IAM Role and Instance Profile with `AmazonSSMManagedInstanceCore`
- Launch Template using the latest Amazon Linux 2 AMI (resolved via SSM Parameter Store)
- Auto Scaling Group spanning both private subnets
- Target Tracking Scaling Policy (CPU-based)

### `deployment.yaml`

GitSync deployment manifest that binds the CloudFormation stack to this repository:

```yaml
stack-name: autoscaling-demo
template-file-path: ./template.yaml
```

---

## Prerequisites

| Requirement | Notes |
|---|---|
| AWS Account | With permissions to create VPC, EC2, ELB, IAM, and CloudFormation resources |
| AWS CLI | Version 2.x or later |
| CloudFormation GitSync | GitHub repository connected to CloudFormation via GitSync |
| Git | For pushing changes that trigger automated deployments |

---

## Parameters

All parameters are pre-configured in `deployment.yaml` and can be overridden at deployment time.

| Parameter | Default | Description |
|---|---|---|
| `ProjectName` | `autoscaling-labWorkshop` | Prefix applied to all resource names |
| `VpcCidr` | `10.0.0.0/16` | CIDR block for the VPC |
| `PublicSubnet1Cidr` | `10.0.1.0/24` | Public subnet in AZ-1 |
| `PublicSubnet2Cidr` | `10.0.2.0/24` | Public subnet in AZ-2 |
| `PrivateSubnet1Cidr` | `10.0.11.0/24` | Private subnet in AZ-1 |
| `PrivateSubnet2Cidr` | `10.0.12.0/24` | Private subnet in AZ-2 |
| `InstanceType` | `t3.micro` | EC2 instance type (`t2.micro`, `t3.micro`, `t3.small`) |
| `MinInstances` | `1` | ASG minimum capacity |
| `DesiredInstances` | `1` | ASG desired capacity |
| `MaxInstances` | `4` | ASG maximum capacity |
| `CpuScaleOutThreshold` | `30` | CPU % threshold that triggers scale-out |

---

## Deployment

This project uses **CloudFormation GitSync**. Once the GitSync connection is configured, every push to the `main` branch automatically creates or updates the CloudFormation stack.

### Initial Setup (one-time)

1. Fork or clone this repository to your GitHub account.
2. In the AWS Console, navigate to **CloudFormation → Stacks → Create stack → With GitSync**.
3. Connect your GitHub repository and point the deployment file to `deployment.yaml`.
4. CloudFormation will read `deployment.yaml` to locate `template.yaml` and deploy the stack.

### Subsequent Changes

```bash
# Edit template.yaml or deployment.yaml, then push
git add .
git commit -m "describe your change"
git push origin main
```

GitSync detects the push and automatically applies the changes to the `autoscaling-demo` stack.

### Manual Deployment (CLI alternative)

```bash
aws cloudformation deploy \
  --stack-name autoscaling-demo \
  --template-file template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
      ProjectName=autoscaling-labWorkshop \
      InstanceType=t3.micro \
      MinInstances=1 \
      DesiredInstances=1 \
      MaxInstances=4 \
      CpuScaleOutThreshold=30
```

---

## Accessing the Application

Once the stack reaches `CREATE_COMPLETE` or `UPDATE_COMPLETE`:

1. In the AWS Console, go to **CloudFormation → Stacks → autoscaling-demo → Outputs**.
2. Copy the value of `AlbDnsName` (e.g., `http://autoscaling-labWorkshop-alb-xxxxxxxx.eu-west-1.elb.amazonaws.com`).
3. Open the URL in a browser.

The page displays:

| Field | Description |
|---|---|
| Instance ID | The EC2 instance currently serving the request |
| Private IP | The private IP of that instance |
| Availability Zone | The AZ the instance resides in |

The page auto-refreshes every 5 seconds. As traffic is round-robined across instances, the **Instance ID will change** with each refresh, confirming that the ALB is distributing requests correctly.

---

## Demonstrating Auto Scaling

### Trigger a CPU Spike

1. Open the application URL in a browser.
2. Click the **"Trigger CPU Stress"** button on any instance's page.
3. This runs `stress --cpu 2 --timeout 300` on that instance for 300 seconds.

### Observe Scale-Out

1. Open the AWS Console → **EC2 → Auto Scaling Groups → autoscaling-labWorkshop-asg**.
2. Click the **Activity** tab — you will see scale-out events appearing within 1–3 minutes.
3. Under **Instance management**, new instances will enter the `InService` state.
4. Refresh the application URL repeatedly — new Instance IDs will appear as new instances join the target group.

### Observe Scale-In

After the stress timeout expires (300 seconds), CPU utilisation drops. CloudFormation's target tracking policy will automatically terminate excess instances once the cool-down period completes.

---

## Outputs

| Output Key | Description |
|---|---|
| `AlbDnsName` | Public HTTP endpoint of the Application Load Balancer |
| `VpcId` | ID of the provisioned VPC |
| `AutoScalingGroupName` | Name of the Auto Scaling Group |
| `PrivateSubnet1Id` | ID of Private Subnet 1 (AZ-1) |

---

## Security Design

| Control | Implementation |
|---|---|
| No direct SSH access | No `KeyName` in the Launch Template; port 22 not open |
| Private EC2 instances | Instances reside in private subnets with no public IPs |
| Least-privilege inbound | EC2 security group allows HTTP only from the ALB security group |
| Secure outbound access | Private instances reach the internet via NAT Gateway only |
| SSM Session Manager | IAM role includes `AmazonSSMManagedInstanceCore` for console access without SSH |
| Infrastructure as Code | All resources defined in a version-controlled CloudFormation template |

---

## Cost Considerations

- **NAT Gateway** — charged per hour and per GB of data processed. Terminate the stack when not in use.
- **ALB** — charged per hour and per LCU. Costs are minimal for demo traffic.
- **EC2 instances** — `t3.micro` is Free Tier eligible (first 12 months). Scaled-out instances incur additional charges.

To delete all resources and stop all charges:

```bash
aws cloudformation delete-stack --stack-name autoscaling-demo
```
