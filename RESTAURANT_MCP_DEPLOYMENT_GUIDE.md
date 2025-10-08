# Restaurant Booking MCP Server - AWS Deployment Guide

## Overview

This guide provides step-by-step instructions for deploying a Restaurant Booking Model Context Protocol (MCP) server on AWS using AWS CDK. The deployment creates a secure, scalable infrastructure with both ECS Fargate and Lambda deployment options.

## Architecture

The deployment includes:
- **VPC Stack**: Virtual Private Cloud with public/private subnets
- **Security Stack**: Cognito User Pool, WAF rules, and IAM roles
- **CloudFront-WAF Stack**: Global content delivery with security
- **MCP Server Stack**: ECS Fargate and Lambda MCP servers

## Prerequisites

### Required Tools
1. **AWS CLI** (v2.0+)
2. **Node.js** (v14+)
3. **AWS CDK** (v2.0+)
4. **Docker** (for container builds)

### AWS Account Setup
1. **AWS Account** with appropriate permissions
2. **AWS CLI configured** with credentials
3. **CDK Bootstrap** completed

## Step-by-Step Deployment

### 1. Environment Setup

```bash
# Install AWS CDK globally
npm install -g aws-cdk

# Verify CDK installation
cdk --version

# Bootstrap CDK (if not already done)
cdk bootstrap
```

### 2. Clone and Prepare Repository

```bash
# Clone the repository
git clone <repository-url>
cd guidance-for-deploying-model-context-protocol-servers-on-aws/source/cdk/ecs-and-lambda

# Install dependencies
npm install
```

### 3. Login to ECR

```bash
# Login to public ECR for base images
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws
```

### 4. Deploy Infrastructure

#### Option A: Deploy All Stacks (Recommended)

```bash
# Deploy all stacks with automatic approval
npx cdk deploy --all --require-approval never
```

#### Option B: Deploy Stacks Individually

```bash
# 1. Deploy VPC Stack
npx cdk deploy MCP-VPC --require-approval never

# 2. Deploy Security Stack
npx cdk deploy MCP-Security --require-approval never

# 3. Deploy CloudFront-WAF Stack
npx cdk deploy MCP-CloudFront-WAF --require-approval never

# 4. Deploy MCP Server Stack
npx cdk deploy MCP-Server --require-approval never
```

### 5. Verify Deployment

#### Check Stack Status

```bash
# List all MCP stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE \
  --query 'StackSummaries[?contains(StackName, `MCP`)].{Name:StackName,Status:StackStatus}' \
  --output table
```

#### Get Deployment Outputs

```bash
# Get CloudFront URL
aws cloudformation describe-stacks \
  --stack-name MCP-Server \
  --query 'Stacks[0].Outputs[?OutputKey==`CloudFrontDistributions`].OutputValue' \
  --output text

# Get Cognito User Pool ID
aws cloudformation describe-stacks \
  --stack-name MCP-Security \
  --query 'Stacks[0].Outputs[?OutputKey==`UserPoolId`].OutputValue' \
  --output text

# Get Cognito Client ID
aws cloudformation describe-stacks \
  --stack-name MCP-Security \
  --query 'Stacks[0].Outputs[?OutputKey==`UserPoolClientId`].OutputValue' \
  --output text
```

### 6. Test Endpoints

#### Health Check

```bash
# Get CloudFront URL
CLOUDFRONT_URL=$(aws cloudformation describe-stacks \
  --stack-name MCP-Server \
  --query 'Stacks[0].Outputs[?OutputKey==`CloudFrontDistributions`].OutputValue' \
  --output text)

# Test ECS endpoint
curl -X GET "https://$CLOUDFRONT_URL/restaurant-booking/"

# Test Lambda endpoint
curl -X GET "https://$CLOUDFRONT_URL/restaurant-booking-lambda/"
```

#### OAuth Metadata Endpoints

```bash
# Test OAuth metadata for ECS
curl -X GET "https://$CLOUDFRONT_URL/restaurant-booking/.well-known/oauth-protected-resource"

# Test OAuth metadata for Lambda
curl -X GET "https://$CLOUDFRONT_URL/restaurant-booking-lambda/.well-known/oauth-protected-resource"
```

## MCP Server Tools

The deployed MCP server provides the following tools:

### 1. get_restaurants
- **Description**: Get list of available restaurants
- **Parameters**: None
- **Example**: Returns JSON list of restaurants from FreeTable API

### 2. create_booking
- **Description**: Create a restaurant booking
- **Parameters**:
  - `restaurantId` (number): ID of the restaurant
  - `tableId` (number): ID of the table
  - `customerName` (string): Customer's full name
  - `customerEmail` (string): Customer's email address
  - `customerPhone` (string): Customer's phone number
  - `bookingDate` (string): Booking date in YYYY-MM-DD format
  - `bookingTime` (string): Booking time in HH:MM format (24-hour)
  - `partySize` (number): Number of people in the party
  - `specialRequests` (string, optional): Any special requests

### 3. update_booking
- **Description**: Update an existing restaurant booking
- **Parameters**:
  - `bookingId` (number): ID of the booking to update
  - `customerName` (string, optional): Updated customer name
  - `customerEmail` (string, optional): Updated customer email
  - `bookingDate` (string, optional): New booking date
  - `bookingTime` (string, optional): New booking time
  - `partySize` (number, optional): New party size
  - `specialRequests` (string, optional): Updated special requests
  - `tableId` (number, optional): New table ID

## Configuration

### Environment Variables

The MCP servers use the following environment variables:

```bash
# AWS Region
AWS_REGION=us-east-1

# Cognito Configuration
COGNITO_USER_POOL_ID=<from CloudFormation output>
COGNITO_USER_POOL_CLIENT_ID=<from CloudFormation output>

# Base URL (automatically set by ALB/CloudFront)
BASE_URL=https://<cloudfront-domain>
```

### FreeTable API Integration

The servers integrate with the FreeTable API:
- **Base URL**: `https://free-table.gyurmatag.workers.dev`
- **Endpoints**:
  - `GET /api/restaurants` - List restaurants
  - `POST /api/bookings` - Create booking
  - `PUT /api/bookings/{id}` - Update booking

## Monitoring and Logs

### CloudWatch Logs

```bash
# View ECS logs
aws logs describe-log-groups --log-group-name-prefix "/aws/ecs/WeatherNodeJsServer"

# View Lambda logs
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/WeatherNodeJsLambda"
```

### ECS Service Status

```bash
# Check ECS service status
aws ecs describe-services \
  --cluster MCPCluster \
  --services WeatherNodeJsServer-WeatherNodeJsService
```

### Lambda Function Status

```bash
# Check Lambda function status
aws lambda get-function --function-name WeatherNodeJsLambda
```

## Security Features

### 1. VPC Configuration
- Private subnets for ECS tasks
- NAT Gateway for outbound internet access
- Security groups with minimal required access

### 2. WAF Protection
- CloudFront WAF for DDoS protection
- Regional WAF for ALB protection
- Rate limiting and bot protection

### 3. Cognito Authentication
- OAuth 2.0 Protected Resource Metadata (RFC9728)
- JWT token validation
- User pool management

### 4. IAM Roles
- Least privilege access
- Task execution roles
- Service roles for Lambda

## Troubleshooting

### Common Issues

#### 1. CDK Bootstrap Required
```bash
# If you get bootstrap error
cdk bootstrap aws://ACCOUNT-NUMBER/REGION
```

#### 2. Docker Login Issues
```bash
# Re-login to ECR
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws
```

#### 3. CloudFormation Stack Rollback
```bash
# Check stack events
aws cloudformation describe-stack-events --stack-name MCP-Server

# Delete failed stack if needed
aws cloudformation delete-stack --stack-name MCP-Server
```

#### 4. ECS Service Not Starting
```bash
# Check ECS service events
aws ecs describe-services \
  --cluster MCPCluster \
  --services WeatherNodeJsServer-WeatherNodeJsService \
  --query 'services[0].events'
```

### Debug Commands

```bash
# Check all resources in a stack
aws cloudformation list-stack-resources --stack-name MCP-Server

# Get detailed stack information
aws cloudformation describe-stacks --stack-name MCP-Server

# Check CloudFront distribution status
aws cloudfront get-distribution --id <distribution-id>
```

## Cost Optimization

### Estimated Monthly Costs (US East 1)
- **VPC (NAT Gateway)**: ~$37.35
- **Application Load Balancer**: ~$16.83
- **CloudFront**: ~$87.96
- **WAF**: ~$10.00
- **ECS Fargate**: ~$36.04
- **Lambda**: ~$0.20
- **Total**: ~$194.18/month

### Cost Reduction Tips
1. **Stop ECS service** when not in use
2. **Use Lambda only** for low-traffic scenarios
3. **Implement CloudWatch alarms** for cost monitoring
4. **Use reserved capacity** for steady workloads

## Cleanup

### Complete Cleanup

```bash
# Destroy all stacks
npx cdk destroy --all --force

# Or destroy individually
npx cdk destroy MCP-Server --force
npx cdk destroy MCP-CloudFront-WAF --force
npx cdk destroy MCP-Security --force
npx cdk destroy MCP-VPC --force
```

### Manual Cleanup (if CDK destroy fails)

```bash
# Delete CloudFormation stacks
aws cloudformation delete-stack --stack-name MCP-Server
aws cloudformation delete-stack --stack-name MCP-CloudFront-WAF
aws cloudformation delete-stack --stack-name MCP-Security
aws cloudformation delete-stack --stack-name MCP-VPC

# Empty S3 buckets
aws s3 rm s3://<bucket-name> --recursive
aws s3api delete-bucket --bucket <bucket-name>

# Delete Cognito User Pool
aws cognito-idp delete-user-pool --user-pool-id <user-pool-id>
```

## Support and Documentation

### Additional Resources
- [AWS CDK Documentation](https://docs.aws.amazon.com/cdk/)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)

### Getting Help
1. Check CloudFormation events for deployment issues
2. Review CloudWatch logs for runtime errors
3. Verify IAM permissions and security group rules
4. Test endpoints individually to isolate issues

---

**Note**: This deployment creates resources that incur AWS charges. Monitor your usage and clean up resources when not needed.
