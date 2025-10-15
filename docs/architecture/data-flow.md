# Data Flow Documentation

## Table of Contents

- [Overview](#overview)
- [User Request Flow](#user-request-flow)
- [Authentication Flow](#authentication-flow)
- [Blockchain Transaction Flow](#blockchain-transaction-flow)
- [Data Persistence Flow](#data-persistence-flow)
- [Caching Strategy](#caching-strategy)
- [Event-Driven Architecture](#event-driven-architecture)
- [Deployment Pipeline Flow](#deployment-pipeline-flow)

## Overview

This document describes how data flows through the Blockchain DApp Platform, from user interactions through the various layers of the application, infrastructure, and back to the user.

### Key Data Flow Patterns

1. **Synchronous Request-Response**: Web/Mobile → API → Database → Response
2. **Cached Responses**: API → Redis → Response (cache hit)
3. **Blockchain Interactions**: User → API → Blockchain Network → Confirmation
4. **Static Asset Delivery**: User → CloudFront → S3 → Content
5. **Event-Driven Processing**: Event → Queue → Worker → Processing

## User Request Flow

### Web Application Request Flow

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant CloudFront
    participant S3
    participant ALB
    participant API
    participant Redis
    participant RDS
    
    User->>Browser: Navigate to app.example.com
    Browser->>CloudFront: Request HTML/JS/CSS
    CloudFront->>CloudFront: Check edge cache
    
    alt Cache Hit
        CloudFront-->>Browser: Return cached content
    else Cache Miss
        CloudFront->>S3: Request from origin
        S3-->>CloudFront: Return content
        CloudFront-->>Browser: Return content + cache
    end
    
    Browser->>Browser: Render application
    Browser->>ALB: API Request (e.g., GET /api/transactions)
    ALB->>API: Forward request
    
    API->>Redis: Check cache
    
    alt Cache Hit
        Redis-->>API: Return cached data
        API-->>Browser: JSON Response
    else Cache Miss
        API->>RDS: Query database
        RDS-->>API: Return data
        API->>Redis: Store in cache
        API-->>Browser: JSON Response
    end
    
    Browser->>User: Display data
```

**Flow Steps**:

1. **User Request**: User navigates to web application URL
2. **DNS Resolution**: Domain resolves to CloudFront distribution
3. **Edge Caching**: CloudFront checks edge location cache
   - **Cache Hit**: Content served from edge location (fastest)
   - **Cache Miss**: Content fetched from S3, cached at edge
4. **Static Content Delivery**: HTML, JavaScript, CSS, images delivered
5. **App Initialization**: React app initializes in browser
6. **API Calls**: App makes API requests to backend
7. **Load Balancing**: ALB distributes requests across API pods
8. **Cache Check**: API checks Redis for cached response
9. **Database Query**: If cache miss, query PostgreSQL database
10. **Response Caching**: Fresh data cached in Redis with TTL
11. **Response**: JSON data returned to client
12. **Rendering**: Browser renders updated UI

### Mobile Application Request Flow

```mermaid
sequenceDiagram
    participant User
    participant MobileApp
    participant ALB
    participant API
    participant Redis
    participant RDS
    
    User->>MobileApp: Launch app
    MobileApp->>MobileApp: Load from local storage
    MobileApp->>ALB: API Request (with auth token)
    ALB->>API: Forward request
    API->>API: Validate JWT token
    
    alt Valid Token
        API->>Redis: Check cache
        alt Cache Hit
            Redis-->>API: Cached data
        else Cache Miss
            API->>RDS: Query database
            RDS-->>API: Fresh data
            API->>Redis: Cache response
        end
        API-->>MobileApp: 200 OK + Data
    else Invalid Token
        API-->>MobileApp: 401 Unauthorized
        MobileApp->>User: Redirect to login
    end
    
    MobileApp->>User: Display data
```

**Key Differences from Web**:
- No CloudFront/S3 layer (app bundle shipped with install)
- Always includes authentication token in requests
- May use local database (SQLite) for offline support
- Background sync when network available

## Authentication Flow

### User Login Flow

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant API
    participant RDS
    participant Redis
    
    User->>Client: Enter credentials
    Client->>API: POST /api/auth/login {email, password}
    API->>API: Hash password
    API->>RDS: Query user by email
    RDS-->>API: User record
    
    alt User Found & Password Match
        API->>API: Generate JWT token
        API->>Redis: Store session (token -> user_id)
        API-->>Client: 200 OK {token, user}
        Client->>Client: Store token (localStorage/secure storage)
        Client->>User: Redirect to dashboard
    else Invalid Credentials
        API-->>Client: 401 Unauthorized {error}
        Client->>User: Display error
    end
```

**Security Considerations**:
- Passwords hashed with bcrypt/argon2 (never stored in plaintext)
- JWT tokens signed with secret key
- Tokens have expiration time (e.g., 24 hours)
- Refresh token mechanism for long-lived sessions
- Session data cached in Redis for fast validation

### Authenticated Request Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Redis
    participant RDS
    
    Client->>API: GET /api/user/profile (Authorization: Bearer <token>)
    API->>API: Extract JWT from header
    API->>API: Verify JWT signature
    
    alt Valid JWT
        API->>Redis: Check session (token)
        alt Session Found
            Redis-->>API: user_id
            API->>RDS: Query user profile
            RDS-->>API: User data
            API-->>Client: 200 OK {profile}
        else Session Expired
            API-->>Client: 401 Unauthorized
        end
    else Invalid JWT
        API-->>Client: 401 Unauthorized
    end
```

## Blockchain Transaction Flow

### Transaction Submission Flow

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant API
    participant BlockchainNode
    participant RDS
    participant Queue
    participant Worker
    
    User->>Client: Initiate transaction
    Client->>API: POST /api/blockchain/transaction {from, to, amount}
    API->>API: Validate request
    API->>RDS: Create pending transaction record
    RDS-->>API: transaction_id
    
    API->>BlockchainNode: Submit transaction to network
    BlockchainNode-->>API: transaction_hash
    
    API->>RDS: Update transaction with hash
    API->>Queue: Enqueue transaction_id for monitoring
    API-->>Client: 202 Accepted {transaction_id, hash}
    Client->>User: Show pending status
    
    loop Monitor Transaction
        Worker->>Queue: Dequeue transaction_id
        Worker->>BlockchainNode: Check transaction status
        BlockchainNode-->>Worker: status (pending/confirmed/failed)
        Worker->>RDS: Update transaction status
        
        alt Confirmed
            Worker->>Client: WebSocket notification
            Client->>User: Show success
        else Failed
            Worker->>Client: WebSocket notification
            Client->>User: Show error
        end
    end
```

**Key Components**:

1. **Transaction Validation**: API validates transaction parameters
2. **Database Record**: Transaction saved as "pending" immediately
3. **Blockchain Submission**: Transaction submitted to blockchain network
4. **Async Monitoring**: Worker polls blockchain for confirmation
5. **Status Updates**: Database updated as transaction progresses
6. **User Notification**: Real-time updates via WebSocket or polling

### Transaction Confirmation Flow

**States**:
1. **Pending**: Submitted to network, awaiting confirmation
2. **Confirmed**: Included in block, waiting for confirmations
3. **Finalized**: Required confirmations met (e.g., 6 blocks)
4. **Failed**: Transaction rejected or expired

## Data Persistence Flow

### Write Operation Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant RDS
    participant Redis
    
    Client->>API: POST /api/resources {data}
    API->>API: Validate data
    API->>RDS: BEGIN TRANSACTION
    API->>RDS: INSERT INTO resources
    RDS-->>API: resource_id
    API->>RDS: COMMIT TRANSACTION
    
    alt Commit Success
        API->>Redis: Invalidate cache (resource_list)
        API->>Redis: Set cache (resource:{id}, data)
        API-->>Client: 201 Created {resource}
    else Commit Failure
        API->>RDS: ROLLBACK
        API-->>Client: 500 Internal Server Error
    end
```

**Database Best Practices**:
- Use transactions for data consistency
- Invalidate related caches on writes
- Set cache for newly created resource
- Return created resource to client

### Read Operation Flow (Cache-Aside Pattern)

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Redis
    participant RDS
    
    Client->>API: GET /api/resource/{id}
    API->>Redis: GET resource:{id}
    
    alt Cache Hit
        Redis-->>API: Cached data
        API-->>Client: 200 OK {resource}
    else Cache Miss
        Redis-->>API: null
        API->>RDS: SELECT * FROM resources WHERE id = ?
        RDS-->>API: Resource data
        
        alt Resource Found
            API->>Redis: SET resource:{id} {data} EX 3600
            API-->>Client: 200 OK {resource}
        else Not Found
            API-->>Client: 404 Not Found
        end
    end
```

## Caching Strategy

### Cache Levels

```mermaid
graph TB
    User[User]
    Browser[Browser Cache]
    CDN[CloudFront Edge Cache]
    AppCache[Application Memory Cache]
    Redis[Redis Cache]
    Database[(PostgreSQL)]
    
    User --> Browser
    Browser --> CDN
    CDN --> AppCache
    AppCache --> Redis
    Redis --> Database
    
    style Browser fill:#e1f5ff
    style CDN fill:#e1f5ff
    style AppCache fill:#fff4e1
    style Redis fill:#fff4e1
    style Database fill:#ffe1e1
```

### Cache TTL Strategy

| Data Type | TTL | Invalidation Strategy |
|-----------|-----|----------------------|
| **Static Assets** (JS/CSS) | 1 year | Cache busting with file hash |
| **HTML** | 5 minutes | CloudFront invalidation on deploy |
| **API List Endpoints** | 5 minutes | Invalidate on CREATE/UPDATE/DELETE |
| **API Detail Endpoints** | 1 hour | Invalidate on UPDATE/DELETE |
| **User Sessions** | 24 hours | Invalidate on logout |
| **Blockchain Data** | 1 minute | Invalidate on new block |

### Cache Invalidation Patterns

**1. Time-based (TTL)**:
```
SET resource:123 {data} EX 3600  # Expires in 1 hour
```

**2. Event-based**:
```
# On resource update
DEL resource:123
DEL resource_list:page:1
```

**3. Tag-based** (Future):
```
# Tag resources
SET resource:123 {data}
SADD tag:user:456 resource:123

# Invalidate all resources for user
SMEMBERS tag:user:456  # Get all keys
DEL resource:123 resource:124...  # Delete all
```

## Event-Driven Architecture

### Message Queue Flow (Future Enhancement)

```mermaid
graph LR
    API[API Server]
    Queue[Message Queue - SQS/RabbitMQ]
    Worker1[Worker 1]
    Worker2[Worker 2]
    Worker3[Worker 3]
    DB[(Database)]
    ExtAPI[External API]
    
    API -->|Publish Event| Queue
    Queue -->|Subscribe| Worker1
    Queue -->|Subscribe| Worker2
    Queue -->|Subscribe| Worker3
    
    Worker1 --> DB
    Worker2 --> ExtAPI
    Worker3 --> DB
```

**Use Cases**:
- Email notifications (async)
- Transaction monitoring
- Report generation
- Blockchain event indexing
- Webhook deliveries

## Deployment Pipeline Flow

### Infrastructure Deployment Flow

```mermaid
graph TB
    Developer[Developer]
    PR[Pull Request]
    TFPlan[Terraform Plan]
    Review[Code Review]
    Merge[Merge to Main]
    Manual[Manual Trigger]
    TFApply[Terraform Apply]
    Global[Global Resources]
    Dev[Dev Environment]
    Staging[Staging Environment]
    Prod[Production Environment]
    
    Developer --> PR
    PR --> TFPlan
    TFPlan --> Review
    Review --> Merge
    Merge --> Manual
    Manual --> TFApply
    TFApply --> Global
    Global --> Dev
    Dev --> Staging
    Staging --> Prod
```

### Application Deployment Flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant CI as GitHub Actions
    participant ECR as Amazon ECR
    participant EKS as Amazon EKS
    participant User as End User
    
    Dev->>GH: Push code to main
    GH->>CI: Trigger workflow
    CI->>CI: npm ci / go test
    CI->>CI: Run tests
    CI->>CI: Security scan (Trivy)
    
    alt Tests Pass
        CI->>CI: Build Docker image
        CI->>ECR: Push image (tag: git-sha)
        CI->>ECR: Scan image for vulnerabilities
        
        alt No Critical Vulnerabilities
            CI->>EKS: kubectl set image deployment/app
            EKS->>EKS: Rolling update
            EKS->>EKS: Health checks
            EKS-->>User: New version available
        else Critical Vulnerabilities Found
            CI->>GH: Fail deployment
            CI->>GH: Report to Security tab
        end
    else Tests Fail
        CI->>GH: Fail workflow
        CI->>Dev: Notification
    end
```

**Key Stages**:
1. **Code Commit**: Developer pushes to repository
2. **Build**: Install dependencies and compile
3. **Test**: Run unit and integration tests
4. **Security Scan**: Trivy scans for vulnerabilities
5. **Containerize**: Build Docker image
6. **Registry**: Push to ECR with git SHA tag
7. **Image Scan**: ECR scans image for CVEs
8. **Deploy**: Update Kubernetes deployment
9. **Rollout**: Rolling update with health checks
10. **Verify**: Post-deployment verification

## Data Backup and Recovery Flow

### Backup Flow

```mermaid
graph LR
    RDS[(RDS Primary)]
    Snapshot[Automated Snapshots]
    S3[S3 Backup Bucket]
    Replica[(RDS Read Replica)]
    
    RDS -->|Daily Snapshot| Snapshot
    RDS -->|Continuous Replication| Replica
    Snapshot -->|Export| S3
```

**Backup Strategy**:
- **Automated Snapshots**: Daily at 3 AM UTC, 7-day retention
- **Manual Snapshots**: Before major changes (deployments, migrations)
- **Read Replicas**: Continuous replication for read scaling and failover
- **Point-in-Time Recovery**: Up to 35 days

### Recovery Flow

**Scenario 1: Data Corruption (Point-in-Time Recovery)**
```mermaid
sequenceDiagram
    participant Ops as Operator
    participant RDS as RDS Service
    participant New as New Instance
    participant App as Application
    
    Ops->>RDS: Identify corruption time
    Ops->>RDS: Restore to point-in-time (T-1 hour)
    RDS->>New: Create new instance from backup
    Ops->>New: Verify data integrity
    Ops->>App: Update connection string
    App->>New: Connect to restored database
```

**Scenario 2: Complete Failure (Snapshot Restore)**
```mermaid
sequenceDiagram
    participant Ops as Operator
    participant Snapshot as Snapshot Store
    participant New as New Instance
    participant App as Application
    
    Ops->>Snapshot: Select latest snapshot
    Ops->>Snapshot: Restore snapshot
    Snapshot->>New: Create new RDS instance
    Ops->>New: Verify data
    Ops->>App: Update DNS/connection
    App->>New: Resume operations
```

## Monitoring Data Flow

### Metrics Collection (Future)

```mermaid
graph TB
    App[Application]
    Metrics[Metrics Endpoint /metrics]
    Prometheus[Prometheus]
    Grafana[Grafana]
    Alert[AlertManager]
    PagerDuty[PagerDuty]
    
    App -->|Expose| Metrics
    Metrics -->|Scrape| Prometheus
    Prometheus -->|Query| Grafana
    Prometheus -->|Trigger| Alert
    Alert -->|Notify| PagerDuty
```

### Log Aggregation (Future)

```mermaid
graph TB
    Pod1[API Pod 1]
    Pod2[API Pod 2]
    Pod3[API Pod 3]
    FluentBit[Fluent Bit]
    CloudWatch[CloudWatch Logs]
    Dashboard[CloudWatch Dashboard]
    
    Pod1 -->|stdout/stderr| FluentBit
    Pod2 -->|stdout/stderr| FluentBit
    Pod3 -->|stdout/stderr| FluentBit
    FluentBit -->|Ship Logs| CloudWatch
    CloudWatch --> Dashboard
```

## Summary

Key data flow patterns in the Blockchain DApp Platform:

1. **Static Content**: CloudFront → S3 (cached at edge)
2. **API Requests**: Client → ALB → API → Cache/DB
3. **Authentication**: JWT-based with Redis session store
4. **Blockchain**: Async submission with worker monitoring
5. **Caching**: Multi-level (browser, CDN, Redis)
6. **Persistence**: Transactional writes, cached reads
7. **Deployment**: Automated CI/CD with security scanning
8. **Backups**: Automated daily snapshots with PITR

For operational procedures, see:
- [Incident Response Runbook](../runbooks/incident-response.md)
- [Rollback Procedure](../runbooks/rollback-procedure.md)
- [Scaling Guide](../runbooks/scaling-guide.md)