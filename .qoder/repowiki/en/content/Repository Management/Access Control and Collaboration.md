# Access Control and Collaboration

<cite>
**Referenced Files in This Document**   
- [models/repo/collaboration.go](file://models/repo/collaboration.go)
- [services/repository/collaboration.go](file://services/repository/collaboration.go)
- [routers/web/repo/setting/collaboration.go](file://routers/web/repo/setting/collaboration.go)
- [models/perm/access_mode.go](file://models/perm/access_mode.go)
- [models/unit/unit.go](file://models/unit/unit.go)
- [models/organization/team.go](file://models/organization/team.go)
- [services/org/team.go](file://services/org/team.go)
- [models/perm/access/access.go](file://models/perm/access/access.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Core Domain Models](#core-domain-models)
3. [Permission Levels and Access Modes](#permission-levels-and-access-modes)
4. [Collaboration Management](#collaboration-management)
5. [Team-Based Access Control](#team-based-access-control)
6. [Web Interface and Service Layer Integration](#web-interface-and-service-layer-integration)
7. [Permission Inheritance and Recalculation](#permission-inheritance-and-recalculation)
8. [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)
9. [Performance Considerations](#performance-considerations)
10. [Best Practices for Least Privilege](#best-practices-for-least-privilege)

## Introduction
Gitea implements a comprehensive access control and collaboration system that enables fine-grained permission management for repositories. The system supports both individual user collaborations and team-based access, with multiple permission levels and inheritance mechanisms. This document provides a detailed analysis of the implementation, focusing on the core components that manage repository access, collaboration workflows, and permission inheritance.

**Section sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L1-L178)
- [services/repository/collaboration.go](file://services/repository/collaboration.go#L1-L125)

## Core Domain Models

### Collaboration Model
The Collaboration model represents the relationship between an individual user and a repository, storing the access mode granted to that user. The model includes fields for repository ID, user ID, and access mode, with appropriate database constraints to ensure data integrity.

```mermaid
classDiagram
class Collaboration {
+int64 ID
+int64 RepoID
+int64 UserID
+AccessMode Mode
+TimeStamp CreatedUnix
+TimeStamp UpdatedUnix
}
class Collaborator {
+User User
+Collaboration Collaboration
}
Collaboration --> User : "references"
Collaborator --> User : "contains"
Collaborator --> Collaboration : "contains"
```

**Diagram sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L15-L35)

### Repository Units
Repository units represent different functional components of a repository (code, issues, wiki, etc.) and their associated permission requirements. Each unit has a maximum access mode that determines the highest level of access required to interact with that unit.

```mermaid
classDiagram
class Unit {
+Type Type
+string NameKey
+string URI
+string DescKey
+int Priority
+AccessMode MaxAccessMode
}
class Type {
+TypeInvalid
+TypeCode
+TypeIssues
+TypePullRequests
+TypeReleases
+TypeWiki
+TypeExternalWiki
+TypeExternalTracker
+TypeProjects
+TypePackages
+TypeActions
}
Unit --> Type : "has"
```

**Diagram sources**
- [models/unit/unit.go](file://models/unit/unit.go#L15-L377)

## Permission Levels and Access Modes

### Access Mode Hierarchy
Gitea implements a hierarchical access model with four distinct permission levels, each granting progressively more privileges. The access modes are implemented as an enumeration with corresponding string representations for display purposes.

```mermaid
classDiagram
class AccessMode {
+AccessModeNone
+AccessModeRead
+AccessModeWrite
+AccessModeAdmin
+AccessModeOwner
}
AccessModeNone <|-- AccessModeRead : "extends"
AccessModeRead <|-- AccessModeWrite : "extends"
AccessModeWrite <|-- AccessModeAdmin : "extends"
AccessModeAdmin <|-- AccessModeOwner : "extends"
```

**Diagram sources**
- [models/perm/access_mode.go](file://models/perm/access_mode.go#L15-L65)

### Permission Level Descriptions
The following table outlines the capabilities associated with each permission level:

| Permission Level | Code Access | Issues Access | Pull Requests | Releases | Wiki | Settings Access |
|------------------|-----------|---------------|---------------|----------|------|----------------|
| Read | View | View | View | View | View | No |
| Write | Push | Create/Edit | Create/Merge | Create | Edit | No |
| Admin | Full | Full | Full | Full | Full | Yes |
| Owner | Full | Full | Full | Full | Full | Full |

**Section sources**
- [models/perm/access_mode.go](file://models/perm/access_mode.go#L15-L65)
- [models/unit/unit.go](file://models/unit/unit.go#L15-L377)

## Collaboration Management

### Adding Collaborators
The process of adding a collaborator involves validating the user, checking for existing collaborations, and establishing the appropriate access level. The system prevents adding organization accounts or blocked users as collaborators.

```mermaid
sequenceDiagram
participant Web as Web Interface
participant Service as Repository Service
participant Model as Repository Model
participant DB as Database
Web->>Service : AddOrUpdateCollaborator(repo, user, mode)
Service->>Model : LoadOwner(ctx)
Service->>Model : IsUserBlockedBy(ctx, u, owner)
alt User is blocked
Service-->>Web : Return ErrBlockedUser
else Valid user
Service->>DB : Get Collaboration(repo_id, user_id)
alt Collaboration exists
DB-->>Service : Return collaboration
Service->>DB : Update collaboration mode
else No collaboration
DB-->>Service : Return nil
Service->>DB : Insert new collaboration
end
Service->>Service : RecalculateUserAccess(ctx, repo, u.ID)
Service-->>Web : Success
end
```

**Diagram sources**
- [services/repository/collaboration.go](file://services/repository/collaboration.go#L1-L125)
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L1-L178)

### Changing Access Levels
Modifying a collaborator's access level involves updating the collaboration record and recalculating the user's effective permissions. The system validates the new access mode and updates both the collaboration and access tables.

```mermaid
flowchart TD
Start([Change Access Mode]) --> ValidateInput["Validate Access Mode"]
ValidateInput --> InputValid{"Valid Mode?"}
InputValid --> |No| ReturnError["Return Error"]
InputValid --> |Yes| GetCollaboration["Get Collaboration Record"]
GetCollaboration --> HasCollaboration{"Collaboration Exists?"}
HasCollaboration --> |No| ReturnError
HasCollaboration --> |Yes| CheckMode["Check Current Mode"]
CheckMode --> SameMode{"Mode Unchanged?"}
SameMode --> |Yes| ReturnSuccess["Return Success"]
SameMode --> |No| UpdateCollaboration["Update Collaboration Mode"]
UpdateCollaboration --> UpdateAccess["Update Access Table"]
UpdateAccess --> Recalculate["Recalculate User Access"]
Recalculate --> ReturnSuccess
ReturnError --> End([Function Exit])
ReturnSuccess --> End
```

**Diagram sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L128-L178)
- [models/perm/access/access.go](file://models/perm/access/access.go#L1-L255)

**Section sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L128-L178)
- [models/perm/access/access.go](file://models/perm/access/access.go#L1-L255)

## Team-Based Access Control

### Team Model Structure
Teams in Gitea are organizational units that can be granted access to repositories. Each team has its own access mode and can be configured to include all repositories or specific ones.

```mermaid
classDiagram
class Team {
+int64 ID
+int64 OrgID
+string LowerName
+string Name
+string Description
+AccessMode AccessMode
+int NumRepos
+int NumMembers
+bool IncludesAllRepositories
+bool CanCreateOrgRepo
}
class TeamUnit {
+int64 ID
+int64 TeamID
+unit.Type Type
+AccessMode AccessMode
}
Team --> TeamUnit : "has many"
Team --> User : "members"
```

**Diagram sources**
- [models/organization/team.go](file://models/organization/team.go#L1-L249)

### Team Repository Management
Teams can be added to or removed from repositories, with appropriate permission checks and access recalculation. The system ensures that only users with appropriate permissions can modify team access.

```mermaid
sequenceDiagram
participant Web as Web Interface
participant Service as Repository Service
participant OrgService as Organization Service
participant Model as Repository Model
Web->>OrgService : GetTeam(ctx, orgID, name)
OrgService-->>Web : Return team
Web->>Service : TeamAddRepository(ctx, team, repo)
Service->>Model : HasTeamRepo(ctx, orgID, teamID, repoID)
alt Team already has access
Model-->>Service : Return true
Service-->>Web : Return error
else No access
Model-->>Service : Return false
Service->>Service : Add repository to team
Service->>Service : Recalculate team accesses
Service-->>Web : Success
end
```

**Diagram sources**
- [routers/web/repo/setting/collaboration.go](file://routers/web/repo/setting/collaboration.go#L1-L217)
- [services/org/team.go](file://services/org/team.go#L1-L352)

**Section sources**
- [routers/web/repo/setting/collaboration.go](file://routers/web/repo/setting/collaboration.go#L1-L217)
- [services/org/team.go](file://services/org/team.go#L1-L352)

## Web Interface and Service Layer Integration

### Request Flow for Collaboration Management
The web interface handles collaboration management requests by delegating to the appropriate service layer functions. This separation of concerns ensures that business logic remains in the service layer while the web layer handles presentation and user interaction.

```mermaid
sequenceDiagram
participant Client as Web Browser
participant Router as Web Router
participant Service as Repository Service
participant Model as Repository Model
Client->>Router : POST /settings/collaboration
Router->>Router : Validate form input
Router->>Model : GetUserByName(ctx, name)
alt User not found
Router-->>Client : Show error
else User found
Model-->>Router : Return user
Router->>Model : IsCollaborator(ctx, repoID, u.ID)
alt Already collaborator
Model-->>Router : Return true
Router-->>Client : Show duplicate error
else Not collaborator
Model-->>Router : Return false
Router->>Service : AddOrUpdateCollaborator(ctx, repo, u, AccessModeWrite)
alt Success
Service-->>Router : Return nil
Router->>Router : Send notification email
Router-->>Client : Redirect with success
else Error
Service-->>Router : Return error
Router-->>Client : Show server error
end
end
end
```

**Diagram sources**
- [routers/web/repo/setting/collaboration.go](file://routers/web/repo/setting/collaboration.go#L1-L217)
- [services/repository/collaboration.go](file://services/repository/collaboration.go#L1-L125)

**Section sources**
- [routers/web/repo/setting/collaboration.go](file://routers/web/repo/setting/collaboration.go#L1-L217)

## Permission Inheritance and Recalculation

### Access Recalculation Process
When collaboration or team membership changes, Gitea recalculates the effective access permissions for affected users. This ensures that the access table accurately reflects the current permission state.

```mermaid
flowchart TD
Start([Access Recalculation]) --> LoadRepo["Load Repository Owner"]
LoadRepo --> IsOrg{"Owner is Organization?"}
IsOrg --> |No| RefreshCollaborators["Refresh Collaborator Accesses"]
IsOrg --> |Yes| LoadTeams["Load Organization Teams"]
LoadTeams --> ProcessTeams["Process Each Team"]
ProcessTeams --> HasAccess{"Team has Repo Access?"}
HasAccess --> |No| NextTeam["Process Next Team"]
HasAccess --> |Yes| LoadMembers["Load Team Members"]
LoadMembers --> UpdateAccess["Update User Access Mode"]
UpdateAccess --> NextTeam
NextTeam --> AllTeamsProcessed{"All Teams Processed?"}
AllTeamsProcessed --> |No| ProcessTeams
AllTeamsProcessed --> |Yes| RefreshCollaborators
RefreshCollaborators --> ApplyMinMode["Apply Minimum Access Mode"]
ApplyMinMode --> UpdateAccessTable["Update Access Table"]
UpdateAccessTable --> End([Recalculation Complete])
```

**Diagram sources**
- [models/perm/access/access.go](file://models/perm/access/access.go#L1-L255)
- [services/repository/collaboration.go](file://services/repository/collaboration.go#L1-L125)

**Section sources**
- [models/perm/access/access.go](file://models/perm/access/access.go#L1-L255)

### Effective Permission Determination
Gitea determines a user's effective permission level by considering multiple factors: direct collaboration, team membership, and repository ownership. The system combines these factors to determine the highest applicable access mode.

```mermaid
flowchart TD
Start([Determine Access]) --> IsOwner{"User is Repository Owner?"}
IsOwner --> |Yes| ReturnOwner["Return Owner Access"]
IsOwner --> |No| CheckTeam{"User in Team with Access?"}
CheckTeam --> |Yes| DetermineTeamMode["Determine Team Access Mode"]
CheckTeam --> |No| CheckCollaborator{"User is Collaborator?"}
CheckCollaborator --> |Yes| ReturnCollaboratorMode["Return Collaboration Mode"]
CheckCollaborator --> |No| IsPublic{"Repository is Public?"}
IsPublic --> |Yes| ReturnRead["Return Read Access"]
IsPublic --> |No| ReturnNone["Return No Access"]
DetermineTeamMode --> IsOwnerTeam{"Team is Owner Team?"}
IsOwnerTeam --> |Yes| ReturnOwner
IsOwnerTeam --> |No| ReturnTeamMode["Return Team Access Mode"]
```

**Diagram sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L150-L178)
- [models/perm/access/access.go](file://models/perm/access/access.go#L1-L255)

**Section sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L150-L178)

## Common Issues and Troubleshooting

### Permission Escalation Prevention
Gitea implements several safeguards to prevent unauthorized permission escalation. These include validation of access modes, ownership checks, and blocking of organization accounts from being added as collaborators.

```mermaid
flowchart TD
Start([Add Collaborator]) --> ValidateUser["Validate User"]
ValidateUser --> IsOrg{"User is Organization?"}
IsOrg --> |Yes| RejectOrg["Reject: Organizations cannot be collaborators"]
IsOrg --> |No| IsBlocked{"User is Blocked?"}
IsBlocked --> |Yes| RejectBlocked["Reject: User is blocked"]
IsBlocked --> |No| IsOwner{"User is Repository Owner?"}
IsOwner --> |Yes| RejectOwner["Reject: Owner cannot be collaborator"]
IsOwner --> |No| CheckExisting{"User already Collaborator?"}
CheckExisting --> |Yes| RejectDuplicate["Reject: Already collaborator"]
CheckExisting --> |No| Proceed["Proceed with addition"]
```

**Diagram sources**
- [routers/web/repo/setting/collaboration.go](file://routers/web/repo/setting/collaboration.go#L1-L217)

**Section sources**
- [routers/web/repo/setting/collaboration.go](file://routers/web/repo/setting/collaboration.go#L1-L217)

### Broken Collaboration Links
When a user account is deleted or deactivated, Gitea handles the situation by creating ghost user records to maintain referential integrity while preventing access. This ensures that collaboration history remains intact without compromising security.

```mermaid
flowchart TD
Start([Get Collaborators]) --> QueryDB["Query Collaboration Table"]
QueryDB --> LoadUsers["Load User Records"]
LoadUsers --> UserExists{"User Exists?"}
UserExists --> |Yes| IncludeUser["Include User in Results"]
UserExists --> |No| CreateGhost["Create Ghost User"]
CreateGhost --> IncludeGhost["Include Ghost User"]
IncludeUser --> NextCollaborator["Process Next"]
IncludeGhost --> NextCollaborator
NextCollaborator --> AllProcessed{"All Collaborators Processed?"}
AllProcessed --> |No| QueryDB
AllProcessed --> |Yes| ReturnResults["Return Collaborator List"]
```

**Diagram sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L84-L128)

**Section sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L84-L128)

## Performance Considerations

### Scalability with Numerous Collaborators
For repositories with many collaborators, Gitea optimizes performance by batching database operations and using efficient query patterns. The system minimizes the number of database round-trips by retrieving multiple records in single queries.

```mermaid
flowchart TD
Start([Get Collaborators]) --> QueryCollaborations["Query Collaboration Table"]
QueryCollaborations --> ExtractUserIDs["Extract User IDs"]
ExtractUserIDs --> BatchQuery["Batch Query Users Table"]
BatchQuery --> CreateMap["Create Users Map"]
CreateMap --> ProcessCollaborations["Process Each Collaboration"]
ProcessCollaborations --> GetUser{"Get User from Map"}
GetUser --> CreateCollaborator["Create Collaborator Object"]
CreateCollaborator --> NextCollaboration["Process Next"]
NextCollaboration --> AllProcessed{"All Processed?"}
AllProcessed --> |No| ProcessCollaborations
AllProcessed --> |Yes| ReturnResults["Return Results"]
```

**Diagram sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L50-L84)

**Section sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L50-L84)

### Caching and Recalculation Optimization
The access recalculation system is designed to minimize unnecessary database operations. It only updates access records when changes occur and uses transactional integrity to ensure data consistency.

```mermaid
flowchart TD
Start([Recalculate Access]) --> LoadExisting["Load Existing Accesses"]
LoadExisting --> DetermineChanges["Determine Required Changes"]
DetermineChanges --> DeleteRemoved["Delete Removed Accesses"]
DeleteRemoved --> InsertNew["Insert New Accesses"]
InsertNew --> UpdateModified["Update Modified Accesses"]
UpdateModified --> CommitTransaction["Commit Transaction"]
CommitTransaction --> End([Recalculation Complete])
```

**Diagram sources**
- [models/perm/access/access.go](file://models/perm/access/access.go#L1-L255)

**Section sources**
- [models/perm/access/access.go](file://models/perm/access/access.go#L1-L255)

## Best Practices for Least Privilege

### Principle of Least Privilege Implementation
Gitea enforces the principle of least privilege by defaulting to the minimum necessary permissions and requiring explicit elevation for higher access levels. This approach minimizes the risk of accidental or malicious changes.

```mermaid
flowchart TD
Start([Grant Access]) --> DefaultRead["Default to Read Access"]
DefaultRead --> JustifyElevation["Require Justification for Higher Access"]
JustifyElevation --> ReviewProcess["Implement Review Process"]
ReviewProcess --> MonitorUsage["Monitor Access Usage"]
MonitorUsage --> RegularReview["Conduct Regular Access Reviews"]
RegularReview --> RemoveUnused["Remove Unused Access"]
RemoveUnused --> End([Maintain Least Privilege])
```

**Section sources**
- [services/repository/collaboration.go](file://services/repository/collaboration.go#L1-L125)
- [routers/web/repo/setting/collaboration.go](file://routers/web/repo/setting/collaboration.go#L1-L217)

### Access Review and Maintenance
Regular access reviews are essential for maintaining security and compliance. Gitea provides tools to list all collaborators and their access levels, making it easier to identify and remove unnecessary permissions.

```mermaid
flowchart TD
Start([Access Review]) --> ListCollaborators["List All Collaborators"]
ListCollaborators --> VerifyNecessity["Verify Access Necessity"]
VerifyNecessity --> IdentifyUnused["Identify Unused Access"]
IdentifyUnused --> RemoveAccess["Remove Unnecessary Access"]
RemoveAccess --> DocumentChanges["Document Changes"]
DocumentChanges --> ScheduleNext["Schedule Next Review"]
ScheduleNext --> End([Complete Review])
```

**Section sources**
- [models/repo/collaboration.go](file://models/repo/collaboration.go#L1-L178)
- [routers/web/repo/setting/collaboration.go](file://routers/web/repo/setting/collaboration.go#L1-L217)