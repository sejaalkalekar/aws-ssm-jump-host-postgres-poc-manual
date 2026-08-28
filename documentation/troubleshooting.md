# Troubleshooting — AWS SSM Jump Host + Private PostgreSQL RDS

## 1. Overview

This document records the main issues encountered while building the **AWS SSM Jump Host + Private PostgreSQL RDS POC — Manual Version**.

The project was implemented manually using:

* AWS Management Console
* AWS CLI
* Windows PowerShell
* AWS Systems Manager
* PostgreSQL
* HAProxy

**Terraform was not used in this version.**

The purpose of this document is to explain the problems, their causes, and the solutions used during implementation.

---

# 2. `psql` Command Not Found on Windows

## Problem

Initially, running:

```powershell
psql -h localhost -p 5432 -U postgres -d postgres
```

returned:

```text
psql : The term 'psql' is not recognized...
```

## Cause

The PostgreSQL `bin` directory was not available in the Windows PATH.

## Solution

The PostgreSQL executable was located using:

```powershell
Get-ChildItem "C:\Program Files\PostgreSQL" -Recurse -Filter psql.exe -ErrorAction SilentlyContinue | Select-Object -ExpandProperty FullName
```

The executable was found at:

```text
C:\Program Files\PostgreSQL\18\bin\psql.exe
```

The full executable path was then used:

```powershell
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -h localhost -p 5432 -U postgres -d postgres
```

## Result

PostgreSQL successfully connected to the local PostgreSQL server.

---

# 3. Verify Local PostgreSQL

After locating `psql.exe`, the PostgreSQL server was verified using:

```sql
SELECT version();
```

The server returned:

```text
PostgreSQL 18.6
```

Connection information was also verified using:

```sql
\conninfo
```

This confirmed that the local PostgreSQL installation was working.

---

# 4. AWS CLI AccessDenied Error

## Problem

When attempting to retrieve the RDS endpoint using:

```powershell
aws rds describe-db-instances --query "DBInstances[0].Endpoint.Address" --output text
```

AWS returned:

```text
AccessDenied
```

The IAM user did not have permission to perform:

```text
rds:DescribeDBInstances
```

## Cause

The IAM user did not have the required RDS read permission.

## Solution

The RDS endpoint was obtained from the AWS Console instead.

The endpoint used in the project was:

```text
ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com
```

## Lesson

AWS CLI commands require the appropriate IAM permissions.

When a CLI command returns `AccessDenied`, check the IAM policy attached to the user or role.

---

# 5. Direct RDS Connection Timed Out

## Problem

An attempt was made to connect directly to the private RDS hostname from Windows:

```powershell
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -h ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com -p 5432 -U postgres -d postgres
```

The result was:

```text
connection timed out
```

## Cause

The RDS database was intentionally deployed privately.

The local Windows machine could resolve the RDS hostname, but it did not have a direct network path into the private AWS subnet.

## Solution

The architecture was designed to use:

```text
Windows
   ↓
HAProxy
   ↓
SSM Port Forwarding
   ↓
Private EC2 Jump Host
   ↓
Private RDS
```

The database was therefore accessed through the intended private access path instead of directly.

---

# 6. Hostname Mapping Not Initially Present

## Problem

The Windows Hosts file initially did not contain the RDS hostname mapping.

The check:

```powershell
Get-Content C:\Windows\System32\drivers\etc\hosts | Select-String "ssm-jump-host-postgres"
```

returned no matching entry.

## Solution

The following entry was added to the Windows Hosts file:

```text
127.0.0.1    ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com
```

Hosts file location:

```text
C:\Windows\System32\drivers\etc\hosts
```

## Result

The real RDS hostname resolved to:

```text
127.0.0.1
```

This allowed the PostgreSQL client to continue using the real RDS hostname while traffic was redirected through HAProxy.

---

# 7. HAProxy Configuration File Location

## Problem

The initial assumed HAProxy configuration path:

```text
C:\ProgramData\haproxy\haproxy.cfg
```

did not exist.

## Solution

The system was searched for the actual configuration file:

```powershell
Get-ChildItem C:\ -Filter haproxy.cfg -Recurse -ErrorAction SilentlyContinue | Select-Object -ExpandProperty FullName
```

The actual configuration was found at:

```text
C:\haproxy\haproxy.cfg
```

## Lesson

Do not assume configuration file locations. Verify the actual installation path.

---

# 8. SSM Tunnel Port Not Available

## Problem

The following command was used:

```powershell
Test-NetConnection 127.0.0.1 -Port 15432
```

The result was:

```text
TcpTestSucceeded : False
```

## Cause

The SSM port-forwarding session was not running.

HAProxy was configured to forward traffic to:

```text
127.0.0.1:15432
```

but no active SSM session was listening on that port.

## Solution

The SSM port-forwarding session was started:

```powershell
aws ssm start-session --target i-0e24c09235ec9db01 --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters "host=ssm-jump-host-postgres.c1wcq6gmo9u9.ap-south-1.rds.amazonaws.com,portNumber=5432,localPortNumber=15432"
```

Successful output:

```text
Port 15432 opened for sessionId ...
Waiting for connections...
```

The port was then verified again:

```powershell
Test-NetConnection 127.0.0.1 -Port 15432
```

Result:

```text
TcpTestSucceeded : True
```

---

# 9. PostgreSQL SSL Test Failed Through HAProxy

## Problem

An initial TLS test using:

```text
sslmode=require
```

returned:

```text
server does not support SSL, but SSL was required
```

## Investigation

The HAProxy configuration was checked:

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

HAProxy was configured in TCP mode, which is appropriate for PostgreSQL TLS pass-through.

The issue was then investigated by testing the SSM tunnel directly.

---

# 10. PostgreSQL Authentication Failure

## Problem

A direct connection through the SSM tunnel initially returned:

```text
FATAL: password authentication failed for user "postgres"
```

## Cause

The PostgreSQL connection was reaching the RDS database, but the supplied password was incorrect.

## Solution

The correct RDS master password was used.

The connection then succeeded.

## Result

The successful connection showed:

```text
psql (18.6, server 18.3)

SSL connection (protocol: TLSv1.3,
cipher: TLS_AES_256_GCM_SHA384,
compression: off, ALPN: postgresql)
```

This confirmed both:

* PostgreSQL connectivity
* TLS encryption

---

# 11. `psql` Not Installed on Jump Host

## Problem

An attempt was made to run:

```bash
psql ...
```

inside the EC2 Jump Host.

The result was:

```text
sh: psql: command not found
```

## Cause

The Jump Host was designed as a forwarding server and did not have the PostgreSQL client installed.

## Solution

No PostgreSQL installation was required.

The PostgreSQL client remained on the local Windows machine.

The Jump Host's role was to provide the private network path through SSM.

## Lesson

A Jump Host does not necessarily need database client software when it is being used only for network forwarding.

---

# 12. Verifying the SSM Tunnel

The SSM session displayed:

```text
Port 15432 opened for sessionId ...
Waiting for connections...
```

The local listener was then verified:

```powershell
Test-NetConnection 127.0.0.1 -Port 15432
```

Successful result:

```text
TcpTestSucceeded : True
```

This confirmed that the local SSM forwarding endpoint was active.

---

# 13. Final TLS Verification

The final TLS test used:

```powershell
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" "host=127.0.0.1 port=15432 user=postgres dbname=postgres sslmode=require"
```

The successful result was:

```text
SSL connection (protocol: TLSv1.3,
cipher: TLS_AES_256_GCM_SHA384,
compression: off, ALPN: postgresql)
```

This was saved as the TLS verification evidence.

---

# 14. Troubleshooting Approach

The project followed a layered troubleshooting approach.

When a connection failed, each layer was checked separately:

```text
Layer 1
Local PostgreSQL Client
        ↓
Layer 2
HAProxy
        ↓
Layer 3
SSM Local Port
        ↓
Layer 4
SSM Session
        ↓
Layer 5
EC2 Jump Host
        ↓
Layer 6
RDS Security Group
        ↓
Layer 7
RDS PostgreSQL
        ↓
Layer 8
TLS
```

This made it easier to identify where the connection was failing.

---

# 15. Useful Verification Commands

## Find PostgreSQL Client

```powershell
Get-ChildItem "C:\Program Files\PostgreSQL" -Recurse -Filter psql.exe -ErrorAction SilentlyContinue | Select-Object -ExpandProperty FullName
```

## Test Local Port

```powershell
Test-NetConnection 127.0.0.1 -Port 15432
```

## Check Hosts Mapping

```powershell
Get-Content C:\Windows\System32\drivers\etc\hosts | Select-String "ssm-jump-host-postgres"
```

## Check HAProxy Configuration

```powershell
Get-Content C:\haproxy\haproxy.cfg
```

## PostgreSQL Version

```sql
SELECT version();
```

## PostgreSQL Connection Information

```sql
\conninfo
```

---

# 16. Lessons Learned

The troubleshooting process provided practical experience with:

* AWS IAM permissions
* Private VPC networking
* Security Groups
* AWS Systems Manager
* SSM port forwarding
* Private RDS connectivity
* PostgreSQL authentication
* HAProxy TCP forwarding
* Windows hostname mapping
* PostgreSQL TLS
* Layer-by-layer troubleshooting

The main lesson was to avoid troubleshooting the entire architecture at once.

Instead, verify each connection layer independently.

---

# 17. Final Result

After troubleshooting, the complete connection path worked successfully:

```text
Windows PostgreSQL Client
        ↓
Real RDS Hostname
        ↓
Windows Hosts File
        ↓
HAProxy :5432
        ↓
SSM Port Forward :15432
        ↓
Private EC2 Jump Host
        ↓
Private RDS PostgreSQL :5432
        ↓
TLS 1.3
```

The manual POC was successfully completed.

---

# 18. Project Status

```text
Manual Implementation
        ✅ COMPLETE

Troubleshooting
        ✅ DOCUMENTED

TLS Verification
        ✅ COMPLETE

Terraform
        🔜 FUTURE VERSION
```

The Terraform version will recreate the same architecture using Infrastructure as Code.
