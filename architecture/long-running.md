# Long-Running Tasks Architecture

This document outlines the architecture for handling long-running tasks that process large data volumes.

## Overview

Long-running tasks are implemented using a combination of AWS Step Functions for orchestration and AWS Fargate for processing, with Lambda functions for coordination.

```mermaid
flowchart TD
    API[API Gateway] --> |Task Request| TaskLambda[Task Initiator Lambda]
    TaskLambda --> StepF[Step Functions]
    StepF --> |Orchestration| TaskStates[Task State Machine]
    
    subgraph "Task State Machine"
        Init[Initialize] --> |Small Tasks| Lambda[Lambda Function]
        Init --> |Large Tasks| Fargate[Fargate Container]
        Lambda --> |Progress Update| Status[Status Update]
        Fargate --> |Progress Update| Status
        Lambda --> |Complete| Complete[Task Complete]
        Fargate --> |Complete| Complete
        Status --> |Pending| Lambda
        Status --> |Pending| Fargate
        Status --> |Failed| Error[Error Handling]
        Error --> Retry[Retry Logic]
        Retry --> Lambda
        Retry --> Fargate
    end
    
    Complete --> Notification[SNS Notification]
    Notification --> MobileApp[Mobile App]
    Lambda --> |Data Access| S3[(S3 Buckets)]
    Fargate --> |Data Access| S3
    Lambda --> |Metadata| DynamoDB[(DynamoDB)]
    Fargate --> |Metadata| DynamoDB
```

## Task Types and Processing

The system supports different processing patterns based on task requirements:

### Lambda-Based Processing

For smaller workloads that can complete within Lambda's execution limits:

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant API as API Gateway
    participant Lambda as Initiator Lambda
    participant Step as Step Functions
    participant Process as Processing Lambda
    participant DB as DynamoDB
    participant S3 as S3 Storage
    
    App->>API: Request task execution
    API->>Lambda: Forward request
    Lambda->>DB: Create task record (PENDING)
    Lambda->>Step: Start execution
    Step->>Process: Execute processing step
    Process->>S3: Read/write data
    Process->>DB: Update progress
    Process->>Step: Complete step
    Step->>Lambda: Execution complete
    Lambda->>DB: Update task (COMPLETE)
    Lambda->>App: Push notification
```

### Fargate-Based Processing

For larger workloads that require more memory, CPU, or time than Lambda provides:

```mermaid
flowchart TD
    StepF[Step Functions] --> |Start Container| FargateTask[Fargate Task]
    
    subgraph "Fargate Container"
        App[Processing Application]
        Progress[Progress Reporting]
    end
    
    FargateTask --> |Pull Data| S3in[(S3 Input)]
    FargateTask --> |Process| App
    App --> Progress
    Progress --> |Update| DynamoDB[(DynamoDB)]
    App --> |Store Results| S3out[(S3 Output)]
    FargateTask --> |Task Complete| StepF
```

## Task Status Tracking

Each task's status is tracked in DynamoDB with the following flow:

```mermaid
stateDiagram-v2
    [*] --> PENDING: Task Created
    PENDING --> PROCESSING: Execution Started
    PROCESSING --> PROCESSING: Progress Update
    PROCESSING --> COMPLETED: Successful Execution
    PROCESSING --> FAILED: Error Occurred
    FAILED --> RETRY_PENDING: Retry Scheduled
    RETRY_PENDING --> PROCESSING: Retry Started
    FAILED --> [*]: Max Retries Exceeded
    COMPLETED --> [*]: Task Complete
```

## Scaling Considerations

- **Fargate Tasks**: Initially configured with minimal resources
  - CPU: 0.25 vCPU
  - Memory: 0.5 GB 
  - Scale up to 4 vCPU and 8 GB for larger tasks

- **Concurrency**:
  - Initial concurrent task limit: 5
  - Configurable limits per user/tenant
  - Queue mechanism for exceeding concurrency limits

- **Data Handling**:
  - Streaming patterns for large datasets
  - Checkpointing for resumable operations
  - Efficient storage patterns (Parquet, compression)

## Monitoring and Observability

- CloudWatch Logs for all container output
- Custom metrics for task progress
- X-Ray for tracing across components
- Alerting on stuck or failed tasks