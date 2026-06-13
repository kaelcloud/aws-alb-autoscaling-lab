# AWS Day 4 Lab: Application Load Balancer + Auto Scaling Group + Multi-AZ Web Server

## Overview

This lab demonstrates how to deploy a simple Nginx web application on AWS using:

- Custom VPC
- Public subnets across multiple Availability Zones
- Application Load Balancer (ALB)
- Target Group
- Launch Template
- Auto Scaling Group (ASG)
- EC2 User Data
- Basic high availability and self-healing test

The goal of this lab is to understand how AWS Load Balancer and Auto Scaling work together to distribute traffic, maintain desired capacity, and improve application availability.

---

## Architecture

```text
Internet User
   |
   v
Application Load Balancer (ALB)
   |
   v
Target Group
   |
   v
Auto Scaling Group
   |-----------------------------|
   v                             v
EC2 Web Server - AZ 1       EC2 Web Server - AZ 2
Nginx                       Nginx
```

---

## Services Used

| AWS Service | Purpose |
|---|---|
| VPC | Provides isolated network environment |
| Public Subnet | Hosts internet-facing resources |
| Internet Gateway | Allows internet access to the VPC |
| Route Table | Routes traffic to local VPC or internet |
| Security Group | Controls inbound and outbound traffic |
| EC2 | Runs Nginx web server |
| Launch Template | Defines EC2 configuration for Auto Scaling |
| Target Group | Registers EC2 instances for Load Balancer |
| Application Load Balancer | Distributes HTTP traffic to healthy EC2 instances |
| Auto Scaling Group | Maintains desired number of EC2 instances |
| CloudWatch | Monitors metrics and Auto Scaling activity |

---

## Lab Objective

By the end of this lab, I was able to:

- Create a VPC with public subnets in multiple Availability Zones.
- Create Security Groups for ALB and EC2.
- Create a Launch Template with Nginx installation using User Data.
- Create a Target Group for EC2 instances.
- Create an internet-facing Application Load Balancer.
- Create an Auto Scaling Group with desired capacity of 2.
- Access the web application using ALB DNS.
- Verify traffic distribution between two EC2 instances.
- Test Auto Scaling self-healing by terminating one EC2 instance.
- Confirm that ASG automatically launched a replacement instance.

---

## Screenshots

### ALB DNS Result
![ALB DNS Result](screenshots/alb-dns-result.jpg)

### Auto Scaling Activity - EC2 Replacement
![ASG Activity](screenshots/asg-activity-replacement.jpg)

### EC2 Instances
![EC2 Instances](screenshots/ec2-instances.jpg)

### Target Group Healthy
![Target Group Healthy](screenshots/target-group-healthy.jpg)

---

## Step 1: Create VPC and Public Subnets

A custom VPC was created for this lab.

Example VPC CIDR:

```text
10.0.0.0/16
```

Two public subnets were created across two Availability Zones.

Example:

```text
Public Subnet 1: 10.0.0.0/20   - ap-southeast-1a
Public Subnet 2: 10.0.16.0/20  - ap-southeast-1b
```

The public subnets were associated with a route table that has internet access through an Internet Gateway.

Route table example:

```text
10.0.0.0/16  -> local
0.0.0.0/0    -> Internet Gateway
```

This makes the subnets public.

---

## Step 2: Create Security Group for ALB

A Security Group was created for the Application Load Balancer.

### ALB Security Group

Inbound rule:

| Type | Protocol | Port | Source |
|---|---|---:|---|
| HTTP | TCP | 80 | 0.0.0.0/0 |

Outbound rule:

| Type | Destination |
|---|---|
| All traffic | 0.0.0.0/0 |

Purpose:

```text
Allow users from the internet to access the Application Load Balancer using HTTP.
```

---

## Step 3: Create Security Group for EC2 Web Servers

A separate Security Group was created for EC2 instances.

### EC2 Security Group

Inbound rules:

| Type | Protocol | Port | Source |
|---|---|---:|---|
| HTTP | TCP | 80 | ALB Security Group |
| SSH | TCP | 22 | My IP |

Purpose:

```text
Allow HTTP traffic only from the Load Balancer.
Allow SSH only from trusted IP for troubleshooting.
```

This is more secure than allowing HTTP directly from the entire internet to EC2.

---

## Step 4: Create Launch Template

A Launch Template was created to define how EC2 instances should be launched by the Auto Scaling Group.

Configuration:

```text
AMI: Amazon Linux 2023
Instance type: t2.micro / t3.micro
Key pair: Existing key pair
Security Group: EC2 Web Server Security Group
Storage: Default 8GB gp3
```

### User Data Script

The following User Data script installs Nginx and creates a simple web page.

```bash
#!/bin/bash
dnf update -y
dnf install nginx -y
systemctl start nginx
systemctl enable nginx

HOSTNAME=$(hostname)

cat > /usr/share/nginx/html/index.html <<EOF
<html>
<head>
<title>Hazran Day 4 AWS Lab</title>
</head>
<body>
<h1>Hello from Hazran Day 4 Auto Scaling Lab</h1>
<p>Hostname: $HOSTNAME</p>
<p>This page is served by Nginx behind an Application Load Balancer.</p>
</body>
</html>
EOF
```

Purpose:

```text
When Auto Scaling launches a new EC2 instance, Nginx is automatically installed and started.
```

---

## Step 5: Create Target Group

A Target Group was created for EC2 instances.

Configuration:

```text
Target type: Instances
Protocol: HTTP
Port: 80
VPC: Day 4 Lab VPC
Health check path: /
```

The Target Group is used by the Load Balancer to identify which EC2 instances are healthy and ready to receive traffic.

Health check flow:

```text
ALB -> Target Group -> EC2 -> HTTP GET /
```

If the EC2 instance returns a successful response, it becomes healthy.

---

## Step 6: Create Application Load Balancer

An internet-facing Application Load Balancer was created.

Configuration:

```text
Load Balancer type: Application Load Balancer
Scheme: Internet-facing
IP address type: IPv4
Listener: HTTP port 80
Subnets: 2 public subnets in 2 Availability Zones
Security Group: ALB Security Group
Default action: Forward to Target Group
```

Purpose:

```text
The ALB receives traffic from users and forwards it to healthy EC2 instances in the Target Group.
```

---

## Step 7: Create Auto Scaling Group

An Auto Scaling Group was created using the Launch Template.

Configuration:

```text
Launch Template: hazran-day4-web-template
VPC: Day 4 Lab VPC
Subnets: 2 public subnets across 2 Availability Zones
Target Group: hazran-day4-web-tg
Desired capacity: 2
Minimum capacity: 1
Maximum capacity: 3
```

Purpose:

```text
Auto Scaling Group maintains the desired number of EC2 instances.
If one instance is terminated or unhealthy, ASG launches a replacement instance.
```

---

## Step 8: Test Load Balancer

After the Auto Scaling Group launched two EC2 instances, both instances were registered in the Target Group.

The ALB DNS name was used to access the web application:

```text
http://<ALB-DNS-NAME>
```

When the page was refreshed multiple times, different hostnames appeared.

Example:

```text
ip-10-0-0-240.ap-southeast-1.compute.internal
ip-10-0-24-51.ap-southeast-1.compute.internal
```

This proves that the Application Load Balancer was distributing traffic across multiple EC2 instances.

---

## Step 9: Test Auto Scaling Self-Healing

To test Auto Scaling self-healing:

1. Go to EC2 Instances.
2. Select one EC2 instance created by the Auto Scaling Group.
3. Terminate the instance.
4. Check Auto Scaling Group activity.
5. Observe that ASG launches a new replacement instance.

Expected behavior:

```text
Before termination:
Running instances = 2
Desired capacity = 2

After terminating one instance:
Running instances = 1
Desired capacity = 2

ASG action:
Launch new EC2 instance

Final state:
Running instances = 2
```

This confirms that Auto Scaling maintains the desired capacity.

---

## Step 10: CloudWatch Monitoring

CloudWatch can be used to monitor:

### EC2 Metrics

- CPU utilization
- Network in/out
- Status checks

### Load Balancer Metrics

- Request count
- Target response time
- HTTP 4xx / 5xx errors
- Healthy host count
- Unhealthy host count

### Auto Scaling Metrics

- Group desired capacity
- Group in-service instances
- Group total instances

CloudWatch helps detect performance issues and can trigger alarms or scaling actions.

---

## Key Concepts Learned

### Load Balancer

A Load Balancer receives traffic from users and distributes it to healthy EC2 instances in the Target Group.

### Target Group

A Target Group contains EC2 instances that receive traffic from the Load Balancer.

### Health Check

The Load Balancer checks whether EC2 instances are healthy before sending traffic to them.

### Auto Scaling Group

An Auto Scaling Group maintains the desired number of EC2 instances. If one instance fails, it launches a replacement instance.

### Launch Template

A Launch Template defines how EC2 instances should be created, including AMI, instance type, Security Group, key pair, and User Data.

### Multi-AZ

Multi-AZ means deploying resources across multiple Availability Zones. This improves availability because the application can continue running even if one Availability Zone has issues.

### CloudWatch

CloudWatch collects metrics and logs from AWS resources. It can be used to monitor performance, create alarms, and trigger scaling actions.

---

## Troubleshooting Notes

### Issue: ALB DNS not loading

Check:

```text
1. ALB state is Active.
2. ALB Security Group allows HTTP port 80 from 0.0.0.0/0.
3. Target Group has healthy targets.
4. EC2 Security Group allows HTTP port 80 from ALB Security Group.
5. Nginx is running on EC2.
6. User Data script completed successfully.
```

### Issue: Target Group unhealthy

Check:

```text
1. EC2 instance is running.
2. Status checks passed.
3. Nginx is installed and running.
4. EC2 Security Group allows HTTP from ALB Security Group.
5. Health check path is correct.
6. Route table and subnet configuration are correct.
```

### Issue: Auto Scaling did not replace instance

Check:

```text
1. Desired capacity value.
2. Minimum and maximum capacity.
3. Launch Template configuration.
4. Auto Scaling Group activity logs.
5. EC2 quota or capacity issue.
```

---

## Interview Explanation

### How does Load Balancer and Auto Scaling work?

```text
A Load Balancer receives traffic from users and distributes it to healthy EC2 instances in a Target Group. Auto Scaling Group maintains the desired number of EC2 instances. If one instance fails or is terminated, Auto Scaling automatically launches a replacement instance using the Launch Template. Once the new instance passes the health check, the Load Balancer can route traffic to it.
```

### How does Multi-AZ improve availability?

```text
Multi-AZ means deploying resources across multiple Availability Zones. If one Availability Zone has an issue, the application can still run from instances in another Availability Zone. In this lab, the Auto Scaling Group launched EC2 instances across two public subnets in different Availability Zones behind an Application Load Balancer.
```

### How do you monitor AWS resources?

```text
I use CloudWatch to monitor metrics such as EC2 CPU utilization, status checks, network traffic, Load Balancer request count, target response time, healthy host count, and unhealthy host count. CloudWatch alarms can also notify the team or trigger Auto Scaling when a threshold is reached.
```

### What did you learn from this lab?

```text
I learned how to deploy a highly available web application using Application Load Balancer and Auto Scaling Group. I also tested self-healing by terminating one EC2 instance, and the Auto Scaling Group automatically launched a replacement instance to maintain the desired capacity.
```

---

## Cleanup

To avoid unnecessary AWS charges, delete the following resources after the lab:

```text
1. Auto Scaling Group
2. EC2 instances created by ASG
3. Application Load Balancer
4. Target Group
5. Launch Template
6. Security Groups
7. VPC, subnets, route tables, and Internet Gateway if no longer needed
```

Always confirm that no EC2 instances, Load Balancers, or NAT Gateways are left running.
