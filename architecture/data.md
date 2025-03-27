# Data Storage Architecture

This document outlines the data storage architecture for the serverless application.

## Overview

The architecture uses a multi-tiered storage approach, with different storage solutions optimized for different data patterns.

```mermaid
flowchart TD
    App[Application] --> |Structured Data| DynamoDB[(DynamoDB)]
    App --> |Media & Files| S3[(S3 Buckets)]
    App --> |Cache| Redis[(ElastiCache Redis)]
    
    subgraph "Data Access Layer"
        DAL[Data Access Layer]
        ORM[ORM/ODM]
        Cache[Cache Client]
    end
    
    DAL --> ORM
    DAL --> Cache
    ORM --> DynamoDB
    ORM --> S3
    Cache --> Redis
```

## Data Models

### Core Data Models

```mermaid
erDiagram
    USER {
        string userId PK
        string username
        string email
        string status
        timestamp createdAt
        timestamp updatedAt
    }
    
    PROFILE {
        string userId PK
        string fullName
        string avatar
        json preferences
        timestamp lastLogin
    }
    
    TASK {
        string taskId PK
        string userId FK
        string type
        string status
        number progress
        json parameters
        timestamp createdAt
        timestamp updatedAt
        string resultLocation
    }
    
    USER ||--o{ PROFILE : "has"
    USER ||--o{ TASK : "owns"
```

## Storage Implementation

### DynamoDB Tables

Primary structured data storage with the following table design:

```mermaid
flowchart TD
    subgraph "Users Table"
        UPID[PK: userId]
        UPID --> UData[User Data]
    end
    
    subgraph "Tasks Table"
        TPK[PK: taskId]
        TSK[SK: userId]
        TPK --> TData[Task Data]
        TSK --> TData
    end
    
    subgraph "Access Patterns"
        AP1[Get User by ID]
        AP2[Get Tasks by User]
        AP3[Get Task by ID]
        AP4[Query Tasks by Status]
    end
    
    AP1 --> UPID
    AP2 --> TSK
    AP3 --> TPK
    AP4 --> TGI[GSI: status-createdAt]
```

### S3 Storage Design

Object storage organized into purpose-specific buckets:

```mermaid
flowchart TD
    subgraph "S3 Storage"
        UserData[User Data]
        TaskData[Task Data]
        SharedData[Shared Resources]
        Backups[System Backups]
    end
    
    UserData --> |Organization| UserPrefix[/users/{userId}/]
    TaskData --> |Organization| TaskPrefix[/tasks/{taskId}/]
    
    subgraph "Lifecycle Policies"
        Hot[Hot Storage: Standard]
        Warm[Warm Storage: IA]
        Cold[Cold Storage: Glacier]
    end
    
    UserData --> Hot
    TaskData --> Hot
    TaskData --> |After 30 days| Warm
    TaskData --> |After 90 days| Cold
```

### Caching Strategy

```mermaid
flowchart LR
    Client[API Request] --> CacheCheck{Cache Hit?}
    CacheCheck -->|Yes| CacheResponse[Return Cached Result]
    CacheCheck -->|No| DBQuery[Query Database]
    DBQuery --> UpdateCache[Update Cache]
    UpdateCache --> Response[Return Result]
    
    subgraph "Cache Layers"
        L1[API Response Cache]
        L2[Query Result Cache]
        L3[Object Cache]
    end
```

## Data Access Patterns

### Read/Write Flow

```mermaid
sequenceDiagram
    participant Client as API Client
    participant Lambda as Lambda Function
    participant DAL as Data Access Layer
    participant Cache as Redis Cache
    participant DB as DynamoDB
    participant S3 as S3 Storage
    
    Client->>Lambda: Request Data
    Lambda->>DAL: Get Data
    DAL->>Cache: Check Cache
    
    alt Cache Hit
        Cache->>DAL: Return Cached Data
    else Cache Miss
        DAL->>DB: Query DB
        DB->>DAL: Return Results
        DAL->>Cache: Update Cache
    end
    
    alt Media References
        DAL->>S3: Request Media
        S3->>DAL: Return Media URLs
    end
    
    DAL->>Lambda: Combined Results
    Lambda->>Client: API Response
```

## Initial Sizing and Scaling

- **DynamoDB**:
  - On-demand capacity mode initially
  - Transition to provisioned capacity with autoscaling when patterns emerge
  - Item size limit consideration: Keep items under 1KB when possible

- **S3**:
  - Initial buckets sized for 50GB
  - Multi-part upload configuration for large files
  - Intelligent tiering for cost optimization

- **ElastiCache**:
  - Start with t3.micro node (0.5GB)
  - Scale vertically to larger node types as needed
  - Add cluster nodes horizontally for higher throughput