# AWS Highly Available Web Application using Auto Scaling and Load Balancer

## Project Overview

This project demonstrates the deployment of a highly available web application on AWS using:

- Amazon VPC
- Public and Private Subnets
- Internet Gateway
- Route Tables
- EC2 Instances
- Launch Template
- Auto Scaling Group
- Application Load Balancer
- Security Groups
- Nginx Web Server

The objective was to create an infrastructure that automatically scales application instances while distributing incoming traffic through an Application Load Balancer.

---

# Architecture

![Architecture](images/architecture.png)

---

# Technologies Used

- AWS EC2
- Amazon VPC
- Auto Scaling Group
- Application Load Balancer
- Launch Template
- Security Groups
- Internet Gateway
- Route Tables
- Nginx
- Amazon Linux

---

# Step 1 — Create VPC

A dedicated Virtual Private Cloud was created to isolate the infrastructure.

![Create VPC](images/vpc-created.png)

**Screenshot**

Replace with

```
Screenshot 2026-07-24 at 5.24.15 PM
```

---

# Step 2 — Create Public and Private Subnets

Four subnets were created.

- Public Subnet 1
- Public Subnet 2
- Private Subnet 1
- Private Subnet 2

This allows high availability across Availability Zones.

![Subnets](images/subnets.png)

Replace with

```
Screenshot 2026-07-24 at 5.24.40 PM
```

---

# Step 3 — Attach Internet Gateway

An Internet Gateway was attached to the VPC to provide Internet connectivity.

![IGW](images/igw.png)

Replace with

```
Screenshot 2026-07-24 at 5.29.06 PM
```

---

# Step 4 — Configure Route Tables

The public route table was configured to route traffic through the Internet Gateway.

Private subnets remained isolated.

![Route Table](images/route-table.png)

Replace with

```
Screenshot 2026-07-24 at 5.31.25 PM
```

---

# Step 5 — Associate Route Tables

Public subnets were associated with the public route table.

Private subnets remained associated with the private route table.

![Route Associations](images/route-association.png)

Replace with

```
Screenshot 2026-07-24 at 5.32.22 PM
```

---

# Step 6 — Create Security Groups

Separate Security Groups were configured.

### EC2 Security Group

Allowed

- SSH (22)
- HTTP (80)

### ALB Security Group

Allowed

- HTTP (80)

![Security Group](images/security-group.png)

Replace with

```
Screenshot 2026-07-24 at 5.33.01 PM
```

---

# Step 7 — Launch EC2 Instance

An EC2 instance was launched inside the public subnet.

Nginx was installed and configured.

```bash
sudo yum install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

![EC2](images/ec2.png)

Replace with

```
Screenshot 2026-07-24 at 5.36.49 PM
```

---

# Step 8 — Verify Nginx

The web server was verified by accessing the instance using its Public IP.

![Nginx](images/nginx.png)

Replace with

```
Screenshot 2026-07-24 at 5.57.19 PM
```

---

# Step 9 — Create Launch Template

A Launch Template was created using the configured EC2 instance.

The template stores:

- AMI
- Instance Type
- Security Group
- User Data

![Launch Template](images/launch-template.png)

Replace with

```
Screenshot 2026-07-24 at 6.08.34 PM
```

---

# Step 10 — Configure User Data

User Data automatically installs Nginx whenever Auto Scaling launches a new instance.

```bash
#!/bin/bash
yum update -y
yum install nginx -y
systemctl enable nginx
systemctl start nginx
```

![User Data](images/user-data.png)

Replace with

```
Screenshot 2026-07-24 at 6.08.34 PM
```

---

# Step 11 — Create Auto Scaling Group

The Auto Scaling Group was configured using the Launch Template.

Configuration included

- Minimum Capacity
- Desired Capacity
- Maximum Capacity

![ASG](images/asg-created.png)

Replace with

```
Screenshot 2026-07-24 at 6.14.42 PM
```

---

# Step 12 — Configure Scaling Policies

Scaling policies ensure automatic creation or termination of EC2 instances based on demand.

![Scaling Policy](images/scaling-policy.png)

Replace with

```
Screenshot 2026-07-24 at 6.41.18 PM
```

---

# Step 13 — Create Target Group

A Target Group was created to register EC2 instances.

Health checks were configured on port 80.

![Target Group](images/target-group.png)

Replace with

```
Screenshot 2026-07-24 at 6.41.29 PM
```

---

# Step 14 — Create Application Load Balancer

An internet-facing Application Load Balancer was deployed.

The ALB distributes traffic across healthy EC2 instances.

![ALB](images/alb-created.png)

Replace with

```
Screenshot 2026-07-24 at 6.41.43 PM
```

---

# Step 15 — Configure Listener

HTTP Listener was configured on port 80.

Traffic forwards to the Target Group.

![Listener](images/listener.png)

Replace with

```
Screenshot 2026-07-24 at 6.41.55 PM
```

---

# Step 16 — Register Healthy Targets

The EC2 instances successfully passed health checks.

Status became **Healthy**.

![Healthy Targets](images/healthy-targets.png)

Replace with

```
Screenshot 2026-07-24 at 6.42.04 PM
```

---

# Step 17 — Verify Load Balancer

The application was successfully accessed using the ALB DNS Name.

Traffic reached healthy EC2 instances automatically.

![ALB DNS](images/alb-working.png)

Replace with

```
Screenshot 2026-07-24 at 7.29.49 PM
```

---

# Challenges Faced

## 1. SSH Host Key Verification Failed

**Problem**

SSH connection failed because the host key stored locally no longer matched the instance.

**Solution**

Removed the old SSH key using:

```bash
ssh-keygen -R <public-ip>
```

---

## 2. Auto Scaling Group Launched Instances in Wrong Subnet

**Problem**

Instances were being launched inside private subnets.

**Solution**

Updated the ASG configuration to use the correct public subnets.

---

## 3. Security Group VPC Mismatch

**Problem**

Launch Template referenced a Security Group from another VPC.

**Solution**

Created a new Security Group inside the correct VPC and updated the Launch Template.

---

## 4. Target Group Health Checks Failed

**Problem**

Instances were marked unhealthy.

**Solution**

Verified:

- Nginx service
- Security Group rules
- HTTP port
- Health Check path

---

## 5. Nginx Not Starting Automatically

**Problem**

New instances created by Auto Scaling did not serve the webpage.

**Solution**

Added Nginx installation and startup commands inside User Data.

---

# Project Outcome

Successfully deployed a highly available AWS infrastructure capable of:

- Automatic scaling
- Load balancing
- Health monitoring
- Fault tolerance
- High availability

---

# Future Improvements

- HTTPS using ACM
- Route 53 Domain
- CloudWatch Monitoring
- SNS Alerts
- Terraform Automation
- CI/CD using GitHub Actions
- Docker Deployment
- Amazon RDS Backend

---

# Author

**Jayanth Somala**

Cloud & DevOps Engineering Student

GitHub: https://github.com/jayanth1542
