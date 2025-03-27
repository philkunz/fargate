# Zebros CRM Data Storage Architecture

This document outlines the data storage architecture for the Zebros CRM system.

## Overview

The architecture uses a multi-tiered storage approach, with different storage solutions optimized for different data patterns.

```mermaid
flowchart TD
    App[Zebros CRM] --> |Structured Data| DynamoDB[(DynamoDB)]
    App --> |Documents & Media| S3[(S3 Buckets)]
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

## CRM Data Models

```mermaid
erDiagram
    CUSTOMER {
        string customerId PK
        string companyName
        string industry
        string tier
        json contactInfo
        timestamp createdAt
        timestamp updatedAt
        string ownerId FK
    }
    
    CONTACT {
        string contactId PK
        string customerId FK
        string firstName
        string lastName
        string email
        string phone
        string position
        json details
        timestamp lastContactedAt
    }
    
    OPPORTUNITY {
        string opportunityId PK
        string customerId FK
        string contactId FK
        string title
        number value
        string stage
        number probability
        timestamp expectedCloseDate
        string ownerId FK
    }
    
    ACTIVITY {
        string activityId PK
        string relatedId FK
        string type
        string subject
        string description
        timestamp scheduledAt
        timestamp completedAt
        string ownerId FK
    }
    
    USER {
        string userId PK
        string username
        string email
        string role
        json permissions
        json preferences
    }
    
    CUSTOMER ||--o{ CONTACT : "has"
    CUSTOMER ||--o{ OPPORTUNITY : "has"
    CONTACT ||--o{ OPPORTUNITY : "involved in"
    CUSTOMER ||--o{ ACTIVITY : "has"
    CONTACT ||--o{ ACTIVITY : "has"
    OPPORTUNITY ||--o{ ACTIVITY : "has"
    USER ||--o{ CUSTOMER : "owns"
    USER ||--o{ OPPORTUNITY : "owns"
    USER ||--o{ ACTIVITY : "owns"
```

## Storage Implementation

### DynamoDB Tables

Primary structured data storage with the following table design:

```mermaid
flowchart TD
    subgraph "Customers Table"
        CPK[PK: customerId]
        CPK --> CData[Customer Data]
    end
    
    subgraph "Contacts Table"
        CTPK[PK: contactId]
        CTSK[SK: customerId]
        CTPK --> CTData[Contact Data]
        CTSK --> CTData
    end
    
    subgraph "Opportunities Table"
        OPK[PK: opportunityId]
        OSK[SK: customerId]
        OPK --> OData[Opportunity Data]
        OSK --> OData
    end
    
    subgraph "Activities Table"
        APK[PK: activityId]
        ASK[SK: relatedId]
        APK --> AData[Activity Data]
        ASK --> AData
    end
    
    subgraph "Access Patterns"
        AP1[Get Customer]
        AP2[List Contacts by Customer]
        AP3[List Opportunities by Stage]
        AP4[List Activities by Date]
        AP5[List Items by Owner]
    end
    
    AP1 --> CPK
    AP2 --> CTSK
    AP3 --> OGSI[GSI: stage-expectedCloseDate]
    AP4 --> AGSI[GSI: scheduledAt-type]
    AP5 --> CGSI[GSI: ownerId-createdAt]
```

### S3 Storage Design

Object storage organized into purpose-specific buckets:

```mermaid
flowchart TD
    subgraph "S3 Storage"
        Documents[Customer Documents]
        Attachments[Email Attachments]
        ImportExport[Import/Export Files]
        Reports[Generated Reports]
        Templates[Email Templates]
        Backups[System Backups]
    end
    
    Documents --> |Organization| DocPrefix[/customers/{customerId}/documents/]
    Attachments --> |Organization| AttPrefix[/customers/{customerId}/attachments/]
    ImportExport --> |Organization| ImpPrefix[/imports/{importId}/]
    Reports --> |Organization| RepPrefix[/reports/{reportId}/]
    
    subgraph "Lifecycle Policies"
        Hot[Hot Storage: Standard]
        Warm[Warm Storage: IA]
        Cold[Cold Storage: Glacier]
    end
    
    Documents --> Hot
    Attachments --> Hot
    ImportExport --> |After 30 days| Warm
    Reports --> |After 60 days| Warm
    Reports --> |After 1 year| Cold
```

### Caching Strategy

```mermaid
flowchart LR
    Client[API Request] --> CacheCheck{Cache Hit?}
    CacheCheck -->|Yes| CacheResponse[Return Cached Result]
    CacheCheck -->|No| DBQuery[Query Database]
    DBQuery --> UpdateCache[Update Cache]
    UpdateCache --> Response[Return Result]
    
    subgraph "CRM Cache Layers"
        L1[Customer List Cache]
        L2[Individual Customer Cache]
        L3[Reports Cache]
        L4[Dashboard Metrics Cache]
    end
```

## CRM-Specific Data Access Patterns

### Common CRM Queries

```mermaid
flowchart TD
    subgraph "Customer Management"
        CM1[Find Customer by ID]
        CM2[Search Customers by Name]
        CM3[Filter Customers by Industry]
        CM4[List Customers by Owner]
    end
    
    subgraph "Sales Pipeline"
        SP1[List Opportunities by Stage]
        SP2[Calculate Pipeline Value]
        SP3[Forecast Close Dates]
        SP4[List Recent Closed Deals]
    end
    
    subgraph "Activity Tracking"
        AT1[Today's Activities by User]
        AT2[Upcoming Activities by Customer]
        AT3[Overdue Activities]
        AT4[Recent Completed Activities]
    end
    
    subgraph "Reporting"
        RP1[Sales by Period]
        RP2[Activity Metrics by User]
        RP3[Customer Acquisition Rate]
        RP4[Lead Conversion Rate]
    end
```

### Read/Write Flow

```mermaid
sequenceDiagram
    participant User as Zebros User
    participant App as Web/Mobile App
    participant API as API Gateway
    participant Lambda as CRM Lambda
    participant DAL as Data Access Layer
    participant Cache as Redis Cache
    participant DB as DynamoDB
    participant S3 as S3 Storage
    
    User->>App: Request Customer Data
    App->>API: API Request
    API->>Lambda: Forward Request
    
    Lambda->>DAL: Get Customer
    DAL->>Cache: Check Cache
    
    alt Cache Hit
        Cache->>DAL: Return Cached Data
    else Cache Miss
        DAL->>DB: Query DynamoDB
        DB->>DAL: Return Results
        DAL->>Cache: Update Cache
    end
    
    DAL->>S3: Get Related Documents
    S3->>DAL: Return Document URLs
    
    DAL->>Lambda: Combined Results
    Lambda->>API: API Response
    API->>App: Customer Data
    App->>User: Display Data
```

## Multi-Tenant Data Isolation

```mermaid
flowchart TD
    Request[API Request] --> |JWT with Tenant ID| Auth[Authorization]
    Auth --> |Tenant Context| DataAccess[Data Access Layer]
    
    subgraph "Data Isolation Strategies"
        Silo[Silo Model]
        Bridge[Bridge Model]
        Pool[Pool Model]
    end
    
    DataAccess --> Silo
    Silo --> |Separate Tables| TenantA[(Tenant A Tables)]
    Silo --> |Separate Tables| TenantB[(Tenant B Tables)]
    
    DataAccess --> Bridge
    Bridge --> |Partition Key Prefix| SharedTables[(Shared Tables)]
    
    DataAccess --> Pool
    Pool --> |Tenant ID Attribute| PooledTables[(Pooled Tables)]
```

## Data Synchronization for Mobile

```mermaid
flowchart TD
    Mobile[Mobile App] --> |Sync Request| API[API Gateway]
    API --> Sync[Sync Lambda]
    
    subgraph "Mobile Data Sync"
        FullSync[Full Synchronization]
        Delta[Delta Synchronization]
        Conflict[Conflict Resolution]
    end
    
    Sync --> FullSync
    Sync --> Delta
    Sync --> Conflict
    
    FullSync --> DB[(DynamoDB)]
    Delta --> ChangeLog[(Change Log)]
    Conflict --> Resolution[Resolution Strategies]
    
    subgraph "Offline Capabilities"
        LocalDB[Local Database]
        QueuedChanges[Queued Changes]
        PendingUploads[Pending Uploads]
    end
```

## Initial Sizing and Scaling

- **DynamoDB**:
  - On-demand capacity mode initially
  - Transition to provisioned capacity with autoscaling when patterns emerge
  - Item size optimization: Keep items under 1KB for core entities

- **S3**:
  - Separate buckets for documents, reports, templates, and backups
  - Multi-part upload configuration for large file attachments
  - Intelligent tiering for cost optimization

- **ElastiCache**:
  - Start with t3.micro node (0.5GB)
  - Scale vertically to larger node types as user base grows
  - Add cluster nodes horizontally for higher throughput

## Data Retention and Compliance

```mermaid
flowchart TD
    Data[CRM Data] --> |Classification| Categories[Data Categories]
    
    subgraph "Retention Policies"
        Business[Business Records]
        Personal[Personal Data]
        System[System Logs]
    end
    
    Categories --> Business
    Categories --> Personal
    Categories --> System
    
    Business --> |7+ Years| LongTerm[Long-term Storage]
    Personal --> |As Needed| GDPR[GDPR Compliance]
    System --> |90 Days| ShortTerm[Short-term Storage]
    
    GDPR --> RightToAccess[Right to Access]
    GDPR --> RightToErase[Right to Erasure]
    GDPR --> DataPortability[Data Portability]
```