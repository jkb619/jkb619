# Deployment Guide

This guide provides step-by-step instructions for deploying the Magic Castle Guest Invitation System to AWS.

## 🎯 Overview

The application is deployed on AWS using:
- **ECS Fargate** for containerized application hosting
- **RDS MariaDB** for database storage
- **Application Load Balancer** for traffic routing
- **Route53** for DNS management
- **Secrets Manager** for secure credential storage

## 📋 Prerequisites

### Required Tools
- [AWS CLI](https://aws.amazon.com/cli/) (v2.x)
- [Terragrunt](https://terragrunt.gruntwork.io/) (latest version)
- [Terraform](https://terraform.io/) (>= 1.0)
- [Docker](https://docker.com/) (latest version)
- [SOPS](https://github.com/getsops/sops) (for secrets management)
- [Make](https://www.gnu.org/software/make/) (for build automation)

### AWS Account Setup
- AWS Account with appropriate permissions
- S3 bucket for Terraform state storage
- DynamoDB table for state locking
- ECR repository for Docker images

## 🚀 Deployment Steps

### Step 1: Environment Setup

1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd mc-guest-reservation
   ```

2. **Configure AWS CLI**:
   ```bash
   aws configure
   # Enter your AWS Access Key ID, Secret Access Key, and region (us-west-1)
   ```

3. **Verify AWS access**:
   ```bash
   aws sts get-caller-identity
   ```

### Step 2: Configure Secrets

1. **Edit secrets file**:
   ```bash
   sops terraform/secrets/config.yaml
   ```

2. **Add required secrets**:
   ```yaml
   database:
     host: "your-db-host"
     port: 3306
     name: "guest_reservations"
     username: "your-db-user"
     db_password: "your-secure-password"
   
   secrets:
     peoplevine_token: "your-peoplevine-token"
     sevenrooms_client_id: "your-sevenrooms-client-id"
    sevenrooms_client_secret: "your-sevenrooms-client-secret"
    sevenrooms_host: "https://api.sevenrooms.com"
    sevenrooms_version: "/2_4"
     ses_access_key_id: "your-ses-access-key"
     ses_secret_access_key: "your-ses-secret-key"
     ses_smtp_password: "your-ses-smtp-password"
   ```

3. **Verify encryption**:
   ```bash
   cat terraform/secrets/config.yaml
   # Should show encrypted content
   ```

### Step 3: Deploy Infrastructure

1. **Deploy all infrastructure**:
   ```bash
   cd terraform
   terragrunt run-all apply --auto-approve --terragrunt-non-interactive
   ```

2. **Monitor deployment**:
   ```bash
   # Check deployment status
   terragrunt run-all plan
   
   # View specific module status
   cd vpc && terragrunt plan
   ```

3. **Verify infrastructure**:
   ```bash
   # Check ECS cluster
   aws ecs describe-clusters --clusters magic-castle-cluster
   
   # Check RDS instance
   aws rds describe-db-instances --db-instance-identifier magic-castle-database
   
   # Check ALB
   aws elbv2 describe-load-balancers --names magic-castle-alb
   ```

### Step 4: Build and Deploy Application

1. **Build Docker image**:
   ```bash
   make build
   ```

2. **Push to ECR**:
   ```bash
   make deploy
   ```

3. **Verify image in ECR**:
   ```bash
   aws ecr describe-images --repository-name mc --image-ids imageTag=latest
   ```

### Step 5: Deploy Application

1. **Check ECS service status**:
   ```bash
   aws ecs describe-services --cluster magic-castle-cluster --services magic-castle-service
   ```

2. **Force new deployment** (if needed):
   ```bash
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --force-new-deployment
   ```

3. **Monitor deployment**:
   ```bash
   # Watch service events
   aws ecs describe-services --cluster magic-castle-cluster --services magic-castle-service \
     --query 'services[0].events[:5]'
   
   # Check task status
   aws ecs list-tasks --cluster magic-castle-cluster --service-name magic-castle-service
   ```

### Step 6: Verify Deployment

1. **Check application health**:
   ```bash
   curl https://guest-reservations.magiccastle-cloud.com/health
   ```

2. **Test API endpoints**:
   ```bash
   # Test invitation creation
   curl -X POST https://guest-reservations.magiccastle-cloud.com/peoplevine-guest-invite \
     -H "Content-Type: application/json" \
     -d '{
       "first_name": "Test",
       "last_name": "User",
       "memberID": "test123",
       "guest_email": "test@example.com"
     }'
   ```

3. **Check ALB target health**:
   ```bash
   aws elbv2 describe-target-health \
     --target-group-arn $(aws elbv2 describe-target-groups \
       --names magic-castle-tg \
       --query 'TargetGroups[0].TargetGroupArn' \
       --output text)
   ```

## 🔧 Configuration Management

### Environment Variables

The application uses environment variables configured in the ECS task definition:

```json
{
  "environment": [
    {
      "name": "FLASK_ENV",
      "value": "production"
    },
    {
      "name": "SECRETS_MANAGER_ARN",
      "value": "arn:aws:secretsmanager:us-west-1:373055206579:secret:magic-castle-secrets-3YpGd9"
    },
    {
      "name": "IMAGE_VERSION",
      "value": "v1.1"
    }
  ]
}
```

### Secrets Management

Secrets are stored in AWS Secrets Manager and accessed by the application:

```json
{
  "secrets": [
    {
      "name": "PEOPLEVINE_TOKEN",
      "valueFrom": "arn:aws:secretsmanager:us-west-1:373055206579:secret:magic-castle-secrets-3YpGd9:PEOPLEVINE_TOKEN::"
    },
    {
      "name": "DB_HOST",
      "valueFrom": "arn:aws:secretsmanager:us-west-1:373055206579:secret:magic-castle-secrets-3YpGd9:DB_HOST::"
    }
  ]
}
```

## 🔄 Update Deployment

### Application Updates

1. **Build new image**:
   ```bash
   make build
   ```

2. **Push to ECR**:
   ```bash
   make deploy
   ```

3. **Update ECS service**:
   ```bash
   aws ecs update-service \
     --cluster magic-castle-cluster \
     --service magic-castle-service \
     --force-new-deployment
   ```

### Infrastructure Updates

1. **Update Terragrunt configuration**:
   ```bash
   cd terraform
   # Edit relevant terragrunt.hcl files
   ```

2. **Apply changes**:
   ```bash
   terragrunt run-all apply
   ```

3. **Verify changes**:
   ```bash
   terragrunt run-all plan
   ```

## 🧪 Testing Deployment

### Health Checks

1. **Application Health**:
   ```bash
   curl https://guest-reservations.magiccastle-cloud.com/health
   ```

2. **Database Connectivity**:
   ```bash
   # Connect to ECS task and test database
   aws ecs execute-command \
     --cluster magic-castle-cluster \
     --task <task-id> \
     --container mc-app \
     --interactive \
     --command "/bin/bash"
   ```

3. **Email Functionality**:
   ```bash
   # Test email sending
   curl -X POST https://guest-reservations.magiccastle-cloud.com/peoplevine-guest-invite \
     -H "Content-Type: application/json" \
     -d '{
       "first_name": "Test",
       "last_name": "User",
       "memberID": "test123",
       "guest_email": "your-email@example.com"
     }'
   ```

### Load Testing

1. **Basic Load Test**:
   ```bash
   # Install Apache Bench
   sudo apt-get install apache2-utils
   
   # Run load test
   ab -n 100 -c 10 https://guest-reservations.magiccastle-cloud.com/health
   ```

2. **API Load Test**:
   ```bash
   # Test invitation creation endpoint
   ab -n 50 -c 5 -p test-data.json -T application/json \
     https://guest-reservations.magiccastle-cloud.com/peoplevine-guest-invite
   ```

## 📊 Monitoring Deployment

### CloudWatch Logs

1. **View Application Logs**:
   ```bash
   aws logs tail /aws/ecs/magic-castle-service/mc-app --follow
   ```

2. **View Cluster Logs**:
   ```bash
   aws logs tail /aws/ecs/magic-castle-cluster --follow
   ```

### ECS Monitoring

1. **Service Status**:
   ```bash
   aws ecs describe-services --cluster magic-castle-cluster --services magic-castle-service
   ```

2. **Task Status**:
   ```bash
   aws ecs list-tasks --cluster magic-castle-cluster --service-name magic-castle-service
   ```

3. **Resource Utilization**:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/ECS \
     --metric-name CPUUtilization \
     --dimensions Name=ServiceName,Value=magic-castle-service Name=ClusterName,Value=magic-castle-cluster \
     --start-time 2024-01-01T00:00:00Z \
     --end-time 2024-01-01T23:59:59Z \
     --period 300 \
     --statistics Average
   ```

## 🚨 Troubleshooting

### Common Issues

1. **ECS Service Not Starting**:
   ```bash
   # Check CloudWatch logs
   aws logs tail /aws/ecs/magic-castle-service/mc-app --follow
   
   # Check task definition
   aws ecs describe-task-definition --task-definition magic-castle-service
   
   # Check security groups
   aws ec2 describe-security-groups --group-ids sg-05581195d90383c95
   ```

2. **Database Connection Issues**:
   ```bash
   # Check RDS status
   aws rds describe-db-instances --db-instance-identifier magic-castle-database
   
   # Check security groups
   aws ec2 describe-security-groups --group-ids <rds-security-group-id>
   
   # Test connectivity from ECS task
   aws ecs execute-command \
     --cluster magic-castle-cluster \
     --task <task-id> \
     --container mc-app \
     --interactive \
     --command "/bin/bash"
   ```

3. **ALB Health Check Failures**:
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

4. **Secrets Manager Access Issues**:
   ```bash
   # Check VPC endpoint
   aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.us-west-1.secretsmanager"
   
   # Check security groups
   aws ec2 describe-security-groups --group-ids sg-0f520e5a6cae19d7b
   
   # Test secrets access
   aws secretsmanager get-secret-value --secret-id magic-castle-secrets
   ```

### Debugging Commands

```bash
# Check all resources
aws ecs describe-services --cluster magic-castle-cluster --services magic-castle-service
aws rds describe-db-instances --db-instance-identifier magic-castle-database
aws elbv2 describe-load-balancers --names magic-castle-alb
aws secretsmanager describe-secret --secret-id magic-castle-secrets

# Check network connectivity
aws ec2 describe-security-groups --group-ids sg-05581195d90383c95
aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.us-west-1.secretsmanager"

# Check logs
aws logs tail /aws/ecs/magic-castle-service/mc-app --follow
aws logs tail /aws/ecs/magic-castle-cluster --follow
```

## 🧹 Cleanup

### Destroy Infrastructure

1. **Destroy all resources**:
   ```bash
   cd terraform
   terragrunt run-all destroy --auto-approve --terragrunt-non-interactive
   ```

2. **Verify cleanup**:
   ```bash
   # Check ECS cluster
   aws ecs describe-clusters --clusters magic-castle-cluster
   
   # Check RDS instance
   aws rds describe-db-instances --db-instance-identifier magic-castle-database
   
   # Check ALB
   aws elbv2 describe-load-balancers --names magic-castle-alb
   ```

### Manual Cleanup

If automatic cleanup fails:

```bash
# Delete ECS service
aws ecs delete-service --cluster magic-castle-cluster --service magic-castle-service

# Delete ECS cluster
aws ecs delete-cluster --cluster magic-castle-cluster

# Delete RDS instance
aws rds delete-db-instance --db-instance-identifier magic-castle-database --skip-final-snapshot

# Delete ALB
aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn>

# Delete secrets
aws secretsmanager delete-secret --secret-id magic-castle-secrets --force-delete-without-recovery
```

## 📚 Additional Resources

- [Infrastructure Documentation](terraform/README.md)
- [API Documentation](docs/api.md)
- [Docker Configuration](docker/README.md)
- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [Terragrunt Documentation](https://terragrunt.gruntwork.io/)
- [Terraform Documentation](https://terraform.io/docs/)
