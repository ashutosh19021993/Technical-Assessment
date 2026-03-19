# Terraform Remote State Backend (S3 with Locking)

## Overview

Terraform remote state is configured using **Amazon S3** to store state files centrally, with **state locking enabled using S3 native lockfile**.

This setup ensures:

- safe concurrent execution of Terraform
- centralized and consistent state management
- recovery using versioned state files
- secure and production-ready infrastructure provisioning

---

# Backend Architecture

The Terraform backend uses:

### 1. Amazon S3
- stores the Terraform state file (`.tfstate`)
- provides durable and centralized storage

### 2. S3 Native Lockfile
- prevents concurrent Terraform operations
- uses `.tflock` file for locking

### 3. IAM Role (EC2 / Jenkins)
- Terraform runs using IAM role attached to EC2 (Jenkins)
- no static AWS credentials required
- improves security and auditability

---



## Terraform Backend Block

terraform {
  backend "s3" {
    bucket = "tf-state-252850512189"
    key    = "dev/terraform.tfstate"
    region = "us-east-1"

    # Recommended in real AWS
    encrypt = true
    dynamodb_table = "terraform-locks"   # optional, recommended
  }
}

terraform {
  backend "s3" {
    bucket = "tf-state-252850512189"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"

    # Recommended in real AWS
    encrypt = true
    dynamodb_table = "terraform-locks"   # optional, recommended
  }
}