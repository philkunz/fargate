# Serverless API Architecture

This document outlines a serverless architecture for a mobile app backend that supports authenticated users and long-running tasks with large data volumes.

## Overview

The architecture follows a serverless approach using AWS services, with authentication provided by Keycloak running in a serverless container.

![Architecture Overview](./architecture/overview.png)

```mermaid
flowchart TD
    MobileApp([Mobile App]) --> |HTTPS| APIGateway[API Gateway]
    APIGateway --> |Authentication| Keycloak[Keycloak Auth]
    APIGateway --> |Regular Requests| Lambda[Lambda Functions]
    APIGateway --> |Long-running Tasks| StepFunctions[Step Functions]
    Lambda --> |Data Storage| DynamoDB[(DynamoDB)]
    Lambda --> |Object Storage| S3[(S3 Buckets)]
    StepFunctions --> |Task Orchestration| Fargate[Fargate Containers]
    Fargate --> |Large Data Processing| S3
    Fargate --> |Results Storage| DynamoDB
    Lambda --> |Notifications| SNS[SNS]
    SNS --> |Push Notifications| MobileApp
```

## Key Components

- **API Gateway**: Entry point for all mobile app requests
- **Lambda Functions**: Handle regular, short-running API requests
- **Step Functions**: Orchestrate complex, long-running workflows
- **Fargate**: Run containerized tasks that need more memory/CPU
- **DynamoDB**: NoSQL database for structured data
- **S3**: Object storage for large files and data sets
- **SNS**: Notification service for real-time updates to mobile app
- **Keycloak**: Authentication service running in a Fargate container

See the architecture directory for detailed design documents:

1. [Authentication Flow](./architecture/auth.md)
2. [API Services](./architecture/api.md)
3. [Long-running Tasks](./architecture/long-running.md)
4. [Data Storage](./architecture/data.md)
5. [Scaling Plan](./architecture/scaling.md)