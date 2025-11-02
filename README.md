# Terraform Infrastructure Setup

## Overview
This project provisions cloud infrastructure using Terraform. It automates the setup of networking, IAM policies, S3, RDS, EC2 auto-scaling, load balancing, and integrates CloudWatch monitoring. It also handles secure credential storage using Secrets Manager and SSL certificate management using both ACM and imported certificates.

## Objectives
- Implement Infrastructure as Code (IaC)
- Set up KMS keys with 90-day rotation for EC2, RDS, S3, and Secrets Manager
- Encrypt sensitive data and integrate KMS into resource definitions
- Store RDS credentials securely using Secrets Manager
- Configure IAM roles for EC2, RDS, S3, and CloudWatch
- Create a secure and scalable auto-scaling web application stack
- Secure endpoints using HTTPS and SSL certificates

## Security & Encryption
### AWS Key Management Service (KMS)
- Separate KMS keys are created for:
  - EC2
  - RDS
  - S3 Buckets
  - Secrets Manager (Database Password & Email Credentials)
- All KMS keys have automatic rotation enabled (90-day rotation).
- Resources (S3, RDS, Secrets, EC2 volumes) are encrypted using these keys.

### Secrets Manager
- RDS database password is generated via Terraform and stored securely using Secrets Manager with a custom KMS key.
- Secret is retrieved in the EC2 user-data script at boot time for application configuration.

## SSL Certificates
- **Dev Environment**: Uses ACM to issue and attach certificates to the ALB.
- **Demo Environment**:
  - SSL cert is manually requested from Namecheap (or another provider).
  - Cert is imported into AWS ACM using CLI.
  - ALB uses the imported certificate for HTTPS.

#### ACM Certificate Import Command
```bash
aws acm import-certificate \
  --certificate fileb://certificate.crt \
  --private-key fileb://private.key \
  --certificate-chain fileb://ca_bundle.crt \
  --region us-east-1
```

---

## Infrastructure Breakdown

### VPC and Networking
- VPC with public and private subnets
- Internet Gateway and route tables

### IAM Roles & Policies
- Roles for EC2, RDS, S3, CloudWatch
- Policies for resource access, logging, encryption, and secret access

### S3 Bucket
- Private with encryption using KMS
- Lifecycle rule: transition to `STANDARD_IA` after 30 days
- Versioning and force destroy enabled

### RDS PostgreSQL
- Encrypted with custom KMS key
- Subnet group: private subnets only
- Public access: Disabled
- Password stored in Secrets Manager
- Parameter group enforces SSL settings and max connections

### EC2 Auto Scaling Group
- Launch Template with custom AMI
- Auto-retrieves DB password from Secrets Manager
- Logs sent to CloudWatch
- Minimum 3, Maximum 5 instances
- Scales based on CPU threshold

### Load Balancer
- ALB configured with listeners for:
  - HTTP (80)
  - HTTPS (443, with imported or ACM cert)
- Security group only allows inbound 80/443
- Instances only accept traffic from ALB

### Route53
- DNS names: `dev.fetchme.me`, `demo.fetchme.me`
- Route53 alias records point to the ALB

---

## CI/CD Workflow

### Trigger: On Pull Request Merge

1. Run Unit Tests
2. Validate Packer Templates
3. Build Application Artifacts
4. Build AMI in **DEV** account
5. Upgrade OS and install system dependencies
6. Install app dependencies and configure startup
7. Share AMI with **DEMO** account
8. Reconfigure GitHub AWS CLI to use DEMO credentials
9. Create a new Launch Template version
10. Update ASG to use new Launch Template
11. Trigger ASG Instance Refresh
12. Wait for refresh to complete (GitHub Actions workflow status matches refresh status)

---

## Deployment Instructions

### Prerequisites
- Terraform
- AWS CLI with IAM credentials (set via env or GitHub Secrets)

### Deploy Steps
```bash
terraform init
terraform validate
terraform plan
terraform apply -auto-approve
```

### Destroy Resources
```bash
terraform destroy -auto-approve
```

---

## Monitoring & Logging

### CloudWatch Metrics
- API Call Count and Latency
- DB Query Time
- S3 Interaction Time
- System Metrics (CPU, Memory, Disk)

### CloudWatch Logs
- App logs forwarded via CloudWatch Agent
- Logs include `INFO`, `WARN`, `ERROR`, and stack traces

#### Sample Log Messages
```txt
INFO: Health check passed
INFO: File uploaded: uploads/img.jpg
INFO: DB query time: 20ms
INFO: S3 response time: 35ms
```

---

## Best Practices
- Use encrypted resources with KMS
- Never expose EC2 instances directly to the internet
- Use Secrets Manager + KMS for secure credentials
- Automate everything with CI/CD
- Use HTTPS (with ACM or imported SSL) for all external access

