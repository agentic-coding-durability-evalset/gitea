# Branching and Protection

<cite>
**Referenced Files in This Document**   
- [protected_branch.go](file://models/git/protected_branch.go)
- [branch.go](file://services/repository/branch.go)
- [protected_branch.go](file://routers/web/repo/setting/protected_branch.go)
- [protected_branch_list.go](file://models/git/protected_branch_list.go)
- [pull.go](file://services/pull/pull.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Core Domain Models](#core-domain-models)
3. [Branch Creation and Deletion](#branch-creation-and-deletion)
4. [Branch Protection Rules](#branch-protection-rules)
5. [Web Interface and Service Layer Integration](#web-interface-and-service-layer-integration)
6. [Protected Branch Configuration Examples](#protected-branch-configuration-examples)
7. [Common Issues and Conflicts](#common-issues-and-conflicts)
8. [Performance Considerations](#performance-considerations)
9. [Best Practices for Git Workflow Integration](#best-practices-for-git-workflow-integration)
10. [Conclusion](#conclusion)

## Introduction
This document provides a comprehensive analysis of the branching and branch protection mechanisms in Gitea, focusing on the implementation details in `models/git/protected_branch.go` and `services/repository/branch.go`. It covers branch creation, deletion, and protection rules, including the invocation relationship between the web interface and service layer through `routers/web/repo/setting/protected_branch.go`. The document also includes concrete examples of protected branch configurations with required pull request reviews, status checks, and push restrictions. It explains domain models for branch protection policies, including required approvers, code owner reviews, and dismissal rules. Additionally, it addresses common issues such as bypassing protection rules and conflicts during branch deletion, provides performance considerations for repositories with numerous branches, and offers best practices for Git workflow integration.

**Section sources**
- [protected_branch.go](file://models/git/protected_branch.go#L1-L610)
- [branch.go](file://services/repository/branch.go#L1-L831)

## Core Domain Models

The core domain model for branch protection in Gitea is the `ProtectedBranch` struct, which encapsulates all the rules and configurations for a protected branch. This struct is defined in `models/git/protected_branch.go` and includes various fields to control different aspects of branch protection.

```mermaid
classDiagram
class ProtectedBranch {
+int64 ID
+int64 RepoID
+*Repository Repo
+string RuleName
+int64 Priority
+bool CanPush
+bool EnableWhitelist
+[]int64 WhitelistUserIDs
+[]int64 WhitelistTeamIDs
+bool EnableMergeWhitelist
+bool WhitelistDeployKeys
+[]int64 MergeWhitelistUserIDs
+[]int64 MergeWhitelistTeamIDs
+bool CanForcePush
+bool EnableForcePushAllowlist
+[]int64 ForcePushAllowlistUserIDs
+[]int64 ForcePushAllowlistTeamIDs
+bool ForcePushAllowlistDeployKeys
+bool EnableStatusCheck
+[]string StatusCheckContexts
+bool EnableApprovalsWhitelist
+[]int64 ApprovalsWhitelistUserIDs
+[]int64 ApprovalsWhitelistTeamIDs
+int64 RequiredApprovals
+bool BlockOnRejectedReviews
+bool BlockOnOfficialReviewRequests
+bool BlockOnOutdatedBranch
+bool DismissStaleApprovals
+bool IgnoreStaleApprovals
+bool RequireSignedCommits
+string ProtectedFilePatterns
+string UnprotectedFilePatterns
+bool BlockAdminMergeOverride
+timeutil.TimeStamp CreatedUnix
+timeutil.TimeStamp UpdatedUnix
}
```

**Diagram sources**
- [protected_branch.go](file://models/git/protected_branch.go#L28-L110)

The `ProtectedBranch` struct includes fields for controlling push permissions, merge permissions, force push permissions, status checks, required approvals, and file patterns. The `RuleName` field can be a specific branch name or a glob pattern to match multiple branches. The `Priority` field determines the order in which rules are applied, with higher priority rules taking precedence.

**Section sources**
- [protected_branch.go](file://models/git/protected_branch.go#L28-L110)

## Branch Creation and Deletion

Branch creation and deletion in Gitea are managed through the `CreateNewBranch` and `DeleteBranch` functions in `services/repository/branch.go`. These functions handle the creation and deletion of branches, including the necessary checks and updates to the repository's state.

### Branch Creation

The `CreateNewBranch` function creates a new branch from an existing branch or commit. It first checks if the branch name is valid and does not conflict with existing branches or tags. If the branch name is valid, it creates a new branch using the `git.Push` function.

```mermaid
sequenceDiagram
participant User
participant BranchService
participant GitRepo
participant Database
User->>BranchService : CreateNewBranch(oldBranchName, branchName)
BranchService->>BranchService : checkBranchName(branchName)
alt Branch name is valid
BranchService->>GitRepo : GetBranchCommit(oldBranchName)
GitRepo-->>BranchService : CommitID
BranchService->>GitRepo : Push(commitID, branchName)
GitRepo-->>BranchService : Success
BranchService->>Database : Insert new branch record
Database-->>BranchService : Success
BranchService-->>User : Success
else Branch name is invalid
BranchService-->>User : Error
end
```

**Diagram sources**
- [branch.go](file://services/repository/branch.go#L36-L76)

### Branch Deletion

The `DeleteBranch` function deletes a branch from the repository. It first checks if the branch can be deleted, which includes verifying that the branch is not the default branch and is not protected. If the branch can be deleted, it removes the branch from the Git repository and updates the database to mark the branch as deleted.

```mermaid
sequenceDiagram
participant User
participant BranchService
participant GitRepo
participant Database
User->>BranchService : DeleteBranch(branchName)
BranchService->>BranchService : CanDeleteBranch(branchName)
alt Branch can be deleted
BranchService->>Database : AddDeletedBranch(branchName)
Database-->>BranchService : Success
BranchService->>GitRepo : DeleteBranch(branchName)
GitRepo-->>BranchService : Success
BranchService->>Database : Update branch record
Database-->>BranchService : Success
BranchService-->>User : Success
else Branch cannot be deleted
BranchService-->>User : Error
end
```

**Diagram sources**
- [branch.go](file://services/repository/branch.go#L224-L278)

**Section sources**
- [branch.go](file://services/repository/branch.go#L36-L278)

## Branch Protection Rules

Branch protection rules in Gitea are implemented through the `ProtectedBranch` struct and associated functions in `models/git/protected_branch.go`. These rules control various aspects of branch behavior, including push permissions, merge permissions, and required approvals.

### Push Permissions

Push permissions are controlled by the `CanPush` and `EnableWhitelist` fields. If `CanPush` is `true`, users with write access to the repository can push to the branch. If `EnableWhitelist` is `true`, only users and teams listed in `WhitelistUserIDs` and `WhitelistTeamIDs` can push to the branch.

```mermaid
flowchart TD
Start([Start]) --> CanPush{"CanPush?"}
CanPush --> |No| ReturnFalse["Return false"]
CanPush --> |Yes| EnableWhitelist{"EnableWhitelist?"}
EnableWhitelist --> |No| CheckWriteAccess["Check write access"]
CheckWriteAccess --> |Has access| ReturnTrue["Return true"]
CheckWriteAccess --> |No access| ReturnFalse
EnableWhitelist --> |Yes| CheckWhitelist["Check whitelist"]
CheckWhitelist --> |In whitelist| ReturnTrue
CheckWhitelist --> |Not in whitelist| ReturnFalse
ReturnTrue --> End([End])
ReturnFalse --> End
```

**Diagram sources**
- [protected_branch.go](file://models/git/protected_branch.go#L109-L154)

### Merge Permissions

Merge permissions are controlled by the `EnableMergeWhitelist` field. If `EnableMergeWhitelist` is `true`, only users and teams listed in `MergeWhitelistUserIDs` and `MergeWhitelistTeamIDs` can merge to the branch. Otherwise, users with write access to the repository can merge to the branch.

```mermaid
flowchart TD
Start([Start]) --> EnableMergeWhitelist{"EnableMergeWhitelist?"}
EnableMergeWhitelist --> |No| CheckWriteAccess["Check write access"]
CheckWriteAccess --> |Has access| ReturnTrue["Return true"]
CheckWriteAccess --> |No access| ReturnFalse["Return false"]
EnableMergeWhitelist --> |Yes| CheckWhitelist["Check whitelist"]
CheckWhitelist --> |In whitelist| ReturnTrue
CheckWhitelist --> |Not in whitelist| ReturnFalse
ReturnTrue --> End([End])
ReturnFalse --> End
```

**Diagram sources**
- [protected_branch.go](file://models/git/protected_branch.go#L155-L180)

### Required Approvals

Required approvals are controlled by the `RequiredApprovals` and `EnableApprovalsWhitelist` fields. If `RequiredApprovals` is greater than 0, a certain number of approvals are required before a pull request can be merged. If `EnableApprovalsWhitelist` is `true`, only users and teams listed in `ApprovalsWhitelistUserIDs` and `ApprovalsWhitelistTeamIDs` can provide approvals.

```mermaid
flowchart TD
Start([Start]) --> RequiredApprovals{"RequiredApprovals > 0?"}
RequiredApprovals --> |No| ReturnTrue["Return true"]
RequiredApprovals --> |Yes| CountApprovals["Count approvals"]
CountApprovals --> |Count >= RequiredApprovals| ReturnTrue
CountApprovals --> |Count < RequiredApprovals| ReturnFalse["Return false"]
ReturnTrue --> End([End])
ReturnFalse --> End
```

**Diagram sources**
- [protected_branch.go](file://models/git/protected_branch.go#L244-L262)

**Section sources**
- [protected_branch.go](file://models/git/protected_branch.go#L109-L262)

## Web Interface and Service Layer Integration

The web interface for managing branch protection rules is implemented in `routers/web/repo/setting/protected_branch.go`. This file contains the handlers for displaying and updating branch protection rules, which interact with the service layer to perform the necessary operations.

### Displaying Branch Protection Rules

The `ProtectedBranchRules` handler retrieves the list of protected branch rules for a repository and displays them in the web interface. It uses the `FindRepoProtectedBranchRules` function to retrieve the rules from the database and passes them to the template for rendering.

```mermaid
sequenceDiagram
participant User
participant WebHandler
participant ServiceLayer
participant Database
User->>WebHandler : GET /settings/branches
WebHandler->>ServiceLayer : FindRepoProtectedBranchRules(repoID)
ServiceLayer->>Database : Query protected branch rules
Database-->>ServiceLayer : List of rules
ServiceLayer-->>WebHandler : List of rules
WebHandler->>User : Render template with rules
```

**Diagram sources**
- [protected_branch.go](file://routers/web/repo/setting/protected_branch.go#L15-L37)

### Updating Branch Protection Rules

The `SettingsProtectedBranchPost` handler updates the branch protection rules for a repository. It retrieves the form data from the request, validates it, and then calls the `CreateOrUpdateProtectedBranch` function to update the rules in the database.

```mermaid
sequenceDiagram
participant User
participant WebHandler
participant ServiceLayer
participant Database
User->>WebHandler : POST /settings/branches/edit
WebHandler->>WebHandler : Validate form data
alt Form data is valid
WebHandler->>ServiceLayer : CreateOrUpdateProtectedBranch(repo, protectBranch, whitelistOptions)
ServiceLayer->>Database : Update protected branch rule
Database-->>ServiceLayer : Success
ServiceLayer-->>WebHandler : Success
WebHandler->>User : Redirect to success page
else Form data is invalid
WebHandler->>User : Show error message
end
```

**Diagram sources**
- [protected_branch.go](file://routers/web/repo/setting/protected_branch.go#L144-L296)

**Section sources**
- [protected_branch.go](file://routers/web/repo/setting/protected_branch.go#L15-L296)

## Protected Branch Configuration Examples

This section provides concrete examples of protected branch configurations, including required pull request reviews, status checks, and push restrictions.

### Example 1: Basic Protection with Required Approvals

This example configures a branch to require at least 2 approvals before a pull request can be merged.

```json
{
  "RuleName": "main",
  "CanPush": false,
  "EnableMergeWhitelist": false,
  "RequiredApprovals": 2,
  "BlockOnRejectedReviews": true,
  "DismissStaleApprovals": true
}
```

### Example 2: Advanced Protection with Status Checks and File Patterns

This example configures a branch to require status checks, specific file patterns, and a whitelist of users who can push.

```json
{
  "RuleName": "release/*",
  "CanPush": true,
  "EnableWhitelist": true,
  "WhitelistUserIDs": [1, 2, 3],
  "WhitelistTeamIDs": [4, 5],
  "EnableStatusCheck": true,
  "StatusCheckContexts": ["ci/circleci", "codeclimate/test"],
  "ProtectedFilePatterns": "config/*;secrets/*",
  "UnprotectedFilePatterns": "docs/*"
}
```

**Section sources**
- [protected_branch.go](file://models/git/protected_branch.go#L28-L110)
- [protected_branch.go](file://routers/web/repo/setting/protected_branch.go#L144-L296)

## Common Issues and Conflicts

This section addresses common issues and conflicts that may arise when working with branch protection rules.

### Bypassing Protection Rules

One common issue is users attempting to bypass protection rules by creating branches with similar names or using force push. Gitea prevents this by enforcing strict branch name validation and requiring explicit permissions for force push.

### Conflicts During Branch Deletion

Another common issue is conflicts during branch deletion, especially when the branch is protected or is the default branch. Gitea handles this by checking the branch's status before deletion and preventing deletion if the branch is protected or is the default branch.

```mermaid
flowchart TD
Start([Start]) --> IsDefault{"Is default branch?"}
IsDefault --> |Yes| ReturnError["Return error"]
IsDefault --> |No| IsProtected{"Is protected?"}
IsProtected --> |Yes| ReturnError
IsProtected --> |No| DeleteBranch["Delete branch"]
DeleteBranch --> End([End])
ReturnError --> End
```

**Diagram sources**
- [branch.go](file://services/repository/branch.go#L224-L278)

**Section sources**
- [branch.go](file://services/repository/branch.go#L224-L278)

## Performance Considerations

This section provides performance considerations for repositories with numerous branches.

### Repository with Numerous Branches

For repositories with numerous branches, it is important to optimize the branch listing and protection rule evaluation processes. Gitea uses caching and efficient database queries to minimize the performance impact of branch operations.

### Caching Divergence Information

Gitea caches divergence information for branches to avoid recalculating it on every request. This improves performance by reducing the number of Git operations required to display branch information.

```mermaid
flowchart TD
Start([Start]) --> GetDivergence{"Get divergence from cache?"}
GetDivergence --> |Yes| ReturnCached["Return cached divergence"]
GetDivergence --> |No| CalculateDivergence["Calculate divergence"]
CalculateDivergence --> CacheDivergence["Cache divergence"]
CacheDivergence --> ReturnCalculated["Return calculated divergence"]
ReturnCached --> End([End])
ReturnCalculated --> End
```

**Diagram sources**
- [branch.go](file://services/repository/branch.go#L104-L139)

**Section sources**
- [branch.go](file://services/repository/branch.go#L104-L139)

## Best Practices for Git Workflow Integration

This section provides best practices for integrating branch protection rules with common Git workflows, such as Git Flow and GitHub Flow.

### Git Flow

In Git Flow, the `main` branch is protected and requires pull requests and approvals for all changes. Feature branches are created from `develop` and merged back into `develop` after review. Release branches are created from `develop` and merged into `main` after testing.

### GitHub Flow

In GitHub Flow, the `main` branch is protected and requires pull requests and approvals for all changes. Feature branches are created from `main` and merged back into `main` after review. Continuous integration and deployment are used to automate testing and deployment.

**Section sources**
- [protected_branch.go](file://models/git/protected_branch.go#L28-L110)
- [branch.go](file://services/repository/branch.go#L36-L278)

## Conclusion

This document has provided a comprehensive analysis of the branching and branch protection mechanisms in Gitea. It has covered the implementation details in `models/git/protected_branch.go` and `services/repository/branch.go`, including branch creation, deletion, and protection rules. It has also explained the invocation relationship between the web interface and service layer through `routers/web/repo/setting/protected_branch.go`. The document has included concrete examples of protected branch configurations with required pull request reviews, status checks, and push restrictions. It has addressed common issues such as bypassing protection rules and conflicts during branch deletion, provided performance considerations for repositories with numerous branches, and offered best practices for Git workflow integration.