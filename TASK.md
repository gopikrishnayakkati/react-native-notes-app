# GitOps Pipeline Setup for Terraform AWS Infrastructure & Deployment

## Overview
This guide helps you set up a GitOps pipeline to deploy AWS infrastructure using Terraform. The pipeline automates deployments when code is pushed to the `main` branch.

## Prerequisites
- A GitHub repository containing Terraform configuration files.
- AWS account with appropriate IAM permissions.
- GitHub Secrets configured:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`
- GitHub Actions Pipeline Documentation link[Refer Here](https://docs.github.com/en/actions)
- Terraform Documentation link [Refer Here](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- Docker file Reference link [Refer Here](https://docs.docker.com/reference/dockerfile/) 
- Docker Cheat Sheet link [Refere Here](https://docs.docker.com/get-started/docker_cheatsheet.pdf)
## Pipeline Configuration

```yaml
name: GitOps Pipeline Deployment

on:
  push:
    branches:
      - main

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Set up Terraform
      uses: hashicorp/setup-terraform@v1
      with:
        terraform_version: 1.4.0

    - name: AWS Credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-south-1

    - name: Initialize Terraform
      run: |
        cd terraform
        terraform init -reconfigure

    - name: Create Terraform Workspace
      run: |
        cd terraform
        terraform workspace new Node.js || terraform workspace select Node.js

    - name: Apply Terraform Configuration
      run: |
        cd terraform
        terraform apply -var-file="Node.js-environment.tfvars" -auto-approve
```

## Pipeline Breakdown

### Trigger Event
The workflow triggers on a push to the `main` branch.

### Steps Explained
1. **Checkout Repository:** Retrieves the latest code.
   ```yaml
   - name: Checkout repository
     uses: actions/checkout@v2
   ```

2. **Set Up Terraform:** Installs Terraform version `1.4.0`.
   ```yaml
   - name: Set up Terraform
     uses: hashicorp/setup-terraform@v1
     with:
       terraform_version: 1.4.0
   ```

3. **Configure AWS Credentials:** Sets up AWS access.
   ```yaml
   - name: AWS Credentials
     uses: aws-actions/configure-aws-credentials@v1
     with:
       aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
       aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
       aws-region: ap-south-1
   ```

4. **Initialize Terraform:** Initializes the Terraform directory.
   ```yaml
   - name: Initialize Terraform
     run: |
       cd terraform
       terraform init -reconfigure
   ```

5. **Create or Select Workspace:** Manages Terraform workspaces.
   ```yaml
   - name: Create Terraform Workspace
     run: |
       cd terraform
       terraform workspace new Node.js || terraform workspace select Node.js
   ```

6. **Apply Terraform Configuration:** Applies the infrastructure changes.
   ```yaml
   - name: Apply Terraform Configuration
     run: |
       cd terraform
       terraform apply -var-file="Node.js-environment.tfvars" -auto-approve
   ```

# Provisioning an AWS VM Using Terraform

## Overview
This guide provides a step-by-step approach to provisioning an AWS Virtual Machine (EC2 instance) using Terraform. By following these instructions, you can create, manage, and deploy cloud resources efficiently.

## Prerequisites

- AWS account with appropriate IAM permissions.
- Terraform installed on your local machine or CI/CD environment.
- AWS CLI configured with valid credentials.
- A set of Terraform configuration files.

## Terraform Configuration Files
Below is an example structure for your Terraform project:

```
terraform/
├── main.tf
├── provider.tf
├── variables.tf
├── outputs.tf
└── Node.js-environment.tfvars
```

### 1. **main.tf**
```hcl
resource "aws_vpc" "main_vpc" {
  cidr_block = var.vpc_cidr

  tags = {
    Name = "MainVPC"
  }
}
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = var.subnet_cidr
  availability_zone = "ap-south-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "MainSubnet"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id
}

resource "aws_route_table" "main_rt" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }
}

resource "aws_route_table_association" "main_rta" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_rt.id
}

resource "tls_private_key" "generated_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "generated_key" {
  key_name   = "generated-key"
  public_key = tls_private_key.generated_key.public_key_openssh
}

# Save private key to a local file
resource "local_file" "private_key" {
  content  = tls_private_key.generated_key.private_key_pem
  filename = "generated-key.pem"
}
resource "aws_security_group" "allow_ssh" {
  vpc_id = aws_vpc.main_vpc.id

  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] 
  }
   ingress {
    from_port        = 8081
    to_port          = 8081
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Allow ssh and 8081"
  }
}
# EC2 Instance
resource "aws_instance" "react_native_notes" {
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = aws_key_pair.generated_key.key_name
  subnet_id     = aws_subnet.main_subnet.id
  security_groups = [aws_security_group.allow_ssh.id]

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update -y",
      "curl -fsSL https://get.docker.com -o install-docker.sh",
      "sudo sh install-docker.sh",
      "sudo docker info",
      "git clone https://github.com/gopikrishnayakkati/react-native-notes-app.git",
      "cd react-native-notes-app",
      "ls -al",
      "sudo docker build -t react-native-notes:1.0 .",
      "sudo docker run -d -p 8081:8081 react-native-notes:1.0 ",
      "sudo docker container ls -a"
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file(local_file.private_key.filename)
      host        = self.public_ip
    }
  }

  tags = {
    Name = "ReactNativeNotesAppInstance"
  }
}
```

### 2. **variables.tf**
```hcl
variable "aws_region" {
  description = "AWS region"
  default     = "ap-south-1"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  default     = "10.0.0.0/16"
}

variable "subnet_cidr" {
  description = "Subnet CIDR block"
  default     = "10.0.0.0/24"
}

variable "ami_id" {
  description = "AMI ID for Ubuntu"
  default     = "ami-00bb6a80f01f03502"
}

variable "instance_type" {
  description = "EC2 instance type"
  default     = "t2.small"
}

variable "key_name" {
  description = "my Key Pair Name"
}

variable "private_key_path" {
  description = "Path to the private key file"
  default     = "generated-key.pem"
}
```

### 3. **outputs.tf**
```hcl
output "instance_public_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.react_native_notes.public_ip
}
```

### 4. **Node.js-environment.tfvars**
```hcl
aws_region      = "ap-south-1"
key_name        = "generated-key"
private_key_path = "generated-key.pem"

```

## Provisioning the AWS VM

### Step 1: Initialize Terraform
Run the following command to initialize the Terraform working directory:
```bash
terraform init
```

### Step 2: Plan the Deployment
Review the changes Terraform will apply:
```bash
terraform plan
```

### Step 3: Apply the Configuration
Apply the configuration to provision the VM:
```bash
terraform apply -auto-approve -var-file="Node.js-envronment.tfvars"
```

### Step 5: Clean Up Resources
* We create pipeline as well to destroy the Infrastructure, run the below command :
```bash
terraform destroy -auto-approve -var-file="Node.js-envronment.tfvars"
```

## Deploying an Application in Docker Container

### Dockerfile Configuration
Below is an example Dockerfile to deploy an application using Node.js in a Docker container:

```Dockerfile
FROM node:18-bullseye
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8081
CMD ["npx", "react-native", "start"]
```

### Key Dockerfile Instructions
- **Base Image:**
  ```Dockerfile
  FROM node:18-bullseye
  ```
  This line specifies the base image as Node.js version 18 on Debian Bullseye.

- **Set Working Directory:**
  ```Dockerfile
  WORKDIR /app
  ```
  Defines `/app` as the working directory inside the container.

- **Copy Dependencies:**
  ```Dockerfile
  COPY package*.json ./
  ```
  Copies `package.json` and `package-lock.json` files to the working directory.

- **Install Dependencies:**
  ```Dockerfile
  RUN npm install
  ```
  Installs the required Node.js packages.

- **Copy Application Files:**
  ```Dockerfile
  COPY . .
  ```
  Copies all files from the current directory to the container.

- **Expose Port:**
  ```Dockerfile
  EXPOSE 8081
  ```
  Opens port 8081 for external access.

- **Start Application:**
  ```Dockerfile
  CMD ["npx", "react-native", "start"]
  ```
  Specifies the command to run the application.

### Building and Running the Docker Container

#### Step 1: Build the Docker Image
```bash
sudo docker build -t react-native-notes:1.0 .
```

#### Step 2: Run the Docker Container
```bash
sudo docker run -d -p 8081:8081 react-native-notes:1.0
```
#### Step 3: List Running Containers
```bash
sudo docker container ls -a
```
#### Step 4: Access the Application
Visit `http://Public-iP:8081` to access the running application.

