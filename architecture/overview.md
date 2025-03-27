# Zebros CRM - Serverless Architecture Overview

This document provides a comprehensive overview of the serverless architecture for the Zebros CRM system.

## High-Level Architecture

```mermaid
flowchart TD
    WebApp([Zebros CRM Web App]) --> |HTTPS| APIGateway[API Gateway]
    MobileApp([Zebros CRM Mobile App]) --> |HTTPS| APIGateway
    
    subgraph "Authentication Layer"
        APIGateway --> |Auth Requests| Keycloak[Keycloak]
        Keycloak --> |User Data| AuthDB[(Aurora PostgreSQL)]
    end
    
    subgraph "API Layer"
        APIGateway --> |Validated Requests| Lambda[Lambda Functions]
        APIGateway --> |WebSocket| WebSocketAPI[WebSocket API]
        APIGateway --> |Long Tasks| StepFunctions[Step Functions]
    end
    
    subgraph "Processing Layer"
        Lambda --> |CRM Operations| Processing[Data Processing]
        StepFunctions --> |Orchestration| LongRunning[Long-running Tasks]
        LongRunning --> Fargate[Fargate Containers]
    end
    
    subgraph "Data Layer"
        Lambda --> |Read/Write| DynamoDB[(DynamoDB)]
        Lambda --> |Documents| S3[(S3 Buckets)]
        Fargate --> |Bulk Data| S3
        Fargate --> |Analytics Results| DynamoDB
    end
    
    subgraph "Communication Layer"
        Lambda --> |Events| EventBridge[EventBridge]
        EventBridge --> SNS[SNS]
        SNS --> |Push Notifications| MobileApp
        SNS --> |Email Notifications| SES[SES]
        SES --> |Customer Emails| Customers([Customers])
        WebSocketAPI --> |Real-time Updates| WebApp
        WebSocketAPI --> |Real-time Updates| MobileApp
    end
```

## Key Design Principles

### 1. Serverless-First Approach

The architecture adopts a serverless-first approach to minimize operational overhead and infrastructure management:

```mermaid
flowchart LR
    Principle[Serverless-First] --> NoServers[No Server Management]
    Principle --> PayPerUse[Pay-per-Use Pricing]
    Principle --> AutoScale[Automatic Scaling]
    Principle --> HighAvail[High Availability]
```

### 2. Small Start, Big Scale

Designed to start with minimal resources but scale to handle growth:

```mermaid
flowchart TD
    subgraph "Initial Deployment"
        MinResources[Minimal Resources]
        SingleInstance[Single Instances]
        OnDemand[On-demand Pricing]
    end
    
    subgraph "Scaled Deployment"
        MultiAZ[Multi-AZ Deployment]
        Redundancy[Component Redundancy]
        AutoScaling[Auto-scaling Policies]
        AdvancedRouting[Advanced Routing]
    end
    
    Initial[Initial State] --> Growth[Gradual Growth] --> Scaled[Scaled State]
    
    Initial --> MinResources
    Initial --> SingleInstance
    Initial --> OnDemand
    
    Scaled --> MultiAZ
    Scaled --> Redundancy
    Scaled --> AutoScaling
    Scaled --> AdvancedRouting
```

### 3. CRM-Specific Features

```mermaid
mindmap
  root((Zebros CRM))
    Contact Management
      Customers
      Leads
      Contacts
    Sales Pipeline
      Opportunities
      Deals
      Forecasting
    Marketing
      Campaigns
      Email Templates
      Analytics
    Customer Service
      Support Tickets
      SLA Management
      Knowledge Base
    Reporting
      Dashboard
      Custom Reports
      Data Exports
    Integrations
      Email
      Calendar
      Third-party APIs
```

### 4. Authentication Architecture

```mermaid
flowchart TD
    Users([Zebros Users]) --> |HTTPS| ALB[Application Load Balancer]
    ALB --> KeycloakService[Keycloak Service]
    
    subgraph "Keycloak on Fargate"
        KeycloakService --> |Clustered| KeycloakTasks[Keycloak Tasks]
        KeycloakTasks --> |Session| Redis[(ElastiCache Redis)]
        KeycloakTasks --> |Data| Aurora[(Aurora PostgreSQL)]
    end
    
    KeycloakService --> |Issue JWT| Users
    Users --> |JWT| APIGateway[API Gateway]
    APIGateway --> |Validate JWT| Lambda[Lambda Functions]
    
    subgraph "Role-Based Access"
        RoleAdmin[Admin Role]
        RoleSales[Sales Role]
        RoleSupport[Support Role]
        RoleReadOnly[Read-Only Role]
    end
    
    KeycloakService --> RoleAdmin
    KeycloakService --> RoleSales
    KeycloakService --> RoleSupport
    KeycloakService --> RoleReadOnly
```

### 5. Long-Running Task Architecture

```mermaid
flowchart TD
    APIGateway[API Gateway] --> |Task Request| InitLambda[Task Initiator Lambda]
    InitLambda --> |Create Record| DynamoDB[(DynamoDB)]
    InitLambda --> |Start Workflow| StepFunctions[Step Functions]
    
    subgraph "CRM Tasks"
        ReportGeneration[Report Generation]
        DataImport[Data Import]
        BulkEmails[Bulk Emails]
        CustomerAnalytics[Customer Analytics]
    end
    
    StepFunctions --> |Small Task| LambdaTask[Lambda Task]
    StepFunctions --> |Large Task| FargateTask[Fargate Task]
    
    LambdaTask --> ReportGeneration
    FargateTask --> DataImport
    LambdaTask --> BulkEmails
    FargateTask --> CustomerAnalytics
    
    LambdaTask --> |Progress| DynamoDB
    FargateTask --> |Progress| DynamoDB
    
    FargateTask --> |Process| LargeData[(S3 Data)]
    
    subgraph "Notification Flow"
        LambdaTask --> |Complete| SNS[SNS]
        FargateTask --> |Complete| SNS
        SNS --> |Notify| Users([Zebros Users])
    end
```

## Component Integration

The following diagram shows how the various components of the architecture integrate:

```mermaid
flowchart TD
    Users([Zebros Users]) --> |Auth Request| Auth[Authentication]
    Users --> |CRM Operations| API[API Gateway]
    Users --> |WebSocket| Realtime[Real-time Updates]
    
    Auth --> |Issue Token| Users
    API --> |Validate| Auth
    API --> Compute[Compute Layer]
    Compute --> Storage[Storage Layer]
    Compute --> Messaging[Messaging Layer]
    
    Messaging --> Realtime
    Messaging --> |Notifications| Users
    Messaging --> |Emails| Customers([Customers])
```

## Technology Choices

```mermaid
mindmap
  root((Zebros CRM))
    Authentication
      Keycloak
      JWT
      OIDC
    Compute
      AWS Lambda
      AWS Fargate
      Step Functions
    Storage
      DynamoDB
      S3
      Aurora Serverless
    API
      API Gateway
      REST
      WebSocket
    Messaging
      SNS
      SES
      EventBridge
    Monitoring
      CloudWatch
      X-Ray
```

## Security Architecture

```mermaid
flowchart TD
    Edge[Edge Security] --> WAF[WAF]
    Edge --> CloudFront[CloudFront]
    
    ApiSecurity[API Security] --> Authorization[JWT Validation]
    ApiSecurity --> RateLimiting[Rate Limiting]
    
    DataSecurity[Data Security] --> Encryption[Encryption at Rest]
    DataSecurity --> Transit[Encryption in Transit]
    DataSecurity --> IAM[IAM Policies]
    
    subgraph "CRM-Specific Security"
        CustomerData[Customer Data Protection]
        DataExports[Export Controls]
        AuditLogging[Audit Logging]
        DataMasking[PII Masking]
    end
    
    NetworkSecurity[Network Security] --> PrivateEndpoints[VPC Endpoints]
    NetworkSecurity --> SecurityGroups[Security Groups]
```

## Deployment and Staging

```mermaid
flowchart LR
    Dev[Development] --> |Automated Tests| Test[Testing]
    Test --> |Approval| Staging[Staging]
    Staging --> |Approval| Prod[Production]
    
    subgraph "Environments"
        Dev
        Test
        Staging
        Prod
    end
    
    subgraph "CI/CD Pipeline"
        Build[Build]
        UnitTest[Unit Tests]
        Deploy[Deploy]
        IntegrationTest[Integration Tests]
    end
    
    Code[Code Changes] --> Build
    Build --> UnitTest
    UnitTest --> Deploy
    Deploy --> IntegrationTest
```

## For Further Details

Please refer to the detailed architecture documents:

1. [Authentication Flow](./auth.md)
2. [API Services](./api.md)
3. [Long-running Tasks](./long-running.md)
4. [Data Storage](./data.md)
5. [Scaling Plan](./scaling.md)
6. [Implementation Plan](./implementation-plan.md)