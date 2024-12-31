# Host-Application-with-Kubernetes-and-AWS-EKS

This guide outlines the process for hosting a dynamic application using Kubernetes on AWS Elastic Kubernetes Service (EKS). The infrastructure will be designed using a 3-tier Virtual Private Cloud (VPC) architecture with automated deployments, database migration, and proper security configuration.

---

## Prerequisites

Before you begin, ensure that the following tools and accounts are set up:

- **GitHub account**
- **Key pairs** to clone GitHub repositories to your local machine
- **Git** (configured with your username and email)
- **Visual Studio Code** (VS Code) installed
- **Docker** and **Docker Hub account**
- **Flyway** (for database migration)
- **AWS CLI** installed and configured to interact with AWS services

---

## AWS Infrastructure Setup (3-Tier VPC)

1. **Create a VPC** in the AWS Console.
2. **Enable DNS hostname** in the VPC to assign memorable hostnames to resources instead of IP addresses.
3. **Set up an Internet Gateway** to allow resources in the VPC to access the internet.
4. **Create public subnets** (1 per Availability Zone), including resources like Bastion Host, NAT Gateway, and Application Load Balancer (ALB). Enable auto-assigning public IPv4 addresses.
5. **Create private subnets** (2 per Availability Zone) for web servers and database instances.
6. **Create route tables**:
   - Public route table: Connect to the Internet Gateway and associate with public subnets.
   - Private route tables: Connect to the NAT Gateway for internet access.
7. **Set up a NAT Gateway** in a public subnet (optional for cost-saving, can be placed in one public subnet).
8. **Create Security Groups**:
   - EC2 instance connect endpoint
   - Database migration server
   - RDS database server
9. **Launch EC2 instance** for database migration (with necessary IAM role).
10. **Create an RDS instance** with the relevant database subnet group and associate with the appropriate security group.
11. **Set up Route 53** for domain name registration, SSL certificate, and create a record set for your application.

---

## Docker File and Image Build

1. **Create a GitHub repository** for your Docker files and clone it to your local machine.
2. **Write a Dockerfile** for the application, specifying the build arguments.
3. **Build the Docker image**:
    ```bash
    docker build \
    --build-arg PERSONAL_ACCESS_TOKEN=<token> \
    --build-arg GITHUB_USERNAME=<username> \
    --build-arg REPOSITORY_NAME=<repo-name> \
    --build-arg WEB_FILE_ZIP=<web-file-zip> \
    --build-arg WEB_FILE_UNZIP=<web-file-unzip> \
    --build-arg DOMAIN_NAME=<domain-name> \
    --build-arg RDS_ENDPOINT=<rds-endpoint> \
    --build-arg RDS_DB_NAME=<db-name> \
    --build-arg RDS_MASTER_USERNAME=<master-username> \
    --build-arg RDS_DB_PASSWORD=<db-password> \
    -t <image-tag> .
    ```

4. **Make the script executable** (Windows users):
    ```bash
    Set-ExecutionPolicy -ExecutionPolicy Unrestricted
    ```

5. **Build the image**:
    - Right-click the file in Visual Studio Code and choose "Open in Integrated Terminal."
    - Run the build command:
    ```bash
    .\<name-of-file>
    ```

---

## Push Docker Image to ECR

1. **Create an IAM user** with admin access for programmatic access to manage AWS resources (note the access keys).
2. **Authenticate** with AWS CLI:
    ```bash
    aws configure
    ```
3. **Create an ECR repository**:
    ```bash
    aws ecr create-repository --repository-name <repository-name> --region <region>
    ```
4. **Push the Docker image** to ECR:
    ```bash
    docker tag <image-tag> <repository-uri>
    aws ecr get-login-password | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
    docker push <repository-uri>
    ```

---

## Migrate SQL Data into RDS

1. **Create an S3 bucket** and upload the SQL migration file.
2. **Create an IAM role** with S3 access.
3. **Launch an EC2 instance** in the private subnet and assign the IAM role.
4. **Use Flyway** to migrate the SQL script from S3 to the RDS instance.
5. **Terminate the EC2 instance** once migration is complete.

---

## Install kubectl, eksctl, and Helm

1. **Install Chocolatey** (Windows package manager):
    ```bash
    Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
    ```

2. **Install kubectl**:
    ```bash
    choco install kubernetes-cli
    ```

3. **Install eksctl** (Download and unzip, then add it to the system path).
4. **Install Helm**:
    ```bash
    choco install kubernetes-helm
    ```

---

## Add Environment Variables to AWS Secrets Manager

1. **Create a secret** in AWS Secrets Manager for the Docker build arguments.
2. **Create an IAM policy** to allow EKS to access the secrets.

---

## Set Up EKS Cluster

1. **Create an IAM role** for the EKS cluster.
2. **Create an EKS cluster** in the AWS Console.
3. **Modify the RDS security group** to allow traffic from the EKS cluster security group.
4. **Create worker nodes** for the EKS cluster with the following policies:
    - `AmazonEKSWorkerNodePolicy`
    - `AmazonEC2ContainerRegistryPullOnly`
    - `AmazonEKS_CNI_Policy`

---

## Kubernetes Manifests

1. **Create a GitHub repository** for Kubernetes manifests and clone it to your local machine.
2. **Create three YAML manifest files**:
   - **Secrets file**: Mount AWS secrets to EKS.
   - **Deployment file**: Manage scaling and deployment of the application.
   - **Service file**: Expose the application using a load balancer.

---

## EKS Deployment

1. **Configure kubectl** with the correct AWS region, EKS cluster name, and AWS account ID.
2. **Create a namespace** in the EKS cluster.
3. **Install the secret store CSI driver** on EKS to allow it to access secrets stored in AWS Secrets Manager.
4. **Associate IAM OIDC provider** with the EKS cluster.
5. **Create IAM service account** for EKS to access AWS resources.
6. **Apply the manifest files**:
    - Apply the `secrets.yaml` file.
    - Apply the `deployment.yaml` file.
    - Apply the `service.yaml` file.

7. **Configure the Load Balancer** and apply a TLS listener for secure communication.
8. **Create a Route 53 A record set** to map the application’s IP address to a domain name.

---

## Conclusion

After completing the steps outlined in this guide, you will have successfully deployed a dynamic application using Kubernetes on AWS EKS. The infrastructure is highly available, secure, and scalable, with the application accessible via a domain name with encrypted traffic.


