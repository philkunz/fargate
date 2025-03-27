# Zebros CRM Long-Running Tasks Architecture

This document outlines the architecture for handling long-running tasks in the Zebros CRM system, including report generation, data imports, and analytics processes.

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
    Notification --> UserApp[Zebros User]
    Lambda --> |Data Access| S3[(S3 Buckets)]
    Fargate --> |Data Access| S3
    Lambda --> |Metadata| DynamoDB[(DynamoDB)]
    Fargate --> |Metadata| DynamoDB
```

## CRM-Specific Long-Running Tasks

The Zebros CRM system includes several types of long-running tasks:

```mermaid
mindmap
  root((Zebros CRM Tasks))
    Reporting
      Sales Performance Reports
      Lead Conversion Analysis
      Revenue Forecasting
      Custom Report Generation
    Data Operations
      Customer Data Import
      Lead List Import
      Product Catalog Updates
      Bulk Data Export
    Communication
      Marketing Campaign Execution
      Bulk Email Sending
      Newsletter Distribution
      SMS Campaigns
    Analytics
      Customer Segmentation
      Sales Pipeline Analysis
      Customer Health Scoring
      Churn Prediction
```

## Task Types and Processing

The system supports different processing patterns based on task requirements:

### Lambda-Based Processing

For smaller CRM workloads that can complete within Lambda's execution limits:

```mermaid
sequenceDiagram
    participant User as Zebros User
    participant API as API Gateway
    participant Lambda as Initiator Lambda
    participant Step as Step Functions
    participant Process as Processing Lambda
    participant DB as DynamoDB
    participant S3 as S3 Storage
    
    User->>API: Request report generation
    API->>Lambda: Forward request
    Lambda->>DB: Create task record (PENDING)
    Lambda->>Step: Start execution
    Step->>Process: Execute report generation
    Process->>S3: Read CRM data
    Process->>DB: Update progress
    Process->>S3: Write report PDF
    Process->>Step: Complete step
    Step->>Lambda: Execution complete
    Lambda->>DB: Update task (COMPLETE)
    Lambda->>User: Push notification
```

### Fargate-Based Processing

For larger CRM workloads that require more memory, CPU, or time than Lambda provides:

```mermaid
flowchart TD
    StepF[Step Functions] --> |Start Container| FargateTask[Fargate Task]
    
    subgraph "Fargate Container"
        App[CRM Processing Application]
        Progress[Progress Reporting]
    end
    
    FargateTask --> |Pull Data| S3in[(S3 Input - CRM Data)]
    FargateTask --> |Process| App
    App --> Progress
    Progress --> |Update| DynamoDB[(DynamoDB)]
    App --> |Store Results| S3out[(S3 Output - Reports)]
    FargateTask --> |Task Complete| StepF
```

## Example: Bulk Customer Import

```mermaid
sequenceDiagram
    participant User as Zebros User
    participant App as Web/Mobile App
    participant API as API Gateway
    participant Lambda as Import Lambda
    participant Step as Step Functions
    participant Fargate as Fargate Container
    participant S3 as S3 Storage
    participant DB as DynamoDB
    
    User->>App: Upload customer list (CSV)
    App->>S3: Upload file
    App->>API: Request import
    API->>Lambda: Start import process
    Lambda->>DB: Create import job record
    Lambda->>Step: Start workflow
    
    Step->>Lambda: Validate file format
    Lambda->>S3: Read sample data
    Lambda->>Step: Validation result
    
    Step->>Fargate: Process import
    Fargate->>S3: Read full file
    Fargate->>Fargate: Transform data
    Fargate->>DB: Begin batch write
    
    loop Processing Batches
        Fargate->>DB: Write customer batch
        Fargate->>DB: Update progress (25%, 50%, 75%)
    end
    
    Fargate->>S3: Write import report
    Fargate->>Step: Processing complete
    
    Step->>Lambda: Finalize import
    Lambda->>DB: Update job status (COMPLETE)
    Lambda->>User: Notify completion
    User->>App: View import results
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

## Task Queue Management

For managing multiple CRM tasks requested by different users:

```mermaid
flowchart TD
    subgraph "Task Prioritization"
        HighPriority[High Priority Queue]
        StandardPriority[Standard Priority Queue]
        LowPriority[Background Queue]
    end
    
    Task[Task Request] --> |Admin User| HighPriority
    Task --> |Standard User| StandardPriority
    Task --> |Scheduled Task| LowPriority
    
    HighPriority --> Scheduler[Task Scheduler]
    StandardPriority --> Scheduler
    LowPriority --> Scheduler
    
    Scheduler --> |Concurrency Control| Executor[Task Executor]
    Executor --> TaskExecution[Task Execution]
```

## Example CRM Reports Implementation

```mermaid
flowchart TD
    ReportAPI[Report API] --> StepF[Step Functions]
    
    subgraph "Sales Performance Report"
        StepF --> CollectData[Collect Sales Data]
        CollectData --> ProcessData[Process Data]
        ProcessData --> GenerateVisuals[Generate Visualizations]
        GenerateVisuals --> FormatReport[Format Report]
        FormatReport --> DeliverReport[Deliver Report]
    end
    
    CollectData --> Lambda1[Lambda: Query Data]
    ProcessData --> Lambda2[Lambda: Data Processing]
    GenerateVisuals --> Fargate1[Fargate: Chart Generation]
    FormatReport --> Fargate2[Fargate: PDF Generation]
    DeliverReport --> Lambda3[Lambda: Notify & Store]
    
    Lambda1 --> DynamoDB[(DynamoDB)]
    Lambda3 --> S3[(S3 - Reports)]
    Lambda3 --> SNS[SNS]
```

## Scaling Considerations

- **Fargate Tasks**: Initially configured with minimal resources
  - CPU: 0.25 vCPU for basic reports
  - Memory: 0.5 GB for standard data processing
  - Scale up to 4 vCPU and 8 GB for large data imports/exports

- **Concurrency**:
  - Initial concurrent task limit: 5 per tenant
  - Premium tier: 20 concurrent tasks
  - Queue mechanism for exceeding concurrency limits

- **Data Handling**:
  - Streaming patterns for large customer datasets
  - Checkpointing for resumable operations
  - Progressive loading for large reports

## Monitoring and Observability

- CloudWatch Logs for all container output
- Custom metrics for CRM task progress
- X-Ray for tracing across components
- Alerting on stuck or failed tasks
- Task history retention for auditing and debugging