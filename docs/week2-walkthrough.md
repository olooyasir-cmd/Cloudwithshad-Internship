# Week 2 Cloud Deployment Walkthrough

## 1. Project Overview

This document explains how I deployed the Task Tracker application using AWS ECS and supporting infrastructure in the `eu-west-1` (Ireland) region.

The deployment uses Amazon ECS with AWS Fargate to run the application, an Application Load Balancer (ALB) to receive external HTTP traffic, Amazon ECR to store the Docker image, and Amazon CloudWatch to collect application logs.

The main objective was to deploy a containerized application in a way that supports reliable access, monitoring, and controlled network communication.

## 2. AWS Resources Created

The following resources were created for the deployment.

| Resource | Name | Purpose |
|---|---|---|
| ECS Cluster | `group3b-cluster` | Provides the logical environment for running containerized applications |
| ECS Task Definition | `group3b-task-tracker` | Defines how the application container should run |
| CloudWatch Log Group | `/ecs/group3b` | Stores application logs for monitoring and troubleshooting |
| Target Group | `group3b-tg` | Routes traffic from the load balancer to healthy application tasks |
| Application Load Balancer | `group3b-alb` | Receives incoming HTTP traffic and forwards it to the application |
| Application Security Group | `group3b-app` | Controls network traffic reaching the ECS application |
| Load Balancer Security Group | ALB security group | Controls incoming and outgoing traffic associated with the load balancer |
| ECS Service | `group3b-svc` | Maintains the desired number of running application tasks |

### 2.1 ECS Cluster

The ECS cluster was named `group3b-cluster`.

It provides the environment in which the application tasks run. AWS Fargate was selected so that I would not need to provision or manage EC2 instances.

### 2.2 ECS Task Definition

The task definition family was named `group3b-task-tracker`.

The application container was named `web`.

Important configuration values included:

- Launch type: AWS Fargate
- CPU: 256 units
- Memory: 512 MiB
- Container port: 8080
- Network mode: `awsvpc`
- CloudWatch log group: `/ecs/group3b`

The task definition specifies the container image, resource allocation, networking, environment configuration, and logging settings.

During development, I created multiple revisions of the task definition, including revisions 5 and 6, to accommodate configuration changes.

### 2.3 CloudWatch Log Group

The CloudWatch log group was named `/ecs/group3b`.

It collects application output from the ECS container. This was particularly useful when investigating application startup problems and deployment failures.

### 2.4 Target Group

The target group was named `group3b-tg`.

It receives traffic forwarded by the Application Load Balancer and directs it to the ECS application tasks.

Important configuration:

- Target type: IP
- Application port: 8080
- Health-check path: `/health`
- Health-check protocol: HTTP

The health check allows the load balancer to determine whether an application task is healthy enough to receive traffic.

### 2.5 Application Load Balancer

The Application Load Balancer was named `group3b-alb`.

It provides an external entry point for users accessing the Task Tracker application.

The load balancer listens for HTTP traffic and forwards requests to the target group.

Application URL:

http://group3b-alb-934606719.eu-west-1.elb.amazonaws.com

### 2.6 Security Groups

Two security groups were configured to control network access.

**Load Balancer Security Group**

This security group controls traffic entering the Application Load Balancer.

The intended inbound rule allows HTTP traffic on port 80 from the internet (`0.0.0.0/0`).

**Application Security Group**

This security group controls traffic reaching the ECS application tasks.

The inbound rule allows application traffic on port 8080, with the source set to the Application Load Balancer's security group.

This is important because the application should receive traffic through the load balancer rather than directly from arbitrary internet addresses.

### 2.7 ECS Service

The ECS service was named `group3b-svc`.

It manages the application tasks and maintains the desired number of running tasks.

Important settings included:

- Cluster: `group3b-cluster`
- Task definition: `group3b-task-tracker`
- Desired task count: 1
- Launch type: Fargate
- Public IP assignment: Enabled
- Load balancer integration: `group3b-tg`

The service also supports deployment updates when a new task definition revision is registered.

## 3. Important Configuration Decisions

Some configuration fields required additional attention during deployment.

| Configuration | Value | Reason |
|---|---|---|
| AWS Region | `eu-west-1` | Keeps the project's resources in the selected region |
| VPC | `vpc-07c9664e65d9dfb53` | Provides the network environment |
| ECS CPU | 256 | Allocates the selected Fargate CPU capacity |
| ECS Memory | 512 MiB | Provides memory for the application |
| Container name | `web` | Identifies the application container |
| Container port | 8080 | Port used by the application |
| Health-check path | `/health` | Allows application health monitoring |
| Desired task count | 1 | Maintains one running application task |
| Public IP | Enabled | Allows the task to obtain external network connectivity |
| Log group | `/ecs/group3b` | Centralizes application logs |

The application task was deployed in the VPC using the configured subnet and security group.

The task's observed private IP during testing was `172.31.43.213`.

## 4. Security Group Rules and Their Importance

Security groups were configured to provide controlled communication between the internet, load balancer, and application.

### Rule 1: Load Balancer Security Group

- Type: HTTP
- Protocol: TCP
- Port: 80
- Source: `0.0.0.0/0`

This permits incoming HTTP requests from internet clients.

### Rule 2: Application Security Group

- Type: Custom TCP
- Protocol: TCP
- Port: 8080
- Source: Load Balancer Security Group

The application security group references the load balancer's security group instead of using an IP range.

This is a more controlled approach because it allows traffic from resources associated with the approved load balancer security group.

It also avoids depending on fixed IP addresses, which can change when load balancer infrastructure changes.

Consequently, users access the application through the load balancer rather than connecting directly to the ECS task.

## 5. Problems Encountered and How They Were Fixed

This section documents the main challenges encountered during the deployment process.

### 5.1 ECS Service Creation Error

During the initial service creation, I encountered an error indicating that the creation of the service was not idempotent.

This happened while the service creation process was being repeated.

**Resolution:**

I checked the ECS console to determine whether the service had already been created. After confirming its existence, I used the existing service and updated its task definition instead of repeatedly attempting to create another service.

### 5.2 Database Authentication Failure

The application initially experienced database authentication problems when connecting to the RDS PostgreSQL database.

The issue was caused by a mismatch between the configured database credentials.

**Resolution:**

I investigated the database connection configuration and corrected the credentials so that the application could authenticate against the database.

### 5.3 Application Health Check Failure

The Application Load Balancer initially reported unhealthy application targets.

This affected the ability of the load balancer to route traffic successfully to the application.

**Resolution:**

I investigated the application startup process, database connection configuration, container port, and health-check endpoint.

The health-check configuration needed to match the application's actual endpoint and listening port.

### 5.4 ECS Task Definition Revisions

Changes to the application configuration required updates to the ECS task definition.

I initially worked with earlier revisions before progressing to later revisions, including revision 5 and revision 6.

**Resolution:**

I registered updated task definition revisions and configured the ECS service to use the intended revision.

### 5.5 ECR Authentication Token Expiration

When pushing the Docker image to Amazon Elastic Container Registry, Docker returned an error indicating that the authorization token had expired.

**Resolution:**

I reauthenticated Docker using the AWS CLI:

```bash
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 221032951827.dkr.ecr.eu-west-1.amazonaws.com
```

ECR authorization tokens are temporary, so authentication must be repeated when they expire.

### 5.6 IAM Permission Error

The IAM identity used for deployment initially lacked permission to perform `ecr:GetAuthorizationToken`.

**Resolution:**

The required ECR permissions were identified, including authorization token retrieval and the permissions needed to upload image layers.

### 5.7 Invalid AWS Security Token

The AWS CLI subsequently returned `InvalidClientTokenId` and `UnrecognizedClientException`.

This indicated that the configured credentials were not accepted by AWS.

**Resolution:**

The AWS CLI credentials needed to be checked and replaced with valid credentials. The identity was verified using:

```bash
aws sts get-caller-identity
```

### 5.8 ECS Deployment Circuit Breaker

During a later deployment attempt, ECS triggered its deployment circuit breaker because it could not successfully start the application tasks.

The ECS service events showed:

```text
CannotPullContainerError:
failed to resolve ref
task-tracker:latest: not found
```

The issue was that the ECS task definition expected the `latest` image tag, while the intended Docker image was tagged `v1`.

**Resolution:**

The image tags needed to be aligned. The `latest` tag could be created from the existing `v1` image and pushed to ECR, or the ECS task definition could be updated to reference the existing `v1` tag.

This demonstrated the importance of ensuring that the image reference in the ECS task definition matches an image that actually exists in ECR.

## 6. Deployment Verification

The following checks were used to assess the deployment.

**ECS service status**

The ECS service was checked to confirm that its desired and running task counts matched.

**Application Load Balancer**

The load balancer DNS address was used to test access to the application.

**Target Group Health**

The target group's health status was checked to confirm whether the ECS task was passing its health checks.

**CloudWatch Logs**

CloudWatch was used to inspect application output and investigate errors.

**Container Image**

The ECR repository was checked to verify that the required image tag was available.

## 7. Deployment Cost and Runtime

The AWS deployment cost was reviewed using AWS Cost Explorer.

The following information should be completed using the actual billing figures.

| Item | Recorded value |
|---|---|
| AWS Region | Europe (Ireland), `eu-west-1` |
| Deployment start date | [INSERT DATE] |
| Deployment end date | [INSERT DATE] |
| Total runtime | [INSERT DURATION] |
| Total AWS cost | GH₵ [INSERT AMOUNT] |
| Cost currency in Cost Explorer | [INSERT CURRENCY] |
| Billing period | [INSERT PERIOD] |

The Cost Explorer report was used to identify the charges associated with the deployment.

The main services to review include:

- Amazon ECS / AWS Fargate
- Elastic Load Balancing
- Amazon CloudWatch
- Amazon ECR
- Amazon RDS, if included in the deployment cost

The final amount should reflect the actual Cost Explorer figures rather than estimated pricing.

## 8. Lessons Learned

This deployment provided practical experience in container orchestration, cloud networking, access control, monitoring, and application deployment.

Some of the key lessons were:

1. ECS task definitions must reference Docker image tags that exist in ECR.
2. IAM permissions must be configured correctly before attempting deployment operations.
3. ECR authentication tokens expire and require reauthentication.
4. Security groups should restrict communication between infrastructure components.
5. Application health checks must match the application's actual configuration.
6. CloudWatch logs and ECS service events are important troubleshooting tools.
7. Task definition revisions provide a way to manage application configuration changes.
8. Cost Explorer is useful for understanding the financial impact of running cloud infrastructure.

## 9. Conclusion

The Week 2 deployment exercise provided hands-on experience creating and configuring an AWS containerized application environment.

Although several issues occurred during deployment, investigating ECS events, reviewing CloudWatch logs, correcting IAM permissions, and aligning container image configurations helped develop a better understanding of AWS deployment and troubleshooting.

The exercise also demonstrated the importance of security, monitoring, reliability, and cost awareness when deploying applications in the cloud.
