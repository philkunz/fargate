# Zebros CRM API Services Architecture

This document outlines the API service architecture for the Zebros CRM system.

## API Gateway Structure

```mermaid
flowchart TD
    API[API Gateway] --> |Authentication| Auth[Authorization Lambda]
    API --> |API Requests| Services[Service Endpoints]
    API --> |WebSocket| RealTime[Real-Time Connections]
    
    subgraph "Service Endpoints"
        Contacts[Contact Management]
        Sales[Sales Pipeline]
        Marketing[Marketing]
        Support[Customer Support]
        Reports[Reporting]
        Admin[Admin Operations]
    end
    
    Auth --> Cache[(Auth Cache)]
    Contacts --> Lambda[Lambda Functions]
    Sales --> Lambda
    Marketing --> Lambda
    Support --> Lambda
    Admin --> Lambda
    Reports --> StepF[Step Functions]
    
    RealTime --> WebSocketLambda[WebSocket Lambda]
    WebSocketLambda --> Connections[(Connection Table)]
```

## API Domain Structure

The Zebros CRM API is organized into several domains:

```mermaid
mindmap
  root((Zebros CRM API))
    Contact Management
      Customers API
      Leads API
      Contacts API
      Companies API
    Sales Pipeline
      Opportunities API
      Deals API
      Forecasting API
      Products API
      Pricing API
    Marketing
      Campaigns API
      Email Templates API
      Analytics API
      Marketing Automation API
    Customer Support
      Tickets API
      SLA Management API
      Knowledge Base API
    Reporting
      Dashboard API
      Custom Reports API
      Data Export API
    Admin
      Users API
      Roles API
      Settings API
      Audit Logs API
```

## Implementation Details

### Lambda-based APIs

Standard CRM operations are implemented using Lambda functions with the following structure:

```mermaid
flowchart LR
    APIGw[API Gateway] --> LambdaProxy[Lambda Proxy]
    LambdaProxy --> |Router| Handlers[Request Handlers]
    
    subgraph "Lambda Function"
        LambdaProxy
        Handlers
        Models[Data Models]
        Validation[Request Validation]
        Auth[Authorization]
    end
    
    Handlers --> |Read/Write| DynamoDB[(DynamoDB)]
    Handlers --> |Files| S3[(S3)]
```

### API Request Flow

```mermaid
sequenceDiagram
    participant Client as Zebros User
    participant Gateway as API Gateway
    participant Lambda as Lambda Function
    participant DB as DynamoDB
    
    Client->>Gateway: API Request with JWT
    Gateway->>Gateway: Validate JWT & Role
    Gateway->>Lambda: Forward Request
    Lambda->>Lambda: Validate Request
    Lambda->>Lambda: Authorization Check
    Lambda->>DB: Data Operation
    DB->>Lambda: Operation Result
    Lambda->>Gateway: Response
    Gateway->>Client: HTTP Response
```

## CRM-Specific API Endpoints

### Customer Management

```mermaid
flowchart TD
    Customers[/customers] --> List[GET /customers]
    Customers --> Create[POST /customers]
    Customers --> GetById[GET /customers/{id}]
    Customers --> Update[PUT /customers/{id}]
    Customers --> Delete[DELETE /customers/{id}]
    Customers --> Notes[/customers/{id}/notes]
    Customers --> Activities[/customers/{id}/activities]
    Customers --> Deals[/customers/{id}/deals]
    
    subgraph "Data Model"
        CustomerModel[Customer Model]
        CustomerModel --> BasicInfo[Basic Information]
        CustomerModel --> ContactInfo[Contact Information]
        CustomerModel --> Classification[Classification & Tags]
        CustomerModel --> Relationships[Relationships]
        CustomerModel --> CustomFields[Custom Fields]
    end
```

### Sales Pipeline

```mermaid
flowchart TD
    Opportunities[/opportunities] --> List[GET /opportunities]
    Opportunities --> Create[POST /opportunities]
    Opportunities --> GetById[GET /opportunities/{id}]
    Opportunities --> Update[PUT /opportunities/{id}]
    Opportunities --> Stage[PUT /opportunities/{id}/stage]
    Opportunities --> Delete[DELETE /opportunities/{id}]
    
    subgraph "Pipeline Stages"
        StageModel[Stage Progression]
        StageModel --> Lead[Lead]
        StageModel --> Qualified[Qualified]
        StageModel --> Proposal[Proposal]
        StageModel --> Negotiation[Negotiation]
        StageModel --> Closed[Closed Won/Lost]
    end
```

## WebSocket Support

For real-time updates to CRM users, the architecture includes WebSocket support:

```mermaid
flowchart TD
    Client[Zebros User] <-->|WebSocket| API[API Gateway]
    API <--> WsConnect[Connect Lambda]
    API <--> WsDisconnect[Disconnect Lambda]
    API <--> WsMessage[Message Lambda]
    WsConnect --> ConnTable[(Connections DynamoDB)]
    WsDisconnect --> ConnTable
    WsMessage --> ConnTable
    
    subgraph "Real-Time Events"
        NewLead[New Lead Created]
        DealUpdated[Deal Updated]
        TaskAssigned[Task Assigned]
        MeetingReminder[Meeting Reminder]
    end
    
    NewLead --> Notification[Notification Lambda]
    DealUpdated --> Notification
    TaskAssigned --> Notification
    MeetingReminder --> Notification
    
    Notification --> API
```

## API Gateway Configuration

- Custom domain with TLS (api.zebroscrm.com)
- API key for rate limiting and usage plans
- Throttling configurations to prevent overload
- Request validation using JSON Schema models
- CORS configuration for web clients
- WebSocket API for real-time updates

## Data Synchronization

For mobile app offline functionality:

```mermaid
sequenceDiagram
    participant Mobile as Mobile App
    participant API as API Gateway
    participant Sync as Sync Lambda
    participant DB as DynamoDB
    
    Mobile->>API: GET /sync (If-Modified-Since)
    API->>Sync: Forward Request
    Sync->>DB: Query Changes
    DB->>Sync: Return Changed Records
    Sync->>Mobile: Return Delta
    
    Mobile->>Mobile: Apply Changes Locally
    
    Mobile->>API: POST /sync (Local Changes)
    API->>Sync: Forward Changes
    Sync->>Sync: Conflict Resolution
    Sync->>DB: Apply Changes
    Sync->>Mobile: Confirm Changes
```

## API Versioning

```mermaid
flowchart LR
    Client[Client App] --> |API Version| Gateway[API Gateway]
    
    Gateway --> |/v1/*| V1Lambda[V1 Lambda]
    Gateway --> |/v2/*| V2Lambda[V2 Lambda]
    
    V1Lambda --> |Read/Write| DynamoDB[(DynamoDB)]
    V2Lambda --> |Read/Write| DynamoDB
    
    subgraph "Versioning Strategy"
        URIVersion[URI Path Versioning]
        HeaderVersion[Custom Header Versioning]
        ContentType[Accept Header Versioning]
    end
```

## Monitoring and Logging

- CloudWatch Logs for all API requests with structured logging
- X-Ray for distributed tracing across components
- Custom metrics for business-level monitoring:
  - Leads created per day
  - Opportunities by stage
  - Conversion rates
  - API response times by endpoint

## Initial Sizing and Scaling

- Lambda functions: 128MB RAM, 3 second timeout for standard APIs
- Lambda functions: 1GB RAM, 15 second timeout for data processing APIs
- API Gateway: Default throttling with burst capacity configured
- DynamoDB: On-demand capacity for initial deployment
- Connection handling: Up to 500 concurrent WebSocket connections