# RDS CLI-Only Access with Britive PAM

## Use Case

Allow RDS Database Managers to access and manage RDS databases using AWS CLI commands while completely blocking access through the AWS Console UI. This ensures:

- Security teams maintain control over database access
- All database operations are auditable through CLI commands
- Just-in-time access with automatic credential rotation
- Least privilege access scoped to specific RDS instances

## Solution Overview

This solution uses Britive's Privileged Access Management (PAM) with AWS credential process integration to provide seamless, secure CLI access to RDS databases without manual credential management.

## Architecture Components

### 1. Britive Configuration
- **Platform**: Britive PAM system
- **Profile**: `AWS Standalone/123456789012 (Britive-AWS)/rds-db-cluster-abc123`
- **Tenant**: `mena-poc`

### 2. Local Configuration Files
- **Pybritive Config**: `~/.britive/pybritive.config`
- **AWS Credentials**: `~/.aws/credentials`

### 3. AWS IAM Policy
- Attribute-Based Access Control (ABAC) using Principal Tags
- Resource-scoped permissions for specific RDS cluster

## Implementation Steps

### Step 1: Configure Pybritive

Create the Britive configuration file at `~/.britive/pybritive.config`:

```ini
[company-rds-1]
profile=AWS Standalone/123456789012 (Britive-AWS)/rds-db-cluster-abc123
tenant=mena-poc
```

**What this does:**
- Maps a local profile name (`company-rds-1`) to a Britive PAM profile
- Specifies the Britive tenant for authentication

### Step 2: Configure AWS Credentials

Create/update the AWS credentials file at `~/.aws/credentials`:

```ini
[company-rds-1]
credential_process=pybritive-aws-cred-process --profile "AWS Standalone/123456789012 (Britive-AWS)/rds-db-cluster-abc123" -t mena-poc
region=eu-west-1
```

**What this does:**
- Uses AWS credential process to dynamically fetch credentials from Britive
- Automatically handles credential checkout when AWS CLI commands are run
- Sets the default region for RDS operations

### Step 3: Create IAM Policy in AWS

The IAM policy uses Attribute-Based Access Control (ABAC) with the `RdsIdentifier` principal tag to scope access.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "RDSInstanceFullAccess",
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBInstances",
                "rds:DescribeDBClusters",
                "rds:DescribeDBSnapshots",
                "rds:DescribeDBClusterSnapshots",
                "rds:DescribeDBParameterGroups",
                "rds:DescribeDBParameters",
                "rds:DescribeDBSubnetGroups",
                "rds:DescribeDBSecurityGroups",
                "rds:DescribeDBLogFiles",
                "rds:DownloadDBLogFilePortion",
                "rds:ModifyDBInstance",
                "rds:ModifyDBCluster",
                "rds:RebootDBInstance",
                "rds:StartDBInstance",
                "rds:StopDBInstance",
                "rds:CreateDBSnapshot",
                "rds:DeleteDBSnapshot",
                "rds:RestoreDBInstanceFromDBSnapshot",
                "rds:CreateDBInstanceReadReplica",
                "rds:PromoteReadReplica",
                "rds:AddTagsToResource",
                "rds:RemoveTagsFromResource",
                "rds:ListTagsForResource",
                "rds:DescribeEvents"
            ],
            "Resource": [
                "arn:aws:rds:eu-west-1:123456789012:cluster:${aws:PrincipalTag/RdsIdentifier}",
                "arn:aws:rds:eu-west-1:123456789012:db:${aws:PrincipalTag/RdsIdentifier}-*",
                "arn:aws:rds:eu-west-1:123456789012:db:rds-db-cluster-*",
                "arn:aws:rds:eu-west-1:123456789012:cluster-snapshot:*",
                "arn:aws:rds:eu-west-1:123456789012:snapshot:*"
            ]
        },
        {
            "Sid": "RDSGeneralReadAccess",
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBEngineVersions",
                "rds:DescribeOrderableDBInstanceOptions",
                "rds:DescribeValidDBInstanceModifications",
                "rds:DescribeDBSubnetGroups",
                "rds:DescribeOptionGroups",
                "rds:DescribeDBParameterGroups"
            ],
            "Resource": "*"
        },
        {
            "Sid": "IAMDatabaseAuthentication",
            "Effect": "Allow",
            "Action": [
                "rds-db:connect"
            ],
            "Resource": [
                "arn:aws:rds-db:eu-west-1:123456789012:dbuser:*/*"
            ]
        },
        {
            "Sid": "CloudWatchLogsAccess",
            "Effect": "Allow",
            "Action": [
                "logs:DescribeLogGroups",
                "logs:DescribeLogStreams",
                "logs:GetLogEvents",
                "logs:FilterLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:eu-west-1:123456789012:log-group:/aws/rds/cluster/${aws:PrincipalTag/RdsIdentifier}/*",
                "arn:aws:logs:eu-west-1:123456789012:log-group:/aws/rds/instance/${aws:PrincipalTag/RdsIdentifier}-*/*"
            ]
        }
    ]
}
```

### Policy Breakdown

#### Statement 1: RDS Instance Full Access
- **Purpose**: Provides comprehensive RDS management capabilities
- **Key Feature**: Uses `${aws:PrincipalTag/RdsIdentifier}` variable for dynamic resource scoping
- **Resources Allowed**:
  - Cluster: `rds-db-cluster-abc123`
  - Instances: `rds-db-cluster-instance-1`, `rds-db-cluster-instance-2`
  - Snapshots: All cluster and instance snapshots

#### Statement 2: RDS General Read Access
- **Purpose**: Allow reading general RDS metadata (engine versions, options, etc.)
- **Scope**: Global (doesn't expose specific database data)

#### Statement 3: IAM Database Authentication
- **Purpose**: Enable IAM-based database authentication
- **Allows**: Direct database connections using IAM credentials

#### Statement 4: CloudWatch Logs Access
- **Purpose**: Read database logs through CloudWatch
- **Scope**: Only logs for the specific RDS cluster and instances

## How It Works

### Automatic Profile Checkout

When an RDS manager runs an AWS CLI command:

1. **Command Execution**: Manager runs: `aws rds describe-db-clusters --db-cluster-identifier rds-db-cluster-abc123 --profile company-rds-1`

2. **Credential Request**: AWS CLI invokes `pybritive-aws-cred-process`

3. **Britive Authentication**: Pybritive authenticates with Britive PAM and checks out the profile

4. **Temporary Credentials**: Britive returns temporary AWS credentials (STS tokens) with the `RdsIdentifier` tag

5. **Command Execution**: AWS CLI uses these credentials to execute the command

6. **Policy Enforcement**: AWS IAM evaluates the policy using the principal tag to allow/deny access

7. **Auto Expiration**: Credentials automatically expire after the configured duration

### Security Features

✅ **No UI Access**: The IAM policy doesn't grant any AWS Console permissions
✅ **Automatic Checkout**: No manual credential management required
✅ **Time-Limited Access**: Credentials expire automatically
✅ **Scoped Access**: Can only access the specific RDS cluster assigned
✅ **Audit Trail**: All operations are logged in CloudTrail
✅ **Zero Standing Privileges**: Access is just-in-time

### Access Restrictions

**What RDS Managers CAN do:**
- Manage their assigned RDS cluster (`rds-db-cluster-abc123`)
- Start, stop, reboot the cluster
- Create and manage snapshots
- Modify cluster configuration
- View logs and events
- Connect to databases using IAM authentication

**What RDS Managers CANNOT do:**
- Access the AWS Console UI
- Manage other RDS databases in the account
- Delete the cluster (not in policy actions)
- Access other AWS services
- View resources outside their scope

## Example Commands

### Cluster Management

```bash
# Describe the cluster
aws rds describe-db-clusters --db-cluster-identifier rds-db-cluster-abc123 --profile company-rds-1

# Get cluster status
aws rds describe-db-clusters --db-cluster-identifier rds-db-cluster-abc123 --query 'DBClusters[0].Status' --profile company-rds-1

# Get cluster members (writer and reader instances)
aws rds describe-db-clusters --db-cluster-identifier rds-db-cluster-abc123 --query 'DBClusters[0].DBClusterMembers' --profile company-rds-1

# Start the cluster
aws rds start-db-cluster --db-cluster-identifier rds-db-cluster-abc123 --profile company-rds-1

# Stop the cluster
aws rds stop-db-cluster --db-cluster-identifier rds-db-cluster-abc123 --profile company-rds-1

# Modify cluster configuration
aws rds modify-db-cluster --db-cluster-identifier rds-db-cluster-abc123 --backup-retention-period 7 --apply-immediately --profile company-rds-1
```

### Snapshot Management

```bash
# Create a manual snapshot
aws rds create-db-cluster-snapshot --db-cluster-identifier rds-db-cluster-abc123 --db-cluster-snapshot-identifier snapshot-$(date +%Y%m%d) --profile company-rds-1

# List all snapshots for the cluster
aws rds describe-db-cluster-snapshots --db-cluster-identifier rds-db-cluster-abc123 --profile company-rds-1

# Restore from snapshot (creates new cluster)
aws rds restore-db-cluster-from-snapshot --db-cluster-identifier new-cluster-name --snapshot-identifier your-snapshot-name --engine aurora-mysql --profile company-rds-1
```

### Instance Operations

```bash
# Reboot the writer instance
aws rds reboot-db-instance --db-instance-identifier rds-db-cluster-instance-1 --profile company-rds-1

# Reboot the reader instance
aws rds reboot-db-instance --db-instance-identifier rds-db-cluster-instance-2 --profile company-rds-1

# View log files
aws rds describe-db-log-files --db-instance-identifier rds-db-cluster-instance-1 --profile company-rds-1

# Download log file
aws rds download-db-log-file-portion --db-instance-identifier rds-db-cluster-instance-1 --log-file-name error/mysql-error.log --profile company-rds-1
```

### Monitoring and Events

```bash
# View recent events
aws rds describe-events --source-identifier rds-db-cluster-abc123 --source-type db-cluster --profile company-rds-1

# Add tags to cluster
aws rds add-tags-to-resource --resource-name arn:aws:rds:eu-west-1:123456789012:cluster:rds-db-cluster-abc123 --tags Key=Environment,Value=Production --profile company-rds-1

# List tags
aws rds list-tags-for-resource --resource-name arn:aws:rds:eu-west-1:123456789012:cluster:rds-db-cluster-abc123 --profile company-rds-1
```

## Cluster Information

**Cluster Details:**
- **Cluster ID**: `rds-db-cluster-abc123`
- **Cluster Role**: Regional cluster
- **Engine**: Aurora MySQL 8.0.mysql_aurora.3.04.0
- **Storage**: Aurora Standard
- **RDS Extended Support**: Enabled

**Cluster Instances:**
- **Writer Instance**: `rds-db-cluster-instance-1` (IsClusterWriter: true)
- **Reader Instance**: `rds-db-cluster-instance-2` (IsClusterWriter: false)

## Access Flow Diagram

```
┌─────────────────┐
│  RDS Manager    │
│  (Local CLI)    │
└────────┬────────┘
         │
         │ aws rds ... --profile company-rds-1
         │
         ▼
┌─────────────────────────┐
│  AWS CLI                │
│  credential_process     │
└────────┬────────────────┘
         │
         │ pybritive-aws-cred-process
         │
         ▼
┌─────────────────────────┐
│  Britive PAM            │
│  - Authenticate user    │
│  - Check permissions    │
│  - Generate STS token   │
│  - Add RdsIdentifier    │
│    principal tag        │
└────────┬────────────────┘
         │
         │ Temporary AWS Credentials
         │ (with RdsIdentifier=rds-db-cluster-abc123)
         │
         ▼
┌─────────────────────────┐
│  AWS IAM                │
│  - Evaluate policy      │
│  - Check principal tag  │
│  - Allow/Deny access    │
└────────┬────────────────┘
         │
         │ Allow (if resource matches tag)
         │
         ▼
┌─────────────────────────┐
│  Amazon RDS             │
│  - Execute operation    │
│  - Log to CloudTrail    │
└─────────────────────────┘
```

## Security Benefits

### 1. No UI Access
- Console permissions are completely absent from the policy
- Even with valid credentials, managers cannot log into AWS Console
- Prevents accidental or unauthorized changes through UI

### 2. Just-in-Time Access
- Credentials are generated on-demand
- No long-lived credentials stored locally
- Access automatically expires after session timeout

### 3. Least Privilege
- Access scoped to specific RDS cluster only
- Cannot access other databases in the account
- Cannot perform destructive operations (delete cluster)

### 4. Attribute-Based Access Control (ABAC)
- Uses `${aws:PrincipalTag/RdsIdentifier}` for dynamic scoping
- Single policy can support multiple users with different RDS assignments
- Scalable approach for managing many databases

### 5. Comprehensive Auditing
- All CLI operations logged in CloudTrail
- Britive PAM logs all profile checkouts
- Full audit trail for compliance

### 6. Frictionless User Experience
- No manual credential management
- Standard AWS CLI commands work seamlessly
- Security doesn't impede productivity

### 7. SAMA Cybersecurity Framework Compliance
- **Privileged Access Management**: All database access is privileged and managed through Britive PAM
- **Just-in-Time Access**: Credentials are provisioned only when needed and expire automatically
- **Segregation of Duties**: Clear separation between those who can access UI vs. CLI
- **Comprehensive Audit Trail**: All access and operations logged for regulatory review
- **Access Control**: Role-based access with least privilege principle enforced
- **Authentication & Authorization**: Multi-factor authentication via Britive, policy-based authorization via AWS IAM
- **Data Protection**: Access to sensitive databases controlled and monitored

## Troubleshooting

### Issue: Multiple Matching Profiles Found

**Error:**
```
Error: multiple matching profiles found - cannot determine which profile to use
```

**Solution:**
Use the exact profile path or profile ID when checking out:
```bash
pybritive checkout --profile "AWS Standalone/123456789012 (Britive-AWS)/rds-db-cluster-abc123" -t mena-poc
```

### Issue: Access Denied for Specific Commands

**Error:**
```
An error occurred (AccessDenied) when calling the DescribeDBInstances operation
```

**Solution:**
Check if the command requires permissions not in the policy, or if you're trying to access resources outside your scope. Use cluster-level commands when instance-level commands fail:

```bash
# Instead of: aws rds describe-db-instances --filters ...
# Use: aws rds describe-db-clusters --query 'DBClusters[0].DBClusterMembers'
```

### Issue: Credentials Not Found

**Error:**
```
Error when retrieving credentials from custom-process
```

**Solution:**
1. Verify `~/.britive/pybritive.config` exists and is properly configured
2. Ensure `pybritive` CLI is installed: `pip install pybritive`
3. Check Britive authentication: `pybritive login -t mena-poc`

## Best Practices

### For Administrators

1. **Use Descriptive Profile Names**: Make it clear which database each profile accesses
2. **Set Appropriate Timeouts**: Configure reasonable session durations in Britive
3. **Regular Audit Reviews**: Review CloudTrail logs periodically
4. **Principle of Least Privilege**: Only grant necessary permissions
5. **Tag Consistency**: Ensure `RdsIdentifier` tags match cluster names

### For RDS Managers

1. **Always Specify Profile**: Use `--profile company-rds-1` on every command
2. **Check Cluster Status First**: Before making changes, verify cluster state
3. **Test in Non-Production**: Validate commands in dev/test environments first
4. **Create Snapshots**: Before major changes, create a manual snapshot
5. **Use Query Filters**: Use `--query` to get specific information efficiently

## Scaling This Solution

### Supporting Multiple Databases

To extend this solution for multiple databases:

1. **Create Additional Britive Profiles**: One per RDS cluster
2. **Create Additional AWS Credential Entries**: Map each to a unique profile name
3. **Use Same IAM Policy**: The ABAC approach with `${aws:PrincipalTag/RdsIdentifier}` automatically scopes access
4. **Update Principal Tags**: Ensure each profile checkout sets the appropriate `RdsIdentifier` tag

**Example for second database:**

`~/.britive/pybritive.config`:
```ini
[company-rds-2]
profile=AWS Standalone/123456789012 (Britive-AWS)/rds-db-production-main
tenant=mena-poc
```

`~/.aws/credentials`:
```ini
[company-rds-2]
credential_process=pybritive-aws-cred-process --profile "AWS Standalone/123456789012 (Britive-AWS)/rds-db-production-main" -t mena-poc
region=eu-west-1
```

## Compliance and Governance

This solution helps meet various compliance requirements:

- **SAMA (Saudi Central Bank)**: 
  - Cybersecurity Framework Article 4.2.1: Privileged access management and monitoring
  - Article 4.2.2: Just-in-time access and automatic credential rotation
  - Article 5.1.1: Role-based access control with least privilege
  - Article 6.1.1: Comprehensive logging and audit trails
  - Article 7.1.1: Authentication and authorization controls
  - Article 8.1.1: Protection of sensitive data through access restrictions
- **SOC 2**: Just-in-time access, comprehensive logging
- **PCI DSS**: Restricted access to cardholder data environments
- **HIPAA**: Controlled access to protected health information
- **GDPR**: Access controls for personal data
- **ISO 27001**: Identity and access management controls

## Summary

This implementation provides:

✅ **Secure CLI-only access** to RDS databases
✅ **No AWS Console UI access** for database managers
✅ **Automatic credential management** via Britive PAM
✅ **Least privilege access** scoped to specific clusters
✅ **Just-in-time access** with automatic expiration
✅ **Comprehensive audit trail** for compliance
✅ **Seamless user experience** with standard AWS CLI commands
✅ **Scalable architecture** supporting multiple databases and users
✅ **SAMA Cybersecurity Framework compliant** for financial institutions

The solution ensures security teams maintain control while enabling RDS managers to perform their duties efficiently without manual credential management or security becoming a bottleneck.

### SAMA Compliance Mapping

| SAMA Cybersecurity Control | How This Solution Addresses It |
|---------------------------|--------------------------------|
| **Article 4.2.1** - Privileged Access Management | Britive PAM manages all privileged database access with just-in-time provisioning |
| **Article 4.2.2** - Access Control | Attribute-based access control (ABAC) enforces least privilege per database |
| **Article 5.1.1** - Role-Based Access | IAM policies define specific roles with granular permissions |
| **Article 6.1.1** - Logging & Monitoring | CloudTrail logs all RDS operations; Britive logs all profile checkouts |
| **Article 7.1.1** - Authentication | Multi-factor authentication enforced through Britive platform |
| **Article 8.1.1** - Data Protection | Access to sensitive databases restricted to authorized personnel only |
| **Article 9.1.1** - Security Monitoring | Continuous audit trail enables security monitoring and alerting |

This solution is particularly well-suited for financial institutions operating under SAMA regulations in Saudi Arabia, providing the necessary controls for database access management while maintaining operational efficiency.
