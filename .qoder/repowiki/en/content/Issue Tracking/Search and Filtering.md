# Search and Filtering

<cite>
**Referenced Files in This Document**   
- [issue_list.go](file://models/issues/issue_list.go)
- [issue_search.go](file://models/issues/issue_search.go)
- [issue_list.go](file://routers/web/repo/issue_list.go)
- [search.go](file://routers/web/repo/search.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Domain Model](#domain-model)
3. [Service Layer Logic](#service-layer-logic)
4. [Repository Integration](#repository-integration)
5. [Search Indexing and Performance](#search-indexing-and-performance)
6. [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)
7. [Performance Considerations](#performance-considerations)

## Introduction
This document provides a comprehensive analysis of the issue search and filtering functionality in Gitea. It covers the implementation details of searching and filtering issues through various criteria such as state, assignee, label, and milestone. The document examines the domain model, service layer logic, database operations, and integration with repository permissions. It also addresses common issues related to search performance and result filtering, along with performance considerations for large datasets and complex queries.

## Domain Model

The domain model for issue search and filtering is primarily defined in `models/issues/issue_list.go` and `models/issues/issue_search.go`. The core structure is the `IssuesOptions` struct which encapsulates all possible search parameters.

The `IssuesOptions` struct contains various fields for filtering issues:
- **State filtering**: Uses `IsClosed` optional boolean to filter by open/closed status
- **Assignee filtering**: Uses `AssigneeID` string field with special values "(none)" and "(any)" 
- **Label filtering**: Supports both `LabelIDs` (integer array) and `IncludedLabelNames`/`ExcludedLabelNames` (string arrays)
- **Milestone filtering**: Uses `MilestoneIDs` array and `IncludeMilestones` string array
- **Project filtering**: Supports filtering by `ProjectID` and `ProjectColumnID`
- **Temporal filtering**: Includes `UpdatedAfterUnix` and `UpdatedBeforeUnix` for time-based queries
- **Sorting**: Implements various sort types through the `SortType` field

The model also includes permission-related fields like `Owner`, `Team`, and `Doer` which are used to scope search results based on user access rights.

**Section sources**
- [issue_search.go](file://models/issues/issue_search.go#L30-L150)

## Service Layer Logic

The service layer implements complex query construction and database operations for issue search. The primary functions are `Issues()` and `IssueIDs()` in `issue_search.go`, which use XORM sessions to build and execute database queries.

The query construction process follows a modular approach:
1. **Condition application**: The `applyConditions()` function orchestrates the application of various filters
2. **Repository conditions**: `applyRepoConditions()` handles repository access filtering
3. **Label conditions**: `applyLabelsCondition()` implements sophisticated label filtering with support for inclusion/exclusion
4. **Milestone conditions**: `applyMilestoneCondition()` handles milestone-based filtering
5. **Project conditions**: `applyProjectCondition()` and `applyProjectColumnCondition()` manage project-related filters
6. **Sorting**: `applySorts()` implements various sorting strategies including special handling for scope-based sorting

The service layer also implements permission checking through functions like `issuePullAccessibleRepoCond()` which ensures users can only access issues in repositories they have permission to view. This function handles both organizational and individual repository access patterns.

The search functionality supports two execution paths:
- Direct database queries using XORM
- Indexed searches using the issue indexer when available

**Section sources**
- [issue_search.go](file://models/issues/issue_search.go#L152-L527)

## Repository Integration

The integration between the search functionality and repository routes is implemented in `routers/web/repo/issue_list.go`. The `SearchIssues()` function serves as the main entry point for issue search requests.

Key aspects of the repository integration include:
- **Parameter parsing**: Converts HTTP request parameters into `IssuesOptions` struct
- **Repository access determination**: Uses `repo_model.SearchRepositoryIDs()` to find repositories the user can access
- **Keyword processing**: Handles search keywords and validates input
- **Indexer integration**: Uses `issue_indexer.SearchIssues()` when available, falling back to database queries
- **Result formatting**: Converts search results to API-compatible format using `convert.ToIssueList()`

The integration also handles various edge cases:
- Empty repository lists (returns empty results rather than searching all repositories)
- Invalid user inputs (returns appropriate HTTP error codes)
- Permission boundaries (respects repository access controls)

The `SearchRepoIssuesJSON()` function provides a specialized endpoint for repository-specific issue searches, used primarily by the frontend for dependency and related issue searches.

**Section sources**
- [issue_list.go](file://routers/web/repo/issue_list.go#L30-L200)

## Search Indexing and Performance

Gitea implements a dual-layer search system that combines database queries with optional indexing for improved performance. The system intelligently chooses between indexed search and direct database queries based on configuration and availability.

When the issue indexer is enabled (`setting.Indexer.IssueIndexerEnabled`), the system uses `issue_indexer.SearchIssues()` to perform searches. This provides faster search capabilities, especially for large repositories. When the indexer is not available, the system falls back to direct database queries using the `Issues()` function.

The indexing system is integrated with the database through the `db_indexer.GetIndexer().FindWithIssueOptions()` method, which allows for consistent query patterns regardless of the underlying search mechanism. This abstraction enables seamless switching between indexed and non-indexed searches.

Database indexing is crucial for search performance, with key indexes on:
- Issue repository ID (`issue.repo_id`)
- Issue state (`issue.is_closed`)
- Assignee relationships (`issue_assignees`)
- Label relationships (`issue_label`)
- Milestone relationships (`issue.milestone_id`)

The system also implements query optimization through:
- Batch operations with `db.DefaultMaxInSize` limits
- Efficient pagination using `ListOptions`
- Selective field loading to minimize data transfer

**Section sources**
- [issue_list.go](file://routers/web/repo/issue_list.go#L150-L200)
- [issue_search.go](file://models/issues/issue_search.go#L480-L527)

## Common Issues and Troubleshooting

Several common issues can arise with issue search and filtering functionality:

**Slow search performance**: This typically occurs with large datasets or complex queries. Solutions include:
- Ensuring the issue indexer is properly configured and running
- Verifying database indexes are in place and optimized
- Checking for missing or corrupted index data
- Monitoring database query performance

**Incorrect result filtering**: Issues with permission-based filtering can occur when:
- Repository access permissions are not properly synchronized
- Team membership changes are not reflected in search results
- Organization-level permissions are not correctly applied
- Cached permission data becomes stale

**Missing search results**: This can happen when:
- The search indexer is not running or has failed
- Index data is incomplete or corrupted
- Database transactions are not properly committed
- Race conditions occur during indexing

**Pagination issues**: Problems with result pagination may arise from:
- Inconsistent total count calculations
- Page size limits exceeding API constraints
- Incorrect page number handling
- Session timeouts during large result set retrieval

Troubleshooting steps include:
1. Verify indexer status and restart if necessary
2. Check database connection and performance
3. Validate user permissions and repository access
4. Review server logs for error messages
5. Test with simplified queries to isolate the issue

**Section sources**
- [issue_list.go](file://routers/web/repo/issue_list.go#L50-L100)
- [issue_search.go](file://models/issues/issue_search.go#L400-L450)

## Performance Considerations

Search operations with large datasets and complex queries require careful performance consideration. The system implements several optimization strategies:

**Query optimization**: The code uses batch operations with `db.DefaultMaxInSize` to prevent excessively large IN clauses. This prevents database performance degradation and memory issues.

**Caching strategies**: While not explicitly implemented in the provided code, the architecture supports caching at multiple levels:
- Repository access permissions
- Frequently accessed issue metadata
- Search result sets for common queries

**Index utilization**: Proper database indexing is critical for search performance. Key indexes should exist on:
- Foreign key relationships (repository, user, milestone, etc.)
- Frequently filtered fields (state, assignee, labels)
- Sort fields (creation time, update time, priority)

**Resource management**: The system handles large result sets through:
- Configurable pagination limits
- Streamed result processing
- Memory-efficient data structures

**Scalability considerations**: For very large installations, additional optimizations may include:
- Sharding search operations
- Implementing read replicas for search queries
- Using more sophisticated indexing solutions
- Caching frequently accessed search results

The performance characteristics of different search operations vary significantly:
- Simple state-based searches are typically fast
- Label-based searches can be slower due to join operations
- Text-based keyword searches depend heavily on indexer performance
- Complex combinations of filters require more database resources

Monitoring and profiling search operations is recommended to identify performance bottlenecks and optimize accordingly.

**Section sources**
- [issue_search.go](file://models/issues/issue_search.go#L200-L300)
- [issue_list.go](file://routers/web/repo/issue_list.go#L100-L150)