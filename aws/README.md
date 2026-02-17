# AWS ECS Deployment Setup

This directory contains configuration files for deploying Gogs to AWS ECS.

## Prerequisites

1. **AWS Account** with the following resources:
   - ECR repository: `gogs`
   - ECS cluster: `gogs-cluster`
   - ECS service: `gogs-service`
   - RDS PostgreSQL database
   - ElastiCache Redis cluster
   - EFS file system for persistent data
   - Application Load Balancer
   - VPC with private subnets

2. **GitHub Secrets** configured in your repository:
   - `AWS_ACCESS_KEY_ID`: AWS access key for deployment
   - `AWS_SECRET_ACCESS_KEY`: AWS secret access key

3. **AWS Secrets Manager** with the following secrets:
   - `gogs/database`: Contains database password
   - `gogs/app`: Contains application secret key

## Current Configuration

### Task Definition Details

- **Family**: `gogs-task`
- **Network Mode**: `awsvpc` (Fargate)
- **CPU**: 256 (0.25 vCPU)
- **Memory**: 512 MB
- **Platform**: Linux/AMD64

### Environment Variables

The following environment variables are configured in the task definition:

- `DB_TYPE`: postgres
- `DB_HOST`: gogs-db.c4ssagigcej8.us-east-1.rds.amazonaws.com:5432
- `DB_NAME`: gogs
- `DB_USER`: gogs
- `DOMAIN`: gogs-alb-374239415.us-east-1.elb.amazonaws.com
- `ROOT_URL`: http://gogs-alb-374239415.us-east-1.elb.amazonaws.com/
- `REDIS_ADDR`: gogs-redis.w4egbv.ng.0001.use1.cache.amazonaws.com:6379

### Secrets (from AWS Secrets Manager)

- `DB_PASSWORD`: Retrieved from `gogs/database` secret
- `SECRET_KEY`: Retrieved from `gogs/app` secret

### Storage

- **EFS File System**: fs-0f00f182e2c134wc0
- **Mount Point**: /data (for repositories and configuration)

## AWS Secrets Manager Setup

### Step 1: Create Database Secret

```bash
aws secretsmanager create-secret \
  --name gogs/database \
  --description "Gogs RDS PostgreSQL credentials" \
  --secret-string '{"password":"YOUR_DB_PASSWORD_HERE"}' \
  --region us-east-1
```

### Step 2: Create Application Secret

```bash
aws secretsmanager create-secret \
  --name gogs/app \
  --description "Gogs application secret key" \
  --secret-string '{"secret_key":"YOUR_SECRET_KEY_HERE"}' \
  --region us-east-1
```

### Step 3: Update IAM Execution Role

Add the following policy to your `ecsTaskExecutionRole`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:585177354759:secret:gogs/database-*",
        "arn:aws:secretsmanager:us-east-1:585177354759:secret:gogs/app-*"
      ]
    }
  ]
}
```

Apply the policy:

```bash
cat > secrets-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:585177354759:secret:gogs/database-*",
        "arn:aws:secretsmanager:us-east-1:585177354759:secret:gogs/app-*"
      ]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name GogsSecretsManagerAccess \
  --policy-document file://secrets-policy.json

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::585177354759:policy/GogsSecretsManagerAccess
```

## CI/CD Pipeline

The deployment pipeline is split into two workflows:

### CI Workflow (`.github/workflows/ci.yml`)

**Triggers**: Pull request to `staging` branch

**Jobs**:
1. **Lint**: Runs `task lint` to check code quality
2. **Test**: Runs `task test` with coverage reporting
3. **Security Scan**: Trivy filesystem vulnerability scan
4. **Go Mod Tidy Check**: Ensures dependencies are properly managed
5. **Build Test**: Validates Docker build without pushing
6. **CI Summary**: Aggregates all check results

### CD Workflow (`.github/workflows/cd.yml`)

**Triggers**: Push to `staging` branch (after PR merge)

**Jobs**:
1. **Build and Push**: 
   - Builds Docker image using `Dockerfile.aws`
   - Tags with commit SHA, short SHA, timestamp, and `latest`
   - Pushes to ECR
   - Scans image with Trivy
2. **Deploy**:
   - Updates ECS task definition with new image
   - Deploys to ECS service
   - Waits for service stability
   - Provides deployment summary

### Image Tags

Each build creates multiple tags:
- Full commit SHA: `abc123def456...`
- Short SHA: `abc123d`
- Timestamp: `20260217-143022`
- Latest: `latest`

## Deployment Process

1. **Create a feature branch** from `staging`
2. **Make your changes** and push
3. **Open a pull request** to `staging`
   - CI workflow runs automatically
   - Checks: linting, tests, security scan, Docker build test
4. **Merge PR** after CI passes
   - CD workflow runs automatically
   - Builds and pushes Docker image to ECR
   - Deploys to ECS service
5. **Monitor deployment** in GitHub Actions and AWS ECS console

## Rolling Updates and Rollback

ECS is configured for:
- **Rolling updates**: New tasks start before old ones stop
- **Auto rollback**: ECS rolls back if health checks fail
- **Health check**: `wget` against `http://localhost:3000/`
- **Stability wait**: GitHub Actions waits for successful deployment

## Monitoring and Logs

### CloudWatch Logs

```bash
# View live logs
aws logs tail /ecs/gogs --follow --region us-east-1

# View specific time range
aws logs tail /ecs/gogs --since 1h --region us-east-1
```

### ECS Service Status

```bash
# Get service details
aws ecs describe-services \
  --cluster gogs-cluster \
  --services gogs-service \
  --region us-east-1

# Get running tasks
aws ecs list-tasks \
  --cluster gogs-cluster \
  --service-name gogs-service \
  --region us-east-1
```

### Application Health

```bash
# Check health endpoint
curl http://gogs-alb-374239415.us-east-1.elb.amazonaws.com/healthcheck
```

## Troubleshooting

### Deployment Fails with "Service Unhealthy"

1. Check CloudWatch logs for application errors:
   ```bash
   aws logs tail /ecs/gogs --follow
   ```

2. Verify secrets are accessible:
   ```bash
   aws secretsmanager get-secret-value --secret-id gogs/database --region us-east-1
   aws secretsmanager get-secret-value --secret-id gogs/app --region us-east-1
   ```

3. Check ECS service events:
   ```bash
   aws ecs describe-services \
     --cluster gogs-cluster \
     --services gogs-service \
     --query 'services[0].events' \
     --region us-east-1
   ```

### Container Fails to Start

1. Verify IAM role permissions:
   - Task execution role can pull from ECR
   - Task execution role can read from Secrets Manager
   - Task role can access EFS

2. Check EFS mount:
   ```bash
   aws efs describe-mount-targets \
     --file-system-id fs-0f00f182e2c134wc0 \
     --region us-east-1
   ```

3. Verify security groups:
   - ECS tasks can reach RDS (port 5432)
   - ECS tasks can reach Redis (port 6379)
   - ECS tasks can reach EFS (port 2049)

### Database Connection Errors

1. Test database connectivity from task:
   ```bash
   # Get task ARN
   TASK_ARN=$(aws ecs list-tasks \
     --cluster gogs-cluster \
     --service-name gogs-service \
     --query 'taskArns[0]' \
     --output text)
   
   # Execute command (requires ECS Exec enabled)
   aws ecs execute-command \
     --cluster gogs-cluster \
     --task $TASK_ARN \
     --container gogs \
     --interactive \
     --command "/bin/sh"
   ```

2. Verify RDS endpoint and credentials in Secrets Manager

3. Check RDS security group allows inbound from ECS security group

### Build Failures

1. Check GitHub Actions logs for specific error
2. Verify `Dockerfile.aws` syntax
3. Ensure all build dependencies are available
4. Check Task version compatibility

## Security Best Practices

1. ✅ **Secrets in Secrets Manager** - Database password and secret key stored securely
2. ✅ **IAM Role-based access** - No hardcoded credentials in task definition
3. ✅ **Vulnerability scanning** - Trivy scans on every build
4. ✅ **Non-root user** - Container runs as `git:git` (UID 1000)
5. ✅ **Private subnets** - ECS tasks run in private subnets
6. ✅ **Security groups** - Least privilege network access
7. ✅ **Encrypted at rest** - EFS and RDS use encryption

## Cost Optimization

Current configuration:
- **Fargate**: 256 CPU, 512 MB memory (~$10-15/month at 100% uptime)
- **RDS**: Check instance type and storage costs
- **EFS**: Pay per GB stored and accessed
- **Secrets Manager**: ~$0.80/month (2 secrets)
- **ECR**: First 500 MB free, then $0.10/GB/month
- **Data transfer**: Depends on usage

## Updating Configuration

### Change Environment Variables

Edit `aws/task-definition.json` and push to `staging` branch. CD pipeline will deploy automatically.

### Change Secrets

```bash
# Update database password
aws secretsmanager update-secret \
  --secret-id gogs/database \
  --secret-string '{"password":"NEW_PASSWORD"}' \
  --region us-east-1

# Update secret key
aws secretsmanager update-secret \
  --secret-id gogs/app \
  --secret-string '{"secret_key":"NEW_SECRET_KEY"}' \
  --region us-east-1

# Restart tasks to pick up new secrets
aws ecs update-service \
  --cluster gogs-cluster \
  --service gogs-service \
  --force-new-deployment \
  --region us-east-1
```

### Scale Service

```bash
# Update desired task count
aws ecs update-service \
  --cluster gogs-cluster \
  --service gogs-service \
  --desired-count 2 \
  --region us-east-1
```

## Support

For issues related to:
- **AWS Infrastructure**: Check AWS documentation and CloudWatch logs
- **GitHub Actions**: Review workflow logs in Actions tab
- **Gogs Application**: See main project README and documentation
- **Docker Build**: Check `Dockerfile.aws` and build logs
