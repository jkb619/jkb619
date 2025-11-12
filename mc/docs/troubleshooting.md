# Troubleshooting Guide

This guide provides comprehensive troubleshooting steps for the Magic Castle Guest Invitation System.

## 🚨 Common Issues

### 1. ECS Service Not Starting

#### Symptoms
- ECS service shows "PENDING" or "STOPPED" status
- Tasks fail to start or keep restarting
- CloudWatch logs show errors

#### Diagnosis Steps

1. **Check ECS service status**:
   ```bash
   aws ecs describe-services --cluster magic-castle-cluster --services magic-castle-service
   ```

2. **Check task status**:
   ```bash
   aws ecs list-tasks --cluster magic-castle-cluster --service-name magic-castle-service
   ```

3. **Check CloudWatch logs**:
   ```bash
   aws logs tail /aws/ecs/magic-castle-service/mc-app --follow
   ```

4. **Check task definition**:
   ```bash
   aws ecs describe-task-definition --task-definition magic-castle-service
   ```

#### Common Causes & Solutions

**Cause**: Invalid task definition
```bash
# Check task definition syntax
aws ecs describe-task-definition --task-definition magic-castle-service
```

**Cause**: Insufficient resources
```bash
# Check CPU/memory allocation
aws ecs describe-task-definition --task-definition magic-castle-service \
  --query 'taskDefinition.{cpu:cpu,memory:memory}'
```

**Cause**: Security group issues
```bash
# Check security group rules
aws ec2 describe-security-groups --group-ids sg-05581195d90383c95
```

**Cause**: Secrets Manager access issues
```bash
# Check VPC endpoint
aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.us-west-1.secretsmanager"

# Check secrets access
aws secretsmanager get-secret-value --secret-id magic-castle-secrets
```

### 2. Database Connection Issues

#### Symptoms
- Application logs show database connection errors
- Health checks fail
- API endpoints return 500 errors

#### Diagnosis Steps

1. **Check RDS instance status**:
   ```bash
   aws rds describe-db-instances --db-instance-identifier magic-castle-database
   ```

2. **Check database connectivity from ECS**:
   ```bash
   # Connect to ECS task
   aws ecs execute-command \
     --cluster magic-castle-cluster \
     --task <task-id> \
     --container mc-app \
     --interactive \
     --command "/bin/bash"
   
   # Test database connection
   python -c "
   import pymysql
   import os
   conn = pymysql.connect(
       host=os.getenv('DB_HOST'),
       user=os.getenv('DB_USER'),
       password=os.getenv('DB_PASSWORD'),
       database=os.getenv('DB_NAME'),
       port=int(os.getenv('DB_PORT', 3306))
   )
   print('Database connection successful')
   conn.close()
   "
   ```

3. **Check security group rules**:
   ```bash
   # Get RDS security group ID
   aws rds describe-db-instances --db-instance-identifier magic-castle-database \
     --query 'DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId' --output text
   
   # Check security group rules
   aws ec2 describe-security-groups --group-ids <rds-security-group-id>
   ```

#### Common Causes & Solutions

**Cause**: Security group blocking connections
```bash
# Add ingress rule for MySQL port
aws ec2 authorize-security-group-ingress \
  --group-id <rds-security-group-id> \
  --protocol tcp \
  --port 3306 \
  --source-group sg-05581195d90383c95
```

**Cause**: Database credentials incorrect
```bash
# Check secrets in Secrets Manager
aws secretsmanager get-secret-value --secret-id magic-castle-secrets
```

**Cause**: Database instance not available
```bash
# Check RDS instance status
aws rds describe-db-instances --db-instance-identifier magic-castle-database \
  --query 'DBInstances[0].DBInstanceStatus'
```

### 3. ALB Health Check Failures

#### Symptoms
- ALB target health shows "unhealthy"
- 502/503 errors from ALB
- Application not receiving traffic

#### Diagnosis Steps

1. **Check target health**:
   ```bash
   aws elbv2 describe-target-health \
     --target-group-arn $(aws elbv2 describe-target-groups \
       --names magic-castle-tg \
       --query 'TargetGroups[0].TargetGroupArn' \
       --output text)
   ```

2. **Check ALB configuration**:
   ```bash
   aws elbv2 describe-load-balancers --names magic-castle-alb
   aws elbv2 describe-target-groups --names magic-castle-tg
   ```

3. **Test health endpoint directly**:
   ```bash
   # Get ECS task IP
   aws ecs describe-tasks --cluster magic-castle-cluster \
     --tasks $(aws ecs list-tasks --cluster magic-castle-cluster --service-name magic-castle-service \
       --query 'taskArns[0]' --output text) \
     --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' --output text
   
   # Test health endpoint
   curl http://<task-ip>:5000/health
   ```

#### Common Causes & Solutions

**Cause**: Security group blocking ALB traffic
```bash
# Check ALB security group
aws ec2 describe-security-groups --group-ids sg-05bf9ec12def62372

# Check ECS security group
aws ec2 describe-security-groups --group-ids sg-05581195d90383c95
```

**Cause**: Application not responding on port 5000
```bash
# Check application logs
aws logs tail /aws/ecs/magic-castle-service/mc-app --follow

# Check if application is listening on port 5000
aws ecs execute-command \
  --cluster magic-castle-cluster \
  --task <task-id> \
  --container mc-app \
  --interactive \
  --command "netstat -tlnp | grep 5000"
```

**Cause**: Health check path incorrect
```bash
# Check target group health check configuration
aws elbv2 describe-target-groups --names magic-castle-tg \
  --query 'TargetGroups[0].HealthCheckPath'
```

### 4. Secrets Manager Access Issues

#### Symptoms
- Application logs show "AccessDenied" errors
- Secrets not being retrieved
- Application fails to start

#### Diagnosis Steps

1. **Check VPC endpoint status**:
   ```bash
   aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.us-west-1.secretsmanager"
   ```

2. **Check VPC endpoint security group**:
   ```bash
   aws ec2 describe-security-groups --group-ids sg-0f520e5a6cae19d7b
   ```

3. **Test secrets access from ECS**:
   ```bash
   # Connect to ECS task
   aws ecs execute-command \
     --cluster magic-castle-cluster \
     --task <task-id> \
     --container mc-app \
     --interactive \
     --command "/bin/bash"
   
   # Test secrets access
   python -c "
   import boto3
   import os
   client = boto3.client('secretsmanager')
   response = client.get_secret_value(SecretId=os.getenv('SECRETS_MANAGER_ARN'))
   print('Secrets access successful')
   "
   ```

#### Common Causes & Solutions

**Cause**: VPC endpoint security group blocking traffic
```bash
# Add ingress rule for HTTPS
aws ec2 authorize-security-group-ingress \
  --group-id sg-0f520e5a6cae19d7b \
  --protocol tcp \
  --port 443 \
  --source-group sg-05581195d90383c95
```

**Cause**: IAM permissions insufficient
```bash
# Check ECS task role permissions
aws iam get-role --role-name magic-castle-ecs-task-role
aws iam list-attached-role-policies --role-name magic-castle-ecs-task-role
```

**Cause**: VPC endpoint not available
```bash
# Check VPC endpoint status
aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.us-west-1.secretsmanager" \
  --query 'VpcEndpoints[0].State'
```

### 5. Email Sending Issues

#### Symptoms
- Invitation creation succeeds but no email sent
- SES errors in application logs
- Email delivery failures

#### Diagnosis Steps

1. **Check SES configuration**:
   ```bash
   # Check SES sending quota
   aws ses get-send-quota
   
   # Check SES sending statistics
   aws ses get-send-statistics
   ```

2. **Check SES domain verification**:
   ```bash
   aws ses get-identity-verification-attributes --identities magiccastle-cloud.com --region us-west-1
   ```

3. **Test SES API from ECS**:
   ```bash
   # Connect to ECS task
   aws ecs execute-command \
     --cluster magic-castle-cluster \
     --task <task-id> \
     --container mc-app \
     --interactive \
     --command "/bin/bash"
   
   # Test SES
   python -c "
   import boto3
   import os
   client = boto3.client('ses')
   response = client.get_send_quota()
   print('SES access successful:', response)
   "
   ```

#### Common Causes & Solutions

**Cause**: SES sandbox mode
```bash
# Check if SES is in sandbox mode
aws ses get-send-quota
# If Max24HourSend is 200, SES is in sandbox mode
```

**Cause**: Invalid SES credentials or permissions
```bash
# Check ECS task role has SES permissions
aws iam get-role --role-name magic-castle-ecs-task-role --query 'Role.AssumeRolePolicyDocument'
aws iam list-attached-role-policies --role-name magic-castle-ecs-task-role
```

**Cause**: Email address not verified
```bash
# Check verified email addresses
aws ses list-verified-email-addresses
```

## 🔧 Debugging Commands

### General Debugging

```bash
# Check all ECS resources
aws ecs describe-clusters --clusters magic-castle-cluster
aws ecs describe-services --cluster magic-castle-cluster --services magic-castle-service
aws ecs list-tasks --cluster magic-castle-cluster --service-name magic-castle-service

# Check all RDS resources
aws rds describe-db-instances --db-instance-identifier magic-castle-database

# Check all ALB resources
aws elbv2 describe-load-balancers --names magic-castle-alb
aws elbv2 describe-target-groups --names magic-castle-tg

# Check all security groups
aws ec2 describe-security-groups --group-ids sg-05bf9ec12def62372 sg-05581195d90383c95 sg-0f520e5a6cae19d7b
```

### Network Debugging

```bash
# Check VPC configuration
aws ec2 describe-vpcs --vpc-ids vpc-052dd1972e02d17a0

# Check subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-052dd1972e02d17a0"

# Check route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-052dd1972e02d17a0"

# Check VPC endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=vpc-052dd1972e02d17a0"
```

### Application Debugging

```bash
# Connect to running container
aws ecs execute-command \
  --cluster magic-castle-cluster \
  --task <task-id> \
  --container mc-app \
  --interactive \
  --command "/bin/bash"

# Check environment variables
env | grep -E "(DB_|SECRETS_|SES_)"

# Check application logs
tail -f /var/log/application.log

# Check network connectivity
curl -v http://localhost:5000/health
```

### Database Debugging

```bash
# Connect to database
aws rds describe-db-instances --db-instance-identifier magic-castle-database \
  --query 'DBInstances[0].Endpoint.Address' --output text

# Test database connection
mysql -h <db-endpoint> -u <username> -p <database-name>

# Check database status
aws rds describe-db-instances --db-instance-identifier magic-castle-database \
  --query 'DBInstances[0].DBInstanceStatus'
```

## 📊 Monitoring & Alerting

### CloudWatch Metrics

```bash
# Check ECS metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=magic-castle-service Name=ClusterName,Value=magic-castle-cluster \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average

# Check ALB metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/magic-castle-alb/0fc7de4c212720ea \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

### Log Analysis

```bash
# Search for errors in logs
aws logs filter-log-events \
  --log-group-name /aws/ecs/magic-castle-service/mc-app \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s)000

# Search for specific patterns
aws logs filter-log-events \
  --log-group-name /aws/ecs/magic-castle-service/mc-app \
  --filter-pattern "database connection" \
  --start-time $(date -d '1 hour ago' +%s)000
```

## 🚨 Emergency Procedures

### Service Recovery

1. **Restart ECS service**:
   ```bash
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --force-new-deployment
   ```

2. **Scale down and up**:
   ```bash
   # Scale down
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --desired-count 0
   
   # Wait for tasks to stop
   aws ecs wait services-stable \
     --cluster magic-castle-cluster \
     --services magic-castle-service
   
   # Scale up
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --desired-count 1
   ```

3. **Rollback to previous task definition**:
   ```bash
   # Get previous task definition
   aws ecs describe-task-definition --task-definition magic-castle-service:13
   
   # Update service to use previous version
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --task-definition magic-castle-service:13
   ```

### Database Recovery

1. **Restore from backup**:
   ```bash
   # List available backups
   aws rds describe-db-snapshots \
     --db-instance-identifier magic-castle-database
   
   # Restore from snapshot
   aws rds restore-db-instance-from-db-snapshot \
     --db-instance-identifier magic-castle-database-restored \
     --db-snapshot-identifier <snapshot-id>
   ```

2. **Point-in-time recovery**:
   ```bash
   aws rds restore-db-instance-to-point-in-time \
     --source-db-instance-identifier magic-castle-database \
     --target-db-instance-identifier magic-castle-database-restored \
     --restore-time 2024-01-15T10:00:00Z
   ```

## 📚 Additional Resources

- [AWS ECS Troubleshooting](https://docs.aws.amazon.com/ecs/latest/developerguide/troubleshooting.html)
- [AWS RDS Troubleshooting](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Troubleshooting.html)
- [AWS ALB Troubleshooting](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html)
- [AWS Secrets Manager Troubleshooting](https://docs.aws.amazon.com/secretsmanager/latest/userguide/troubleshooting.html)
- [Application Documentation](../README.md)
- [Infrastructure Documentation](terraform/README.md)
- [API Documentation](docs/api.md)
