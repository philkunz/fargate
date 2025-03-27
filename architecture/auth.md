# Zebros CRM Authentication Architecture

This document describes the authentication flow for the Zebros CRM system using Keycloak in a serverless environment.

## Authentication Flow

```mermaid
sequenceDiagram
    participant User as Zebros User
    participant App as Web/Mobile App
    participant API as API Gateway
    participant KC as Keycloak
    participant OIDC as OIDC Provider (optional)
    
    User->>App: Access CRM
    App->>KC: Request authentication
    KC->>App: Return login form or redirect
    
    alt Social Login
        App->>OIDC: Authenticate with provider
        OIDC->>KC: Return tokens
    else Username/Password
        User->>App: Enter credentials
        App->>KC: Submit credentials
    end
    
    KC->>KC: Validate credentials
    KC->>App: Issue JWT tokens (access + refresh)
    App->>API: CRM request with JWT token
    API->>API: Validate JWT
    API->>App: Return API response
    
    Note over App, API: Token Refresh Flow
    
    App->>KC: Request refresh using refresh token
    KC->>App: Issue new JWT tokens
```

## Role-Based Access Control

```mermaid
flowchart TD
    User[Zebros User] --> |Assigned to| Roles[User Roles]
    
    subgraph "CRM Roles"
        Admin[Administrator]
        Sales[Sales Representative]
        Manager[Sales Manager]
        Support[Support Agent]
        ReadOnly[Read-Only User]
    end
    
    Roles --> Admin
    Roles --> Sales
    Roles --> Manager
    Roles --> Support
    Roles --> ReadOnly
    
    subgraph "Resource Access"
        Customers[Customer Records]
        Leads[Lead Management]
        Opportunities[Sales Opportunities]
        Reports[Reports & Analytics]
        Settings[System Settings]
    end
    
    Admin --> |Full Access| Customers
    Admin --> |Full Access| Leads
    Admin --> |Full Access| Opportunities
    Admin --> |Full Access| Reports
    Admin --> |Full Access| Settings
    
    Sales --> |Read/Write| Customers
    Sales --> |Read/Write| Leads
    Sales --> |Read/Write| Opportunities
    Sales --> |Read| Reports
    
    Manager --> |Read/Write| Customers
    Manager --> |Read/Write| Leads
    Manager --> |Read/Write| Opportunities
    Manager --> |Read/Write| Reports
    Manager --> |Limited| Settings
    
    Support --> |Read/Write| Customers
    Support --> |Read| Leads
    Support --> |Read| Opportunities
    
    ReadOnly --> |Read| Customers
    ReadOnly --> |Read| Leads
    ReadOnly --> |Read| Opportunities
    ReadOnly --> |Read| Reports
```

## Keycloak in Serverless Mode

Keycloak is deployed as a containerized application running on AWS Fargate, allowing for serverless operation.

```mermaid
flowchart LR
    ELB[Application Load Balancer] --> Fargate[Fargate - Keycloak]
    Fargate --> Aurora[(Aurora PostgreSQL)]
    
    subgraph "Keycloak Container"
        KC[Keycloak Server] --> Cache[In-memory Cache]
        KC --> Themes[Zebros Themes]
        KC --> EmailTemplates[Email Templates]
    end
    
    Aurora --> S3[(S3 - Config Backup)]
```

## Key Components

1. **Keycloak Container**:
   - Runs in Fargate for serverless operation
   - Configured for auto-scaling based on demand
   - Optimized for low-memory footprint
   - Customized with Zebros branding

2. **Database**:
   - Aurora Serverless PostgreSQL for user data
   - Auto-scales based on demand
   - Automated backups
   - Encrypted at rest

3. **JWT Validation**:
   - API Gateway validates JWT tokens
   - Public key caching for performance
   - Role-based claims for authorization

4. **Single Sign-On**:
   - Web and mobile app use the same authentication
   - Optional integration with corporate identity providers
   - Support for SAML and OIDC protocols

## Implementation Considerations

- **Cold Start Time**: Minimize by using provisioned concurrency
- **Session Management**: Use client-side session storage when possible
- **Token Lifecycle**: 
  - Short-lived access tokens (1 hour)
  - Longer refresh tokens (7 days) with rotation
- **User Provisioning**: 
  - Self-registration with admin approval workflow
  - Bulk user import for onboarding teams
  - User sync with external systems (optional)

## Multi-Tenant Considerations

For Zebros CRM multi-tenant deployment:

```mermaid
flowchart TD
    subgraph "Multi-Tenant Authentication"
        ALB[Application Load Balancer] --> Keycloak[Keycloak]
        Keycloak --> |Tenant A| RealmA[Realm A]
        Keycloak --> |Tenant B| RealmB[Realm B]
        Keycloak --> |Tenant C| RealmC[Realm C]
    end
    
    subgraph "Tenant Isolation"
        RealmA --> |Users & Roles| TenantADB[(Tenant A Data)]
        RealmB --> |Users & Roles| TenantBDB[(Tenant B Data)]
        RealmC --> |Users & Roles| TenantCDB[(Tenant C Data)]
    end
    
    RealmA --> |JWT with Tenant ID| API[API Gateway]
    RealmB --> |JWT with Tenant ID| API
    RealmC --> |JWT with Tenant ID| API
    
    API --> Lambda[Lambda Functions]
    Lambda --> |Data Isolation| DynamoDB[(DynamoDB)]
```

## Scaling Considerations

- Initial deployment with minimal resources (1 Fargate task)
- Auto-scaling policy based on CPU/memory utilization
- Ability to add read replicas for the database as user count grows
- Caching strategy for frequently accessed authentication data