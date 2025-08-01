# AWS Fargate Example

A minimal example demonstrating how to deploy a simple web application to AWS Fargate using Amazon ECS.

## Purpose

This project shows the complete workflow for:
- Containerizing a simple web application with Docker
- Configuring an ECS task definition for Fargate
- Deploying a serverless container to AWS

The example serves a basic "Hello from Fargate!" webpage using nginx.

## Prerequisites

- AWS CLI configured with appropriate permissions
- Docker installed and configured
- AWS account with the following services enabled:
  - Amazon ECS
  - Amazon ECR
  - AWS Fargate
  - CloudWatch Logs

## Required IAM Permissions

Your AWS user/role needs permissions for:
- ECS (creating services, tasks, task definitions)
- ECR (pushing/pulling container images)
- CloudWatch Logs (creating log groups)
- IAM (using the execution role)

## Configuration Variables

Before using this example, you need to replace the following variables in `task-def.json`:

### Required Replacements

| Variable       | Location     | Description                   | Example         |
|----------------|--------------|-------------------------------|-----------------|
| `<account_id>` | Line 7 & 11  | Your 12-digit AWS account ID  | `123456789012`  |

### Optional Customizations

You may also want to customize these values based on your preferences:

| Field            | Current Value          | Description                        |
|------------------|------------------------|------------------------------------|
| `family`         | `my-fargate-task`      | Task definition family name        |
| `cpu`            | `256`                  | CPU units (256 = 0.25 vCPU)        |
| `memory`         | `512`                  | Memory in MB                       |
| `awslogs-region` | `us-west-2`            | AWS region for CloudWatch logs     |
| `awslogs-group`  | `/ecs/my-fargate-task` | CloudWatch log group name          |
| Container `name` | `my-nginx`             | Container name within the task     |
| ECR repository   | `my-fargate-app`       | ECR repository name                |

## Deployment Steps

### 1. Create ECR Repository
```bash
aws ecr create-repository --repository-name my-fargate-app --region us-west-2
```

### 2. Build and Push Docker Image
```bash
# Get login token for ECR
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <account_id>.dkr.ecr.us-west-2.amazonaws.com

# Build the image
docker build -t my-fargate-app .

# Tag the image
docker tag my-fargate-app:latest <account_id>.dkr.ecr.us-west-2.amazonaws.com/my-fargate-app:latest

# Push to ECR
docker push <account_id>.dkr.ecr.us-west-2.amazonaws.com/my-fargate-app:latest
```

### 3. Create CloudWatch Log Group
```bash
aws logs create-log-group --log-group-name /ecs/my-fargate-task --region us-west-2
```

### 4. Create ECS Task Execution Role
```bash
# Create the role (if it doesn't exist)
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'

# Attach the managed policy
aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### 5. Register Task Definition
```bash
aws ecs register-task-definition --cli-input-json file://task-def.json --region us-west-2
```

### 6. Create ECS Cluster
```bash
aws ecs create-cluster --cluster-name my-fargate-cluster --region us-west-2
```

### 7. Run the Task
```bash
aws ecs run-task \
  --cluster my-fargate-cluster \
  --task-definition my-fargate-task \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxxx],securityGroups=[sg-xxxxxx],assignPublicIp=ENABLED}" \
  --region us-west-2
```

> **Note**: Replace `subnet-xxxxxx` and `sg-xxxxxx` with your actual subnet and security group IDs. The security group should allow inbound traffic on port 80.

## Testing

Once deployed, you can find the public IP of your task in the ECS console and access your application at `http://<public-ip>`. You should see the "It works 🎉" message.

## Cleanup

To avoid ongoing charges, clean up the resources:

```bash
# Stop running tasks
aws ecs list-tasks --cluster my-fargate-cluster --region us-west-2
aws ecs stop-task --cluster my-fargate-cluster --task <task-arn> --region us-west-2

# Delete cluster
aws ecs delete-cluster --cluster my-fargate-cluster --region us-west-2

# Delete ECR repository
aws ecr delete-repository --repository-name my-fargate-app --force --region us-west-2

# Delete log group
aws logs delete-log-group --log-group-name /ecs/my-fargate-task --region us-west-2
```

## Cost Considerations

- Fargate pricing is based on vCPU and memory resources used
- This minimal example (0.25 vCPU, 512MB RAM) costs approximately $0.01-0.02 per hour
- CloudWatch logs have minimal costs for this simple application
- Remember to stop tasks when not needed to avoid unnecessary charges

## Troubleshooting

- Check CloudWatch logs at `/ecs/my-fargate-task` for container logs
- Ensure your security group allows inbound traffic on port 80
- Verify that your subnets have internet access (public subnets or private with NAT gateway)
- Confirm that the ECR image was pushed successfully