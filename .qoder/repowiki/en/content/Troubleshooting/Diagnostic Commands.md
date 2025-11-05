# Diagnostic Commands

<cite>
**Referenced Files in This Document**   
- [cmd/doctor.go](file://cmd/doctor.go)
- [cmd/doctor_test.go](file://cmd/doctor_test.go)
- [services/doctor/doctor.go](file://services/doctor/doctor.go)
- [services/doctor/dbconsistency.go](file://services/doctor/dbconsistency.go)
- [services/doctor/storage.go](file://services/doctor/storage.go)
- [services/doctor/repository.go](file://services/doctor/repository.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Doctor Command Architecture](#doctor-command-architecture)
3. [Check Registration and Execution](#check-registration-and-execution)
4. [Core Diagnostic Checks](#core-diagnostic-checks)
5. [Database Consistency Checks](#database-consistency-checks)
6. [Storage Integrity Verification](#storage-integrity-verification)
7. [Repository Health Checks](#repository-health-checks)
8. [CLI Usage and Examples](#cli-usage-and-examples)
9. [Web Interface Integration](#web-interface-integration)
10. [Common Issues and Solutions](#common-issues-and-solutions)

## Introduction
Gitea's diagnostic tools provide comprehensive health checking and repair capabilities for Gitea instances. The doctor command system enables administrators to identify and resolve various issues related to database consistency, storage integrity, and repository health. This document details the architecture and implementation of the diagnostic system, focusing on the services/doctor package and its integration with both CLI and web interfaces.

**Section sources**
- [cmd/doctor.go](file://cmd/doctor.go#L0-L37)
- [services/doctor/doctor.go](file://services/doctor/doctor.go#L0-L138)

## Doctor Command Architecture

```mermaid
graph TD
A[CLI Command] --> B[cmd/doctor.go]
B --> C[services/doctor/doctor.go]
C --> D[Individual Check Modules]
D --> E[dbconsistency.go]
D --> F[storage.go]
D --> G[repository.go]
D --> H[Other Check Files]
C --> I[Check Registration]
C --> J[Execution Engine]
B --> K[Flag Processing]
B --> L[Logger Setup]
```

**Diagram sources**
- [cmd/doctor.go](file://cmd/doctor.go#L0-L215)
- [services/doctor/doctor.go](file://services/doctor/doctor.go#L0-L138)

**Section sources**
- [cmd/doctor.go](file://cmd/doctor.go#L0-L215)
- [services/doctor/doctor.go](file://services/doctor/doctor.go#L0-L138)

## Check Registration and Execution

```mermaid
sequenceDiagram
participant CLI as CLI Command
participant Doctor as Doctor Service
participant Check as Individual Check
participant Logger as Diagnostic Logger
CLI->>Doctor : Execute runDoctorCheck()
Doctor->>Doctor : Process flags and arguments
Doctor->>Doctor : Setup diagnostic logger
Doctor->>Doctor : Determine checks to run
loop For each check
Doctor->>Check : Call Run() method
Check->>Logger : Output progress
Check->>Check : Perform diagnostic logic
alt Issue found
Check->>Logger : Report warning/error
alt autofix enabled
Check->>Check : Apply fix
Check->>Logger : Report fix applied
end
else No issues
Check->>Logger : Report OK status
end
end
Doctor->>Logger : Report completion summary
```

**Diagram sources**
- [cmd/doctor.go](file://cmd/doctor.go#L118-L166)
- [services/doctor/doctor.go](file://services/doctor/doctor.go#L81-L120)

**Section sources**
- [cmd/doctor.go](file://cmd/doctor.go#L32-L75)
- [services/doctor/doctor.go](file://services/doctor/doctor.go#L81-L120)

## Core Diagnostic Checks

```mermaid
classDiagram
class Check {
+string Title
+string Name
+bool IsDefault
+func Run(ctx, logger, autofix) error
+bool AbortIfFailed
+bool SkipDatabaseInitialization
+int Priority
+bool InitStorage
}
class Doctor {
+[]*Check Checks
+Register(*Check) void
+RunChecks(ctx, colorize, autofix, []*Check) error
+SortChecks([]*Check) void
}
Doctor --> Check : contains
Check <|-- DBConsistencyCheck : implements
Check <|-- StorageCheck : implements
Check <|-- RepositoryCheck : implements
class DBConsistencyCheck {
+checkDBConsistency(ctx, logger, autofix) error
}
class StorageCheck {
+checkStorage(opts) func(ctx, logger, autofix) error
+commonCheckStorage(logger, autofix, opts) error
}
class RepositoryCheck {
+handleDeleteOrphanedRepos(ctx, logger, autofix) error
}
```

**Diagram sources**
- [services/doctor/doctor.go](file://services/doctor/doctor.go#L0-L138)
- [services/doctor/dbconsistency.go](file://services/doctor/dbconsistency.go#L0-L274)
- [services/doctor/storage.go](file://services/doctor/storage.go#L0-L270)
- [services/doctor/repository.go](file://services/doctor/repository.go#L0-L78)

**Section sources**
- [services/doctor/doctor.go](file://services/doctor/doctor.go#L0-L138)

## Database Consistency Checks

```mermaid
flowchart TD
Start([Start Database Check]) --> CountOrphaned
CountOrphaned["Count orphaned objects<br/>Labels, Issues, PullRequests,<br/>Attachments, Releases, etc."]
CountOrphaned --> FoundIssues{Orphaned objects found?}
FoundIssues --> |Yes| AutofixEnabled{Autofix enabled?}
FoundIssues --> |No| ReportOK["Report: OK"]
AutofixEnabled --> |Yes| DeleteOrphaned["Delete orphaned objects"]
DeleteOrphaned --> ReportFixed["Report: Fixed count"]
AutofixEnabled --> |No| ReportWarning["Report: Warning - orphaned objects found"]
ReportFixed --> End([End])
ReportWarning --> End
ReportOK --> End
subgraph PostgreSQL Specific
CountBadSequences["Count bad sequence values"]
CountBadSequences --> FixBadSequences["Fix sequence values"]
FixBadSequences --> ReportSequencesFixed["Report: Sequences updated"]
end
```

**Diagram sources**
- [services/doctor/dbconsistency.go](file://services/doctor/dbconsistency.go#L0-L274)

**Section sources**
- [services/doctor/dbconsistency.go](file://services/doctor/dbconsistency.go#L0-L274)

## Storage Integrity Verification

```mermaid
flowchart TD
Start([Start Storage Check]) --> InitStorage["Initialize storage system"]
InitStorage --> CheckAttachments["Check attachments storage"]
CheckAttachments --> CheckLFS["Check LFS storage"]
CheckLFS --> CheckAvatars["Check user avatars storage"]
CheckAvatars --> CheckRepoAvatars["Check repository avatars storage"]
CheckRepoAvatars --> CheckArchives["Check repository archives storage"]
CheckArchives --> CheckPackages["Check packages storage"]
subgraph Individual Storage Check
A["Iterate through all objects"] --> B{Object orphaned?}
B --> |Yes| C["Add to deletion list"]
B --> |No| D["Continue"]
C --> E["Process next object"]
D --> E
E --> F{All objects processed?}
F --> |No| A
F --> |Yes| G{Autofix enabled?}
G --> |Yes| H["Delete objects in deletion list"]
G --> |No| I["Report orphaned objects count"]
H --> J["Report deletion results"]
I --> J
J --> K([Return results])
end
```

**Diagram sources**
- [services/doctor/storage.go](file://services/doctor/storage.go#L0-L270)

**Section sources**
- [services/doctor/storage.go](file://services/doctor/storage.go#L0-L270)

## Repository Health Checks

```mermaid
flowchart TD
Start([Start Repository Check]) --> CountOrphanedRepos["Count repositories with non-existent owners"]
CountOrphanedRepos --> FoundOrphaned{Orphaned repositories found?}
FoundOrphaned --> |No| ReportOK["Report: OK"]
FoundOrphaned --> |Yes| AutofixEnabled{Autofix enabled?}
AutofixEnabled --> |No| ReportWarning["Report: Warning - orphaned repositories found"]
AutofixEnabled --> |Yes| DeleteRepos["Delete orphaned repositories"]
DeleteRepos --> BatchProcessing{All repositories deleted?}
BatchProcessing --> |No| ContinueBatch["Process next batch"]
BatchProcessing --> |Yes| ReportFixed["Report: Orphaned repositories deleted"]
ContinueBatch --> DeleteRepos
ReportWarning --> End([End])
ReportFixed --> End
ReportOK --> End
```

**Diagram sources**
- [services/doctor/repository.go](file://services/doctor/repository.go#L0-L78)

**Section sources**
- [services/doctor/repository.go](file://services/doctor/repository.go#L0-L78)

## CLI Usage and Examples

```mermaid
flowchart TD
A[Command Execution] --> B{Command Type}
B --> |check| C[Run diagnostic checks]
B --> |recreate-table| D[Recreate database tables]
B --> |convert| E[Convert database encoding]
C --> F{Flags provided?}
F --> |list| G[List available checks]
F --> |all| H[Run all checks]
F --> |run| I[Run specified checks]
F --> |default| J[Run default checks]
F --> |fix| K[Enable autofix mode]
F --> |log-file| L[Set log output file]
G --> M[Display check table]
H --> N[Execute all registered checks]
I --> O[Execute specified checks]
J --> P[Execute default checks]
K --> Q[Apply fixes automatically]
L --> R[Redirect logs to file/stdout]
```

**Diagram sources**
- [cmd/doctor.go](file://cmd/doctor.go#L32-L75)

**Section sources**
- [cmd/doctor.go](file://cmd/doctor.go#L32-L75)

## Web Interface Integration

```mermaid
graph TD
A[Web Interface] --> B[API Endpoint]
B --> C[Doctor Service]
C --> D[Execute Checks]
D --> E[Format Results]
E --> F[Return JSON Response]
F --> G[Web UI Display]
H[CLI Interface] --> I[Doctor Command]
I --> C
C --> J[Format Console Output]
J --> K[Display Results]
C --> L[Shared Diagnostic Logic]
L --> M[Database Consistency]
L --> N[Storage Integrity]
L --> O[Repository Health]
```

**Diagram sources**
- [cmd/doctor.go](file://cmd/doctor.go#L0-L215)
- [services/doctor/doctor.go](file://services/doctor/doctor.go#L0-L138)

## Common Issues and Solutions

```mermaid
flowchart TD
A[Common Issues] --> B[Database Inconsistencies]
A --> C[Storage Problems]
A --> D[Repository Issues]
B --> B1["Orphaned records (labels, issues, pull requests)"]
B --> B2["Missing foreign key references"]
B --> B3["Incorrect counter values"]
B --> B4["Sequence value mismatches (PostgreSQL)"]
C --> C1["Orphaned attachment files"]
C --> C2["Orphaned LFS objects"]
C --> C3["Orphaned avatar images"]
C --> C4["Orphaned repository archives"]
C --> C5["Orphaned package blobs"]
D --> D1["Repositories with non-existent owners"]
D --> D2["Missing repository directories"]
D --> D3["Corrupted Git repositories"]
B1 --> E[Solutions]
B2 --> E
B3 --> E
B4 --> E
C1 --> E
C2 --> E
C3 --> E
C4 --> E
C5 --> E
D1 --> E
D2 --> E
D3 --> E
E --> F["Run doctor check with appropriate flags"]
F --> G["Use --fix to automatically repair"]
G --> H["Verify results and restart Gitea if needed"]
```

**Section sources**
- [services/doctor/dbconsistency.go](file://services/doctor/dbconsistency.go#L0-L274)
- [services/doctor/storage.go](file://services/doctor/storage.go#L0-L270)
- [services/doctor/repository.go](file://services/doctor/repository.go#L0-L78)