# Kubernetes Cluster on AWS EC2 using Terraform + Ansible

This guide explains **step-by-step how to create a Kubernetes cluster on AWS EC2 instances** using:

* **AWS EC2 + VPC networking**
* **Terraform** for infrastructure provisioning
* **Ansible** for Kubernetes cluster setup with `kubeadm`
* **Flannel** as the CNI network plugin
* **Metrics Server** for cluster metrics
* **NGINX Ingress Controller** for external traffic routing

The process is split into two layers:

1️⃣ **AWS Infrastructure Setup**
2️⃣ **Kubernetes Cluster Deployment using Ansible**

---

# Architecture

```
AWS Cloud
│
├── VPC
│    ├── Public Subnet
│    ├── Internet Gateway
│    └── Route Table
│
├── Security Group
│
├── EC2 Instances
│    ├── Master Node
│    └── Worker Node(s)
│
└── SSH Access
        │
        ▼
Terraform Provisioning
        │
        ▼
Ansible Kubernetes Setup
        │
        ├── Install containerd
        ├── Install kubeadm / kubelet / kubectl
        ├── Initialize master node
        ├── Join worker nodes
        ├── Install Flannel networking
        ├── Install Metrics Server
        └── Install Ingress Controller
```

---

# Prerequisites

You must have:

* AWS Account
* SSH Key Pair
* Local machine with internet access

Install the following tools locally.

---

# Install Required Tools

## Install Terraform

Download Terraform from:

```
https://developer.hashicorp.com/terraform/downloads
```

Verify installation:

```bash
terraform -version
```

---

## Install Ansible

```bash
pip install ansible
```

Verify:

```bash
ansible --version
```

---

## Install AWS CLI

Mac:

```bash
brew install awscli
```

Linux:

```bash
sudo apt install awscli
```

Verify:

```bash
aws --version
```

---

# Configure AWS Credentials

Run:

```bash
aws configure
```

Enter:

```
AWS Access Key ID
AWS Secret Access Key
Region: ap-south-1
Output Format: json
```

Test:

```bash
aws ec2 describe-regions
```

---

# Step 1 — Create AWS SSH Key Pair

Create a key pair:

AWS Console → EC2 → **Key Pairs**

Create new key:

```
k8s-key
```

Download:

```
k8s-key.pem
```

Set permission:

```bash
chmod 400 k8s-key.pem
```

Move to ssh directory:

```bash
mv k8s-key.pem ~/.ssh/
```

---

# Step 2 — Create Project Structure

```
k8s-aws-cluster/
│
├── terraform/
│   ├── provider.tf
│   ├── vpc.tf
│   ├── security-group.tf
│   ├── ec2.tf
│   └── outputs.tf
│
├── ansible/
│   ├── inventory.ini
│   ├── site.yml
│   └── roles/
│       ├── common
│       ├── master
│       ├── worker
│       ├── metrics
│       └── ingress
│
└── README.md
```

---

# Step 3 — Terraform AWS Infrastructure

Go to Terraform folder:

```bash
cd terraform
```

---

# provider.tf

```hcl
provider "aws" {
  region = "ap-south-1"
}
```

---

# Create VPC

vpc.tf

```hcl
resource "aws_vpc" "k8s_vpc" {
  cidr_block = "10.0.0.0/16"
}
```

---

# Create Subnet

```hcl
resource "aws_subnet" "k8s_subnet" {
  vpc_id     = aws_vpc.k8s_vpc.id
  cidr_block = "10.0.1.0/24"
}
```

---

# Internet Gateway

```hcl
resource "aws_internet_gateway" "k8s_igw" {
  vpc_id = aws_vpc.k8s_vpc.id
}
```

---

# Route Table

```hcl
resource "aws_route_table" "k8s_route" {
  vpc_id = aws_vpc.k8s_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.k8s_igw.id
  }
}
```

---

# Associate Route Table

```hcl
resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.k8s_subnet.id
  route_table_id = aws_route_table.k8s_route.id
}
```

---

# Security Group

security-group.tf

```hcl
resource "aws_security_group" "k8s_sg" {

  vpc_id = aws_vpc.k8s_vpc.id

  ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 6443
    to_port = 6443
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 10250
    to_port = 10250
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 30000
    to_port = 32767
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 8472
    to_port = 8472
    protocol = "udp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

}
```

---

# Step 4 — Create EC2 Instances

ec2.tf

```hcl
resource "aws_instance" "master" {

  ami           = "ami-0f5ee92e2d63afc18"
  instance_type = "t3.medium"

  subnet_id = aws_subnet.k8s_subnet.id

  vpc_security_group_ids = [
    aws_security_group.k8s_sg.id
  ]

  key_name = "k8s-key"

  tags = {
    Name = "k8s-master"
  }

}
```

Worker Node

```hcl
resource "aws_instance" "worker" {

  ami           = "ami-0f5ee92e2d63afc18"
  instance_type = "t3.medium"

  subnet_id = aws_subnet.k8s_subnet.id

  vpc_security_group_ids = [
    aws_security_group.k8s_sg.id
  ]

  key_name = "k8s-key"

  tags = {
    Name = "k8s-worker"
  }

}
```

---

# Step 5 — Create Infrastructure

Initialize Terraform:

```bash
terraform init
```

Preview changes:

```bash
terraform plan
```

Deploy:

```bash
terraform apply
```

Type:

```
yes
```

---

# Step 6 — Get Instance Public IP

```bash
terraform state list
```

Then:

```bash
terraform show
```

Or create outputs:

outputs.tf

```hcl
output "master_ip" {
 value = aws_instance.master.public_ip
}

output "worker_ip" {
 value = aws_instance.worker.public_ip
}
```

Run:

```bash
terraform output
```

Example:

```
master_ip = 13.x.x.x
worker_ip = 3.x.x.x
```

---

# Step 7 — Configure Ansible Inventory

Go to ansible folder.

Edit inventory.ini

```
[masters]
master ansible_host=MASTER_IP

[workers]
worker ansible_host=WORKER_IP

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/k8s-key.pem
ansible_python_interpreter=/usr/bin/python3
```

---

# Step 8 — Test Connectivity

Run:

```bash
ansible all -i inventory.ini -m ping
```

Expected:

```
master | SUCCESS
worker | SUCCESS
```

---

# Step 9 — Deploy Kubernetes using Ansible

Run playbook:

```bash
ansible-playbook -i inventory.ini site.yml
```

The playbook performs:

* Disable swap
* Install containerd
* Install Kubernetes packages
* Initialize control plane
* Configure kubeconfig
* Install Flannel networking
* Generate join token
* Join worker nodes
* Install metrics-server
* Install ingress controller

---

# Step 10 — Verify Cluster

SSH into master:

```bash
ssh ubuntu@MASTER_IP
```

Export kubeconfig:

```bash
export KUBECONFIG=$HOME/.kube/config
```

Check nodes:

```bash
kubectl get nodes
```

Expected:

```
master   Ready
worker   Ready
```

---

# Check System Pods

```bash
kubectl get pods -A
```

---

# Check Metrics

```bash
kubectl top nodes
```

---

# Check Ingress

```bash
kubectl get pods -n ingress-nginx
```

---

# Scaling Cluster

Add new worker node to inventory:

```
[workers]
worker1 ansible_host=IP1
worker2 ansible_host=IP2
```

Run playbook again:

```bash
ansible-playbook -i inventory.ini site.yml
```

Node joins automatically.

---

# Destroy Infrastructure

```bash
terraform destroy
```

Confirm:

```
yes
```

All AWS resources will be removed.

---

# Future Improvements

Possible enhancements:

* HA control plane (3 masters)
* Prometheus + Grafana monitoring
* ArgoCD GitOps
* Dynamic AWS inventory
* Automated node scaling
* Etcd backup automation

---

# Author

Felix Harrison
DevOps Engineer

Stack used:

* Terraform
* Ansible
* Kubernetes
* AWS EC2
* Flannel
* NGINX Ingress
* Metrics Server
