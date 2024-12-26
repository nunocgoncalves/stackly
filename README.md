# Stackly System Design

## 1. System Overview

### Architecture Components
```mermaid
graph TB
    subgraph "Control Plane"
        TC[Temporal Cluster]
        CS[Codec Server]
        AS[API Server]
        PC[Provider Config]
    end
    
    subgraph "Storage Layer"
        S3[S3 - State Storage]
        DB[(Database)]
        VC[Vault Cluster]
    end
    
    subgraph "Workers"
        W1[Worker Agent 1]
        W2[Worker Agent 2]
        W3[Worker Agent 3]
    end
    
    subgraph "Clients"
        CLI[CLI]
        UI[Web UI]
        GH[GitHub Webhooks]
    end
    
    CLI --> AS
    UI --> AS
    GH --> AS
    AS --> TC
    TC --> CS
    CS --> VC
    W1 & W2 & W3 --> TC
    W1 & W2 & W3 --> PC
    TC --> S3
    AS --> DB
```

### Core Flows

#### GitHub Integration Flow
```mermaid
sequenceDiagram
    participant GitHub
    participant API
    participant Cache
    participant Worker
    
    GitHub->>API: Push Event
    API->>GitHub: Fetch Changes
    API->>Cache: Update Cache
    
    Worker->>Cache: Request Code
    Cache-->>Worker: Return Code
    
    alt Cache Miss
        Worker->>GitHub: Fetch Direct
        GitHub-->>Worker: Return Code
        Worker->>Cache: Update Cache
    end
```

#### Stack Operation Flow
```mermaid
sequenceDiagram
    participant User
    participant API
    participant Temporal
    participant Worker
    participant S3
    
    User->>API: Request Operation
    API->>Temporal: Schedule Workflow
    Temporal->>Worker: Execute Activities
    Worker->>S3: Manage State
    Worker-->>Temporal: Report Status
    Temporal-->>API: Operation Result
    API-->>User: Operation Complete
```

## 2. Core Components

### API Server

#### Services
- Stack Management (gRPC)
- Worker Registration
- Policy Enforcement
- Provider Configuration CRUD
- GitHub Webhook Handler

#### Provider Configuration
- Configuration validation
- Secure credential storage
- Environment variable mapping
- Provider alias management

### Control Plane

#### Temporal Workflow Engine
- Namespace isolation per account
- Multi-tenancy support
- Worker task routing
- Workflow scheduling

#### Worker Management
```go
type WorkerCapabilities struct {
    Features    []string           // e.g. "terraform", "ansible"
    Labels      map[string]string  // e.g. "environment": "prod"
    Resources   ResourceLimits     // CPU/memory constraints
    Version     string            // Worker version
}
```

#### Codec Server
- Workflow log encryption
- Vault Transit integration
- Key version management

## 3. Stack Model

### Stack Definition
```mermaid
graph TB
    SC[Stack Code] --> |1:N| SS[Stack State]
    SC --> |Contains| TF[Terraform Module]
    SC --> |Contains| Config[Stack Config]
    SS --> |References| State[Terraform State]
    SS --> |Has| Meta[Stack Metadata]
```

### Configuration Schema
```hcl
stack "example" {
  # Stable identifier across repository moves
  name = "network-prod"
  description = "Production network infrastructure"
  
  # Optional path override if different from config location
  source = "./modules/network"
  
  # Pre-configured backend reference
  backend = "prod-state"
  
  # Provider configurations
  providers = {
    aws = ["prod_account", "monitoring_account"]
    gcp = ["default"]
  }
  
  # Stack dependencies (not module dependencies)
  depends_on = ["shared-vpc", "security-groups"]
  
  # Execution policies
  policies = [
    "deployment_window",
    "production_approval"
  ]
  
  # Signal-based hooks
  hooks {
    pre_plan = ["validate_quotas"]
    post_plan = ["cost_check"]
    pre_apply = ["backup_state"]
    post_apply = ["notify_slack"]
    pre_destroy = ["backup_state"]
    post_destroy = ["cleanup_monitoring"]
  }
  
  # Workflow configuration
  workflow {
    timeout = "2h"
    retries = 3
    retry_interval = "5m"
  }
}
```

### Variable Management
```mermaid
flowchart TB
    subgraph "Resolution Order"
        Config[Config Defaults]
        Previous[Previous Deployment]
        Runtime[Runtime Input]
        Signal[Signal Input]
    end
    
    Config --> Resolver
    Previous --> Resolver
    Runtime --> Resolver
    Signal --> |Missing Vars| Resolver
    
    Resolver --> |Encrypt| Store[Store in DB]
    Resolver --> Execute[Execute Workflow]
```

## 4. Workflow Engine

### TerraformDSLWorkflow

#### Components
```mermaid
graph TB
    subgraph "Parser"
        P1[HCL Parser] --> P2[Module Analysis]
        P2 --> P3[Dependency Graph]
    end
    
    subgraph "Scheduler"
        S1[Activity Generator] --> S2[Execution Plan]
        S2 --> S3[Parallel Groups]
        S3 --> S4[Sequential Chains]
    end
    
    subgraph "Activities"
        A1[Plan] --> A2[Apply]
        A2 --> A3[Output Collection]
    end
    
    P3 --> S1
    S4 --> A1
```

#### Activity Management
```go
type Activity struct {
    Name string
    Type string         // "module", "resource", etc.
    Path string         // Module path
    DependsOn []string
    Inputs map[string]interface{}
    Outputs []string
    ExecutionMode string // "sequential", "parallel"
}
```

### Event System

#### Event Flow
```mermaid
stateDiagram-v2
    [*] --> ModuleRegistered: on_push
    ModuleRegistered --> PolicyCheck: pre_workflow
    PolicyCheck --> PlanInitiated: on_plan
    PlanInitiated --> PlanCompleted: post_plan
    PlanCompleted --> ApprovalRequired: on_approval
    ApprovalRequired --> DeployInitiated: on_deploy
    DeployInitiated --> ActivityStarted: pre_activity
    ActivityStarted --> ActivityCompleted: post_activity
    ActivityCompleted --> DeployCompleted: post_deploy
    DeployCompleted --> [*]
```

### Hook System
```mermaid
sequenceDiagram
    participant Workflow
    participant SignalHandler
    participant Hook
    participant Vault
    
    Workflow->>SignalHandler: Hook Event
    SignalHandler->>Vault: Decrypt Previous Vars
    SignalHandler->>Hook: Execute with Vars
    Hook-->>SignalHandler: Response
    SignalHandler->>Vault: Encrypt New Vars
    SignalHandler-->>Workflow: Continue
```

## 5. Security Model

### Authentication & Authorization
```mermaid
sequenceDiagram
    participant User
    participant UI
    participant API
    participant GitHub
    participant TokenService
    
    User->>UI: Login Request
    UI->>GitHub: OAuth2 Redirect
    GitHub-->>UI: Auth Code
    UI->>API: Auth Code
    API->>GitHub: Validate Token
    GitHub-->>API: User Info
    API->>TokenService: Generate JWT
    TokenService-->>UI: Access Token
```

### Secret Management
- Variable encryption with Vault Transit
- Provider credential isolation
- Key rotation policies
- Secure hook variable sharing

### Worker Security
- Dynamic mTLS certificates
- Namespace isolation
- Capability verification
- Environment separation

### State Security
- Encrypted S3 backend
- Access control via backend configs
- State locking mechanism
- Version control

## 6. Implementation Details

### Database Schema
```sql
-- Stack registration
CREATE TABLE stacks (
    id UUID PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    description TEXT,
    repo_url TEXT NOT NULL,
    repo_path TEXT NOT NULL,
    current_commit VARCHAR(40) NOT NULL,
    config JSONB NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Stack deployments
CREATE TABLE stack_deployments (
    id UUID PRIMARY KEY,
    stack_id UUID REFERENCES stacks(id),
    commit_hash VARCHAR(40) NOT NULL,
    tf_state_key TEXT NOT NULL,
    encrypted_vars BYTEA NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Provider configurations
CREATE TABLE provider_configs (
    id UUID PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    provider TEXT NOT NULL,
    config JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Operation history
CREATE TABLE operations (
    id UUID PRIMARY KEY,
    deployment_id UUID REFERENCES stack_deployments(id),
    type TEXT NOT NULL,
    status TEXT NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    error_details JSONB,
    metadata JSONB
);

-- Git cache
CREATE TABLE git_cache (
    repo_url TEXT NOT NULL,
    commit_hash VARCHAR(40) NOT NULL,
    path TEXT NOT NULL,
    content BYTEA NOT NULL,
    cached_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (repo_url, commit_hash, path)
);
```

### Validation Rules
```mermaid
graph LR
    subgraph "Validation Phases"
        A[Syntax Check] --> B[Policy Compliance]
        B --> C[Security Rules]
    end
    
    subgraph "Actions"
        C --> D{Decision}
        D -->|Pass| E[Deploy]
        D -->|Fail| F[Reject]
    end
```
