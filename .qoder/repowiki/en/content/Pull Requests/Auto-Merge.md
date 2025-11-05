# Auto-Merge

<cite>
**Referenced Files in This Document**   
- [services/pull/merge.go](file://services/pull/merge.go)
- [models/pull/automerge.go](file://models/pull/automerge.go)
- [services/automerge/automerge.go](file://services/automerge/automerge.go)
- [routers/api/v1/repo/pull.go](file://routers/api/v1/repo/pull.go)
- [models/issues/pull.go](file://models/issues/pull.go)
- [services/pull/check.go](file://services/pull/check.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Domain Model](#domain-model)
3. [Auto-Merge Triggering Mechanism](#auto-merge-triggering-mechanism)
4. [Implementation Details](#implementation-details)
5. [User Interface and API Integration](#user-interface-and-api-integration)
6. [Relationship with Status Checks and Branch Protection](#relationship-with-status-checks-and-branch-protection)
7. [Common Issues and Error Handling](#common-issues-and-error-handling)
8. [Best Practices](#best-practices)
9. [Conclusion](#conclusion)

## Introduction
Gitea's auto-merge functionality enables pull requests to be automatically merged when all required status checks pass and approval requirements are met. This feature streamlines the development workflow by reducing manual intervention in the merge process. The system is designed to ensure code quality and compliance with repository policies while enabling efficient integration of changes. This document provides a comprehensive analysis of the auto-merge implementation, covering its domain model, triggering mechanism, integration with status checks and branch protection rules, and best practices for team usage.

## Domain Model
The auto-merge functionality in Gitea is built around a well-defined domain model that represents the state and configuration of scheduled merges. The core entity is the `AutoMerge` struct, which stores information about pull requests scheduled for automatic merging. This struct contains essential fields including the pull request ID, the user who scheduled the merge (DoerID), the merge style to be used, an optional merge message, and a flag indicating whether the source branch should be deleted after merging. The model also includes a timestamp for when the merge was scheduled.

The pull request entity itself contains an `AutoMergeEnabled` flag that indicates whether auto-merge has been enabled for a specific pull request. This flag is part of the broader pull request state management system and works in conjunction with other status indicators such as mergeability and conflict status. The domain model supports multiple merge strategies including merge commits, rebase, squash, and fast-forward only, allowing repository administrators to configure their preferred workflow.

The relationship between the pull request and auto-merge entities is one-to-one, with each pull request capable of having at most one scheduled auto-merge operation. This design prevents conflicts that could arise from multiple merge schedules and ensures a clear audit trail of merge intentions. The model also includes error handling capabilities, with provisions for storing error messages when merge attempts fail, though this feature is noted as pending implementation in the codebase.

**Section sources**
- [models/pull/automerge.go](file://models/pull/automerge.go#L1-L104)
- [models/issues/pull.go](file://models/issues/pull.go#L1-L799)

## Auto-Merge Triggering Mechanism
The auto-merge triggering mechanism in Gitea is event-driven and operates through a background job processing system. When a pull request is updated with new commits, the system automatically checks whether auto-merge can proceed. This process begins when a commit is pushed to the head branch of a pull request, which triggers a webhook that notifies the auto-merge service.

The triggering mechanism relies on a queue-based architecture where pull request merge checks are processed asynchronously. When a pull request is scheduled for auto-merge, it is added to a dedicated queue (`pr_auto_merge`) that processes merge eligibility checks. The system periodically evaluates the status of scheduled merges, checking whether all required status checks have passed and approval requirements are satisfied.

The actual merge triggering occurs when the `handlePullRequestAutoMerge` function processes a pull request from the queue. This function first verifies that the pull request is still eligible for auto-merge by checking that the head commit hasn't changed since the merge was scheduled. It then evaluates the pull request's mergeability by checking status check results and approval status. If all conditions are met, the merge operation is executed immediately.

The system also includes safeguards against race conditions and concurrent modifications. A global lock is acquired during the merge process to prevent multiple merge attempts on the same pull request. Additionally, the system validates that the pull request hasn't been manually merged or closed before attempting an auto-merge, ensuring data consistency.

**Section sources**
- [services/automerge/automerge.go](file://services/automerge/automerge.go#L1-L279)
- [services/pull/check.go](file://services/pull/check.go#L1-L480)

## Implementation Details
The implementation of Gitea's auto-merge functionality is distributed across multiple packages and services, with clear separation of concerns. The core merge logic resides in the `services/pull/merge.go` file, which contains the `Merge` function responsible for executing the actual merge operation. This function performs comprehensive validation before proceeding, including checking merge permissions, verifying the merge strategy is allowed, and acquiring a global lock to prevent concurrent operations.

The merge process follows a transactional pattern, with database operations wrapped in transactions to ensure atomicity. When a merge is executed, the system creates a temporary repository for the merge operation, applies the appropriate merge strategy (merge, rebase, squash, or fast-forward), and then pushes the result back to the base repository. After a successful merge, the system updates the pull request status, creates appropriate notifications, and handles cross-references such as closing related issues.

The auto-merge scheduling is implemented in the `services/automerge/automerge.go` file, which contains the `ScheduleAutoMerge` function. This function creates a database record of the scheduled merge and adds the pull request to the auto-merge queue for processing. The system uses a unique queue to prevent duplicate processing of the same pull request, ensuring that only one merge attempt is processed at a time.

Error handling is comprehensive, with specific error types defined for different failure scenarios such as merge conflicts, unrelated histories, and diverging fast-forward-only branches. When a merge fails, the system logs the error but does not remove the pull request from the auto-merge schedule, allowing for retry when conditions improve. However, the code includes a TODO note indicating that failed merge attempts should eventually display error messages on the pull request page.

**Section sources**
- [services/pull/merge.go](file://services/pull/merge.go#L1-L753)
- [services/automerge/automerge.go](file://services/automerge/automerge.go#L1-L279)

## User Interface and API Integration
Gitea provides both web interface and API endpoints for enabling and managing auto-merge functionality. The web interface integration allows users to enable auto-merge directly from the pull request page through a dedicated button or option. When a user clicks the auto-merge button, a JavaScript event handler triggers an API call to schedule the merge. The frontend code in `web_src/js/features/repo-issue-pull.ts` handles the user interaction, showing loading states and handling redirects after the operation completes.

The API integration is exposed through the `/api/v1/repos/{owner}/{repo}/pulls/{index}/merge` endpoint, which accepts a request body containing the merge style, merge message, and auto-merge flag. The API controller in `routers/api/v1/repo/pull.go` processes these requests, validating user permissions and delegating to the appropriate service functions. The API supports both immediate merging and scheduling for auto-merge when checks succeed.

When auto-merge is enabled through either interface, the system creates a comment on the pull request indicating that the merge has been scheduled. This provides transparency to all collaborators about the merge status. The comment includes information about the merge style and the user who scheduled the merge, creating an audit trail of merge intentions.

The integration also includes client-side polling to update the merge button state. The `initRepoPullMergeBox` function in the frontend code periodically checks the merge eligibility status and updates the UI accordingly. This ensures that users see real-time information about whether their pull request is ready to be merged or if additional checks are still pending.

**Section sources**
- [routers/api/v1/repo/pull.go](file://routers/api/v1/repo/pull.go#L1-L799)
- [web_src/js/features/repo-issue-pull.ts](file://web_src/js/features/repo-issue-pull.ts#L1-L133)

## Relationship with Status Checks and Branch Protection
The auto-merge functionality in Gitea is deeply integrated with the repository's status checks and branch protection rules, ensuring that automated merges comply with the same quality standards as manual merges. Before an auto-merge is executed, the system verifies that all required status checks have passed, including CI/CD pipeline results, code coverage metrics, and any custom status checks configured for the repository.

Branch protection rules play a crucial role in determining auto-merge eligibility. The system checks whether the pull request has received the required number of approvals, ensuring that code review requirements are met before automatic merging. It also verifies that there are no outstanding requested changes that would block the merge. For repositories with protected files, the system confirms that no protected files have been modified in a way that would violate protection rules.

The implementation includes special handling for different merge check types, distinguishing between general merges, manual merges, and auto-merges. When processing an auto-merge request, the system temporarily relaxes certain branch protection checks that are not relevant to the automated context. For example, the branch protection check can be skipped for auto-merge operations, allowing the system to proceed when all other conditions are met.

The relationship between auto-merge and status checks is bidirectional. Not only do status checks determine auto-merge eligibility, but changes to the base branch that affect status checks also trigger re-evaluation of scheduled auto-merges. When the base branch is updated, the system automatically rechecks all pull requests targeting that branch, ensuring that auto-merge decisions are based on the most current status information.

**Section sources**
- [services/pull/check.go](file://services/pull/check.go#L1-L480)
- [services/automerge/automerge.go](file://services/automerge/automerge.go#L1-L279)

## Common Issues and Error Handling
Several common issues can arise with Gitea's auto-merge functionality, particularly related to merge conflicts and timing issues. One frequent problem occurs when merge conflicts develop after auto-merge has been enabled but before the merge is executed. This can happen when multiple pull requests target the same code areas and are processed in sequence. The system detects merge conflicts through Git's merge command output and logs them as `ErrMergeConflicts`, preventing the merge from proceeding.

Another common issue is merge divergence, where the head branch falls behind the base branch, making a fast-forward merge impossible. This is particularly relevant for repositories configured with fast-forward-only merge policies. The system detects this condition and returns an `ErrMergeDivergingFastForwardOnly` error, blocking the auto-merge attempt until the branch is updated.

Failed merge attempts are handled gracefully, with the system logging the error but maintaining the auto-merge schedule for future retry. However, the current implementation does not provide feedback to users about why a merge failed, which can lead to confusion. The code includes a TODO note acknowledging this limitation and suggesting the addition of an error message field to display failure reasons on the pull request page.

Race conditions can also occur when multiple users attempt to merge the same pull request simultaneously. The global lock mechanism prevents concurrent merge operations, but users may experience delays or errors if they attempt to merge a pull request that is already being processed. The system could be improved by providing clearer feedback about the merge queue status and estimated processing time.

**Section sources**
- [services/pull/merge.go](file://services/pull/merge.go#L1-L753)
- [services/automerge/automerge.go](file://services/automerge/automerge.go#L1-L279)

## Best Practices
To safely use auto-merge in team environments, several best practices should be followed. First, repositories should have comprehensive status checks configured, including automated testing, code quality analysis, and security scanning. This ensures that auto-merge only proceeds when code meets the team's quality standards. The status checks should be reliable and fast to minimize the window between when a pull request becomes mergeable and when it is actually merged.

Teams should establish clear guidelines for when auto-merge is appropriate. It is generally recommended for routine changes, bug fixes, and minor updates, while reserving manual review and merge for significant architectural changes or sensitive code areas. The approval requirements should be set appropriately, typically requiring at least one or two approvals from team members with relevant expertise.

To prevent merge conflicts and integration issues, teams should encourage small, focused pull requests that modify limited code areas. This reduces the likelihood of conflicts with other ongoing work. Regularly updating feature branches with the latest changes from the base branch also helps maintain mergeability.

Monitoring and alerting should be implemented to detect issues with the auto-merge system. Teams should review failed merge attempts regularly and address any recurring problems with status checks or merge conflicts. The audit trail of auto-merge operations should be reviewed periodically to ensure compliance with team policies.

Finally, teams should educate all members about how auto-merge works and its implications for the development workflow. This includes understanding that enabling auto-merge commits the team to merging the changes once all checks pass, so it should only be used when the changes are fully ready for integration.

## Conclusion
Gitea's auto-merge functionality provides a powerful mechanism for streamlining the pull request workflow while maintaining code quality and compliance with repository policies. The implementation is robust, with comprehensive validation, error handling, and integration with status checks and branch protection rules. By leveraging a queue-based architecture and global locking, the system ensures reliable and consistent merge operations.

The domain model clearly separates the concerns of pull request management and merge scheduling, allowing for flexible configuration of merge strategies and post-merge actions. The integration with both web interface and API provides multiple avenues for users to enable and manage auto-merge, while the client-side polling ensures up-to-date status information.

While the current implementation is effective, there are opportunities for improvement, particularly in providing better feedback about merge failures and enhancing the user interface to show merge queue status. These enhancements would make the auto-merge feature even more transparent and user-friendly.

Overall, auto-merge is a valuable feature for teams looking to optimize their development workflow, reducing manual overhead while maintaining high standards for code quality and review processes.