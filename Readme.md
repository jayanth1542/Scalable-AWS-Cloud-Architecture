# Scalable AWS Cloud Architecture

A highly available, auto-scaling 3-tier web architecture built on AWS — VPC with public/private subnets across two Availability Zones, an Auto Scaling Group behind an Application Load Balancer, and RDS for the database tier.

## Architecture Overview

```
                        Internet
                            |
                    [Internet Gateway]
                            |
                 ┌──────────────────────┐
                 │   Application Load    │
                 │      Balancer         │  (public subnets, both AZs)
                 └──────────┬───────────┘
                            |
              ┌─────────────┴─────────────┐
              │                           │
     [EC2 - ap-south-1a]         [EC2 - ap-south-1b]
              │                           │
        Auto Scaling Group (min 1 / desired 2 / max 3)
                            |
                 ┌──────────────────────┐
                 │      Amazon RDS       │  (private subnets, Multi-AZ)
                 └──────────────────────┘
```

**Services used:** VPC, Subnets (public + private, 2 AZs), Internet Gateway, Route Tables, Security Groups, EC2, Launch Templates, Auto Scaling Groups, Application Load Balancer, Target Groups, RDS, CloudWatch

---

## 1. Networking Foundation

Built a custom VPC (`10.0.0.0/16`) with 2 public and 2 private subnets spread across `ap-south-1a` and `ap-south-1b` for high availability. Attached an Internet Gateway and configured route tables so only the public subnets route outbound traffic to `0.0.0.0/0`.

![VPC and Subnets](PASTE_SCREENSHOT_LINK_HERE)
![Route Table with IGW route](PASTE_SCREENSHOT_LINK_HERE)
![Resource Map confirming subnet-to-IGW routing](PASTE_SCREENSHOT_LINK_HERE)

**Design note:** Kept subnet selection out of the Launch Template and delegated it to the Auto Scaling Group instead, so instances distribute automatically across both AZs rather than being pinned to one.

---

## 2. Compute — Launch Template & Auto Scaling Group

Created a Launch Template (not the older Launch Config, to reflect current best practice) with Amazon Linux 2023, `t2.micro`, and a user-data script that installs and configures nginx, serving a page showing the instance's own ID and Availability Zone — this makes it possible to visually confirm load balancing is actually distributing traffic.

```bash
#!/bin/bash
sudo dnf install -y nginx

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

sudo tee /usr/share/nginx/html/index.html > /dev/null << EOF
<!DOCTYPE html>
<html>
<head><title>ScalableAWSCloud Demo</title></head>
<body style="font-family: sans-serif; text-align: center; margin-top: 50px;">
  <h1>Hello from EC2!</h1>
  <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
  <p><strong>Availability Zone:</strong> $AZ</p>
</body>
</html>
EOF

sudo systemctl start nginx
sudo systemctl enable nginx
```

The Auto Scaling Group launches across both public subnets with a target-tracking scaling policy (50% average CPU), min 1 / desired 2 / max 3.

![Launch Template configuration](PASTE_SCREENSHOT_LINK_HERE)
![Auto Scaling Group with instances across both AZs](PASTE_SCREENSHOT_LINK_HERE)

---

## 3. Load Balancing

An internet-facing Application Load Balancer spans both public subnets and forwards HTTP traffic to a Target Group (target type: Instances, for native ASG integration). Security groups restrict traffic so only the ALB can reach EC2 on port 80.

![Target Group health checks passing](PASTE_SCREENSHOT_LINK_HERE)
![ALB configuration](PASTE_SCREENSHOT_LINK_HERE)

**Live proof of load balancing** — refreshing the ALB's DNS name shows the response alternating between different Instance IDs and Availability Zones:

![Instance 1 response - ap-south-1a](PASTE_SCREENSHOT_LINK_HERE)
![Instance 2 response - ap-south-1b](PASTE_SCREENSHOT_LINK_HERE)

---

## 4. Database — RDS

Added an RDS instance in the private subnets (DB Subnet Group spanning both AZs), with public access disabled and a security group that only accepts inbound connections from the EC2 security group — the database is never reachable from the internet directly.

![RDS configuration](PASTE_SCREENSHOT_LINK_HERE)

---

## 5. Monitoring

CloudWatch alarms track average CPU utilization (drives the ASG scaling policy) and unhealthy host count on the Target Group, to catch application-level failures that CPU alone wouldn't reveal.

![CloudWatch alarms](PASTE_SCREENSHOT_LINK_HERE)

---

## Challenges Encountered

Real infrastructure work involves debugging, not just clicking through a console. A few issues came up during this build:

**1. SSH host key verification failures**
Kept failing when connecting to a rebuilt instance. Diagnosed by checking the exact error (host identity mismatch) and clearing the stale entry in `~/.ssh/known_hosts` with `ssh-keygen -R <host>`, then re-verifying the new fingerprint on reconnect.

**2. Auto Scaling Group launching into the wrong (private) subnet**
Instances came up healthy but had no public IP and no internet access. Traced it by checking each instance's Subnet ID in the console and cross-referencing against the VPC resource map — the ASG's network settings included a private subnet alongside the public ones. Fixed by editing the ASG's subnet list to include only the public subnets.

**3. "Security groups in the launch template are not linked to the VPC" error**
Hit this when the Launch Template's security group belonged to the default VPC rather than the custom one. Security groups are VPC-scoped, so this failed at creation time rather than launching something broken — fixed by creating/selecting a security group explicitly scoped to the project VPC.

**4. Subnet had an internet route but instances still had no public IP**
Learned that a route to an Internet Gateway and a subnet's "auto-assign public IPv4" setting are two independent things — a subnet can be internet-*routable* without automatically giving instances a public IP. Enabled auto-assign on the public subnets and relaunched instances to pick up public IPs.

---

## What I'd Do Differently at Scale

- **Infrastructure as Code**: this was built manually via the console for hands-on learning; a production version would use Terraform or CloudFormation for repeatability and version control.
- **Secrets management**: RDS credentials would move to AWS Secrets Manager instead of being embedded in configuration.
- **Tighter security groups**: SSH restricted to a specific IP rather than left open during the build/debug phase; HTTP restricted to the ALB's security group only.
- **Read replicas**: for read-heavy workloads, since RDS itself doesn't horizontally scale the way the EC2 tier does.
