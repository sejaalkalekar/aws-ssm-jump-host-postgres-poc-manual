# AWS SSM Jump Host + Private PostgreSQL RDS POC — Manual Version

A hands-on AWS Cloud & DevOps project demonstrating how to securely access a **private PostgreSQL RDS database** from a local Windows machine without exposing the database to the public internet.

> **Implementation:** Manual AWS Console + local configuration
> **Terraform:** Not used in this version
> **Project Status:** Manual implementation completed
> **Next Phase:** Rebuild the same architecture using Terraform

---

## 🎯 Project Objective

The objective of this Proof of Concept (POC) is to create a secure way for a developer to connect from a local machine to a **private PostgreSQL RDS database** inside AWS.

The solution uses:

* Amazon VPC
* Private Subnets
* Private EC2 Jump Host
* AWS Systems Manager Session Manager
* SSM Port Forwarding
* Amazon RDS for PostgreSQL
* Security Groups
* IAM
* HAProxy
* Windows Hosts File
* PostgreSQL `psql`
* TLS encryption

The database is **not exposed to the public internet**.

---

# 🏗️ Architecture

```text
                         LOCAL WINDOWS MACHINE
                                  │
                                  │ PostgreSQL Client
                                  ▼
                  Real RDS Hostname
                                  │
                                  │ Windows Hosts Mapping
                                  ▼
                            127.0.0.1
                                  │
                                  ▼
                              HAProxy
                              :5432
                                  │
                                  ▼
                         SSM Port Forwarding
                             :15432
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │     EC2 Jump Host       │
                    │     Private Subnet      │
                    │     10.0.1.112           │
                    │     No Public IP        │
                    └────────────┬────────────┘
                                 │
                                 │ PostgreSQL :5432
                                 ▼
                    ┌─────────────────────────┐
                    │     Amazon RDS          │
                    │     PostgreSQL          │
                    │     Private Subnet      │
                    │     10.0.2.5            │
                    └─────────────────────────┘
```

---

# 🔐 Security Architecture

The project follows a private-access model.

### EC2 Jump Host

* Located in a private subnet
* No public IP address
* No inbound SSH access
* No SSH keys required
* Managed through AWS Systems Manager

### PostgreSQL RDS

* Deployed in private subnets
* No public access
* PostgreSQL port: `5432`
* Security Group allows PostgreSQL traffic from the Jump Host Security Group

### AWS Systems Manager

Session Manager is used to access the Jump Host without opening SSH port `22`.

SSM Port Forwarding provides the connection path from the local machine to the private RDS database.

---

# ☁️ AWS Resources

The following AWS resources were created manually:

### Networking

* VPC
* Private Subnet 1
* Private Subnet 2
* Route Table
* Route Table Associations

### VPC Endpoints

* SSM Endpoint
* SSM Messages Endpoint
* EC2 Messages Endpoint

### Security Groups

* SSM VPC Endpoint Security Group
* Jump Host Security Group
* RDS PostgreSQL Security Group

### IAM

* IAM Role for EC2
* `AmazonSSMManagedInstanceCore`

### Compute

* Private EC2 Jump Host
* Instance Type: `t3.micro`

### Database

* Amazon RDS PostgreSQL
* RDS Subnet Group
* Private RDS Security Group

### Local Components

* PostgreSQL `psql` client
* HAProxy
* Windows Hosts File
* AWS CLI

---

# 🔄 Connection Flow

The final connection path is:

```text
Windows PostgreSQL Client
            │
            ▼
Real RDS Hostname
            │
            ▼
Windows hosts file
            │
            ▼
127.0.0.1:5432
            │
            ▼
         HAProxy
            │
            ▼
127.0.0.1:15432
            │
            ▼
   SSM Port Forwarding
            │
            ▼
     EC2 Jump Host
            │
            ▼
      Private RDS
            │
            ▼
     PostgreSQL :5432
```

---

# 🛠️ Manual Implementation

This project was completed **entirely manually**.

No Terraform, CloudFormation, CDK, or other Infrastructure as Code tool was used to create the infrastructure.

The resources were created and configured using:

* AWS Management Console
* AWS CLI
* Windows PowerShell
* PostgreSQL `psql`
* HAProxy
* Windows Hosts File

---

# 📋 Implementation Steps

## Step 1 — Prepare Laptop

Installed and configured the required local tools:

* AWS CLI
* PostgreSQL client
* HAProxy
* PowerShell

---

## Step 2 — Create VPC

Created the project VPC:

```text
Name: ssm-jump-host-vpc
CIDR: 10.0.0.0/16
```

DNS resolution was enabled.

---

## Step 3 — Create Private Subnets

Created two private subnets:

```text
private-subnet-1
CIDR: 10.0.1.0/24
AZ: ap-south-1a
```

```text
private-subnet-2
CIDR: 10.0.2.0/24
AZ: ap-south-1b
```

---

## Step 4 — Create Route Table

Created the private route table:

```text
ssm-jump-host-private-rt
```

Associated it with the private subnets.

---

## Step 5 — Create SSM VPC Endpoints

Created interface endpoints for:

```text
ssm
ssmmessages
ec2messages
```

These endpoints allow the private EC2 instance to communicate with AWS Systems Manager without requiring internet access.

---

## Step 6 — Create Endpoint Security Group

Created:

```text
ssm-vpc-endpoint-sg
```

Allowed HTTPS traffic required for Systems Manager communication.

---

## Step 7 — Create IAM Role

Created:

```text
ssm-jump-host-role
```

Attached:

```text
AmazonSSMManagedInstanceCore
```

This allows the EC2 instance to register with Systems Manager.

---

## Step 8 — Create Jump Host Security Group

Created:

```text
ssm-jump-host-sg
```

No inbound SSH access was required.

---

## Step 9 — Launch Private EC2

Launched the Jump Host in the private subnet.

Example:

```text
Instance ID: i-0e24c09235ec9db01
Instance Type: t3.micro
Private IP: 10.0.1.112
Public IP: None
```

---

## Step 10 — Verify SSM Registration

Verified that the EC2 instance appeared as a managed instance in AWS Systems Manager.

---

## Step 11 — Test Session Manager

Successfully connected to the private EC2 instance using Session Manager.

No SSH key or inbound port 22 was required.

---

## Step 12 — Create RDS Subnet Group

Created an RDS subnet group using the private subnets.

---

## Step 13 — Create Private PostgreSQL RDS

Created a PostgreSQL RDS database in the private network.

Example:

```text
Engine: PostgreSQL
Port: 5432
Public Access: No
```

---

## Step 14 — Configure RDS Security Group

Configured the RDS Security Group to allow:

```text
PostgreSQL
TCP
5432
Source: Jump Host Security Group
```

This prevents direct access from arbitrary sources.

---

## Step 15 — Test Jump Host → RDS

Verified that the Jump Host could reach the RDS PostgreSQL endpoint on port `5432`.

---

## Step 16 — Configure SSM Port Forwarding

Created an SSM port-forwarding session:

```text
Local Port: 15432
Remote Host: RDS PostgreSQL
Remote Port: 5432
```

The local port forwards traffic through the private Jump Host to the private RDS database.

---

## Step 17 — Configure HAProxy

HAProxy was configured in TCP mode.

Example configuration:

```text
global
    log stdout format raw local0

defaults
    mode tcp
    timeout connect 10s
    timeout client  1m
    timeout server  1m

frontend postgres
    bind *:5432
    default_backend postgres_rds

backend postgres_rds
    server rds 127.0.0.1:15432
```

HAProxy listens on local port `5432` and forwards traffic to the SSM tunnel on port `15432`.

---

## Step 18 — Configure Hostname Mapping

The real RDS hostname was mapped to localhost in the Windows hosts file.

Example:

```text
127.0.0.1    ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com
```

This allows applications to continue using the **real RDS hostname** while traffic is redirected through the local HAProxy/SSM path.

---

## Step 19 — Test PostgreSQL Using Real Hostname

PostgreSQL was successfully accessed using the real RDS hostname.

Example:

```text
psql -h ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com -p 5432 -U postgres -d postgres
```

The connection resolved to:

```text
Host Address: 127.0.0.1
```

PostgreSQL successfully returned the server version.

---

## Step 20 — Verify TLS

TLS was successfully verified through the SSM forwarding path.

The successful connection reported:

```text
SSL connection
protocol: TLSv1.3
cipher: TLS_AES_256_GCM_SHA384
compression: off
```

This confirms that the PostgreSQL connection was encrypted using TLS 1.3.

---

# 🧪 Final Validation

The final architecture successfully demonstrated:

```text
Local Windows Machine
        │
        ▼
Real RDS Hostname
        │
        ▼
Hostname Mapping
        │
        ▼
HAProxy
        │
        ▼
SSM Port Forwarding
        │
        ▼
Private EC2 Jump Host
        │
        ▼
Private PostgreSQL RDS
```

The RDS database remained private while allowing controlled access from the local machine.

---

# 📸 Evidence

Screenshots documenting the implementation are stored in:

```text
/evidence
```

Important evidence includes:

```text
32-hostname-mapping.png
33-real-hostname-postgresql-test.png
34-ssm-port-forwarding-active.png
35-ssm-tunnel-port-verification.png
36-tls-verification.png
```

Additional screenshots document the AWS infrastructure created during the earlier implementation steps.

---

# 🔒 Security Considerations

This project intentionally avoids exposing PostgreSQL to the public internet.

### No Public RDS

The RDS database is private.

### No SSH

The Jump Host does not require inbound SSH access.

### IAM-Based Access

AWS Systems Manager is used for access to the Jump Host.

### Security Group Restrictions

RDS PostgreSQL access is restricted to the Jump Host Security Group.

### TLS

The PostgreSQL connection was verified using TLS 1.3.

---

# ⚠️ Security Warning

Never commit the following to GitHub:

```text
AWS Access Key
AWS Secret Access Key
RDS Password
Private Credentials
Session Tokens
.env files containing secrets
```

Use placeholders when documenting credentials.

---

# 📚 What I Learned

This project helped me practice:

* AWS VPC networking
* Private subnet design
* VPC endpoints
* IAM roles
* EC2
* Security Groups
* Amazon RDS
* PostgreSQL
* AWS Systems Manager
* Session Manager
* SSM Port Forwarding
* HAProxy TCP proxying
* Windows hostname mapping
* PostgreSQL TLS
* Secure private-resource access
* AWS troubleshooting
* Cloud infrastructure documentation

---

# 🚀 Future Terraform Version

This repository represents the **Manual Version** of the project.

Terraform was intentionally **not used** in this implementation.

The next phase will recreate the **same architecture using Terraform**.

The future Terraform version will provision the same major components using Infrastructure as Code:

```text
VPC
│
├── Private Subnets
│
├── Route Tables
│
├── SSM VPC Endpoints
│
├── Security Groups
│
├── IAM Role
│
├── Private EC2 Jump Host
│
└── Private PostgreSQL RDS
```

The Terraform implementation will focus on:

* Infrastructure as Code
* Repeatability
* Variables
* Outputs
* Terraform state
* Resource dependencies
* Automated infrastructure provisioning

The manual and Terraform versions will therefore demonstrate both:

**Understanding how to build the infrastructure manually**

and

**Understanding how to automate the same infrastructure with Terraform.**

---

# 📌 Project Status

```text
Manual AWS Implementation
        ✅ COMPLETE
```

---

## 👩‍💻 Project Type

**AWS Cloud & DevOps Hands-on Project**

**Category:** AWS / DevOps / Networking / PostgreSQL

**Implementation:** Manual

**Automation:** Terraform version planned separately

**Status:** Manual POC Completed
