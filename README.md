# aws-prod-vpc-project

# AWS Production-Grade Project: Deploying Applications in a Secure VPC

This repository contains a step-by-step tutorial for implementing a production-grade AWS project, deploying applications securely within a Virtual Private Cloud (VPC) using public and private subnets, an auto-scaling group, a load balancer, a NAT Gateway, and a Bastion host. This setup follows AWS best practices for secure application deployment, as outlined in the AWS Zero to Hero series by Abhishek Veeramalla. By following this guide, you’ll learn how to create a robust AWS architecture that ensures high availability, scalability, and security.

This tutorial assumes you have an AWS account and basic familiarity with AWS services like EC2, VPC, and SSH. No screenshots are included, but the instructions are detailed to help you replicate the project.

## 📑 Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step-by-Step Implementation](#step-by-step-implementation)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [Conclusion](#conclusion)

## Project Overview

This project demonstrates how to:
- Create a VPC with public and private subnets across two availability zones for high availability.
- Deploy a NAT Gateway and Application Load Balancer in the public subnet.
- Launch EC2 instances in private subnets using an auto-scaling group.
- Use a Bastion host to securely access EC2 instances in private subnets.
- Deploy a simple Python web application and access it via the load balancer.

### Architecture
The architecture includes:
- **VPC**: Contains public and private subnets in two availability zones (e.g., us-east-1a and us-east-1b).
- **Public Subnet**: Hosts the NAT Gateway (for outbound internet access from private subnets) and the Application Load Balancer (to route external traffic to EC2 instances).
- **Private Subnet**: Hosts EC2 instances running the application, managed by an auto-scaling group for scalability.
- **Bastion Host**: An EC2 instance in the public subnet for secure SSH access to private subnet instances.
- **Internet Gateway**: Enables public subnet resources to communicate with the internet.

The traffic flow is:
- Users access the application via the load balancer (public subnet) over the internet.
- The load balancer routes traffic to EC2 instances in the private subnet.
- EC2 instances use the NAT Gateway for outbound internet access, masking their private IP addresses.

## Prerequisites
- An AWS account with sufficient permissions to create VPCs, EC2 instances, load balancers, and NAT Gateways.
- A local machine with SSH client and a key pair (.pem file) for AWS.
- Basic knowledge of Linux commands and Python for deploying the application.
- Ensure you have free-tier eligible resources or monitor costs to avoid unexpected charges.

## Step-by-Step Implementation

### Step 1: Create a VPC
1. **Access the AWS Management Console** and navigate to the VPC service by searching for "VPC" in the search bar.
2. Click **Create VPC** and select **VPC and more** for automated configuration.
3. Configure the VPC:
   - **Name**: `aws-prod-example`
   - **IPv4 CIDR block**: Use the default (e.g., `10.0.0.0/16`) for 65,536 IP addresses.
   - **Number of Availability Zones**: 2 (e.g., us-east-1a and us-east-1b).
   - **Number of public subnets**: 2.
   - **Number of private subnets**: 2.
   - **NAT Gateways**: Select "1 per AZ" (one NAT Gateway per availability zone).
   - **VPC endpoints**: Set to "None" (remove any default S3 endpoint).
4. Click **Create VPC**. AWS will create:
   - A VPC with public and private subnets in two availability zones.
   - An Internet Gateway attached to the VPC.
   - Route tables: Public subnets route to the Internet Gateway; private subnets route to the NAT Gateway.
5. Wait for the VPC creation to complete (may take a few minutes). If you encounter an error about reaching the elastic IP address limit, go to the EC2 console, navigate to **Elastic IPs**, and release unused IP addresses from previous projects. Retry the VPC creation.

### Step 2: Create a Launch Template for Auto Scaling
1. In the AWS Management Console, go to the **EC2** service.
2. Navigate to **Auto Scaling Groups** under the "Auto Scaling" section and click **Create Auto Scaling Group**.
3. First, create a **Launch Template**:
   - **Name**: `aws-prod-example`.
   - **Description**: "Launch template for deploying apps in AWS private subnet".
   - **AMI**: Select an Ubuntu AMI (e.g., Ubuntu Server 20.04 LTS, free-tier eligible).
   - **Instance Type**: Choose `t2.micro` (free-tier eligible).
   - **Key Pair**: Select an existing key pair (e.g., `aws-login`) for SSH access.
   - **Security Group**: Create a new security group named `aws-prod-example`.
     - Add inbound rules:
       - **SSH (Port 22)**: Source `Anywhere` (0.0.0.0/0) for Bastion host access.
       - **Custom TCP (Port 8000)**: Source `Anywhere` (0.0.0.0/0) for the application.
   - **VPC**: Select the `aws-prod-example` VPC created in Step 1.
   - Leave other settings (e.g., EBS volumes) as default.
4. Click **Create Launch Template**.

### Step 3: Create an Auto Scaling Group
1. In the **Auto Scaling Groups** section, click **Create Auto Scaling Group**.
2. Configure:
   - **Name**: `aws-prod-example`.
   - **Launch Template**: Select `aws-prod-example`.
   - **VPC**: Choose `aws-prod-example`.
   - **Subnets**: Select the two private subnets (e.g., us-east-1a and us-east-1b private subnets).
   - **Desired Capacity**: Set to `2` (two EC2 instances).
   - **Minimum Capacity**: Set to `2`.
   - **Maximum Capacity**: Set to `4` (allows scaling up to four instances during high traffic).
   - **Scaling Policies**: Select "None" for now (configure advanced scaling later if needed).
   - **Notifications**: Skip SNS notifications.
3. Click **Create Auto Scaling Group**. Wait for the group to create two EC2 instances (one in us-east-1a, one in us-east-1b). Verify in the EC2 console under **Instances** that both instances are running and in the correct availability zones.

### Step 4: Create a Bastion Host
1. In the **EC2** console, click **Launch Instance**.
2. Configure:
   - **Name**: `bastion-host`.
   - **AMI**: Select the same Ubuntu AMI used in the launch template.
   - **Instance Type**: `t2.micro`.
   - **Key Pair**: Use the same key pair (`aws-login`).
   - **Network Settings**:
     - **VPC**: Select `aws-prod-example`.
     - **Subnet**: Choose a public subnet (e.g., us-east-1a public subnet).
     - **Auto-assign Public IP**: Enable.
   - **Security Group**: Create a new security group with:
     - **SSH (Port 22)**: Source `Anywhere` (0.0.0.0/0).
3. Click **Launch Instance**. Wait for the instance to be in the "Running" state.

### Step 5: Deploy the Application
1. **Copy the SSH Key to the Bastion Host**:
   - On your local machine, locate your SSH key (e.g., `aws-login.pem`).
   - Use the `scp` command to copy the key to the Bastion host:
     ```bash
     scp -i aws-login.pem aws-login.pem ubuntu@<bastion-host-public-ip>:~/
     ```
     Replace `<bastion-host-public-ip>` with the Bastion host’s public IP from the EC2 console.
   - Confirm the transfer by SSHing into the Bastion host:
     ```bash
     ssh -i aws-login.pem ubuntu@<bastion-host-public-ip>
     ```
     Run `ls` to verify `aws-login.pem` is present.
2. **SSH into a Private EC2 Instance**:
   - From the Bastion host, SSH into one of the private EC2 instances using its private IP (e.g., `10.0.1.4`):
     ```bash
     ssh -i aws-login.pem ubuntu@<private-ec2-ip>
     ```
   - Verify you’re logged into the private instance.
3. **Install the Python Application**:
   - Create a simple HTML file (`index.html`) on the private EC2 instance:
     ```bash
     nano index.html
     ```
     Add:
     ```html
     <!DOCTYPE html>
     <html>
     <head>
       <title>My First AWS Project</title>
     </head>
     <body>
       <h1>My First AWS Project</h1>
       <p>Demonstrating apps in a private subnet.</p>
     </body>
     </html>
     ```
     Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).
   - Run a Python HTTP server on port 8000:
     ```bash
     python3 -m http.server 8000
     ```
   - Keep the terminal open to keep the server running.
4. For demonstration purposes, install the application on only one EC2 instance to test load balancing behavior later.

### Step 6: Create a Target Group
1. In the **EC2** console, navigate to **Target Groups** under "Load Balancing".
2. Click **Create Target Group** and configure:
   - **Target Type**: Instances.
   - **Name**: `aws-prod-example`.
   - **Protocol**: HTTP.
   - **Port**: `8000` (matches the Python server port).
   - **VPC**: Select `aws-prod-example`.
   - **Health Check**: Use default HTTP settings.
3. Select the two EC2 instances from the auto-scaling group and click **Include as pending below**.
4. Click **Create Target Group**. Wait for the target group to be created.

### Step 7: Create an Application Load Balancer
1. In the **EC2** console, navigate to **Load Balancers** and click **Create Load Balancer**.
2. Choose **Application Load Balancer** and configure:
   - **Name**: `aws-prod-example`.
   - **Scheme**: Internet-facing.
   - **IP address type**: IPv4.
   - **VPC**: Select `aws-prod-example`.
   - **Mappings**: Select both public subnets (us-east-1a and us-east-1b).
   - **Security Group**: Create a new security group with:
     - **HTTP (Port 80)**: Source `Anywhere` (0.0.0.0/0).
   - **Listeners**: Add a listener for HTTP on port 80, forwarding to the `aws-prod-example` target group.
3. Click **Create Load Balancer**. Wait for the load balancer to become active (may take a few minutes).
4. If you encounter an error (e.g., "Port 80 not reachable"), edit the load balancer’s security group:
   - Go to **Security Groups**, select the load balancer’s security group, and add an inbound rule for **HTTP (Port 80)** from `Anywhere` (0.0.0.0/0).
   - Save the rule and wait for the change to take effect.

### Step 8: Test the Application
1. In the **Load Balancers** section, select your load balancer (`aws-prod-example`) and copy its DNS name (e.g., `aws-prod-example-1234567890.us-east-1.elb.amazonaws.com`).
2. Open a browser and navigate to `http://<load-balancer-dns>`.
3. You should see the HTML page (“My First AWS Project”) if the request hits the EC2 instance with the running application. If it hits the other instance (without the application), you may see an error, demonstrating load balancing behavior.
4. **Assignment**: Deploy the same Python application on the second EC2 instance:
   - SSH into the second private EC2 instance via the Bastion host (as in Step 5).
   - Create an `index.html` file with different content, e.g.:
     ```html
     <!DOCTYPE html>
     <html>
     <head>
       <title>My Second AWS Project</title>
     </head>
     <body>
       <h1>My Second AWS Project</h1>
       <p>Demonstrating load balancing in a private subnet.</p>
     </body>
     </html>
     ```
   - Run `python3 -m http.server 8000`.
   - Refresh the load balancer’s DNS URL multiple times. You should see “My First AWS Project” and “My Second AWS Project” alternately, confirming load balancing across both instances.

## Troubleshooting
- **VPC Creation Fails**: Ensure you have enough elastic IP addresses. Release unused ones in the EC2 console.
- **Load Balancer Inaccessible**: Verify the load balancer’s security group allows HTTP (port 80) traffic. Check the target group’s health checks; ensure the application is running on port 8000.
- **SSH Issues**: Confirm the Bastion host and private EC2 instances are in the same VPC, and the SSH key is correctly copied to the Bastion host.
- **Unhealthy Targets**: Ensure the Python server is running on the correct port (8000) and the EC2 instance’s security group allows port 8000 from the load balancer.

## Cleanup
To avoid AWS charges:
1. Delete the load balancer (`aws-prod-example`) in the EC2 console.
2. Terminate the auto-scaling group and its EC2 instances.
3. Delete the Bastion host instance.
4. Delete the VPC (`aws-prod-example`) in the VPC console, ensuring all resources (subnets, NAT Gateways, route tables, Internet Gateway) are removed.

## Conclusion
Congratulations! You’ve implemented a production-grade AWS architecture with a secure VPC, auto-scaling group, load balancer, NAT Gateway, and Bastion host. This project showcases key DevOps skills like infrastructure setup, secure access management, and load balancing. Try the assignment to deploy the application on both EC2 instances to observe load balancing in action.

Feel free to contribute to this repository by suggesting improvements or sharing your results in the issues section!

## 📢 Credits

This project is inspired by the [AWS Zero to Hero series by Abhishek Veeramalla](https://www.youtube.com/watch?v=GkKNxyLp_V0&list=PLdpzxOOAlwvLNOxX0RfndiYSt1Le9azze).  
The tutorial has been adapted and extended into a GitHub-friendly guide with detailed instructions for hands-on implementation and personal learning.

## References
- [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/)
- [AWS EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/)
- [AWS Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/)
