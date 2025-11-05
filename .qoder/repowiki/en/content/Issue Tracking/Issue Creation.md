# Issue Creation

<cite>
**Referenced Files in This Document**   
- [issue.go](file://models/issues/issue.go)
- [issue.go](file://services/issue/issue.go)
- [template.go](file://modules/issue/template/template.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Domain Model](#domain-model)
3. [Service Layer Logic](#service-layer-logic)
4. [Issue Template Processing](#issue-template-processing)
5. [Repository Permissions and Access Control](#repository-permissions-and-access-control)
6. [Integration with Labels and Milestones](#integration-with-labels-and-milestones)
7. [Common Issues and Error Handling](#common-issues-and-error-handling)
8. [Performance Considerations](#performance-considerations)

## Introduction
This document provides a comprehensive analysis of issue creation in Gitea, covering both web interface and API implementations. It details the domain model, service layer logic, template processing, permission system, and integration points for labels and milestones. The documentation also addresses common issues such as permission errors and template parsing failures, along with performance considerations for large repositories.

## Domain Model

The Issue domain model in Gitea represents both issues and pull requests within a repository. The core structure is defined in `models/issues/issue.go` and contains essential fields for tracking and managing issues.

```mermaid
classDiagram
class Issue {
+int64 ID
+int64 RepoID
+int64 Index
+int64 PosterID
+string Title
+string Content
+bool IsClosed
+bool IsPull
+int64 MilestoneID
+int64 AssigneeID
+timeutil.TimeStamp DeadlineUnix
+timeutil.TimeStamp CreatedUnix
+timeutil.TimeStamp UpdatedUnix
+timeutil.TimeStamp ClosedUnix
}
class Label {
+int64 ID
+string Name
+string Color
+string Description
}
class Milestone {
+int64 ID
+string Title
+string Description
+timeutil.TimeStamp Deadline
+timeutil.TimeStamp ClosedDate
}
class User {
+int64 ID
+string Name
+string Email
}
Issue "1" -- "1" Repository : belongs to
Issue "1" -- "1" User : posted by
Issue "1" -- "1" User : assigned to
Issue "1" -- "0..*" Label : has
Issue "1" -- "0..1" Milestone : belongs to
```

**Diagram sources**
- [issue.go](file://models/issues/issue.go#L150-L250)

**Section sources**
- [issue.go](file://models/issues/issue.go#L150-L800)

## Service Layer Logic

The service layer for issue creation is implemented in `services/issue/issue.go` and provides the business logic for creating, updating, and managing issues. The `NewIssue` function serves as the primary entry point for issue creation, handling validation, database operations, and event triggering.

```mermaid
sequenceDiagram
participant User as "User"
participant Service as "Issue Service"
participant Model as "Issue Model"
participant DB as "Database"
participant Notify as "Notification Service"
User->>Service : Create Issue Request
Service->>Service : Validate Poster Permissions
Service->>Model : Create Issue Object
Service->>DB : Begin Transaction
Model->>DB : Insert Issue
DB-->>Model : Issue ID
Model->>DB : Insert Labels
Model->>DB : Insert Assignees
Model->>DB : Insert Project Association
DB-->>Service : Commit Transaction
Service->>Model : Find Mentions
Model-->>Service : Mentioned Users
Service->>Notify : Trigger New Issue Event
Service->>Notify : Trigger Label Change Event
Service->>Notify : Trigger Milestone Change Event
Notify-->>Service : Events Processed
Service-->>User : Issue Created Response
```

**Diagram sources**
- [issue.go](file://services/issue/issue.go#L15-L50)

**Section sources**
- [issue.go](file://services/issue/issue.go#L15-L387)

## Issue Template Processing

Issue templates in Gitea are processed through the `modules/issue/template/template.go` module, which handles validation and rendering of template content. The system supports YAML-based templates with various field types including input, textarea, dropdown, and checkboxes.

```mermaid
flowchart TD
Start([Template Processing]) --> ValidateMetadata["Validate Template Metadata"]
ValidateMetadata --> MetadataValid{"Metadata Valid?"}
MetadataValid --> |No| ReturnMetadataError["Return Validation Error"]
MetadataValid --> |Yes| ValidateFields["Validate Template Fields"]
ValidateFields --> FieldsValid{"Fields Valid?"}
FieldsValid --> |No| ReturnFieldError["Return Field Validation Error"]
FieldsValid --> |Yes| ProcessValues["Process User Input Values"]
ProcessValues --> RenderMarkdown["Render to Markdown"]
RenderMarkdown --> ApplyFormatting["Apply Field Formatting"]
ApplyFormatting --> GenerateContent["Generate Final Issue Content"]
GenerateContent --> End([Return Processed Content])
ReturnMetadataError --> End
ReturnFieldError --> End
```

**Diagram sources**
- [template.go](file://modules/issue/template/template.go#L15-L50)

**Section sources**
- [template.go](file://modules/issue/template/template.go#L15-L487)

## Repository Permissions and Access Control

Issue creation is governed by repository permissions, ensuring that only authorized users can create issues. The permission system checks both direct repository access and user blocking status before allowing issue creation.

```mermaid
graph TD
A[Create Issue Request] --> B{User Authenticated?}
B --> |No| C[Reject Request]
B --> |Yes| D{User Blocked?}
D --> |Yes| E[Reject - User Blocked]
D --> |No| F{Has Repository Access?}
F --> |No| G[Reject - Insufficient Permissions]
F --> |Yes| H{Valid Issue Data?}
H --> |No| I[Reject - Invalid Data]
H --> |Yes| J[Create Issue]
J --> K[Update Repository Stats]
K --> L[Notify Subscribers]
L --> M[Issue Created Successfully]
```

**Diagram sources**
- [issue.go](file://services/issue/issue.go#L20-L35)

## Integration with Labels and Milestones

Issue creation integrates with labeling and milestone assignment systems, allowing for immediate categorization and tracking. The service layer handles the association of labels and milestones during the creation process.

```mermaid
classDiagram
class IssueCreation {
+CreateIssue()
+AssignLabels()
+AssignMilestone()
+AssignProject()
+NotifySubscribers()
}
class LabelService {
+GetLabelsByID()
+ValidateLabelAccess()
+AssignLabelToIssue()
}
class MilestoneService {
+GetMilestoneByID()
+ValidateMilestoneStatus()
+UpdateMilestoneCounters()
}
class ProjectService {
+GetProjectByID()
+ValidateProjectAccess()
+AssignIssueToProject()
}
IssueCreation --> LabelService : uses
IssueCreation --> MilestoneService : uses
IssueCreation --> ProjectService : uses
```

**Diagram sources**
- [issue.go](file://services/issue/issue.go#L30-L45)

## Common Issues and Error Handling

The issue creation system implements comprehensive error handling for common scenarios such as permission errors and template parsing failures. The system provides specific error types to help diagnose and resolve issues.

```mermaid
stateDiagram-v2
[*] --> Idle
Idle --> PermissionCheck : Create Issue
PermissionCheck --> BlockedUser : User Blocked
PermissionCheck --> InsufficientAccess : No Repo Access
PermissionCheck --> Validation : Proceed
Validation --> InvalidTitle : Title Empty
Validation --> InvalidContent : Content Invalid
Validation --> TemplateError : Template Parse Failed
Validation --> Database : Valid Data
Database --> InsertFailed : DB Error
Database --> Success : Issue Created
BlockedUser --> ErrorResponse
InsufficientAccess --> ErrorResponse
InvalidTitle --> ErrorResponse
InvalidContent --> ErrorResponse
TemplateError --> ErrorResponse
InsertFailed --> ErrorResponse
Success --> Notify
Notify --> SuccessResponse
ErrorResponse --> [*]
SuccessResponse --> [*]
```

**Diagram sources**
- [issue.go](file://services/issue/issue.go#L20-L60)
- [issue.go](file://models/issues/issue.go#L20-L50)

## Performance Considerations

Issue creation performance is optimized through transaction management and batch operations, particularly important for large repositories with extensive issue tracking requirements.

```mermaid
graph TB
subgraph "Performance Optimizations"
A[Transaction Management]
B[Batch Database Operations]
C[Lazy Loading of Attributes]
D[Cached Repository Access]
E[Asynchronous Notifications]
end
subgraph "Bottlenecks"
F[Large Template Processing]
G[Multiple Label Assignment]
H[Complex Permission Checks]
I[Notification Overhead]
end
A --> X[Reduced Round Trips]
B --> X
C --> Y[Reduced Memory Usage]
D --> Y
E --> Z[Improved Response Time]
F --> W[Processing Delays]
G --> W
H --> W
I --> W
```

**Diagram sources**
- [issue.go](file://services/issue/issue.go#L25-L40)
- [issue.go](file://models/issues/issue.go#L300-L350)