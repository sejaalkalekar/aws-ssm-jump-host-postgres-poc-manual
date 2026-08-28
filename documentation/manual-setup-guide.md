# Manual Setup Guide — AWS SSM Jump Host + Private PostgreSQL RDS

## 1. Overview

This guide documents the **manual implementation** of the AWS SSM Jump Host + Private PostgreSQL RDS POC.

The complete infrastructure was created and configured manually using the AWS Management Console, AWS CLI, Windows PowerShell, PostgreSQL client, HAProxy, and the Windows Hosts file.

**Terraform was not used in this version.**

A separate Terraform version of the same project will be created later.

---

# 2. Project Objective

The objective is to securely connect from a local Windows machine to a private PostgreSQL RDS database inside AWS without exposing the database to the public internet.

The final connection path is:

```text
Windows Laptop
      │
      ▼
Real RDS Hostname
      │
      ▼
Windows Hosts File
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
AWS SSM Port Forwarding
      │
      ▼
Private EC2 Jump Host
      │
      ▼
Private PostgreSQL RDS
```

---

# 3. Prerequisites

The following tools were required on the local Windows machine:

* AWS CLI
* PostgreSQL client (`psql`)
* HAProxy
* Windows PowerShell

AWS requirements:

* AWS account
* IAM user with required permissions
* AWS region configured
* Permission to create VPC, EC2, RDS, IAM, Security Groups, and SSM resources

---

# 4. AWS Region

The project was created in:

```text
ap-south-1
```

---

# 5. VPC Configuration

Created the VPC:

```text
Name: ssm-jump-host-vpc
CIDR: 10.0.0.0/16
```

DNS resolution was enabled.

---

# 6. Private Subnets

Two private subnets were created.

### Private Subnet 1

```text
Name: private-subnet-1
CIDR: 10.0.1.0/24
Availability Zone: ap-south-1a
```

### Private Subnet 2

```text
Name: private-subnet-2
CIDR: 10.0.2.0/24
Availability Zone: ap-south-1b
```

The Jump Host was placed in the first private subnet.

The RDS subnet group used the private subnets.

---

# 7. Route Table

Created:

```text
ssm-jump-host-private-rt
```

The route table was associated with the private subnets.

---

# 8. SSM VPC Endpoints

Interface VPC endpoints were created for:

```text
com.amazonaws.ap-south-1.ssm
com.amazonaws.ap-south-1.ssmmessages
com.amazonaws.ap-south-1.ec2messages
```

These endpoints allow the private EC2 instance to communicate with AWS Systems Manager without requiring a public IP or internet-facing access.

---

# 9. Endpoint Security Group

Created:

```text
ssm-vpc-endpoint-sg
```

The Security Group allows HTTPS traffic required for Systems Manager communication.

---

# 10. IAM Role

Created the EC2 IAM role:

```text
ssm-jump-host-role
```

Attached policy:

```text
AmazonSSMManagedInstanceCore
```

This allows the EC2 instance to register with AWS Systems Manager.

---

# 11. Jump Host Security Group

Created:

```text
ssm-jump-host-sg
```

No inbound SSH access was required.

The Jump Host is accessed through AWS Systems Manager Session Manager.

---

# 12. Launch Private EC2

A private EC2 instance was launched as the Jump Host.

Configuration:

```text
Instance ID: i-0e24c09235ec9db01
Instance Type: t3.micro
Private IP: 10.0.1.112
Public IP: None
```

The IAM role `ssm-jump-host-role` was attached to the instance.

---

# 13. Verify SSM Registration

The EC2 instance was verified in AWS Systems Manager.

The instance successfully registered as a managed instance.

This confirmed that:

```text
Private EC2
     ↓
SSM VPC Endpoints
     ↓
AWS Systems Manager
```

was working.

---

# 14. Test Session Manager

A Session Manager session was started to access the private Jump Host.

The session provided a shell similar to:

```text
sh-5.2$
```

No SSH connection or SSH key was required.

---

# 15. Create RDS Subnet Group

An RDS subnet group was created using the private subnets:

```text
private-subnet-1
private-subnet-2
```

This allowed the RDS database to be deployed privately.

---

# 16. Create PostgreSQL RDS

A private Amazon RDS PostgreSQL database was created.

Configuration:

```text
Engine: PostgreSQL
Port: 5432
Public Access: No
```

The RDS endpoint was:

```text
ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com
```

The database remained inside the private AWS network.

---

# 17. Configure RDS Security Group

Created/configured the RDS Security Group:

```text
ssm-jump-host-rds-sg
```

Inbound rule:

```text
Type: PostgreSQL
Protocol: TCP
Port: 5432
Source: Jump Host Security Group
```

This allows PostgreSQL traffic from the Jump Host while preventing unrestricted access.

---

# 18. Test Jump Host → RDS

The Jump Host was used to verify that the RDS endpoint was reachable on port `5432`.

The database was not publicly accessible.

---

# 19. Configure SSM Port Forwarding

SSM port forwarding was configured from the local Windows machine.

The forwarding configuration was:

```text
Local Port: 15432
Remote Host: RDS PostgreSQL endpoint
Remote Port: 5432
```

Example command:

```powershell
aws ssm start-session --target i-0e24c09235ec9db01 --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters "host=ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com,portNumber=5432,localPortNumber=15432"
```

Successful output:

```text
Port 15432 opened for sessionId ...
Waiting for connections...
```

---

# 20. Configure HAProxy

HAProxy was installed and configured on the local Windows machine.

Configuration file:

```text
C:\haproxy\haproxy.cfg
```

Configuration:

```text
global
    log stdout format raw local0

defaults
    mode tcp
    timeout connect 10s
    timeout client 1m
    timeout server 1m

frontend postgres
    bind *:5432
    default_backend postgres_rds

backend postgres_rds
    server rds 127.0.0.1:15432
```

HAProxy listens on:

```text
127.0.0.1:5432
```

and forwards traffic to:

```text
127.0.0.1:15432
```

The SSM tunnel then forwards the connection to the private RDS database.

---

# 21. Configure Windows Hostname Mapping

The Windows Hosts file was modified:

```text
C:\Windows\System32\drivers\etc\hosts
```

The following entry was added:

```text
127.0.0.1    ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com
```

This makes the real RDS hostname resolve to localhost.

The hostname remains unchanged for the PostgreSQL client.

---

# 22. Test PostgreSQL Using Real Hostname

PostgreSQL was tested using the real RDS hostname:

```powershell
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -h ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com -p 5432 -U postgres -d postgres
```

The connection successfully reached PostgreSQL through the local forwarding setup.

The connection information showed:

```text
Host:
ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com

Host Address:
127.0.0.1

Server Port:
5432
```

PostgreSQL version was also successfully verified.

---

# 23. Verify TLS

TLS was verified by connecting through the SSM tunnel with SSL required.

Command:

```powershell
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" "host=127.0.0.1 port=15432 user=postgres dbname=postgres sslmode=require"
```

The successful connection reported:

```text
SSL connection (protocol: TLSv1.3,
cipher: TLS_AES_256_GCM_SHA384,
compression: off, ALPN: postgresql)
```

This confirms that the PostgreSQL connection was encrypted using TLS 1.3.

---

# 24. Final Connection Test

The complete connection path was successfully demonstrated:

```text
Local Windows Machine
        │
        ▼
Real RDS Hostname
        │
        ▼
Windows Hosts File
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
Private EC2 Jump Host
        │
        ▼
Private PostgreSQL RDS
```

---

# 25. Security Summary

The final design provides:

* Private RDS deployment
* No public RDS access
* Private EC2 Jump Host
* No inbound SSH requirement
* IAM-based SSM access
* Security Group-based database access
* SSM port forwarding
* TLS-encrypted PostgreSQL connection
* No VPN requirement
* No SSH keys required

---

# 26. Evidence

Screenshots were collected throughout the implementation.

Important evidence includes:

```text
32-hostname-mapping.png
33-real-hostname-postgresql-test.png
34-ssm-port-forwarding-active.png
35-ssm-tunnel-port-verification.png
36-tls-verification.png
```

Additional screenshots document the earlier AWS infrastructure setup.

---

# 27. Cleanup

After all documentation and evidence have been collected, the AWS resources can be destroyed.

Cleanup should include:

1. Delete RDS database
2. Delete RDS subnet group
3. Terminate EC2 Jump Host
4. Delete SSM VPC endpoints
5. Delete endpoint Security Group
6. Delete RDS Security Group
7. Delete Jump Host Security Group
8. Delete IAM role
9. Delete private subnets
10. Delete route table
11. Delete VPC

Before deletion, verify that no other AWS resources depend on the VPC.

---

# 28. Manual Project Completion

The manual implementation is considered complete after:

* Infrastructure is successfully tested
* PostgreSQL connection works
* Real RDS hostname works
* SSM port forwarding works
* HAProxy works
* TLS is verified
* Evidence is collected
* Documentation is committed to GitHub
* AWS infrastructure is safely destroyed

---

# 29. Future Terraform Version

This project was intentionally completed manually.

Terraform was **not used** during this implementation.

The next phase is to recreate the same architecture using Terraform.

The Terraform version will demonstrate:

* Infrastructure as Code
* Terraform providers
* Variables
* Outputs
* Resource dependencies
* Security Groups as code
* VPC configuration as code
* EC2 provisioning
* RDS provisioning
* VPC endpoints
* IAM configuration
* Repeatable infrastructure deployment

The manual project and Terraform project will therefore remain separate.

```text
Manual Version
aws-ssm-jump-host-postgres-poc-manual
        │
        │
        ▼
Understand and validate the architecture
        │
        ▼
Terraform Version
aws-ssm-jump-host-postgres-poc-terraform
        │
        ▼
Automate the same architecture
```

---

# 30. Final Status

```text
Manual AWS Implementation
        ✅ COMPLETED

Documentation
        ✅ COMPLETED

Evidence
        ✅ COLLECTED

Terraform
        🔜 FUTURE VERSION
```

**Project:** AWS SSM Jump Host + Private PostgreSQL RDS POC

**Implementation:** Manual

**AWS Region:** ap-south-1

**Status:** Completed
