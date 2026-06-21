# Host a Dynamic Application on AWS using Kubernetes (EKS)

This guide outlines the process for hosting a dynamic application using Kubernetes on AWS Elastic Kubernetes Service (EKS). The infrastructure will be designed using a 3-tier VPC. 

---

## Architecture Overview

The application is deployed using the following AWS services:

* Amazon EKS (Kubernetes cluster)
* Amazon EC2 (migration server + compute resources)
* Amazon RDS (MySQL database)
* Amazon ECR (container registry)
* AWS Secrets Manager (secure credentials storage)
* Amazon VPC (custom networking)
* AWS Route 53 (DNS management)
* Application Load Balancer / Network Load Balancer (traffic routing)

---

## Prerequisites

Before starting, you should have:

* GitHub account (private repository access)
* Git installed locally
* Visual Studio Code
* Docker installed + Docker Hub account
* AWS CLI configured
* SSH key pair for GitHub authentication
* Basic understanding of containers and Kubernetes

---

## 1. VPC & Networking Setup

A custom Virtual Private Cloud (VPC) was created to isolate and secure all resources.

### Network Design:

* 2 Availability Zones
* 6 Subnets total:

  * 2 Public subnets
  * 2 Private application subnets
  * 2 Private database subnets

### Core Components:

* Internet Gateway attached to VPC
* NAT Gateway deployed in public subnet with Elastic IP
* Route tables:

  * Public subnets → Internet Gateway
  * Private subnets → NAT Gateway
* DNS hostnames enabled for internal resolution

---

## 2. Security Groups

3 security groups were configured to control traffic between resources:

### EC2 Instance Connect Endpoint

* Inbound: default
* Outbound: SSH access to VPC CIDR

### Database Migration EC2

* Inbound: SSH from EICE security group
* Outbound: default

### Amazon RDS (MySQL)

* Inbound: MySQL/Aurora from migration EC2 security group
* Outbound: default

---

## 3. Secure EC2 Access (No Bastion Host)

Instead of a traditional bastion host, an EC2 Instance Connect Endpoint (EICE) was used:

* Deployed in a private subnet
* Provides secure SSH access to private EC2 instances
* Eliminates the need for public-facing bastion hosts

---

## 4. S3 Storage

* An S3 bucket was created to store application source code
* EC2 instances retrieve application files from S3 during setup

---

## 5. IAM Configuration

IAM roles were created for EC2 instances to securely access AWS services.

### Policies included:

* S3 read access (ListBucket, GetObject)
* Secrets Manager access (GetSecretValue, DescribeSecret)

These policies were attached to an IAM role assigned to EC2 instances.

---

## 6. Domain & SSL Setup

* Domain registered using Route 53
* SSL certificate created using AWS Certificate Manager
* Certificate validated for HTTPS encryption

---

## 7. Database Layer (RDS + Secrets Manager)

### Database Setup

* DB subnet group created (private DB subnets only)
* MySQL database deployed using Amazon RDS

### Secrets Management

* Database credentials stored in AWS Secrets Manager
* Application retrieves credentials at runtime (no hardcoding)

---

## 8. Database Migration Server

* Temporary EC2 instance launched for database migrations
* IAM role attached for S3 + Secrets Manager access
* Migration scripts executed using Flyway via user data

---

## 9. AWS CLI Access

An IAM user was created with programmatic access:

```bash id="v7g0u9"
aws configure --profile <username>
```

---

## 10. Git & SSH Setup

SSH key generation:

```bash id="k2j8fd"
ssh-keygen -t rsa -b 2048
```

* Public key added to GitHub
* Private repository cloned using SSH

---

## 11. Docker Image Build & Deployment

A Docker image was created for the application.

### Build Image:

```bash id="w9x2lm"
docker build \
--build-arg PERSONAL_ACCESS_TOKEN=<token> \
--build-arg GITHUB_USERNAME=<username> \
--build-arg REPOSITORY_NAME=<repo> \
--build-arg WEB_FILE_ZIP=<file.zip> \
--build-arg DOMAIN_NAME=<domain> \
--build-arg RDS_ENDPOINT=<endpoint> \
--build-arg RDS_DB_NAME=<db-name> \
--build-arg RDS_MASTER_USERNAME=<username> \
--build-arg RDS_DB_PASSWORD=<password> \
-t <image-tag> .
```

### Push to Amazon ECR:

```bash id="9qz3kt"
aws ecr create-repository --repository-name <repo> --region <region>

aws ecr get-login-password | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com

docker tag <image-tag> <repository-uri>
docker push <repository-uri>
```

---

## 12. Amazon EKS Cluster Setup

The application was deployed using Amazon EKS.

### IAM Roles:

* EKS Cluster Role (control plane permissions)
* Worker Node Role:

  * EKSWorkerNodePolicy
  * EC2ContainerRegistryReadOnly
  * EKS_CNI_Policy

### Cluster Setup:

* EKS cluster created
* Managed node group configured
* IAM user access granted to cluster

---

## 13. Kubernetes Deployment

Kubernetes manifests (provided by the project) were used to deploy the application onto the EKS cluster.

These included:

* Deployment file → defines application pods
* Service file → exposes application via load balancer
* Secrets configuration → integrates AWS Secrets Manager

The manifests were applied to the cluster using Kubernetes CLI.

---

## 14. Load Balancer & Exposure

Once the service was deployed:

* Kubernetes automatically provisioned a Network Load Balancer
* This exposed the application to the internet

---

## 15. DNS Configuration

* Load balancer DNS was retrieved after deployment
* A Route 53 record set was created pointing to the NLB
* This connected the application to the custom domain
---

## Conclusion

Successfully deployed a dynamic application using Kubernetes on AWS EKS. The infrastructure is highly available, secure, and scalable.


