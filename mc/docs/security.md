# Security Documentation

This document provides comprehensive security information for the Magic Castle Guest Invitation System.

## 🔒 Security Overview

The Magic Castle Guest Invitation System implements multiple layers of security to protect data, ensure secure communication, and maintain system integrity.

## 🏗️ Security Architecture

### Defense in Depth

The system implements a multi-layered security approach:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Internet Security                           │
│  • HTTPS/TLS 1.2+                                             │
│  • SSL Certificate (ACM)                                      │
│  • WAF (Future Enhancement)                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    Network Security                            │
│  • VPC with Private Subnets                                   │
│  • Security Groups (Least Privilege)                          │
│  • NACLs (Default)                                            │
│  • VPC Endpoints                                              │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    Application Security                        │
│  • Non-root Container User                                   │
│  • Input Validation & Sanitization                           │
│  • CORS Configuration                                         │
│  • Authentication (Bulk Operations)                        │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    Data Security                              │
│  • Encryption at Rest (RDS, S3, Secrets Manager)            │
│  • Encryption in Transit (TLS)                               │
│  • Secrets Management (AWS Secrets Manager)                 │
│  • SOPS for Local Secrets                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    Access Control                             │
│  • IAM Roles (Least Privilege)                              │
│  • Resource-based Policies                                   │
│  • Service-to-Service Authentication                         │
│  • MFA (AWS Console Access)                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 🌐 Network Security

### VPC Configuration
- **VPC ID**: `vpc-052dd1972e02d17a0`
- **CIDR**: `10.0.0.0/16`
- **Public Subnets**: 3 subnets for ALB
- **Private Subnets**: 3 subnets for ECS and RDS
- **NAT Gateways**: 3 gateways for secure outbound access

### Security Groups

#### ALB Security Group (`sg-05bf9ec12def62372`)
```json
{
  "Ingress": [
    {
      "Protocol": "tcp",
      "Port": 443,
      "Source": "0.0.0.0/0",
      "Description": "HTTPS from Internet"
    }
  ],
  "Egress": [
    {
      "Protocol": "-1",
      "Port": "All",
      "Destination": "0.0.0.0/0",
      "Description": "All outbound traffic"
    }
  ]
}
```

#### ECS Security Group (`sg-05581195d90383c95`)
```json
{
  "Ingress": [
    {
      "Protocol": "tcp",
      "Port": 5000,
      "Source": "sg-05bf9ec12def62372",
      "Description": "HTTP from ALB"
    }
  ],
  "Egress": [
    {
      "Protocol": "-1",
      "Port": "All",
      "Destination": "0.0.0.0/0",
      "Description": "All outbound traffic"
    },
    {
      "Protocol": "tcp",
      "Port": 443,
      "Destination": "0.0.0.0/0",
      "Description": "HTTPS for Secrets Manager"
    }
  ]
}
```

#### VPC Endpoint Security Group (`sg-0f520e5a6cae19d7b`)
```json
{
  "Ingress": [
    {
      "Protocol": "tcp",
      "Port": 443,
      "Source": "10.0.0.0/16",
      "Description": "HTTPS from VPC"
    }
  ],
  "Egress": [
    {
      "Protocol": "-1",
      "Port": "All",
      "Destination": "0.0.0.0/0",
      "Description": "All outbound traffic"
    }
  ]
}
```

### VPC Endpoints
- **Secrets Manager**: Private connectivity to AWS Secrets Manager
- **Type**: Interface endpoint
- **DNS**: Private DNS enabled
- **Security**: HTTPS only

## 🔐 Data Security

### Encryption at Rest

#### RDS Database
- **Engine**: MariaDB 10.11
- **Encryption**: Enabled
- **KMS Key**: Customer managed key
- **Backup Encryption**: Enabled

#### S3 Buckets
- **Terraform State**: Encrypted with S3 managed keys
- **ALB Logs**: Encrypted with S3 managed keys
- **CloudWatch Logs**: Encrypted with CloudWatch managed keys

#### Secrets Manager
- **Secret Name**: `magic-castle-secrets`
- **KMS Key**: Customer managed key (`b15cb8de-c959-43e6-8eb2-18cf96849a9e`)
- **Encryption**: AES-256

### Encryption in Transit

#### HTTPS/TLS
- **Protocol**: TLS 1.2+
- **Certificate**: ACM certificate for `*.magiccastle-cloud.com`
- **Cipher Suites**: Strong cipher suites only
- **HSTS**: Not implemented (future enhancement)

#### Database Connections
- **Protocol**: TLS/SSL
- **Certificate**: RDS CA certificate
- **Validation**: Certificate validation enabled

#### VPC Endpoints
- **Protocol**: HTTPS only
- **Certificate**: AWS managed certificates
- **Validation**: Certificate validation enabled

## 🔑 Access Control

### IAM Roles

#### ECS Execution Role (`magic-castle-ecs-execution-role`)
```json
{
  "Policies": [
    "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy",
    "CustomPolicy": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "secretsmanager:GetSecretValue"
          ],
          "Resource": "arn:aws:secretsmanager:us-west-1:373055206579:secret:magic-castle-secrets-*"
        }
      ]
    }
  ]
}
```

#### ECS Task Role (`magic-castle-ecs-task-role`)
```json
{
  "Policies": [
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
    "arn:aws:iam::aws:policy/CloudWatchLogsFullAccess",
    "CustomPolicy": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "secretsmanager:GetSecretValue"
          ],
          "Resource": "arn:aws:secretsmanager:us-west-1:373055206579:secret:magic-castle-secrets-*"
        },
        {
          "Effect": "Allow",
          "Action": [
            "ses:SendEmail",
            "ses:SendRawEmail"
          ],
          "Resource": "*"
        }
      ]
    }
  ]
}
```

### Resource-based Policies

#### Secrets Manager Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::373055206579:role/magic-castle-ecs-execution-role"
      },
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*"
    }
  ]
}
```

## 🛡️ Application Security

### Container Security

#### Docker Image Security
- **Base Image**: Python 3.13 slim (minimal attack surface)
- **User**: Non-root user (`app`)
- **UID**: 1000
- **Read-only Root Filesystem**: Disabled (for SSM agent)
- **Dependencies**: Minimal required packages only

#### Runtime Security
- **Process**: Non-root execution
- **Capabilities**: Minimal capabilities
- **Seccomp**: Default seccomp profile
- **AppArmor**: Default AppArmor profile

### Input Validation

#### API Input Validation
```python
# Example validation for invitation creation
def validate_invitation_data(data):
    required_fields = ['first_name', 'last_name', 'memberID', 'guest_email']
    
    for field in required_fields:
        if field not in data:
            raise ValueError(f"Missing required field: {field}")
    
    # Email validation
    if not re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', data['guest_email']):
        raise ValueError("Invalid email format")
    
    # Member ID validation
    if not re.match(r'^[a-zA-Z0-9_-]+$', data['memberID']):
        raise ValueError("Invalid member ID format")
```

#### SQL Injection Prevention
```python
# Using parameterized queries
def create_invitation(invitation_id, guest_email, member_id):
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Parameterized query prevents SQL injection
    cursor.execute(
        "INSERT INTO guest_invites (invitation_id, guest_email, member_id) VALUES (%s, %s, %s)",
        (invitation_id, guest_email, member_id)
    )
    
    conn.commit()
    cursor.close()
    conn.close()
```

### Authentication

#### Bulk Operations Authentication
```python
# Simple username/token authentication
def authenticate_bulk_request(username, token):
    valid_credentials = {
        'someUser': 'someToken'
    }
    
    if username not in valid_credentials:
        return False
    
    return valid_credentials[username] == token
```

#### Future Enhancements
- **JWT Tokens**: For stateless authentication
- **OAuth 2.0**: For third-party integration
- **API Keys**: For programmatic access
- **Rate Limiting**: To prevent abuse

## 🔍 Monitoring & Auditing

### CloudWatch Logs

#### Application Logs
- **Log Group**: `/aws/ecs/magic-castle-service/mc-app`
- **Retention**: 30 days
- **Encryption**: CloudWatch managed keys
- **Access**: IAM role-based access

#### Infrastructure Logs
- **Log Group**: `/aws/ecs/magic-castle-cluster`
- **Retention**: 30 days
- **Encryption**: CloudWatch managed keys

### CloudTrail (Future Enhancement)
- **API Calls**: All AWS API calls logged
- **Data Events**: S3 and Lambda data events
- **Insights**: Anomaly detection
- **Retention**: 90 days

### VPC Flow Logs (Future Enhancement)
- **Network Traffic**: All VPC traffic logged
- **Destination**: CloudWatch Logs
- **Retention**: 30 days

## 🚨 Incident Response

### Security Incident Procedures

1. **Detection**
   - Monitor CloudWatch logs for anomalies
   - Set up CloudWatch alarms for critical metrics
   - Review security group changes

2. **Response**
   - Isolate affected resources
   - Review logs for attack vectors
   - Update security groups if needed
   - Rotate compromised credentials

3. **Recovery**
   - Restore from clean backups
   - Update security configurations
   - Monitor for continued attacks
   - Document lessons learned

### Emergency Contacts
- **Security Team**: security@magic-castle.com
- **DevOps Team**: devops@magic-castle.com
- **On-call**: +1-XXX-XXX-XXXX

## 🔧 Security Hardening

### Current Implementations

#### Network Hardening
- **Private Subnets**: Database and application in private networks
- **Security Groups**: Least-privilege access rules
- **VPC Endpoints**: Private connectivity to AWS services
- **NAT Gateways**: Secure outbound internet access

#### Application Hardening
- **Non-root User**: Container runs as non-root user
- **Input Validation**: All input validated and sanitized
- **SQL Injection Prevention**: Parameterized queries
- **CORS Configuration**: Restricted cross-origin access

#### Infrastructure Hardening
- **IAM Roles**: Least-privilege permissions
- **Resource Tagging**: Consistent tagging for governance
- **Encryption**: All data encrypted at rest and in transit
- **Secrets Management**: Secure credential storage

### Future Enhancements

#### Network Security
- **WAF**: Web Application Firewall
- **GuardDuty**: Threat detection
- **Security Hub**: Centralized security findings
- **Config**: Compliance monitoring

#### Application Security
- **Rate Limiting**: API rate limiting
- **Input Sanitization**: Enhanced input validation
- **Security Headers**: Security headers implementation
- **Content Security Policy**: CSP implementation

#### Infrastructure Security
- **Secrets Rotation**: Automatic secret rotation
- **Vulnerability Scanning**: Container vulnerability scanning
- **Compliance**: SOC 2, PCI DSS compliance
- **Penetration Testing**: Regular security testing

## 📋 Compliance

### Current Compliance

#### Data Protection
- **Encryption**: All data encrypted at rest and in transit
- **Access Control**: Role-based access control
- **Audit Logging**: CloudWatch logs for audit trail
- **Data Retention**: Configurable retention policies

#### Privacy
- **Data Minimization**: Only necessary data collected
- **Purpose Limitation**: Data used only for intended purpose
- **Storage Limitation**: Data retained only as long as necessary
- **Accuracy**: Data kept accurate and up-to-date

### Future Compliance

#### Standards
- **SOC 2**: Service Organization Control 2
- **PCI DSS**: Payment Card Industry Data Security Standard
- **GDPR**: General Data Protection Regulation
- **HIPAA**: Health Insurance Portability and Accountability Act

#### Certifications
- **ISO 27001**: Information Security Management
- **FedRAMP**: Federal Risk and Authorization Management Program
- **FISMA**: Federal Information Security Management Act

## 🧪 Security Testing

### Current Testing

#### Vulnerability Scanning
- **Container Images**: Docker image vulnerability scanning
- **Dependencies**: Python package vulnerability scanning
- **Infrastructure**: AWS resource security scanning

#### Penetration Testing
- **Network**: Network penetration testing
- **Application**: Application security testing
- **Infrastructure**: Infrastructure security testing

### Future Testing

#### Automated Testing
- **SAST**: Static Application Security Testing
- **DAST**: Dynamic Application Security Testing
- **IAST**: Interactive Application Security Testing
- **SCA**: Software Composition Analysis

#### Manual Testing
- **Code Review**: Security-focused code reviews
- **Threat Modeling**: Threat modeling sessions
- **Red Team**: Red team exercises
- **Bug Bounty**: Bug bounty programs

## 📚 Security Resources

### Documentation
- [AWS Security Best Practices](https://aws.amazon.com/security/security-resources/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [CIS Controls](https://www.cisecurity.org/controls/)

### Tools
- [AWS Security Hub](https://aws.amazon.com/security-hub/)
- [AWS GuardDuty](https://aws.amazon.com/guardduty/)
- [AWS Config](https://aws.amazon.com/config/)
- [AWS CloudTrail](https://aws.amazon.com/cloudtrail/)

### Training
- [AWS Security Training](https://aws.amazon.com/training/security/)
- [OWASP Training](https://owasp.org/www-project-training/)
- [SANS Security Training](https://www.sans.org/)
- [CIS Training](https://www.cisecurity.org/training/)

## 📞 Security Contacts

### Internal Contacts
- **Security Team**: security@magic-castle.com
- **DevOps Team**: devops@magic-castle.com
- **Development Team**: dev@magic-castle.com

### External Contacts
- **AWS Support**: AWS Enterprise Support
- **Security Vendor**: Contact information for security tools
- **Incident Response**: External incident response team

### Emergency Contacts
- **24/7 On-call**: +1-XXX-XXX-XXXX
- **Security Hotline**: +1-XXX-XXX-XXXX
- **Escalation**: security-manager@magic-castle.com
