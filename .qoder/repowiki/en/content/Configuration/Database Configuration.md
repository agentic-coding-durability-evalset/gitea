# Database Configuration

<cite>
**Referenced Files in This Document**   
- [database.go](file://modules/setting/database.go)
- [engine_init.go](file://models/db/engine_init.go)
- [migrations.go](file://models/migrations/migrations.go)
- [app.ini](file://docker/root/etc/templates/app.ini)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Supported Database Types](#supported-database-types)
3. [Configuration Settings](#configuration-settings)
4. [Connection String Formats](#connection-string-formats)
5. [Connection Pooling Configuration](#connection-pooling-configuration)
6. [Database Schema and Migration](#database-schema-and-migration)
7. [Performance Considerations](#performance-considerations)
8. [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)
9. [Security Implications](#security-implications)

## Introduction

Gitea's database configuration system provides flexible support for multiple database backends, allowing administrators to choose the most appropriate storage solution for their deployment. The configuration is managed through the `database.go` module in the `modules/setting` package, which handles database connection parameters, connection pooling, and integration with Gitea's ORM layer. The system supports MySQL, PostgreSQL, SQLite3, and MSSQL databases, with configuration options that can be set through the app.ini configuration file or environment variables.

The database configuration is critical to Gitea's operation, as it determines how the application connects to and interacts with the underlying data store. Proper configuration ensures optimal performance, reliability, and security for the entire Gitea instance.

**Section sources**
- [database.go](file://modules/setting/database.go#L1-L225)

## Supported Database Types

Gitea supports four database types, each with specific configuration requirements and use cases:

1. **MySQL**: A popular open-source relational database management system known for its performance and reliability in web applications.
2. **PostgreSQL**: An advanced open-source database with strong support for SQL standards, extensibility, and complex queries.
3. **SQLite3**: A lightweight, file-based database that requires no separate server process, ideal for small to medium deployments.
4. **MSSQL**: Microsoft SQL Server, providing enterprise-level database capabilities for Windows environments.

The supported database types are defined in the `SupportedDatabaseTypes` variable, while `DatabaseTypeNames` provides user-friendly names for display purposes. The system includes type checking methods such as `IsMySQL()`, `IsPostgreSQL()`, `IsMSSQL()`, and `IsSQLite3()` to facilitate conditional logic based on the database type.

SQLite3 support is conditional and must be enabled at build time using the appropriate build flags. The `EnableSQLite3` boolean variable controls whether SQLite3 functionality is available in the compiled binary.

**Section sources**
- [database.go](file://modules/setting/database.go#L10-L15)

## Configuration Settings

The database configuration is stored in the `Database` struct, which contains the following key settings:

- **Type**: The database type (mysql, postgres, mssql, sqlite3)
- **Host**: The database server hostname or IP address
- **Name**: The name of the database
- **User**: The database username for authentication
- **Passwd**: The password for database authentication
- **Schema**: The database schema (primarily used with PostgreSQL)
- **SSLMode**: SSL/TLS connection mode for secure connections
- **Path**: File path for SQLite3 databases
- **MysqlCharset**: MySQL character set specification
- **CharsetCollation**: Character set collation settings
- **Timeout**: Connection timeout in seconds

These settings are loaded from the configuration file's `[database]` section using the `LoadDBSetting()` function, which reads values from the `CfgProvider` configuration source. Default values are applied for certain settings when not explicitly configured, such as a default timeout of 500 milliseconds for SQLite3 databases.

**Section sources**
- [database.go](file://modules/setting/database.go#L16-L45)

## Connection String Formats

Gitea generates database-specific connection strings through the `DBConnStr()` function, which creates appropriate connection strings based on the configured database type:

### MySQL Connection Format
For MySQL databases, the connection string follows the format:
```
user:password@tcp(host:port)/database?parseTime=true&tls=mode
```
When using Unix sockets, the format changes to use "unix" instead of "tcp". The SSL mode is translated from PostgreSQL-style values ("disable") to MySQL-compatible values ("false").

### PostgreSQL Connection Format
PostgreSQL connections use a URL-based format:
```
postgres://user:password@host:port/database?sslmode=mode
```
The `getPostgreSQLConnectionString()` function handles the construction of this URL, including proper URL encoding of special characters in credentials. For Unix socket connections, the host parameter is passed as a query parameter.

### MSSQL Connection Format
MSSQL connections use a semicolon-separated key-value format:
```
server=host; port=port; database=name; user id=user; password=password;
```
The `ParseMSSQLHostPort()` function extracts the host and port from the HOST configuration, supporting both colon (:) and comma (,) as separators.

### SQLite3 Connection Format
SQLite3 connections use a file-based format:
```
file:path?cache=shared&mode=rwc&_busy_timeout=timeout&_txlock=immediate&_journal_mode=mode
```
The connection string includes parameters for shared caching, write access mode, busy timeout, transaction locking, and journal mode configuration.

**Section sources**
- [database.go](file://modules/setting/database.go#L93-L132)

## Connection Pooling Configuration

Gitea implements comprehensive connection pooling to optimize database performance and resource utilization. The connection pool settings include:

- **MaxIdleConns**: Maximum number of idle connections in the pool (default: 2)
- **MaxOpenConns**: Maximum number of open connections to the database (default: 0, unlimited)
- **ConnMaxLifetime**: Maximum lifetime of a connection before it's closed (3 seconds for MySQL, 0 for others)
- **DBConnectRetries**: Number of connection attempts before failing (default: 10)
- **DBConnectBackoff**: Delay between connection retry attempts (default: 3 seconds)

These settings are configured through the `InitEngine()` function in `models/db/engine_init.go`, which sets the pool parameters on the XORM engine. The connection pool helps prevent resource exhaustion by reusing existing connections and limiting the total number of concurrent connections to the database server.

The default `ConnMaxLifetime` of 3 seconds for MySQL is designed to work around MySQL's default connection timeout (typically 8 hours), ensuring connections are refreshed before they become stale. For other database types, connections can remain open indefinitely.

**Section sources**
- [database.go](file://modules/setting/database.go#L46-L52)
- [engine_init.go](file://models/db/engine_init.go#L58-L64)

## Database Schema and Migration

Gitea implements an automated database migration system to manage schema changes across versions. The migration system is defined in `models/migrations/migrations.go` and follows a sequential approach with over 300 migration steps as of the current version.

Key aspects of the migration system include:

- **Version Tracking**: The current database version is stored in the `version` table, with the version number corresponding to the last applied migration ID plus one.
- **Automated Migration**: When `AutoMigration` is enabled (default), Gitea automatically applies pending migrations on startup.
- **Migration Order**: Migrations are applied in strict sequence, with each migration having a unique ID number.
- **Fresh Installation Handling**: For new installations, the version table is initialized with the current expected version, skipping all migrations.

The migration system ensures database schema compatibility across Gitea versions and provides a reliable mechanism for evolving the database structure as new features are added. Administrators can manually run migrations using the `gitea migrate` command if automatic migration fails.

**Section sources**
- [database.go](file://modules/setting/database.go#L53)
- [migrations.go](file://models/migrations/migrations.go#L500-L530)

## Performance Considerations

Proper database configuration is essential for optimal Gitea performance. Key performance-related settings include:

- **Connection Limits**: Configuring appropriate `MaxOpenConns` and `MaxIdleConns` values prevents database server overload while ensuring sufficient connections for application needs.
- **Query Logging**: The `LogSQL` setting enables SQL query logging for debugging and performance analysis.
- **Slow Query Detection**: The `SlowQueryThreshold` setting (default: 5 seconds) identifies and logs queries that exceed the specified duration.
- **Iterate Buffer Size**: The `IterateBufferSize` setting (default: 50) controls the number of records processed in batch operations.

For production deployments, it's recommended to:
- Set `MaxOpenConns` to a reasonable limit based on the database server's capacity
- Monitor slow queries and optimize database indexes accordingly
- Use connection pooling to reduce connection overhead
- Regularly monitor database performance metrics

The XORM engine hook system, implemented in `engine_hook.go`, provides performance monitoring capabilities, including tracing of database operations and logging of slow queries.

**Section sources**
- [database.go](file://modules/setting/database.go#L52-L53)
- [engine_init.go](file://models/db/engine_init.go#L75-L78)

## Common Issues and Troubleshooting

Several common issues may arise with database configuration:

### Connection Failures
- **Authentication Errors**: Verify username, password, and database name in the configuration
- **Network Issues**: Check that the database server is reachable from the Gitea server
- **Socket Permissions**: For Unix socket connections, ensure proper file permissions
- **Firewall Restrictions**: Verify that firewall rules allow database connections

### Migration Errors
- **Version Mismatch**: Ensure the database version matches the expected version for the Gitea release
- **Insufficient Permissions**: The database user must have sufficient privileges to create tables and modify schema
- **Corrupted Database**: In rare cases, database corruption may require restoration from backup

### Performance Bottlenecks
- **Connection Pool Exhaustion**: Increase `MaxOpenConns` or optimize application logic to release connections promptly
- **Slow Queries**: Enable `LogSQL` and analyze slow queries for optimization opportunities
- **Lock Contention**: For SQLite3, consider the journal mode and timeout settings to reduce contention

When troubleshooting database issues, check the Gitea logs for detailed error messages and consider running the `gitea doctor` command, which includes database connectivity tests.

**Section sources**
- [database.go](file://modules/setting/database.go#L131)
- [engine_init.go](file://models/db/engine_init.go#L25-L35)

## Security Implications

Database configuration has several security implications that administrators should consider:

- **Password Security**: Database passwords should be stored securely and not exposed in configuration files accessible to unauthorized users
- **SSL/TLS Encryption**: Use SSL mode "require" or "verify-full" for encrypted connections to remote databases
- **Least Privilege**: The database user should have only the minimum required privileges (typically DML operations, not DDL)
- **SQLite3 File Permissions**: For SQLite3 databases, ensure the database file has appropriate file system permissions
- **Connection Security**: Avoid using plaintext passwords in connection strings when possible

The configuration system helps mitigate some security risks by allowing environment variable substitution in the app.ini file, enabling secrets to be provided at runtime rather than stored in configuration files.

**Section sources**
- [database.go](file://modules/setting/database.go#L38-L39)
- [app.ini](file://docker/root/etc/templates/app.ini#L20-L25)