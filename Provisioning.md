# Terraform Files for Various Projects

This document showcases a collection of **Terraform configurations** I developed across multiple backend-focused projects. Each module and setup reflects the specific needs of the project, ranging from provisioning **Kubernetes clusters**, **Redis caching**, **EC2 instances**, and **custom VPCs** to setting up infrastructure for scalability and performance. The files demonstrate my ability to provision AWS infrastructure and integrate backend services, emphasizing modularity and reusability.

---

## Overview of Projects

### Custom VPC for Secure Networking\*\*

- Provisioned a custom **VPC** with public and private subnets to host backend services securely.
- Configured public subnets for load balancers and private subnets for sensitive resources like databases and caches.

### EKS Cluster for Microservices\*\*

- Set up a **Kubernetes cluster** to host microservices for a backend-heavy application.
- Configured worker nodes, autoscaling groups, and networking to ensure high availability.

### Redis Cache for Session Management\*\*

- Deployed a **Redis cluster** to handle caching and session data for a high-traffic application.
- Integrated Redis with existing backend services to optimize performance.

### EC2 Instances for Legacy Applications\*\*

- Provisioned EC2 instances for deploying monolithic legacy applications.
- Configured security groups, AMIs, and scaling policies for optimized performance.

## Root Files

### `main.tf`

```hcl

terraform {
  backend "s3" {
    bucket         = "my-backend-terraform-state"
    key            = "terraform/state"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock"
  }
}


provider "aws" {
  region = var.aws_region
}

# VPC Module
module "vpc" {
  source         = "./modules/vpc"
  vpc_name       = var.vpc_name
  cidr_block     = var.vpc_cidr
  public_subnets = var.public_subnets
  private_subnets = var.private_subnets
}

# EKS Cluster Module
module "eks_cluster" {
  source          = "./modules/eks-cluster"
  cluster_name    = var.eks_cluster_name
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.public_subnets
  node_instance_type = var.node_instance_type
}

# Redis Module
module "redis" {
  source        = "./modules/redis"
  vpc_id        = module.vpc.vpc_id
  subnet_ids    = module.vpc.private_subnets
  redis_name    = var.redis_name
  node_type     = var.redis_node_type
}

# EC2 Instances Module
module "ec2_instances" {
  source           = "./modules/ec2-instances"
  instance_count   = var.instance_count
  instance_type    = var.instance_type
  ami_id           = var.ami_id
  subnet_id        = module.vpc.public_subnets[0]
  security_group_id = module.vpc.security_group_id
}

variable "aws_region" {
  description = "AWS region for the infrastructure"
  default     = "us-east-1"
}

variable "vpc_name" {
  description = "Name of the VPC"
  default     = "backend-vpc"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "public_subnets" {
  description = "CIDR blocks for public subnets"
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnets" {
  description = "CIDR blocks for private subnets"
  default     = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "eks_cluster_name" {
  description = "Name of the EKS cluster"
  default     = "backend-eks"
}

variable "node_instance_type" {
  description = "Instance type for EKS worker nodes"
  default     = "t3.medium"
}

variable "redis_name" {
  description = "Name of the Redis cluster"
  default     = "backend-redis"
}

variable "redis_node_type" {
  description = "Node type for Redis"
  default     = "cache.t3.micro"
}

variable "instance_count" {
  description = "Number of EC2 instances"
  default     = 2
}

variable "instance_type" {
  description = "Type of EC2 instances"
  default     = "t3.micro"
}

variable "ami_id" {
  description = "AMI ID for EC2 instances"
  default     = "ami-12345678"
}

resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
  tags = {
    Name = var.vpc_name
  }
}

resource "aws_subnet" "public" {
  count                   = length(var.public_subnets)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnets[count.index]
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.vpc_name}-public-${count.index + 1}"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.vpc_name}-private-${count.index + 1}"
  }
}


module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  cluster_name    = var.cluster_name
  cluster_version = "1.25"
  vpc_id          = var.vpc_id
  subnet_ids      = var.subnet_ids
  node_groups = {
    worker_group = {
      desired_capacity = 2
      max_capacity     = 3
      min_capacity     = 1
      instance_type    = var.node_instance_type
    }
  }
}


resource "aws_elasticache_subnet_group" "redis" {
  name       = "${var.redis_name}-subnet-group"
  subnet_ids = var.subnet_ids
}

resource "aws_elasticache_cluster" "redis" {
  cluster_id           = var.redis_name
  engine               = "redis"
  node_type            = var.node_type
  num_cache_nodes      = 1
  subnet_group_name    = aws_elasticache_subnet_group.redis.name
  security_group_ids   = [var.security_group_id]

  tags = {
    Name = var.redis_name
  }
}
```
