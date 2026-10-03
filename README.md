# AWS EC2 Auto Scaling Web Server

[![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazonaws)](https://aws.amazon.com/)
[![EC2](https://img.shields.io/badge/Amazon%20EC2-Auto%20Scaling-blue?logo=amazonec2)](https://aws.amazon.com/ec2/)
[![CloudWatch](https://img.shields.io/badge/CloudWatch-Monitoring-purple?logo=amazoncloudwatch)](https://aws.amazon.com/cloudwatch/)
[![Apache](https://img.shields.io/badge/Apache-HTTP%20Server-red?logo=apache)](https://httpd.apache.org/)

---

## Overview

This project sets up an **EC2 Auto Scaling environment on AWS**. EC2 instances are automatically provisioned from a Launch Template and managed by an Auto Scaling Group (ASG). Apache web server is deployed automatically via EC2 User Data. Scaling behavior is driven by a **Target Tracking policy** on **Average CPU Utilization**.

**Flow:**
```
Launch Template → Auto Scaling Group → EC2 Instances (Apache via User Data)
                                              ↓
                                    CloudWatch CPU Metrics
                                              ↓
                                  Target Tracking Policy
                                    ↙               ↘
                               Scale Out          Scale In
```

---

## AWS Services Used

| Service | Purpose |
|---|---|
| Amazon EC2 | Hosts the web server instances |
| EC2 Launch Template | Reusable instance configuration |
| EC2 Auto Scaling Group | Manages instance count automatically |
| Amazon CloudWatch | Monitors CPU utilization |
| Target Tracking Policy | Adjusts capacity based on CPU target |
| Security Group | Controls inbound/outbound traffic |
| EC2 User Data | Automates Apache installation on launch |

---

## Configuration

| Setting | Value |
|---|---|
| AWS Region | `us-east-1` (N. Virginia) |
| Launch Template | `autoscaling-web-template` |
| Auto Scaling Group | `autoscaling-web-asg` |
| Instance Type | `t3.micro` |
| AMI | Amazon Linux 2023 |
| Web Server | Apache HTTP Server (`httpd`) |
| Scaling Policy | Target Tracking |
| Metric | Average CPU Utilization |
| CPU Target | `50%` |
| Minimum Capacity | `1` |
| Desired Capacity | `2` |
| Maximum Capacity | `3` |
| Instance Warmup | `300 seconds` |

---

## Implementation Steps

### 1. AWS Region

Selected `us-east-1` — US East (N. Virginia).

![AWS Region](screenshots/01-aws-region.jpg)

---

### 2. Security Group

Created security group `autoscaling-web-sg` to allow HTTP access for web server testing.

![Security Group](screenshots/02-security-group.jpg)

---

### 3. EC2 User Data — Automated Web Server Deployment

Each instance installs Apache and creates a webpage showing its own instance ID automatically on launch.

```bash
#!/bin/bash

dnf update -y
dnf install -y httpd

systemctl enable httpd
systemctl start httpd

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

INSTANCE_ID=$(curl -s \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head><title>AWS Auto Scaling Web Server</title></head>
<body>
  <h1>AWS Auto Scaling Web Server</h1>
  <h2>Auto Scaling is working!</h2>
  <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
  <p>Apache installed automatically via EC2 User Data.</p>
</body>
</html>
EOF
```

> IMDSv2 token flow is used for secure metadata retrieval.

![User Data](screenshots/03-user-data.jpg)
![User Data (v2)](screenshots/03-user-data-v2.jpg)

---

### 4. Launch Template

Created `autoscaling-web-template` with:
- AMI: Amazon Linux 2023
- Instance type: `t3.micro`
- Security group: `autoscaling-web-sg`
- User Data: Apache install script above

![Launch Template](screenshots/04-launch-template.jpg)

---

### 5. Auto Scaling Group

Created `autoscaling-web-asg` using the Launch Template above.

![ASG Launch Template](screenshots/05-asg-launch-template.jpg)
![ASG Launch Template (v2)](screenshots/05-asg-launch-template-v2.jpg)

**Network configuration:**

![ASG Network](screenshots/06-asg-network.jpg)

**Capacity settings:**

| Setting | Value |
|---|---|
| Minimum | `1` |
| Desired | `2` |
| Maximum | `3` |

![ASG Capacity](screenshots/07-asg-capacity.jpg)

**ASG created:**

![ASG Created](screenshots/09-asg-created.jpg)

---

### 6. Target Tracking Scaling Policy

Configured a Target Tracking policy:
- Metric: Average CPU Utilization
- Target: `50%`
- Instance warmup: `300 seconds`

![Scaling Policy](screenshots/08-scaling-policy.jpg)
![Scaling Policy Verification](screenshots/14-scaling-policy.jpg)

---

## Scaling Demonstration

### Initial State — 2 Instances Running

![Two EC2 Instances](screenshots/10-two-ec2-instances.jpg)
![Two In-Service](screenshots/13-asg-two-inservice.jpg)

---

### Web Server Verification (Instance 1 & 2)

Each instance displays its own unique instance ID confirming independent automated provisioning.

![Web Server Instance 1](screenshots/11-web-server-instance-1.jpg)
![Web Server Instance 2](screenshots/12-web-server-instance-2.jpg)

---

### Generate CPU Load

```bash
sudo dnf install -y stress
stress --cpu 2 --timeout 600
```

![High CPU CloudWatch](screenshots/15-high-cpu-cloudwatch.jpg)

---

### Scale-Out — 3 Instances

ASG scaled out to 3 instances after CPU crossed the 50% target.

![Scale-Out to 3 Instances](screenshots/16-scale-out-3-instances.jpg)

The third instance was also automatically configured with Apache via User Data:

![Third Instance Web Server](screenshots/17-third-instance-web-server.jpg)

---

### Scale-In

After stopping the load, ASG scaled back in.

![Scale-In](screenshots/18-scale-in-2-instances.jpg)
![Final ASG Verification](screenshots/19-final-asg-verification.jpg)

---

## Scaling Timeline Summary

| Stage | Observed State |
|---|---|
| Initial | 2 EC2 instances running |
| Load generated | CPU utilization elevated |
| Scale-out | 3 EC2 instances (all InService) |
| New instance check | Third instance served Apache page |
| Scale-in | Capacity reduced after load stopped |

---

## Repository Structure

```
AWS-Auto-Scaling-Web-Server/
├── README.md
└── screenshots/
    ├── 01-aws-region.jpg
    ├── 02-security-group.jpg
    ├── 03-user-data.jpg
    ├── 03-user-data-v2.jpg
    ├── 04-launch-template.jpg
    ├── 05-asg-launch-template.jpg
    ├── 05-asg-launch-template-v2.jpg
    ├── 06-asg-network.jpg
    ├── 07-asg-capacity.jpg
    ├── 08-scaling-policy.jpg
    ├── 09-asg-created.jpg
    ├── 10-two-ec2-instances.jpg
    ├── 11-web-server-instance-1.jpg
    ├── 12-web-server-instance-2.jpg
    ├── 13-asg-two-inservice.jpg
    ├── 14-scaling-policy.jpg
    ├── 15-high-cpu-cloudwatch.jpg
    ├── 16-scale-out-3-instances.jpg
    ├── 17-third-instance-web-server.jpg
    ├── 18-scale-in-2-instances.jpg
    └── 19-final-asg-verification.jpg
```

---

## Cleanup

After completing the demo, delete resources to avoid charges:

1. Delete the Auto Scaling Group (this terminates managed EC2 instances)
2. Delete the Launch Template
3. Remove the Security Group
4. Verify no EC2/EBS resources remain in the AWS console

---

## References

- [EC2 Auto Scaling User Guide](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
- [EC2 Launch Templates](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-templates.html)
- [Target Tracking Scaling Policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
- [EC2 User Data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
- [Amazon CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
