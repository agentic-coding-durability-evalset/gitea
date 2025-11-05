# CI/CD Runners

<cite>
**Referenced Files in This Document**   
- [runner.go](file://models/actions/runner.go)
- [runner_token.go](file://models/actions/runner_token.go)
- [runners.go](file://routers/web/shared/actions/runners.go)
- [admin/runners.go](file://routers/api/v1/admin/runners.go)
- [shared/runners.go](file://routers/api/v1/shared/runners.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Runner Architecture and Data Model](#runner-architecture-and-data-model)
3. [Runner Registration and Token Management](#runner-registration-and-token-management)
4. [Admin Interface for Runner Management](#admin-interface-for-runner-management)
5. [API Endpoints for Runner Operations](#api-endpoints-for-runner-operations)
6. [Runner Status and Monitoring](#runner-status-and-monitoring)
7. [Security and Access Control](#security-and-access-control)
8. [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)
9. [Performance Considerations](#performance-considerations)
10. [Best Practices for CI/CD Infrastructure](#best-practices-for-ci/cd-infrastructure)

## Introduction
Gitea provides comprehensive CI/CD runner management capabilities through its admin interface, enabling administrators to register, monitor, and manage self-hosted runners at various levels of the system hierarchy. This documentation details the implementation of runner management features, focusing on the relationship between controllers, models, and services that handle runner operations. The system supports global, organization/user-level, and repository-level runners, with robust token-based authentication and comprehensive monitoring capabilities.

**Section sources**
- [runner.go](file://models/actions/runner.go#L1-L50)
- [runner_token.go](file://models/actions/runner_token.go#L1-L30)

## Runner Architecture and Data Model

```mermaid
classDiagram
class ActionRunner {
+int64 ID
+string UUID
+string Name
+string Version
+int64 OwnerID
+int64 RepoID
+string Description
+int Base
+string RepoRange
+string TokenHash
+string TokenSalt
+TimeStamp LastOnline
+TimeStamp LastActive
+[]string AgentLabels
+bool Ephemeral
+TimeStamp Created
+TimeStamp Updated
+TimeStamp Deleted
+Status() RunnerStatus
+EditableInContext(ownerID, repoID) bool
+LoadAttributes(ctx) error
+GenerateToken() error
+CanMatchLabels(jobRunsOn) bool
}
class ActionRunnerToken {
+int64 ID
+string Token
+int64 OwnerID
+int64 RepoID
+bool IsActive
+TimeStamp Created
+TimeStamp Updated
+TimeStamp Deleted
+GetLatestRunnerToken(ctx, ownerID, repoID) (*ActionRunnerToken, error)
+NewRunnerToken(ctx, ownerID, repoID) (*ActionRunnerToken, error)
+UpdateRunnerToken(ctx, r, cols) error
}
class FindRunnerOptions {
+ListOptions
+[]int64 IDs
+int64 RepoID
+int64 OwnerID
+string Sort
+string Filter
+Option[bool] IsOnline
+bool WithAvailable
+ToConds() Cond
+ToOrders() string
}
ActionRunner --> FindRunnerOptions : "used in"
ActionRunnerToken --> ActionRunner : "associated with"
```

**Diagram sources**
- [runner.go](file://models/actions/runner.go#L40-L397)
- [runner_token.go](file://models/actions/runner_token.go#L1-L125)

**Section sources**
- [runner.go](file://models/actions/runner.go#L40-L397)
- [runner_token.go](file://models/actions/runner_token.go#L1-L125)

## Runner Registration and Token Management

```mermaid
sequenceDiagram
participant Admin as "Administrator"
participant Web as "Web Interface"
participant Service as "Runner Service"
participant DB as "Database"
Admin->>Web : Request registration token
Web->>Service : GetLatestRunnerToken(ownerID, repoID)
Service->>DB : Query ActionRunnerToken table
DB-->>Service : Return existing token or none
alt Token doesn't exist or inactive
Service->>DB : Create new token with NewRunnerToken
DB-->>Service : Confirm creation
Service->>DB : Invalidate previous tokens
end
Service-->>Web : Return token
Web-->>Admin : Display registration token
Admin->>Runner : Configure runner with token
Runner->>Service : Register with token and UUID
Service->>DB : Validate token and create ActionRunner record
DB-->>Service : Confirm registration
Service-->>Runner : Registration successful
```

**Diagram sources**
- [runner_token.go](file://models/actions/runner_token.go#L70-L120)
- [runners.go](file://routers/web/shared/actions/runners.go#L100-L150)

**Section sources**
- [runner_token.go](file://models/actions/runner_token.go#L70-L120)
- [runners.go](file://routers/web/shared/actions/runners.go#L100-L150)

## Admin Interface for Runner Management

```mermaid
flowchart TD
Start([Admin Interface]) --> LoadContext["Load Runners Context"]
LoadContext --> CheckPageType{"Page Type?"}
CheckPageType --> |Repo Settings| SetRepoContext["Set RepoID, IsRepo=true"]
CheckPageType --> |Org Settings| SetOrgContext["Set OwnerID, IsOrg=true"]
CheckPageType --> |Admin Panel| SetAdminContext["Set IsAdmin=true"]
CheckPageType --> |User Settings| SetUserContext["Set OwnerID, IsUser=true"]
SetRepoContext --> FetchRunners["Fetch Runners with FindRunnerOptions"]
SetOrgContext --> FetchRunners
SetAdminContext --> FetchRunners
SetUserContext --> FetchRunners
FetchRunners --> CheckToken["Check Registration Token"]
CheckToken --> |Token Missing/Inactive| CreateToken["Create New RunnerToken"]
CheckToken --> |Token Valid| UseToken["Use Existing Token"]
CreateToken --> StoreToken["Store in ActionRunnerToken table"]
UseToken --> DisplayRunners["Display Runners List"]
StoreToken --> DisplayRunners
DisplayRunners --> UserAction{"User Action?"}
UserAction --> |Edit Runner| EditRunner["Load Runner Details"]
UserAction --> |Delete Runner| DeleteRunner["Confirm Deletion"]
UserAction --> |Reset Token| ResetToken["Create New Token"]
EditRunner --> UpdateDB["Update ActionRunner record"]
DeleteRunner --> RemoveDB["Delete from ActionRunner table"]
ResetToken --> InvalidateOld["Invalidate previous tokens"]
UpdateDB --> Success["Show Success Message"]
RemoveDB --> Success
InvalidateOld --> Success
Success --> End([Interface Updated])
```

**Diagram sources**
- [runners.go](file://routers/web/shared/actions/runners.go#L50-L360)

**Section sources**
- [runners.go](file://routers/web/shared/actions/runners.go#L50-L360)

## API Endpoints for Runner Operations

```mermaid
classDiagram
class AdminRunnersAPI {
+GetRegistrationToken(ctx) void
+CreateRegistrationToken(ctx) void
+ListRunners(ctx) void
+GetRunner(ctx) void
+DeleteRunner(ctx) void
}
class SharedRunnersAPI {
+GetRegistrationToken(ctx, ownerID, repoID) void
+ListRunners(ctx, ownerID, repoID) void
+GetRunner(ctx, ownerID, repoID, runnerID) void
+DeleteRunner(ctx, ownerID, repoID, runnerID) void
+getRunnerByID(ctx, ownerID, repoID, runnerID) (*ActionRunner, bool)
}
class RepoActionAPI {
+GetRunner(ctx) void
+DeleteRunner(ctx) void
}
class OrgActionAPI {
+GetRunner(ctx) void
+DeleteRunner(ctx) void
}
AdminRunnersAPI --> SharedRunnersAPI : "delegates to"
RepoActionAPI --> SharedRunnersAPI : "delegates to"
OrgActionAPI --> SharedRunnersAPI : "delegates to"
SharedRunnersAPI --> ActionRunner : "operates on"
SharedRunnersAPI --> ActionRunnerToken : "operates on"
```

**Diagram sources**
- [admin/runners.go](file://routers/api/v1/admin/runners.go#L1-L105)
- [shared/runners.go](file://routers/api/v1/shared/runners.go#L1-L107)
- [runner.go](file://models/actions/runner.go#L1-L397)

**Section sources**
- [admin/runners.go](file://routers/api/v1/admin/runners.go#L1-L105)
- [shared/runners.go](file://routers/api/v1/shared/runners.go#L1-L107)

## Runner Status and Monitoring

```mermaid
stateDiagram-v2
[*] --> DetermineStatus
DetermineStatus --> Offline : "LastOnline > 1 minute ago"
DetermineStatus --> Idle : "LastActive > 10 seconds ago"
DetermineStatus --> Active : "Recently active"
state "Runner Status" as RunnerStatus {
[*] --> Offline
Offline --> Idle : "Heartbeat received"
Idle --> Active : "Job started"
Active --> Idle : "Job completed"
Idle --> Offline : "No heartbeat for 1 minute"
Active --> Offline : "No heartbeat for 1 minute"
}
Click "DetermineStatus" as DetermineStatusAction
Click "RunnerStatus" as MonitoringProcess
note right of DetermineStatusAction
Checks LastOnline and LastActive
timestamps against current time
end note
note left of MonitoringProcess
Runners update their status
through periodic heartbeats
end note
```

**Diagram sources**
- [runner.go](file://models/actions/runner.go#L100-L130)

**Section sources**
- [runner.go](file://models/actions/runner.go#L100-L130)

## Security and Access Control

```mermaid
flowchart TD
Request --> CheckContext["Determine Context Type"]
CheckContext --> |Admin| AdminAccess["Allow access to all runners"]
CheckContext --> |Organization| OrgAccess["Allow access to org runners"]
CheckContext --> |Repository| RepoAccess["Allow access to repo runners"]
CheckContext --> |User| UserAccess["Allow access to user runners"]
AdminAccess --> ValidatePermission["Check admin privileges"]
OrgAccess --> ValidatePermission
RepoAccess --> ValidatePermission
UserAccess --> ValidatePermission
ValidatePermission --> CheckOwnership["Verify EditableInContext"]
CheckOwnership --> |True| GrantAccess["Allow operation"]
CheckOwnership --> |False| DenyAccess["Return 404 Not Found"]
GrantAccess --> ExecuteOperation["Perform requested action"]
ExecuteOperation --> UpdateTimestamps["Update LastOnline/LastActive"]
UpdateTimestamps --> SuccessResponse["Return success"]
DenyAccess --> ErrorResponse["Return 404 error"]
style AdminAccess fill:#f9f,stroke:#333
style OrgAccess fill:#f9f,stroke:#333
style RepoAccess fill:#f9f,stroke:#333
style UserAccess fill:#f9f,stroke:#333
```

**Diagram sources**
- [runner.go](file://models/actions/runner.go#L140-L170)
- [shared/runners.go](file://routers/api/v1/shared/runners.go#L74-L106)

**Section sources**
- [runner.go](file://models/actions/runner.go#L140-L170)
- [shared/runners.go](file://routers/api/v1/shared/runners.go#L74-L106)

## Common Issues and Troubleshooting

```mermaid
flowchart LR
Issue --> Connectivity["Runner Connectivity Problems"]
Issue --> Authentication["Authentication Failures"]
Issue --> Resource["Resource Exhaustion"]
Issue --> Registration["Registration Token Issues"]
Connectivity --> |Symptoms| C_Symptoms["No heartbeat, Offline status"]
Connectivity --> |Causes| C_Causes["Network issues, Firewall blocking"]
Connectivity --> |Solutions| C_Solutions["Check network, Open ports 3000"]
Authentication --> |Symptoms| A_Symptoms["401 errors, Registration failed"]
Authentication --> |Causes| A_Causes["Invalid token, Expired token"]
Authentication --> |Solutions| A_Solutions["Reset token, Re-register runner"]
Resource --> |Symptoms| R_Symptoms["Slow jobs, Timeouts"]
Resource --> |Causes| R_Causes["Insufficient CPU/Memory, High load"]
Resource --> |Solutions| R_Solutions["Scale resources, Optimize workflows"]
Registration --> |Symptoms| Reg_Symptoms["Token not found, 404 errors"]
Registration --> |Causes| Reg_Causes["Token invalidated, Permissions"]
Registration --> |Solutions| Reg_Solutions["Generate new token, Check scope"]
C_Symptoms --> Diagnose
A_Symptoms --> Diagnose
R_Symptoms --> Diagnose
Reg_Symptoms --> Diagnose
Diagnose --> Logs["Check server and runner logs"]
Logs --> Identify["Identify root cause"]
Identify --> Resolve["Apply appropriate solution"]
Resolve --> Verify["Verify fix with test job"]
```

**Diagram sources**
- [runner.go](file://models/actions/runner.go#L1-L397)
- [runner_token.go](file://models/actions/runner_token.go#L1-L125)
- [runners.go](file://routers/web/shared/actions/runners.go#L50-L360)

**Section sources**
- [runner.go](file://models/actions/runner.go#L1-L397)
- [runner_token.go](file://models/actions/runner_token.go#L1-L125)

## Performance Considerations

```mermaid
graph TD
Performance --> Scalability["Scalability Challenges"]
Performance --> Database["Database Performance"]
Performance --> API["API Efficiency"]
Performance --> Monitoring["Monitoring Overhead"]
Scalability --> |Large Runner Counts| S_Issues["1000+ runners"]
Scalability --> |Solutions| S_Solutions["Horizontal scaling, Load balancing"]
Database --> |Query Patterns| D_Queires["Indexed queries on LastOnline"]
Database --> |Optimizations| D_Optimize["Use FindRunnerOptions.ToConds"]
Database --> |Bottlenecks| D_Bottlenecks["COUNT queries on large tables"]
API --> |Rate Limiting| A_Rate["Implement rate limiting"]
API --> |Pagination| A_Pagination["Use ListOptions for pagination"]
API --> |Caching| A_Caching["Cache runner lists when possible"]
Monitoring --> |Heartbeat Frequency| M_Heartbeat["Balance between accuracy and load"]
Monitoring --> |Status Updates| M_Updates["Batch updates when feasible"]
S_Issues --> Recommendations
D_Bottlenecks --> Recommendations
A_Rate --> Recommendations
M_Heartbeat --> Recommendations
Recommendations --> R1["Use dedicated runner management instances"]
Recommendations --> R2["Implement connection pooling"]
Recommendations --> R3["Monitor database query performance"]
Recommendations --> R4["Optimize heartbeat intervals"]
Recommendations --> R5["Use read replicas for reporting"]
```

**Diagram sources**
- [runner.go](file://models/actions/runner.go#L300-L397)
- [runner_token.go](file://models/actions/runner_token.go#L1-L125)

**Section sources**
- [runner.go](file://models/actions/runner.go#L300-L397)
- [runner_token.go](file://models/actions/runner_token.go#L1-L125)

## Best Practices for CI/CD Infrastructure

```mermaid
erDiagram
ACTION_RUNNER_TOKEN ||--o{ ACTION_RUNNER : "generates"
ACTION_RUNNER_TOKEN {
int64 id PK
string token UK
int64 owner_id FK
int64 repo_id FK
boolean is_active
timestamp created
timestamp updated
timestamp deleted
}
ACTION_RUNNER ||--o{ ACTION_TASK : "executes"
ACTION_RUNNER {
int64 id PK
string uuid UK
string name
int64 owner_id FK
int64 repo_id FK
string description
string token_hash
string token_salt
timestamp last_online
timestamp last_active
boolean ephemeral
timestamp created
timestamp updated
timestamp deleted
}
ACTION_TASK {
int64 id PK
int64 runner_id FK
int64 repo_id FK
int status
timestamp started
timestamp finished
}
USER ||--o{ ACTION_RUNNER : "owns"
REPOSITORY ||--o{ ACTION_RUNNER : "contains"
USER ||--o{ ACTION_RUNNER_TOKEN : "owns"
REPOSITORY ||--o{ ACTION_RUNNER_TOKEN : "owns"
```

**Diagram sources**
- [runner.go](file://models/actions/runner.go#L1-L397)
- [runner_token.go](file://models/actions/runner_token.go#L1-L125)

**Section sources**
- [runner.go](file://models/actions/runner.go#L1-L397)
- [runner_token.go](file://models/actions/runner_token.go#L1-L125)