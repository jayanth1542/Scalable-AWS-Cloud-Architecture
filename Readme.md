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
                            

**Services used:** VPC, Subnets (public + private, 2 AZs), Internet Gateway, Route Tables, Security Groups, EC2, Launch Templates, Auto Scaling Groups, Application Load Balancer, Target Groups, RDS, CloudWatch

---

## 1. Networking Foundation

Built a custom VPC (`10.0.0.0/16`) with 2 public and 2 private subnets spread across `ap-south-1a` and `ap-south-1b` for high availability. Attached an Internet Gateway and configured route tables so only the public subnets route outbound traffic to `0.0.0.0/0`.

*   **VPC Successfully Created**:
    ![VPC and Subnets](https://github.com/user-attachments/assets/5ac0cbe3-5efc-454d-b78c-cf498ebdf015)
*   **VPC List & CIDR Block Settings**:
    ![Route Table with IGW route](https://github.com/user-attachments/assets/05c80ef5-9105-4926-b478-48663a734cb5)
*   **Creating Public Subnets**:
    ![Resource Map confirming subnet-to-IGW routing](https://github.com/user-attachments/assets/04b85e98-37a5-4ba2-beb5-08a548955801)
*   **Internet Gateway Attachment**:
    ![Subnet auto-assign public IP setting](https://github.com/user-attachments/assets/af414579-1bc6-4a92-9488-b0e900f1a378)
*   **Route Table Configured**:
    ![Security group configuration](https://github.com/user-attachments/assets/27a8ae29-d06c-4fd8-b1d7-fce1d422fe43)
*   **Route Table Subnet Associations**:
    ![Security group inbound rules](https://github.com/user-attachments/assets/beaac2c7-92d5-48e0-846b-9b23bd6ea64d)
*   **Interactive VPC Resource Map**:
    ![VPC Details & Resource Map](https://github.com/user-attachments/assets/a3f5d9b7-b80d-4809-badc-e973d98ded3a)

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

*   **Launch Template Configuration**:
    ![Launch Template configuration](https://github.com/user-attachments/assets/5a4016b8-3f62-421b-94d8-949d52807cd2)
*   **Launch Instance Wizard Network Settings**:
    ![Launch Instance Wizard](https://github.com/user-attachments/assets/ac79cd46-f403-4cde-829c-79ad7b245942)
*   **Baseline Instances in Running State**:
    ![EC2 instance details - AZ 1a](https://github.com/user-attachments/assets/980c91f3-78c9-494e-932d-8db4b63a8c67)
*   **Auto Scaling Group Details**:
    ![Auto Scaling Group configuration](https://github.com/user-attachments/assets/4b9ff3c6-b7c4-4bf7-b12d-a029c7f517c3)
*   **ASG Subnet Distribution (ap-south-1a & ap-south-1b)**:
    ![ASG network settings across both AZs](https://github.com/user-attachments/assets/e9edc74b-0485-4848-b9d9-a8f88e1ea534)
*   **ASG Scaling Capacity Limits**:
    ![ASG capacity and scaling policy](https://github.com/user-attachments/assets/7028e8ea-de10-47c1-892d-57bca408ff8c)

---

## 3. Load Balancing

An internet-facing Application Load Balancer spans both public subnets and forwards HTTP traffic to a Target Group (target type: Instances, for native ASG integration). Security groups restrict traffic so only the ALB can reach EC2 on port 80.

*   **Application Load Balancer Details**:
    ![ALB configuration](https://github.com/user-attachments/assets/7028e8ea-de10-47c1-892d-57bca408ff8c)
*   **Target Group Health Checks (2/2 Healthy Targets)**:
    ![Target Group health checks passing](https://github.com/user-attachments/assets/4b9ff3c6-b7c4-4bf7-b12d-a029c7f517c3)

**Live proof of load balancing** — refreshing the ALB's DNS name shows the response alternating between different Instance IDs and Availability Zones:

*   **Instance 1 response (ap-south-1a)**:
    ![Instance 1 response - ap-south-1a](https://github.com/user-attachments/assets/db379a81-d97c-4c9f-9d76-b0d5dc36c17a)
*   **Instance 2 response (ap-south-1b)**:
    ![Instance 2 response - ap-south-1b](https://github.com/user-attachments/assets/38a1feee-0c73-4ea5-8767-51733c4dcadd)

*(Direct access to instances using public IPs is also verified: [Instance 1 Public IP](https://github.com/user-attachments/assets/450b5577-f98a-4362-9d58-5a430f0288f8) & [Instance 2 Public IP](https://github.com/user-attachments/assets/398e25e3-4eaf-4844-962e-819f596db583))*

---


## 4. Monitoring & Self-Healing Verification

CloudWatch alarms track average CPU utilization (drives the ASG scaling policy) and unhealthy host count on the Target Group, to catch application-level failures that CPU alone wouldn't reveal.

### High Availability Self-Healing Demonstration
To verify the self-healing and recovery capabilities of our Auto Scaling infrastructure:

1.  **Initiated Instance Termination**: Terminated a running instance in the console to simulate server failure.
    ![Simulating failure - Instance state terminating](https://github.com/user-attachments/assets/718fe2f6-1040-4cd3-8bfd-f9a127b41c7a)
    ![Simulating failure - Confirmation prompt](https://github.com/user-attachments/assets/33263c86-68dd-466a-867e-bbe81b31dafd)
2.  **Target Removed**: The ALB Target Group marked the host unhealthy and pulled it from traffic routing.
3.  **ASG Replaced Instance**: The Auto Scaling Group detected the capacity drop and initialized a replacement instance.
    ![ASG replacing instance - initializing state](https://github.com/user-attachments/assets/c37b3279-1a66-4780-ac67-d7614af1b017)
    ![ASG state - 2/3 healthy during replacement](https://github.com/user-attachments/assets/5a937b9c-e3c5-4639-b96b-aa9682d2a5e7)
4.  **Capacity Restored**: The new instance completed initialization, passed health checks, and registered back to the load balancer automatically.
    ![ASG state - healed and complete](https://github.com/user-attachments/assets/1fd46970-76c4-46e8-b39b-4c91dd8f61f5)

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
