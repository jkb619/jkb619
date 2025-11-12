# Architecture Documentation

This document provides a comprehensive overview of the Magic Castle Guest Invitation System architecture.

## 🏗️ System Architecture

### High-Level Architecture

The system follows a modern microservices architecture deployed on AWS with the following key components:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Internet                                 │
└─────────────────────┬───────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────────┐
│                    Route53 DNS                                 │
│              guest-reservations.magiccastle-cloud.com              │
└─────────────────────┬───────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────────┐
│              Application Load Balancer                          │
│                    (HTTPS:443)                                  │
│              magic-castle-alb                                   │
└─────────────────────┬───────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────────┐
│                    VPC                                         │
│              vpc-052dd1972e02d17a0                             │
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────────────────────────┐ │
│  │  Public Subnets │    │         Private Subnets            │ │
│  │                 │    │                                     │ │
│  │  ┌───────────┐ │    │  ┌─────────────┐  ┌─────────────┐  │ │
│  │  │    ALB    │ │    │  │    ECS      │  │    RDS      │  │ │
│  │  │           │ │    │  │  Fargate    │  │  MariaDB    │  │ │
│  │  └───────────┘ │    │  │             │  │             │  │ │
│  └─────────────────┘    │  └─────────────┘  └─────────────┘  │ │
│                         │                                     │ │
│                         │  ┌─────────────────────────────────┐ │ │
│                         │  │      VPC Endpoints             │ │ │
│                         │  │   Secrets Manager              │ │ │
│                         │  └─────────────────────────────────┘ │ │
│                         └─────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────────┐
│                    AWS Services                                 │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Secrets   │  │     SES     │  │ CloudWatch  │             │
│  │   Manager   │  │             │  │    Logs     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

## 🔧 Component Details

### 1. Application Layer

#### Flask Application
- **Framework**: Flask 3.0.0
- **Language**: Python 3.13
- **Container**: Docker with Python 3.13 slim base image
- **Port**: 5000
- **Health Check**: `/health` endpoint

#### API Endpoints
- `GET /health` - Health check
- `POST /peoplevine-guest-invite` - Create single invitation
- `POST /peoplevine-generate-invitations` - Bulk invitation generation
- `GET /guest-invite-accept` - Accept invitation
- `GET /admin-dump-database` - List all invitations

### 2. Infrastructure Layer

#### ECS Fargate
- **Cluster**: `magic-castle-cluster`
- **Service**: `magic-castle-service`
- **Task Definition**: `magic-castle-service:14`
- **CPU**: 512 units
- **Memory**: 1024 MB
- **Platform**: Fargate
- **Execute Command**: Enabled for debugging

#### Application Load Balancer
- **Type**: Application Load Balancer
- **Scheme**: Internet-facing
- **Listeners**: HTTPS (443) only
- **Target Group**: `magic-castle-tg`
- **Health Check**: `/health` endpoint
- **SSL Certificate**: ACM certificate for `*.magiccastle-cloud.com`

#### RDS Database
- **Engine**: MariaDB 10.11
- **Instance**: db.t3.micro
- **Storage**: 20 GB GP2
- **Backup**: 7 days retention
- **Multi-AZ**: Disabled (cost optimization)
- **Encryption**: Enabled at rest

### 3. Security Layer

#### Security Groups

**ALB Security Group** (`sg-05bf9ec12def62372`):
- Ingress: HTTPS (443) from 0.0.0.0/0
- Egress: All traffic to 0.0.0.0/0

**ECS Security Group** (`sg-05581195d90383c95`):
- Ingress: HTTP (5000) from ALB security group
- Egress: All traffic to 0.0.0.0/0, HTTPS (443) to 0.0.0.0/0

**VPC Endpoint Security Group** (`sg-0f520e5a6cae19d7b`):
- Ingress: HTTPS (443) from VPC CIDR
- Egress: All traffic to 0.0.0.0/0

#### IAM Roles

**ECS Execution Role** (`magic-castle-ecs-execution-role`):
- ECR read access
- CloudWatch Logs write access
- Secrets Manager read access

**ECS Task Role** (`magic-castle-ecs-task-role`):
- Secrets Manager read access
- SES API send access (SendEmail, SendRawEmail)
- CloudWatch Logs write access

### 4. Data Layer

#### Secrets Management
- **Service**: AWS Secrets Manager
- **Secret Name**: `magic-castle-secrets`
- **KMS Key**: Customer managed key
- **Secrets**: Database credentials, API tokens
- **Note**: SES authentication handled via ECS task role (no SMTP credentials stored)

#### Database Schema
```sql
CREATE TABLE guest_invites (
    invitation_id VARCHAR(36) PRIMARY KEY,
    guest_email VARCHAR(255) NOT NULL,
    member_id VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    used BOOLEAN DEFAULT FALSE
);
```

### 5. Network Layer

#### VPC Configuration
- **VPC ID**: `vpc-052dd1972e02d17a0`
- **CIDR**: `10.0.0.0/16`
- **Public Subnets**: 3 subnets across AZs
- **Private Subnets**: 3 subnets across AZs
- **NAT Gateways**: 3 gateways for private subnet internet access
- **Internet Gateway**: For public subnet internet access

#### VPC Endpoints
- **Secrets Manager**: Private connectivity to AWS Secrets Manager
- **Type**: Interface endpoint
- **DNS**: Private DNS enabled

### 6. Monitoring Layer

#### CloudWatch Logs
- **ECS Logs**: `/aws/ecs/magic-castle-service/mc-app`
- **Cluster Logs**: `/aws/ecs/magic-castle-cluster`
- **Retention**: 30 days

#### Health Monitoring
- **Application**: `/health` endpoint
- **ALB**: HTTP health checks on port 5000
- **ECS**: Container health monitoring

## 🔄 Data Flow

### 1. Invitation Creation Flow

```
Client Request → ALB → ECS → Database → Secrets Manager → SES → Email Sent
     ↓              ↓      ↓        ↓           ↓          ↓
   HTTPS:443    HTTP:5000  Store   Get Creds   Send Email  Guest Receives
```

### 2. Invitation Acceptance Flow

```
Guest Click → ALB → ECS → Database → Update Status → Response
     ↓          ↓      ↓        ↓           ↓
   HTTPS:443  HTTP:5000  Query   Mark Used   Success
```

### 3. Bulk Invitation Generation Flow

```
Client Request → ALB → ECS → Authentication → Database → Multiple IDs → Response
     ↓              ↓      ↓        ↓            ↓           ↓
   HTTPS:443    HTTP:5000  Verify  Generate    Store IDs    Return IDs
```

## 🔒 Security Architecture

### Network Security
- **Private Subnets**: Database and application in private networks
- **Security Groups**: Least-privilege network access
- **VPC Endpoints**: Private connectivity to AWS services
- **NAT Gateways**: Secure outbound internet access

### Data Security
- **Encryption at Rest**: RDS, S3, Secrets Manager
- **Encryption in Transit**: HTTPS, TLS for database connections
- **Secrets Management**: AWS Secrets Manager with KMS encryption
- **SOPS**: Encrypted secrets in version control

### Access Control
- **IAM Roles**: Service-to-service authentication
- **Least Privilege**: Minimal required permissions
- **Resource Tagging**: Consistent tagging for governance

## 📊 Performance Architecture

### Scalability
- **ECS Auto Scaling**: CPU and memory-based scaling
- **ALB**: Automatic traffic distribution
- **RDS**: Manual scaling (can be automated)
- **Fargate**: Serverless container platform

### Performance Optimization
- **Connection Pooling**: Database connection optimization
- **Caching**: Application-level caching
- **CDN**: Not implemented (can be added)
- **Load Balancing**: ALB with health checks

## 🔄 Deployment Architecture

### CI/CD Pipeline
```
Code Push → Docker Build → ECR Push → ECS Update → Health Check → Deployment Complete
```

### Infrastructure as Code
- **Terragrunt**: Infrastructure management
- **Terraform**: Resource provisioning
- **SOPS**: Secrets management
- **Git**: Version control

## 🧪 Testing Architecture

### Health Checks
- **Application**: `/health` endpoint
- **Infrastructure**: ALB and ECS health checks
- **Database**: Connection health checks
- **Secrets**: Access health checks

### Monitoring
- **CloudWatch**: Metrics and logs
- **ECS**: Service and task monitoring
- **ALB**: Load balancer metrics
- **RDS**: Database performance insights

## 🚨 Disaster Recovery

### Backup Strategy
- **RDS**: Automated backups (7 days retention)
- **Secrets**: AWS Secrets Manager replication
- **Infrastructure**: Terraform state in S3
- **Application**: Docker images in ECR

### Recovery Procedures
- **Database**: Point-in-time recovery
- **Application**: ECS service restart
- **Infrastructure**: Terraform apply
- **Secrets**: Secrets Manager restore

## 📈 Cost Optimization

### Resource Sizing
- **ECS Tasks**: 512 CPU / 1024 MB memory
- **RDS Instance**: db.t3.micro
- **Storage**: GP2 with 20 GB initial allocation

### Cost Management
- **Reserved Instances**: Consider for RDS
- **Spot Instances**: Not applicable for Fargate
- **S3 Lifecycle**: Log retention policies
- **CloudWatch**: Log retention policies

## 🔮 Future Enhancements

### Scalability Improvements
- **Multi-AZ RDS**: For high availability
- **ECS Service Discovery**: For microservices
- **API Gateway**: For API management
- **CloudFront**: For CDN capabilities

### Security Enhancements
- **WAF**: Web Application Firewall
- **GuardDuty**: Threat detection
- **Config**: Compliance monitoring
- **Secrets Rotation**: Automatic secret rotation

### Monitoring Improvements
- **X-Ray**: Distributed tracing
- **CloudWatch Insights**: Log analytics
- **Custom Metrics**: Application metrics
- **Alerting**: Proactive monitoring

## 📚 Additional Resources

- [Application Documentation](../README.md)
- [Infrastructure Documentation](terraform/README.md)
- [API Documentation](docs/api.md)
- [Deployment Guide](docs/deployment.md)
- [Docker Configuration](docker/README.md)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
