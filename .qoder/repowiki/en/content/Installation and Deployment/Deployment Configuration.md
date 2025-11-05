# Deployment Configuration

<cite>
**Referenced Files in This Document**   
- [setting.go](file://modules/setting/setting.go)
- [database.go](file://modules/setting/database.go)
- [server.go](file://modules/setting/server.go)
- [security.go](file://modules/setting/security.go)
- [cache.go](file://modules/setting/cache.go)
- [session.go](file://modules/setting/session.go)
- [log.go](file://modules/setting/log.go)
- [repository.go](file://modules/setting/repository.go)
- [mailer.go](file://modules/setting/mailer.go)
- [actions.go](file://modules/setting/actions.go)
- [app.ini](file://docker/root/etc/templates/app.ini)
</cite>

## Table of Contents
1. [Configuration Loading Process](#configuration-loading-process)
2. [Configuration Hierarchy](#configuration-hierarchy)
3. [app.ini Structure](#appini-structure)
4. [Database Configuration](#database-configuration)
5. [Server Configuration](#server-configuration)
6. [Security Configuration](#security-configuration)
7. [Cache Configuration](#cache-configuration)
8. [Session Configuration](#session-configuration)
9. [Logging Configuration](#logging-configuration)
10. [Repository Configuration](#repository-configuration)
11. [Mailer Configuration](#mailer-configuration)
12. [Actions Configuration](#actions-configuration)
13. [Deployment Scenarios](#deployment-scenarios)
14. [Common Configuration Issues](#common-configuration-issues)
15. [Configuration Management Best Practices](#configuration-management-best-practices)

## Configuration Loading Process

The configuration loading process in Gitea is managed through the `modules/setting/setting.go` file, which orchestrates the initialization and loading of various configuration components. The process begins with the initialization of the configuration provider through `InitCfgProvider`, which reads the configuration file (typically app.ini) and sets up the configuration provider. This provider is then used throughout the application to access configuration values.

The configuration loading process follows a specific sequence to ensure dependencies are properly resolved. The `LoadCommonSettings` function orchestrates the loading of various configuration sections in a predetermined order: run mode, logging, server, SSH, OAuth2, security, attachment, LFS, time, repository, avatars, repo avatar, packages, actions, UI, admin, API, metrics, camo, i18n, git, mirror, markup, global lock, and other settings. This sequential loading ensures that dependent configurations are available when needed.

The configuration system uses a hierarchical approach where values can be overridden at different levels. The `loadCommonSettingsFrom` function handles the loading of common settings from the configuration provider, ensuring that all necessary configuration sections are properly initialized. The system also includes validation and error handling to ensure configuration integrity, with fatal errors logged when critical configuration issues are detected.

**Section sources**
- [setting.go](file://modules/setting/setting.go#L68-L106)

## Configuration Hierarchy

Gitea's configuration system follows a hierarchical structure with multiple sources that can override each other. The hierarchy, from highest to lowest precedence, consists of command-line flags, environment variables, and the app.ini configuration file. This allows for flexible configuration management across different deployment environments.

Command-line flags provide the highest level of precedence and are typically used for temporary overrides or during initial setup. These flags can override any configuration value and are processed during application startup. Environment variables offer the next level of precedence and are commonly used in containerized environments or when integrating with external configuration management systems. They follow the pattern GITEA_SECTION_KEY (e.g., GITEA_SERVER_DOMAIN) and provide a convenient way to inject configuration values without modifying configuration files.

The app.ini file serves as the base configuration layer and contains the default values for all configuration options. This INI-formatted file is organized into sections, with each section containing key-value pairs for specific functionality. The configuration system processes these sources in order, with higher-precedence sources overriding values from lower-precedence sources. This hierarchical approach enables consistent configuration management across development, testing, and production environments while allowing for environment-specific overrides when necessary.

The configuration loading process ensures that all values are properly validated and that required settings are present before the application starts. This prevents runtime errors due to missing or invalid configuration and provides clear error messages when configuration issues are detected.

**Section sources**
- [setting.go](file://modules/setting/setting.go#L154-L182)

## app.ini Structure

The app.ini configuration file is organized into sections, each dedicated to a specific aspect of Gitea's functionality. Each section contains key-value pairs that configure various aspects of the application. The file follows the standard INI format with section headers enclosed in square brackets and key-value pairs separated by equals signs.

The configuration file includes several core sections:
- **[server]**: Contains server-related settings such as domain, HTTP port, protocol, and root URL
- **[database]**: Configures database connection parameters including type, host, name, user, and password
- **[security]**: Manages security-related settings such as secret keys, installation lock, and authentication options
- **[repository]**: Controls repository-specific settings including root path, default branch, and creation limits
- **[session]**: Configures session management including provider, cookie settings, and lifetime
- **[log]**: Defines logging configuration with mode, level, and output paths
- **[mailer]**: Sets up email delivery configuration for SMTP or sendmail
- **[cache]**: Configures caching behavior and adapter settings

Each section contains specific configuration options relevant to its domain. For example, the [server] section includes settings for HTTP_ADDR, HTTP_PORT, PROTOCOL, and ROOT_URL, while the [database] section contains DB_TYPE, HOST, NAME, USER, and PASSWD. The configuration system supports various data types including strings, integers, booleans, and durations, with appropriate parsing and validation for each type.

The app.ini file also supports environment variable substitution through the use of $VARIABLE_NAME syntax, which is particularly useful in containerized deployments. This allows configuration values to be injected at runtime without modifying the configuration file itself.

**Section sources**
- [app.ini](file://docker/root/etc/templates/app.ini#L1-L62)

## Database Configuration

The database configuration in Gitea is managed through the [database] section of the app.ini file and the corresponding database.go module. This configuration controls the connection to the underlying database system that stores Gitea's data. The system supports multiple database types including MySQL, PostgreSQL, MSSQL, and SQLite3, with the type specified by the DB_TYPE parameter.

Key configuration options include:
- **DB_TYPE**: Specifies the database type (mysql, postgres, mssql, or sqlite3)
- **HOST**: Database server host and port (e.g., localhost:5432)
- **NAME**: Database name
- **USER**: Database username
- **PASSWD**: Database password
- **PATH**: File path for SQLite databases
- **SSL_MODE**: SSL connection mode (disable, require, verify-ca, verify-full)
- **MAX_IDLE_CONNS**: Maximum number of idle connections in the pool
- **MAX_OPEN_CONNS**: Maximum number of open connections to the database
- **CONN_MAX_LIFETIME**: Maximum lifetime of a connection before it's closed
- **AUTO_MIGRATION**: Whether to automatically apply database schema migrations

The database configuration also includes performance-related settings such as connection timeouts, retry behavior, and query logging. The DB_RETRIES parameter specifies the number of times to retry a failed connection, while DB_RETRY_BACKOFF determines the delay between retry attempts. The LOG_SQL option enables SQL query logging for debugging purposes, and SLOW_QUERY_THRESHOLD sets the duration threshold for logging slow queries.

For SQLite databases, additional configuration options include SQLITE_TIMEOUT (query timeout in milliseconds) and SQLITE_JOURNAL_MODE (journaling mode). The system automatically creates the database directory if it doesn't exist and validates the connection parameters during startup.

**Section sources**
- [database.go](file://modules/setting/database.go#L54-L91)

## Server Configuration

The server configuration in Gitea controls the application's network behavior and HTTP server settings. Managed through the [server] section of app.ini and the server.go module, this configuration determines how Gitea accepts and processes incoming requests. The configuration includes settings for network interfaces, ports, protocols, and URL handling.

Key server configuration options include:
- **DOMAIN**: The domain name where Gitea is accessible
- **HTTP_ADDR**: Network interface to bind to (0.0.0.0 for all interfaces)
- **HTTP_PORT**: Port number for HTTP/HTTPS connections
- **PROTOCOL**: Server protocol (http, https, fcgi, fcgi+unix, http+unix)
- **ROOT_URL**: Full public URL of the Gitea instance
- **APP_DATA_PATH**: Path for storing application data
- **STATIC_ROOT_PATH**: Root path for static files
- **ENABLE_GZIP**: Whether to enable GZIP compression for responses
- **OFFLINE_MODE**: Whether to run in offline mode (disables external network requests)

The server configuration also includes SSL/TLS settings for HTTPS connections, including CERT_FILE and KEY_FILE for SSL certificate and key paths. For ACME-based certificate management, the ENABLE_ACME option enables automatic certificate provisioning with Let's Encrypt. The configuration supports proxy protocol for load balancer integration and includes settings for graceful restarts and timeouts.

The system automatically validates the ROOT_URL during startup to ensure it's properly formatted and includes appropriate handling for sub-path deployments. The USE_SUB_URL_PATH option enables sub-path handling for debugging without a reverse proxy. The configuration also includes settings for Pprof debugging, asset URL generation, and landing page selection.

**Section sources**
- [server.go](file://modules/setting/server.go#L277-L303)

## Security Configuration

The security configuration in Gitea is managed through the [security] section of app.ini and the security.go module. This configuration controls authentication, authorization, and other security-related aspects of the application. The security settings are critical for protecting user data and preventing unauthorized access.

Key security configuration options include:
- **INSTALL_LOCK**: Indicates whether the installation is complete and locked
- **SECRET_KEY**: Secret key used for session encryption and other cryptographic operations
- **LOGIN_REMEMBER_DAYS**: Number of days to remember user logins
- **REVERSE_PROXY_AUTHENTICATION_USER**: HTTP header for reverse proxy authentication
- **MIN_PASSWORD_LENGTH**: Minimum required password length
- **PASSWORD_HASH_ALGO**: Password hashing algorithm (pbkdf2, scrypt, argon2)
- **DISABLE_GIT_HOOKS**: Whether to disable Git hooks (recommended for security)
- **DISABLE_WEBHOOKS**: Whether to disable webhooks
- **CSRF_COOKIE_HTTP_ONLY**: Whether to mark CSRF cookies as HTTP-only

The configuration also includes settings for two-factor authentication enforcement, password complexity requirements, and successful token caching. The SECRET_KEY should be kept confidential and rotated periodically in production environments. The system supports loading secret values from external files using URI syntax (e.g., file:///path/to/secret) for improved security.

The security configuration includes protections against reverse proxy abuse with REVERSE_PROXY_LIMIT and REVERSE_PROXY_TRUSTED_PROXIES settings. These control how many proxy headers are trusted and which network ranges are considered trusted proxies. The configuration also includes options for recording user signup metadata and disabling query-based authentication tokens for improved security.

**Section sources**
- [security.go](file://modules/setting/security.go#L0-L185)

## Cache Configuration

The cache configuration in Gitea controls the application's caching behavior and performance optimization. Managed through the [cache] section of app.ini and the cache.go module, this configuration determines how frequently accessed data is stored to improve response times and reduce database load.

Key cache configuration options include:
- **ADAPTER**: Cache adapter type (memory, redis, memcache, twoqueue)
- **INTERVAL**: Cache update interval in seconds
- **HOST**: Connection string for external cache servers
- **ITEM_TTL**: Default time-to-live for cached items
- **LAST_COMMIT_CACHE**: Configuration for last commit cache

The system supports multiple cache adapters, with "memory" being the default in-memory cache. For distributed deployments, Redis and Memcache adapters enable shared caching across multiple instances. The twoqueue adapter implements a two-level queue cache that combines in-memory and persistent storage.

The cache configuration includes specific settings for the last commit cache, which stores information about repository commits to improve performance. The LAST_COMMIT.TTL setting controls how long commit information is cached, while LAST_COMMIT.COMMITS_COUNT limits the number of commits stored in the cache.

For Memcache, the configuration includes a maximum TTL of 30 days, with values exceeding this limit stored as Unix timestamps. The system automatically validates the cache adapter during startup and logs errors for invalid configurations. The cache service is initialized early in the application lifecycle to ensure it's available for other components.

**Section sources**
- [cache.go](file://modules/setting/cache.go#L0-L85)

## Session Configuration

The session configuration in Gitea manages user session storage and behavior. Controlled through the [session] section of app.ini and the session.go module, this configuration determines how user authentication sessions are maintained across requests.

Key session configuration options include:
- **PROVIDER**: Session storage provider (memory, file, redis, mysql, postgres, etc.)
- **PROVIDER_CONFIG**: Provider-specific configuration (e.g., file path for file storage)
- **COOKIE_NAME**: Name of the session cookie
- **GC_INTERVAL_TIME**: Session garbage collection interval in seconds
- **SESSION_LIFE_TIME**: Maximum session lifetime in seconds
- **COOKIE_SECURE**: Whether to mark session cookies as secure (HTTPS only)
- **DOMAIN**: Cookie domain for cross-subdomain sessions
- **SAME_SITE**: SameSite attribute for CSRF protection (none, lax, strict)

The session system supports multiple storage backends, allowing for flexibility in deployment scenarios. The "memory" provider stores sessions in memory, suitable for single-instance deployments. File-based storage writes sessions to the file system, while database and Redis providers enable distributed session storage for clustered deployments.

The configuration includes security features such as secure cookies (requiring HTTPS) and SameSite attributes to prevent CSRF attacks. The session lifetime settings control how long users remain logged in, with garbage collection running periodically to remove expired sessions. The system automatically encrypts session data and validates configuration settings during startup.

For file-based session storage, the PROVIDER_CONFIG specifies the directory path where session files are stored. The system validates that this path is writable and doesn't conflict with other application data paths to prevent data corruption.

**Section sources**
- [session.go](file://modules/setting/session.go#L0-L76)

## Logging Configuration

The logging configuration in Gitea controls the application's logging behavior and output destinations. Managed through the [log] section of app.ini and the log.go module, this configuration determines what information is logged, at what level, and where it's stored.

Key logging configuration options include:
- **MODE**: Default log output mode (console, file, conn)
- **LEVEL**: Minimum log level to record (none, error, warn, info, debug, trace)
- **ROOT_PATH**: Root directory for log files
- **BUFFER_LEN**: Size of the log message buffer
- **ENABLE_SSH_LOG**: Whether to enable SSH-specific logging

The system supports multiple log writers with configurable levels and output formats. The configuration includes specialized sections for different log types:
- **[log.logger.access]**: Access log configuration
- **[log.logger.router]**: Router log configuration
- **[log.logger.xorm]**: Database query log configuration

Each log writer can be configured independently with settings for output mode, level, prefix, expression filters, and formatting flags. File-based log writers support rotation based on size or time, with configurable maximum file sizes, daily rotation, and retention periods. The system also supports network-based logging to TCP, UDP, or Unix socket destinations.

The logging configuration includes template support for access logs, allowing customization of the logged information. The ACCESS_LOG_TEMPLATE setting defines the format of access log entries, with placeholders for remote host, identity, timestamp, request details, response status, and user agent.

The system validates log configuration during startup and provides fallback mechanisms for invalid settings. The logging infrastructure is initialized early in the application lifecycle to ensure all components can write log entries from the beginning of execution.

**Section sources**
- [log.go](file://modules/setting/log.go#L0-L268)

## Repository Configuration

The repository configuration in Gitea controls the behavior and storage of Git repositories. Managed through the [repository] section of app.ini and the repository.go module, this configuration determines how repositories are created, stored, and accessed.

Key repository configuration options include:
- **ROOT**: Root directory for storing repositories
- **DISABLE_HTTP_GIT**: Whether to disable HTTP Git access
- **USE_COMPAT_SSH_URI**: Whether to use compatible SSH URIs
- **GO_GET_CLONE_URL_PROTOCOL**: Protocol for Go get clone URLs
- **MAX_CREATION_LIMIT**: Maximum number of repositories a user can create
- **DEFAULT_BRANCH**: Default branch name for new repositories
- **SCRIPT_TYPE**: Default shell for repository scripts

The configuration includes settings for repository creation policies, including default visibility (public, private, or last user preference) and restrictions on repository adoption. The system supports configurable repository units that determine which features are enabled for repositories, such as issues, pull requests, wiki, and packages.

The repository configuration also includes settings for pull requests, issues, and releases. The pull request configuration controls merge strategies, commit message generation, and work-in-progress detection. Issue settings include configurable lock reasons and pinning limits. Release settings control file type restrictions and pagination.

For repository uploads, the configuration includes limits on file types, maximum file size, and number of files per upload. The system validates repository paths during startup to ensure they're accessible and doesn't conflict with other application data paths.

The configuration supports character set detection for repository files, with a configurable order of character sets to try when detecting file encoding. This helps ensure proper display of files with non-UTF-8 encodings.

**Section sources**
- [repository.go](file://modules/setting/repository.go#L0-L366)

## Mailer Configuration

The mailer configuration in Gitea controls the application's email delivery system. Managed through the [mailer] section of app.ini and the mailer.go module, this configuration determines how Gitea sends notification emails to users.

Key mailer configuration options include:
- **ENABLED**: Whether email delivery is enabled
- **PROTOCOL**: Mail delivery protocol (smtp, smtps, smtp+starttls, sendmail)
- **FROM**: Sender email address
- **SUBJECT_PREFIX**: Prefix for email subjects
- **SMTP_ADDR**: SMTP server address
- **SMTP_PORT**: SMTP server port
- **USER**: SMTP authentication username
- **PASSWD**: SMTP authentication password
- **SENDMAIL_PATH**: Path to sendmail binary

The configuration supports both SMTP and sendmail delivery methods. For SMTP, the system supports various security modes including plain SMTP, SMTPS (SSL/TLS), and STARTTLS. The configuration includes settings for HELO hostname, client certificates, and server certificate validation.

The mailer configuration includes security features such as envelope sender address override and attachment image embedding. The FROM_DISPLAY_NAME_FORMAT setting allows customization of the sender display name using Go template syntax.

The system validates mail configuration during startup, checking for required settings and connection parameters. For SMTP delivery, it attempts to resolve the SMTP server address and warns about insecure connections to non-local servers. The configuration also includes timeout settings for sendmail operations and CRLF conversion options.

The mailer system integrates with other configuration sections, such as the [service] section's REGISTER_EMAIL_CONFIRM setting, which controls whether email confirmation is required for user registration.

**Section sources**
- [mailer.go](file://modules/setting/mailer.go#L0-L300)

## Actions Configuration

The actions configuration in Gitea controls the built-in CI/CD workflow system. Managed through the [actions] section of app.ini and the actions.go module, this configuration determines how workflow jobs are executed, logged, and stored.

Key actions configuration options include:
- **ENABLED**: Whether the actions system is enabled
- **LOG_RETENTION_DAYS**: Number of days to retain workflow logs
- **LOG_COMPRESSION**: Compression method for logs (none, zstd)
- **ARTIFACT_RETENTION_DAYS**: Number of days to retain workflow artifacts
- **DEFAULT_ACTIONS_URL**: Default URL for action references
- **ZOMBIE_TASK_TIMEOUT**: Timeout for tasks that appear to be stuck
- **ENDLESS_TASK_TIMEOUT**: Timeout for tasks that run indefinitely
- **ABANDONED_JOB_TIMEOUT**: Timeout for jobs without recent activity

The configuration includes storage settings for workflow logs and artifacts, allowing them to be stored in different locations with various storage backends. The system supports configurable retention policies to manage disk usage over time.

The actions configuration also includes settings for workflow execution, including timeouts for different task states and detection of abandoned jobs. The SKIP_WORKFLOW_STRINGS setting defines commit message patterns that skip workflow execution, such as "[skip ci]" or "[ci skip]".

The DEFAULT_ACTIONS_URL setting determines where action references without explicit URLs are resolved, supporting both GitHub (https://github.com) and self-hosted Gitea instances. This allows workflow files to reference actions using short names while controlling where they're actually retrieved from.

The system validates actions configuration during startup, ensuring that storage configurations are valid and that retention periods are reasonable. The configuration is designed to balance functionality with resource usage, particularly in environments with limited storage capacity.

**Section sources**
- [actions.go](file://modules/setting/actions.go#L0-L123)

## Deployment Scenarios

Gitea can be configured for various deployment scenarios to meet different performance, scalability, and security requirements. The configuration system supports multiple deployment patterns through appropriate settings in the app.ini file and environment variables.

For high-traffic environments, the configuration should optimize performance through caching, database connection pooling, and resource limits. Key settings include increasing MAX_OPEN_CONNS and MAX_IDLE_CONNS in the [database] section, configuring Redis or Memcache for distributed caching in the [cache] section, and adjusting session lifetime and garbage collection intervals in the [session] section. The server configuration should enable GZIP compression and consider using a reverse proxy for static file serving.

For distributed setups, the configuration must support multiple Gitea instances sharing the same data. This requires external storage for sessions (using Redis or database providers), shared file storage for repositories and attachments (using network file systems), and proper coordination of background tasks. The cache configuration should use Redis or Memcache to ensure consistent caching across instances. The database configuration should include appropriate connection pooling and retry settings to handle network latency.

For security-hardened installations, the configuration should follow security best practices. This includes setting appropriate file permissions, using HTTPS with valid certificates, configuring strong password policies, and restricting access to sensitive endpoints. The security configuration should enforce minimum password length, enable two-factor authentication, and disable unnecessary features like Git hooks and webhooks. The server configuration should run Gitea under a dedicated user account and avoid running as root.

Containerized deployments benefit from environment variable configuration, allowing settings to be injected at runtime without modifying configuration files. The app.ini template in the docker directory demonstrates this approach, using $VARIABLE_NAME syntax for dynamic configuration. This enables consistent deployment across different environments while maintaining configuration flexibility.

**Section sources**
- [app.ini](file://docker/root/etc/templates/app.ini#L1-L62)

## Common Configuration Issues

Several common issues can arise when configuring Gitea, typically related to syntax errors, parameter conflicts, or unexpected behavior due to configuration overrides. Understanding these issues helps prevent deployment problems and ensures smooth operation.

Configuration syntax errors are among the most common issues, typically involving incorrect INI file formatting, missing section headers, or invalid values. The system validates configuration during startup and logs fatal errors for invalid syntax. Common syntax mistakes include using spaces around equals signs, missing quotes for values with special characters, and incorrect boolean values (should be true/false, not yes/no).

Parameter conflicts occur when settings from different sources contradict each other. For example, setting PROTOCOL to "https" while also enabling ACME requires proper certificate configuration. The system detects some conflicts during startup and logs appropriate error messages. Another common conflict is between OFFLINE_MODE and features that require external network access, such as avatar downloads or webhook deliveries.

Unexpected behavior due to configuration overrides can occur when environment variables or command-line flags override app.ini settings. This is particularly common in containerized environments where environment variables take precedence. Administrators should carefully review all configuration sources to understand which values are actually being used.

Path-related issues are common, particularly when paths are not absolute or when multiple components use overlapping paths. The system includes path validation to detect overlapping configurations and prevent data corruption. File permission issues can also occur when Gitea cannot write to configured directories, particularly for logs, sessions, and temporary files.

Database connection issues are frequent, especially when migrating between database types or when network connectivity is problematic. The configuration includes retry mechanisms and connection validation to handle transient issues, but persistent problems require careful review of connection parameters and network configuration.

**Section sources**
- [setting.go](file://modules/setting/setting.go#L68-L106)

## Configuration Management Best Practices

Effective configuration management is essential for maintaining reliable and secure Gitea deployments. Following best practices ensures consistency, security, and ease of maintenance across different environments.

Version control for configuration files is critical. The app.ini file should be stored in a version control system alongside other infrastructure code, allowing for tracking changes, rolling back to previous configurations, and maintaining audit trails. Configuration changes should follow a proper review and approval process before deployment to production.

Secret management is paramount for security. Sensitive values like SECRET_KEY, database passwords, and SMTP credentials should not be stored directly in configuration files. Instead, use external secret management systems or environment variables with restricted access. The file URI syntax (file:///path/to/secret) allows secrets to be stored in separate files with appropriate file permissions.

Configuration validation should be performed before deployment. This includes syntax checking, value validation, and testing in non-production environments. Automated validation scripts can help catch common mistakes before they affect users. The system's built-in validation during startup provides immediate feedback on configuration issues.

Environment-specific configurations should be managed through configuration layers. Use environment variables for differences between development, testing, and production environments, while keeping common settings in the app.ini file. This approach maintains consistency while allowing necessary variations.

Regular configuration reviews and audits help maintain security and performance. Periodically review all configuration settings to ensure they align with current requirements and security policies. Remove unused or deprecated settings and update values as needed based on usage patterns and performance monitoring.

Backup and recovery procedures should include configuration files. Regular backups of the app.ini file and any external configuration sources ensure that configurations can be restored in case of system failure. Document the configuration management process and train administrators on proper procedures.

Monitoring configuration changes and their impact helps identify issues early. Implement logging and alerting for configuration-related events, such as failed startup due to configuration errors or changes to critical security settings. This provides visibility into the configuration state and helps maintain system integrity.

**Section sources**
- [setting.go](file://modules/setting/setting.go#L154-L182)