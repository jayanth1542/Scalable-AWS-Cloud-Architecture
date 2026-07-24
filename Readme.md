# AWS High-Availability & Auto-Scaling Architecture: Production-Ready Web Tier

A production-grade, highly available, multi-AZ, load-balanced, and auto-scaling AWS infrastructure. This architecture is designed to handle variable web traffic, self-heal from instance failures, and enforce strict network isolation using a least-privilege security model.

---

## 🏗️ Architecture Overview

The system spans two Availability Zones (AZs) to ensure high availability. Traffic enters through an internet-facing **Application Load Balancer (ALB)**, which distributes incoming HTTP requests across a fleet of EC2 instances managed by an **Auto Scaling Group (ASG)**.

```mermaid
graph TD
    Internet((Internet)) -->|Port 80| ALB[Application Load Balancer]
    
    subgraph VPC ["AWS VPC (10.0.0.0/16)"]
        IGW[Internet Gateway] <--> RouteTable[Public Route Table]
        
        subgraph AZ_A ["Availability Zone A (us-east-1a)"]
            subgraph PublicSubnetA ["Public Subnet (10.0.1.0/24)"]
                ALB_ENI_A[ALB interface]
                EC2_A[EC2 Instance - ASG]
            end
            subgraph PrivateSubnetA ["Private Subnet (10.0.3.0/24 - Future DB)"]
                DB_A[(RDS Primary)]
            end
        end

        subgraph AZ_B ["Availability Zone B (us-east-1b)"]
            subgraph PublicSubnetB ["Public Subnet (10.0.2.0/24)"]
                ALB_ENI_B[ALB interface]
                EC2_B[EC2 Instance - ASG]
            end
            subgraph PrivateSubnetB ["Private Subnet (10.0.4.0/24 - Future DB)"]
                DB_B[(RDS Replica)]
            end
        end
    end

    RouteTable -.--> PublicSubnetA
    RouteTable -.--> PublicSubnetB
    ALB -->|Target Group HTTP:80| EC2_A
    ALB -->|Target Group HTTP:80| EC2_B
    
    classDef public fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef private fill:#efebe9,stroke:#5d4037,stroke-width:2px;
    classDef edge fill:#fff,stroke:#333,stroke-width:2px;
    class PublicSubnetA,PublicSubnetB public;
    class PrivateSubnetA,PrivateSubnetB private;
```

### Key Architectural Decisions
*   **Launch Templates vs. Launch Configurations**: Utilizing modern **Launch Templates** instead of legacy Launch Configurations. Templates allow versioning, support multiple instance types, and are required for newer AWS features like Spot/On-Demand mix and Capacity Blocks.
*   **Target Tracking Scaling Policy**: Configured using average CPU utilization (Target: 50%). Compared to step scaling, Target Tracking self-adjusts without requiring manual, hand-tuned CloudWatch alarm thresholds.
*   **Least-Privilege Security Groups**: The EC2 instances are locked down to accept traffic **only** from the ALB security group. No raw internet access on port 80 is permitted directly to the instances.

---

## 🛠️ Step-by-Step Implementation

### Phase 1: Networking Foundation (VPC, Subnets, IGW, Route Tables)
1.  **VPC Creation**: Formed an isolated virtual network with CIDR block `10.0.0.0/16`.
2.  **Multi-AZ Subnets**: Created 2 public subnets in separate AZs (`10.0.1.0/24` in `AZ-a` and `10.0.2.0/24` in `AZ-b`) to ensure fault tolerance.
3.  **Private Subnets (Interview Ready)**: Added private subnets in both AZs (`10.0.3.0/24` and `10.0.4.0/24`) to establish a separate database/backend tier not exposed to the internet.
4.  **Internet Gateway (IGW)**: Attached an IGW to the VPC to enable outbound and inbound internet routing for public assets.
5.  **Routing & Subnet Association**: Defined a route table mapping `0.0.0.0/0` traffic to the IGW and associated it with **both** public subnets.

> [!WARNING]
> **Common Gotcha**: If you forget to associate your public subnets with the route table pointing to the IGW, your instances will launch successfully but will not be able to fetch updates, communicate with the internet, or register to target groups.

#### Visual Evidence (Networking Setup)
*   **VPC Resource Map**: ![VPC Resource Map](https://github.com/user-attachments/assets/8dbe4aee-9d7e-449f-b33f-5479f5c55605)
*   **Subnet Configurations**: ![Subnet Configurations](https://github.com/user-attachments/assets/db379a81-d97c-4c9f-9d76-b0d5dc36c17a)
*   **VPC Route Tables**: ![VPC Route Tables](https://github.com/user-attachments/assets/38a1feee-0c73-4ea5-8767-51733c4dcadd)
*   **Subnets List View**: ![Subnets List View](https://github.com/user-attachments/assets/450b5577-f98a-4362-9d58-5a430f0288f8)
*   **Internet Gateway Attachment**: ![Internet Gateway Attachment](https://github.com/user-attachments/assets/398e25e3-4eaf-4844-962e-819f596db583)

---

### Phase 2: Compute Tier (Launch Template & Auto Scaling Group)
1.  **Launch Template**: Configured a baseline AMI (Amazon Linux 2023) using the `t2.micro` free tier. Included a shell startup script under **User Data** to configure a web page printing metadata:
    ```bash
    #!/bin/bash
    dnf update -y
    dnf install -y httpd
    systemctl start httpd
    systemctl enable httpd
    
    # Retrieve instance details using IMDSv2
    TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
    INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
    AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
    
    echo "<h1>Hello from Web Server</h1><p>Instance ID: <b>$INSTANCE_ID</b></p><p>Availability Zone: <b>$AZ</b></p>" > /var/www/html/index.html
    ```
2.  **Auto Scaling Group (ASG)**: Configured the group to span both public subnets with capacity bounds of:
    *   **Min**: 1
    *   **Desired**: 2
    *   **Max**: 4
3.  **Target Tracking Scaling Policy**: Tied scaling to a CPU metric target of `50%`.

#### Visual Evidence (Compute & Scaling)
*   **Launch Template Setup**: ![Launch Template Setup](https://github.com/user-attachments/assets/5a937b9c-e3c5-4639-b96b-aa9682d2a5e7)
*   **Auto Scaling Group Configurations**: ![Auto Scaling Group Configurations](https://github.com/user-attachments/assets/bc91bbd2-529e-478f-8269-a5a4eb4d3ade)
*   **ASG Activity History**: ![ASG Activity History](https://github.com/user-attachments/assets/5a4016b8-3f62-421b-94d8-949d52807cd2)
*   **User Data Script Config**: ![User Data Script Config](https://github.com/user-attachments/assets/3c66bef9-acac-4d7b-bfbd-0219a39c0963)
*   **ASG Target Scaling Policies**: ![ASG Target Scaling Policies](https://github.com/user-attachments/assets/1fd46970-76c4-46e8-b39b-4c91dd8f61f5)

---

### Phase 3: Load Balancing & Strict Network Isolation
1.  **ALB Security Group**: Allowed ingress on HTTP (`Port 80`) from the public internet (`0.0.0.0/0`).
2.  **EC2 Security Group**: Allowed incoming HTTP (`Port 80`) traffic **strictly restricted** to references of the ALB's Security Group ID. Direct traffic from the public internet to the instances is completely blocked.
3.  **Application Load Balancer (ALB)**: Provisioned a public-facing Application Load Balancer routed across public subnets.
4.  **Target Group**: Set up HTTP-based target grouping on Port 80, configured with health checks targeting `/`. Associated the ASG directly with this Target Group.

> [!IMPORTANT]
> **Security Best Practice**: Limiting EC2 ingress to the ALB security group protects the web servers from direct scans, custom payload injections, and DDoS attacks targeting the instances' public IPs.

#### Visual Evidence (Load Balancing & Security)
*   **Security Group Rules**: ![Security Group Rules](https://github.com/user-attachments/assets/c37b3279-1a66-4780-ac67-d7614af1b017)
*   **ALB Listener Config**: ![ALB Listener Config](https://github.com/user-attachments/assets/51b7001c-54aa-425a-9093-9828c3052200)
*   **Target Group Registered Targets**: ![Target Group Registered Targets](https://github.com/user-attachments/assets/718fe2f6-1040-4cd3-8bfd-f9a127b41c7a)
*   **ALB Details & DNS Name**: ![ALB Details & DNS Name](https://github.com/user-attachments/assets/33263c86-68dd-466a-867e-bbe81b31dafd)
*   **Target Group Health Check Configurations**: ![Target Group Health Check Configurations](https://github.com/user-attachments/assets/980c91f3-78c9-494e-932d-8db4b63a8c67)
*   **Security Group Associations**: ![Security Group Associations](https://github.com/user-attachments/assets/4b9ff3c6-b7c4-4bf7-b12d-a029c7f517c3)

---

### Phase 4: Proactive Observability (CloudWatch Alarms)
Monitoring metrics play a vital role in keeping production systems online. We created two targeted CloudWatch alarms:
1.  **ASG High CPU Alarm**: Triggers a scale-out event if average CPU across the ASG fleet exceeds `70%` for a sustained period of 5 minutes.
2.  **Unhealthy Host Alarm**: Monitors the ALB Target Group for any hosts failing health check responses (`UnhealthyHostCount > 0`).

> [!TIP]
> While high CPU alarms tell you if your cluster is busy, the **Unhealthy Host Count** alarm tells you if the application is broken. Operationally, this is the most critical metric for routing on-call support response.

#### Visual Evidence (Monitoring & Alarms)
*   **CloudWatch CPU Alarm**: ![CloudWatch CPU Alarm](https://github.com/user-attachments/assets/7028e8ea-de10-47c1-892d-57bca408ff8c)
*   **Target Group Metrics Dashboard**: ![Target Group Metrics Dashboard](https://github.com/user-attachments/assets/c42d5be7-f665-4611-9183-00466e73219a)
*   **Unhealthy Host Count Alarm**: ![Unhealthy Host Count Alarm](https://github.com/user-attachments/assets/c5fa62ca-7c1c-4354-804e-039712646844)
*   **Alarm State Transition View**: ![Alarm State Transition View](https://github.com/user-attachments/assets/e9edc74b-0485-4848-b9d9-a8f88e1ea534)

---

### Phase 5: Verification & High-Availability Testing

To prove the high-availability and fault tolerance claims of this design, the following functional validation tests were executed:

#### 1. Load Balancer Round-Robin Test
By visiting the DNS name of the ALB and performing successive refreshes, we observed that traffic is routed seamlessly back and forth between instances in Availability Zone A (`us-east-1a`) and Availability Zone B (`us-east-1b`), indicating round-robin routing is operational.

```
Request 1 -> ALB -> Instance A (us-east-1a)
Request 2 -> ALB -> Instance B (us-east-1b)
```

#### 2. Auto-Scaling Self-Healing Test
To simulate a real-world hardware failure or web server crash:
1.  We manually terminated one of the running EC2 instances via the AWS console.
2.  The ALB Target Group health checks immediately identified the host as unhealthy.
3.  The Auto Scaling Group detected the drop in desired capacity (from 2 to 1).
4.  Within minutes, a replacement EC2 instance was automatically launched, initialized via the User Data script, marked healthy, and reregistered behind the Load Balancer with zero application downtime.

#### Visual Evidence (Testing & Healing)
*   **ALB DNS Response Instance A**: ![ALB DNS Response Instance A](https://github.com/user-attachments/assets/ac79cd46-f403-4cde-829c-79ad7b245942)
*   **ALB DNS Response Instance B**: ![ALB DNS Response Instance B](https://github.com/user-attachments/assets/a3f5d9b7-b80d-4809-badc-e973d98ded3a)
*   **Instance State Terminating Simulation**: ![Instance State Terminating Simulation](https://github.com/user-attachments/assets/beaac2c7-92d5-48e0-846b-9b23bd6ea64d)
*   **ASG Registering Replacement Instance**: ![ASG Registering Replacement Instance](https://github.com/user-attachments/assets/27a8ae29-d06c-4fd8-b1d7-fce1d422fe43)
*   **EC2 Instance Lifecycle Log**: ![EC2 Instance Lifecycle Log](https://github.com/user-attachments/assets/af414579-1bc6-4a92-9488-b0e900f1a378)
*   **Unhealthy Target Removed From Pool**: ![Unhealthy Target Removed From Pool](https://github.com/user-attachments/assets/04b85e98-37a5-4ba2-beb5-08a548955801)
*   **Capacity Restored Activity log**: ![Capacity Restored Activity log](https://github.com/user-attachments/assets/05c80ef5-9105-4926-b478-48663a734cb5)
*   **Running Fleet Self-Healed**: ![Running Fleet Self-Healed](https://github.com/user-attachments/assets/5ac0cbe3-5efc-454d-b78c-cf498ebdf015)

---

## 🚀 Key Takeaways for DevOps Interviews
*   **Highly Available Pattern**: Running an Auto Scaling Group across multiple subnets in different AZs protects against AZ outages.
*   **Security Perimeter**: Establishing strict Security Group ingress chains prevents raw internet access to compute nodes.
*   **Actionable Observability**: Highlighting the importance of alarm configurations on `UnhealthyHostCount` over raw CPU tells interviewers you know how to operate apps in production, not just configure them.
