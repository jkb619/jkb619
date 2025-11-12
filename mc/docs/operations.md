# Operations Guide

This guide provides comprehensive operational procedures for the Magic Castle Guest Invitation System.

## 🎯 Overview

This operations guide covers day-to-day operations, monitoring, maintenance, and incident response procedures for the Magic Castle Guest Invitation System.

## 📊 Monitoring & Alerting

### CloudWatch Metrics

#### Key Metrics to Monitor

**ECS Metrics**:
- `CPUUtilization` - Target: < 70%
- `MemoryUtilization` - Target: < 80%
- `RunningTaskCount` - Target: 1
- `DesiredTaskCount` - Target: 1

**ALB Metrics**:
- `TargetResponseTime` - Target: < 2 seconds
- `HTTPCode_Target_2XX_Count` - Target: > 95%
- `HTTPCode_Target_4XX_Count` - Target: < 5%
- `HTTPCode_Target_5XX_Count` - Target: < 1%

**RDS Metrics**:
- `CPUUtilization` - Target: < 70%
- `FreeableMemory` - Target: > 100 MB
- `DatabaseConnections` - Target: < 80% of max
- `FreeStorageSpace` - Target: > 1 GB

#### Setting Up Alarms

```bash
# ECS CPU Utilization Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-High-CPU" \
  --alarm-description "ECS service high CPU utilization" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ServiceName,Value=magic-castle-service Name=ClusterName,Value=magic-castle-cluster \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-west-1:373055206579:ops-alerts

# ALB Response Time Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "ALB-High-Response-Time" \
  --alarm-description "ALB high response time" \
  --metric-name TargetResponseTime \
  --namespace AWS/ApplicationELB \
  --statistic Average \
  --period 300 \
  --threshold 2 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=LoadBalancer,Value=app/magic-castle-alb/0fc7de4c212720ea \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-west-1:373055206579:ops-alerts
```

### Log Monitoring

#### Application Logs
```bash
# Monitor application logs
aws logs tail /aws/ecs/magic-castle-service/mc-app --follow

# Search for errors
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

#### Infrastructure Logs
```bash
# Monitor cluster logs
aws logs tail /aws/ecs/magic-castle-cluster --follow

# Monitor ALB access logs
aws logs tail /aws/elasticloadbalancing/magic-castle-alb --follow
```

## 🔧 Daily Operations

### Health Checks

#### Application Health
```bash
# Check application health
curl -s https://guest-reservations.magiccastle-cloud.com/health | jq .

# Expected response:
# {
#   "status": "healthy"
# }
```

#### Infrastructure Health
```bash
# Check ECS service status
aws ecs describe-services --cluster magic-castle-cluster --services magic-castle-service \
  --query 'services[0].{Status:status,RunningCount:runningCount,DesiredCount:desiredCount}'

# Check ALB target health
aws elbv2 describe-target-health \
  --target-group-arn $(aws elbv2 describe-target-groups \
    --names magic-castle-tg \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)

# Check RDS instance status
aws rds describe-db-instances --db-instance-identifier magic-castle-database \
  --query 'DBInstances[0].DBInstanceStatus'
```

### Performance Monitoring

#### Resource Utilization
```bash
# Check ECS resource utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=magic-castle-service Name=ClusterName,Value=magic-castle-cluster \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average

# Check RDS resource utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=magic-castle-database \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

#### Database Performance
```bash
# Check database connections
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=magic-castle-database \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average

# Check free storage space
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=magic-castle-database \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

## 🔄 Maintenance Procedures

### Application Updates

#### Deploy New Version
```bash
# 1. Build new Docker image
make build

# 2. Push to ECR
make deploy

# 3. Update ECS service
aws ecs update-service \
  --cluster magic-castle-cluster \
  --service magic-castle-service \
  --force-new-deployment

# 4. Monitor deployment
aws ecs wait services-stable \
  --cluster magic-castle-cluster \
  --services magic-castle-service

# 5. Verify deployment
curl -s https://guest-reservations.magiccastle-cloud.com/health | jq .
```

#### Rollback Procedure
```bash
# 1. Get previous task definition
aws ecs describe-task-definition --task-definition magic-castle-service:13

# 2. Update service to previous version
aws ecs update-service \
  --cluster magic-castle-cluster \
  --service magic-castle-service \
  --task-definition magic-castle-service:13

# 3. Monitor rollback
aws ecs wait services-stable \
  --cluster magic-castle-cluster \
  --services magic-castle-service
```

### Infrastructure Updates

#### Update Infrastructure
```bash
# 1. Review changes
cd terraform
terragrunt run-all plan

# 2. Apply changes
terragrunt run-all apply

# 3. Verify changes
terragrunt run-all plan
```

#### Update Secrets
```bash
# 1. Edit secrets
sops terraform/secrets/config.yaml

# 2. Update Secrets Manager
cd terraform/secrets-manager
terragrunt apply

# 3. Restart ECS service to pick up new secrets
aws ecs update-service \
  --cluster magic-castle-cluster \
  --service magic-castle-service \
  --force-new-deployment
```

### Database Maintenance

#### Backup Verification
```bash
# Check recent backups
aws rds describe-db-snapshots \
  --db-instance-identifier magic-castle-database \
  --query 'DBSnapshots[?Status==`available`]' \
  --output table
```

#### Performance Tuning
```bash
# Check slow query log
aws rds describe-db-log-files \
  --db-instance-identifier magic-castle-database \
  --filename-contains slow

# Download slow query log
aws rds download-db-log-file-portion \
  --db-instance-identifier magic-castle-database \
  --log-file-name slowquery.log \
  --starting-token 0 \
  --max-items 1000
```

## 🚨 Incident Response

### Incident Classification

#### Severity Levels
- **P1 - Critical**: Service completely down, data loss
- **P2 - High**: Service degraded, significant impact
- **P3 - Medium**: Minor issues, workaround available
- **P4 - Low**: Cosmetic issues, no impact

#### Response Times
- **P1**: 15 minutes
- **P2**: 1 hour
- **P3**: 4 hours
- **P4**: 24 hours

### Incident Response Procedures

#### P1 - Critical Incident
1. **Immediate Response** (0-15 minutes):
   ```bash
   # Check service status
   aws ecs describe-services --cluster magic-castle-cluster --services magic-castle-service
   
   # Check application health
   curl -s https://guest-reservations.magiccastle-cloud.com/health
   
   # Check logs for errors
   aws logs tail /aws/ecs/magic-castle-service/mc-app --follow
   ```

2. **Initial Assessment** (15-30 minutes):
   ```bash
   # Check all infrastructure components
   aws ecs describe-clusters --clusters magic-castle-cluster
   aws rds describe-db-instances --db-instance-identifier magic-castle-database
   aws elbv2 describe-load-balancers --names magic-castle-alb
   
   # Check security groups
   aws ec2 describe-security-groups --group-ids sg-05581195d90383c95
   ```

3. **Recovery Actions** (30-60 minutes):
   ```bash
   # Restart ECS service
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --force-new-deployment
   
   # Scale down and up if needed
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --desired-count 0
   
   aws ecs wait services-stable \
     --cluster magic-castle-cluster \
     --services magic-castle-service
   
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --desired-count 1
   ```

#### P2 - High Incident
1. **Assessment** (0-30 minutes):
   ```bash
   # Check specific component
   aws ecs describe-services --cluster magic-castle-cluster --services magic-castle-service
   
   # Check metrics
   aws cloudwatch get-metric-statistics \
     --namespace AWS/ECS \
     --metric-name CPUUtilization \
     --dimensions Name=ServiceName,Value=magic-castle-service Name=ClusterName,Value=magic-castle-cluster \
     --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 300 \
     --statistics Average
   ```

2. **Resolution** (30-60 minutes):
   ```bash
   # Apply fixes based on assessment
   # Restart service if needed
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --force-new-deployment
   ```

### Emergency Procedures

#### Service Recovery
```bash
# Complete service restart
aws ecs update-service \
  --cluster magic-castle-cluster \
  --service magic-castle-service \
  --desired-count 0

aws ecs wait services-stable \
  --cluster magic-castle-cluster \
  --services magic-castle-service

aws ecs update-service \
  --cluster magic-castle-cluster \
  --service magic-castle-service \
  --desired-count 1

aws ecs wait services-stable \
  --cluster magic-castle-cluster \
  --services magic-castle-service
```

#### Database Recovery
```bash
# Restore from latest backup
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier magic-castle-database-restored \
  --db-snapshot-identifier $(aws rds describe-db-snapshots \
    --db-instance-identifier magic-castle-database \
    --query 'DBSnapshots[0].DBSnapshotIdentifier' \
    --output text)
```

## 📈 Performance Optimization

### Application Performance

#### Database Optimization
```bash
# Check database performance
aws rds describe-db-instances --db-instance-identifier magic-castle-database \
  --query 'DBInstances[0].{PerformanceInsightsEnabled:PerformanceInsightsEnabled,PerformanceInsightsRetentionPeriod:PerformanceInsightsRetentionPeriod}'

# Enable Performance Insights if not enabled
aws rds modify-db-instance \
  --db-instance-identifier magic-castle-database \
  --enable-performance-insights \
  --performance-insights-retention-period 7
```

#### Caching Implementation
```python
# Add Redis caching (future enhancement)
import redis
import json

redis_client = redis.Redis(host='redis-endpoint', port=6379, db=0)

def get_cached_data(key):
    cached = redis_client.get(key)
    if cached:
        return json.loads(cached)
    return None

def set_cached_data(key, data, ttl=3600):
    redis_client.setex(key, ttl, json.dumps(data))
```

### Infrastructure Performance

#### Auto Scaling
```bash
# Enable auto scaling for ECS service
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/magic-castle-cluster/magic-castle-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 \
  --max-capacity 10

# Create scaling policy
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/magic-castle-cluster/magic-castle-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name magic-castle-scaling-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    }
  }'
```

## 🔍 Troubleshooting

### Common Issues

#### Service Not Starting
```bash
# Check task definition
aws ecs describe-task-definition --task-definition magic-castle-service

# Check logs
aws logs tail /aws/ecs/magic-castle-service/mc-app --follow

# Check security groups
aws ec2 describe-security-groups --group-ids sg-05581195d90383c95
```

#### Database Connection Issues
```bash
# Check RDS status
aws rds describe-db-instances --db-instance-identifier magic-castle-database

# Check security groups
aws ec2 describe-security-groups --group-ids <rds-security-group-id>

# Test connection from ECS
aws ecs execute-command \
  --cluster magic-castle-cluster \
  --task <task-id> \
  --container mc-app \
  --interactive \
  --command "/bin/bash"
```

#### ALB Health Check Failures
```bash
# Check target health
aws elbv2 describe-target-health \
  --target-group-arn $(aws elbv2 describe-target-groups \
    --names magic-castle-tg \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)

# Check security groups
aws ec2 describe-security-groups --group-ids sg-05bf9ec12def62372
```

## 📋 Operational Checklists

### Daily Checklist
- [ ] Check application health endpoint
- [ ] Review CloudWatch metrics
- [ ] Check ECS service status
- [ ] Review application logs for errors
- [ ] Check ALB target health
- [ ] Verify database connectivity

### Weekly Checklist
- [ ] Review performance metrics
- [ ] Check backup status
- [ ] Review security group rules
- [ ] Update dependencies if needed
- [ ] Review capacity utilization
- [ ] Check certificate expiration

### Monthly Checklist
- [ ] Review and update documentation
- [ ] Perform security review
- [ ] Check compliance status
- [ ] Review cost optimization
- [ ] Update disaster recovery procedures
- [ ] Conduct incident response drill

## 📚 Additional Resources

### Documentation
- [Application Documentation](../README.md)
- [Infrastructure Documentation](terraform/README.md)
- [API Documentation](docs/api.md)
- [Security Documentation](docs/security.md)
- [Troubleshooting Guide](docs/troubleshooting.md)

### AWS Resources
- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [AWS RDS Documentation](https://docs.aws.amazon.com/rds/)
- [AWS ALB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/)
- [AWS CloudWatch Documentation](https://docs.aws.amazon.com/cloudwatch/)

### Tools
- [AWS CLI](https://aws.amazon.com/cli/)
- [Terragrunt](https://terragrunt.gruntwork.io/)
- [Docker](https://docker.com/)
- [SOPS](https://github.com/getsops/sops)

## 📞 Contact Information

### Operations Team
- **Primary On-call**: ops-primary@magic-castle.com
- **Secondary On-call**: ops-secondary@magic-castle.com
- **Manager**: ops-manager@magic-castle.com

### Emergency Contacts
- **24/7 Hotline**: +1-XXX-XXX-XXXX
- **Escalation**: ops-manager@magic-castle.com
- **AWS Support**: AWS Enterprise Support
