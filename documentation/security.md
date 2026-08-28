# Security Documentation — AWS SSM Jump Host + Private PostgreSQL RDS

## 1. Overview

This document describes the security design used in the **AWS SSM Jump Host + Private PostgreSQL RDS POC — Manual Version**.

The project was implemented manually using the AWS Management Console, AWS CLI, Windows PowerShell, AWS Systems Manager, HAProxy, and PostgreSQL.

**Terraform was not used in this version.**

The main security objective was to provide controlled access to a private PostgreSQL RDS database without exposing the database directly to the public internet.

---

# 2. Security Architecture

The project follows a private-access architecture:

```text
                     LOCAL WINDOWS MACHINE
                              │
                              │
                              ▼
                       HAProxy :5432
                              │
                              ▼
                     SSM Port Forwarding
                              │
                              ▼
                  ┌──────────────────────┐
                  │   Private EC2        │
                  │   Jump Host          │
                  │   No Public IP       │
                  └──────────┬───────────┘
                             │
                             │ TCP 5432
                             ▼
                  ┌──────────────────────┐
                  │   Private RDS        │
                  │   PostgreSQL         │
                  │   No Public Access   │
                  └──────────────────────┘
```

---

# 3. Private RDS

The PostgreSQL RDS instance was configured as a private database.

### Security characteristics

* Public access disabled
* Database deployed in private subnets
* PostgreSQL port: `5432`
* Access controlled using a Security Group
* No direct internet access

The RDS endpoint was only reachable from the private AWS networking path.

---

# 4. Private EC2 Jump Host

The EC2 Jump Host was deployed inside a private subnet.

Configuration included:

```text
Instance Type: t3.micro
Private IP: 10.0.1.112
Public IP: None
```

The instance did not require a public IP address.

This reduces the attack surface because the Jump Host is not directly exposed to the internet.

---

# 5. AWS Systems Manager

AWS Systems Manager Session Manager was used instead of traditional SSH access.

### Traditional approach

```text
Internet
   ↓
Public EC2
   ↓
SSH :22
   ↓
Private Resource
```

### Project approach

```text
Local Machine
      ↓
AWS Systems Manager
      ↓
Private EC2
      ↓
Private RDS
```

No inbound SSH port `22` was required.

---

# 6. IAM Security

The EC2 instance used an IAM role:

```text
ssm-jump-host-role
```

The role included:

```text
AmazonSSMManagedInstanceCore
```

This provides the permissions required for Systems Manager functionality.

Using an IAM role avoids storing long-term AWS credentials on the EC2 instance.

---

# 7. VPC Endpoints

The private VPC used Systems Manager VPC endpoints:

```text
SSM
SSMMessages
EC2Messages
```

These endpoints allow the private EC2 instance to communicate with Systems Manager without requiring a public IP or internet gateway for the SSM communication path.

---

# 8. Security Groups

Security Groups were used to restrict network access.

## Jump Host Security Group

The Jump Host Security Group did not require inbound SSH access.

This supports the Session Manager-based access model.

## RDS Security Group

The RDS Security Group allowed PostgreSQL traffic:

```text
Protocol: TCP
Port: 5432
Source: Jump Host Security Group
```

This is preferable to allowing:

```text
0.0.0.0/0
```

because database access is restricted to the intended application path.

---

# 9. Network Isolation

The project uses private subnets for the important infrastructure.

```text
VPC
10.0.0.0/16
│
├── Private Subnet 1
│   10.0.1.0/24
│   │
│   └── EC2 Jump Host
│
└── Private Subnet 2
    10.0.2.0/24
    │
    └── RDS PostgreSQL
```

The RDS database is therefore isolated from direct public access.

---

# 10. SSM Port Forwarding

SSM port forwarding was used to create a controlled connection from the local machine to the private database.

Example:

```text
Local Machine
127.0.0.1:15432
       │
       ▼
SSM Port Forwarding
       │
       ▼
Private Jump Host
       │
       ▼
RDS:5432
```

The RDS database itself remains private.

---

# 11. HAProxy Security Role

HAProxy was configured in TCP mode.

```text
Local :5432
     ↓
HAProxy
     ↓
Local :15432
     ↓
SSM Tunnel
     ↓
RDS :5432
```

HAProxy does not terminate PostgreSQL TLS in this configuration.

It forwards TCP traffic through the SSM tunnel.

---

# 12. Hostname Mapping

The Windows Hosts file maps the real RDS hostname to localhost:

```text
127.0.0.1    ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com
```

This allows the PostgreSQL client to use the original database hostname while the traffic is routed through the local forwarding layer.

---

# 13. TLS Encryption

TLS was successfully verified during the final PostgreSQL connection test.

The connection reported:

```text
SSL connection
protocol: TLSv1.3
cipher: TLS_AES_256_GCM_SHA384
compression: off
```

This confirms that the PostgreSQL connection was encrypted using TLS 1.3.

---

# 14. Credential Security

Sensitive credentials must never be committed to GitHub.

The following information must remain private:

```text
AWS Access Keys
AWS Secret Access Keys
AWS Session Tokens
RDS Password
IAM Credentials
Database Credentials
Private Keys
```

The repository contains configuration and documentation only.

Passwords and secret credentials are intentionally excluded.

---

# 15. Principle of Least Privilege

The project follows the principle of least privilege where possible.

Examples include:

* EC2 uses an IAM role instead of stored AWS credentials
* RDS access is restricted through Security Groups
* RDS is not publicly accessible
* SSH access is not required
* SSM provides controlled instance access
* Database access is restricted to the Jump Host path

---

# 16. Security Benefits

The architecture provides several security benefits:

### No Public Database

The PostgreSQL RDS instance is private.

### No SSH Exposure

Port `22` does not need to be opened for remote administration.

### No Public Jump Host

The EC2 Jump Host does not have a public IP.

### IAM-Based Instance Access

Systems Manager provides access using AWS IAM permissions.

### Restricted Database Access

RDS accepts PostgreSQL traffic from the intended Jump Host Security Group.

### Encrypted Database Connection

TLS 1.3 was successfully verified for the PostgreSQL connection.

---

# 17. Security Limitations

This project is a Proof of Concept and should be reviewed before being used in a production environment.

Possible production improvements include:

* AWS CloudTrail monitoring
* VPC Flow Logs
* CloudWatch monitoring and alerts
* More restrictive IAM policies
* Centralized logging
* AWS Secrets Manager for database credentials
* Automated credential rotation
* Additional network controls
* Formal certificate management
* Multi-account architecture
* Stronger operational controls

---

# 18. Manual Implementation

All security controls in this version were configured manually.

The project did not use:

* Terraform
* CloudFormation
* AWS CDK
* Other Infrastructure as Code tools

The future Terraform version will recreate these security controls as code.

---

# 19. Security Validation

The following security checks were successfully demonstrated:

| Security Check        | Result           |
| --------------------- | ---------------- |
| RDS Public Access     | Disabled         |
| EC2 Public IP         | None             |
| SSH Requirement       | Not required     |
| SSM Access            | Working          |
| SSM Port Forwarding   | Working          |
| RDS Security Group    | Restricted       |
| Real RDS Hostname     | Working          |
| PostgreSQL Connection | Working          |
| TLS                   | TLS 1.3 verified |

---

# 20. Conclusion

The manual implementation successfully demonstrated a secure method for accessing a private PostgreSQL RDS database from a local machine.

The architecture avoids direct public exposure of the database and uses:

```text
IAM
+
AWS Systems Manager
+
Private EC2
+
Security Groups
+
SSM Port Forwarding
+
HAProxy
+
TLS
```

to provide controlled and encrypted database access.

**Manual Version:** Completed

**Terraform Version:** Planned as a separate implementation using the same architecture.
