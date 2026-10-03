# 🚀 AWS EC2 Auto Scaling Web Server

> A highly scalable web server environment built using Amazon EC2 Auto Scaling, Launch Template, CloudWatch Target Tracking, and Apache Web Server.

---

## 📌 Project Overview

This project demonstrates how to configure an **EC2 Auto Scaling environment on AWS**.

The environment automatically launches EC2 instances using a **Launch Template** and adjusts the number of running instances based on **CPU utilization** using an Amazon EC2 Auto Scaling Target Tracking policy.

The web server is automatically installed and configured using **EC2 User Data**.

### Key Features

* EC2 Launch Template
* Auto Scaling Group
* Minimum, Desired, and Maximum capacity configuration
* Apache Web Server installation using User Data
* CloudWatch CPU-based Target Tracking
* Automatic Scale-Out
* Automatic Scale-In
* Instance-specific web page displaying the EC2 Instance ID

---

## 🏗️ Architecture

```text
                    AWS Region
                    us-east-1
                        |
                        |
              Launch Template
        autoscaling-web-template
                        |
                        |
              Auto Scaling Group
              autoscaling-web-asg
                        |
             +----------+----------+
             |          |          |
          EC2 #1     EC2 #2     EC2 #3
             |          |          |
             +----------+----------+
                  Apache Web Server
                        |
                        |
              CloudWatch Metrics
                        |
             Target Tracking Policy
```

---

## ☁️ AWS Services Used

| Service             | Purpose                                     |
| ------------------- | ------------------------------------------- |
| Amazon EC2          | Hosts the web server instances              |
| EC2 Launch Template | Defines the configuration for new instances |
| Amazon CloudWatch   | Provides CPU utilization metrics            |
| Security Groups     | Controls inbound and outbound traffic       |
| Apache HTTP Server  | Serves the web application                  |

---

## 🌎 AWS Region

**Region:** US East (N. Virginia)

**Region Code:** `us-east-1`

![AWS Region](screenshots/01-aws-region.png)

---

## 🔐 Security Group

Security Group:

```text
autoscaling-web-sg
```

The security group allows HTTP traffic so that the Apache web server can be accessed from a browser.

![Security Group](screenshots/02-security-group.png)

---

## 🚀 Launch Template

Launch Template:

```text
autoscaling-web-template
```

The Launch Template defines the configuration used by the Auto Scaling Group when launching new EC2 instances.

It includes:

* EC2 instance configuration
* Security Group
* Amazon Machine Image
* User Data
* Web server installation configuration

![Launch Template](screenshots/04-launch-template.png)

---

## 📝 User Data

EC2 User Data automatically performs the following tasks when a new instance launches:

1. Updates system packages
2. Installs Apache HTTP Server
3. Starts Apache
4. Enables Apache at boot
5. Retrieves the EC2 Instance ID using IMDSv2
6. Creates the web page automatically

Example:

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
<head>
    <title>AWS Auto Scaling Web Server</title>
</head>
<body>
    <h1>AWS Auto Scaling Web Server</h1>
    <h2>Auto Scaling is working!</h2>
    <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
    <p>Apache Web Server installed automatically using User Data.</p>
</body>
</html>
EOF
```

![User Data](screenshots/03-user-data.png)

---

## ⚙️ Auto Scaling Group

Auto Scaling Group:

```text
autoscaling-web-asg
```

The Auto Scaling Group uses the Launch Template to automatically create and manage EC2 instances.

![ASG Launch Template](screenshots/05-asg-launch-template.png)

![ASG Network](screenshots/06-asg-network.png)

---

## 📊 Capacity Configuration

The Auto Scaling Group was configured with:

| Setting          | Value |
| ---------------- | ----: |
| Minimum Capacity |     1 |
| Desired Capacity |     2 |
| Maximum Capacity |     3 |

![ASG Capacity](screenshots/07-asg-capacity.png)

The final configuration was verified after completing the scaling tests.

![Final ASG Verification](screenshots/19-final-asg-verification.png)

---

## 📈 Scaling Policy

A **Target Tracking Scaling Policy** was configured using average CPU utilization.

```text
Metric: Average CPU Utilization
Target Value: 50%
```

When the average CPU utilization increases above the target, Auto Scaling can launch additional instances.

When CPU utilization decreases, Auto Scaling can reduce the number of instances, subject to the configured minimum capacity.

![Scaling Policy](screenshots/14-scaling-policy.png)

---

## 🌐 Web Server Deployment

Apache HTTP Server is installed automatically on EC2 instances through User Data.

The web page displays the instance ID, making it possible to verify that different EC2 instances are serving the application.

### Instance 1

![Web Server Instance 1](screenshots/11-web-server-instance-1.png)

### Instance 2

![Web Server Instance 2](screenshots/12-web-server-instance-2.png)

---

## 📊 Auto Scaling Test

### 1. Initial State

The Auto Scaling Group was initially configured with:

```text
Minimum  = 1
Desired  = 2
Maximum  = 3
```

Two EC2 instances were running.

![Two EC2 Instances](screenshots/10-two-ec2-instances.png)

![Two InService Instances](screenshots/13-asg-two-inservice.png)

---

### 2. Scale-Out Test

CPU load was generated on an EC2 instance using the `stress` utility.

```bash
sudo dnf install -y stress
stress --cpu 2 --timeout 600
```

This increased CPU utilization and allowed the Target Tracking scaling policy to respond.

![High CPU](screenshots/15-high-cpu-cloudwatch.png)

The Auto Scaling Group then launched a third EC2 instance.

```text
2 instances → 3 instances
```

![Three Instances](screenshots/16-scale-out-3-instances.png)

---

### 3. New Instance Verification

The newly launched instance automatically installed Apache through User Data.

The web page was accessed using the public IPv4 address of the new instance.

![Third Instance Web Server](screenshots/17-third-instance-web-server.png)

---

### 4. Scale-In Test

After stopping the CPU stress test, CPU utilization decreased.

The Auto Scaling Group subsequently reduced the number of instances according to the scaling policy and minimum capacity.

```text
3 instances → 2 instances → 1 instance
```

![Scale-In](screenshots/18-scale-in-2-instances.png)

---

## 🔄 Scaling Behavior Summary

```text
Initial:
Desired Capacity = 2

        ↓
   High CPU Load
        ↓
   Scale-Out
        ↓
3 EC2 Instances

        ↓
   CPU Load Stopped
        ↓
    Scale-In
        ↓
2 EC2 Instances

        ↓
Further Scale-In
        ↓
1 EC2 Instance
```

The Auto Scaling Group successfully demonstrated both **scale-out** and **scale-in** behavior.

---

## 📁 Project Structure

```text
AWS-Auto-Scaling-Web-Server/
│
├── README.md
│
└── screenshots/
    ├── 01-aws-region.png
    ├── 02-security-group.png
    ├── 03-user-data.png
    ├── 04-launch-template.png
    ├── 05-asg-launch-template.png
    ├── 06-asg-network.png
    ├── 07-asg-capacity.png
    ├── 08-scaling-policy.png
    ├── 09-asg-created.png
    ├── 10-two-ec2-instances.png
    ├── 11-web-server-instance-1.png
    ├── 12-web-server-instance-2.png
    ├── 13-asg-two-inservice.png
    ├── 14-scaling-policy.png
    ├── 15-high-cpu-cloudwatch.png
    ├── 16-scale-out-3-instances.png
    ├── 17-third-instance-web-server.png
    ├── 18-scale-in-2-instances.png
    └── 19-final-asg-verification.png
```

---

## 🧪 Testing Performed

The following tests were successfully performed:

* [x] Launch Template created
* [x] Auto Scaling Group created
* [x] Minimum capacity configured
* [x] Desired capacity configured
* [x] Maximum capacity configured
* [x] Apache installed automatically using User Data
* [x] EC2 web server verified
* [x] Target Tracking scaling policy configured
* [x] CPU load generated
* [x] Scale-out verified
* [x] Third instance launched
* [x] Third instance web server verified
* [x] CPU load stopped
* [x] Scale-in verified
* [x] Final ASG configuration verified

---

## 🎯 Conclusion

This project demonstrates an EC2-based web server environment with automatic scaling.

The Auto Scaling Group uses a Launch Template to create EC2 instances and a CloudWatch Target Tracking policy based on CPU utilization to adjust capacity.

The implementation successfully demonstrated:

* Automated EC2 instance provisioning
* Automatic Apache web server installation
* CPU-based scale-out
* CPU-based scale-in
* Dynamic instance management using Amazon EC2 Auto Scaling

---

## 👨‍💻 Project Information

**Project:** AWS EC2 Auto Scaling Web Server

**AWS Region:** `us-east-1`

**Auto Scaling Group:** `autoscaling-web-asg`

**Launch Template:** `autoscaling-web-template`

**Scaling Target:** `50% Average CPU Utilization`

**Capacity:** `Minimum 1 | Desired 2 | Maximum 3`
| EC2 Auto Scaling    | Automatically manages EC2 instance capacity |
                  Target = 50%
                CPU Utilization

